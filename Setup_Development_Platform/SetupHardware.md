Here is the GitHub-ready Markdown version. I removed the citation clutter and formatted the procedures, warnings, commands, tables, and workflow so it can be pasted directly into a `README.md`.

# Dell Latitude 7490 Ubuntu Hardware Test and Upgrade Guide

This guide provides a step-by-step procedure for:

1. Testing the major hardware components of a **Dell Latitude 7490** using Ubuntu tools.
2. Establishing a known-good hardware baseline.
3. Upgrading the system to **32 GB RAM**.
4. Replacing the existing SSD with a larger **M.2 NVMe SSD**.
5. Cloning the existing Ubuntu installation using `dd`.
6. Expanding the cloned filesystem to use the larger SSD.
7. Performing final post-upgrade validation.

> [!WARNING]
> The `dd` utility can completely destroy data if the source and destination devices are reversed. Always verify disk **model, size, serial number, and device name** before running `dd`.

---

# 1. Target Configuration

A useful target configuration for a Linux Data Engineering workstation is:

| Component            | Target                               |
| -------------------- | ------------------------------------ |
| System               | Dell Latitude 7490                   |
| Processor            | Intel 8th Generation Core i5/i7      |
| Memory               | 32 GB DDR4                           |
| Memory Configuration | 2 × 16 GB                            |
| Storage              | 1 TB or 2 TB M.2 NVMe SSD            |
| Operating System     | Ubuntu Linux                         |
| Desktop              | GNOME or lightweight LXQt            |
| Primary Use          | Linux / Data Engineering Development |

---

# 2. Prepare Ubuntu for Hardware Testing

Update Ubuntu:

```bash
sudo apt update
sudo apt upgrade -y
```

Install useful hardware diagnostic utilities:

```bash
sudo apt install -y \
    smartmontools \
    nvme-cli \
    lm-sensors \
    memtester \
    stress-ng \
    pciutils \
    usbutils \
    ethtool \
    hdparm \
    alsa-utils \
    v4l-utils \
    libinput-tools
```

Create a directory for hardware reports:

```bash
mkdir -p ~/7490-hardware-test
cd ~/7490-hardware-test
```

---

# 3. Identify the Computer

Check the manufacturer:

```bash
sudo dmidecode -s system-manufacturer
```

Check the model:

```bash
sudo dmidecode -s system-product-name
```

Check the BIOS version:

```bash
sudo dmidecode -s bios-version
```

Expected output should resemble:

```text
Dell Inc.
Latitude 7490
1.xx.x
```

Save the complete DMI report:

```bash
sudo dmidecode > ~/7490-hardware-test/dmidecode.txt
```

---

# 4. Test the CPU

## Identify the Processor

Run:

```bash
lscpu
```

For a shorter report:

```bash
lscpu | grep -E 'Model name|Socket|Core|Thread|CPU\(s\)'
```

Many Latitude 7490 systems use 8th-generation Intel processors such as the Core i5-8350U.

An i5-8350U provides:

```text
4 cores
8 threads
```

## CPU Stress Test

Run a five-minute stress test:

```bash
stress-ng --cpu 0 --timeout 5m --metrics-brief
```

Monitor temperatures in another terminal:

```bash
watch -n 2 sensors
```

If sensors have not been configured:

```bash
sudo sensors-detect
```

Then run:

```bash
sensors
```

Watch for:

* Crashes
* Unexpected shutdowns
* Excessive thermal throttling
* Abnormally high temperatures
* Fan problems

Some thermal throttling during an artificial stress test is not necessarily evidence of hardware failure.

---

# 5. Test the Existing Memory

## Identify Installed RAM

Run:

```bash
free -h
```

Obtain detailed DIMM information:

```bash
sudo dmidecode --type memory
```

For a shorter report:

```bash
sudo dmidecode --type memory | \
grep -E 'Size:|Type:|Speed:|Configured Memory Speed:|Manufacturer:|Part Number:'
```

Example:

```text
Size: 8192 MB
Size: 8192 MB
Type: DDR4
Speed: 2400 MT/s
```

## Target Memory Upgrade

The target configuration is:

```text
32 GB total
2 × 16 GB
DDR4-2400
260-pin SO-DIMM
1.2 V
Non-ECC
```

## Test Existing RAM

Do not allocate all available RAM to `memtester` while Ubuntu is running.

For a system with 16 GB:

```bash
sudo memtester 10G 2
```

For a system with 32 GB:

```bash
sudo memtester 24G 2
```

