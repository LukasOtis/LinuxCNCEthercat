## LinuxCNC + EtherCAT (Beckhoff I/O + Stepperonline A6 Servo) setup log 

The setup is split into 3 parts:
1. Installing LinuxCNC and Ethercat support for it
2. I then decided to first get a minimal communication running for the I/O and drives. This is not complety neccecary, but makes sure things behave as expected
3. Getting a bench-setup LinuxCNC running, with all I/O and motor functions accesible


what you will need:
- A PC with two ethernet ports, or one ethernet port and wifi-caipability
- Your Ethercat components. for me this is a 
    - Beckhoff coupler and ditial I/O cards for limits, probes as well as analog I/O to controll the main spindle.
    EK1100 EtherCAT-Coupler + EL1008 8K. Dig. In + EL2008 8K. Dig. Out + EL2624 4K. Relais Out + EL3102 2K. Ana. In +/-10V, DIFF + EL4022 2Ch. Ana. Out 4-20mA, 12bit
    - For the axis drives a Stepperonline A6 Series servo drive(s)
    - the correponding networking and power cables
- A usb stick for installing the Linux CNC image

For the following first steps, you only need your PC and a USB stick.
---
STEP
1 

# 1.1) Install LinuxCNC + EtherCAT support

For this section, see the forum post for additional details: https://forum.linuxcnc.org/ethercat/45336-ethercat-installation-from-repositories-how-to-step-by-step

## 1.1.1) Install LinuxCNC
1. Download the current LinuxCNC ISO from the downloads page and flash it onto a usb drive (over 4GB) using balena etcher or simmilar
2. Install the OS (normal install flow).
3. After first boot, open the `README.md` in your home directory (this is the “post-install checklist” shipped with the image) and run the recommended package updates:
   - `apt update` / `apt upgrade`
   - GUI / Qt packages as suggested (often needed for common LinuxCNC GUIs)

## 1.1.2) Install EtherCAT integration packages
in the `README.md` it also states to install the EtherCAT integration packages: (the Forum post has an updated command, as these are listed as dependencies in the current 2.9.2 LinuxCNC version)
```bash
sudo apt install linuxcnc-ethercat
```
Notes: This took some time on my machine

---

# 1.2) Configure the EtherCAT master 

To configure the EtherCAT master on the PC to talk to the different connected devises, we only need to set the correct output port. 
So the physical connection you want to use. Everything else will be handeled by the master itself.

For this:
## 1.2.1 Pick the NIC (network card) by MAC address (i chose the onboard NIC for ethercat, and have a PCI adapter for network connection)
Identify NICs and their MAC addresses:
```bash
ip a
```
or (if you just want to MAC adress)
```bash
ip -br link
```
For ease of use, it is recommencded to use the "first" NIC, as another configuration could lead to inconsisten behaviour.
Note down that MAC adress

## 1.2.2 Configure `/etc/ethercat.conf`
Edit:
```bash
sudo geany /etc/ethercat.conf
```
Set the MAC adress to the MASTER0_DEVICE as well as the DEVOCE_MODULES to generic:

```conf
MASTER0_DEVICE="xx:aa:yy:zz:bb:cc"
DEVICE_MODULES="generic"
```

This is all the installation and configuration needed to start the ethercat service. 

--->>>>   At this tage, it is helpfull to connect **atleast one ethercat device** to the NIC we just set the MAC adress to.
For me, this is only the Beckhoff Coupler and Components. 
I setup a "test bench" setup for now and have not integrate the setup into the machine hardware at this stage.

# 1.3) Start EtherCAT master service

Start the ethercat services
```bash
sudo systemctl enable ethercat.service
sudo systemctl start ethercat.service
sudo systemctl status ethercat.service
sudo chmod 666 /dev/EtherCAT0
```
If no errors come up, the service is healthy.


# 1.4) Basic validation

you should be able to query slaves:
```bash
ethercat slaves
```
and expext something like:

