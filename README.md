# Ubuntu 24 audio fix for the Lenovo Legion Pro 7 16IAX10H
Fix for Ubuntu 24.04 LTS on the Lenovo Legion Pro 7, based on https://github.com/nadimkobeissi/16iax10h-linux-sound-saga

### Specs of computer used for this guide:

Lenovo Legion Pro 7 16IAX10H laptop with Intel Ultra 9 275HX, NVIDIA 5080 Max-Q, 32GB RAM, and 1TB NVMe SSD running Ubuntu 24.04 LTS. If Secure Boot is required, additional signing steps will be required that are out of the scope of this guide.

## Setup and dependencies

Ideally, first disable Secure Boot from BIOS before continuing. It'll make things a little easier for the custom kernel and can be re-enabled later.

```bash
sudo apt update
sudo apt install -y build-essential libncurses-dev flex bison libssl-dev libelf-dev dwarves bc rsync dkms git wget xz-utils pahole gawk
```

## Clone the repository

```bash
mkdir -p ~/src
cd ~/src
git clone https://github.com/nadimkobeissi/16iax10h-linux-sound-saga.git
cd 16iax10h-linux-sound-saga
```

## Install the AW88399 firmware
```bash
sudo cp -f fix/firmware/aw88399_acf.bin /lib/firmware/aw88399_acf.bin
sudo chmod 0644 /lib/firmware/aw88399_acf.bin
```

```bash
cd ~/src
wget https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.19.11.tar.xz
tar -xf linux-6.19.11.tar.xz
cd linux-6.19.11
cp -f ~/src/16iax10h-linux-sound-saga/fix/patches/16iax10h-audio-linux-6.19.11.patch .
patch -p1 < 16iax10h-audio-linux-6.19.11.patch
```

## Start from your Ubuntu kernel config

```bash
cp /boot/config-$(uname -r) .config
```

```bash
scripts/config --module SND_HDA_SCODEC_AW88399
scripts/config --module SND_HDA_SCODEC_AW88399_I2C
scripts/config --module SND_SOC_AW88399
scripts/config --enable SND_SOC_SOF_INTEL_TOPLEVEL
scripts/config --module SND_SOC_SOF_INTEL_COMMON
scripts/config --module SND_SOC_SOF_INTEL_MTL
scripts/config --module SND_SOC_SOF_INTEL_LNL
```

For NVMe root drives:

```bash
scripts/config --enable BLK_DEV_NVME
scripts/config --enable NVME_CORE
```

Gives custom kernel a unique local version

```bash
scripts/config --set-str LOCALVERSION "-16iax10h-audio"
make olddefconfig
```

Check release string with:

```bash
make kernelrelease
```

It should show 

```bash
6.19.11-16iax10h-audio
```

## Build and install the kernel

```bash
make -j"$(nproc)"
make -j"$(nproc)" modules
sudo make modules_install
sudo make install
```

## Add the required DSP kernel parameter

Open GRUB:

```bash
sudo nano /etc/default/grub
```

Find the following line:

```bash
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"
```

and change it to:

```bash
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash snd_intel_dspcfg.dsp_driver=3"
```

and apply it with:

```bash
sudo update-grub
```

## Generate initamfs for the new kernel

```bash
KREL="$(cat include/config/kernel.release)"
sudo update-initramfs -c -k "$KREL"
```

Reboot with

```bash
sudo reboot
```

Check that the following shoues kernel version `6.19.11-16iax10h-audio` and `snd_intel_dspcfg.dsp_driver=3` as the `/proc/cmdline` output.

```bash
uname -r
cat /proc/cmdline
```

## Install the patched ALSA UCM2 config

```bash
sudo cp -f ~/src/16iax10h-linux-sound-saga/fix/ucm2/HiFi-analog.conf /usr/share/alsa/ucm2/HDA/HiFi-analog.conf
sudo cp -f ~/src/16iax10h-linux-sound-saga/fix/ucm2/HiFi-mic.conf /usr/share/alsa/ucm2/HDA/HiFi-mic.conf
```

Find ALSA card:

```bash
alsaucm listcards
```

If the card is `hw:1`, apply:

```bash
alsaucm -c hw:1 reset
alsaucm -c hw:1 reload
systemctl --user restart pipewire pipewire-pulse wireplumber
amixer -c 1 sset Master 100%
amixer -c 1 sset Headphone 100%
amixer -c 1 sset Speaker 100%
```

If the card is `hw:0`, use `hw:0` and `amixer -c 0` instead.

Verify the fix loaded:

```bash
dmesg | grep -Ei 'aw88399|sof|hda|snd'
lsmod | grep -Ei 'aw88399|sof|snd_hda'
alsaucm listcards
pactl list sinks short
```

## To rollback in case of any issues

At boot, pick the previous Ubuntu kernel from GRUB’s “Advanced options for Ubuntu.”

Once booted into the old kernel, the custom kernel files can found:

```bash
ls /boot | grep 16iax10h
```

Then remove the matching custom files only, for example:

```bash
sudo rm -f /boot/*6.19.11-16iax10h-audio*
sudo rm -rf /lib/modules/6.19.11-16iax10h-audio
sudo update-grub
```

## Resources

https://github.com/nadimkobeissi/16iax10h-linux-sound-saga
https://github.com/marco-giunta/legion-pro7-gen10-audio
https://github.com/marco-giunta/legion-pro7-gen10-audio/blob/legion_audio/docs/secure_boot.md
