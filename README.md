# pi-btmesh
###### Bluetooth Mesh API for Raspberry Pi Zero
## Notice:
If you are using Raspberry Pi 2, 3, 3+ or 4, follow [this](https://3pl46c46ctx02p7rzdsvsg21-wpengine.netdna-ssl.com/wp-content/uploads/2020/04/Developer-Study-Guide-How-to-Deploy-BlueZ-on-a-Raspberry-Pi-Board-as-a-Bluetooth-Mesh-Provisioner.pdf) official Bluetooth guide.

## Installation
Bluetooth Mesh requires some cryptographical functions which are not embedded into the kernel by default. Hence the kernel needs to be recompiled. You ***can*** follow the official guide linked above, however in my case the compilation took 23 hours. If you want to be done within 30 minutes, follow these steps:

### Step 1: Set up the connection with your board
In the following steps this guide assumes that you have a working Raspberry Pi zero with the newest Raspbian Lite and you can access it via ssh or any other way. There are numerous guides on how to achieve this, for instance [this one](https://dev.to/vorillaz/headless-raspberry-pi-zero-w-setup-3llj).
Enter the board, and update the system by executing:
```
sudo apt-get update -y
sudo apt-get upgrade -y
sudo apt-get dist-upgrade -y
```

### Step 2: Cross-compile the Kernel on your Linux machine

Download the required tools:
```
sudo apt install git bc bison flex libssl-dev make libc6-dev libncurses5-dev
git clone https://github.com/raspberrypi/tools ~/tools
```
Add the tools to PATH:
```
echo PATH=\$PATH:~/tools/arm-bcm2708/arm-linux-gnueabihf/bin >> ~/.bashrc
source ~/.bashrc
```
Download and untar the kernel:
```
wget https://github.com/raspberrypi/linux/archive/raspberrypi-kernel_1.20200212-1.tar.gz
tar -xvf raspberrypi-kernel_1.20200212-1.tar.gz
cd linux-raspberrypi-kernel_1.20200212-1/
```
Apply the default configuration for Pi Zero:
```
KERNEL=kernel
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- bcmrpi_defconfig
```
Enter configuration menu:
```
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- menuconfig
```
Navigate to ***Cryptographic API menu***, and include the following by pressing ```y```:
- CCM support
- CMAC support
- Include User-space interface for hash algorithms
- Include User-space interface for symmetric key cipher algorithms
- Include User-space interface for AEAD cipher algorithms

Double press the ```Esc``` key to go back until promptede to save the configuration. Select ```save```.

Compile the kernel:
```
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- zImage modules dtbs -j $(nproc)
```
This should take several minutes.
### Step 3: Copy the kernel image to your Pi Zero
Execute
```
lsblk
```
Now remove the memory card from your Pi and insert it into your computer. Execute the command again.
```
lsblk
```
A new entry should have been added to the output, in my case it looks like this:
```
mmcblk0     179:0    0  14,9G  0 disk 
├─mmcblk0p1 179:1    0   256M  0 part /media/oilymacaroni/boot
└─mmcblk0p2 179:2    0  14,6G  0 part /media/oilymacaroni/rootfs
```
The filesystems have mounted automatically. If it doesn't happen, use ```sudo mount <filesystem> <directory>``` to mount them in a desired location.

Install the kernel modules onto the card (replace the INSTALL_MOD_PATH with your own):
```
sudo env PATH=$PATH make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- INSTALL_MOD_PATH=/media/oilymacaroni/rootfs modules_install
```
Copy the kernel image and device tree blobs onto the card:
```
sudo cp /media/oilymacaroni/boot/$KERNEL.img /media/oilymacaroni/boot/$KERNEL-backup.img
sudo cp arch/arm/boot/zImage /media/oilymacaroni/boot/$KERNEL.img
sudo cp arch/arm/boot/dts/*.dtb /media/oilymacaroni/boot/
sudo cp arch/arm/boot/dts/overlays/*.dtb* /media/oilymacaroni/boot/overlays/
sudo cp arch/arm/boot/dts/overlays/README /media/oilymacaroni/boot/overlays/
```
Now unmount the SD card and boot up your Pi Zero. Execute
```
uname -a
```
and check, if the shown kernel build time matches today's date.
The kernel upgrade was successful.

### Step 4: Install the mesh drivers

Install the dependencies:
```
sudo apt-get install -y git bc libusb-dev libdbus-1-dev libglib2.0-dev libudev-dev libical-dev libreadline-dev autoconf bison flex libssl-dev
```

Install the Json-C library. ```apt``` provides version 0.12, but a newer one is required, so the installation is done manually. Firstly, check if the old ```json-c``` is removed and if not, remove it:
```
sudo apt-get remove libjson-c-dev
```

Download and untar the Json-C version 0.13:
```
cd ~
wget https://s3.amazonaws.com/json-c_releases/releases/json-c-0.13.tar.gz
tar -xvf json-c-0.13.tar.gz
cd json-c-0.13/
```

Build and install the library:
```
./configure --prefix=/usr --disable-static && make
sudo make install
```

Download and untar BlueZ
```
cd ~
wget http://www.kernel.org/pub/linux/bluetooth/bluez-5.54.tar.xz
tar -xvf bluez-5.54.tar.xz
cd bluez-5.54/
```

Build BlueZ with mesh support and install it:
```
./configure --enable-mesh --enable-testing --enable-tools --prefix=/usr --mandir=/usr/share/man --sysconfdir=/etc --localstatedir=/var
sudo make
sudo make install
```

Swap the old ```bluetooth-meshd``` for the new one. Firstly, back up the old daemon:
```
sudo cp /usr/lib/bluetooth/bluetoothd /usr/lib/bluetooth/bluetoothd-550.orig
```

Now symlink the old one to the new one by executing:
```
sudo ln -sf /usr/libexec/bluetooth/bluetoothd /usr/lib/bluetooth/bluetoothd
sudo systemctl daemon-reload
```

Create a directory for provisioning databases:
```
cd ~
mkdir .config
cd .config/
mkdir meshctl
cp ~/bluez-5.54/tools/mesh-gatt/local_node.json ~/.config/meshctl/
cp ~/bluez-5.54/tools/mesh-gatt/prov_db.json ~/.config/meshctl/
```

Finally, check if the new clients are present in version 5.54
```
bluetoothd -v
meshctl -v
mesh-cfgclient -v
```

The installation is complete.

### Step 5: Intializing the network:

Now follow [this](https://3pl46c46ctx02p7rzdsvsg21-wpengine.netdna-ssl.com/wp-content/uploads/2020/04/Developer-Study-Guide-How-to-Deploy-BlueZ-on-a-Raspberry-Pi-Board-as-a-Bluetooth-Mesh-Provisioner.pdf) official guide, which shows how to create networks and provision devices.

## Releases
Under the ```releases``` tab you can find ready-to-go images for Pi Zero with BT Mesh API installed and functional.
