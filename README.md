# StadiaController
StadiaController reverse engineering and experimenting.  
Read the [blog post](https://garyodernichts.blogspot.com/2023/01/looking-into-stadia-controller.html) for more information about the flashing process.  

## fcb_parser
Parses FlexSPI Configuration blocks (`flashloader_fcb_*.bin`).  
See the [fcb_parser README](fcb_parser/README.md) for more information.

## stadiatool
Tool to interact with the Stadia Controller.  
See the [stadiatool README](stadiatool/README.md) for more information.

## Files
The following files can be directly downloaded from Google's Stadia site:
- [bruce_dvt_a_dev_signed.bin](https://stadia.google.com/controller/data/bruce_dvt_a_dev_signed.bin)
- [bruce_dvt_a_stage_signed.bin](https://stadia.google.com/controller/data/bruce_dvt_a_stage_signed.bin)
- [bruce_pvt_a_prod_signed.bin](https://stadia.google.com/controller/data/bruce_pvt_a_prod_signed.bin)
- [flashloader_fcb_get_vendor_id.bin](https://stadia.google.com/controller/data/flashloader_fcb_get_vendor_id.bin)
- [flashloader_fcb_w25q128jw.bin](https://stadia.google.com/controller/data/flashloader_fcb_w25q128jw.bin)
- [restricted_ivt_flashloader.bin](https://stadia.google.com/controller/data/restricted_ivt_flashloader.bin)

Archived firmwares:
- []

# Blog Post

## Looking into the Stadia Controller Bluetooth Mode Website

With the end of Google's Stadia platform on January 18, 2023, Google published a website allowing people to "Switch the Stadia Controller to Bluetooth mode".
This seems pretty cool, but there are two points listed under "Important things to know" which I didn't like:

- **Switching is permanent**
    Once you switch your controller to Bluetooth mode, you can’t change it back to use Wi-Fi on Stadia. You can still play wired with USB in Bluetooth mode.
- **Available until December 31, 2023**
    You can switch to Bluetooth mode, check the controller mode, and check for Bluetooth updates until Dec 31, 2023.

While permanent switching is not a huge issue, since Stadia isn't available anymore, and the Bluetooth mode is way more useful, I still wanted to have the option to switch back.
Since the Stadia Controller's WiFi approach is rather unique, I didn't want to just disable it and no longer have the option to look into it.
 
But only one year to update the firmware and then you're stuck in "Wi-Fi mode" forever? I guess Google really wants to forget about Stadia forever, and get rid of the site after a year.
 
So I started looking into the switching process on the site, to try and avoid those limitations. I also reverse engineered some parts of the binaries hosted on the site, more about that later.

## Analyzing the Bluetooth mode website

The JavaScript used by the site is minified which won't give us function and variable names. It doesn't stop us from seeing what it does and analyzing the packets using Wireshark though.
Note that most of the flashing process seems to be standard NXP stuff, and only contains some minor adjustments by Google. 

The site uses WebUSB and WebHID to communicate with the controller. It filters for several different Vendor and Product ID combinations, to determine the state/mode the controller is currently in.

The switcher loads several files from the data endpoint, which we'll take a look at in more detail later. From taking a rough look at the files and the logs in the JS, the "Bluetooth mode switcher" actually flashes a firmware update to the controller. So from now on I'll be referring to this as "flashing the Bluetooth firmware" and the site as "flashing tool/site".
 
 The site starts by checking the firmware revision and battery percentage while the controller is in the normal, powered on mode, this is referred to as "OEM Mode".
 
## OEM Mode
While in OEM mode, after plugging in the controller to the PC without holding down any buttons, the site communicates with the controller using WebUSB.

It starts by checking the first two bytes of the serial number from the USB string descriptor. There are some prefixes which are not allowed to be flashed. The serial prefix is also used to determine if this controller is a development controller (dvt) or a production controller (pvt).

It then retrieves the current firmware revisions using USB control request 0x81.
Firmware revisions less than 0x4E3E0 are referred to as gotham, while all later revisions are called bruce. gotham being the old Wi-Fi firmware, while bruce is the new Bluetooth firmware.

After that the battery percentage is requested using control request 0x83 and retrieved with request 0x84. This value is used to check if the controller has enough charge (more than 10%) to perform the flashing process.

After all that info has been retrieved, the site asks us to unplug the controller and turn it off.
 
## Bootloader
The site now wants the user to hold down the Options button, while plugging the controller back in. This will enter the Bootloader.
Not much to say about this mode. The site asks us to press Options + Assistant + A + Y while in the Bootloader, which will enter the SDP Mode.
 
## SDP Mode
SDP (Serial Download Protocol) Mode allows sending several low-level commands to the controller.
The flasher uses WebHID to send and receive commands.
It starts by uploading a signed Flashloader binary (restricted_ivt_flashloader.bin) into the controller's memory (@0x20000000), by using the SDP WRITE_FILE command.
It then jumps to the uploaded Flashloader binary (@0x20000400) using a JUMP_ADDRESS command.
The controller is now running the Flashloader.

## Flashloader
The Flashloader is a bit more advanced than the previous modes. It can also receive and send several commands via USB, and the flasher site once again uses WebHID to send and receive those commands.
Google seems to have chosen a restricted version of this Flashloader though, since only a few commands actually used by the flasher are available.
Also only a few, small memory regions are allowed to be read and written using the WriteMemory and ReadMemory commands.

The Flashloader is used to actually write the new firmware into the controllers flash storage.
 
**Detecting the MCU Type**
The site starts by detecting the MCU type, by reading from 0x400D8260. There are two supported types (106XA0 and 106XA1), if the detected type doesn't match one of them it will throw an error.
 
**Detecting the Flash Type**
Since different Stadia Controller models seem to have different flash storage types, the exact chip is now detected. Detecting the flash type is a bit of an interesting approach. 
To communicate with the flash storage a FlexSPI configuration block needs to be loaded and applied. To determine the flash type, the site retrieves the device ID from the flash. It starts by uploading a special configuration block for determining this ID (flashloader_fcb_get_vendor_id.bin) into memory (@0x00002000), and applies this configuration using the ConfigureMemory command.
This configuration block contains some sane values for the different flash chips, and also contains a lookup table (LUT) with different FlexSPI sequences which will be sent to the flash chip.
For the get_vendor_id configuration the first sequence in the LUT, usually used for reading from flash, has been replaced with a Read Manufacture ID/ Device ID command.
Now comes the interesting part: The site now directly configures the FlexSPI registers using ReadMemory/WriteMemory Flashloader commands via USB.
It configures the FlexSPI FIFO and sends the Read Device ID command from the LUT sequence.
It then retrieves the result from the first RX FIFO Data Register.
It seems like writing to and reading from those few FlexSPI registers is explicitly allowed in the flashloader.

**Setting up the Flash Storage**
Now that the flash type is known the site can load the proper configuration block for that chip.
There are two supported flash types (Giga-16m and Winbond-16m).
To setup the Winbond chip an entire flash configuration block (flashloader_fcb_w25q128jw.bin) is loaded and applied.
For the Giga the flash is automatically configured by the Flashloader based on a simple configuration value (0xC0000206).
 
**Flashing the Firmware**
Now that everything is ready the actual firmware flashing can begin.
After clearing GPR Flags 4-6, the site loads the signed target firmware image (<bruce/gotham>_<dvt/pvt>_a_<dev/stage/prod>_signed.bin) and parses some build info values from it.
It also determines where in the flash the firmware should be flashed to. To flash data the site sends a FlashEraseRegion command to erase and unlock the flash, followed by a WriteMemory command to write to the flash mapped in memory @0x60040000.
The IVT (Image Vector Table) is now flashed to @0x60001000 (only if the image contains one), and the actual firmware application gets flashed to the proper slot location (Application A / Application B).

**Cleaning up**
Now that the firmware is flashed, GPR6 is set to the proper application slot and a Reset command is issued to restart the controller.
And that's basically it, the controller is now running the newly flashed firmware.

## Dumping the old Firmware
As mentioned in the beginning, it is not possible to revert to the old Wi-Fi firmware using the Stadia mode switching site, once the new Bluetooth firmware has been flashed.
While the site does seem to technically support flashing the old Wi-Fi firmware, and also has references to the firmware files required for it, all those files lead to a 404 and can't be downloaded.
So to preserve the old Firmware I had to dump it from the controller itself.
 
I tried to read from the flash memory region while in the Flashloader, which only results in errors. It seems like reading from flash is not allowed by the restricted Flashloader.
 
But I had another idea...
Remember that we have direct access to some of the FlexSPI registers, which are used to determine the flash type?
 
Instead of applying the get_vendor_id configuration block and sending the Read Device ID command, I tried applying the proper flash configuration and sending a Read Data command over the registers.
That surprisingly did work without any issues. I could now issue FlexSPI read commands via USB and dump the flash.
 
Since only reading the first register of the RX FIFO Data Registers is allowed by the restricted Flashloader, I had to dump the flash 4-bytes at a time, which did take several hours.
At the end I had a full dump of the Stadia controller flash though!

## Finishing up
During the testing I started reimplementing parts of the site in Python which I called stadiatool, which also allowed me to mess around with the Flashloader commands.
After dumping the flash, I extended the tool to allow flashing the firmware as well.
Note that this was a pretty quick project which is why the code might seem rushed.
You can find the GitHub repo here.

That's it for now, I might take a look at analyzing the firmwares themselves next.
