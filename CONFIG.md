# OpenVPN Server Configuration Guide

In this document, we will continue from where we left off after setting up the Easy-RSA environment and generating the necessary certificates and keys. This guide will help you configure the OpenVPN server and finalize its setup on Ubuntu 22.04 LTS.

## Step 9: Copy Certificates and Keys to OpenVPN Directory

After generating the required keys and certificates using Easy-RSA, you need to copy them to the OpenVPN directory.

1. Create a directory to store the server configuration files:

   ```bash
   mkdir -p /etc/openvpn/server
   ```

2. Copy the server certificate, key, and CA certificate to the OpenVPN directory:

   ```bash
   cp /etc/easy-rsa/easyrsa3/pki/ca.crt /etc/openvpn/server/
   cp /etc/easy-rsa/easyrsa3/pki/issued/server.crt /etc/openvpn/server/
   cp /etc/easy-rsa/easyrsa3/pki/private/server.key /etc/openvpn/server/
   cp /etc/easy-rsa/easyrsa3/pki/dh.pem /etc/openvpn/server/
   ```

   **Note:** It is crucial to secure the `/etc/openvpn` directory since it contains sensitive certificates and keys. You may want to set appropriate permissions to prevent unauthorized access.

## Step 10: Create the OpenVPN Server Configuration File

Next, create the configuration file for the OpenVPN server.

1. Create a new configuration file:

   ```bash
   nano /etc/openvpn/server/server.conf
   ```

2. Add the following configuration to the file:

   ```
   port 1194
   proto udp  # Specifies that the VPN should use the UDP protocol.
   dev tun    # Indicates that a TUN device should be used (for routing IP packets).

   ca /etc/openvpn/server/ca.crt
   cert /etc/openvpn/server/server.crt
   key /etc/openvpn/server/server.key
   dh /etc/openvpn/server/dh.pem

   server 10.8.0.0 255.255.255.0  # Defines the IP address range that will be assigned to connected clients.
   ifconfig-pool-persist ipp.txt

   push "dhcp-option DNS 8.8.8.8"
   push "dhcp-option DNS 8.8.4.4"

   keepalive 10 120
   cipher AES-256-CBC
   user nobody
   group nogroup
   persist-key
   persist-tun

   status openvpn-status.log
   verb 3

   # Reminder: Replace [YOUR_SERVER_IP] with the actual server IP or domain name.

   # Allow specific ports without VPN
   # Use firewall rules to limit port ranges that require VPN
   ```

3. Save and close the file (`CTRL + X`, then `Y`, and `Enter`).

## Step 12: Disable and Reset UFW Rules (Optional)

If you need to clear existing firewall rules and start with a clean slate, you can disable and reset UFW. **Proceed with caution**, especially if you are managing a remote server, as incorrect steps can cause you to lose access.

1. **Disable UFW:**

   ```bash
   ufw disable
   ```

   **WARNING:** Disabling UFW will leave your server without firewall protection temporarily. Ensure that this action is safe in your environment.

2. **Reset UFW Rules:**

   ```bash
   ufw reset
   ```

   **WARNING:** This command will remove all existing rules. Ensure you know the correct rules to reapply afterward, especially for SSH access, to avoid being locked out.

3. **Re-enable UFW and Apply New Rules:**

   - After resetting, add the necessary rules (e.g., for SSH, HTTP, HTTPS, etc.).
   - Enable UFW again:

   ```bash
   ufw enable
   ```

   **WARNING:** Always verify that your SSH rule is configured before enabling UFW to avoid accidental lockout.

## Step 13: Configure Firewall Rules

To allow VPN traffic, you need to configure your firewall to open the necessary port and to restrict other traffic as per your requirement.

1. Allow UDP traffic on port 1194 (default OpenVPN port):

   ```bash
   ufw allow 1194/udp
   ```

2. Allow traffic to be forwarded:

   ```bash
   ufw allow OpenSSH

   # WARNING: Enabling the firewall may cause loss of SSH connection if rules are incorrect. Double-check SSH rules first.
   ufw enable  # WARNING: Enabling the firewall may cause loss of SSH connection if rules are incorrect. Double-check SSH rules first.

   # WARNING: This command modifies system settings. Make sure IP forwarding is needed before proceeding.
   echo 1 > /proc/sys/net/ipv4/ip_forward  # WARNING: This command modifies system settings. Make sure IP forwarding is needed before proceeding.
   ```

3. Add the following line to `/etc/sysctl.conf` to ensure IP forwarding is enabled permanently:

   ```bash
   net.ipv4.ip_forward = 1
   ```

