# OpenVPN Client Configuration Guide for WSL

This guide will help you set up an OpenVPN client on **WSL (Windows Subsystem for Linux)** to connect to your OpenVPN server.

## Step 1: Install OpenVPN on WSL

First, you need to install OpenVPN in your WSL environment:

1. **Open your WSL terminal**.
2. Run the following commands to update your system and install OpenVPN:

   ```bash
   sudo apt update
   sudo apt install openvpn -y
   ```

## Step 2: Transfer the OpenVPN Client Configuration and Certificates

You need to have the `client1.ovpn` configuration file and related certificates (`ca.crt`, `client1.crt`, `client1.key`) available in your WSL environment. You can achieve this in one of the following ways:

### Option 1: Transfer from Windows to WSL

1. If these files are already on your Windows system, you can move them to WSL.
2. Your Windows filesystem is accessible from WSL through `/mnt/c/`. For example, if your files are located in `C:\Users\YourUsername\Downloads\`, they can be accessed in WSL at `/mnt/c/Users/YourUsername/Downloads/`.
3. Copy these files to your WSL home directory for easier access:

   ```bash
   cp /mnt/c/Users/YourUsername/Downloads/client1.ovpn ~/
   cp /mnt/c/Users/YourUsername/Downloads/*.crt ~/
   cp /mnt/c/Users/YourUsername/Downloads/*.key ~/
   ```

### Option 2: Use `scp` from the OpenVPN Server

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

   - Open a browser in your Windows system and visit [https://whatismyipaddress.com/](https://whatismyipaddress.com/).
   - The IP address displayed should be the **public IP of the VPN server**, not your local IP.

2. **Ping the VPN Server**:

   - From the WSL terminal, ping the internal IP address of the VPN server:
     ```bash
     ping 10.8.0.1
     ```
     You should receive responses, which indicates that the VPN tunnel is working correctly.

3. **Check Routing**:
   - If you want to confirm that all your traffic is being routed through the VPN, you can use:
     ```bash
     traceroute google.com  # On Linux (requires `traceroute` package)
     ```
     The output should show that the traffic is being routed through the VPN.

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

- **Run WSL with Administrator Privileges**: To ensure proper routing, you may need to run WSL with elevated privileges. Right-click on your terminal icon and choose **Run as administrator**.

- **Public and Private IP**: Make sure to replace `[YOUR_SERVER_IP]` in the `client1.ovpn` configuration with your actual server's public IP or domain name.

This completes the client setup for WSL. Let me know if you need further assistance or additional configuration steps!
