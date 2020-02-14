---
layout: post
title: "Towards better Windows VM under GNU/Linux"
date: 2020-02-14 20:27:00 +0800
---

Recently, I have to use some on-line meeting softwares whose desktop version support only Windows. So that once again, I need to install a Windows virtual machine on my laptop running Debian. I used to use qemu with kvm which deliver great IO performance with virtio drivers. However, its graphics performance is unbearable, even with the spice protocol and the qxl driver installed. I need to find a better Windows VM solution for GNU/Linux. Actually, there are lots of good VM softwares for GNU/Linux, for example, VirtualBox, qemu with kvm, and vmware series.

# qemu?

I tried qemu with the VMWare SVGA-II compatible adapter as my first attempt, a.k.a the `-vga vmware` option. It actually performs better than the default standard VGA adapter, though still with minor screen tearing. At first, I think the default driver in the Windows guest is a bottleneck for achieving the full performance of this adapter, and installing the driver provided in vmware's guest tools would help. So I managed to extract the vmware's guest tools: I download the newest ISO for the tools from https://packages.vmware.com/tools/esx/latest/windows/x86/. According to some web pages, running the `setup.exe` in the ISO with a `/a` parameter would extract all the drivers included in the ISO. However, this binary refuses to execute under non-vmware guest environments. So I downloaded and installed vmware workstation player which is free of charge for non-commercial usage. Then install a Windows VM inside the player. And finally succeed extracting the video driver. After I installed this video driver inside the qemu's Windows guest, the VM crashed with vmware related error message from qemu. At first glance, this seems like some version miss match between the driver and qemu. To verify it, I downloaded an older version of the tools whose release date matches my qemu version, but it still does not work, though with different error messages.

Then I tried qemu with the spice protocol. Spice is a remote graphics access protocol, but there are thousands of articles about its great performance on the Internet, so I think it deserves a try. But after appending `-vga qxl -spice port=5900,addr=127.0.0.1,disable-ticketing ` options to qemu command line, and install spice drivers on the Windows guest, the GUI feels even worse than the default VGA adapter with more CPU consumption (for image encoding on the VM side and decoding on the remote side?). spice should be a great protocol for remote access over Internet, but it is not-so-optimized for local usage.

# VirtualBox?

Then I gave VirtualBox a try.

For qemu, `virtio-scsi` disk driver support passing through the `unmap` command with the following options:

```shell
    -drive if=none,file=</path/to/raw/block/device>,format=raw,id=hd,discard=unmap \
    -device virtio-scsi-pci,id=scsi \
    -device scsi-hd,drive=hd \
```

The `unmap` command is used by many thin provision block device (e.g. zfs's zvol, lvm's thinpool) to reclaim space: when the guest VM issues trim on its filesystem (`fstrim` under GNU/Linux), the filesystem's driver would issue `unmap` command on those unused space for the block device. Unlike normal trim on SSD drivers, when these thin provision block device receives `unmap` command, it would release the space used by these blocks. This feature is extremely useful for Windows VM. For example, on each major version update, a `windows.old` folder would be create. Without the `unmap` passing through, deleting `windows.old` would not release the space on the host side.

After some searching, I found something in VirtualBox's document: https://www.virtualbox.org/manual/ch08.html#vboxmanage-storageattach. According to the document, though not supported in GUI, the user could use CLI to enable this feature:

```shell
vboxmanage storageattach <VM name> \
    --storagectl SATA --port 0 --type hdd \
    --discard on \
    --medium '<path/to/xxx.vdi>'
```

The `--discard on` option enables discard support. However, the document says this feature is only supported with VDI disk file format. First, we cannot use raw block device for VDI disk file. Second,  the VDI file's allocation unit is 1MB, only when all disk pages in same MB boundary are `unmap`ed, the space should be released. NTFS allocate disk space in 4K pages by default. I succeed to change the allocation unit of NTFS to 1MB on Windows VM installation: just open a command prompt before the installation start with `Shift-F10`:

```
diskpart
> select disk 0
> select partition 2
> format fs=ntfs quick unit=1024K
> exit
exit
```

and then continue with the installation. However, I experience lock up frequently, the VM would hang and the VirtualBox process would begin to read from disk at around 300MB/s endlessly, then I have to reboot the host OS. I found a 3-years-old bug report in https://www.virtualbox.org/ticket/16450 for it, and the bug is still here 3 years later. Trying to get VirtualBox to work wasted me a LOT of time, since each attempt involves a reboot. And It is still not working.

# vmware workstation player

So there is only one choice left, the vmware workstation player. It do provide good graphics performance over all the other VM softwares. The only shortcoming is no support for passing `unmap` to the raw block device.

As a workaround, I found a way to create a 'dual bootable' Windows VM: it could boot under both vmware workstation player and qemu. When a trimming is needed, I could boot it with qemu and do the trimming.

Since the `discard=unmap` option is only supported in `virtio-scsi` driver when using qemu, the Windows VM needs to support virtio device on early boot. The easier way is, install the Windows VM in qemu first (so that the virtio driver would be emebed in the Windows' 'initramfs'), then boot from the same raw block device in vmware workstation player and install vmware tools and drivers. Of course with dark magics from Windows, one could install the Windows VM in vmware first, and then install virtio driver in qemu (https://superuser.com/questions/1057959/windows-10-in-kvm-change-boot-disk-to-virtio/1095112). But doing the qemu part first makes life MUCH easier.

Please note that, vmware worksatation player boots VMs in efi mode by default, while qemu boots VMs in bios mode by default. I choose to let vmware boots in bios mode. Changing the `firmware = "efi"` line to `firmware = "bios"` in the vmware VM's `*.vmx` file does the trick.

I use zfs on my laptop. I create a zfs sparse zvol with 128KB block size:

```shell
sudo zfs create -V 64G -s -b $((128 * 1024)) -o primarycache=metadata <some pool>/win10
sudo setfacl -m u:<user name>:rw /dev/zvol/<some pool>/win10
```

The `-s` stands for sparse. The `-b` option set the block size. The `-o primarycache=metadata` avoids zfs' ARC cache the VM's disk inside host memory. The `setfacl` changes ACL to the volume, so that no root permission is required to access the volume. After formating the VM's NTFS with 128K allocation unit, the zfs' lz4 compression works great for the volume (hopefully it should perform better than NTFS' compression).

# Conclusion

Finally, I got a Windows VM with acceptable graphics performance under GNU/Linux with `unmap` support. Since the graphics driver inside the guest VM does not provide hardware video decoding, the CPU usage during video playback is higher than native Windows machine. But we have lived without hardware video decoding in web browsers under GNU/Linux for a long time, don't we?