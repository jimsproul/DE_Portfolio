# Laptop Checkout Steps

Do these before installing other tools

For a **used laptop running Ubuntu 24.04**, I would treat checkout like a burn-in/acceptance test rather than simply verifying that it boots. The most important failure areas on an older laptop are the **SSD/NVMe, RAM, battery, thermals, ports, display, keyboard/trackpad, Wi-Fi, and suspend/resume**.

I recommend doing the tests below **before investing time configuring the machine**.

## 1. Start with a physical inspection

Before running software tests, inspect the machine carefully.

Check the chassis for cracks, bent corners, missing screws, swollen areas around the battery, loose hinges, and signs of liquid damage. Examine the USB, HDMI/DisplayPort, Ethernet, audio, SD-card, and charging ports for damage.

Then manually test:

* Every keyboard key
* Trackpad movement, click, and gestures
* Display brightness
* Dead/stuck pixels
* Webcam
* Microphone
* Speakers and headphone jack
* Wi-Fi
* Bluetooth
* Ethernet
* Every USB port
* Video output
* SD-card reader, if equipped
* AC adapter
* Battery operation
* Lid close/open
* Suspend/resume

A laptop that passes diagnostics but has an intermittent USB port or bad hinge still isn't a good machine.

---

# 2. Establish exactly what hardware you bought

Start by creating a hardware inventory.

```bash
sudo dmidecode -t system
sudo dmidecode -t bios
sudo dmidecode -t memory

lscpu
free -h
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS,MODEL
lspci
lsusb
```

For a shorter overview:

```bash
sudo lshw -short
```

If `lshw` isn't installed:

```bash
sudo apt update
sudo apt install lshw
```

Verify that the reported **CPU, RAM, SSD capacity, Wi-Fi adapter, GPU, and other hardware match what the seller advertised**.

For example, if you purchased a machine advertised as 16 GB RAM and 512 GB NVMe, don't assume Ubuntu is seeing all of it.

---

# 3. Verify Ubuntu itself

Check the release and kernel:

```bash
cat /etc/os-release
uname -a
```

You should see Ubuntu 24.04.x LTS.

Then bring the installation completely current:

```bash
sudo apt update
sudo apt full-upgrade -y
sudo apt autoremove --purge -y
sudo apt clean
```

Reboot:

```bash
sudo reboot
```

After reboot, verify that the package system isn't reporting broken dependencies:

```bash
sudo apt update
sudo apt check
```

Ideally:

```text
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
```

with no dependency errors.

---

# 4. Check for failed system services

This is one of the first things I'd check on a used Linux machine.

```bash
systemctl --failed
```

Ideally:

```text
0 loaded units listed.
```

A failed service isn't automatically evidence of bad hardware, but investigate anything reported.

Also run:

```bash
systemctl --state=failed
```

---

# 5. Look for kernel and hardware errors

Ubuntu's system journal is particularly useful because `journalctl` lets you inspect kernel and system-service messages. ([Ubuntu Manpages][1])

Start with the current boot:

```bash
sudo journalctl -b -p err
```

Then:

```bash
sudo journalctl -b -p warning
```

And kernel messages:

```bash
sudo journalctl -k -b
```

I also like:

```bash
sudo journalctl -k -b | grep -iE \
"error|fail|fault|timeout|reset|corrupt|critical|thermal|nvme|ata|i/o"
```

Don't assume every warning indicates a problem. Linux kernel logs can contain benign ACPI and firmware warnings.

What concerns me are **repeated** messages such as:

```text
I/O error
nvme timeout
controller reset
filesystem error
machine check
hardware error
thermal shutdown
GPU hang
USB disconnect
```

---

# 6. Test the SSD/NVMe carefully

For a used laptop, this is one of the most important tests.

Install SMART utilities:

```bash
sudo apt install smartmontools
```

Identify the drive:

```bash
lsblk
```

Typical devices will be:

```text
/dev/nvme0
/dev/nvme0n1
```

or:

```text
/dev/sda
```