The arguments mean:

```text
24G = approximately 24 GB tested
2   = two test passes
```

Individual tests should report:

```text
ok
```

---

# 6. Test the Existing SSD

## Identify Storage

Run:

```bash
lsblk -o NAME,SIZE,MODEL,SERIAL,TYPE,FSTYPE,MOUNTPOINTS
```

Then:

```bash
sudo fdisk -l
```

An NVMe disk will normally appear similar to:

```text
/dev/nvme0n1
```

A SATA disk may appear as:

```text
/dev/sda
```

List NVMe devices:

```bash
sudo nvme list
```

## Check NVMe Health

For an NVMe drive:

```bash
sudo nvme smart-log /dev/nvme0
```

You can also use:

```bash
sudo smartctl -a /dev/nvme0
```

For a SATA SSD:

```bash
sudo smartctl -a /dev/sda
```

Important values include:

```text
critical_warning
temperature
available_spare
percentage_used
data_units_read
data_units_written
power_on_hours
unsafe_shutdowns
media_errors
num_err_log_entries
```

Ideally:

```text
media_errors = 0
```

## Basic SSD Read Test

For an NVMe drive:

```bash
sudo hdparm -Tt /dev/nvme0n1
```

This is a basic sanity test rather than a comprehensive SSD benchmark.

## Basic Filesystem Write Test

Change to the temporary directory:

```bash
cd /tmp
```

Create a 2 GB test file:

```bash
sync
dd if=/dev/zero of=ssd-test.bin bs=1G count=2 conv=fdatasync status=progress
```

Remove it:

```bash
rm ssd-test.bin
```

This tests filesystem writes without performing destructive raw-disk writes.

---

# 7. Test Graphics and Display

Identify the graphics controller:

```bash
lspci | grep -Ei 'VGA|Display|3D'
```

Check the active driver:

```bash
sudo lshw -C display
```

For Intel graphics, look for something similar to:

```text
configuration: driver=i915
```

## Check Resolution

Under X11:

```bash
xrandr
```

A Full HD display should normally report:

```text
1920x1080
```

## Check for Display Problems

Display full-screen solid backgrounds using:

* Black
* White
* Red
* Green
* Blue

Inspect the display for:

* Dead pixels
* Stuck pixels
* Backlight problems
* Flickering
* Uneven brightness
* Image retention

---

# 8. Test Keyboard and Touchpad

## Keyboard

Open a text editor or terminal and test:

```text
ABCDEFGHIJKLMNOPQRSTUVWXYZ
abcdefghijklmnopqrstuvwxyz
1234567890
```

Also test:

* Enter
* Space
* Tab
* Backspace
* Shift
* Ctrl
* Alt
* Escape
* Arrow keys
* Function keys
* Keyboard backlight, if installed

## Touchpad

List input devices:

```bash
libinput list-devices
```

Monitor input events:

```bash
sudo libinput debug-events
```

Test:

* Pointer movement
* Left click
* Right click
* Tap-to-click
* Two-finger scrolling
* Multi-touch gestures

Press `Ctrl+C` to stop the event monitor.

---

# 9. Test Audio

## Speakers

List playback devices:

```bash
aplay -l
```

Run:

```bash
speaker-test -c 2 -t wav
```

You should hear:

```text
Front Left
Front Right
```

Press `Ctrl+C` to stop.

## Microphone

List recording devices:

```bash
arecord -l
```

Record ten seconds:

```bash
arecord -d 10 -f cd ~/mic-test.wav
```

Play the recording:

```bash
aplay ~/mic-test.wav
```

Also test the headphone/headset jack if available.

---

# 10. Test the Webcam

List video devices:

```bash
v4l2-ctl --list-devices
```

Then:

```bash
ls /dev/video*
```

For a GUI webcam test, install Cheese:

```bash
sudo apt install -y cheese
```

Run:

```bash
cheese
```

Check:

* Image quality
* Focus
* Frame rate
* Camera activity light
* Microphone operation

---

# 11. Test Wi-Fi

Identify the wireless adapter:

```bash
lspci | grep -i network
```

Check NetworkManager:

```bash
nmcli device
```

List available wireless networks:

```bash
nmcli device wifi list
```

Inspect the wireless interface:

```bash
iw dev
```

Test basic IP connectivity:

```bash
ping -c 10 1.1.1.1
```

Test DNS:

```bash
ping -c 10 google.com
```

Both should work reliably.

---

# 12. Test Ethernet

Connect an Ethernet cable.

