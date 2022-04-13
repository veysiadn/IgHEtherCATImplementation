# IgH EtherCAT Master Installation Guide
This repository contains installation of IgH EtherCAT Master stack. If you need better real-time performance you can install [RT_PREEMPT Patch](https://github.com/veysiadn/RT_PREEMPT_INSTALL), or [Xenomai Patch](https://github.com/veysiadn/xenomai-install). Hope this repository will save time for you.

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

It is important to check the Etherlab documentation for configuration, for this part refer to [IgH EtherCAT Library Documentation](https://etherlab.org/download/ethercat/ethercat-1.5.2.pdf) Chapter 9.2, Table 9.1 : You can check the document for detailed instruction on configuration. If you want to use generic driver, configuration below will work fine for you.

    sudo ./configure --enable-8139too=no --prefix=/opt/etherlab
    sudo -s
    make 
    make modules 
    make install
    make modules_install
    
-> After succesfull (error free) installation, we'll need to check HWAddr (Hardware Address, also known as the MAC Address) of the NIC we'd like to use 
(example: eth0) and record it. We'll need to type it in later. You can check your NIC's MAC address by : 

    sudo ifconfig
  
 -> and now copy MAC address (HWAddr), we will use it in the next step. Note don't copy wirelles adapter's MAC Address, when you type when you type command above there'll be three different sections. Copy the MAC Address of the one starting with letter `e`, not the wireless adapter that starts with `w`. Be careful to choose correct one. `If you don't copy the correct MAC Address your implementation won't work.`
    
    sudo mkdir /etc/sysconfig/
    
    sudo cp /opt/etherlab/etc/sysconfig/ethercat /etc/sysconfig/
    
    sudo nano /etc/sysconfig/ethercat

-> You need to change the values for MASTER0_DEVICE and DEVICE_MODULES, MASTER0_DEVICE value must be the MAC address of the Ethernet card you've selected, and DEVICE_MODULES value must be the driver you'd like to use for that device, in this case it will be generic.

-> For a development system, "generic" is fine. For better real-time performance, native drivers must be used. However not all NIC drivers are supported by IgH.

-> If you want to use native drivers provided by IgH, check your network card interface driver by ;

     lshw -C network | grep driver

if you see your network card driver, compare it with supported NIC drivers from  IgH from here : [IgH EtherCAT Official Page](https://etherlab.org/en/ethercat/hardware.php) (IgH EtherCAT Native Driver Supported Hardware)
. If you don't see your NIC don't worry, you can use generic driver. Besides currently with this kernel only generic driver works. If you're using newer kernels type "generic", native drivers not supported for newer kernels. Once you change you change your ethercat config file parameters should look like below.


Example:
```
    MASTER0_DEVICE="XX:XX:XX:XX:XX:XX"

    DEVICE_MODULES="generic"
```
#### Last Steps : 
```
       cd /opt/etherlab
```    
-> Copy the initialization script (If this doesn't work, make sure that there isn't a /etc/init.d/ethercat already. If so, remove it), change its ownership properties.

       sudo cp ./etc/init.d/ethercat /etc/init.d/

       sudo chmod a+x /etc/init.d/ethercat

       sudo ln -s /opt/etherlab/bin/ethercat /usr/local/bin/ethercat

       sudo nano /etc/udev/rules.d/99-EtherCAT.rules
  ## Enter the following contents:
  ```
    KERNEL=="EtherCAT[0-9]*", MODE="0664", GROUP="users"
 ```
 save and exit, then:
 
     sudo udevadm control --reload 
     sudo cp /etc/sysconfig/ethercat /etc
     cd /etc
     sudo mv ethercat ethercat.conf

## Now we can test our installation

     sudo /etc/init.d/ethercat start
 after running this command you must see something like Starting EtherCAT master done.
 
 -> If you want to start ethercat from terminal directly without changing directory  you can create symbolic link: 
 
     sudo ln -s /etc/init.d/ethercat /usr/local/bin/ethercatctl
 
 -> And now you can test it.
 
     sudo ethercatctl start  
     dmesg
     
after this command you should see something like this, it doesn't have to be same : 

[ 2038.604876] EtherCAT: Master driver 1.5.2 334c34cfd2e5

[ 2038.605018] EtherCAT: 1 master waiting for devices.

[ 2038.968282] ec_r8169 Gigabit Ethernet driver 2.3LK-NAPI loaded

[ 2038.968303] ec_r8169 0000:03:00.0: can't disable ASPM; OS doesn't have ASPM control

[ 2038.977080] EtherCAT: Accepting DC:FE:07:21:A6:75 as main device for master 0.

[ 2038.977099] ec_r8169 0000:03:00.0 ecm0 (uninitialized): RTL8168g/8111g at 0xffffc90002936000, dc:fe:07:21:a6:75, XID 0c000880 IRQ 127

[ 2038.977106] ec_r8169 0000:03:00.0 ecm0 (uninitialized): jumbo features [frames: 9200 bytes, tx checksumming: ko]

[ 2039.042040] EtherCAT 0: Starting EtherCAT-IDLE thread.

 -> The EtherLAB EtherCAT master is now running on the system. The next task is to setup the system so that other programs can use the master. You need to add /opt/etherlab/lib to your /etc/ld.so.conf so that the user programs calling it can link to the shared object.

     sudo nano /etc/ld.so.conf

THIS LINE WILL ALREADY EXIST ==> include /etc/ld.so.conf.d/*.conf

Underneath it, add:

    /opt/etherlab/lib

So, when you're done, the file will look like the following:

    include /etc/ld.so.conf.d/*.conf

    /opt/etherlab/lib

Save and exit, then we need to update the system from the configuration file.

     sudo ldconfig

You can see if it got installed by running:

     ldconfig -v | grep libether*
     
-> if you want to test your program under stress test ;
 
     sudo apt install stress
     stress -v -c 8 -i 10 -d 8
 

### BONUS : Qt Installation 
```sh
sudo apt-get install  qtcreator qt5-default qt5-doc qt5-doc-html qtbase5-doc-html qtbase5-examples –y 
sudo /sbin/ldconfig -v
```

If you face any problem you can check these sources : 

[EtherLAB Mailing List Implementation ](https://lists.etherlab.org/pipermail/etherlab-dev/2014/000384.html)

[EtherLAB Documentation ](https://etherlab.org/download/ethercat/ethercat-1.5.2.pdf)

[Source Code IgH EtherCAT](https://gitlab.com/etherlab.org/ethercat.git) (IgH EtherCAT repo page)
