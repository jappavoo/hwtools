# HW tools (`hw`)

This document outlines the step-by-step installation, hardware wiring configuration, and multi-user administration workflow for the unified laboratory power, console, and TFTP boot manager tool (`hw`).



---

## 1. Physical Power Hardware Setup

The automation framework uses a **Raspberry Pi 4 Model B** as the head (`JA-HW_HEAD`) node . It interfaces to the power supply with each target victim node (`JA-HW-VIC1`, `JA-HW-VIC2`, etc.) through two independent
pathways: **Power Control Switch** and **Power Status Detection**.

### Isolation Safety Mandate
**Do not connect a Raspberry Pi GPIO header pin directly to a target computer's motherboard switch pins or power rails.** 
Voltage differences will permanently destroy the controller chip on the Pi or fry the host motherboard. You must use isolated relay modules equipped with optocouplers for all physical signal bridges.

### Wiring Specification Matrix

#### Module A: Power Control (Output Relay)
*This replaces or splits the physical power button on the target computer case.*

*   Connect Pi Header Pin 4 (5V VCC Power) to Module A `VCC`
*   Connect Pi Header Pin 6 (GND System Ground) to Module A `GND`
*   Connect Pi Header Pin 12 (GPIO 18 / Command Line Out) to Module A `IN`
*   **Relay Output Side:** Connect the `NO` (Normally Open) and `COM` (Common) screw terminals directly in parallel across the target PC motherboard's front panel `PWR_SW` headers. 

#### Module B: Power Detection (Input Relay)
*This senses whether the target computer is on by checking for voltage on its USB bus.*

*   Connect Target Host PC Cut USB Cable Red Wire (+5V VBUS Rail) to Module B `VCC` AND `IN` (Bridge these together)
*   Connect Target Host PC Cut USB Cable Black Wire (GND Ground) to Module B `GND`
*   Connect Module B Output `NO` (Normally Open Screw Terminal) to Pi Header Pin 18 (GPIO 24 / Sensor Input Line)
*   Connect Module B Output `COM` (Common Screw Terminal) to Pi Header Pin 14 (GND System Ground)
*   *Note: Ensure "Deep Sleep" / "ErP Ready" energy savings states are Disabled inside the Target Host PC BIOS configuration so the USB 5V rail cuts out completely when the machine powers down.*

see wiring.txt for more info

---

## 2. Physical Serial Console Setup

The head node use USB serial adaptors to connect to the on board 9 pin RS232 serial ports of the victims.

`udev` rules are used to ensure that the serial adaptors for each victim will always be mapped to /dev/ttyVIC1, /dev/ttyVIC2, etc.

`picocom` is use to access and lock the node for use by a user.



## 3. Physical Networking Setup

The head node's internal ethernet port is connected to the campus network via a lab wall port.  Additionally, it is connect
to a private 1 gigbit switch via a usb ethernet port.  The victims are connected to the private switch to form a private `192.168.1/24`
network.  IP management is static with the following structure

1. head: 192.168.1.1
2. vic1: 192.168.1.100
3. vic2: 192.168.1.200
etc.

The head is configured to run a network boot service and NAT on the private network for the victims.


## 4. Server OS Directory & Group Provisioning

The following is the setup on the head node to initialize the filesystem structure and group privileges:

`hw` uses `dialout` as the single access group registry.
Users provisioned by `hw user add` are added to `dialout` and `gpio`.

```bash
# Create shared terminal logging environment
sudo mkdir -p /var/log/serial

# Create system TFTP target locations
sudo mkdir -p /srv/tftp/vic1 /srv/tftp/vic2 /srv/tftp/users

# Make the dialout group own the laboratory directory trees
sudo chown -R root:dialout /srv/tftp /var/log/serial
sudo chmod -R 775 /srv/tftp /var/log/serial

# Apply the SGID bit so new files inherit the group ownership automatically

find /srv/tftp -type d -exec sudo chmod g+s {} +
find /var/log/serial -type d -exec sudo chmod g+s {} +
```

---

## 5. Script Installation