List interfaces:

```bash
ip link
```

Determine the Ethernet interface name.

Example:

```text
enp0s31f6
```

Check the link:

```bash
sudo ethtool enp0s31f6
```

Look for:

```text
Link detected: yes
Speed: 1000Mb/s
Duplex: Full
```

Test connectivity:

```bash
ping -c 10 1.1.1.1
```

---

# 13. Test Bluetooth

Check radio status:

```bash
rfkill
```

Start the Bluetooth utility:

```bash
bluetoothctl
```

At the prompt:

```text
power on
scan on
```

Verify that nearby Bluetooth devices appear.

When finished:

```text
quit
```

---

# 14. Test USB Ports

Insert a USB flash drive into each USB port individually.

Check detection:

```bash
lsusb
```

Check storage:

```bash
lsblk
```

Monitor kernel messages while inserting and removing devices:

```bash
sudo dmesg -w
```

Move the USB device between every available USB port.

Verify that every port detects the device reliably.

---

# 15. Test USB-C / Thunderbolt

Install the Thunderbolt management utility if necessary:

```bash
sudo apt install -y bolt
```

Check for Thunderbolt devices:

```bash
boltctl
```

or:

```bash
boltctl list
```

If available, test the USB-C port using:

* USB-C storage
* USB-C dock
* USB-C Ethernet
* USB-C display adapter

Thunderbolt support depends on the specific Latitude 7490 configuration.

---

# 16. Test HDMI and External Display

Connect an external monitor.

Under X11:

```bash
xrandr
```

Verify that Ubuntu detects both the internal and external displays.

Test:

* Mirrored display
* Extended desktop
* Native resolution
* External audio, if supported

---

# 17. Test the SD Card Reader

Insert an SD card.

Run:

```bash
lsblk
```

Then:

```bash
sudo dmesg | tail -30
```

After Ubuntu mounts the card, create a test file:

```bash
touch /media/$USER/*/7490-test.txt
```

Verify the file exists and can be read.

Remove it afterward.

---

# 18. Test Battery Health

List power devices:

```bash
upower -e
```

Identify the battery device.

It will commonly resemble:

```text
/org/freedesktop/UPower/devices/battery_BAT0
```

Display battery information:

```bash
upower -i /org/freedesktop/UPower/devices/battery_BAT0
```

Look for values such as:

```text
energy-full
energy-full-design
percentage
capacity
```

Check current charge:

```bash
cat /sys/class/power_supply/BAT0/capacity
```

Check charging state:

```bash
cat /sys/class/power_supply/BAT0/status
```

## Estimate Battery Health

Use:

```text
Current Full-Charge Capacity
──────────────────────────── × 100
Original Design Capacity
```

Example:

```text
45 Wh
───── × 100 = 75%
60 Wh
```

This battery retains approximately **75% of its original capacity**.

---

# 19. Test Suspend and Resume

Suspend Ubuntu:

```bash
systemctl suspend
```

Wait approximately 30–60 seconds.

Wake the laptop.

Verify that the following still operate:

* Wi-Fi
* Bluetooth
* Audio
* Touchpad
* Keyboard
* Display
* USB

Check the journal:

```bash
journalctl -b | grep -iE 'suspend|resume|error|fail'
```

---

# 20. Check for Hardware and Kernel Errors

Check kernel warnings and errors:

```bash
sudo dmesg --level=err,warn
```

Check serious journal errors:

```bash
journalctl -p 3 -b
```

Some logged messages may be harmless. Investigate repeated hardware, I/O, PCIe, NVMe, memory, or driver errors.

---

# 21. Save the Pre-Upgrade Hardware Baseline

Save a hardware inventory:

```bash
sudo lshw > ~/7490-hardware-test/lshw.txt
```

Save PCI information:

```bash
lspci -nnk > ~/7490-hardware-test/lspci.txt
```

Save USB information:

```bash
lsusb > ~/7490-hardware-test/lsusb.txt
```

Save storage information:

```bash
lsblk -f > ~/7490-hardware-test/lsblk.txt
```

Save memory information:

```bash
free -h > ~/7490-hardware-test/memory.txt
```

You now have a **known-good pre-upgrade baseline**.

---

# 22. Upgrade the RAM

## Target

```text
32 GB total
2 × 16 GB
DDR4-2400
260-pin SO-DIMM
1.2 V
Non-ECC
```

## Step 1 — Record Existing Memory

Run:

```bash
sudo dmidecode --type memory
```

