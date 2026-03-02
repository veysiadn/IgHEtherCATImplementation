# IgH EtherCAT Master Installation Guide

This repository contains a step-by-step guide for installing the IgH EtherCAT Master stack on Linux. If you need better real-time performance, you can also install the [RT_PREEMPT Patch](https://github.com/veysiadn/RT_PREEMPT_INSTALL) or the [Xenomai Patch](https://github.com/veysiadn/xenomai-install).

## Compatibility & Version Notes

| Component | Supported Versions | Notes |
|---|---|---|
| IgH EtherCAT Master | stable-1.5 (≈ v1.5.2) | Tested with kernels up to ~5.x |
| Linux Kernel | 4.x - 5.x | Native drivers **not** supported on 5.10+ / 6.x kernels; use the generic driver instead |
| Ubuntu | 18.04, 20.04, 22.04 | Confirmed to work; Ubuntu 22.04+ requires generic driver |
| Debian | 9 (Stretch), 10 (Buster), 11 (Bullseye) | Should work with the same notes as Ubuntu |

> **Note:** On newer kernels (5.10 and above, including all 6.x kernels), the IgH native NIC drivers fail to compile. Always use `DEVICE_MODULES="generic"` on modern distributions.

## Prerequisites

Install the required build tools and kernel headers before proceeding:

```bash
sudo apt update
sudo apt install -y build-essential autoconf automake libtool \
    linux-headers-$(uname -r) net-tools git
```

## IgH EtherCAT Master Stack Installation

### Step 1 – Clone and prepare the source

```bash
git clone https://gitlab.com/etherlab.org/ethercat.git ethercat-hg
cd ethercat-hg
git checkout stable-1.5
sudo ./bootstrap
cd
sudo mv ethercat-hg /usr/local/src/
sudo ln -s /usr/local/src/ethercat-hg ~/ethercat
```

### Step 2 – Configure and build

Move into the source directory:

```bash
cd ~/ethercat
```

For detailed configuration options, refer to [IgH EtherCAT Library Documentation](https://etherlab.org/download/ethercat/ethercat-1.5.2.pdf), Chapter 9.2, Table 9.1. The command below uses the generic driver and installs to `/opt/etherlab`, which works for most setups:

```bash
sudo ./configure --enable-8139too=no --prefix=/opt/etherlab
sudo -s
make
make modules
make install
make modules_install
```

### Step 3 – Find the MAC address of your Ethernet card

After a successful, error-free build, identify the MAC address (HWAddr) of the NIC you want to use for EtherCAT:

```bash
sudo ifconfig
```

Copy the MAC address of your wired Ethernet interface — the one whose name starts with `e` (e.g., `eth0`, `enp3s0`). **Do not copy the wireless adapter's MAC address** (interfaces starting with `w`). Using the wrong MAC address will prevent the master from working.

### Step 4 – Configure the EtherCAT master

```bash
sudo mkdir -p /etc/sysconfig/
sudo cp /opt/etherlab/etc/sysconfig/ethercat /etc/sysconfig/
sudo nano /etc/sysconfig/ethercat
```

Set the following two values in the file:
- `MASTER0_DEVICE` — the MAC address of the Ethernet card you selected above
- `DEVICE_MODULES` — the driver to use (`generic` is recommended; see note below)

**Choosing a driver:**

- For development or on kernels newer than 5.10, always use `"generic"`.
- For better real-time performance on older kernels, you can try a native driver. Check your NIC driver with:

```bash
lshw -C network | grep driver
```

Then compare it against the [IgH EtherCAT Supported Hardware list](https://etherlab.org/en/ethercat/hardware.php). If your driver is not listed, fall back to `"generic"`.

When finished, the relevant lines in the config file should look like this:

```
MASTER0_DEVICE="XX:XX:XX:XX:XX:XX"
DEVICE_MODULES="generic"
```

### Step 5 – Final setup

```bash
cd /opt/etherlab
```

Copy the initialization script. If the command fails with a file-already-exists error, remove `/etc/init.d/ethercat` first and retry.

```bash
sudo cp ./etc/init.d/ethercat /etc/init.d/
sudo chmod a+x /etc/init.d/ethercat
sudo ln -s /opt/etherlab/bin/ethercat /usr/local/bin/ethercat
```

Create the udev rule so user applications can access the EtherCAT device:

```bash
sudo nano /etc/udev/rules.d/99-EtherCAT.rules
```

Add the following line, then save and exit:

```
KERNEL=="EtherCAT[0-9]*", MODE="0664", GROUP="users"
```

Reload udev and copy the config file to `/etc`:

```bash
sudo udevadm control --reload
sudo cp /etc/sysconfig/ethercat /etc
cd /etc
sudo mv ethercat ethercat.conf
```

## Testing the Installation

Start the EtherCAT master:

```bash
sudo /etc/init.d/ethercat start
```

You should see output similar to: `Starting EtherCAT master done.`

Optionally, create a symbolic link so you can start/stop the master from any directory:

```bash
sudo ln -s /etc/init.d/ethercat /usr/local/bin/ethercatctl
```

Then test with:

```bash
sudo ethercatctl start
dmesg
```

The `dmesg` output should contain lines like the following (exact values will differ):

```
[ 2038.604876] EtherCAT: Master driver 1.5.2 334c34cfd2e5
[ 2038.605018] EtherCAT: 1 master waiting for devices.
[ 2038.968282] ec_r8169 Gigabit Ethernet driver 2.3LK-NAPI loaded
[ 2038.977080] EtherCAT: Accepting DC:FE:07:21:A6:75 as main device for master 0.
[ 2039.042040] EtherCAT 0: Starting EtherCAT-IDLE thread.
```

## Linking the EtherCAT Library

The EtherLAB EtherCAT master is now running. To allow user-space programs to link against it, add `/opt/etherlab/lib` to your dynamic linker configuration:

```bash
sudo nano /etc/ld.so.conf
```

The file already contains the line `include /etc/ld.so.conf.d/*.conf`. Add `/opt/etherlab/lib` on a new line beneath it:

```
include /etc/ld.so.conf.d/*.conf
/opt/etherlab/lib
```

Save and exit, then update the linker cache:

```bash
sudo ldconfig
```

Verify the library is found:

```bash
ldconfig -v | grep libether*
```

## Optional: Stress Testing

To test your program under a CPU/IO stress load:

```bash
sudo apt install stress
stress -v -c 8 -i 10 -d 8
```

## BONUS: Qt Installation

> **Compatibility note:** The `qt5-default` package was removed from Ubuntu starting with version 21.04 and from Debian 11+. Use `qtbase5-dev` instead.

**Ubuntu 20.04 and older / Debian 10 and older:**
```sh
sudo apt-get install -y qtcreator qt5-default qt5-doc qt5-doc-html qtbase5-doc-html qtbase5-examples
sudo /sbin/ldconfig -v
```

**Ubuntu 21.04+ / Debian 11+:**
```sh
sudo apt-get install -y qtcreator qtbase5-dev qt5-doc qt5-doc-html qtbase5-doc-html qtbase5-examples
sudo /sbin/ldconfig -v
```

## Troubleshooting & References

If you encounter issues, the following resources may help:

- [EtherLAB Mailing List (implementation example)](https://lists.etherlab.org/pipermail/etherlab-dev/2014/000384.html)
- [EtherLAB Official Documentation](https://gitlab.com/etherlab.org/ethercat/-/jobs/8139472655/artifacts/raw/pdf/ethercat_doc.pdf)
- [IgH EtherCAT Source Repository](https://gitlab.com/etherlab.org/ethercat.git)
