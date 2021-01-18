# A working QEMU configuration to try the tutorial

Let's install the required packages for this to work:
```
pacman -S qemu swtpm
```

You will also need to download the OVMF bios files from the [Fedora website](https://bodhi.fedoraproject.org/updates/?search=edk2) as the one in the Arch repo are broken and will hang at boot, just pick the first one then click on `Builds`, pick the only result then download a file that should match the name of `edk2-ovmf` (it should be under `noarch`).  
Open and extract the RPM file that you downloaded and copy `OVMF_CODE.secboot.fd` and `OVMF_VARS.secboot.fd` (you'll find them both in the archive) into your VM folder that you'll create below.

Create a folder for your VM files:
```
mkdir myVM
then myVM
```

Download and move the Arch Linus ISO into this folder.

We'll create a swtpm socket so we can setup a TPM inside the VM.  
First, create a folder by the name of `tpm`:
```
mkdir tpm
```

Then create and open the socket:
```
swtpm socket --tpm2 --tpmstate dir=tpm --ctrl type=unixio,path=tpm/swtpm-sock
```
*Note:* the `swtpm` process will close everytime the QEMU VM will shutdown, forcing you to restart it everytime.  
*Note2*: you may encounter an issue with the TPM if you try to follow the instructions 7b where the partition will not unlock. To fix this problem, just fully shut down the VM then restart it, for some reasons, the secret you may add to the TPM will not be available until the next full VM reboot.

Let's create the QEMU disk images:
```
qemu-img create myDisk.img 16G
qemu-img create EfiTool.img 100M
```

You may now boot the VM:
```
qemu-system-x86_64 --enable-kvm -m 2048 \
	-drive if=pflash,format=raw,unit=0,file=OVMF_CODE.secboot.fd,readonly=on \
	-drive if=pflash,format=raw,unit=1,file=OVMF_VARS.secboot.fd \
	-vga virtio -machine pc-q35-2.5 \
	-net user,hostfwd=tcp::10022-:22 -net nic \
	-cdrom archlinux-202x.xx.xx-x86_64.iso \
	-drive file=myDisk.img,format=raw,index=0,media=disk \
	-drive file=EfiTool.img,format=raw,index=1,media=disk \
	-chardev socket,id=chrtpm,path=tpm/swtpm-sock -tpmdev emulator,id=tpm0,chardev=chrtpm \
	-device tpm-tis,tpmdev=tpm0
```
*Note:* change the Arch Linux ISO name to whatever you downloaded.

On the booted VM, start the `sshd` service and change the root password:
```
systemctl start sshd
passwd
```

Now login on your VM:
```
$ ssh localhost -p10022
```

You may now proceed to follow the tutorial.

___

Once you reached the step 6b, we will create a new FAT32 file system stored on the `EfiTool.img` disk so we can enroll the keys.

Open a terminal and type `gdisk /dev/sdb` (it should be sdb).

Type all these commands in this very specific order to create a 100M EFI partition on the `sdb` device:

```
o (create a new GUID partition table)
y (confirm the disk erasing)
n (create a new partition)
[Enter] (pick the default option)
0 (try to use the very first available sector)
[Enter] (pick the default option which is the last available sector on the disk)
ef00 (the code that correspond to the EFI System Partition partition type)
w (write the changes to the disk)
y (confirm the changes)
```

Now create a FAT32 partition on the disk:
```
mkfs.fat -F32 /dev/sdb1
```

Mount it in a temporary folder:
```
mkdir /tmp/efitool
mount /dev/sdb1 /tmp/efitool
```

Then move the auth files and the `EfiTool.efi` binary:
```
cd /root/secureboot_keys; cp *.auth /tmp/efitool
mkdir -p /tmp/efitool/EFI/BOOT
cp /usr/share/efitools/efi/KeyTool.efi /tmp/efitool/EFI/BOOT/BOOTX64.EFI
```

Unmount and delete the temporary folder:
```
unmount /dev/sdb1
rmdir /tmp/efitool
```

You may carry on with the tutorial as usual now.