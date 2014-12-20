###BrcmPatchRAM

Most Broadcom USB Bluetooth devices make use of a system called RAMUSB.

RAMUSB allows the firmware for the device to be updated on-the-fly, however any updates previously applied are lost when shutting down the machine.

The Broadcom Windows driver will upload firmware into the Broadcom Bluetooth device on every startup, however for OS X this functionality is not supported out of the box.

BrcmPatchRAM kext is an OS X driver which applies PatchRAM updates for Broadcom RAMUSB based devices.

It will apply the firmware update to your Broadcom Bluetooth device on every startup / wakeup, identical to the Windows drivers.

The firmware applied is extracted from the Windows drivers and the functionality should be equal to Windows.

Note that the original Apple Broadcom bluetooth devices are not RAMUSB devices, and thus do not have the same firmware mechanism.

####Installation

BrcmPatchRAM can be installed either through Clover kext injection or placed in /System/Library/Extensions for legacy installations.

####Details

BrcmPatchRAM consists of 3 parts:

 - BrcmPatchRAM itself communicates with supported Broadcom Bluetooth USB devices (as configured in the Info.plist), and detects if they require a firmware update. 
 
  If a firmware update is required, the matching firmware data will be uploaded to the 
	device and the device will be reset.
	
 - BrcmFirmwareStore is a shared resource which holds all the configured firmwares for different Broadcom Bluetooth USB devices.

   Some devices require device specific firmware, while others can use the newest version available in the Windows drivers without issue.

   New firmwares are added/configured on a regular basis to support devices, so be sure to follow release updates, or log an issue if you find your device is not supported.

	Firmwares can be stored using zlib compression in order to keep the configuration size manageable.
	
 - BrcmNonApple is a code-less Plug-In kext which ensures that the default Apple Broadcom Bluetooth USB drivers are matched against the 3rd party devices supported by BrcmPatchRAM.

   Through the plist configuration 3rd party devices are handled using com.apple.iokit.BroadcomBluetoothHostControllerUSBTransport.
	Additionally a device description is merged which is used in the System profiler to identify the device.

After the device firmware is uploaded, the device control is handed over to Apple's BroadcomBluetoothHostControllerUSBTransport.
This means that for all intents and purposes your device will be native on OS X and support all functionalities fully.

It is possible to use the Continuity Activation Patch in combination with BrcmPatchRAM through Clover or through dokterdok's script: https://github.com/dokterdok/Continuity-Activation-Tool 

For Clover use the following kext patch:

    <dict>
        <key>Comment</key>
        <string>IOBluetoothFamily - Continuity &amp; Hand-off</string>
        <key>Find</key>
        <data>i4eMAQAA</data>
        <key>Name</key>
        <string>IOBluetoothFamily</string>
        <key>Replace</key>
        <data>uA8AAACQ</data>
    </dict>
	
####Troubleshooting

After installing BrcmPatchRAM, even though your Bluetooth icon may show up, it could be that the firmware has not been properly updated.

Verify the firmware is updated by going to System Information and check the Bluetooth firmware version number under the Bluetooth information panel.

If the version number is "4096", this means no firmware was updated for your device and it will not work properly.

Verify any errors in the system log by running the following command in the terminal:

    cat /var/log/system.log | grep -i brcm[fp]
	
Ensure you check only the latest boot messages, as the system.log might go back several days.

If the firmware upload failed with an error, try installing the debug version of BrcmPatchRAM in order to get more detailed information in the log.

In order to report an error log an issue on github with the following information:
  
 - Device product ID
 - Device vendor ID
 - BrcmPatchRAM version used
 - Dump of BrcmPatchRAM debug output from /var/log/system.log showing the firmware upload failure

####Firmware Compatibility

Some USB devices are very firmware specific and trying to upload any other firmware for the same chipset into them will fail.

This usually displays in the system log as:
    
    BrcmPatchRAM: Version 0.5 starting.
	BrcmPatchRAM: USB [0a5c:21e8 5CF3706267E9 v274] "BCM20702A0" by "Broadcom Corp"
	BrcmFirmwareStore: Retrieved firmware for firmware key "BCM20702A1_001.002.014.1443.1612_v5708".
	BrcmFirmwareStore: Decompressed firmware (29714 bytes --> 70016 bytes).
	BrcmPatchRAM: device request failed (0xe000404f).
	BrcmPatchRAM: Failed to reset the device (0xe00002d5).
	BrcmPatchRAM: Unable to get device status (0xe000404f).
	BrcmPatchRAM: Firmware upgrade completed successfully.

The errors in between mean the firmware was not uploaded successfully, and the device will most likely need a specific firmware configured.

For other devices the newest firmware available (even though not specified specifically in the Windows drivers) works fine.

#### New devices

In order to support a new device, the firmware for the device needs to be extracted from existing Windows drivers.

A copy of the (current) latest Broadcom USB bluetooth drivers can be found here:
http://drivers.softpedia.com/get/BLUETOOTH/Broadcom/ASUS-X99-DELUXE-Broadcom-Bluetooth-Driver-6515800-12009860.shtml#download

*Should you come across newer drivers than 12.0.0.9860, please let me know.*

In order to get the device specific firmware for your device take the following steps:

 - Look up your USB device vendor and product ID, in this example we will be using the BCM94352Z PCI NGFF WiFi/BT combo card, for which the vendor is 0930 and product ID 0233.
  
 - Extract the Windows Bluetooth driver package and open the bcbtums-win8x64-brcm.inf file.
  
 - Find your vendor / device ID combination in the .inf file
    
        %BRCM20702.DeviceDesc%=BlueRAMUSB0223, USB\VID_0930&PID_0223       ; 20702A1 Toshiba 4352
				
 - Locate the mentioned "RAMUSB0223" device in the .inf file:
  
		;;;;;;;;;;;;;RAMUSB0223;;;;;;;;;;;;;;;;;
		[RAMUSB0223.CopyList]
		bcbtums.sys
		btwampfl.sys
		BCM20702A1_001.002.014.1443.1457.hex
	
		
 -  Copy the firmware hex file matching your device from the Windows package, in this case "BCM20702A1_001.002.014.1443.1457.hex"
	
	
 -  The firmware file can now optionally be compressed using the included zlib.pl script:
	
	    zlib.pl deflate BCM20702A1_001.002.014.1443.1457.hex > BCM20702A1_001.002.014.1443.1457.zhx
	
 - After this a hex dump can be created for pasting into a plist editor:
	
	    xxd -ps BCM20702A1_001.002.014.1443.1457.zhx > BCM20702A1_001.002.014.1443.1457.dmp
		
 - Using a plist editor create a new firmware key under the *BcmFirmwareStore/Firmwares* dictionary.
 
  Note that the version number displayed in OS X is the last number in the file name (1457 in our sample) + 4096.

  So in this case the firmware version in OS X would be: "*c14 v5553*".
	
 - After configuring a key under *BcmFirmwareStore/Firmwares*, add your device ID as a new device for BrcmPatchRAM.
 
 Configure the earlier firmware using its unique firmware key.
 
 - Open the BrcmNonApple.kext Info.plist and configured your device for both AppleUSBMergeNub as well as BroadcomBluetoothHostControllerUSBTransport.
 
 To do this make a copy of an existing device and modify the device vendor id, product id and USB description as required.
