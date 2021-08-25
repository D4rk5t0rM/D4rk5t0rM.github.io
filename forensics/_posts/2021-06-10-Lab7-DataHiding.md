 this lab we'll simulate the whole forensics process with a Windows file system as if we were the forensics investigators.

# Disclaimer
Not all tools/files/VMs/networks will be available. These are notes taken for labs duiring a course, so some stuff is copyright protected and can't be shared. Where possible I'll try to link the resources.

# Concept
This lab simulates a real world forensic case where the student will act as forensic investigator. A fictive company has had a data breach on one of their employees computers and has ordered a full investigation to determine the root cause (and verify that the employee is a victim and not involved in this).

# Scenario
You have been hired by local law enforcement to act as an expert witness in a legal case. They have provided you with some key artifacts recovered from an individual suspected from planning an armed robbery. The suspect denies any knowledge about this crime so it will be up to the forensics experts to shed some light on this difficult case.

# Forensic artifacts
- USB device: `usb.dd`
- Registry Hives: DEFAULT, NTUSER.DAT, SAM, SECURITY, SOFTWARE, SYSTEM
- Log File: Setupapi.dev.log
- sticky note with: `banana DES`

# Forensics report
## Try to answer (at least) the following questions:
1. Has this USB Storage Device been involved in any illegal activities?
2. If so: What activities? All possible details you can find about the activity (names, dates,
locations)
3. Can the USB stick be linked to the suspect his Personal Computer with the artifacts provided?
4. If so: Any suspicious actions on the PC linked to the USB stick?
5. Meaning of the sticky note recovered from the suspects monitor?

# The USB
The USB device has a hidden file called `.The Message.txt` this file contains a message to the rober's partner telling them their plan to rob the victim; Meaning that this USB was involved in illegal activities. This is further confirmed by the `blueprints.jpg` picture on this disk.

Appernetly, according to the blueprints the criminals want to rob Scrooge McDuck