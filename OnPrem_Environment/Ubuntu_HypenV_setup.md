Yes. With **Windows 11 Pro and 64 GB RAM**, Hyper-V is an excellent way to run a substantial Ubuntu development VM. For the kind of data-engineering lab you have been building, I would use **8 virtual CPUs, 16 GB RAM, and a 160 GB dynamically expanding VHDX** as the initial configuration. You can enlarge the disk later.

Ubuntu **26.04.1 LTS Desktop is currently available**, with the official AMD64 desktop ISO around 6 GB. ([Ubuntu][1])

## Recommended configuration

| Component      |                         Recommendation |
| -------------- | -------------------------------------: |
| Hypervisor     |                      Microsoft Hyper-V |
| VM Generation  |                       **Generation 2** |
| CPU            |               **8 virtual processors** |
| RAM            |                              **16 GB** |
| Dynamic Memory |                      **Off initially** |
| Virtual Disk   | **160 GB VHDX, dynamically expanding** |
| OS             |   Ubuntu **26.04.1 LTS Desktop AMD64** |
| Firmware       |                                   UEFI |
| Secure Boot    |             Enabled, Microsoft UEFI CA |
| Network        |       Hyper-V Default Switch initially |
| Checkpoints    |                 Production checkpoints |
| VM storage     |            SSD/NVMe strongly preferred |

Because your host has 64 GB RAM, reserving a fixed 16 GB for Ubuntu still leaves roughly 48 GB for Windows. That's preferable to aggressive dynamic-memory management for a development VM running PostgreSQL, Docker, Airflow, Spark, Kubernetes, etc.

---

# 1. Verify hardware virtualization

Before enabling Hyper-V, verify that virtualization is enabled.

Press:

```text
Ctrl + Shift + Esc
```

Select:

```text
Performance
→ CPU
```

Look for:

```text
Virtualization: Enabled
```

If it says:

```text
Virtualization: Disabled
```

you'll need to enable Intel VT-x/VT-d or AMD-V/SVM in your motherboard's UEFI/BIOS.

You can also check from PowerShell:

```powershell
systeminfo
```

Near the bottom, look for the Hyper-V requirements.

---

# 2. Enable Hyper-V

Windows 11 Pro supports Hyper-V. Canonical also documents Hyper-V as supported on Windows 11 Pro, Enterprise, and Education. ([Ubuntu][2])

The simplest method is PowerShell.

Open:

```text
Start
→ search "PowerShell"
→ Run as administrator
```

Run:

```powershell
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All
```

Restart Windows when prompted.

Alternatively:

```text
Control Panel
→ Programs
→ Turn Windows features on or off
```

Enable:

```text
☑ Hyper-V
    ☑ Hyper-V Management Tools
    ☑ Hyper-V Platform
```

Then reboot.

---

# 3. Verify Hyper-V

After restarting:

```text
Start
→ Hyper-V Manager
```

You should see your Windows computer listed on the left.

You can also check with PowerShell:

```powershell
Get-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V
```

You want:

```text
State : Enabled
```

---

# 4. Download Ubuntu 26.04.1

Download the official:

```text
Ubuntu 26.04.1 LTS Desktop
AMD64
```

from Canonical. ([Ubuntu][1])

The filename should be:

```text
ubuntu-26.04.1-desktop-amd64.iso
```

Don't use ARM64 on a normal Intel/AMD Windows desktop.

