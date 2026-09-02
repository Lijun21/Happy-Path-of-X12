Test Framework

test frameworks like TestView are built in a pretty standard way across server manufacturing


does Testview contains stress test? 
****

I’m working for Hyve Solutions, customer Oracle cloud x12 server, there’s Assemly team, once done assembly there’s SLT, server level test, if it fails send to repair, I work in repair team, I’m able to view SLT test in a internal software TestView, it failed on Power_ON, hostNIC test cases, but I’m able to boot up the server successfully with zero error on bench, I’m kind of curious what’s the difference if SLT test and bench test, all just triggering BMC some commands, then BMC power on host, hostNIC load OS, it’s super simple, why SLT test takes 1 hours for it?


Great question — and it explains a situation you’ll hit constantly in repair: fails SLT, passes bench. The difference is that your bench test and SLT are answering two different questions:
* Bench: “Can this server boot once, right now, in friendly conditions?”
* SLT: “Will this server survive years in an Oracle datacenter, so it never comes back?”
What SLT is actually doing during that hour, beyond your simple power-on:
Strict, scripted checks with thresholds. SLT doesn’t just watch the OS come up. It queries the BMC and OS for every expected component — exact DIMM count and speed, all NICs present at full link speed (25/100G, not just “link up”), correct firmware versions on BMC/BIOS/NICs, sensor readings within limits. One NIC negotiating at a lower speed = fail, even though the server “boots fine.”
Real environment. SLT runs on the test rack’s network fabric with production-like cabling and switches. Your bench probably uses a different cable, different switch, maybe a 1G management port. A hostNIC that fails against the SLT switch (marginal transceiver, flaky SI on one lane, auto-negotiation quirk) can look perfect on a bench link.
Stress and duration. Much of that hour is load testing — CPU/memory stress, NIC traffic tests pushing actual bandwidth, thermal soak with fans and temps monitored. Marginal hardware (a DIMM that errors only when hot, a NIC that drops packets under load) passes a 5-minute boot but fails 40 minutes of hammering.
Timing sensitivity. SLT scripts have timeouts. If Power_ON took longer than the allowed window — slow first-boot memory training, BMC slow to respond — that’s a fail, even though the server eventually got there. This is a classic “fails SLT, boots fine manually” cause.
Zero-tolerance logging. SLT scrapes BMC SEL and dmesg for any correctable errors or warnings. You’d never notice a handful of corrected memory errors on a bench boot; SLT counts them and fails the unit.
So for your Power_ON + hostNIC failure, before concluding “no trouble found,” worth checking in TestView: did Power_ON fail on timeout vs. an actual fault? Did hostNIC fail on link speed, packet loss under traffic, or firmware version mismatch? The failure detail usually tells you whether you’re chasing a marginal part (reseat/replace NIC or cable-side transceiver) or just a slow boot that tripped a timeout.
Also a practical repair-team reality: if you NTF it and send it back through SLT, a marginal part will often fail again — the SLT environment reproduces what your bench can’t. Two failures on the same test case is a strong signal to swap the suspect FRU even if the bench shows zero errors.


I can’t see Hyve’s internal TestView code, but test frameworks like this are built in a pretty standard way across server manufacturing, and yes — under the hood they’re mostly using the same commands you type manually, just wrapped in automation. The typical structure:
Same commands, scripted. A test case is usually a Python/shell script that runs things like ipmitool power on, Redfish API calls to the BMC (POST /redfish/v1/Systems/1/Actions/ComputerSystem.Reset), SSH into the host OS, ethtool, ip link, ping/iperf for NIC traffic. When you power on via BMC on the bench, you’re literally issuing the same IPMI/Redfish command the script does.
The difference is what’s wrapped around the command:
1. Timeouts. The script sends power-on, then polls — “is host power state ON within X seconds? Did the OS respond on the network within Y seconds?” You, manually, just wait as long as it takes. The script gives up at the deadline and marks FAIL. That’s why “boots fine on bench” and “Power_ON timeout in SLT” coexist.
2. Pass/fail criteria. After the command, the script parses output against expected values: exact link speed, exact device count, error counters at zero. You eyeball “looks good”; the script string-matches.
3. Sequencing and no retries. SLT runs cases in a fixed order with fresh state. Some frameworks retry once, many don’t — a single slow response is a fail. On bench, you unconsciously retry (“hmm, try again”) and the second attempt works.
4. Log scraping. After each case, the framework typically pulls BMC SEL and console logs and greps for error patterns.
For your two failures, this points somewhere useful: hostNIC failing on timeout often means the OS/NIC came up slower than the script’s window, or the DHCP/link negotiation on the SLT network took too long — not necessarily a dead NIC. Common physical causes that pass bench but time out in SLT: marginal transceiver/DAC cable seating, a NIC that takes long to train on the SLT switch, or slow firmware initialization.
Two things worth doing in TestView: find the exact timeout value and the log tail from the failed run — it usually shows how far the sequence got before the clock ran out. And check whether Power_ON and hostNIC failures are related: if Power_ON was also a timeout, the whole boot may have been slow (memory training on first boot, BMC busy), and hostNIC just inherited the delay downstream.
If you can, ask the SLT/test engineering team for the test case source or spec doc — most repair teams get read access, and seeing the actual timeout values and expected strings makes root-causing these “timeout but works on bench” units much faster.