The script `hw` features a dual-mode behaviour. It can be installed directly onto the head node as well as onto individual developers' local dev environments.

### On the head:

Checkout the hwtools repo and link the script to a /usr/local/bin

```bash
sudo ln -s hwtools/hw /usr/local/bin/hw
```

### On a Local Workstation:

1. Checkout the hwtools repo and ensure can be found in your path 
   ```bash
   mkdir -p ~/bin
   ln -s hwtools/hw ~/bin/hw
   ```
   
2. Configure your local SSH client configuration file (`~/.ssh/config`) on your workstation to automatically map your personal identity key and user ID whenever you talk to the head node and victims.
   The following are example stanza's from my ssh config.  You will need to adjust as needed
   ```text
   Host JA-HW-HEAD ja-hw-head head
     TCPKeepAlive yes
     ServerAliveInterval 30
     Hostname <external IP of head>
     IdentityFile ~/.ssh/JA-HW-HEAD
     ProxyJump bu

   Host JA-HW-VIC1 ja-hw-vic1 vic1
     TCPKeepAlive yes
     ServerAliveInterval 30
     Hostname vic1
     ProxyJump ja-hw-head

   Host JA-HW-VIC2 ja-hw-vic1 vic2
     TCPKeepAlive yes
     ServerAliveInterval 30
     Hostname vic2
     ProxyJump ja-hw-head
	 
   Host bu bugit bu2 csa2 csa2.bu.edu
     HostName csa2.bu.edu
     User jappavoo
   ```

