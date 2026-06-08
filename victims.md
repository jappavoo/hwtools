# Lab Automation: Adding a New Victim

This guide outlines the hardware integration and software registration steps required to expand the laboratory cluster with an additional target host machine (e.g., `vic3`).

---

## 1. Hardware Integration Steps

To add a new system, you must duplicate the isolated interface circuit onto two available GPIO header pins on the Raspberry Pi 4. 

### Step 1: Identify Available GPIO Pins
Refer to your script mapping variables and choose two unassigned GPIO lines on the Pi 4 header. For this example, we will provision:
*   **Output Control Line:** `GPIO 20` (Physical Header Pin 38)
*   **Input Sensor Line:** `GPIO 21` (Physical Header Pin 40)

### Step 2: Wire the Power Switch Relay (Module A)
1. Mount a new 3.3V optocoupler relay module inside your controller enclosure.
2. Connect Pi **Pin 4 (5V)** to the relay `VCC` pin.
3. Connect Pi **Pin 6 (GND)** to the relay `GND` pin.
4. Connect Pi **Pin 38 (GPIO 20)** to the relay `IN` pin.
5. Route a twisted pair of wires from the relay **`NO`** and **`COM`** screw terminals inside the new host chassis.
6. Connect these wires directly in parallel across the motherboard's **`PWR_SW`** front panel header pins.

### Step 3: Wire the Power Status Sensor Relay (Module B)
1. Mount a second 3.3V optocoupler relay module.
2. Cut an old USB cable. Strip the internal insulation to expose the **Red (+5V)** and **Black (GND)** wires. Insulate and ignore the Green and White data wires.
3. Connect the **Red wire** to Module B's **`VCC`** pin AND **`IN`** pin simultaneously using a jumper wire bridge.
4. Connect the **Black wire** to Module B's **`GND`** pin.
5. Plug this USB connector into any available external USB port on the back of your new host machine.
6. Connect a wire from Module B's **`NO`** screw terminal to Pi **Pin 40 (GPIO 21)**.
7. Connect a wire from Module B's **`COM`** screw terminal to Pi **Pin 14 (GND)**.

---

## 2. Server OS Directory Provisioning

Log into the Raspberry Pi 4 terminal and create the dedicated TFTP directory bucket where `dnsmasq` will look for the new system's network boot components.

```bash
# Create the root TFTP boot folder for the new host
sudo mkdir -p /srv/tftp/vic3

# Each host gets a production slot that stores immutable boot images
sudo mkdir -p /srv/tftp/vic3/production

# Ensure group ownership and permissions inherit correctly
sudo chown root:dialout /srv/tftp/vic3
sudo chown root:dialout /srv/tftp/vic3/production
sudo chmod 775 /srv/tftp/vic3
sudo chmod 775 /srv/tftp/vic3/production
```

Place production boot images in `/srv/tftp/vic3/production/`. If there is only one kernel/initrd pair, `hw activate production vic3` will choose it automatically; if there are multiple, rerun with explicit filenames. A common naming pattern is `vmlinuz.prod`, `initrd.img.prod`, `vmlinuz.prodA`, `initrd.img.prodA`, etc.

For PXE command lines, use this tree:
```bash
sudo mkdir -p /srv/tftp/pxelinux.cfg/production/vic3
sudo mkdir -p /srv/tftp/pxelinux.cfg/users/jappavoo/vic3
sudo ln -sf /srv/tftp/pxelinux.cfg/users/jappavoo/vic3/01-xx-xx-xx-xx-xx-xx /srv/tftp/pxelinux.cfg/01-xx-xx-xx-xx-xx-xx
```
Production cmdline files live under `/srv/tftp/pxelinux.cfg/production/<host>/`, staged edits live under `/srv/tftp/pxelinux.cfg/users/<user>/<host>/`, and the active MAC file at `/srv/tftp/pxelinux.cfg/<mac>` is a symlink to one of them.
Use `hw cmdline seed <host>` to copy production into your staging area, edit the file locally, then use `hw cmdline upload <host> <file>` to push the edited copy back without hand-writing TFTP paths.