```text
0 0:0 PREOP + EK1100 EtherCAT-Koppler (2A E-Bus) 
1 0:1 PREOP + EL1008 8K. Dig. Eingang 24V, 3ms 
2 0:2 PREOP + EL2008 8K. Dig. Ausgang 24V, 0.5A 
3 0:3 PREOP + EL2624 4K. Relais Ausgang, Schlie�er (125V AC / 30V DC) 
4 0:4 PREOP + EL3102 2K. Ana. Eingang +/-10V, DIFF 
5 0:5 PREOP + EL4022 2Ch. Ana. Output 4-20mA, 12bit
```

Interpretation:
- Seeing a list of terminals/slaves means **the master can talk to the bus** and identify devices.
- At this stage, slaves show up in **PREOP**. That is normal. Slaves are discovered and basic communication works, but the process data (PDO mapping) is not active.
 
Optional detail query:
```bash
ethercat slaves -v
```
This is useful to sanity-check vendor/product IDs (e.g., verifying genuine Beckhoff vendor ID).


# 1.5) Device permissions for `/dev/EtherCAT0` (to avoid manual chmod after reboot)

## Add a udev rule
Create/edit:
```bash
sudo geany /etc/udev/rules.d/99-ethercat.rules
```

Add:
```bash
KERNEL=="EtherCAT[0-9]", MODE="0777"
```

Reload rules and reboot:
```bash
sudo udevadm control --reload-rules
sudo reboot
```

This concludes the initial setup of the Ethercat service. 
Now its time to start to configure the setup to the system you want to use

---

STEP
2 

I started with a minimal version for first the Beckhoff system and then the Servo drive

# 2.1) Minimal Beckhoff-only configuration (no drives yet)

Goal: bring Beckhoff terminals (from the PREOP state) to **OP**, and get I/O pins in to show up in HAL for LinuxCNC to acsess.

Right now, the master sees the connected devises and can identify them, but does not know what kind of data they want to recieve and send. 
For this, we need a to set up the structure for the communication, which is done using a ".xml" file. This sets the order for the PDO (process data objects) within the cyclic ethercat communication.

This now is dependend on your system and the acutal components your master can see.
So, with 
```bash
ethercat slaves
```
you get the list of the slaves in your system (as posted above). 
All of the devises are connected to master 0, with IDs fro 0 to 5 starting with the ethercat coupler and then the IO cards.

## 2.1.1) Create a minimal `ethercat-conf.xml`
```bash
mkdir -p ~/ethercat-tests
nano ~/ethercat-tests/ethercat-conf.xml
```
After setting up a folder create a first .XML **based on the list** of ethercat devises or ask your favorite AI tool to do so.
In my example this looks like this:

```xml
<masters>
  <master idx="0" appTimePeriod="1000000" refClockSyncCycles="1000">

    <slave idx="0" type="EK1100" name="ek1100"/>

    <slave idx="1" type="EL1008" name="di8"/>
    <slave idx="2" type="EL2008" name="do8"/>
    <slave idx="3" type="EL2624" name="relay4"/>
    <slave idx="4" type="EL3102" name="ai2"/>
    <slave idx="5" type="EL4022" name="ao2"/>

  </master>
</masters>
```

This example looks quite simple, as alot of deviceses come supported in LinuxCNC-Ethercat out of the box. You can see a full list here:
https://github.com/linuxcnc-ethercat/linuxcnc-ethercat/blob/master/documentation/DEVICES.md

Notes:
- `idx` must match the **physical order** on the EtherCAT chain (EK1100 first, then terminals in order). Especially important that the ID and that it matches the id in your system. 
- `name` is a individual prefix used for HAL pin naming (helpful for readability). KEEP THAT CONSISTENT
- `appTimePeriod="1000000"` corresponds to a 1 ms cycle. Good starting point

## 2.1.2)  Parse test; does `lcec_conf` (LinuxCNC EtherCAT _conf) accept the XML?

```bash
cd ~/ethercat-tests
```
now go into halrun
```bash
halrun
```
now run

```hal
loadusr -W lcec_conf ethercat-conf.xml
show pin lcec
```
Expect:

