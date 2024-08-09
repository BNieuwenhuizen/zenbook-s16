# Zenbook S 16 issues with Linux

I have a new Zenbook S 16 (UM5606WA) with a Ryzen HX AI 370, and since the experience hasn't been flawless this is where I document flaws and my debugging steps.

## SW stack

Archlinux with kernel 6.10.2, systemd-boot for the bootloader.


Note: Kernel 6.11-rc2 seems to break hibernate for me. Need to bisect.

Note:  Updating to linux-firmware-git freezes the display for me at random times, though ssh still seems to work. Need to escalate to AMD.

## No Bluetooth

Need https://patchwork.kernel.org/project/bluetooth/patch/53ccc7377341b64ff3fbdde3df28cbd14f245340.camel@nuclearsunshine.com/ to detect the device correctly.

## Hangs with PSR

Having PSR (Panel Self Refresh) enabled can cause hangs at this point. This is a power saving feature that should help on idle. To disable it, boot the kernel with the parameter `amdgpu.dcdebugmask=0x600`.

How to do this depends on your distro & bootloader. If you use systemd-boot, just add it in `/boot/loader/entries/*.conf`.

## Audio is low/tinny.

It seems by default only the tweeters get used and not the subwoofer, as those get detected as front and rear channels respectively. To fix create a file `/etc/pipewire/pipewire-pulse.conf.d/99-speaker-routing.conf` with the following content:

```
# Pipewire thinks tweeters are 4.0 fronts and woofers are 4.0 rears; upmix.
context.modules = [
    {   name = libpipewire-module-loopback
        args = {
            node.description = "Stereo to 4.0 upmix"
            audio.position = [ FL FR ]
            capture.props = {
                node.name = "sink.upmix_4_0"
                media.class = "Audio/Sink"
            }
            playback.props = {
                node.name = "playback.upmix-4.0"
                audio.position = [ FL FR RL RR]
                target.object = "alsa_output.pci-0000_c4_00.6.analog-surround-40"
                stream.dont-remix = true
                node.passive = true
                channelmix.upmix = true
                channelmix.upmix-method = simple
            }
        }
    }
]
```

Then in pavucontrol or pwvucontrol:

1. Set the default output (or the output of your application) to `Stereo to 4.0 upmix`.
2. In Configuration(pavucontrol)/Cards(pwvucontrol) set `Family 17h/19h HD Audio Controller` to `Analog Surround 4.0 Output + Analog Stereo Input`.

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

Standby power usage seems to be about 30% per day, or 1.0-1.1 W. Note that things like the keyboard backlight keep working in standby, so hopefully there are some ways to improve this still.

## Idle battery usage

A Windows review had idle power usage around `5` Watt (https://www.ultrabookreview.com/68996-asus-zenbook-s16-review/#battery-life-8211-excellent-runtimes-with-hawk-point), however on Linux the lowest I've seen is `6.8` and that includes setting the display to 60 Hz and 1% brightness in KDE, which Ultrabookreview haven't done as part of these numbers.