4. Apply the changes:

   ```bash
   sysctl -p
   ```

5. Configure the firewall to allow traffic for specific ports without VPN:

   ```bash
   ufw allow 80/tcp  # Allow HTTP traffic without VPN
   ufw allow 443/tcp # Allow HTTPS traffic without VPN
   ufw allow 943     # Allow port 943 without VPN (OpenVPN admin UI, if used)
   ufw allow 2223/tcp # Allow port 2223 without VPN
   ufw allow OpenSSH # Allow SSH traffic without VPN
   ```

## Step 11: Enable and Start OpenVPN Service

Now that the configuration file is in place and the firewall rules are configured, you can start and enable the OpenVPN service.

1. Enable the OpenVPN service to start on boot:

   ```bash

   # WARNING: Ensure all configurations are correct before enabling OpenVPN to start on boot. Mistakes might lead to loss of access.
   systemctl enable openvpn-server@server  # WARNING: Ensure all configurations are correct before enabling OpenVPN to start on boot.
   ```

2. Start the OpenVPN service:

   ```bash

   # WARNING: Starting the VPN service might interrupt existing network connections. Be cautious if working remotely.
   systemctl start openvpn-server@server  # WARNING: Starting the VPN service might interrupt existing network connections. Be cautious if working remotely.
   ```

3. Check the status to ensure the service is running:

   ```bash
   systemctl status openvpn-server@server
   ```

   You should see the service in the **active (running)** state.

To allow VPN traffic, you need to configure your firewall to open the necessary port and to restrict other traffic as per your requirement.

1. Allow UDP traffic on port 1194 (default OpenVPN port):

   ```bash
   ufw allow 1194/udp
   ```

2. Allow traffic to be forwarded:

   ```bash
   ufw allow OpenSSH
   ufw enable
   echo 1 > /proc/sys/net/ipv4/ip_forward
   ```

3. Add the following line to `/etc/sysctl.conf` to ensure IP forwarding is enabled permanently:

   ```bash
   net.ipv4.ip_forward = 1
   ```

4. Apply the changes:

   ```bash
   sysctl -p
   ```

5. Configure the firewall to allow traffic for ports 80 and 443 without VPN, but require VPN for ports 50000 and upwards:

   ```bash
   ufw allow 80/tcp
   ufw allow 443/tcp
   ufw allow from any to any port 50000:65535 proto tcp comment 'VPN required for high ports'
   ```

## Step 13: Generate Client Certificates and Keys

To allow clients to connect to the VPN, you need to generate certificates and keys for each client.

1. Navigate to the Easy-RSA directory:

   ```bash
   cd /etc/easy-rsa/easyrsa3
   ```

2. Generate a client certificate and key:

   ```bash
   ./easyrsa gen-req client1 nopass
   ```

3. Sign the client certificate request:

   ```bash
   ./easyrsa sign-req client client1
   ```

4. Copy the client certificates and key to a secure location to distribute to the client:

   ```bash
   cp pki/issued/client1.crt /etc/openvpn/client/
   cp pki/private/client1.key /etc/openvpn/client/
   cp pki/ca.crt /etc/openvpn/client/
   ```

   **Note:** Keep the client certificates and keys secure as they grant access to the VPN. Distributing them improperly may compromise the security of your VPN.

## Step 14: Create Client Configuration File

Create a client configuration file that will be used by clients to connect to the VPN.

1. Create a new configuration file:

   ```bash
   nano /etc/openvpn/client/client1.ovpn
   ```

2. Add the following content to the file:

   ```
   client
   dev tun  # Indicates that a TUN device should be used.
   proto udp  # Specifies that the VPN should use the UDP protocol.
   remote [YOUR_SERVER_IP] 1194  # Replace [YOUR_SERVER_IP] with the actual IP address or domain name of the OpenVPN server.
   resolv-retry infinite
   nobind
   persist-key
   persist-tun

   ca ca.crt
   cert client1.crt
   key client1.key

   remote-cert-tls server
   cipher AES-256-CBC
   verb 3
   ```

3. Save and close the file (`CTRL + X`, then `Y`, and `Enter`).

---

## Security Best Practices

- **Use Strong Passphrases:** Always use strong passphrases for the CA and server keys to enhance security.
- **Restrict Access to OpenVPN Server:** Use a firewall to restrict access to the OpenVPN server to trusted IP addresses.
- **Keep Software Updated:** Regularly update OpenVPN, Easy-RSA, and other related software to protect against known vulnerabilities.

These steps complete the basic configuration of an OpenVPN server and client. Let me know if you need additional steps, further explanations, or help troubleshooting any part of the setup!
