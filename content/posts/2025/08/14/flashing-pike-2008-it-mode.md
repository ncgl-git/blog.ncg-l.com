---
title: "Flashing a Pike 2008 Raid Controller to IT Mode"
date: 2025-08-14T20:20:40-04:00
draft: false
---

# How to Flash your Pike 2008 / LSI SAS Controller into IT Mode

For my homelab Proxmox nodes, I chose an Asus Z9PA-D8 motherboard. I’ll cover why in another post, but if you’ve ever browsed eBay for these boards, you’ve probably seen them bundled with an Asus Pike 2008 RAID controller. It sits in the 5th PCIe slot seen [here](https://www.manualslib.com/manual/476554/Asus-Z9pa-D8.html?page=39#manual).

Each of my nodes has:

- 1x SSD for the OS
- 4x [2TB 2.5" Seagate BarraCuda drives](https://www.amazon.com/Seagate-BarraCuda-Internal-2-5-Inch-ST2000LM015/dp/B01LX13P71?th=1)

Initially, I had the Pike 2008 configured for RAID 10. Even accounting for these being 5400 RPM laptop drives, my write speeds were awful - about 5 MB/s to the single logical volume per node.

Not wanting to rewire everything to different SATA ports, I decided to flash the Pike into IT mode. My goals were:

- 1. Confirm the RAID controller was the bottleneck.
- 2. Move entirely to a distributed software storage solution. (Yes, I ignored all the “software RAID is better” posts until now.)

Below are the steps I followed, plus the resources I found most helpful.

FreeDOS Link: https://www.freedos.org/download/

Helpful Forum Post: https://forums.servethehome.com/index.php?threads/tutorial-updating-ibm-m1015-lsi-9211-8i-firmware-on-uefi-systems.11462/

1. Download the FreeDos img:
	https://www.freedos.org/download/

2. Find a dedicated USB Boot Drive, and plug it in. Find what device it was assigned:  <br> `lsblk`

3. Write the image to a dedicate USB Boot drive - this formats the drive:
	<br> `sudo dd if=<YOUR DOWNLOAD LOCATION> of=<YOUR USB DRIVE> bs=4M status=progress conv=fsync`

	for example:	<br>
	`sudo dd if=/home/ncgl/Downloads/fd14full.img of=/dev/sdX bs=4M status=progress conv=fsync`

4. Mount the drive: <br>
	`mount /dev/sdX1 /mnt/<SOME MOUNT DIRECTORY YOU CREATED>`

5. In the helpful forum post above, there are only a few files that you need from the zipped archive. Drag and drop them into the base directory using your file explorer or `cp` them with the command line.

	1. `2118it.bin`
	2. `sas2flash/p19/dos/sas2flsh.exe`

6. Unmount <br>
	`umount /mnt/<SOME MOUNT DIRECTORY YOU CREATED>`

7. I've always found the boot process for this motherboard to be confusing, but essentially on boot press F8 for god knows how long until you get a popup for what device you want to boot off of - select your USB.

8. If everything works you should be prompted with a FreeDOS screen - DO NOT choose installation, we just need the shell. 

9. Once in the shell you run these commands: <br>
	`sas2flsh.exe -listall`   <-- you should see your LSI controller printed out from this <br>
	`sas2flsh.exe -o -e 6`    <-- This assumes that this is your only raid controller. If its not, you'll need to change this. <br>
	`sas2flsh.exe -o -f 2118it.bin`  <br>
	`sas2flsh.exe -list`      <-- you should see your LSI controller, but now its 'IT' <br>

10. On reboot you should see each of your drives individually. 

You should be able to recreate using the steps outlined here, but if that forum post is taken down, etc, here is a copy of the same zipped archive [here.](https://prod-uscigni-public.s3.us-east-2.amazonaws.com/blog_assets/LSI-9211-8i.zip)

For complete transparency, I had never done this before and found ChatGPT to be immensely helpful.
