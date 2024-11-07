![AIT Logo](https://www.ait.co.th/wp-content/uploads/2023/03/logo.png)

# OpenVPN Installation & Configuration Guide for Mobile Devices

## Client Configuration Guide

<style>
{% include styles.css %}
</style>

<div class="container">
    <div class="sidebar">
        <h2>Table of Contents</h2>
        <ul>
            <li><a href="INSTALL">Server Installation Guide</a></li>
            <li><a href="CONFIG">Server Configuration Guide</a></li>
            <li><a href="CLIENT">Client Configuration Guide</a></li>
            <li><a href="OPERATIONS">Operations Guide</a></li>
            <li><a href="ADVANCED">Advanced Topics Guide</a></li>
            <li><a href="FAQ">Frequently Asked Questions</a></li>
        </ul>
    </div>
</div>

## Prparation create the client certificates and keys on the VPN server

    ```bash
    cd /etc/easy-rsa/easyrsa3
    ./easyrsa gen-req client1 nopass
    ./easyrsa sign-req client client1
    cp pki/issued/client1.crt /etc/openvpn/client/
    cp pki/private/client1.key /etc/openvpn/client/
    cp pki/ca.crt /etc/openvpn/client/
    ```

## Step 1: (On the client) Install OpenVPN on Ubuntu

First, you need to install OpenVPN in your WSL environment:

1. **Open your WSL terminal**.
2. Run the following commands to update your system and install OpenVPN:

   ```bash
   sudo apt update
   sudo apt install openvpn -y
   ```

## Step 2: Transfer the OpenVPN Client Configuration and Certificates (On the server)

On the server You can create a Template configuration file like so:

```bash
nano client1.ovpn
```

and enter the following contents:

```bash
client
dev tun
proto udp
remote 167.172.84.88 1194  # Replace with your server's IP or domain
resolv-retry infinite
nobind
persist-key
persist-tun

ca ca.crt
cert client1.crt
key client1.key

remote-cert-tls server
data-ciphers AES-256-GCM:AES-128-GCM:CHACHA20-POLY1305

verb 3

```

You need to have the `client1.ovpn` configuration file and related certificates (`ca.crt`, `client1.crt`, `client1.key`) available in your WSL environment. You can achieve this in one of the following ways:

### Use `scp` to copy the config from the OpenVPN Server )on the client)

If the configuration files are still on your OpenVPN server, you can use `scp` to securely copy them to your WSL instance:

```bash
scp user@server-ip:/etc/openvpn/client/client1.ovpn ~/
scp user@server-ip:/etc/openvpn/client/*.crt ~/
scp user@server-ip:/etc/openvpn/client/*.key ~/
```

Replace `user` with your username and `server-ip` with the IP address of your OpenVPN server.

## Step 3: Connect to the VPN

Once the client configuration file (`client1.ovpn`) and certificates are in place, you can initiate the VPN connection.

1. Use the following command to start OpenVPN:

   ```bash
   sudo openvpn --config ~/client1.ovpn
   ```

2. You should see connection logs in the terminal, eventually showing that the VPN connection is established.

## Step 4: Verify the VPN Connection

Once connected, verify that the VPN is working correctly.

1. **Check Your IP Address**:
   ```bash
   ifconfig
   ```
2. **Ping the VPN Server**:

   - From the terminal, ping the internal IP address of the VPN server:
     ```bash
     ping 10.8.0.1
     ```
     You should receive responses, which indicates that the VPN tunnel is working correctly.

## Troubleshooting Tips

1. **Permission Issues**:

   - Make sure you run the `openvpn` command with `sudo` to have the necessary permissions to create a tunnel interface.

2. **Network Issues**:

   - If you are unable to connect, check if the Windows firewall or any antivirus software is blocking OpenVPN.

3. **Check Logs**:

   - If the connection fails, review the logs in the WSL terminal for more details on the issue.

   ```bash
   journalctl -u openvpn
   ```

## Important Note

- **Public and Private IP**: Make sure to replace `[YOUR_SERVER_IP]` in the `client1.ovpn` configuration with your actual server's public IP or domain name.