If the system contains:

```text
16 GB + Empty
```

you may only need a compatible second 16 GB DIMM.

If it contains:

```text
8 GB + 8 GB
```

replace both modules:

```text
16 GB + 16 GB
```

## Step 2 — Shut Down

```bash
sudo poweroff
```

Disconnect:

* AC adapter
* USB devices
* Ethernet
* External monitors
* Dock

Do not work on the laptop while it is suspended.

## Step 3 — Remove the Bottom Cover

Place the laptop upside down on a clean, nonconductive surface.

Remove or loosen the bottom-cover screws.

Carefully release and remove the bottom cover.

> [!IMPORTANT]
> Disconnect the internal battery from the motherboard before replacing RAM or the SSD.

Avoid touching exposed electrical contacts.

## Step 4 — Remove Existing Memory

Release the retaining clips on both sides of the SO-DIMM.

The module should rise at an angle.

Slide it carefully out of the socket.

## Step 5 — Install New Memory

Align the notch in the new SO-DIMM with the socket.

Insert the module at approximately the same angle used during removal.

Push it completely into the connector.

Press the module downward until both retaining clips engage.

Repeat for the second module.

## Step 6 — Reconnect and Boot

Reconnect the internal battery.

Reinstall the bottom cover.

Boot Ubuntu.

Check memory:

```bash
free -h
```

Then:

```bash
sudo dmidecode --type memory
```

You should see two modules of approximately:

```text
16384 MB
16384 MB
```

## Step 7 — Test New Memory

Run:

```bash
sudo memtester 24G 2
```

Verify that the tests complete without errors.

---

# 23. Prepare for the SSD Upgrade

The target SSD should be:

```text
M.2 2280
NVMe
PCIe
1 TB or 2 TB
```

A modern PCIe Gen4 NVMe drive can generally operate in an older PCIe system at the interface speed supported by the laptop.

---

# 24. Prepare the New SSD for Cloning

Because the Latitude 7490 normally has one internal M.2 SSD position, use a USB NVMe enclosure:

```text
Existing SSD
     │
     │ Internal M.2
     ▼
Dell Latitude 7490
     │
     │ USB
     ▼
NVMe USB Enclosure
     │
     ▼
New SSD
```

> [!IMPORTANT]
> Make sure the enclosure supports **NVMe M.2 drives**. Some inexpensive M.2 enclosures support SATA M.2 drives only.

---

# 25. Boot From an Ubuntu Live USB

For the safest clone, do not clone the filesystem while the installed Ubuntu system is actively using it.

Create an Ubuntu Live USB.

Boot the Latitude from the USB drive.

Select:

```text
Try Ubuntu
```

Do not start an Ubuntu installation.

Connect the new SSD through the USB NVMe enclosure.

---

# 26. Identify the Old and New SSDs

Run:

```bash
lsblk -o NAME,SIZE,MODEL,SERIAL,TRAN
```

Example:

```text
NAME        SIZE  MODEL              SERIAL       TRAN
nvme0n1     256G  Samsung PM981      XXXXXXXX     nvme
sda         931G  Crucial P3         YYYYYYYY     usb
```

In this **example only**:

```text
/dev/nvme0n1 = OLD 256 GB SSD
/dev/sda     = NEW 1 TB SSD
```

Also run:

```bash
sudo fdisk -l
```

And:

```bash
lsblk -o NAME,SIZE,MODEL,SERIAL
```

> [!CAUTION]
> Do not assume that `/dev/nvme0n1` and `/dev/sda` will represent the same drives on your machine.
>
> Identify each disk by **size, model, serial number, and connection type**.

Before proceeding, establish:

```text
SOURCE      = OLD SSD
DESTINATION = NEW SSD
```

---

# 27. Unmount the SSD Partitions

Check mounted filesystems:

```bash
lsblk
```

If Ubuntu automatically mounted partitions on either SSD, unmount them.

Examples:

```bash
sudo umount /dev/nvme0n1p1
sudo umount /dev/nvme0n1p2
sudo umount /dev/sda1
sudo umount /dev/sda2
```

Only run commands for partitions that actually exist and are mounted.

---

# 28. Clone the SSD With `dd`

## Critical `dd` Rule

Remember:

```text
if = INPUT FILE  = SOURCE
of = OUTPUT FILE = DESTINATION
```

Therefore:

```text
OLD SSD ───────────────────────► NEW SSD
SOURCE                           DESTINATION
  if=                                of=
```

Reversing the devices will overwrite the original disk.

