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

# Ensure group ownership and permissions inherit correctly
sudo chown root:dialout /srv/tftp/vic3
sudo chmod 775 /srv/tftp/vic3
```

---

## 3. Persistent Serial Interface Naming (udev Rules)

To stop Linux from randomly swapping your serial adapters between `/dev/ttyUSB0` and `/dev/ttyUSB1` when the Pi restarts, you must assign a persistent alias matching your script (e.g., `ttyVIC3`).

### Step 1: Identify the Serial Adapter Serial Number
Plug your new USB-to-Serial adapter into the Pi and run `lsusb` or parse `dmesg` logs to extract its unique vendor attributes:
```bash
udevadm info --name=/dev/ttyUSB0 --attribute-walk | grep -E "serial|idVendor|idProduct"
```
*Look for output lines resembling:*
*   `ATTRS{idVendor}=="0403"`
*   `ATTRS{idProduct}=="6001"`
*   `ATTRS{serial}=="FT92A3B4"`

### Step 2: Write a Custom udev Rule
Open your custom rules file:
```bash
sudo nano /etc/udev/rules.d/99-usb-serial.rules
```
Append a new hardware assignment line matching the extracted device parameters exactly:
```text
SUBSYSTEM=="tty", ATTRS{idVendor}=="0403", ATTRS{idProduct}=="6001", ATTRS{serial}=="FT92A3B4", SYMLINK+="ttyVIC3"
```
Save and exit (`Ctrl+O`, `Enter`, `Ctrl+X`).

### Step 3: Reload the Device Subsystem
```bash
sudo udevadm control --reload-rules && sudo udevadm trigger
```
Verify that the persistent symlink has generated successfully:
```bash
ls -l /dev/ttyVIC3
```
*Expected Output:* `lrwxrwxrwx 1 root root 7 Jun  6 09:00 /dev/ttyVIC3 -> ttyUSB0`

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
