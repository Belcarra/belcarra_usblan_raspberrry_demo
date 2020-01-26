# Raspbian IOT Demo


The Belcarra Raspian IOT Demo focuses on using Networking over USB to allow low-cost Raspberry Pi based systems running Raspian to connect to the Windows, Linux or Macintosh-based systems.

Networking over USB provides a simple and secure low-cost networking solution to allow Raspberry Pi based IOT devices to interact with applications running on Windows, Linux and Macintosh.


##Preparation

This section needs to outline general principles of Raspian setup on micro SSD flashcards and point to more specific instructions elsewhere.

Will need to outline our assumptions:
- FAT FS for some configuration
- Linux partition for rest
- Use of Raspian 
- Simple Demo - ECM and RNDIS

## Preliminary steps
After a basic install of Raspian, two additional steps are needed to configure the USB subsystem to operate as needed.   These steps are very simple and are
described below.  After doing this, the network protocols ECM, RNDIS become available immediately as described in the examples.   The additional protocols EEM
and NCM are also available by enabling/building correponding modules.


Depending on Raspberry model, the file config.txt in /boot (the FAT partition as viewed from Linux) should contain either of the following two lines
```
dtoverlay=dwc2  (Raspberry Pi W 0)
dwoverlay=dwc-otg (other models of Raspberry)
```

In the Linux root partition, the file /etc/modules should contain the following 2 lines.
```
i2c-dev
libcompositeConfigure (via FAT FS) to enable USB Device
```

If these steps have been taken, then the Raspberry can be readily configured from a script to become a gadget with a wide range of characteristics.  Some variants will require non-default kernel modules to be present, but these modules are all included in the standard distribution.

## Examples
### Example 1: a flexible ECM device
The following script takes two arguments: a vendor ID and a device ID, both 4 hex digits, leading 0's if necessary. Note that the preliminary configuration is required for the final activation to work.  In addition, you can activate the various pause lines to watch the progress of the script.
```
#!/bin/bash
set -x
pause(){
        echo -n "$1"
        read reply
}
pid=$2
vid=$1
# Create a gadget called testdrive
cd /sys/kernel/config/usb_gadget/
mkdir testdrive
cd testdrive
#pause "Testdrive created"
echo "0x$vid" > idVendor
echo "0x$pid" > idProduct
echo 0x0100 > bcdDevice
echo 0x0200 > bcdUSB
s=strings/0x409
mkdir -p $s
#pause "Strings subdirectory created"
echo "fedcba9876543210" > $s/serialnumber
echo Belcarra > $s/manufacturer
echo RaspberryTestdrive > $s/product
# Create a configuration
mkdir -p configs/c.1/$s
echo "Config 1: ECM" > configs/c.1/$s/configuration
echo 250 > configs/c.1/MaxPower
# Create a function
mkdir -p functions/ecm.usb0
HOST="00:11:22:33:44:55"
SELF="55:44:33:22:11:00"
echo $HOST > functions/ecm.usb0/host_addr
#echo $SELF > functions/ecm.usb0/dev_addr
ln -s functions/ecm.usb0 configs/c.1/
```

# Activate
```
ls /sys/class/udc > UDC

```

Assume that this script is called ecm and is in the current working directory

Then we run the following
```
sudo $PWD/ecm 15ec d031
```

This will produce the following output (all of which can be suppressed):

> pid=d031
> vid=15ec
> cd /sys/kernel/config/usb_gadget/
> mkdir testdrive
> cd testdrive
> echo 0x15ec
> echo 0xd031
> echo 0x0100
> echo 0x0200
> s=strings/0x409
> mkdir -p strings/0x409
> echo fedcba9876543210
> echo Belcarra
> echo RaspberryTestdrive
> mkdir -p configs/c.1/strings/0x409
> echo 'Config 1: ECM'
> echo 250
> mkdir -p functions/ecm.usb0
> HOST=00:11:22:33:44:55
> SELF='55:44:33;22:11:00'
> echo 00:11:22:33:44:55
> ln -s functions/ecm.usb0 configs/c.1/
> ls /sys/class/udc

Running lsmod will now show that the following related modules are loaded:
```
usb_f_ecm
u_ether
dwc2
libcomposite
udc_core
```

We have made the Raspberry a USB gadget using the CDC ECM protocol using device ID 15EC/D031 
For the time being, this is a one-way process: if you want a different protocol, reboot the Raspberry and run a new script.  The next example shows a more complex device.
Example 2: A Composite Gadget
```
#!/bin/bash
set -x Verbose
pause(){
        echo -n "$1"
        read reply
}
pid=$2 Product ID is SECOND argument (4 hex digits)
vid=$1 Vendor ID is FIRST argument (4 hex digits)
# Create a gadget called testdrive
cd /sys/kernel/config/usb_gadget/
mkdir testdrive Create a gadget called testdrive
cd testdrive Set gadget properties (for the USB descriptors)
#pause "Testdrive created"
echo "0x$vid" > idVendor
echo "0x$pid" > idProduct
echo 0x0100 > bcdDevice
echo 0x0200 > bcdUSB
s=strings/0x409
mkdir -p $s
#pause "Strings subdirectory created"
echo "fedcba9876543210" > $s/serialnumber
echo Belcarra > $s/manufacturer
echo RaspberryTestdrive > $s/product
# Create a configuration
mkdir -p configs/c.1/$s
echo "Config 1: ACMECM" > configs/c.1/$s/configuration
echo 250 > configs/c.1/MaxPower
Create component functions, first ACM (serial) This makes the serial component positioned at interface 00
# Create serial function
mkdir -p functions/acm.usb0
ln -s functions/acm.usb0 configs/c.1/
# Create a network function
mkdir -p functions/ecm.usb0
HOST="00:11:22:33:44:55"
SELF="55:44:33;22:11:00"
echo $HOST > functions/ecm.usb0/host_addr
#echo $SELF > functions/ecm.usb0/dev_addr
ln -s functions/ecm.usb0 configs/c.1/
```

# Activate
```
ls /sys/class/udc > UDC
```

After this script is run then the Raspberry becomes a USB device with a two function composite configuration, the first two interfaces runnning CDC ACM (serial) and the second CDC ECM (like the first script).

More network configurations.
Here is a list of the basic USB network protocols
- ECM subset
- ECM
- EEM
- NCM
- RNDIS
All of these protocols are supported in the Debian Linux kernels but only following are included in readily available Raspian builds
- ECM subset
- ECM
- RNDIS
You can also build composite devices with other protocols such as the following
- HID 
- printer
- MIDI
- obex
- ACM
- serial (slightly different, consult Linux kernel docs for details).



#Advanced Demo - EEM and NCM
This section needs to outline what is needed to upgrade Gadget to allow the use of EEM or NCM protocols.

What additional Debian pkg’s are required? GCC, Kernel headers for the installed kernel, Kernel Module build utilities
Instructions on configuring and building additional Gadget modules
Simple scripts that can be installed in Linux /usr/local/bin/ to set up Gadget for EEM or NCM with specified VID/PID
Composite Demo -- see Example 2


