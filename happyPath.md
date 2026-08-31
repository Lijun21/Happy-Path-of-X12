I have worked with X12 for a week, some notions I dicovered helpful, known what to expect as each step and where, what triggers what, helps a lot in diganose issue with the server.
each step can go wrong, I may expand the doc in other file, this doc focus on happy path of X12 and what is indicates. 

step 1, plug server into wall outlet.

It wakes up BMC microprocessor. It's bootup takes 4 stages: 
    1. Boot ROM, find firmware in flash memeory and verify its signature, then jump to it. 
    2. Bootloader(U-Boot), initialize the BMC's DRAM, then loads the real firmware.
    3. Embeded Linux, a stripped down linux distro, It starts the network stack, and launches services. 
    4. Scan and monitor. reads sensors over I2C buses(temperature, volages on every rail, fan speeds), reads FRU EEPROMs that identify what boards are installed, spin fans to a safe speed, and starts logging.

    then wait for power on command, when it comes, BMC along with CPLD kicks off the main power sequence. watches each rail's Power Good signal come up in order, and only then relases the host CPU from reset.?
    If a rail fails to come up, the BMC aborts the sequence and logs the fault. 


Power LEDs linking green. Indicate AC present, standby only(5VSB or12 VSB).
    power sequenceing for BMC, its CPU, DRAM, IO
    PSU delivers bulk 12V(plus 5V/3.3V standby) and asserts a "Power Good" signal. The real power sequencing happends on MB.
    MB has converters, small circuits called VRM(voltage regulater module)
    a few MOSFETs, a coil, and capacitors. 
    it takes 12V and converts it down further into many small rails - 1V for CPU core, 1.2V for memory, 1.8V for IO, and so on. 
    power sequencing:
    CPU core transistor have shunk so small that they can switch with tiny voltages, it wants low voltage 0.7 to 1.2V
    I/O  need higher voltage, Signals leaving the chip must drive long traces, survice noise, and match what the other chip expects. 1.8V, 3.3V.
    Momory has its own standard. DDR4 1.2V, DDR5 1.1V, VPP 2.5V.
    Analog circuits, PPLS, SerDes, ADCs want clean, dedicated rails.
    Legacy. Fan 12V, USB 5V.

    5VSB VS 5V 
        Yes, SB = standby. Here’s how to decode all these names:
        The P prefix = “power rail” (some schematics use V instead, or nothing).
        The voltage, with V replacing the decimal point:
        * P3V3 = 3.3V
        * P1V0 = 1.0V
        * P1V2 = 1.2V
        * P0V75 = 0.75V
        * P12V = 12V
        * P5V = 5V
        The suffix tells you the domain or the customer:
        * _STBY or _SB or _AUX = standby domain (alive on wall power)
        * _BMC = feeds the BMC
        * _DDR or _MEM = feeds memory
        * _CPU or _VCORE / _VDDCR = CPU core rail
        * _PCH = chipset
        * _S5, _S3, _S0 = tied to ACPI sleep states (S0 = fully on, S5 = soft-off; a rail named _S5 is alive even in soft-off — basically another way of saying standby)
        So reading a schematic: P3V3_STBY = “3.3-volt rail in the standby domain,” P1V2_DDR = “1.2-volt rail feeding the DDR memory.” You might also see PGD or PWRGD suffixes on signal names — those are the Power Good signals we discussed, e.g., P1V0_BMC_PWRGD = “the 1.0V BMC rail is up and in spec.”
        Vendors vary a bit (Intel reference designs, Dell, Supermicro each have house styles), but this pattern covers most of what you’ll see. Once you can read rail names, a board’s power tree and its sequencing order almost explain themselves — which is exactly why the naming is so disciplined.



Fans should be on 100%. 
    Fan are controled by its own circuits call PMW signal, no signal = 100% speed. When power first arrives, nobody is generating PWM yet(the BMC is still booting), so the fans default to full blast. It's not a command, it's absence of a command. 
    Once the BMC reaches stage 3/4 and its fan-control service starts, it begins generating PWM based on temperater sensor readings. temperatoer up, BMC raise it.




Host power on, main rail on, PSU LED solid green. Full 12V/5V conversion.








HOST CPU vs BMC CPU voltage change
    Host CPU use DVFS to adjust voltage and frequency to save energy when idal, and increase voltage and frequency when need to run intense load. Those circuit output as the input of next circuit, are tity capacitors need to be filled with current, high voltage filled them faster, and the result comes out faster, so the clock frequency has to raise together. 

    BMC CPU runs ARM cores at a fixed voltage and fixed frequency. around 800 MHz TO 1.2GHz.


X12 



