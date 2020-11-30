# IgHEtherCATImplementation
This repository contains implementation of IgH EtherCAT Master on Ubuntu 14.04.6 LTS ; kernel 4.4.x
  
# Start from scratch : 
-> check network card interface driver by ;

    $ lshw -C network

if you see your network card driver compare it with supported NIC drivers from both IgH and RTnet

https://etherlab.org/en/ethercat/hardware.php (IgH EtherCAT official page)

http://hg.code.sf.net/p/etherlabmaster/code (IgH EtherCAT repo page)

http://www.rtnet.org/   (RTnet official page.)

https://sourceforge.net/p/rtnet/news/  (RTnet repo page.)

-> make sure that your kernel version is 4.4.x (for IgH EtherCAT) and 3.2.x for RTnet , check by  ;

    $ uname -r 

## Before start to build run these commands to get required libraries for installation.

    $ sudo apt-get update
    $ sudo apt-get install git build-essential automake autoconf libtool pkg-config cmake intltool autoconf-archive libpcre3-dev libglib2.0-dev libgtk-3-dev libxml2-utils libnuma-dev libssl-dev libtool libncurses5 libncurses5-dev autogen libudev-dev libelf-dev
    $ sudo apt-get install kernel-package fakeroot zlib1g-dev bin86 g++ mercurial

    $ git clone https://github.com/veysiadn/IgHEtherCATImplementation
    $ cd IgHEtherCATImplementation
    $ xz -cd linux-4.4.240.tar.xz | tar xvf -
    $ cd linux-4.4.240
    $ xzcat ../patch-4.4.240-rt209.patch.xz | patch -p1
    $ sudo cp /boot/config-4.4.0-148-generic .config
    $ sudo mv ../linux-4.4.240 /usr/src/ -f
    $ cd /usr/src/linux-4.4.240
#### Config file that is referred in here is my kernel file it can vary.Check your boot folder.

    $ sudo make  menuconfig

## In menu that will be show up we select;
 Processor type and features -> Preemption Model -> Fully Preemptible Kernel (RT).

    $ sudo -s
    $ make -j4
    $ make && make modules && make modules_install && make install
    $ reboot
 ## After reboot to make sure about installation check kernel version again 
    $ uname -v


## IgH EtherCAT Master Stack Installation

    $ hg clone http://hg.code.sf.net/p/etherlabmaster/code ethercat-hg
    $ cd ethercat-hg
    $ hg update stable-1.5
    $ sudo  ./bootstrap 
    $ cd
    $ sudo mv ethercat-hg /usr/local/src/
    $ cd /usr/local/src/
    $ sudo ln -s ethercat-hg ethercat

Move into the source directory

    $ cd ethercat

## Configuration part is important for this part refer to

https://etherlab.org/download/ethercat/ethercat-1.5.2.pdf

Chapter 9.2 table 9.1 : configuration options in my case my laptop has

r8169 driver, I have to enable it.You can check the document for detailed instruction.

    $ sudo ./configure --enable-8139too=no --enable-r8169 --prefix=/opt/etherlab
    $ sudo -s
    $ make 
    $ make modules 
    $ make install
    $ make modules_install
-> after succesfull (error free) installation the we'll check HWAddr (Hardware Address, also known as the MAC Address) of the adapter we'd like to use 

(example: eth0) and record it. We'll need to type it in later.


    $ sudo ifconfig
    
    $ sudo mkdir /etc/sysconfig/
    
    $ sudo cp /opt/etherlab/etc/sysconfig/ethercat /etc/sysconfig/
    
    $ sudo nano /etc/sysconfig/ethercat

-> You need to setup this file as prescribed in the EtherCAT manual. You definitely need to change the values for MASTER0_DEVICE, which need the MAC address of the Ethernet card you've selected, and then the driver you'd like to use for that device.

-> For a development system, "generic" is fine. For a production system, the hope is that you've selected a target machine with a supported network device. We typically used cards supported by the r8169 driver, but check the hardware specs if you’re unsure.

Example:

MASTER0_DEVICE="00:0C:29:09:E0:D7"

DEVICE_MODULES="r8169"


    $ cd /opt/etherlab
    $ exit
    
-> Copy the initialization script (If this doesn't work, make sure that there isn't a /etc/init.d/ethercat already. If so, remove it), change its ownership properties.

      $ sudo cp ./etc/init.d/ethercat /etc/init.d/

      $ sudo chmod a+x /etc/init.d/ethercat

      $ sudo ln -s /opt/etherlab/bin/ethercat /usr/local/bin/ethercat

      $ sudo nano /etc/udev/rules.d/99-EtherCAT.rules
      
## Enter the following contents:

    KERNEL=="EtherCAT[0-9]*", MODE="0664", GROUP="users"

    $ sudo cp /etc/sysconfig/ethercat /etc
    $ cd /etc
    $ sudo mv ethercat ethercat.conf

## Now we can test our installation

    $ sudo /etc/init.d/ethercat start
    $ dmesg
    
after this command you should see something like this : 

[ 2038.604876] EtherCAT: Master driver 1.5.2 334c34cfd2e5

[ 2038.605018] EtherCAT: 1 master waiting for devices.

[ 2038.968282] ec_r8169 Gigabit Ethernet driver 2.3LK-NAPI loaded

[ 2038.968303] ec_r8169 0000:03:00.0: can't disable ASPM; OS doesn't have ASPM control

[ 2038.977080] EtherCAT: Accepting DC:FE:07:21:A6:75 as main device for master 0.

[ 2038.977099] ec_r8169 0000:03:00.0 ecm0 (uninitialized): RTL8168g/8111g at 0xffffc90002936000, dc:fe:07:21:a6:75, XID 0c000880 IRQ 127

[ 2038.977106] ec_r8169 0000:03:00.0 ecm0 (uninitialized): jumbo features [frames: 9200 bytes, tx checksumming: ko]

[ 2039.042040] EtherCAT 0: Starting EtherCAT-IDLE thread.

If you face any problem you can check these threads : 

[EtherLAB Mailing List Implementation ](https://lists.etherlab.org/pipermail/etherlab-dev/2014/000384.html)

[EtherLAB Documentation ](https://etherlab.org/download/ethercat/ethercat-1.5.2.pdf)

