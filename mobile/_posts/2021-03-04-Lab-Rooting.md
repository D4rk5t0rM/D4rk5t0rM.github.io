# Lab part I: rooting an android emulator - SOLUTION

If you don't have an old android device to root, then follow this lab.

In this lab you will root an android emulator.


## Prerequisites

  - Install latest android studio (should be already done).
    - Choose custom installation.
  - When done installing, create an empty solution of choice.
  - After some minutes, AVD manager is available in the Tools menu.
    - If not: double press shift and type AVD manager.
  - Create a new Pixel 3a android 28 (**Google API**) device.
    - **Don't choose an emulator with Google Play in description !!**
  - Name your device in the last step (an easy name without whitespaces).
  - Close android studio.
  - Install platform tools on your host (laptop)

## How to boot the emulator
  - Execute the following command:
    - `$**~/Android/Sdk/emulator/emulator** -avd **My-first-emulator** -writable-system -selinux disabled -qemu -enable-kvm`
    - Please replace the **bold** (`** <text> **`) content with the exact **path** and **emulator name**.
  - The emulator should pop up.
 

## Goal

A rooted android emulator.

## How to test if the lab succeeded.

- When opening RootChecker, SuperSU should popup to grant Rootchecker for root privileges.
- When running RootChecker, the verifRoot button should work.

## Lab

Below you find the order of actions you have to take.
Fastboot and flashing boot images is not possible on emulators.
The steps for rooting are a little bit different than seen on the slides.
Therefore the actions below are described abstract on purpose.
The challenge of this lab is to make them concrete.

- Warning:

    - Don't ask the coaches immediatly for answers.
    - Read, understand, try, fail, try again, research more, ...

- Steps:
    - Verify the emulator is listed. (Which command?)
    - Execute adb root and adb remount.
      - Why? **otherwise the directories are read only**
      - If remount does not work, try to use the ./adb remount command in the platfom-tools folders (see Prerequisites)
    - Install rootchecker.apk (see Leho for download).
      - Donwload from Leho
      - `adb install /path/to/rootchecker.apk`
    - Verify that your are not root yet!
      - **open rootchecker, skip al steps, click verify root**
    - Clone the https://github.com/0xFireball/root_avd on the host
    - Install Superuser.apk (find it in root_avd/SuperSU/common) on the emulator.
      - `adb install root_avd/SuperSU/common/Superuser.apk`
    - Push the `SuperSU/x86/su` to `/system/xbin/su`
      - `adb push root_avd/SuperSU/x86/su /system/xbin/su`
      - What is the main difference between the x86/su and the Superuser.apk?
        - **Superuser.apk is just a verification apk that monitors other apps that are trying to get root access**
      - What is the goal of these two different things? (Think about it while completing the lab).
       - **See above**
    - Open the SuperUser application and install the app in **normal** mode.
      - Errors can be ignored.
      - Don't reboot but tab ok.
    - We need to add persmissions to our new su.
      - `adb shell chmod 0755 /system/xbin/su` **// extra permissions**
      - `adb shell setenforce 0` **// disable firewall**
      - `adb shell su --install` **// install the su comman**
      - `adb shell su --daemon &` **// This command will Run SuperSU’s su as daemon.**
      - Describe for each step the purpose (research it).
    - See the section (how to test) to see if the lab succeeded.