For NVMe:

```bash
sudo smartctl -a /dev/nvme0
```

For SATA:

```bash
sudo smartctl -a /dev/sda
```

SMART is specifically designed to expose drive health information, and `smartctl` supports short and extended self-tests. ([Ubuntu Manpages][2])

Pay particular attention to NVMe values such as:

```text
Critical Warning
Percentage Used
Data Units Written
Power On Hours
Unsafe Shutdowns
Media and Data Integrity Errors
Error Information Log Entries
Temperature
```

You want:

```text
Critical Warning: 0x00
Media and Data Integrity Errors: 0
```

`Percentage Used` gives you useful information about SSD wear.

---

# 7. Run the SSD's internal self-test

If supported:

```bash
sudo smartctl -t short /dev/nvme0
```

or:

```bash
sudo smartctl -t short /dev/sda
```

Wait for the specified test duration, then:

```bash
sudo smartctl -a /dev/nvme0
```

For SATA drives you can also run:

```bash
sudo smartctl -t long /dev/sda
```

The long test can take considerably longer, but it's worthwhile on an unknown used drive. Ubuntu's hardware troubleshooting documentation specifically recommends SMART when investigating potentially failing disks. ([Ubuntu Documentation][3])

---

# 8. Check the filesystem

First determine the filesystem:

```bash
lsblk -f
```

For the root filesystem, don't run a destructive repair operation against the mounted filesystem.

You can at least inspect boot-time filesystem messages:

```bash
sudo journalctl -b | grep -iE "ext4|filesystem|fsck"
```

Look for messages indicating filesystem corruption, repeated recovery, or I/O errors.

If you want a complete `fsck`, boot from an Ubuntu USB drive and run it against the unmounted Linux partition.

---

# 9. Test the RAM

This is another test I would consider mandatory.

Ubuntu's hardware troubleshooting guidance recommends Memtest86-style testing because bad RAM can masquerade as software instability. ([Ubuntu Documentation][3])

If Memtest is available from GRUB, reboot and select:

**Memory test / Memtest86+**

Let it complete **at least one full pass**.

For a laptop you're planning to use as a Docker/Kubernetes/data-engineering lab, I would prefer:

**4 complete passes or overnight testing.**

You want:

```text
Errors: 0
```

Even **one reproducible memory error is unacceptable**.

---

# 10. Stress-test CPU and memory

Install:

```bash
sudo apt install stress-ng
```

Determine CPU count:

```bash
nproc
```

Then run a moderate 10-minute test:

```bash
stress-ng --cpu 0 --vm 2 --vm-bytes 50% --timeout 10m --metrics-brief
```

`--cpu 0` tells stress-ng to use all available CPUs.

While it runs, monitor temperatures.

Install:

```bash
sudo apt install lm-sensors
sudo sensors-detect
```

Then:

```bash
watch -n 2 sensors
```

You're looking for three things:

**stability, temperature, and throttling behavior.**

The laptop should not crash, freeze, spontaneously reboot, or generate hardware errors.

---

# 11. Check CPU thermal throttling

Install:

```bash
sudo apt install linux-tools-common
```

You can inspect CPU frequency with:

```bash
watch -n 1 "grep -E 'cpu MHz' /proc/cpuinfo"
```

Also watch:

```bash
watch -n 1 sensors
```

During `stress-ng`, temperatures will rise rapidly.

The cooling fan should increase speed and temperatures should eventually stabilize rather than continuously climb toward thermal shutdown.

A used laptop with dried thermal paste or a dust-packed heatsink often reveals itself here.

---

# 12. Check the battery

Run:

```bash
upower -i $(upower -e | grep BAT)
```

Look for:

```text
state
energy
energy-full
energy-full-design
percentage
capacity
```

The most interesting comparison is approximately:

```text
energy-full
```

versus:

```text
energy-full-design
```

For example:

```text
energy-full-design: 60 Wh
energy-full:        48 Wh
```

would indicate roughly:

$$
48 / 60 = 80\%
$$

