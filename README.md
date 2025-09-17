# PiHole with PiVPN and Unbound on VPS 
 These are my install notes for creating a virtual private server (VPS; Amazon AWS EC2 free tier) with PiHole, PiVPN (wireguard), and unbound on the VPS to connect to my clients (phone, laptop, family's devices, etc). I’m able to connect to PiVPN with my wireguard profile that I generated (e.g., for my phone), and access my PiHole admin page. Split-tunnelling is used to only route the DNS (vs., all data) for speed and to save bandwidth on the free tier.

## What is this?
* This setup lets you run [PiHole](https://github.com/pi-hole/pi-hole), from anywhere, for free without needing any hardware
* Basically, you'll be setting up PiHole on a virtual private server (VPS), connecting to your virtual PiHole using a VPN called PiVPN.
* This setup forces your devices to use only the DNS provided by the PiVPN connection (i.e., the PiHole; this is called a split-tunnel), not your full data (i.e., full tunnel). This makes it fast/light and keeps it free.
* This setup uses WireGuard to connect to your PiVPN (and PiHole) which makes it fairly easy to add to devices.
* I used a free tier of Amazon Web Services [AWS] but this should work on whatever ones you choose (e.g., Google, etc.)
* You can then keep connected to PiHole from any devices (e.g., laptop, phone, etc.), from anywhere (i.e., not just on your home network)
* It's relatively easy to do yourself, and since it's all done manually (vs., a script) you can learn a bit as you go!
* update: added unbound recursive DNS server for safety/privacy!

# Table of Contents
* [Create a VPS](#create-a-VPS)
* [Install PiHole](#Install-PiHole)
* [Install and setup VPN](#Install-and-setup-your-VPN)
* [Install Unbound](#Install-Unbound)
* [Notes and Troubleshooting](#Notes)

# Create a VPS
## Step 1: Create a free Ubuntu server in AWS


* Create an AWS account
* Go to EC2, click launch instance, select “free tier” and choose Ubuntu (I picked 20.04)
* Manually configure, and click through each step until you get to Security groups, and add the following: Custom UDP, Port Range: 51820, Source: 0.0.0.0/0, and Description: Wireguard
* Download your new keypair and save it to .ssh (on mac, I created a folder called .ssh in my main user folder). I called mine PiVPNHOLE
* In your EC2 terminal, note your PublicDNS (IPv4), it’ll look like: ec2-###.location.com, I call this [your host] below
* Click Elastic IP to create an Elastic IP, then click actions and associate, and associate the Elastic IP to your new server

**Note:** If you are planning to use this as a VPN (no split tunelling), use LightSail. AWS has a £0.12 / gb cost on outbound transfers. This means that if you use 1tb / month, you'll spend £120. If you use Lightsail, the £3.50 tier gets you 1tb / month which saves you £116.50.

# Install PiHole
## Step 2: Use Terminal to connect to your new Ubuntu server

```ssh -i /Users/[your user]/.ssh/PiVPNHOLE.pem ubuntu@[your host]``` 

## Step 3: install PiHole
Installing Pihole on Fedora Server 42
Published: 2025-05-06
Updated: 2025-08-15
I've installed Pihole probably a dozen times at this point. The installation guide that Pihole provides is super helpful, but I've had to do some additional steps to get it working in my environment, specifically Fedora Server 42.

First, I had to change SELinux to permissive so that I could install pihole. To set SELinux to permissive edit the SELinux configuration file with sudo vim /etc/selinux/config. Then modify the SELinux parameter by changing the line SELINUX=enforcing to SELINUX=permissive. Save the changes to the file, then reboot the system for the changes to take effect.

Fedora 42 isn't officially supported yet, so I had to force it with the command sudo curl -sSL https://install.pi-hole.net | sudo PIHOLE_SKIP_OS_CHECK=true bash. This will run through the install script. Once it's completed, I've had to open up some ports allowing http, https, dns, etc:
firewall-cmd --permanent --add-service=http --add-service=https --add-service=dns --add-service=dhcp --add-service=dhcpv6 --add-service=ntp
firewall-cmd --permanent --new-zone=ftl
firewall-cmd --permanent --zone=ftl --add-interface=lo
firewall-cmd --reload

Once the installation is complete, I like to change the randomly generated password to something I'll remember. To do this, run sudo pihole setpassword, enter the new password, then reboot the system.

At this point, you can re-enable SELinux by again editing /etc/selinux/config and changing SELINUX=permissive back to SELINUX=enforcing. Save the file and reboot. Thank you to Oliver for pointing out that we can set SELinux back to enforcing after everything gets set up.

In most cases, this should be the end, but in both a virtual environment and bare metal, I've had pihole fail to start with the error /run/log/pihole/pihole.log: No such file or directory, at least on Fedora 42. To solve this, I had to create the folder and give the proper permissions to pihole.
sudo mkdir -p /run/log/pihole
sudo chown pihole:pihole /run/log/pihole
sudo systemctl restart pihole-FTL

This will fix the problem, but I've noticed that after the server reboots, I've had to run these commands again. To solve this problem, I've had to automate this with the systemd mechanism called tmpfiles.d. Creating/editing the file sudo vim /etc/tmpfiles.d/pihole.conf and then adding the line d /run/log/pihole 0755 pihole pihole -. This automatically creates a folder ("d"), providing the pihole group and owner the 0755 permission, and "-" meaning no age limit (meaning do not delete).

Hooray! Pihole is now making my network be a more sacred place.

[fedora@vps-9deb0f59 ~]$ sudo vi /etc/pihole/pihole.toml change port to be 8010 for webserver

Thanks for reading. Feel free to send comments, questions, or recommendations to hey@chuck.is.
# Install and setup your VPN
## Step 4: install wireguard
```bash
#!/bin/bash

# A simple WireGuard management script for Fedora, inspired by PiVPN.
# Must be run with root privileges.

# --- Configuration ---
WG_CONF="/etc/wireguard/wg0.conf"
CLIENT_CONF_DIR="/etc/wireguard/clients"
SERVER_SUBNET="10.8.0.1/24"
SERVER_PORT="51820"

# --- Colors ---
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[0;33m'
NC='\033[0m' # No Color

# --- Helper Functions ---
function check_root() {
    if [[ "$EUID" -ne 0 ]]; then
        echo -e "${RED}Error: This script must be run as root. Please use 'sudo'.${NC}"
        exit 1
    fi
}

function install_wireguard() {
    if [ -f "$WG_CONF" ]; then
        echo -e "${YELLOW}WireGuard appears to be already installed. Exiting setup.${NC}"
        return
    fi

    echo -e "${GREEN}Starting WireGuard installation for Fedora...${NC}"

    # Install packages
    dnf install wireguard-tools qrencode -y

    # Create directories and set permissions
    mkdir -p /etc/wireguard
    chmod 700 /etc/wireguard
    mkdir -p "$CLIENT_CONF_DIR"
    chmod 700 "$CLIENT_CONF_DIR"

    # Generate server keys
    wg genkey | tee /etc/wireguard/server_private.key | wg pubkey > /etc/wireguard/server_public.key
    chmod 600 /etc/wireguard/server_private.key

    SERVER_PRIV_KEY=$(cat /etc/wireguard/server_private.key)

    # Configure firewall with firewalld
    echo -e "${YELLOW}Configuring firewall...${NC}"
    firewall-cmd --permanent --zone=public --add-port=${SERVER_PORT}/udp
    firewall-cmd --permanent --zone=public --add-masquerade
    firewall-cmd --reload

    # Enable IP forwarding
    echo "net.ipv4.ip_forward=1" > /etc/sysctl.d/99-wireguard.conf
    sysctl -p /etc/sysctl.d/99-wireguard.conf

    # Create server config file
    echo -e "${YELLOW}Creating server configuration file...${NC}"
    cat > "$WG_CONF" << EOF
[Interface]
Address = ${SERVER_SUBNET}
ListenPort = ${SERVER_PORT}
PrivateKey = ${SERVER_PRIV_KEY}
PostUp = firewall-cmd --zone=public --add-port=${SERVER_PORT}/udp; firewall-cmd --zone=public --add-masquerade
PostDown = firewall-cmd --zone=public --remove-port=${SERVER_PORT}/udp; firewall-cmd --zone=public --remove-masquerade

# --- CLIENTS ---
EOF

    # Start and enable the service
    systemctl enable --now wg-quick@wg0

    echo -e "${GREEN}WireGuard has been installed and started successfully!${NC}"
}

function add_client() {
    if [ ! -f "$WG_CONF" ]; then
        echo -e "${RED}WireGuard is not installed. Please run the installer first.${NC}"
        return
    fi

    read -p "Enter a name for the new client: " CLIENT_NAME
    # Sanitize client name
    CLIENT_NAME=$(echo "$CLIENT_NAME" | tr -dc '[:alnum:]_-')

    if grep -q "# Client: ${CLIENT_NAME}$" "$WG_CONF"; then
        echo -e "${RED}Client '${CLIENT_NAME}' already exists.${NC}"
        return
    fi

    # Find the next available IP address
    LAST_IP=$(grep -o '[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}' "$WG_CONF" | tail -1)
    LAST_OCTET=$(echo "$LAST_IP" | cut -d . -f 4)
    NEXT_OCTET=$((LAST_OCTET + 1))
    CLIENT_IP="10.8.0.${NEXT_OCTET}/32"

    # Generate client keys
    CLIENT_PRIV_KEY=$(wg genkey)
    CLIENT_PUB_KEY=$(echo "$CLIENT_PRIV_KEY" | wg pubkey)
    CLIENT_PRESHARED_KEY=$(wg genpsk)
    
    # Force IPv4 for the public endpoint IP
    SERVER_ENDPOINT_IP=$(curl -4 -s ifconfig.me)
    if [ -z "$SERVER_ENDPOINT_IP" ]; then
        echo -e "${RED}Could not determine server's public IPv4 address. Please check connectivity.${NC}"
        return 1
    fi

    SERVER_PUB_KEY=$(cat /etc/wireguard/server_public.key)
    # Get the server's VPN IP to use as the DNS server
    SERVER_VPN_IP=$(echo ${SERVER_SUBNET} | cut -d'/' -f1)

    # Add peer to server config
    cat >> "$WG_CONF" << EOF

# Client: ${CLIENT_NAME}
[Peer]
PublicKey = ${CLIENT_PUB_KEY}
PresharedKey = ${CLIENT_PRESHARED_KEY}
AllowedIPs = ${CLIENT_IP}
EOF

    # Restart service to apply changes
    systemctl restart wg-quick@wg0

    # Create a split-tunnel client config for Pi-hole DNS
    CLIENT_CONF_FILE="${CLIENT_CONF_DIR}/${CLIENT_NAME}.conf"

    cat > "$CLIENT_CONF_FILE" << EOF
[Interface]
PrivateKey = ${CLIENT_PRIV_KEY}
Address = $(echo ${CLIENT_IP} | cut -d'/' -f1)/24
DNS = ${SERVER_VPN_IP}

[Peer]
PublicKey = ${SERVER_PUB_KEY}
PresharedKey = ${CLIENT_PRESHARED_KEY}
Endpoint = ${SERVER_ENDPOINT_IP}:${SERVER_PORT}
AllowedIPs = ${SERVER_VPN_IP}/32
PersistentKeepalive = 25
EOF
    # Set secure permissions for the client config
    chmod 600 "$CLIENT_CONF_FILE"

    echo -e "${GREEN}Split-tunnel client '${CLIENT_NAME}' added successfully!${NC}"
    echo -e "Config file is located at: ${YELLOW}${CLIENT_CONF_FILE}${NC}"
    echo -e "This config will ONLY route DNS traffic through the VPN."
}

function remove_client() {
    if [ ! -f "$WG_CONF" ]; then
        echo -e "${RED}WireGuard is not installed.${NC}"
        return
    fi
    
    list_clients
    if [ $? -ne 0 ]; then return; fi

    read -p "Enter the name of the client to remove: " CLIENT_NAME
    CLIENT_NAME=$(echo "$CLIENT_NAME" | tr -dc '[:alnum:]_-')

    if ! grep -q "# Client: ${CLIENT_NAME}$" "$WG_CONF"; then
        echo -e "${RED}Client '${CLIENT_NAME}' not found.${NC}"
        return
    fi

    # Remove the peer block using sed
    sed -i "/# Client: ${CLIENT_NAME}$/,/^\s*$/d" "$WG_CONF"

    # Path to client config file
    CLIENT_CONF_FILE="${CLIENT_CONF_DIR}/${CLIENT_NAME}.conf"
    if [ -f "$CLIENT_CONF_FILE" ]; then
        rm "$CLIENT_CONF_FILE"
    fi

    systemctl restart wg-quick@wg0
    echo -e "${GREEN}Client '${CLIENT_NAME}' has been removed.${NC}"
}

function list_clients() {
    if [ ! -f "$WG_CONF" ]; then
        echo -e "${RED}WireGuard is not installed.${NC}"
        return 1
    fi

    CLIENTS=$(grep '# Client:' "$WG_CONF" | cut -d ' ' -f 3)
    if [ -z "$CLIENTS" ]; then
        echo -e "${YELLOW}No clients have been added yet.${NC}"
        return 1
    fi
    
    echo -e "${GREEN}--- Existing Clients ---${NC}"
    grep -E "^# Client:|^AllowedIPs =" "$WG_CONF" | paste - - | awk '{printf "- %-20s (IP: %s)\n", $3, $6}'
    echo ""
    return 0
}

function show_client_details() {
    if [ ! -d "$CLIENT_CONF_DIR" ] || [ -z "$(ls -A $CLIENT_CONF_DIR)" ]; then
        echo -e "${RED}No client configuration files found in ${CLIENT_CONF_DIR}.${NC}"
        return
    fi

    echo "Available client configs:"
    ls -1 "$CLIENT_CONF_DIR" | sed 's/\.conf$//'
    echo ""
    read -p "Enter the name of the client to show details for: " CLIENT_NAME
    
    CLIENT_CONF_FILE="${CLIENT_CONF_DIR}/${CLIENT_NAME}.conf"
    if [ ! -f "$CLIENT_CONF_FILE" ]; then
        echo -e "${RED}Config file for '${CLIENT_NAME}' not found.${NC}"
        return
    fi

    # MODIFIED: Add a sub-menu to choose what to display
    echo -e "\nWhat would you like to display for client ${YELLOW}${CLIENT_NAME}${NC}?"
    echo " 1) QR Code"
    echo " 2) Config File Text"
    read -p "Please select an option [1-2]: " display_choice

    case $display_choice in
        1)
            echo -e "\n${GREEN}--- QR Code for ${CLIENT_NAME} ---${NC}"
            qrencode -t ansiutf8 < "$CLIENT_CONF_FILE"
            ;;
        2)
            echo -e "\n${GREEN}--- Config File for ${CLIENT_NAME} ---${NC}"
            cat "$CLIENT_CONF_FILE"
            ;;
        *)
            echo -e "${RED}Invalid option.${NC}"
            ;;
    esac
}

function uninstall_wireguard() {
    echo -e "${RED}This will stop and disable WireGuard, remove all configuration files, and close the firewall port.${NC}"
    read -p "Are you sure you want to uninstall WireGuard? [y/N]: " CONFIRM
    if [[ "$CONFIRM" != "y" && "$CONFIRM" != "Y" ]]; then
        echo "Uninstall cancelled."
        return
    fi

    systemctl stop wg-quick@wg0
    systemctl disable wg-quick@wg0

    firewall-cmd --permanent --zone=public --remove-port=${SERVER_PORT}/udp
    firewall-cmd --permanent --zone=public --remove-masquerade
    firewall-cmd --reload

    rm -rf /etc/wireguard
    rm -f /etc/sysctl.d/99-wireguard.conf

    echo -e "${GREEN}WireGuard has been uninstalled.${NC}"
    read -p "Do you want to remove the wireguard-tools and qrencode packages? [y/N]: " REMOVE_PACKAGES
    if [[ "$REMOVE_PACKAGES" == "y" || "$REMOVE_PACKAGES" == "Y" ]]; then
        dnf remove wireguard-tools qrencode -y
        echo -e "${GREEN}Packages have been removed.${NC}"
    fi
}


# --- Main Menu ---
function show_menu() {
    clear
    echo "======================================="
    echo "   WireGuard Management Script"
    echo "======================================="
    echo "1. Install WireGuard"
    echo "2. Add a New Client (for Pi-hole DNS)"
    echo "3. Remove a Client"
    echo "4. List All Clients"
    # MODIFIED: Updated menu text for clarity
    echo "5. Show Client Details (QR/Config)"
    echo "6. Uninstall WireGuard"
    echo "7. Exit"
    echo "======================================="
    read -p "Please select an option [1-7]: " menu_choice
    
    case $menu_choice in
        1) install_wireguard ;;
        2) add_client ;;
        3) remove_client ;;
        4) list_clients ;;
        # MODIFIED: Call the renamed function
        5) show_client_details ;;
        6) uninstall_wireguard ;;
        7) exit 0 ;;
        *) echo -e "${RED}Invalid option, please try again.${NC}" ;;
    esac
    
    echo ""
    read -p "Press Enter to return to the menu..."
}

# --- Script Execution ---
check_root
while true; do
    show_menu
done
```

## Step 5: create your user profiles

```pivpn add -n [config name]```
*  where [config name] is a unique name for each of your devices (e.g., mphone, mlaptop). You can repeat this step for as many devices that you want to connect to your Pi-hole.

## Step 6: add split-tunnel connection to your config
```sudo nano /etc/wireguard/configs/[config name].conf```
* Change AllowedIPs from "0.0.0.0/0, ::0" to "[PiHole IP address]/32, [DNS IP]/32". 
* The [DNS IP] is listed in [interface] and by default 10.6.0.1/32.
* Note spit-tunnelling only routes the DNS (i.e., PiHole ad-blocking) vs., all of your data through your VPN which will save bandwidth to keep you on the free tier.
* You'll need to repeat this for each [config name] you created in step 5.

## Step 7: display your config QR code to connect your mobile device
```pivpn -qr [config name, e.g., mphone]```

## Step 8: download your configs for laptops/desktops

```scp -i /Users/[Your Name]/.ssh/PiVPNHOLE.pem ubuntu@[your host]:~/configs/[config name, e.g., mlaptop] [destination on your computer, e.g., ~/Documents/wireguard]```

Here's an example because this can be confusing:

```scp -i /Users/example/.ssh/PiVPNHOLE.pem ubuntu@ec2-1-2-3-4.location.compute.amazonaws.com:~/configs/ /Users/example/wireguard```

This will download all of your config files to a folder on your computer called wireguard

## Step 9: Install Wireguard on your device

* Scan your QR code for your mobile devices, and/or install the downloaded configs for your laptop/desktop/other devices, turn them on.
* I set them to "on-demand" meaning it'll always be on
* Go check out your PiHole at the address you saved in Step 2!

# Install Unbound
## Step 9: Install Unbound

* Connect back to your Ubuntu VPS in terminal
* Follow this [guide](https://docs.pi-hole.net/guides/unbound/), it's pretty straight forward
* One thing to note, when you get to Configure Unbound instruction, it'll say "/etc/unbound/unbound.conf.d/pi-hole.conf", you actually need to add "sudo nano" to the start of that code in your Terminal to be able to create and paste in the configuration (or else you'll just get an error). Then just copy/paste in the text from the guide and hit save/exit.
```sudo nano /etc/unbound/unbound.conf.d/pi-hole.conf```

# Notes
## Troubleshooting
* Before being able to remotely log in, I had to run the command ```chmod 600 /Users/[your name]/.ssh/PiVPNHOLE.pem```
* After clicking "generate keys" in PiVpn, you may get `/tmp/setupVars.conf permission denied`. I solved this by deleting that file.
* You may need to run the piVpn script as sudo. Run with `curl -L https://install.pivpn.io |  sudo bash`

## Credits
* Thanks to @SuspectTyrannosaurus for fixing creating user profiles.
* Special thanks to @DasJason for inspiring this project, troubleshooting, and various code tips/tricks.
* Thanks to Thank you to @afruitpie for helping me figure out split-tunnelling and how to download the configuration files. 
* Thanks also to @kryten2k35 for making sure this method of PiHole isn't exposed to the entire world (i.e., double checking port 53 is closed so the DNS isn't public).
* Thanks to [@bee-san](https://github.com/bee-san) for troubleshooting tips and solutions

<h2>Support this project</h2>

If you found this guide helpful, please consider buying me a coffee by clicking the link below. I'll do my best to keep this guide up to date and as user-friendly as possible. Thank you and take good care!

[![Donate](https://img.shields.io/badge/Donate-PayPal-green.svg)](https://www.paypal.com/cgi-bin/webscr?cmd=_donations&business=R4QX73RWYB3ZA)



