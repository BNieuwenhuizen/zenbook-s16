# Zenbook S 16 issues with Linux

I have a new Zenbook S 16 (UM5606WA) with a Ryzen HX AI 370, and since the experience hasn't been flawless this is where I document flaws and my debugging steps.

## SW stack

Archlinux with kernel 6.10.2, systemd-boot for the bootloader.


Note: Kernel 6.11-rc2 seems to break hibernate for me. Need to bisect.

Note:  Updating to linux-firmware-git freezes the display for me at random times, though ssh still seems to work. Need to escalate to AMD.

## No Bluetooth

Need https://patchwork.kernel.org/project/bluetooth/patch/20240804195319.3499920-1-bas@basnieuwenhuizen.nl/ to detect the device correctly.

## Hang on shutdown

I've had two hangs on shutdown, though it ain't consistent. Need to see if I can get some dmesg from this time.

## Hang on writing initramfs to /boot

Yes ... I have no clue why, but this happened multiple times, and always when updating the initramfs. 

Note that the EFI partition is only 256 MiB and I opted to not create a new one, so with 2 kernels (disabled the fallback initramfs to make space) it is quite full, hovering around 220 of 256 MB.

Currently reworking things to put the kernel/initramfs on a XBOOTLDR partition, will see if fastboot works with that.


## Keyboard backlight keeps fading in/out

And keyboard backlight controls don't work. Looks like my keyboard here is not connected as an USB device, and the asus_wmi driver controls the keyboard backlight.

This drivers uses WMI, which ends up evaluating ACPI functions. These functions then come down to `SLKB` and `GLKB`:


```
^^PCI0.SBRG.EC0.SLKB (IIA1)
Return (One)
```

```
If (^^PCI0.SBRG.EC0.GLKB (One))
{
	Local0 = ^^PCI0.SBRG.EC0.GLKB (0x03)
	Local0 <<= 0x08
	Local0 += ^^PCI0.SBRG.EC0.GLKB (0x02)
	Local0 |= 0x00050000
	Local0 |= 0x00200000
	Local0 |= 0x00100000
	Return (Local0)
}

Return (0x8000)
```

And when looking in these methods we get

```
   Scope (_SB.PCI0.SBRG.EC0)
   {
        ...

        Method (GLKB, 1, NotSerialized)
        {
            If ((Arg0 == One))
            {
                Local0 = (KBLC & 0x80)
                If (Local0)
                {
                    Return (One)
                }
                Else
                {
                    Return (Zero)
                }
            }
            ElseIf ((Arg0 == 0x02))
            {
                Return (KBLV) /* \_SB_.KBLV */
            }
            ElseIf ((Arg0 == 0x03))
            {
                Return (0x80)
            }

            Return (Ones)
        }

        Method (SLKB, 1, NotSerialized)
        {
            KBLV = (Arg0 & 0x7F)
            If ((Arg0 & 0x80))
            {
                Local0 = DerefOf (PWKB [KBLV])   # PWKB is a lookup table mapping 0-3 to 0-255
            }
            Else
            {
                Local0 = Zero
            }

            ST9E (0x1F, 0xFF, Local0)   # seems to be just giving a command to the EC
            Return (One)
        }
```

When broken, `GLKB` will return the value set by `SLKB`, but it seems the value is just ignored. If I reboot from Windows, the keyboard backlight actually works though.

What we can infer here:

* In both the good and bad state `KBLC & 0x80` is true, as `GLKB 1` returns `1` for both of them. (And I can't find anywhere in my DSDT where this is written)
* ACPI caches the value so we don't to a full EC readback on `GLKB`, and this shortcut actually works correctly.


Next steps: probe EC?

There is an ec-probe utility in https://github.com/hirschmann/nbfc but it only dumps zeroes for me ...



## Standby Battery Duration

Had my laptop suspend for 8 hours now and the battery went from 96% to 86%, suggesting ~30% battery per day which isn't great. That said, I bet the keyboard backlight still being on in standby isn't helping.