You will need to ask an existing lab member add your user name to  JA-HW-HEAD and give you key and the ip address
of the head node.  See Administrative command next for details.  Also if you don't have an user on a BU server
that has public access you can either use the vpn (in which case you won't need the ProxJump config) or talk
to a lab member to help get you an account on a public BU server.

---

## 6. Administrative User Management Commands
The script includes built-in commands to manage user access profiles. If these commands are run by an unprivileged user, the script will automatically check permissions and prompt for sudo elevation.

### List Active Lab Users

To see all local users who currently have permission to access the hardware systems:
```bash
hw user
```

### Provision a New Lab User (Passwordless Security Enforced)
To add a new researcher, create their Linux profile, block password entries, and configure an isolated TFTP directory:
```bash
hw user add jappavoo
```
*Note: This command generates a secure Ed25519 SSH keypair on the fly, locks the password matrix, stores the keypair in the new user's `~/.ssh/` as `id_ed25519` and `id_ed25519.pub`, and outputs the raw private key to the screen for secure delivery to the user workstation.*

### De-provision a Lab User
To cleanly remove a user, delete their account, and clear out their staged development files to reclaim disk space:
```bash
hw user del jappavoo
```
---

## 7. Standard Laboratory Workspace Workflow
Researchers use these identical execution commands natively on the Raspberry Pi or directly inside their own local development workstations. Workstation commands transparently match your active login profile (`$USER`), converting file uploads into network transfers (`scp`) and running remote connections over secure SSH shells automatically.

### Step 1: Check Lab Availability Status
Run `hw` with no arguments to get an overview of the lab, see which machines are turned on, and find out who holds active console locks:
```bash
hw
```

### Step 2: Upload Development Files
Compile your custom kernel or ramdisk on your workstation, and immediately push the file into your personal TFTP sandbox space on the Pi:
```bash
hw upload ~/src/linux/arch/arm/boot/zImage
```
*Note: If the filename contains `vmlinuz` or `zImage`, the script automatically registers it as your active boot asset on upload.*

### Step 2b: List Staging or Production Assets
```bash
# Default: list your staged files
hw ls

# Explicit staging view
hw ls stage

# Production files for all hosts
hw ls prod

# Production files for a single host
hw ls prod vic1
```
Listings now show file date, size, and md5sum.
`hw ls` defaults to `stage`.

### Step 3: Switch Active Images (Optional)
If you have multiple builds staged in your directory and want to change your active image choice:
```bash
hw activate zImage-stable-backup

# Select the per-host boot source
hw activate production vic1
hw activate staging vic1

# Select different staged kernel/initrd files per host
hw activate staging vic1 zImage-vic1 initrd-vic1
hw activate staging vic2 zImage-vic2 initrd-vic2

# If multiple production images exist, rerun with explicit filenames
hw activate production vic1 vmlinuz.prodA initrd.img.prodA
```

### Step 4: Acquire System Lock and Open Console
To open a serial tunnel to a host machine, use the direct hostname shortcut:
```bash
hw vic1
```
*The Safety Interlock:* This shortcut launches `picocom -b 115200 -f h` on `/dev/ttyVIC1`. `picocom` takes an advisory `flock` on the tty itself, and `hw` checks that tty lock before allowing power actions. You now hold the exclusive hardware lock for `vic1`. No other user can send power actions to this machine while your console window remains open.

### Step 5: Power Cycling and Controlling the Target
Because the script tracks real-time voltage on the host's USB rails, it acts intelligently based on whether the machine is currently running or turned off. 

While keeping your console session running in one window, open a second terminal panel on your machine to manage the system state.

#### The Fast Workflow: Soft Rebooting (System is Responsive)
If your target system is alive, responding to your console inputs, or accessible over the network, **do not use hardware power commands to cycle the machine**. Issuing an OS-level reboot is significantly faster and prevents filesystem corruption.
1. Update your staged boot assets using `hw upload` or `hw activate`, or select `hw activate production vic1` / `hw activate staging vic1` to choose the host boot source. If a host has multiple production images, rerun `hw activate production <host> <kernel> <initrd>` with explicit filenames.
2. Inside your active host terminal or over an SSH session, issue a standard software reboot:
   ```bash
   sudo reboot
   ```
3. The host will safely unmount filesystems, restart, and immediately query the Pi's TFTP directories to pull your newly activated kernel files.

#### The Fallback Workflow: Hardware Control Commands (System is Frozen/Off)
If the host system has suffered a kernel panic, is completely unresponsive, or is powered off, use the script to force a hardware-level state change. The tool supports three physical actions:

#### Action A: Power On (TFTP Boot Launch)
To initiate a fresh cold boot from a fully powered-down state:
```bash
hw vic1 on
```
1. The script verifies that you hold the active `picocom` console session for `vic1`.
2. It dynamically switches the master symlinks inside `/srv/tftp/vic1/` to point to your personal active staging folder.
3. It pulses the virtual hardware power button for 1 second to wake the motherboard and launch the network boot.

#### Action B: Graceful ACPI Power Off
If you need to turn a running system off but your network stack is down or you cannot log in to issue a command:
```bash
hw vic1 off
```
*What it does:* It pulses the motherboard's power button for 1 second. The host OS interprets this physical ACPI event and initiates a clean shutdown sequence. The script prints `SUCCESS` once the USB rails drop to zero.

#### Action C: Emergency Hardware Hard Cut (Force Off)
If your custom kernel crashes during initialization, experiences a total freeze, or stops responding to ACPI power button taps:
```bash
hw vic1 force-off
```
*What it does:* It overrides all software layers by holding the virtual power button down for 5 continuous seconds at the hardware level. This forces the motherboard to immediately cut power to the rails.

### Step 6: Query Telemetry Log Files
The script auto-rotates and logs all serial communications to `/var/log/serial/`. Users can safely browse or audit session histories across the lab group:
```bash
# List all generated logs for a specific host
hw log vic1

# Same as above, explicitly selecting the list subcommand
hw log ls vic1

# Read the contents of the absolute newest session trace
hw log cat vic1

# Read a specific log file directly by basename
hw log vic1con.2026-06-08_08-00-00.log

# Delete one exact log file
hw log clean vic1con.2026-06-08_08-00-00.log

# Delete an explicit list of exact log files after confirmation
hw log clean vic1con.2026-06-08_08-00-00.log vic2con.2026-06-08_08-00-00.log

# Preview and confirm a wildcard or regex match set before deleting
hw log clean vic1con.2026-06-08_22*
hw log clean re:vic1con.*\\.log
```