Show faulty vs hwdiag IO config 
You’ve hit the key distinction: monitoring is continuous and automatic; the command just reads the ledger.
Ever since the BMC’s Linux loaded, its fault-manager service has been watching in the background — polling sensors, listening for error interrupts, evaluating rules (“voltage out of range for X seconds = fault”). Each confirmed problem gets written into its fault database with a timestamp. So when you SSH in and run a “show faults” command, nothing new is scanned — it just displays everything accumulated since boot (actually since the log was last cleared, since these logs persist in flash across reboots). That’s why it returns instantly.
show /System/Open_Problems in ILOM is exactly that: a read-only query of the already-diagnosed, still-active fault list. “Open” means not yet repaired/cleared — ILOM’s fault manager correlates raw sensor events into diagnosed problems (with fault IDs and suspect components), and this command shows the current open ones. Passive display, triggers nothing.
hwdiag is the different animal — that’s ILOM’s active diagnostic shell. Running something like hwdiag io config makes the BMC go query the hardware right now — walking the I2C/PCIe sideband to enumerate what IO devices are present and how they’re configured (slots, risers, what’s plugged where). So yes, that one does trigger real reads of the hardware at your command, though “io config” is still just discovery/read-out — it inspects configuration rather than stress-testing anything. Other hwdiag subcommands can go further and actively exercise components.
Rule of thumb: show commands = read the accumulated ledger (passive). hwdiag = go touch the hardware now (active). One caveat: exact hwdiag subcommand behavior varies by platform and firmware version, so check your server’s service manual for what your specific model implements.


Useful command for X12 
Let me search for current documentation on this, since hwdiag is platform-specific and poorly documented publicly.
Here’s the real command set from Oracle’s official diagnostics guide. First, how to get in: for Oracle X7/X8/X9 servers type start /SP/diag/shell at the ILOM prompt; for X4–X6 use set SESSION mode=restricted. Then organized by what you want to debug:
CPU
* hwdiag cpu info all — dump CPU devices and registers (-v for verbose)
* hwdiag cpu pirom_info all — reads the CPU’s built-in info ROM
* hwdiag system fabric test all — tests the system fabric including QPI bus speed, PCIe link speed, and memory frequency — genuinely active
DIMM
* hwdiag mem info all — memory configuration as detected
* hwdiag mem spd all — reads DIMM SPD info: size, speed, voltage — a real I2C read of each DIMM’s EEPROM; a DIMM that won’t answer here is dead/unseated
SSD / NVMe
* hwdiag io nvme_info -v — reads VPD and management interface data from NVMe devices
* hwdiag io nvme_test -v — checks PCIe link width and speed of NVMe drives — catches a drive that trained at x1 instead of x4
PCIe / Risers
* hwdiag pci scan — scans all PCIe devices right now — your “is the card even seen” test
* hwdiag pci info all — link width/speed per device (riser problems show up as missing or degraded links)
* hwdiag pci status all, hwdiag pci read / dump — raw register reads for deep debugging
LED
* hwdiag led get all — state of every LED (compare against what’s physically lit)
* hwdiag led info all — the LED control registers behind them
FRU / presence / I2C
* hwdiag i2c scan all — displays accessible i2c devices — actively probes the sideband bus where all FRU EEPROMs and sensors live
* hwdiag i2c test all — connectivity test that returns pass or fail — finds the stuck-bus problem I mentioned earlier
* hwdiag fan info / hwdiag fan get — fan presence and live RPM
Power rails (ties right back to our sequencing discussion)
* hwdiag power get volts all — live voltage of every rail
* hwdiag cpld vr_check — prints voltage regulator status — asks that sequencing CPLD what it saw
* hwdiag cpld log read — the CPLD’s own event log (rail failures during sequencing land here)
Whole-system views
* hwdiag system summary, hwdiag system thermal, hwdiag system version
* hwdiag system port80 — displays host boot progress codes — the BIOS postcode trail, gold for “host won’t boot” debugging
Useful options: -v (verbose), -l filename (log output), -n (loop a command repeatedly), -m (monitoring interval) — that -n/-m combo is great for catching intermittent sensor flakiness. And note not all commands exist on every platform — run hwdiag -h on your box to see your actual list, and help hwdiag -h -v for the full tree. Logs from these runs are readable in the same shell: ls to list, then e.g. cat hwdiag/hwdiag_i2c_test.log.
All of these are “trigger real reads now” commands in the sense that they query hardware at the moment you run them — but they’re inspection-grade (safe to run on a live system), not stress tests.


What kind of signal or error, or at what stage would I see those errors indicate different components have issue. The flow : plug in wall → PSU standby LED (standby power OK) → fans 100% (no PWM yet) → BMC stages 1–2 silent (maybe a link light appears here, ignore it) → Linux up → fans calm + heartbeat LED blinks → DHCP gets IP on the management port → SSH works, ssh into BMC, show faulty, hwdiag IO config, then ask BMC to boot up host server, SSD lights solid green on. HostNIC ip up. That’s it? Now this newly assembled server is completely in good health? At least before stress test?