remaining capacity.

For an older used business laptop, some degradation is expected. I'd consider **80%+ quite satisfactory**, **70–80% usable**, and below roughly **60–70%** a candidate for battery replacement, depending on price and intended use.

Also unplug the laptop and actually operate it from the battery for a while. Software-reported health isn't a substitute for observing whether it suddenly drops from, say, 40% to 5%.

---

# 13. Test suspend/resume repeatedly

This is particularly important with Linux laptops.

Run:

```bash
systemctl suspend
```

Wake it.

Verify:

* Display returns
* Keyboard works
* Trackpad works
* Wi-Fi reconnects
* Bluetooth returns
* Audio works
* USB devices return

Do this **5–10 times**.

Also test closing the lid.

Afterward:

```bash
sudo journalctl -b | grep -iE "suspend|resume|sleep"
```

Suspend/resume problems frequently expose BIOS/ACPI compatibility issues.

---

# 14. Check firmware and BIOS updates

Ubuntu uses `fwupd` to manage firmware updates on supported hardware. ([Ubuntu][4])

Check detected devices:

```bash
fwupdmgr get-devices
```

Refresh metadata:

```bash
sudo fwupdmgr refresh
```

Check available updates:

```bash
fwupdmgr get-updates
```

Don't automatically flash firmware in the middle of your initial hardware testing. First establish that the machine is stable and make sure it's connected to AC power.

---

# 15. Test Wi-Fi thoroughly

Identify the adapter:

```bash
lspci -nnk | grep -A3 -i network
```

or for USB Wi-Fi:

```bash
lsusb
```

Check connection:

```bash
nmcli device status
```

Then:

```bash
ip addr
ip route
```

Test network stability:

```bash
ping -c 100 1.1.1.1
```

And DNS:

```bash
ping -c 20 ubuntu.com
```

You want essentially no unexplained packet loss on a good local connection.

---

# 16. Test Ethernet

If the laptop has Ethernet:

```bash
nmcli device status
```

Plug it into your network and check negotiated link information:

```bash
sudo ethtool <interface>
```

For example:

```bash
sudo ethtool enp0s31f6
```

Look for:

```text
Speed: 1000Mb/s
Duplex: Full
Link detected: yes
```

assuming it's a Gigabit Ethernet laptop connected to a Gigabit network.

---

# 17. Test USB ports

Insert a known-good USB device into **every port individually**.

Run:

```bash
lsusb
```

and:

```bash
lsblk
```

For a USB storage device, actually copy a large file to and from it.

Monitor:

```bash
sudo dmesg -w
```

Then connect/disconnect the device.

Repeated USB resets or disconnect/reconnect events while the device isn't being touched can indicate a bad port, cable, controller, or device.

---

# 18. Test display and GPU

Identify the GPU:

```bash
lspci -nnk | grep -A3 -E "VGA|3D|Display"
```

Install utilities:

```bash
sudo apt install mesa-utils
```

Check rendering:

```bash
glxinfo -B
```

Then:

```bash
glxgears
```

The important part isn't the `glxgears` FPS number; you're checking that accelerated graphics work without corruption, freezes, or crashes.

Also manually test:

* Minimum brightness
* Maximum brightness
* Full-screen video
* External monitor
* HDMI/DisplayPort
* Lid movement

Move the lid slowly while watching the display. Intermittent flickering can expose a failing display cable.

---

# 19. Test audio, microphone, camera and Bluetooth

For audio devices:

```bash
pactl list short sinks
pactl list short sources
```

Use Ubuntu's Sound settings to test the speakers and microphone.

For webcam detection:

```bash
ls /dev/video*
```

Install a simple camera application if needed:

```bash
sudo apt install cheese
```

Then:

```bash
cheese
```

For Bluetooth:

```bash
bluetoothctl show
```

Actually pair a Bluetooth device rather than simply verifying that the controller exists.

---

# 20. Check the firmware/ACPI implementation