---

## 3. Persistent Serial Interface Naming (udev Rules)

To stop Linux from randomly swapping your serial adapters between `/dev/ttyUSB0` and `/dev/ttyUSB1` when the Pi restarts, you must assign a persistent alias matching your script (e.g., `ttyVIC3`).

### Step 1: Identify Stable Device Properties
Plug your new USB-to-Serial adapter into the Pi and collect stable attributes:
```bash
udevadm info -q property -n /dev/ttyUSB0 | grep -E "ID_SERIAL_SHORT|^ID_PATH="
```
Use `ID_SERIAL_SHORT` when available; otherwise use `ID_PATH` for deterministic mapping.

### Step 2: Write a Custom udev Rule
Open your custom rules file:
```bash
sudo nano /etc/udev/rules.d/99-serial.rules
```
Use this pattern (includes comments for future device onboarding):
```text
# Apply to all USB serial adapters so ownership is consistent.
SUBSYSTEM=="tty", KERNEL=="ttyUSB[0-9]*", GROUP="dialout", MODE="0660"

# Existing lab mappings:
SUBSYSTEM=="tty", KERNEL=="ttyUSB*", ENV{ID_PATH}=="platform-fd500000.pcie-pci-0000:01:00.0-usb-0:1.4:1.0", SYMLINK+="ttyVIC1"
SUBSYSTEM=="tty", KERNEL=="ttyUSB*", ENV{ID_SERIAL_SHORT}=="FTE6J1CF", SYMLINK+="ttyVIC2"

# New device template:
# 1) Find identifier: udevadm info -q property -n /dev/ttyUSBX | grep -E "ID_SERIAL_SHORT|^ID_PATH="
# 2) Prefer ID_SERIAL_SHORT; use ID_PATH if serial is missing.
# 3) Add one line per new victim alias (ttyVIC3, ttyVIC4, ...):
# SUBSYSTEM=="tty", KERNEL=="ttyUSB*", ENV{ID_SERIAL_SHORT}=="<SERIAL>", SYMLINK+="ttyVIC3"
# SUBSYSTEM=="tty", KERNEL=="ttyUSB*", ENV{ID_PATH}=="<PATH>", SYMLINK+="ttyVIC3"
```
Save and exit (`Ctrl+O`, `Enter`, `Ctrl+X`).

### Step 3: Reload the Device Subsystem
```bash
sudo udevadm control --reload-rules && sudo udevadm trigger
```
Verify that the persistent symlink has generated successfully:
```bash
ls -l /dev/ttyUSB* /dev/ttyVIC*
stat -c '%n %U:%G %a' /dev/ttyUSB0 /dev/ttyUSB1
```
The `/dev/ttyUSB*` devices should now be owned by group `dialout` with mode `0660`.

---

## 4. Registering the Host in the Script

Open your master program configuration using your text editor:
```bash
sudo nano /usr/local/bin/hw
```

Locate the **`SYSTEM_MAP` associative array block** (found near line 14 within Chunk 1) and append a new entry line definition matching your newly integrated physical layout pins and directory links:

```text
SYSTEM_MAP["vic3"]="20:21:ttyVIC3:\$BASE_TFTP_DIR/vic3"
```

### Syntax Layout Parameter Breakdown:
*   `vic3`: The unique human-readable hostname string users pass to the script.
*   `20`: The output line number (`GPIO 20`) managing Module A's power switch relay.
*   `21`: The input line number (`GPIO 21`) monitoring Module B's USB power status sensor relay.
*   `ttyVIC3`: The precise file name string matching your newly instantiated path inside `/dev/`.
*   `$BASE_TFTP_DIR/vic3`: The direct system path string targeted during TFTP image activation rewires.

Save the changes and exit. The new machine is now completely integrated and will instantly display inside your hardware tracking status grid the next time you type `hw`.