Your flow is good but it stops one stage too early — and each stage has its own failure signature. Here’s the full map:
Stage: wall power → standby
* No PSU LED at all → PSU dead, or upstream (PDU/cord/breaker)
* PSU LED amber/blinking abnormally → PSU internal fault; check its own event via BMC later
* Fans don’t spin at all → standby rail dead, or fan power path broken
Stage: BMC boot (silent zone)
* Fans stuck at 100% forever, no heartbeat LED → BMC stuck in stage 1–2 (flash corrupt, DDR fail) — serial console is your only window
* Heartbeat blinks but no DHCP/SSH → BMC network config or cable/VLAN issue, not hardware
Stage: SSH in, before host power-on
* show faults / Open_Problems already lists entries → the BMC’s background monitor caught something (a rail out of spec, missing PSU, over-temp) even before host boot
* hwdiag i2c test fails → a device is hanging the sideband bus — often a bad FRU, backplane, or riser
* hwdiag power get volts all shows a standby-domain rail off-nominal → regulator problem
Stage: host power-on (the big one your flow skips over): When you command power-on, the CPLD runs the main rail sequence. Failures here:
* Server tries to power on and immediately drops back off → a main rail failed Power Good; check hwdiag cpld vr_check and cpld log read — this is power sequencing failing in real life
* Powers on, fans up, but no video/no boot → now watch hwdiag system port80: the BIOS writes progress codes as it initializes. Where the code stops tells you what died:
    * stuck very early → CPU/socket problem (bent pins on a new build!)
    * stuck at memory training codes → DIMM issue — bad, unseated, wrong slot population; usually also an ILOM fault event naming the exact DIMM
    * stuck at PCIe enumeration → a bad card or riser hanging the bus (pull cards to isolate)
* Boots but ILOM logs “correctable memory errors” → DIMM works but marginal — reseat or replace before trusting it
Stage: host up. SSD solid green just means power/presence — it says nothing about the drive’s health. NIC IP up means PHY + driver + DHCP work.
So: no, not yet fully healthy. Your flow proves the power path, BMC, and basic host boot. Before calling a new build good, the standard remaining checklist:
1. Full inventory match — does BIOS/OS see all CPUs, all DIMMs (full capacity!), all drives, all NICs? A DIMM that failed training gets silently mapped out — 512GB installed, 480GB visible. hwdiag mem info all or OS-level check. Same for PCIe link widths: hwdiag pci info all — a x16 card running at x8 “works” but is broken.
2. Zero events in a clean log — clear the ILOM event log, run for a while, confirm nothing new appears (especially correctable ECC errors, which are early warnings).
3. SSD real health — SMART/NVMe health via the OS (smartctl, nvme smart-log), not the LED.
4. Then the stress test — memory test, CPU burn-in, disk I/O — because marginal components (a DIMM that errors only when hot, a PSU that sags only at load) only reveal themselves under stress. That’s exactly what burn-in exists for.
A new build that passes inventory-match + clean log + burn-in is what you can honestly call healthy.



The sequence of Happy path with right signal for a completely healthy server before stress test? 

Here’s the complete happy path, every stage with its “correct” signal:
Phase 1: Standby (wall power only)
1. Plug AC → PSU standby LED on (often blinking green = “AC OK, standby only”)
2. Fans blast 100% (no PWM controller yet — this noise is correct)
3. ~30–90 seconds of silence (BMC stages 1–2: ROM → U-Boot → kernel)
4. Heartbeat LED starts rhythmic blink → BMC Linux is up
5. Fans calm down to low speed → fan-control service alive, temps read OK
6. MGMT port: link light + activity blinks → DHCP lease acquired
7. SSH into BMC succeeds
Phase 2: Pre-host checks (still standby) 8. show /System/Open_Problems → zero entries (or clear stale ones and confirm nothing returns) 9. hwdiag i2c test all → PASS (sideband bus healthy, all FRUs answering) 10. hwdiag power get volts all → all standby rails within spec 11. hwdiag mem spd all, fan info → every installed DIMM and fan present and readable
Phase 3: Host power-on 12. Command power-on → main rails sequence up; server stays on (no immediate drop-off = all Power Goods asserted) 13. Fans briefly ramp, then modulate with temperature 14. hwdiag system port80 → codes advancing steadily, no long stall; ends at the “boot to OS” code 15. Video/console shows BIOS splash → memory training passed 16. BIOS inventory matches physically installed hardware exactly: CPU count, full RAM capacity, all drives, all NICs
Phase 4: Host OS up 17. OS boots, host NIC gets IP 18. SSD activity LEDs flicker with I/O (solid green = present; flicker = actually working) 19. hwdiag pci info all → every card at full expected link width/speed (x16 = x16, Gen4 = Gen4) 20. In OS: smartctl/nvme smart-log → all drives healthy, zero media errors 21. All DIMMs visible to OS at rated speed; dmesg clean of hardware errors
Phase 5: Soak 22. Clear ILOM event log, let the machine idle 30–60 min 23. Re-check Open_Problems and event log → still empty (no correctable ECC drips, no rail warnings, no thermal events)
Green light for stress test.
The one-line summary of the whole thing: every stage has a watcher, and health means every watcher saw what it expected — PSU LED for AC, heartbeat for BMC, PWM calm-down for sensors, Power Good chain for rails, port80 for BIOS, inventory match for silicon, clean logs for everything since. The happy path isn’t the absence of signals — it’s the presence of all the right ones in the right order.