TestView script send Power On command is the same as I run ILOM start script Host/Councle, it’s subsequent command Power On the host, then download OS via HOST NIC, am I right? The difference is on bench I don’t verify anything, but on Script, it parses out against expected values, exact PCIE link speed, if degraded, it fails Power On? That seems like a wrong report to log out, link degrade relate to Powee On?



You’re mostly right, and your instinct about the reporting is good. Let me split it:
The command part — correct. TestView’s power-on is the same as your start /System (or start /SYS on older ILOMs) — both just tell the BMC/ILOM to power the host. And yes, in SLT the host then PXE-boots a test OS over the host NIC. Same mechanics you use.
The verification part — partially correct. A well-written Power_ON test case should only verify power-related things: did the power state flip to ON, did POST complete, did the host reach the OS/heartbeat within the timeout. PCIe link speed should be a separate test case (usually something like PCIE_Check or the hostNIC case itself). You’re right that failing “Power_ON” because a link degraded would be misleading — good frameworks don’t do that.
But here’s how these get tangled in practice, which may explain your unit:
1. Cascading failures. If the NIC is slow or flaky, PXE download stalls → the host never reaches “OS booted” heartbeat within the window → Power_ON times out, even though power itself was fine. The root cause is the NIC, but the first case to hit its deadline is Power_ON. Then hostNIC also fails downstream. One marginal NIC, two failure reports — and neither report names the actual mechanism.
2. Overloaded test cases. Some frameworks do cram checks into Power_ON — “boot and verify basic inventory” — which is arguably bad design, but it happens. Then a degraded link fails a case whose name doesn’t reflect the real problem.
Either way, your read is right: the test case name tells you where the clock ran out, not necessarily what’s broken. That’s exactly why you need the log detail, not just the pass/fail column.
For your unit specifically, Power_ON (need to check) + hostNIC (timeout) together smells like scenario 1: a single slow point in the boot-over-network path — NIC training slowly against the SLT switch, marginal cable/transceiver, or slow PXE/DHCP — causing both cases to blow their timeouts, while your bench boot (local SSD? different network? no deadline?) sails through.
One question that would nail it: on your bench, are you booting the same way SLT does — PXE over the same NIC — or booting from local disk? If bench boots from SSD, you’ve never actually exercised the path that failed, and “zero errors on bench” doesn’t clear the NIC.


Bench boot over the same NIC
That’s a useful data point — it means the NIC, at least against your bench’s switch and cable, can train a link and complete a PXE download. So the failure lives in one of the remaining differences:
1. The clock. Your bench has no deadline. Time your bench PXE boot from power-on command to OS up — if it’s, say, 8 minutes and SLT’s window is 6, the unit “works” everywhere but fails SLT every time. Compare against the timeout value in the TestView case (worth asking test engineering for it).
2. The SLT-side hardware. Different switch, different port, different cable/transceiver than your bench. A NIC can negotiate cleanly with one switch and slowly (or at degraded speed) with another — auto-negotiation and link training are a two-party handshake, and marginal signal integrity shows up only against certain link partners or cable lengths. Two checks worth doing:
* In TestView history, has this same SLT slot/port failed other units on hostNIC? If yes, suspect the rack’s cable/transceiver/switch port, not your server.
* Has this same unit failed on different SLT slots? If it fails everywhere while other units pass those slots, the server’s NIC side is the suspect.
3. Shared-infrastructure timing. SLT racks often run many units against one DHCP/PXE/image server. Under load, DHCP offers and image downloads slow down, and a borderline-slow unit tips over the timeout while your bench — one server, idle PXE server — never sees that pressure. If this is the cause, you’d expect intermittent failures across many units at busy times, which TestView history would also show.
So a practical sequence for this unit: pull the failed run’s log tail and see how far boot got (no DHCP? DHCP but stalled download? OS up but heartbeat late?) — each points to a different segment. Reseat the NIC and transceiver/cable ends anyway (cheap, fixes real marginal cases). Then retest, ideally on a different SLT slot. If it passes, note the original slot in your repair record — if that slot keeps eating units, that’s a test-infrastructure ticket, not a repair.
The log tail is the highest-value item here: “failed at DHCP” vs “failed mid-download” vs “OS booted but reported late” are three different repairs.




Can I write my own TestView app? 