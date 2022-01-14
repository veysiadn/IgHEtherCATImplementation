# IgH EtherCAT Implementation Guide
This repository contains implementation of IgH EtherCAT Master on Ubuntu 14.04.6 LTS ; kernel 4.4.x. If you want to install with different kernel, installation steps are same, you just have to download your desired kernel sources from [Linux Kernel Sources](https://mirrors.edge.kernel.org/pub/linux/kernel/) and [RT_PREEMPT Patch Sources](https://mirrors.edge.kernel.org/pub/linux/kernel/projects/rt/). Hope this repository will save time for you.
  
# Start from scratch : 
-> If you want to use native drivers provided by IgH check network card interface driver by ;

     lshw -C network | grep driver

if you see your network card driver, compare it with supported NIC drivers from  IgH.

[IgH EtherCAT Official Page](https://etherlab.org/en/ethercat/hardware.php) (IgH EtherCAT Native Driver Supported Hardware)

[Source Code IgH EtherCAT](https://gitlab.com/etherlab.org/ethercat.git) (IgH EtherCAT repo page)


-> Check your kernel version ;

     uname -r 

## Before starting to build, run these commands to get required libraries for building/installation.

     sudo apt-get update
     sudo apt-get install git build-essential automake autoconf libtool pkg-config cmake linux-source bc kmod cpio flex -y
     sudo apt-get install intltool autoconf-archive libpcre3-dev libglib2.0-dev libgtk-3-dev libxml2-utils -y
     sudo apt-get install libnuma-dev libssl-dev libtool libncurses5 libncurses5-dev autogen libudev-dev libelf-dev stress -y
     sudo apt-get install kernel-package fakeroot zlib1g-dev bin86 g++ bison -y

## RT_PREEMPT patch Installation 
#### You can download kernel version from                   : [Linux Kernel Sources](https://mirrors.edge.kernel.org/pub/linux/kernel/) 
#### You can download RT_Preempt version with same kernel   : [RT_PREEMPT Patch Sources](https://mirrors.edge.kernel.org/pub/linux/kernel/projects/rt/)
#### If you want to use different kernel change the 4.4.240 part with your kernel version.
     mkdir sources
     cd sources
#### This part is for different kernel
     wget https://mirrors.edge.kernel.org/pub/linux/kernel/v5.x/linux-5.9.1.tar.xz
     wget https://mirrors.edge.kernel.org/pub/linux/kernel/projects/rt/5.9/patch-5.9.1-rt20.patch.xz
     xz -cd linux-5.9.1.tar.xz | tar xvf -
     cd linux-5.9.1
     xzcat ../patch-5.9.1-rt20.patch.xz | patch -p1
     sudo mv ../linux-5.9.1 /usr/src/ -f
     cd /usr/src/linux-5.9.1
##### Different kernel part finished. You can skip step below if you're using different kernel.
     git clone https://github.com/veysiadn/IgHEtherCATImplementation
     cd IgHEtherCATImplementation
     xz -cd linux-4.4.240.tar.xz | tar xvf -
     cd linux-4.4.240
     xzcat ../patch-4.4.240-rt209.patch.xz | patch -p1
     sudo cp /boot/config-4.4.0-148-generic .config
     sudo mv ../linux-4.4.240 /usr/src/ -f
     cd /usr/src/linux-4.4.240
#### Config file that is referred in here is my kernel file it can vary.Check your boot folder.

     sudo make  menuconfig

## In menu that will be show up we select;
 Processor type and features -> Preemption Model -> Fully Preemptible Kernel (RT).
 
 Alternatively you can configure text file.
 When measuring system latency all kernel debug options should be turned off. They require much overhead and distort the measurement result. Examples for those debug mechanism are:

DEBUG_PREEMPT

Lock Debugging (spinlocks, mutexes, etc. . . )

DEBUG_OBJECTS

…

Some of those debugging mechanisms (like lock debugging) produce a randomized overhead in a range of some micro seconds to several milliseconds depending on the kernel configuration as well as on the compile options (DEBUG_PREEMPT has a low overhead compared to Lock Debugging or DEBUG_OBJECTS).

However, in the first run of a real-time capable Linux kernel it might be advisable to use those debugging mechanisms. This helps to locate fundamental problems.

[Wiki-RT-Linux](https://wiki.linuxfoundation.org/realtime/documentation/howto/applications/preemptrt_setup/)  (RT-Linux-Wiki)

CONFIG_SYSTEM_TRUSTED_KEYS="" , this part should be empty.

For additional kernel configurations check [My-Xenomai-Installation](https://github.com/veysiadn/xenomai-install) (Configurations For Realtime). Just ignore ACPI settings and Xenomai related configurations and apply all other configurations, for better real-time performance. Note that only Fully Preemptible Kernel option is enough, but if you want better performance you can try those options as well.


CONFIG_PREEMPT_RT_FULL

CONFIG_CPU_FREQ=n

CONFIG_CPU_IDLE=n

CONFIG_NO_HZ_FULL=y

CONFIG_RCU_NOCB_CPU=y
 
     sudo -s
     make -j4
     make && make modules && make modules_install && make install
     reboot
 ### If your system doesn't start after building check this thread [Compressing initramfs](https://stackoverflow.com/questions/51669724/install-rt-linux-patch-for-ubuntu)
 ## After reboot to make sure about installation check kernel version again 
     uname -v


## IgH EtherCAT Master Stack Installation

     git clone https://gitlab.com/etherlab.org/ethercat.git ethercat-hg
     cd ethercat-hg
     git checkout stable-1.5
     sudo  ./bootstrap 
     cd
     sudo mv ethercat-hg /usr/local/src/
     cd /usr/local/src/
     sudo ln -s /usr/local/src/ethercat-hg ~/ethercat

Move into the source directory

     cd ~/ethercat

## Configuration part is important for this part refer to

[IgH EtherCAT Library Documentation](https://etherlab.org/download/ethercat/ethercat-1.5.2.pdf)

Chapter 9.2 table 9.1 : configuration options in my case my laptop has

r8169 driver, I have to enable it.You can check the document for detailed instruction.

    sudo ./configure --enable-8139too=no --enable-r8169 --prefix=/opt/etherlab
    sudo -s
    make 
    make modules 
    make install
    make modules_install
-> after succesfull (error free) installation the we'll check HWAddr (Hardware Address, also known as the MAC Address) of the adapter we'd like to use 

(example: eth0) and record it. We'll need to type it in later.


    sudo ifconfig
    
    sudo mkdir /etc/sysconfig/
    
    sudo cp /opt/etherlab/etc/sysconfig/ethercat /etc/sysconfig/
    
    sudo nano /etc/sysconfig/ethercat

-> You need to setup this file as prescribed in the EtherCAT manual. You definitely need to change the values for MASTER0_DEVICE, which need the MAC address of the Ethernet card you've selected, and then the driver you'd like to use for that device.

-> For a development system, "generic" is fine. For a production system, the hope is that you've selected a target machine with a supported network device. We typically used cards supported by the r8169 driver, but check the hardware specs if you’re unsure. If you're using newer kernels type "generic", it's not supported for newer kernels.You can see supported native driver kernel version in Etherlab webpage.

Example:

MASTER0_DEVICE="00:0C:29:09:E0:D7"

DEVICE_MODULES="r8169"


       cd /opt/etherlab
    
-> Copy the initialization script (If this doesn't work, make sure that there isn't a /etc/init.d/ethercat already. If so, remove it), change its ownership properties.

       sudo cp ./etc/init.d/ethercat /etc/init.d/

       sudo chmod a+x /etc/init.d/ethercat

       sudo ln -s /opt/etherlab/bin/ethercat /usr/local/bin/ethercat

       sudo nano /etc/udev/rules.d/99-EtherCAT.rules
       
       sudo udevadm control --reload 
      
## Enter the following contents:

    KERNEL=="EtherCAT[0-9]*", MODE="0664", GROUP="users"

     sudo cp /etc/sysconfig/ethercat /etc
     cd /etc
     sudo mv ethercat ethercat.conf

## Now we can test our installation

     sudo /etc/init.d/ethercat start
     dmesg

 -> If you want to start ethercat from terminal directly without changing directory  you can create symbolic link: 
 
     sudo ln -s /etc/init.d/ethercat /usr/local/bin/ethercatctl
 
 -> And now you can test it.
 
     sudo ethercatctl start  
 
after this command you should see something like this : 

[ 2038.604876] EtherCAT: Master driver 1.5.2 334c34cfd2e5

[ 2038.605018] EtherCAT: 1 master waiting for devices.

[ 2038.968282] ec_r8169 Gigabit Ethernet driver 2.3LK-NAPI loaded

[ 2038.968303] ec_r8169 0000:03:00.0: can't disable ASPM; OS doesn't have ASPM control

[ 2038.977080] EtherCAT: Accepting DC:FE:07:21:A6:75 as main device for master 0.

[ 2038.977099] ec_r8169 0000:03:00.0 ecm0 (uninitialized): RTL8168g/8111g at 0xffffc90002936000, dc:fe:07:21:a6:75, XID 0c000880 IRQ 127

[ 2038.977106] ec_r8169 0000:03:00.0 ecm0 (uninitialized): jumbo features [frames: 9200 bytes, tx checksumming: ko]

[ 2039.042040] EtherCAT 0: Starting EtherCAT-IDLE thread.

 -> if you want to test your program under stress test ;
 
     sudo apt install stress
     stress -v -c 8 -i 10 -d 8
 
-> The EtherLAB EtherCAT master is now running on the system. The next task is to setup the system so that other programs can use the master. You need to add /opt/etherlab/lib to your /etc/ld.so.conf so that the user programs calling it can link to the shared object.

     sudo nano /etc/ld.so.conf

THIS LINE WILL ALREADY EXIST ==> include /etc/ld.so.conf.d/*.conf

Underneath it, add:

/opt/etherlab/lib

So, when you're done, the file will look like the following:

include /etc/ld.so.conf.d/*.conf

/opt/etherlab/lib

Exit, save, then we need to update the system from the configuration file.

     sudo ldconfig

You can see if it got installed by running:

     ldconfig -v | grep libether*


### BONUS : Qt Installation 
```sh
sudo apt-get install  qtcreator qt5-default qt5-doc qt5-doc-html qtbase5-doc-html qtbase5-examples –y 
sudo /sbin/ldconfig -v
```

If you face any problem you can check these threads : 

[EtherLAB Mailing List Implementation ](https://lists.etherlab.org/pipermail/etherlab-dev/2014/000384.html)

[EtherLAB Documentation ](https://etherlab.org/download/ethercat/ethercat-1.5.2.pdf)

