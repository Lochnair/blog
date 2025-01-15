---
title: Flashing a custom ROM on my Redmi Note 11 (China) [evergo]
date: 2025-01-15 13:21:24
tags:
---

## Background

Awhile ago, I bought a [Xiaomi Redmi Note 11](https://www.gsmarena.com/xiaomi_redmi_note_11_(china)-11181.php) (the Chinese version - 21091116AC). Which was not what I had in mind I'll be honest with you, as the listing showed a Snapdragon version of it, and not the Mediatek based model I got.

But fine, it wasn't worth the hassle trying to return it, and it did what I needed it to, so I kept it.

However, since these are pretty old it came with firmware V14.0.2.0.TGBINXN (keep that INXN in mind, it'll be relevant), meaning the security patches were from late 2023, not ideal, even if my main use case for this device is a glorified hotspot.

## The beginning
So I did as one does, and headed over to [XM Firmware Updater](https://xmfirmwareupdater.com/hyperos/evergo/) to find the latest firmware available for this device. The latest version I found was the new HyperOS firmware - OS1.0.1.0.TGBCNXM (China) or OS1.0.1.0.TGBINXM (India).

Now, since my device is the Chinese variant I (foolishly) assumed that I should download the TGBCNXM version and flash that.

But as the firmware it came with weirdly was named TGBINXN and not TGBINXM like I thought it should be, I figured it was prudent to make sure to have a backup of all partitions before I start toying with it.

## Backup
This led me to a nifty tool called [mtkclient](https://github.com/bkerler/mtkclient), which allows you to read/write and erase partitions (and potentially unlock the bootloader as well), without having to modify the software on the device.

I'll assume Linux here, but the mtkclient documentation explains how to use it on MacOS and Windows as well.

Install the latest version from GitHub with pipx (if you haven't used it before, run `pipx ensurepath` to get mtkclient in your path)
```bash
pipx install git+https://github.com/bkerler/mtkclient.git
```

Now, power off your device if you haven't already (and keep the USB cable disconnected), and start mtkclient (there's also a GUI client `mtk_gui` if you prefer using that):
```bash
mtk rl ./evergo_backup/ --skip=userdata
```

What this will do is read all partitions except userdata (but you can include that as well if you want to, I just want to save time and space) into evergo_backup.

To let mtk start reading we need to get our phone into BROM mode.
So with the device turned off, press both volume down/up and insert the USB cable. Then you should see mtk react and start reading in a few second.

If not, keep the volumes buttons pressed and hold the power button in as well until you see a reaction.

Now get yourself a cup of your favorite poison, and wait until it's complete. Check the collapsed section for which files you should be seeing.

<details>

<summary>List of dumped partitions</summary>

```
boot_a.bin
boot_b.bin
boot_para.bin
countrycode.bin
cust.bin
dpm_a.bin
dpm_b.bin
dtbo_a.bin
dtbo_b.bin
efuse.bin
expdb.bin
ffu.bin
flashinfo.bin
frp.bin
gpt_backup.bin
gpt.bin
gsort.bin
gz_a.bin
gz_b.bin
lk_a.bin
lk_b.bin
logo_a.bin
logo_b.bin
mcupm_a.bin
mcupm_b.bin
md1img_a.bin
md1img_b.bin
metadata.bin
misc.bin
nvcfg.bin
nvdata.bin
nvram.bin
otp.bin
para.bin
persist.bin
pi_img_a.bin
pi_img_b.bin
preloader_evergo.bin
proinfo.bin
protect1.bin
protect2.bin
rescue.bin
scp_a.bin
scp_b.bin
sec1.bin
seccfg.bin
spmfw_a.bin
spmfw_b.bin
sspm_a.bin
sspm_b.bin
super.bin
tee_a.bin
tee_b.bin
vbmeta_a.bin
vbmeta_b.bin
vbmeta_system_a.bin
vbmeta_system_b.bin
vbmeta_vendor_a.bin
vbmeta_vendor_b.bin
vendor_boot_a.bin
vendor_boot_b.bin
```

</details>

Should you need to restore your backup, to flash everything back do:

```bash
mtk wl ./evergo_dump/
```

## First attempts at flashing
As my goal was to eventually get over to a custom ROM, I decided I'd be best of using the latest version of MIUI (and not HyperOS), I downloaded V14.0.4.0.TGBINXM, rebooted my phone to the bootloader and tried to run the included `flash_all.sh` script.

When complete it automatically rebooted, but sadly nothing's ever easy and after it attempted to boot a few times, I got the dreaded "NV data is corrupted" error.

So I recovered from my backup like shown above, and decided I'd try to flash a custom ROM directly instead.

I chose a ROM to test, flashing it's boot partition (which includes recovery), ran `fastboot reboot recovery`, and tried to flash the ROM via ADB sideload.

But that too would not work:
> [ERROR:delta_performer.cc(747)] Unable to initialize partition metadata for slot B
>
> [ERROR:download_action.cc(227)] Error ErrorCode::kInstallDeviceOpenError (7) in DeltaPerformer's Write method when processing the received payload -- Terminating processing

## Investigating the stock ROM
So I decided I needed to figure out what on earth was wrong with this stock ROM.
Seeing as the ROM it came with seemed like a global/Indian one, despite the device being the Chinese variant, I had ChatGPT write me a couple scripts to compare my firmware dump to the downloaded V14.0.2.0 for with China and India.

<details>

<summary>Script to copy partititon A files</summary>

```bash
#!/bin/bash

# Check if correct arguments are provided
if [ "$#" -ne 2 ]; then
    echo "Usage: $0 <source_directory> <destination_directory>"
    exit 1
fi

# Define source and destination directories from arguments
source_dir="$1"
destination_dir="$2"

# Check if the source directory exists
if [ ! -d "$source_dir" ]; then
    echo "Error: Source directory '$source_dir' does not exist."
    exit 1
fi

# Create destination directory if it doesn't exist
mkdir -p "$destination_dir"

# Loop through files in source directory
for file in "$source_dir"/*; do
    # Only consider .bin files
    if [[ "$file" == *.bin ]]; then
        filename=$(basename "$file")

        # Process only the 'A' partitions and non-A/B partitions
        if [[ "$filename" == *"_a.bin" ]]; then
            # Remove the '_a' suffix and change extension to .img
            new_filename="${filename/_a/}"
            new_filename="${new_filename/.bin/.img}"

            # Copy the file with the new name to the destination directory
            cp "$file" "$destination_dir/$new_filename"
            echo "Copied: $file -> $destination_dir/$new_filename"
        elif [[ "$filename" != *"_b.bin" ]]; then
            # For non-A/B partitions, just change the extension from .bin to .img
            new_filename="${filename/.bin/.img}"

            # Copy the file with the new name to the destination directory
            cp "$file" "$destination_dir/$new_filename"
            echo "Copied: $file -> $destination_dir/$new_filename"
        fi
    fi
done

echo "All relevant files have been copied and renamed."

```

</details>

<details>

<summary>Script to compare the partitions (Python)</summary>

```python
#!/usr/bin/env python3
import os
import sys
import hashlib

# Color Codes for output
RED = "\033[31m"
GREEN = "\033[32m"
RESET = "\033[0m"

def compare_files(fw_file, dump_file):
    # Get the size of the firmware file
    fw_size = os.path.getsize(fw_file)

    # Open both files for binary reading
    with open(fw_file, 'rb') as fw_f, open(dump_file, 'rb') as dump_f:
        # Read up to the size of the firmware image
        fw_data = fw_f.read(fw_size)
        dump_data = dump_f.read(fw_size)

        # Compare the two files
        if fw_data == dump_data:
            print(f"{GREEN}Match: {os.path.basename(fw_file)} is identical in both firmware and dump.{RESET}")
        else:
            print(f"{RED}Mismatch: {os.path.basename(fw_file)} differs between firmware and dump.{RESET}")

def main(fw_dir, dump_dir):
    # Check if the firmware and dump directories exist
    if not os.path.isdir(fw_dir):
        print(f"Error: Firmware directory '{fw_dir}' does not exist.")
        sys.exit(1)

    if not os.path.isdir(dump_dir):
        print(f"Error: Dump directory '{dump_dir}' does not exist.")
        sys.exit(1)

    # Loop through firmware files in firmware directory
    for fw_filename in os.listdir(fw_dir):
        fw_file = os.path.join(fw_dir, fw_filename)

        # Only consider .img files (firmware image files)
        if fw_filename.endswith('.img') and os.path.isfile(fw_file):
            # Look for the corresponding file in the dump directory (based on file name)
            dump_file = os.path.join(dump_dir, fw_filename)

            if os.path.isfile(dump_file):
                print(f"Comparing {fw_file} to {dump_file}...")
                compare_files(fw_file, dump_file)
            else:
                print(f"Warning: Corresponding dump file for {fw_filename} not found in dump directory.")

    print("Comparison complete.")

if __name__ == '__main__':
    if len(sys.argv) != 3:
        print("Usage: python compare_fw_to_dump.py <firmware_directory> <dump_directory>")
        sys.exit(1)

    fw_dir = sys.argv[1]
    dump_dir = sys.argv[2]

    main(fw_dir, dump_dir)

```

</details>

Using those scripts, I could quickly compare 40+ partitions between the stock firmware and the downloaded V14.0.2.0:

```bash
./copy_part_a.sh ./evergo-dump/ ./cmp/
./compare.py ../Downloads/evergo_in_global_images_V14.0.2.0.TGBINXM_13.0/images ./cmp/
```

**Warning**: The comparision script loads the whole file into memory, so if you have super.bin in there, it'll load 16GB+ into memory, which will quickly starve your system if you don't have enough.

<details>

<summary>Comparison results with Indian version of V14.0.2.0</summary>

```
Comparing ../Downloads/evergo_in_global_images_V14.0.2.0.TGBINXM_13.0/images/pi_img.img to cmp/pi_img.img...
Match: pi_img.img is identical in both firmware and dump.
Comparing ../Downloads/evergo_in_global_images_V14.0.2.0.TGBINXM_13.0/images/scp.img to cmp/scp.img...
Match: scp.img is identical in both firmware and dump.
Comparing ../Downloads/evergo_in_global_images_V14.0.2.0.TGBINXM_13.0/images/tee.img to cmp/tee.img...
Match: tee.img is identical in both firmware and dump.
Warning: Corresponding dump file for userdata.img not found in dump directory.
Comparing ../Downloads/evergo_in_global_images_V14.0.2.0.TGBINXM_13.0/images/dpm.img to cmp/dpm.img...
Match: dpm.img is identical in both firmware and dump.
Comparing ../Downloads/evergo_in_global_images_V14.0.2.0.TGBINXM_13.0/images/cust.img to cmp/cust.img...
Mismatch: cust.img differs between firmware and dump.
Comparing ../Downloads/evergo_in_global_images_V14.0.2.0.TGBINXM_13.0/images/rescue.img to cmp/rescue.img...
Mismatch: rescue.img differs between firmware and dump.
Comparing ../Downloads/evergo_in_global_images_V14.0.2.0.TGBINXM_13.0/images/vbmeta_system.img to cmp/vbmeta_system.img...
Match: vbmeta_system.img is identical in both firmware and dump.
Comparing ../Downloads/evergo_in_global_images_V14.0.2.0.TGBINXM_13.0/images/vbmeta_vendor.img to cmp/vbmeta_vendor.img...
Match: vbmeta_vendor.img is identical in both firmware and dump.
Comparing ../Downloads/evergo_in_global_images_V14.0.2.0.TGBINXM_13.0/images/super.img to cmp/super.img...
Mismatch: super.img differs between firmware and dump.
Comparing ../Downloads/evergo_in_global_images_V14.0.2.0.TGBINXM_13.0/images/mcupm.img to cmp/mcupm.img...
Match: mcupm.img is identical in both firmware and dump.
Comparing ../Downloads/evergo_in_global_images_V14.0.2.0.TGBINXM_13.0/images/boot.img to cmp/boot.img...
Mismatch: boot.img differs between firmware and dump.
Comparing ../Downloads/evergo_in_global_images_V14.0.2.0.TGBINXM_13.0/images/vbmeta.img to cmp/vbmeta.img...
Match: vbmeta.img is identical in both firmware and dump.
Comparing ../Downloads/evergo_in_global_images_V14.0.2.0.TGBINXM_13.0/images/md1img.img to cmp/md1img.img...
Match: md1img.img is identical in both firmware and dump.
Comparing ../Downloads/evergo_in_global_images_V14.0.2.0.TGBINXM_13.0/images/sspm.img to cmp/sspm.img...
Match: sspm.img is identical in both firmware and dump.
Comparing ../Downloads/evergo_in_global_images_V14.0.2.0.TGBINXM_13.0/images/spmfw.img to cmp/spmfw.img...
Match: spmfw.img is identical in both firmware and dump.
Comparing ../Downloads/evergo_in_global_images_V14.0.2.0.TGBINXM_13.0/images/lk.img to cmp/lk.img...
Mismatch: lk.img differs between firmware and dump.
Comparing ../Downloads/evergo_in_global_images_V14.0.2.0.TGBINXM_13.0/images/dtbo.img to cmp/dtbo.img...
Match: dtbo.img is identical in both firmware and dump.
Comparing ../Downloads/evergo_in_global_images_V14.0.2.0.TGBINXM_13.0/images/gz.img to cmp/gz.img...
Match: gz.img is identical in both firmware and dump.
Comparison complete. 
```

</details>

From the results, you can see most of the ROM is the same as the official Indian version of the ROM,
with some key differences.

Despite all that, I was never really ever able to find out why I got NV data corruption problems after flashing the official ROM.

I saw someone that likely has the same weird version of the device on the [Telegram group](https://t.me/mt6833unified_chat) for this phone, which had some success by flashing someone else's backup of their NV data.

So I figured I'd try the same, and flashed that on my device together with the official ROM for India.
And that _did_ end up getting my device to boot with the official ROM, **but** it didn't detect the SIM card anymore, and baseband version was unknown and no IMEI's showing. With custom ROM's showing the same behaviour, something was clearly wrong.

While I can't be sure, my assumption was that it could have something to do with their being a mismatch with what the NV data was saying my IMEI's was, and what my modem actually has in reality.

So I tried to Modem Meta, SN write tool and TFT unlock tool to see if I could correct the NV data I flashed to the correct IMEI - but all to no avail (they wouldn't recognize the phone).


## Give up, or?
With so many failures, and too many hours spent, I was close to giving up. But I realized there was one last thing I could attempt before throwing in the towel - GSI's.

[Generic System Image](https://developer.android.com/topic/generic-system-image?hl=id) (GSI) ROM's - are generic ROM's that can run on a plethora of devices. Since these require less modifications to the stock ROM, I figured _maybe_ that could be an option for me.

So I went to the Project Treble [GSI list](https://github.com/phhusson/treble_experimentations/wiki/Generic-System-Image-%28GSI%29-list) and picked out a ROM. I don't have a strong preference for any of these, but I've tested [LinageOS (A15)](https://github.com/MisterZtr/LineageOS_gsi/releases) by MisterZtr, and [Project Matrixx (A14)](https://github.com/ChonDoit/treble_matrixx_patches/releases/tag/A14) by ChonDoit.

After downloading and decompressing the system image, I rebooted my phone to the bootloader (`adb reboot bootloader`), and then into fastbootd (userspace mode): `fastboot reboot fastboot`.

This is because the bootloader doesn't have access to the inner logical partitions in the super partition (e.g. system), which we need access to.

Now in fastbootd, let's flash our GSI:

```bash
fastboot flash system LineageOS-22.1-20250113-VANILLA-EXT4-GSI.img
```

Sadly though, this would not work either:
```bash
Resizing 'system_a'                                FAILED (remote: 'Not enough space to resize partition')
fastboot: error: Command failed
```

Now that's weird, why doesn't it fit? Let's check the partition size:
```bash
fastboot getvar all 2>&1 | grep partition-size:system_a
(bootloader) partition-size:system_a:0x53225000
```

Converted to decimal that's 1394757632 bytes, so 1.39 GB. Now how big was the LinageOS image again?
```bash
ls -l LineageOS-22.1-20250113-VANILLA-EXT4-GSI.img
-rw-rw-rw-@ 1 lochnair  staff  2572169216 Jan  1  2009 LineageOS-22.1-20250113-VANILLA-EXT4-GSI.img
```

Right, that's a whole lot more (2.57 GB), no wonder it won't work.

So what I tried at first was to manually resize the partition using `fastboot resize-logical-partition`, but that wouldn't work either. Then I realized the product partition was immense, around 4.4 GB, and with everything else, there probably just wasn't enough space for a larger system partition.

What can we do then? Well as it turns out, the product partition is used alongside the system partition (see [here](https://source.android.com/docs/core/architecture/partitions/product-partitions) for details). And as you might guess, our GSI ROM probably won't need any of that.

First attempt was trying to just delete the product_a partition and flash the system image after:

```bash
fastboot delete-logical-partition system_a
fastboot flash system_a LineageOS-22.1-20250113-VANILLA-EXT4-GSI.img
couldn't parse max-download-size 'no'
Sending 'system_a' (2511884 KB)                    FAILED (remote: 'Invalid size')
fastboot: error: Command failed
```

But no, no such luck this time too. After faffing about with this for awhile, I remembered having toyed with the Android partition tools trying to figure out why custom ROM's wouldn't flash on the stock ROM.

In that case, could I build my own super.img to flash with everything needed in it?
And as in turns out, the answer is yes.

I got the necessary tools for this with the `android-tools` package on Arch, but you should be able to find it for your distribution as well.

Firstly, let's unpack the super.bin we dumped earlier to get each individual partition:

```bash
mkdir ./evergo-gsi
cd ./evergo-gsi
lpunpack ../evergo-dump/super.bin
```

Then we need to convert each file to a sparse image:

```bash
img2simg ./mi_ext_a.bin ./mi_ext_a.img
img2simg ./system_a.bin ./system_a.img
img2simg ~/Downloads/LineageOS-22.1-20250113-VANILLA-EXT4-GSI.img ./system_ext_a.img
img2simg ./vendor_a.bin ./vendor_a.img
```

Then we can create the super image with lpmake:

```bash
lpmake \
        --metadata-size 65536 \
        --super-name super \
        --sparse \
        --virtual-ab \
        --metadata-slots 3 \
        --device super:8226127872 \
        --force-full-image \
        --group qti_dynamic_partitions_a:8226127872 \
        --group qti_dynamic_partitions_b:8226127872 \
        --partition mi_ext_a:readonly:282624:qti_dynamic_partitions_a \
        --partition product_a:readonly:347040000:qti_dynamic_partitions_a \
        --partition system_a:readonly:3672169216:qti_dynamic_partitions_a \
        --partition system_ext_a:readonly:778567680:qti_dynamic_partitions_a \
        --partition vendor_a:readonly:1404674048:qti_dynamic_partitions_a \
        --image=mi_ext_a=./mi_ext_a.img \
        --image=system_a=./system_a.img \
        --image=system_ext_a=./system_ext_a.img \
        --image=vendor_a=./vendor_a.img \
        --output ./super.img
```

I've tried to keep it as close to stock, and just lower the size of product_a. I'm not sure if it matters, but I decided to keep it defined, and just not include the image for it.

If you need to compare the super image to your stock dump, remove the `--sparse` argument, so you'll be able to use lpdump on it to compare.

If everything ran successfully, it's now time to flash our super image!
Note that I've included a format of userdata here (as you'll have to), so make sure to have a backup of any data you care about.

The `oem cdms` command isn't very well documented, but it seems to stop dm-verity issues from happening.

```bash
fastboot flash super ./super.img
fastboot format userdata
fastboot oem cdms
fastboot reboot
```

Now, hopefully your phone should restart into LinageOS, or whatever GSI ROM you chose.
If it loops and ends up in recovery, what worked for me was to factory reset the phone there and then reboot again.

With a little luck, you shouldn't have any issues now :)
Do note though, that Android 15 GSI's currently have issues with the fingerprint readers, so if you need that stick to A13 or A14.

At least on Project Matrixx I haven't encountered any issues yet, and I hope it'll stay that way.
If you got this far, thanks for coming along on this crazy and frustrating journey with me :)