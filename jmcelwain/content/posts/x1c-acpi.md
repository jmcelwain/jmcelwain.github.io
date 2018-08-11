+++
tags = ["linux", "arch"]
categories = ["code"]
title = "Restoring S3 Suspend to X1 Carbon (6th Gen) on Linux"
date = "2018-08-11"
+++

My current laptop is the second in the X1 Carbon lineup I've owned. In general, it's been painless running Linux. However, in this current iteration, I was surprised to find that suspend-to-ram functionality isn't enabled in UEFI. Apparently, Lenovo is catering to a new Microsoft sleep state *Windows Modern Standby* that allows for devices to be more easily woken up when in deep sleep (think mobile phones). There's a long thread on the Arch forums about this, and I was able to successfully follow [this blog post](https://delta-xi.net/#056) in order to patch my system, with a few exceptions.


When applying the patch with the following command
```
patch --verbose < X1C6_S3_DSDT.patch
```
Hunk #7 failed to apply. 

Looking at the patch itself, this is an easy removal of two lines, and I was able to complete the patch by hand without issue.


```
@@ -415,9 +351,7 @@
     Name (SS1, 0x00)
     Name (SS2, 0x00)
     Name (SS3, One)
-    One
     Name (SS4, One)
-    One
     OperationRegion (GNVS, SystemMemory, 0xAB54E000, 0x0767)
     Field (GNVS, AnyAcc, Lock, Preserve)
     {systemd-boot
 ```
 
 The instructions to path also are specific to Grub, however, getting things working with `systemd-boot` was pretty easy. I followed the Arch wiki when installing, and my system boots from an entry located in `/boot/entries/arch.conf`. Updating this configuration to map the patched ACPI table into memory on boot required as single additional `initrd` declaration.
 
  ```
initrd /intel-ucode.img
initrd /acpi_override
initrd /initramfs-linux.img
 ```

Modifying the default suspend mode required adding an additional kernel flag to the end of the `options` declaration.
```
options quiet rw intel_pstate=no_hwp mem_sleep_default=deep
```

After rebooting, everything works, and I can suspend the system with `systemctl suspend -i` or closing the lid. While I've found battery life on Linux to be a bit worse than Windows, the ability to close the lid and go into deep sleep makes such a huge difference when taking a laptop to a cafe or on a trip without have to constantly shutdown and restart the system. Hopefully this will be patched in a future BIOS from Lenovo, but the patch itself was very easy and well document, even though it's always a little scary to have to get your hands dirty.
