# Making Huawei/Brovi E3372 4G USB Modems Work with GLinet Routers

## The Problem

Huawei E3372 USB modems (and rebranded versions like Brovi E3372) operate in "HiLink" mode, which causes connectivity issues when used with GLinet travel routers. When you plug in the modem, the router may show it as connected but with no internet access.

This happens because:
1. The modem creates its own network (usually 192.168.8.x)
2. The router doesn't properly initialize the USB interface
3. Network routing isn't set up correctly after reboot

## The Solution

This guide provides a reliable fix that works even after router reboots:

### 1. Access Your Router via SSH

Connect to your GLinet router's WiFi and SSH into it:
```
ssh root@192.168.8.1
```
(Use your router's IP address and password)

### 2. Change Router's LAN IP Address

If your router uses 192.168.8.x (same as the modem):
1. Go to router admin panel → Network → LAN
2. Change the router's IP from 192.168.8.1 to 192.168.9.1
3. Save and wait for router to reboot
4. Reconnect to WiFi and SSH to the new IP: `ssh root@192.168.9.1`

### 3. Install Required Packages

```
opkg update
opkg install kmod-usb-net kmod-usb-net-rndis kmod-usb-net-cdc-ether kmod-usb-net-huawei-cdc-ncm usb-modeswitch
```

### 4. Create Automatic Configuration Script

Create a script that will run whenever the modem is connected:

```
cat > /etc/hotplug.d/usb/10-huawei-modem << 'EOF'
#!/bin/sh

# Only run script on add action for the specific USB device
if [ "$ACTION" = "add" ] && [ "$PRODUCT" = "12d1/155e/ffff" ]; then
  logger -t modem "Huawei modem connected, starting initialization..."
  
  # Wait for device initialization
  sleep 15
  
  # Set up the usb0 interface
  if ifconfig usb0 2>/dev/null; then
    logger -t modem "Setting up usb0 interface"
    ifconfig usb0 192.168.8.100 netmask 255.255.255.0
    route add default gw 192.168.8.1 usb0
    
    # Add DNS settings
    echo "nameserver 8.8.8.8" > /tmp/resolv.conf.auto
    echo "nameserver 8.8.4.4" >> /tmp/resolv.conf.auto
    
    logger -t modem "Initialization complete"
  else
    logger -t modem "usb0 interface not found"
  fi
fi
EOF
```

⚠️ **Important**: The script above is configured for Huawei modems with ID `12d1/155e`. If your modem has a different ID, you'll need to modify it.

### 5. Make the Script Executable

```
chmod +x /etc/hotplug.d/usb/10-huawei-modem
```

### 6. Find Your Modem's ID (If Needed)

If the default script doesn't work, you need to find your specific modem's ID:

1. Install USB utilities:
   ```
   opkg update && opkg install usbutils
   ```

2. Connect your modem and check its ID:
   ```
   lsusb
   ```

3. Look for your modem in the output (something like `Bus 001 Device 006: ID 12d1:155e Mobile Mobile`)

4. Note the ID (in this example, `12d1:155e`)

5. Edit the script to replace `12d1/155e/ffff` with your modem's ID (format: replace colon with slash and add `/ffff` at the end)

### 7. Reboot and Test

```
reboot
```

After your router reboots, connect the modem and wait about a minute for it to initialize.

### 8. Verify It's Working

```
logread | grep modem
```

You should see messages about the modem initialization. Check internet connectivity by opening a website or running:
```
ping 8.8.8.8
```

## Compatible Hardware

### Tested Router Models:
- GLinet AX1800

### Tested Modem Models:
- Huawei E3372 (ID: 12d1:155e)
- Brovi E3372-32S

## Troubleshooting

If internet still doesn't work:

1. **Check if the modem is detected**:
   ```
   lsusb
   ```

2. **Verify interface creation**:
   ```
   ip a | grep usb
   ```

3. **Check routing table**:
   ```
   route -n
   ```
   (Should show a default route through 192.168.8.1)

4. **Check modem configuration**:
   - Ensure SIM card is properly inserted
   - Verify the modem works when connected directly to a computer
   - Check if cellular network has good signal in your area

## How This Solution Works

1. The script detects when your specific USB modem is connected
2. It waits for the device to initialize
3. It configures the USB interface with a static IP (192.168.8.100)
4. It adds a route that sends internet traffic through the modem
5. It configures DNS servers for name resolution

This approach works around the "HiLink" mode of the modem without requiring firmware changes.

## Search Keywords
Huawei E3372 GLinet, Brovi USB modem router, E3372h-153 fix, HiLink mode router compatibility, 4G dongle no internet, OpenWRT USB modem setup, GLinet travel router 4G internet connection