For a deeper test, Ubuntu provides the Firmware Test Suite (`fwts`), which can examine ACPI, firmware, CPU configuration, thermal behavior, battery behavior, UEFI, and other platform-level functions. ([Ubuntu Wiki][5])

Install it:

```bash
sudo apt install fwts
```

Run the standard batch tests:

```bash
sudo fwts
```

This generates:

```text
results.log
```

Don't panic over every warning; older laptops commonly generate ACPI/firmware warnings. **Failures combined with actual operational symptoms** deserve investigation.

---

# 21. Run a final one-hour burn-in

Once the individual tests pass, I'd run the machine hard for about an hour.

Open one terminal:

```bash
stress-ng --cpu 0 --vm 2 --vm-bytes 50% --timeout 1h --metrics-brief
```

Second terminal:

```bash
watch -n 2 sensors
```

Third terminal:

```bash
sudo journalctl -kf
```

You're looking for:

```text
No crash
No freeze
No reboot
No memory errors
No disk errors
No NVMe resets
No thermal shutdown
No kernel hardware errors
```

That is much more convincing than simply booting Ubuntu and browsing the web.

## My acceptance criteria

For a used Ubuntu laptop that you're going to use as a technical/data-platform lab, I'd use this checklist:

| Component   | Acceptance criterion                                |
| ----------- | --------------------------------------------------- |
| CPU         | Correct model; stable under sustained load          |
| RAM         | Correct capacity; **zero Memtest errors**           |
| SSD/NVMe    | SMART healthy; **zero media/data integrity errors** |
| Filesystem  | No corruption/I/O errors                            |
| Temperature | Stable under sustained load                         |
| Fan         | Operates normally and responds to load              |
| Battery     | Preferably ≥70–80% original capacity                |
| Display     | No objectionable dead pixels/flickering             |
| Keyboard    | Every key works                                     |
| Trackpad    | Movement/click/gestures work                        |
| Wi-Fi       | Stable connection                                   |
| Ethernet    | Correct link speed                                  |
| Bluetooth   | Successfully pairs                                  |
| USB         | Every port tested                                   |
| Video out   | External display works                              |
| Audio       | Speakers/headphones/mic work                        |
| Webcam      | Working                                             |
| Suspend     | Repeated suspend/resume works                       |
| Ubuntu      | Fully updated; no broken packages                   |
| systemd     | No unexplained failed services                      |
| Kernel logs | No recurring hardware/I/O errors                    |
| Firmware    | Recognized and checked                              |
| Burn-in     | At least 1 hour without errors                      |

For the **Dell Latitude 7490 you've been setting up as your Data Engineering Platform Lab**, I would go one step further: perform this entire acceptance test **before installing Docker, PostgreSQL, Airflow, Spark, Kubernetes, Jenkins, Terraform, etc.** That gives you a clean hardware/OS baseline. Once it passes, save the outputs of `lshw`, `smartctl`, `sensors`, `systemctl --failed`, and the journal checks as a **Day-0 baseline**. If something later becomes unstable under Docker/Kubernetes/Spark workloads, you'll have evidence that the underlying laptop was healthy before the lab stack was installed.

[1]: https://manpages.ubuntu.com/manpages/jammy/man1/journalctl.1.html?utm_source=chatgpt.com "Ubuntu Manpage: journalctl - Query the systemd journal"
[2]: https://manpages.ubuntu.com/manpages/noble/man8/smartctl.8.html?utm_source=chatgpt.com "Ubuntu Manpage: smartctl - Control and Monitor Utility for SMART Disks"
[3]: https://help.ubuntu.com/community/FaultyHardware?utm_source=chatgpt.com "FaultyHardware - Community Help Wiki"
[4]: https://ubuntu.com/project/docs/SRU/reference/exception-firmware-updates/?utm_source=chatgpt.com "Firmware Updates - Ubuntu project documentation"
[5]: https://wiki.ubuntu.com/FirmwareTestSuite/Reference?utm_source=chatgpt.com "FirmwareTestSuite/Reference - Ubuntu Wiki"