omponent Pins: 
```text
Owner Type Dir Value Name 
4 u32 OUT 0x00000001 lcec.conf.master-count 
4 u32 OUT 0x00000006 lcec.conf.slave-count
```

Expected output indicators:
- `lcec.conf.master-count` should be `1`
- `lcec.conf.slave-count` should match your XML (here: `6`)

This confirms:
- the XML is syntactically valid (for `lcec_conf`)
- the EtherCAT master is accessible
- the master/slave counts match expectations

---
# 2.2) By Bring up I/O into HAL we can also see and test each input and output

## 2.2.1) Load realtime components

Still inside `halrun`:

(or see below)
```hal
loadrt threads name1=fast period1=1000000
loadrt lcec
addf lcec.read-all fast
addf lcec.write-all fast
start
```
**In linux CNC the thread is most often called 'servo-thread', but for me that was to long to type**

This does:
- Creates a 1 kHz realtime thread called `fast`.
- Loads `lcec` (the realtime EtherCAT component).
- Adds EtherCAT read/write functions to the thread so process data is exchanged cyclically.
- `start` starts the thread.

## 2.2.2) Check basic status pins

(still inside `halrun`, after starting the thread)

```hal
show pin lcec.read-all.time
show pin lcec.0.link-up
show pin lcec.0.slaves-responding
```
Expect:
- `lcec.read-all.time` should be non-zero (the function is running).
- `lcec.0.link-up` should be TRUE.
- `lcec.0.slaves-responding` should match your number of slaves.

## 2.3) **optional cross-check** from another terminal

In a separate terminal (but keep the other terminal open):

```bash
ethercat master
```
expect: (want to see that the master is active and (typically) in an operational phase)

Master0 
Phase: Operation 
Active: yes Slaves: 6 
Ethernet devices: Main: 90:1b:0e:93:2f:69 (attached) 
Link: UP 
...

Tx frames: 700434 Tx bytes: 42243879 Rx frames: 700433 Rx bytes: 42243819 Tx errors: 0 Tx frame rate [1/s]: 1000 1000 842 Tx rate [KByte/s]: 59.8 59.8 50.3 Rx frame rate [1/s]: 1000 1000 842 Rx rate [KByte/s]: 59.8 59.8 50.3 Common: Tx frames: 700434 Tx bytes: 42243879 Rx frames: 700433 Rx bytes: 42243819 Lost frames: 0 Tx frame rate [1/s]: 1000 1000 842 Tx rate [KByte/s]: 59.8 59.8 50.3 Rx frame rate [1/s]: 1000 1000 842 Rx rate [KByte/s]: 59.8 59.8 50.3 Loss rate [1/s]: 0 -0 0 Frame loss [%]: 0.0 -0.0 0.0 Distributed clocks: Reference clock: Slave 5 DC reference time: 824077604538952000 Application time: 824077741093189522 2026-02-10 22:29:01.093189522


now rechecking 
```bash
ethercat slaves
```
in the seperate window should also show them now in 'op'erating mode!

---
**If the halrun termial was closed, you can get it back by:**

```bash
cd ~/ethercat-test/
loadusr -W lcec_conf ethercat-conf.xml
halrun

loadrt threads name1=fast period1=1000000
loadrt lcec
addf lcec.read-all fast
addf lcec.write-all fast
start
```

# 2.4) Manual I/O testing in HAL

Now that the pins exist, you can test I/O just like Linux CNC will then acsess it later one. So testing the acutal hardware

## 2.4.1) Toggle a digital output (example EL2008)

still within `halrun`:

```hal
setp lcec.0.do8.dout-0 TRUE
setp lcec.0.do8.dout-0 FALSE
```

Notes:
- Channel numbering is zero-based here (`dout-0` is the first channel).
- Continue with `dout-1`, `dout-2`, etc.

## 2.4.2) Read a digital input (example EL1008)

```hal
show pin lcec.0.di8.din-0
```
You’ll typically see both normal and inverted pins!

---
now you can `exit` halrun and close the terminals

