# Ubuntu Partition Recovery

Here's how to recover a Ubuntu/Linux partition after it is "accidentally" deleted by a Windows 10 update. Apparently, Windows 10 update fell on its head while trying to do its thing.

## Background
The computer has both Windows 10 and Ubuntu 14 installed using the Grub bootloader to dual boot between to two operating systems.

When the Linux partition is deleted, the Grub bootloader no longer works. When you attempt to boot the computer, it enters a special rescue mode, and presents an error.

```
error: no such partition.
Entering rescue mode...
grub rescue>
```

This is because part of the bootloader, the mini rescue bootloader, lives between the master boot record and the first partition. The rest of the Grub bootloader lives in a directory on the Linux partition. If we can repair the Linux partition then we can repair the Grub bootloader and return the machine back to normal.

## Approach
Clone the existing drive so we have a backup in case things go wrong, find the start and end sectors of the deleted partition, and recover the Linux partition.

## Step 🍊: Remove the Original Drive
Shutdown the computer, remove the original drive, and set it aside for now.

## Step 🍎: Create a Bootable USB Stick
Download an ISO matching your OS version. Mine is Ubuntu 14 which I downloaded from [here](http://releases.ubuntu.com/14.04/ubuntu-14.04.5-desktop-amd64.iso). Create a bootable USB stick/flash drive using the instructions found [here](https://tutorials.ubuntu.com/tutorial/tutorial-create-a-usb-stick-on-ubuntu).

## Step 🍐: Erase the Spare Drive

Erasing the spare drive makes it easier to differentiate between the original and the spare drive. Attach the spare drive to the computer and boot in to Linux using the USB stick. For Ubuntu, select "Try Ubuntu" from the Welcome screen. Make sure that only the spare drive is attached to the computer at this point.

| Device Path | Name |
| --- | --- |
| /dev/sda | Spare Drive |


Using the `GParted` partition editor, delete the existing partitions from the spare drive. Note: You must unmount a partition before you can delete it. The result should be a single unallocated partition.

![Empty Spare Drive](https://d.pr/FREE/PUOGpp+)

## Step 🍋: Clone the Original Drive
Power off the computer and attach both the original drive and the spare drive.

| Device Path | Name |
| --- | --- |
| /dev/sda | Spare/Empty Drive |
| /dev/sdb | Original Drive |

Boot into Linux using the USB stick and verify which drives are attached to which device files. Use the `fdisk` command to display the partition information for the attached drives.

```
sudo fdisk -l
```


![Insert image here](https://d.pr/free/i/FCNOwN+)


Clone the original drive (/dev/sdb) to the spare drive (/dev/sda) using the dd command described [here](https://www.howtogeek.com/howto/19141/clone-a-hard-drive-using-an-ubuntu-live-cd/) and [here](https://askubuntu.com/questions/523037/how-would-i-speed-up-a-full-disk-dd). Check one last time that you have the dd options for input file (if) and output file (of) assigned correctly. **No really, make sure you are copying from the original drive to the spare/empty drive.**

```
sudo dd if=/dev/sdb of=/dev/sda bs=100M
```
The dd command can take a while to clone a drive. In my case, a 500GB drive took about 1.5 hours to complete (~100MB/sec transfer rate).

If you want to get status updates for the dd command you can use the method described [here](https://askubuntu.com/questions/215505/how-do-you-monitor-the-progress-of-dd). Open a new terminal and use the following command.

```
watch -n5 'sudo kill -USR1 $(pgrep ^dd)'

```
Switch back to the terminal running the dd command to view the status updates. If you are using a hard drive with rotating disks/platters, the transfer rate will drop throughout the clone/copy process.

![Insert Image Here](https://d.pr/FREE/QD0sB0+)


After the clone is complete, perform a sanity check. Verify that the partitions for the original  drive and the copy match by using the `fdisk` command.

```
sudo fdisk -l
```

![Image](https://d.pr/FREE/upwlsj+)

Next, power down the computer, remove the clone and put it somewhere safe. 


## Step 🍌: Find the Magic Numbers
In order to recover the Linux partition we need the first sector and the last sector of the deleted partition. This is easier than you might think using `GParted` partition editor. 

| Device Path | Name |
| --- | --- |
| /dev/sda | Original Drive |


With the the original drive attached to the computer, boot into Linux using the USB stick, and start the `GParted` tool.

Note: You may see a warning about a corrupted partition table on `/dev/sdb`. This is a problem with the USB stick which you can disregard. I clicked "Yes" to confirm that my drive was using a GPT partition table, and "Ok" to acknowledge that the GPT table was corrupt and that a backup table is being used.

The main `GParted` window should appear and the tool will scan the drive. In the figure below, you can see the deleted partition is listed as unallocated. Right click on the unallocated partition and select Information.

![Image](https://d.pr/FREE/cBiSkD+)

The information dialog box will display the first sector and the last sector. These are the magic numbers we will need to recover the deleted partition. Record the first and last sector numbers using a text editor like gedit.

![Image](https://d.pr/FREE/nDDINK+)

## Step 🍉: Fix the Partition Table
Once we have the magic numbers, we can edit the partition table to recover the partition using the `parted` partition editor tool described [here](https://ubuntuforums.org/showthread.php?t=1376383) and [here](https://ubuntuforums.org/showthread.php?t=370121). Run the `parted` command to enter interactive mode.

```
sudo parted /dev/sda
```

You will now see a `parted` prompt. First, set the units to sectors.

```
(parted) unit s
```

Then, enter the rescue command with the start sector and end sector. Substitute your own numbers for 879626238 (the start sector) and 967852031 (the end sector).

```
(parted) rescue 879626238 967852031
```
You should see the `parted` tool respond that a logical partition was found and ask if you would like to add it to the partition table. Respond "Yes"!

![Image](https://d.pr/FREE/OYvjdV+)

Quit the the partition editor.

```
(parted) quit
```

Switch back to the `GParted` tool and refresh the table by selecting  "Refresh Devices" from the menu. You should now see the recovered Linux partition!

![Image](https://d.pr/FREE/cQi1fr+)


## Step 🍓: Reboot
Shutdown Ubuntu/Linux and reboot. The computer should now boot normally with Grub. In my case, I booted into Windows to let it finish the update process that started this whole mess :)