## Example Clone

Assume that you have positively established:

```text
OLD SSD = /dev/nvme0n1
NEW SSD = /dev/sda
```

Run:

```bash
sudo dd \
    if=/dev/nvme0n1 \
    of=/dev/sda \
    bs=64M \
    status=progress \
    conv=fsync
```

After completion:

```bash
sync
```

### Command Explanation

| Argument          | Purpose                        |
| ----------------- | ------------------------------ |
| `if=/dev/nvme0n1` | Input/source disk              |
| `of=/dev/sda`     | Output/destination disk        |
| `bs=64M`          | Copy using 64 MB blocks        |
| `status=progress` | Display progress               |
| `conv=fsync`      | Flush output before completion |

> [!CAUTION]
> Substitute the actual devices identified on your machine. Do **not** blindly copy the example command.

---

# 29. Clone the Entire Disk, Not a Partition

For a complete disk clone, use the disk:

```text
/dev/nvme0n1
```

Do **not** clone only:

```text
/dev/nvme0n1p1
```

A whole-disk clone preserves the disk structure, including:

```text
Physical Disk
│
├── GPT Partition Table
│
├── EFI System Partition
│
├── Ubuntu Partition
│
├── Swap, if present
│
└── Other Partitions
```

---

# 30. Verify the Clone

After `dd` completes:

```bash
sync
```

Inspect the new disk:

```bash
sudo fdisk -l /dev/sda
```

Then:

```bash
lsblk -f
```

Compare the partition tables:

```bash
sudo fdisk -l /dev/nvme0n1
```

and:

```bash
sudo fdisk -l /dev/sda
```

The destination should contain a partition structure corresponding to the original SSD.

---

# 31. Install the New SSD

Shut down the Live Ubuntu environment:

```bash
sudo poweroff
```

Disconnect:

* AC power
* USB Live drive
* USB NVMe enclosure
* Other peripherals

Open the bottom cover.

Disconnect the internal battery.

Remove the original M.2 SSD.

Install the newly cloned M.2 SSD.

Secure the SSD.

Reconnect the internal battery.

Reinstall the bottom cover.

---

# 32. Boot From the New SSD

Power on the Latitude.

Ubuntu should boot from the cloned SSD just as it did from the original drive.

Verify:

```bash
lsblk -o NAME,SIZE,MODEL,FSTYPE,MOUNTPOINTS
```

Then:

```bash
df -h
```

---

# 33. Understand the Remaining Unallocated Space

Suppose you cloned:

```text
256 GB SSD
     │
     │ dd
     ▼
1 TB SSD
```

The new SSD initially contains approximately the same partition layout as the old drive.

Conceptually:

```text
1 TB SSD
┌─────────────────────────┬──────────────────────────────────┐
│ ~256 GB cloned layout   │ ~744 GB unallocated             │
└─────────────────────────┴──────────────────────────────────┘
```

This is expected.

`dd` cloned the original disk layout exactly.

The additional storage must now be assigned to a partition and filesystem.

---

# 34. Expand the Ubuntu Partition

First inspect the disk:

```bash
lsblk
```

Then:

```bash
sudo fdisk -l
```

Install `growpart`:

```bash
sudo apt install -y cloud-guest-utils
```

Suppose the disk contains:

```text
nvme0n1
├── nvme0n1p1    EFI
└── nvme0n1p2    Ubuntu
```

and partition 2 is the final partition on the disk.

Expand it:

```bash
sudo growpart /dev/nvme0n1 2
```

The syntax is:

```text
growpart DISK PARTITION_NUMBER
```

Therefore:

```bash
sudo growpart /dev/nvme0n1 2
```

means:

```text
Disk:      /dev/nvme0n1
Partition: 2
```

> [!IMPORTANT]
> Do not blindly run this command if your partition layout differs. Inspect `lsblk` and `fdisk -l` first.

---

# 35. Expand the Filesystem

Determine the filesystem:

```bash
lsblk -f
```

If the Ubuntu filesystem is `ext4` and resides directly on partition 2:

```bash
sudo resize2fs /dev/nvme0n1p2
```

Verify:

```bash
df -h /
```

Ubuntu should now have access to most of the new SSD capacity.

> [!WARNING]
> If your Ubuntu installation uses **LVM, encryption, or another storage layout**, the expansion procedure is different. Do not blindly run `resize2fs` against an LVM or encrypted layout.

---

# 36. Check the New SSD

List NVMe drives:

```bash
sudo nvme list
```