As this concludes the Beckhoff setup!


# 2.5) Add the servo drive(s)

This is now quite simmiar to the beckhoff IO. 
Adding the servo drive to the Network and powerering it up, it should show up as a recognized slave.

```bash
ethercat slaves
```
expect:

```text
...
5 0:5 PREOP EL4022 
6 0:6 PREOP AS715N (the servo)
```

In this first setup, i dont want to drive the servo, but make sure that the communication is setup corrently.
And again, we have to specify the structure of communication in for the linux-cnc ethercat in the .xml file. 

## 2.5.1) update linuc cnc ethercat conf 

```bash
cd ~/ethercat-tests
geany ~/ethercat-tests/ethercat-conf.xml
```

Also for servo drives there are a plenty of drives supported in LinuxCNC-Ethercat. 
As of now, my drives are not on the list, however there are also alot of examples in the forum. Otherwise it is possible to generate an lcec config from the live bus (but i havent tried that).

i used the .xml example from https://github.com/CollinBardini/linuxcnc-a6-servo/blob/main/ethercat-conf.xml

and in my setup the updated .xml is now with the driver looks like this:
```xml
<masters>
  <master idx="0" appTimePeriod="1000000" refClockSyncCycles="1000">
    <slave idx="0" type="EK1100" name="ek1100"/>

    <slave idx="1" type="EL1008" name="di8"/>
    <slave idx="2" type="EL2008" name="do8"/>
    <slave idx="3" type="EL2624" name="relay4"/>
    <slave idx="4" type="EL3102" name="ai2"/>
    <slave idx="5" type="EL4022" name="ao2"/>

    <!-- StepperOnline A6 / AS715N -->
    <slave idx="6" type="generic" vid="00400000" pid="00000715" configPdos="true" name="A61">
      <dcConf assignActivate="300" sync0Cycle="*1" sync0Shift="25000" />
      <watchdog divider="2498" intervals="1000" />
      <syncManager idx="2" dir="out">
        <pdo idx="1600">
          <pdoEntry idx="6040" subIdx="00" bitLen="16" halPin="control-word" halType="u32" />
          <pdoEntry idx="6060" subIdx="00" bitLen="8"  halPin="control-mode" halType="s32" />
          <pdoEntry idx="607A" subIdx="00" bitLen="32" halPin="target-position" halType="s32" />
          <pdoEntry idx="60FF" subIdx="00" bitLen="32" halPin="target-velocity" halType="s32"/>
          <pdoEntry idx="607C" subIdx="00" bitLen="32" halPin="home-offset" halType="s32"/>
          <pdoEntry idx="6098" subIdx="00" bitLen="8"  halPin="homing-method" halType="s32" />
          <pdoEntry idx="6099" subIdx="01" bitLen="32" halPin="homing-high-velocity" halType="u32"/>
          <pdoEntry idx="6099" subIdx="02" bitLen="32" halPin="homing-low-velocity"  halType="u32"/>
          <pdoEntry idx="609A" subIdx="00" bitLen="32" halPin="homing-acceleration"  halType="u32"/>
        </pdo>
      </syncManager>
      <syncManager idx="3" dir="in">
        <pdo idx="1A00">
          <pdoEntry idx="6041" subIdx="00" bitLen="16" halPin="status-word"     halType="u32" />
          <pdoEntry idx="6061" subIdx="00" bitLen="8"  halPin="mode-display"    halType="s32" />
          <pdoEntry idx="6064" subIdx="00" bitLen="32" halPin="actual-position" halType="s32" />
          <pdoEntry idx="606C" subIdx="00" bitLen="32" halPin="actual-velocity" halType="s32"/>
          <pdoEntry idx="6077" subIdx="00" bitLen="16" halPin="actual-torque"   halType="s32"/>
        </pdo>
      </syncManager>
    </slave>
  </master>
</masters>
```

At first glance, alot more complicated. But there is a structure in this madness:
Most servo drives communicate using a standadised format name 'cia402' which defines a standard dictionary of variables for all servo drives.
In this example you can see that for the 'out' and 'in' directory: each index in the PDO object from the drive has a specific index and meaning that we can then access as a halPin. 
This translates the dataobject from the drive into something we can acutally use.