[Ubuntu 26.04.1 Desktop download](https://ubuntu.com/download/desktop?utm_source=chatgpt.com)

I'd save it somewhere permanent such as:

```text
C:\ISO\ubuntu-26.04.1-desktop-amd64.iso
```

rather than leaving it in Downloads.

---

# 5. Optionally verify the ISO checksum

This is good practice.

Open PowerShell:

```powershell
Get-FileHash C:\ISO\ubuntu-26.04.1-desktop-amd64.iso -Algorithm SHA256
```

You'll get something resembling:

```text
Algorithm  Hash
---------  ----
SHA256     A1B2C3...
```

Compare that value against Canonical's official `SHA256SUMS` file.

Canonical publishes the checksum alongside the ISO. ([Ubuntu Releases][3])

---

# 6. Decide where the VM will live

If possible, put the VM on your fastest NVMe SSD.

For example:

```text
D:\Hyper-V\
```

with:

```text
D:\Hyper-V\Ubuntu2604\
```

Eventually you'll have something like:

```text
D:\Hyper-V\Ubuntu2604\
├── Virtual Machines\
└── Virtual Hard Disks\
    └── Ubuntu2604.vhdx
```

Avoid placing a high-I/O VM on a slow external USB drive.

For your planned data-engineering lab, **SSD latency matters more than it might for a generic Linux VM**.

---

# 7. Create the virtual machine

Open:

```text
Hyper-V Manager
```

Right-click your computer and select:

```text
New
→ Virtual Machine
```

The wizard starts.

---

# 8. Name the VM

Enter:

```text
Ubuntu-26.04-DE
```

I like descriptive VM names because you may later add things such as:

```text
Ubuntu-26.04-DE
Ubuntu-26.04-K8s
Ubuntu-26.04-Server
Windows-Test
```

Check:

```text
☑ Store the virtual machine in a different location
```

For example:

```text
D:\Hyper-V\Ubuntu2604
```

Select **Next**.

---

# 9. Choose Generation 2

This is important.

Select:

```text
Generation 2
```

Do **not** use Generation 1 for a new Ubuntu 26.04 VM.

Generation 2 gives you:

```text
UEFI firmware
Secure Boot
modern virtual hardware
SCSI boot disks
better Hyper-V architecture
```

Select:

```text
Next
```

---

# 10. Assign memory

Enter:

```text
16384 MB
```

because:

```text
16 GB × 1024 = 16384 MB
```

For your machine, I recommend initially clearing:

```text
☐ Use Dynamic Memory for this virtual machine
```

So Ubuntu always receives:

```text
16 GB
```

Microsoft supports Dynamic Memory with 64-bit Ubuntu, but fixed memory is predictable and works particularly well for databases, JVM workloads, Spark, Docker and Kubernetes. ([Microsoft Learn][4])

You can experiment with Dynamic Memory later.

---

# 11. Configure networking

Select:

```text
Default Switch
```

Hyper-V normally creates this automatically.

This gives the VM:

```text
Ubuntu VM
    │
    │ virtual NIC
    ▼
Hyper-V Default Switch
    │
    │ NAT
    ▼
Windows 11
    │
    ▼
Internet
```

This is the easiest starting configuration.

Don't create an External Virtual Switch yet unless you specifically need the Ubuntu VM to appear as an independent machine directly on your physical LAN.

---

# 12. Create the virtual disk

Choose:

```text
Create a virtual hard disk
```

I'd configure:

```text
Name: Ubuntu2604.vhdx

Location:
D:\Hyper-V\Ubuntu2604\Virtual Hard Disks\

Size:
160 GB
```

Hyper-V uses VHDX.

A dynamically expanding 160 GB disk **does not immediately consume 160 GB** of physical SSD space.

It grows as Ubuntu stores data.

For example:

```text
Maximum VM disk:       160 GB
Actual initial usage:  perhaps ~15–25 GB
```

For your intended lab:

```text
120 GB   workable
160 GB   recommended
200 GB   excellent
```

I'd choose **160 GB** unless host storage is tight.

---

# 13. Attach the Ubuntu ISO

At:

```text
Installation Options
```

choose:

```text
Install an operating system from a bootable image file
```

Browse to:

```text
C:\ISO\ubuntu-26.04.1-desktop-amd64.iso
```

Then:

```text
Next
→ Finish
```

Don't start the VM yet.

---

# 14. Configure the CPU

Right-click:

```text
Ubuntu-26.04-DE
```

Select:

```text
Settings
```

Go to:

```text
Processor
```

Change:

```text
Number of virtual processors: 8
```

So the core configuration becomes:

```text
Host
Windows 11 Pro
64 GB RAM
        │
        ▼
Hyper-V
        │
        ▼
Ubuntu 26.04.1
8 vCPU
16 GB RAM
160 GB VHDX
```

---

# 15. Check Secure Boot

Still under VM Settings:

```text
Security
```

Enable:

```text
☑ Enable Secure Boot
```

For the template choose:

```text
Microsoft UEFI Certificate Authority
```

This is the important part.

Do **not** leave it on the Windows-only certificate template if Ubuntu refuses to boot.

The configuration should be approximately:

```text
Secure Boot: Enabled

Template:
Microsoft UEFI Certificate Authority
```

---

# 16. Configure checkpoints

Go to:

```text
Checkpoints
```

Enable:

```text
☑ Enable checkpoints
```

and preferably:

```text
Production checkpoints
```

with:

```text
☑ Create standard checkpoints if production checkpoint fails
```

Checkpoints will be useful while experimenting with:

```text
Docker
Kubernetes
PostgreSQL
Airflow
Spark
Terraform
Jenkins
```

but don't use checkpoints as your only backup mechanism.

---

# 17. Review boot order

Under:

```text
Firmware
```

verify that the virtual DVD drive containing the Ubuntu ISO is ahead of the VHDX for the initial installation.

Something like:

```text
DVD Drive
Hard Drive
Network Adapter
```

After installation Ubuntu will boot from the VHDX.

---

# 18. Start the VM

Right-click the VM:

```text
Connect
```

Then click:

```text
Start
```

You should see the Ubuntu boot menu.

Choose:

```text
Try or Install Ubuntu
```

---

# 19. Start the Ubuntu installer

Once the Ubuntu graphical installer starts, select:

```text
Install Ubuntu
```

Choose your:

```text
Language
Keyboard layout
Timezone
```

Networking should normally work automatically through the Hyper-V Default Switch.

---

# 20. Choose installation type

Ubuntu 26.04's exact installer wording may differ slightly, but choose the normal/default Ubuntu Desktop installation.

For your development environment, install the normal collection of applications rather than trying to minimize the OS aggressively.

You have plenty of resources:

```text
8 CPU
16 GB RAM
160 GB disk
```

---

# 21. Disk configuration

When Ubuntu asks where to install, choose the option equivalent to:

```text
Erase disk and install Ubuntu
```

This sounds dangerous, but **inside Hyper-V it refers to your virtual 160 GB VHDX**.

It does **not** mean your Windows physical disk.

Ubuntu sees something similar to:

```text
/dev/sda
    160 GB
```

but that "drive" is actually:

```text
Ubuntu2604.vhdx
```

on Windows.

Unless you have a specific reason otherwise, allow Ubuntu to create its standard partitions automatically.

---

# 22. Create your Linux account

Enter your information.

For example:

```text
Your name:         Jim Sproul
Computer name:     ubuntu-de
Username:          jim
Password:          ********
```

I recommend requiring the password when logging in.

---

# 23. Complete installation

Let Ubuntu finish.

Then choose:

```text
Restart Now
```

You may see a message telling you to remove the installation media.

Hyper-V can effectively remove it for you.

If necessary:

```text
VM
→ Settings
→ SCSI Controller
→ DVD Drive
```

change the media to:

```text
None
```

Then reboot.

---

# 24. Update Ubuntu immediately

After logging into Ubuntu, open Terminal:

```text
Ctrl + Alt + T
```

Run:

```bash
sudo apt update
```

Then:

```bash
sudo apt full-upgrade -y
```

Then:

```bash
sudo apt autoremove --purge -y
```

and:

```bash
sudo apt clean
```

Restart:

```bash
sudo reboot
```

---

# 25. Verify Ubuntu version

After reboot:

```bash
lsb_release -a
```

You should see something identifying Ubuntu 26.04.

Also:

```bash
cat /etc/os-release
```

And check the kernel:

```bash
uname -a
```

Ubuntu 26.04 ships with the Linux 7.0 series kernel according to Canonical. ([Ubuntu][1])

---

# 26. Verify the VM sees 8 CPUs

Run:

```bash
nproc
```

You want:

```text
8
```

More detail:

```bash
lscpu
```

You should see something along the lines of:

```text
CPU(s): 8
Hypervisor vendor: Microsoft
Virtualization type: full
```

---

# 27. Verify the RAM

Run:

```bash
free -h
```

You should see approximately:

```text
Mem:    15Gi
```

It won't necessarily show exactly `16 GiB` because some memory is reserved by the OS/kernel.

---

# 28. Verify disk capacity

Run:

```bash
lsblk
```

Then:

```bash
df -h /
```

You should have roughly the expected capacity minus partitioning/filesystem overhead.

---

# 29. Verify Hyper-V integration

Modern Ubuntu kernels already contain Microsoft's Linux Hyper-V integration drivers. Microsoft specifically notes that supported Linux distributions contain built-in Linux Integration Services, so you generally should **not download an old external LIS package**. ([Microsoft Learn][4])

Check:

```bash
lsmod | grep hv
```

You'll probably see modules such as:

```text
hv_vmbus
hv_storvsc
hv_netvsc
hv_utils
```

Also:

```bash
dmesg | grep -i hyper-v
```

or:

```bash
dmesg | grep -i hyperv
```

That confirms Ubuntu recognizes the Hyper-V environment.

---

# 30. Test network connectivity

Run:

```bash
ip addr
```

Then:

```bash
ip route
```

Then:

```bash
ping -c 4 8.8.8.8
```

and:

```bash
ping -c 4 google.com
```

Finally:

```bash
curl https://ifconfig.me
```

If these work, you have:

```text
Ubuntu
   ↓
Hyper-V virtual NIC
   ↓
Default Switch
   ↓
Windows NAT
   ↓
Internet
```

---

# 31. Install basic development utilities

For your environment, I would immediately install:

```bash
sudo apt install -y \
    build-essential \
    git \
    curl \
    wget \
    unzip \
    zip \
    jq \
    tree \
    htop \
    net-tools \
    openssh-server \
    ca-certificates \
    gnupg \
    software-properties-common
```

Then:

```bash
sudo systemctl enable --now ssh
```

Check:

```bash
systemctl status ssh
```

Get the Ubuntu IP:

```bash
hostname -I
```

From Windows PowerShell you can then test:

```powershell
ssh jim@<ubuntu-ip>
```

---

# 32. Don't expect Windows-style Hyper-V Enhanced Session Mode

There's an important distinction here.

Canonical notes that manually installing Ubuntu from an ISO does **not automatically provide** the clipboard-sharing, dynamic-resolution and shared-folder enhancements associated with some Ubuntu Hyper-V Quick Create images. ([Ubuntu][2])

Current Microsoft documentation also describes standard Enhanced Session Mode device sharing primarily in terms of Windows guests. ([Microsoft Learn][5])

Therefore, for this Ubuntu installation I would use:

```text
Hyper-V VMConnect
    ↓
Ubuntu graphical console
```

for initial setup, and then increasingly use:

```text
SSH
VS Code Remote SSH
RDP if desired
web interfaces
```

for development.

That tends to be much more productive than trying to make VMConnect behave like VMware Workstation's desktop integration.

---

# 33. Install VS Code integration from Windows

For development, one of the best configurations is actually:

```text
Windows 11
│
├── VS Code
│      │
│      └── Remote SSH
│             │
│             ▼
│         Ubuntu VM
│
└── Hyper-V
       │
       ▼
 Ubuntu 26.04
```

You keep VS Code's Windows GUI while the actual code executes inside Linux.

Inside Ubuntu:

```bash
sudo systemctl enable --now ssh
```

Find the address:

```bash
hostname -I
```

Then connect from VS Code using the **Remote - SSH** extension.

This eliminates most concerns about VM graphics performance.

---

# 34. Create your clean-baseline checkpoint

Once Ubuntu is:

```text
installed
updated
networking working
SSH working
basic utilities installed
```

shut it down:

```bash
sudo poweroff
```

In Hyper-V Manager:

```text
right-click Ubuntu-26.04-DE
→ Checkpoint
```

Rename the checkpoint:

```text
01 - Clean Ubuntu 26.04.1
```

This becomes an extremely useful recovery point.

---

# 35. Recommended VM architecture for your machine

I'd ultimately structure your workstation like this:

```text
WINDOWS 11 PRO HOST
64 GB RAM
Physical CPU
NVMe SSD
│
├── Windows
│   ├── Browser
│   ├── VS Code
│   ├── Git
│   ├── PowerShell
│   └── Hyper-V Manager
│
└── Hyper-V
      │
      └── Ubuntu-26.04-DE
            │
            ├── 8 vCPU
            ├── 16 GB RAM
            ├── 160 GB VHDX
            │
            ├── Git
            ├── Python 3
            ├── Rust
            ├── Docker
            │
            ├── PostgreSQL container
            ├── dbt
            ├── Airflow
            ├── PySpark
            ├── Jenkins
            │
            ├── Kubernetes
            │   ├── kind
            │   └── containers
            │
            └── Terraform
```

That's a very good division of responsibility for the data-platform lab you've been constructing.

## I would specifically use these Hyper-V settings

```text
VM Name:
Ubuntu-26.04-DE

Generation:
2

Processors:
8

Memory:
16384 MB

Dynamic Memory:
Disabled

VHDX:
160 GB
Dynamically Expanding

Secure Boot:
Enabled

Secure Boot Template:
Microsoft UEFI Certificate Authority

Network:
Default Switch

Checkpoints:
Production

ISO:
ubuntu-26.04.1-desktop-amd64.iso
```

### One change I'd consider later

Once you start running **Docker + PostgreSQL + Airflow + Spark + Kubernetes simultaneously**, 16 GB becomes much tighter than the CPU allocation.

Since you have **64 GB in the Windows machine**, I'd eventually consider:

```text
Normal development:
8 CPU
16 GB RAM

Heavy data engineering:
8–12 CPU
24 GB RAM

Heavy Spark/Kubernetes lab:
12 CPU
32 GB RAM
```

I would **not** start with 32 GB. Your proposed **8 CPU / 16 GB / 160 GB** VM is the right first configuration. It gives you enough capacity to build a serious Linux data-engineering environment while leaving Windows plenty of resources.

[1]: https://ubuntu.com/download/desktop?utm_source=chatgpt.com "Download Ubuntu Desktop | Ubuntu"
[2]: https://ubuntu.com/server/docs/how-to/virtualisation/ubuntu-on-hyper-v/?utm_source=chatgpt.com "How to set up Ubuntu on Hyper-V - Ubuntu Server documentation"
[3]: https://us.releases.ubuntu.com/26.04/?utm_source=chatgpt.com "Ubuntu 26.04.1 LTS (Resolute Raccoon)"
[4]: https://learn.microsoft.com/en-us/windows-server/virtualization/hyper-v/supported-ubuntu-virtual-machines-on-hyper-v?utm_source=chatgpt.com "Supported Ubuntu virtual machines on Hyper-V | Microsoft Learn"
[5]: https://learn.microsoft.com/en-ie/windows-server/virtualization/hyper-v/enhanced-session-mode?utm_source=chatgpt.com "Share devices with Hyper-V Windows virtual machines | Microsoft Learn"