Check NVMe health:

```bash
sudo nvme smart-log /dev/nvme0
```

Also check SMART data:

```bash
sudo smartctl -a /dev/nvme0
```

Review:

```text
critical_warning
temperature
available_spare
percentage_used
media_errors
power_on_hours
```

For a healthy new drive, `media_errors` should normally be:

```text
0
```

---

# 37. Verify TRIM

Check the periodic TRIM timer:

```bash
systemctl status fstrim.timer
```

You can manually run TRIM:

```bash
sudo fstrim -av
```

Ubuntu should report the amount of space trimmed on supported filesystems.

---

# 38. Final RAM and SSD Validation

Check memory:

```bash
free -h
```

Check storage:

```bash
lsblk -o NAME,SIZE,MODEL
```

The target should now resemble:

```text
Dell Latitude 7490
│
├── Intel 8th-generation CPU
├── 32 GB DDR4
├── 1 TB / 2 TB NVMe SSD
└── Ubuntu Linux
```

## Final Memory Test

```bash
sudo memtester 24G 2
```

## Final CPU Test

```bash
stress-ng --cpu 0 --timeout 10m --metrics-brief
```

## Final SSD Health Check

```bash
sudo smartctl -a /dev/nvme0
```

## Final Temperature Check

```bash
sensors
```

## Final Kernel Error Check

```bash
sudo dmesg --level=err,warn
```

## Final Filesystem Check

```bash
df -h
```

---

# 39. Recommended Overall Upgrade Sequence

```text
1. Update Ubuntu
        ↓
2. Inventory hardware
        ↓
3. Test CPU
        ↓
4. Test existing RAM
        ↓
5. Test existing SSD / SMART
        ↓
6. Test graphics and display
        ↓
7. Test keyboard and touchpad
        ↓
8. Test speakers, microphone and headphones
        ↓
9. Test webcam
        ↓
10. Test Wi-Fi
        ↓
11. Test Bluetooth
        ↓
12. Test Ethernet
        ↓
13. Test every USB port
        ↓
14. Test USB-C / Thunderbolt
        ↓
15. Test HDMI / external monitor
        ↓
16. Test SD reader
        ↓
17. Test battery
        ↓
18. Test suspend / resume
        ↓
19. Review kernel errors
        ↓
20. Save known-good baseline
        ↓
21. Upgrade RAM to 32 GB
        ↓
22. Test new RAM
        ↓
23. Install new NVMe in USB enclosure
        ↓
24. Boot Ubuntu Live USB
        ↓
25. Identify disks by MODEL + SERIAL
        ↓
26. Unmount source/destination partitions
        ↓
27. Clone OLD SSD → NEW SSD with dd
        ↓
28. Verify clone
        ↓
29. Shut down
        ↓
30. Physically install new SSD
        ↓
31. Boot cloned Ubuntu
        ↓
32. Expand partition
        ↓
33. Expand filesystem
        ↓
34. Verify TRIM
        ↓
35. Test new SSD
        ↓
36. Repeat complete hardware validation
```

---

# 40. The `dd` Command to Remember

The generic command is:

```bash
sudo dd \
    if=/dev/OLD_DISK \
    of=/dev/NEW_DISK \
    bs=64M \
    status=progress \
    conv=fsync
```

Remember:

```text
if = INPUT FROM
of = OUTPUT TO
```

Visualize it as:

```text
              dd
       ┌───────────────┐
       │               │
       ▼               ▼

    if=OLD           of=NEW
      │                 ▲
      │                 │
      └─────────────────┘
             COPY

       OLD ───────► NEW
```

> [!CAUTION]
> `dd` does not ask whether you really intended to overwrite the destination.
>
> Before pressing **Enter**, run:
>
> ```bash
> lsblk -o NAME,SIZE,MODEL,SERIAL,TRAN
> ```
>
> Verify the source and destination one final time.

---

# 41. Final System

After successful completion, the Latitude 7490 becomes a capable Linux development platform:

```text
Dell Latitude 7490
│
├── Ubuntu Linux
│
├── 32 GB DDR4 RAM
│
├── 1–2 TB NVMe SSD
│
├── Git
│
├── Python
│
├── PostgreSQL
│
├── Docker
│
├── dbt
│
├── Airflow
│
├── PySpark
│
├── Jenkins
│
├── Terraform
│
├── Kubernetes
│
└── VS Code
```

This provides a strong hardware foundation for a local **Data Engineering and Platform Engineering learning laboratory**.