To test: save and exit.

## 2.5.2) check he XML is syntactically valid

Firt we can make sure that the pins are showing up.

```bash
halrun
```

then

```text
loadusr -W lcec_conf ethercat-conf.xml
show pin lcec
```

Expected output indicators:
- `lcec.conf.master-count` should be `1`
- `lcec.conf.slave-count` should match your XML (here: `7`)

# 2.6) test drive communication

## 2.6.1) checking cia402 object mapping

instide 'halrun', (again) startup a linuxcnc ethercat threat
```hal
loadrt threads name1=fast period1=1000000
loadrt lcec
addf lcec.read-all fast
addf lcec.write-all fast
start
```

now get info on the drive pins as set in the '.xml': (with the last .XXX being your drive name)

```bash
show pin lcec.0.A61
```
expect:
slave-online = TRUE, slave-oper = TRUE, slave-state-op = TRUE; meaning the cyclic PDO exchange is running.

This also shows that the cia402 keys are mapped (**more on that later**): control-word (6040), status-word (6041), control-mode (6060), mode-display (6061), target-position (607A), actual-position (6064), target-velocity (60FF), actual-velocity (606C), actual-torque (6077)


## 2.6.2) live track rotor postition

now you should be able to do some sanity checks by live testing the position of the motor using
```bash
show pin lcec.0.A61.actual-position
```
rotate the output shaft of the drive and rerun the command to see changes


# 2.7) !

At this point you should have now sucsefully:
- EtherCAT master installed + bound to the correct NIC and startup permissions set
- Beckhoff and drive slaves discovered
- a minimal `ethercat-conf.xml` for Beckhoff IO and servo drive
- `lcec` running in a realtime thread
- I/O & drive functions visible as HAL pins and manually testable

This concludes the ground work and or what is required before acustally using the devices in our linux cnc machine

time for a little break! and 'exit' the halrun and close terminals
---

STEP
3 

This step is setting up a hardware abtraction layer (hal-file) and base machine file (ini) so that linucCNC can acess what we just set up

# 3.1) Get the cia402 component and test that for the specific drive 

## 3.1.1) Prereqs and folder setup

```bash
sudo apt update
sudo apt install -y git
mkdir -p ~/dev
```
## 3.1.2) Install the cia402.comp (component)

If you are using cia402 compatible drives, this makes enable/fault-reset/mode handling cleaner once you move into HAL/INI

```bash
cd ~/dev
git clone https://github.com/dbraun1981/hal-cia402
cd hal-cia402
sudo halcompile --install cia402.comp
```
## 3.1.3) Verify it’s loadable and print the names (optional)

```bash
cd ~/ethercat-test/
halrun
```
inside halrun (get the system back online)

```hal
loadusr -W lcec_conf ethercat-conf.xml
loadrt threads name1=fast period1=1000000

loadrt lcec
loadrt cia402 count=1

addf lcec.read-all fast
addf cia402.0.read-all fast
addf cia402.0.write-all fast
addf lcec.write-all fast

start
```
then link drive to cia302 component (check drive name; here A6):

```hal
# status/controlword
net a6-statusword     lcec.0.A61.status-word     => cia402.0.statusword
net a6-controlword    cia402.0.controlword      => lcec.0.A61.control-word

# mode
net a6-opmode         cia402.0.opmode           => lcec.0.A61.control-mode
net a6-opmode-display lcec.0.A61.mode-display    => cia402.0.opmode-display

# position/velocity in drive units
net a6-act-pos        lcec.0.A61.actual-position => cia402.0.drv-actual-position
net a6-act-vel        lcec.0.A61.actual-velocity => cia402.0.drv-actual-velocity
net a6-tgt-pos        cia402.0.drv-target-position => lcec.0.A61.target-position
net a6-tgt-vel        cia402.0.drv-target-velocity => lcec.0.A61.target-velocity
```

now:

```hal
show pin cia402.0
````

to get a list of all the inputs and outputs for the cia402 component "talking" to the drive

look for:
- cia402.0.statusword = 0x1650 (non-zero, live from drive) and the decoded bits make sense
- stat-remote = TRUE
- stat-target-reached = TRUE
- opmode-display = 8 and opmode = 8 → CSP selected and confirmed (drive display should show the same)
- drv-actual-position and drv-actual-velocity are non-zero and changing → feedback is wired correctly
- nets (<== a6-* / ==> a6-*) show the drive PDO pins are correctly connected to the cia402 component.

At this point, halrun has proven the EtherCAT + CiA402 contract is correct.


# 3.2) Get a base .hal file

I would recommend to get a base .hal from the forum. For example from Rod who also wrote the step-by-step for the install
https://github.com/rodw-au/linuxcnc-cia402/tree/main

I will be using a different one as a base, which already uses my type of drives. They are quite simmilar, but this one structured somewhat different which makes it easier for me to grasp.
https://github.com/CollinBardini/linuxcnc-a6-servo/blob/main/A6-machine.hal

Important is, that the naming in the linuxcncethercat config (.xml) file is consisten with the naming in the .hal file
A solid approach is to paste the current XML, the cia402.0 pin names and one or two example .hal files into an LLM to get starting file. 

As i have addional IO, manly for controlling the VFD and auxilery components such as pumps, lights, coolant and so on I wont be using the servo drives internal homing feature and instead just use the linux cnc homing.
At least for now, as i get the system up and running.

At this stage i have not really set any funtions, more or less only loaded the Pins for the beckhoff IO:
and have adapted the drive section to the naming in my config .xml file:

```ini
# EtherCAT starter HAL 
# -----------------------------
# Motion core
loadrt [KINS]KINEMATICS
loadrt [EMCMOT]EMCMOT servo_period_nsec=[EMCMOT]SERVO_PERIOD num_joints=[KINS]JOINTS

# -----------------------------
# EtherCAT + CiA402
loadusr -W lcec_conf ethercat-conf.xml
loadrt lcec
loadrt cia402 count=1

# Servo-thread scheduling (order matters)
addf lcec.read-all            servo-thread
addf cia402.0.read-all        servo-thread
addf motion-command-handler   servo-thread
addf motion-controller        servo-thread
addf cia402.0.write-all       servo-thread
addf lcec.write-all           servo-thread

# Allow LinuxCNC to enable (iocontrol)
setp iocontrol.0.emc-enable-in 1


# -----------------------------
# Beckhoff I/O (EL1008 / EL2008 / EL2624 / EL3102 / EL4022)

# --- Digital inputs (EL1008) ---
net di0  <= lcec.0.di8.din-0
net di1  <= lcec.0.di8.din-1
net di2  <= lcec.0.di8.din-2
net di3  <= lcec.0.di8.din-3
net di4  <= lcec.0.di8.din-4
net di5  <= lcec.0.di8.din-5
net di6  <= lcec.0.di8.din-6
net di7  <= lcec.0.di8.din-7

# wiring placeholders (uncomment + set numbers once you know them)
# net estop-loop-ok  <= lcec.0.di8.din-X
# net x-home-sw      <= lcec.0.di8.din-X

# --- Digital outputs (EL2008) ---
# Safe default: force low on startup
setp lcec.0.do8.dout-0 0
setp lcec.0.do8.dout-1 0
setp lcec.0.do8.dout-2 0
setp lcec.0.do8.dout-3 0
setp lcec.0.do8.dout-4 0
setp lcec.0.do8.dout-5 0
setp lcec.0.do8.dout-6 0
setp lcec.0.do8.dout-7 0

# Convenience: DO0 = machine enabled indicator (LED)
net machine-enabled  iocontrol.0.user-enable-out  => lcec.0.do8.dout-0

# --- Relay outputs (EL2624) ---
setp lcec.0.relay4.dout-0 0
setp lcec.0.relay4.dout-1 0
setp lcec.0.relay4.dout-2 0
setp lcec.0.relay4.dout-3 0

