# Full Disk Encryption on Garuda Linux backed by TPM 2.0

*Very important note*:
Make a backup of your data. If something fails, this guide could lead to irrevertible data loss!

# Introduction

Tested on:
* garuda-dr460nized-linux-zen-210406 using GUI installer with default partitioning + FDE option.
* 5.11.11-zen1-1-zen

note: It should work on Arch Linux with minor changes but I haven't tested it.

## Preparations
**Note:**
Do not reboot your system until you've finished all the steps or you won't be able to boot. 
1. Edit the file /etc/crypttab and change:

from:
```
# <name>               <device>                         <password> <options>
luks-6eb5f3a9-c2e3-467f-b432-dc7b027d446e UUID=6eb5f3a9-c2e3-467f-b432-dc7b027d446e     /crypto_keyfile.bin luks
```
to:
```sh
# <name>               <device>                         <password> <options>
#luks-6eb5f3a9-c2e3-467f-b432-dc7b027d446e UUID=6eb5f3a9-c2e3-467f-b432-dc7b027d446e     /crypto_keyfile.bin luks
luks-6eb5f3a9-c2e3-467f-b432-dc7b027d446e UUID=6eb5f3a9-c2e3-467f-b432-dc7b027d446e none discard

```
2. Delete the file "/crypto_keyfile.bin"
4. Edit the intial ramdisk conf file `/etc/mkinitcpio.conf`
Change this line from:
`FILES="/crypto_keyfile.bin"`
to:
`#FILES="/crypto_keyfile.bin"`
6. Based on which method you choose set the hooks in `/etc/mkinitcpio.conf`
Change this line `HOOKS="base udev autodetect modconf block keyboard keymap consolefont plymouth encrypt filesystems"` to the following:

**For Method 1 - Clevis:**
`HOOKS="base udev autodetect modconf block keyboard keymap consolefont clevis encrypt filesystems"`

**For Method 2 - Custom**
`HOOKS="base udev autodetect modconf block keyboard keymap consolefont encrypt-tpm encrypt filesystems"`

* * *
## Method 1 - Clevis

**Setup**:

1. Install the following packages.
    ```sh
    pacman --needed -S clevis tpm2-tools luksmeta libpwquality
    ```
2. Add `clevis` binding to your LUKS device
note: set the PCR IDs based on your paranoia settings...
    ```sh
    clevis luks bind -d <device> tpm2 '{"pcr_ids":"0,2,4,7"}'
    ```
3. Install the `clevis` hook
    ```sh
    ./install.sh
    ```

4. Generate `initramfs` image.
    ```sh
    mkinitcpio -p linux-zen
    ```
5. Reboot your system.

Now your disk should get decrypted using the key from TPM.

**Note:**
If integrity on your system is changed you will get prompted to manually enter the password for decryption since TPM will not be able to unseal the key.

It is actually recomennded to test this.
1. Open your UFEI settings 
2. Find the TPM settings (most common location is in security)
3. Delete the keys.
4. Boot
Now you will be notified that the TPM key could not be unsealed, and you will be prompted to enter a password for decryption, to fix this follow the next section **"Clevis Binding"**.

**Regenerate Clevis Binding**
To generate a new Clevis pin after changes in system configuration that result in different PCR values, run**

1. Find the slot used for the Clevis pin
`cryptsetup luksDump /dev/sdX`
2. Remove the Clevis binding, run:
`clevis luks unbind -d /dev/sdX -s keyslot`
3. Add a new Clevis binding.
`
clevis luks bind -d <device> tpm2 '{"pcr_ids":"0,2,4,7"}'
`
4. Reboot, the disk now should be decrypted using the key from TPM.

**Avoid password prompt in GRUB (OPTIONAL)**
1.Rebuilt the EFI image using:
```
objcopy \
    --add-section .osrel="/usr/lib/os-release" --change-section-vma .osrel=0x20000 \
    --add-section .cmdline="/proc/cmdline" --change-section-vma .cmdline=0x30000 \
    --add-section .linux="/boot/vmlinuz-linux-zen" --change-section-vma .linux=0x40000 \
    --add-section .initrd="/boot/initramfs-linux-zen.img" --change-section-vma .initrd=0x3000000 \
    "/usr/lib/systemd/boot/efi/linuxx64.efi.stub" "/boot/efi/EFI/Garuda/grubx64.efi"
```
2. Regenerate the Clevis binding.

* * *
## Method 2 - Custom (WIP)

Create the key
```
dd if=/dev/random of=/root/secret.bin bs=32 count=1
```

#Add key to TPM
```
tpm2_createpolicy --policy-pcr -l sha1:0,2,4,7 -L policy.digest
tpm2_createprimary -C e -g sha1 -G rsa -c primary.context
tpm2_create -g sha256 -u obj.pub -r obj.priv -C primary.context -L policy.digest -a "noda|adminwithpolicy|fixedparent|fixedtpm" -i /root/secret.bin
tpm2_load -C primary.context -u obj.pub -r obj.priv -c load.context
tpm2_evictcontrol -C o -c load.context 0x81000000
rm load.context obj.priv obj.pub policy.digest primary.context
```

...

***
Sources:

https://wiki.archlinux.org/index.php/Trusted_Platform_Module

https://github.com/pawitp/arch-luks-tpm

https://pawitp.medium.com/full-disk-encryption-on-arch-linux-backed-by-tpm-2-0-c0892cab9704

https://github.com/kishorv06/arch-mkinitcpio-clevis-hook

https://bentley.link/secureboot/

https://github.com/archont00/arch-linux-luks-tpm-boot
