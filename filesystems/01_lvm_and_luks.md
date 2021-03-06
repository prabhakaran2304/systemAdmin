# Creating a LUKS-encrypted Logical Volume.

2016-10-20: Updated for the RHEL 7 family.

## LUKS

### Reference

- [Linuxgeek page](https://www.linux-geex.com/centos-7-how-to-setup-your-encrypted-filesystem-in-less-than-15-minutes/)
- [Geekswing page](http://geekswing.com/geek/how-to-encrypt-a-filesystem-on-redhat-6-4centos-6-4-linux-fips-or-no-fips/)
- [CentOS 7 page](https://www.linux-geex.com/centos-7-how-to-setup-your-encrypted-filesystem-in-less-than-15-minutes/)


### Commands

- Don't be afraid of `man` pages! I'm just giving the LVM creation commands a lick and a promise here, because I use them all the time. If you're not familiar, look them up and let me know what you'd like to see expanded here.
  - The really oddball bit, here, is that I'm mirroring my logical volume across two devices. That's definitely worth a perusal of [lvcreate(1M)](http://linux.die.net/man/8/lvcreate). Particularly, you need to specify which Physical Volumes get the mirrors.
- One thing I noticed the first time I tried this is that the `shred` command also has a `--zero` option to write zeroes after it's done shredding the doc. Don't use that. Part of the point of using shred is that you have a bunch of random bytes before and after the looks-random-but-isn't LUKS filesystems.


Create the Logical Volume first. You can examine `/proc/crypto` for supported ciphers.

```
s_pv_devices="/dev/sdb1 /dev/sdc1"
s_vg_name=<volume group name>
i_lv_mb=8192
s_mount_name=<name of final filesystem>
s_cipher='aes-cbc-essiv:sha256'
i_keysize=256

pvcreate ${s_pv_devices}

vgcreate \
  ${s_vg_name} \
  ${s_pv_devices}

lvcreate \
  --mirrors 1 \
  --nosync \
  --size ${i_lv_mb}M \
  --name ${s_mount_name}.enc \
  ${s_vg_name} \
  ${s_pv_devices}

lvdisplay /dev/${s_vg_name}/${s_mount_name}.enc

```

If you mirrored your logical volumes, check that the `Mirrored volumes` item in the output of `lvdisplay` is accurate.

Next, write a bunch of random information to the LV, create a LUKS environment over the top of it, and make a filesystem.

```
shred --verbose \
  --iterations=2 \
  /dev/mapper/${s_vg_name}-${s_mount_name}.enc

cryptsetup \
  --verify-passphrase \
  --cipher ${s_cipher} \
  --key-size ${i_keysize} \
  luksFormat \
  /dev/${s_vg_name}/${s_mount_name}.enc

cryptsetup luksOpen \
  /dev/${s_vg_name}/${s_mount_name}.enc \
  ${s_mount_name}

mkfs.xfs /dev/mapper/${s_mount_name}

```


Put entries in `/etc/crypttab` and `/etc/fstab`. This `crypttab` entry doesn't consider placing the password in a file for automatic mounting, so the `fstab` entry instructs the OS to not mount the filesystem at boot time.

```
cat <<EOCRYPTO >>/etc/crypttab
${s_mount_name} /dev/${s_vg_name}/${s_mount_name}.enc none noauto

EOCRYPTO

s_mountpoint=<mountpoint on the root filesystem>
cat <<EOFSTAB >>/etc/fstab
/dev/mapper/${s_mount_name}  ${s_mountpoint}  xfs  noauto,nodev,nosuid,defaults 1 2

EOFSTAB

```


And on reboot, you enter these commands.

```
cryptsetup luksOpen \
  /dev/${s_vg_name}/${s_mount_name}.enc \
  ${s_mount_name}
mount /${s_mountpoint}

```

Now, how about automating the mount? This is a vulnerability, so *DO NOT DO THIS* if you really want to be secure. You will be leaving the password for this secured filesystem somewhere discoverable. If that password is found, your security is worthless.

First, add a keyfile as a key for your filesystem, and add that keyfile as a valid key for the filesystem.

```
d_keydir=/root/.secret
df_keyfile=${d_keydir}/keyfile

mkdir --parents --mode 700 ${d_keydir}

dd if=/dev/urandom of=${df_keyfile} bs=1024 count=4

chmod 400 ${df_keyfile}


cryptsetup luksAddKey /dev/${s_vg_name}/${s_mount_name}.enc ${df_keyfile}

```

You'll be asked to enter the original key to verify your authority to do this.

Now, identify the Block ID of your encrypted LV (`${s_mount_name}.enc`), and create an entry in `/etc/crypttab` for it.

```
blkid | grep ${s_mount_name} | grep -v enc

s_uuid=< UUID of the encrypted LV >

cat <<EOCRYPT >>/etc/crypttab
${s_mount_name} UUID="${s_uuid}" ${df_keyfile}
EOCRYPT

cat <<EOFSTAB >>/etc/fstab
UUID=${s_uuid} ${s_mountpoint}  xfs nodev,nosuid,defaults 1 2

EOFSTAB

```

(Don't forget to edit /etc/crypttab to remove the manual mount line, and remove the first pass, "`noauto`" line from the entry in /etc/fstab. Mounting by UUID isn't much more secure, but every little bit helps.)

Reboot the host and verify that the filesystem gets correctly mounted at boot time.