# Examples (uncomment if desired)
# net coolant-flood  iocontrol.0.coolant-flood  => lcec.0.relay4.dout-0
# net coolant-mist   iocontrol.0.coolant-mist   => lcec.0.relay4.dout-1

# --- Analog inputs (EL3102) ---
net ai0  <= lcec.0.ai2.ain-0-val
net ai1  <= lcec.0.ai2.ain-1-val

# --- Analog outputs (EL4022, 4-20 mA) ---
# Safe default: disabled
setp lcec.0.ao2.aout-0-enable 0
setp lcec.0.ao2.aout-1-enable 0
# When ready:
# setp lcec.0.ao2.aout-0-enable 1
# net ao0-cmd  some-signal  => lcec.0.ao2.aout-0-value


# -----------------------------
# Drive / Axis wiring (Joint 0 -> A6 via cia402)

# Mode: Cyclic Synchronous Position (CSP, CiA402 mode 8)
setp cia402.0.csp-mode 1

# Position scale for 4mm pitch
# 17-bit -> 131072 counts/rev; counts/mm = 131072 / 4 = 32768
setp cia402.0.pos-scale 32768

# Motion <-> cia402
net x-enable        <= joint.0.amp-enable-out  => cia402.0.enable
net x-amp-fault     => joint.0.amp-fault-in    <= cia402.0.drv-fault
net x-pos-cmd       <= joint.0.motor-pos-cmd   => cia402.0.pos-cmd
net x-pos-fb        => joint.0.motor-pos-fb    <= cia402.0.pos-fb

# EtherCAT PDOs -> cia402 (feedback/state)
net x-statusword       lcec.0.A61.status-word     => cia402.0.statusword
net x-opmode-display   lcec.0.A61.mode-display    => cia402.0.opmode-display
net x-drv-act-pos      lcec.0.A61.actual-position => cia402.0.drv-actual-position
net x-drv-act-vel      lcec.0.A61.actual-velocity => cia402.0.drv-actual-velocity

# cia402 -> EtherCAT PDOs (commands)
net x-controlword      cia402.0.controlword         => lcec.0.A61.control-word
net x-opmode           cia402.0.opmode              => lcec.0.A61.control-mode
net x-drv-target-pos   cia402.0.drv-target-position => lcec.0.A61.target-position
net x-drv-target-vel   cia402.0.drv-target-velocity => lcec.0.A61.target-velocity

# monitoring
net x-drv-act-torque   lcec.0.A61.actual-torque
```


# 3.3) Setup a linux cnc config

- With the .hal prepared, use the stepconfig to generate a bench LinuxCNC config. Just cklick trough, no settings here. this generates all needed files in the named config folder
- copy known XML and starting HAL into this folder
- go into the generated .ini look for the hal section and only let the prepared starting HAL be loaded:

```ini
[HAL]
HALFILE = ethercat.hal
```

Save and you should be able to start the linux cnc bench config!

now you can see and set all pins also via 
- Open Halshow (Machine → Show HAL Configuration)
- Go to Pins
- Search lcec.0.do8.dout-0 (or lcec.0.relay4.dout-0)
- Select/watch the pin → use Set / Clear (or toggle)

enabling the machine, the servodrive should run as well. With the base settings, a following error can occur. 
Twaking accel and following error settings allowed me to test also higher motor speeds:

'''ini
[AXIS_X]
MAX_VELOCITY = 50.0
MAX_ACCELERATION = 750.0
MIN_LIMIT = -0.001
MAX_LIMIT = 200.0

[JOINT_0]
TYPE = LINEAR
HOME = 0.0
MIN_LIMIT = -0.001
MAX_LIMIT = 200.0
MAX_VELOCITY = 50.0
MAX_ACCELERATION = 500.0
FERROR = 5
MIN_FERROR = .5
```

This concluds the bench setup.
-Ethercat Bus is running
-Input/Ouputs can be read and servo drive also functions
-LinuxCNC can accsess all Ethercat components+
