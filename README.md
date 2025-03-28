# Making Huawei/Brovi E3372 4G USB Dongles Work with GLinet Travel Routers

## Introduction

This guide provides a solution for connecting Huawei E3372 and Brovi E3372-32S 4G USB modems to GLinet travel routers (AX1800, MT300N, AR750S, etc.). It addresses the common "subnet conflict" error and network connectivity issues caused by HiLink mode operation, without requiring firmware flashing or hardware modification.

*This guide was created based on information from the following helpful resources:*
- [Zedt.eu - Making the Brovi E3372-325 4G/LTE modem work on OpenWRT 22.03](https://zedt.eu/tech/linux/making-the-brovi-e3372-325-4g-lte-modem-work-on-openwrt-22-03/)
- [GLinet Forums - Will Huawei E3372 work?](https://forum.gl-inet.com/t/will-huawei-e3372-work/24296)
- [0xf8.org - Flashing a Huawei E3372h 4G LTE stick from Hilink to Stick mode](https://www.0xf8.org/2017/01/flashing-a-huawei-e3372h-4g-lte-stick-from-hilink-to-stick-mode/)

## Understanding the Problem

The Brovi E3372-32S (similar to Huawei E3372h series) operates in "HiLink" mode, which causes compatibility issues when connected to routers like the GLinet AX1800. The main problems are:

1. **HiLink Mode Operation**: In HiLink mode, the dongle functions as its own router with:
   - Its own subnet (typically 192.168.8.x)
   - Built-in DHCP server
   - Web interface at 192.168.8.1
   - Network Address Translation (NAT)

2. **Router-within-Router Issues**: When connected to a router, this creates a "double NAT" situation that can cause:
   - Network addressing conflicts
   - Routing problems
   - Connection failures

There are two main approaches to solving this problem:
- **Solution 1**: Work around the HiLink mode by resolving subnet conflicts (simpler, non-invasive)
- **Solution 2**: Flash the dongle to "Stick" mode (more complex, requires hardware manipulation)

This guide focuses on Solution 1, which worked successfully for my GLinet AX1800 travel router.

## Solution 1: Working Around HiLink Mode (Recommended)

### Required Equipment
- Brovi E3372-32S 4G USB dongle (or similar Huawei E3372h models in HiLink mode)
- GLinet AX1800 travel router (should work for other GLinet models)
- Computer to access the router administration interface

### Step-by-Step Instructions

1. **Access the Router Admin Interface**
   - Connect to your GLinet router via Wi-Fi or Ethernet
   - Open a web browser and navigate to 192.168.8.1 (default router IP)
   - Log in with your admin credentials

2. **Change Router LAN Subnet to Avoid Conflict**
   - Navigate to "Network" → "LAN"
   - Change the LAN IP address from 192.168.8.1 to 192.168.9.1
   - Keep subnet mask as 255.255.255.0
   - Save settings and allow the router to reboot

3. **Reconnect to Router**
   - Connect to the router's Wi-Fi again if disconnected
   - Access the admin interface at the new IP: 192.168.9.1

4. **Connect USB Dongle**
   - Insert the Brovi E3372-32S USB dongle into the router's USB port
   - Wait for the router to detect the dongle (usually takes 30-60 seconds)

5. **Configure Internet Connection**
   - Navigate to "Internet" → "Connect to Internet"
   - You should see the USB modem as an available connection option
   - Select it and choose "DHCP" as the connection method
   - Apply settings

### How This Solution Works

Instead of changing the dongle's firmware or operation mode, we're working around its behavior:
- The Brovi dongle continues to use its built-in 192.168.8.x subnet
- The GLinet router now uses 192.168.9.x for its LAN
- This prevents the subnet conflict and allows proper routing between networks
- The router can now properly route traffic from its LAN to the internet via the dongle

This approach maintains all the benefits of HiLink mode (simplified setup, built-in web UI) while avoiding addressing conflicts.

## Solution 2: Advanced Setup (Optional)

If the basic solution doesn't work, you may need to install additional packages to help with USB mode switching. SSH into the router and run:

```
opkg update && opkg install usb-modeswitch kmod-usb-net-rndis kmod-usb-net-cdc-ether
```

Create necessary configuration files:

```
vi /etc/hotplug.d/usb/10-brovi
```

With content:
```
EXPECTED_IP="192.168.8."
if ! ls /tmp/brovi.lock ; then
  if [ "${PRODUCT}" = "3566/2001/ffff" ] ; then
    logger -t Brovi Script started.
    if [ "${ACTION}" = "add" ] ; then
      logger -t Brovi +++ Detected INTERFACE: ${INTERFACE} +++
      if [ "${INTERFACE}" = "224/1/3" ] ; then
          logger -t Brovi Interfaces changed - modeswitching...
          touch /tmp/brovi.lock
          logger -t Brovi Step 1...
          usbmode -s -c /etc/brovi.json
          sleep 10
          if ! /sbin/ip a show | /bin/grep -q ${EXPECTED_IP} ; then
            logger -t Brovi Step 2...
            usbmode -s -c /etc/brovi.json
          else
            logger -t Brovi Skipping step 2 as interface is already connected...
          fi
        rm /tmp/brovi.lock
      fi
    fi
  fi
  logger -t Brovi Script completed.
fi
exit 0
```

And create:
```
vi /etc/brovi.json
```

With content:
```
{
        "messages" : [ ],
        "devices" : {
                "3566:2001": {
                        "*": {
                                "t_class": 224,
                                "mode": "Huawei",
                                "msg": [  ]
                        }
                }
        }
}
```

Make the script executable:
```
chmod +x /etc/hotplug.d/usb/10-brovi
```

This script helps with mode detection and switching for specific variants of these modems, but most users won't need this approach.

## Solution 3: Flashing to Stick Mode (For Advanced Users)

As a last resort, you can convert the dongle from HiLink to Stick mode. This requires:
- Linux environment (can be a virtual machine)
- Physical modification of the dongle (using the "needle method")
- Flashing new firmware

This approach is beyond the scope of this guide but is documented at [0xf8.org](https://www.0xf8.org/2017/01/flashing-a-huawei-e3372h-4g-lte-stick-from-hilink-to-stick-mode/).

## Rollback Procedure

If you need to revert changes from Solution 1:

1. **Access Router at New IP**
   - Connect to the router's Wi-Fi
   - Open a web browser and navigate to 192.168.9.1
   - Log in with your admin credentials

2. **Restore Original LAN IP**
   - Navigate to "Network" → "LAN"
   - Change the LAN IP address back from 192.168.9.1 to 192.168.8.1
   - Keep subnet mask as 255.255.255.0
   - Save settings and allow the router to reboot

3. **Remove Advanced Configuration (if implemented)**
   - SSH into the router (now at 192.168.8.1)
   - Remove the configuration files:
     ```
     rm /etc/hotplug.d/usb/10-brovi
     rm /etc/brovi.json
     ```

4. **Uninstall Packages (if installed)**
   - Run the following to remove installed packages:
     ```
     opkg remove usb-modeswitch kmod-usb-net-rndis kmod-usb-net-cdc-ether
     ```

5. **Reboot Router**
   - Either reboot through the admin interface or run `reboot` via SSH

## Troubleshooting

1. **If Dongle Isn't Detected**
   - Disconnect and reconnect the USB dongle
   - Try a different USB port if available
   - Ensure the dongle has proper power (some routers have USB ports with limited power)
   - If you need to check USB devices but `lsusb` command is not found, install USB utilities:
     ```
     opkg update && opkg install usbutils
     ```
   - After installing, run `lsusb` to see all connected USB devices and identify your dongle

2. **If Internet Still Doesn't Work**
   - Check if the SIM card in the dongle is active and has data
   - Verify the dongle works when connected directly to a computer
   - Check router logs via SSH with `logread`

3. **If Subnet Conflict Persists**
   - Try using a different subnet (e.g., 192.168.10.x)
   - Make sure to reconnect to the new IP after changing

## Technical Background

The Brovi/Huawei E3372 dongles come in two variants:
- **HiLink mode** (firmware version 22.x): Acts as a router with its own subnet, DHCP server, and web interface at 192.168.8.1. Model numbers typically have the format E3372h-xxx.
- **Stick mode** (firmware version 21.x): Acts as a regular USB modem with AT commands. Model numbers typically have the format E3372s-xxx.

The Brovi E3372-32S appears to be a rebranded or OEM version of the Huawei E3372 series. While it may not carry Huawei branding, it operates in the same way as Huawei's HiLink devices. The vendor/device ID is typically 3566:2001 (may appear as Mobile Mobile in lsusb), which differs from standard Huawei IDs.

These devices support LTE Cat 4 with download speeds up to 150 Mbps and upload speeds up to 50 Mbps, with fallback to HSPA+/UMTS/EDGE when necessary.

## Related Search Terms and Compatible Devices

### Search Terms
- Huawei E3372 GLinet setup
- Brovi E3372 USB modem router
- E3372h-153 subnet conflict
- 4G dongle double NAT problem
- HiLink mode router compatibility
- OpenWRT USB modem setup
- GLinet travel router 4G setup
- Router within router fix
- USB modem 192.168.8.1 conflict

### Compatible GLinet Router Models
- GLinet AX1800
- GLinet MT300N
- GLinet AR750S
- GLinet Slate
- GLinet Mango
- GLinet Beryl
- Other OpenWRT-based GLinet models

### Compatible USB Modem Models
- Huawei E3372h-153
- Huawei E3372h-320
- Huawei E3372h-325
- Brovi E3372-32S
- Megafon M150-2
- Telekom Speedstick V
- Other E3372 variants in HiLink mode
