I have used claude to package the Android Auto Files from Squooky and the SSH unlocker. 
Requirements:
-ENet Cable w/ ENet to USB C adapter (or your phone's matching USB port form factor)
-Wifi/Data connetion
-Android's ethernet tethering must be enabled AFTER connecting to the car's ports and attaching to phone

Instructions:
1.) Download the app from the Release section "app-release.apk" 

2.) Install APK on your Android Phone

3.) Connect USB C to ENet adapter with car in key on engine off

4.) Enable Ethernet Tethering in Settings on Android Phone

5.) Open App and ensure you have green check boxes for connectivity, wait a few seconds for your car to be detected, setup should continue to the next screen on its own.

6.) Select "This is my car - Continute" (You may need to choose the "Use this one" button if multiple are listed and a failure occures)

7.) Select "Enable SSH", after a few seconds the install screen should appear

8.) Select "Find Head unit" Once

9.) Ensure Battery on Android Phone is over 20% for flash, then select "Install Now"

10.) After install completes, hold the power button/volume knob on radio for 30 seconds/until radio reboots

As mentioned below, do not expect it to work out of the box, you most likely will need to delete your phone from the connected devices and reconnect it as a bluetooth media/music device. Then select the Android Auto Tab and "Open Android Auto"

Hope this helps!

PS: There are extra debug features to backup and restore all changes, log all interactions with the vehicle and log errors that can be used for repairs later to the app/installer. Some debug features may not work as intended, but main flow appears to work perfect. All credit to Squooky and the SSH Unlock tool

____________________________________


# Overview
I'm a believer of humans writing things, but this is well beyond my skillset, so I (me, Ronnie) am writing this (the words on the page you're reading). I'm not a C-dev, but I do know these things can be done... So... This repository contains code for the J29 Toyota Supra to run Android Auto natively on the NBT Evo head unit. Yeah, talk about a big one. I'll preface it with this: Big screw you to all the companies who have profited for **years** on MMI units. You could have drove innovation, yet you chose to implement a "just good enough" solution to get people to bite. Okay, now that that's out of the way...

# A Note on Bench Setups & Retrofits
I cannot provide support for bench setups. I cannot provide support for retrofits. I cannot guarantee hardware compatibility. I'm not familiar enough with 4 -> 6, or 5 -> 6, and cannot verify the hardware will work. I need to test on things I have easy access to, and I just don't have that hardware.
I went through maybe 60-90 rounds of testing on my real car, and I cannot do that with you remotely.

## License
NBT-AAD is licensed under the **PolyForm Noncommercial License 1.0.0** (`SPDX-License-Identifier: PolyForm-Noncommercial-1.0.0`).
The full text is embedded in `src/main.cpp` / `src/license_text.hpp` and printed by `./nbt-aad --license`.

Required Notice: Copyright (c) 2026 nanaisu - NBT-AAD (https://github.com/Sq00ky/Supra-iDrive6-Android-Auto)

NBT-AAD is free of charge. Selling it, or bundling it with anything sold, is not permitted.
If you paid for it, you were scammed: official releases are only published at https://github.com/Sq00ky/Supra-iDrive6-Android-Auto
Please report any sellers in here: https://github.com/Sq00ky/Supra-iDrive6-Android-Auto/issues

---

The only authorized bundlers is TurboCode for directly supporting the development of this project by offering free SSH unlocks, and providing guidance for native integration.

## Disclaimer
**I am not responsible for anything you do on your device, if you execute a command, or change files, do so with care - if you break something, only Toyota/BMW can reflash and virginize your head unit. The code is shipped as-is. You may fork the repo, clone it, change whatever you like. I cannot ensure compatibility.**

## Requirements
- Temporarily Apple CarPlay (Looking into WiFi Direct compatibility)
- SSH Access to your NBT Evo head unit.
- An Ethernet -> ODB2 cable
- A computer w/ an SSH Client

## No Flashdrive? No Problem.
Grab a QNX Compiled SCP Binary and run:
```
ssh root@<hu-ip> 'cat > /tmp/scp' < /path/to/scp/scp
ssh root@<hu-ip>
mv /tmp/scp /bin/scp
chmod +x /bin/scp
```

## Instructions
Installation instructions can be found in the [install.md file](https://github.com/Sq00ky/Supra-iDrive6-Android-Auto/blob/main/install.md)

### Common Problems
1. We've noticed running AAIDrive in parallel can break NBT-AAD. We'd suggest uninstalling it for the time being until I have some more time to experiment.
2. We've also noticed that a /dev/btspp0 not found error. This seems to come an issue with the head unit not fully opening a Bluetooth channel. I recommend trying to re-pair your phone, or going into the device settings in iDrive and go through the list. Tap "Phone", tap "Bluetooth Audio", tap "Apps", and see if any of them trigger a Bluetooth connection again, and see if your phone starts searing for Android Auto.

## Building Yourself
The code needs to be compatible with Armv7LE. I built it with [QNX Momentics](https://archive.org/details/qnxsdp-65.7z) because that's probably what the developers iDrive6 did. Be warned, this is from archive.org. Run it in a VM. I cannot guarntee the safety/security of the software. It is as simple as creating a new project in Momentics, adding all the files to the project folder, right clicking on your project folder, selecting "clean", and then "build". 

### Feeling Generous?
I don't charge for this project, I don't charge for deployments, if this has helped you and you're feeling generous, here's my paypal: https://paypal.me/banananaisu
Please do not feel obligated to send me anything. I've had a couple people ask, if it makes you feel better, you can buy me a drink. I like Red Wine & Bourbon :) My goal in this was never to make any money. It is to provide a cheap/low cost solution so we don't have to deal with MMI products & don't have to deal with people selling native Android Auto integrations.
