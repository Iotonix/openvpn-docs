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

   ```bash
   chmod 700 /etc/openvpn/server/
   ```

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
        data-ciphers AES-256-GCM:AES-128-GCM:CHACHA20-POLY1305
        user nobody
        group nogroup
        persist-key
        persist-tun

        status openvpn-status.log
        verb 3
   # Allow specific ports without VPN
   # Use firewall rules to limit port ranges that require VPN (see below)
   ```

3. Save and close the file (`CTRL + X`, then `Y`, and `Enter`).

## Step 11: Disable and Reset UFW Rules (Optional)

If you need to clear existing firewall rules and start with a clean slate, you can disable and reset UFW. **Proceed with caution**, especially if you are managing a remote server, as incorrect steps can cause you to lose access.

1.  **Disable UFW:**

    ```bash
    ufw disable
    # In our case we had many ports open and will them all require to fo through VPN.
    ufw reset
    ```

    **WARNING:** Disabling UFW will leave your server without firewall protection temporarily. Ensure that this action is safe in your environment.

2.  **Modify /etc/ufw/before.rules:**

    ```bash
    nano /etc/ufw/before.rules
    ```

    Our Example we include:

    ```bash
        # /etc/ufw/before.rules

        # START UFW BEFORE RULES
        *filter
        :ufw-before-input - [0:0]
        :ufw-before-output - [0:0]
        :ufw-before-forward - [0:0]
        :ufw-not-local - [0:0]
        # End of required lines

        # Allow all on loopback interface
        -A ufw-before-input -i lo -j ACCEPT
        -A ufw-before-output -o lo -j ACCEPT

        # Allow established and related connections
        -A ufw-before-input -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
        -A ufw-before-output -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
        -A ufw-before-forward -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT

        # Drop invalid packets
        -A ufw-before-input -m conntrack --ctstate INVALID -j ufw-logging-deny
        -A ufw-before-input -m conntrack --ctstate INVALID -j DROP

        # Allow specific ICMP types for INPUT
        -A ufw-before-input -p icmp --icmp-type destination-unreachable -j ACCEPT
        -A ufw-before-input -p icmp --icmp-type time-exceeded -j ACCEPT
        -A ufw-before-input -p icmp --icmp-type parameter-problem -j ACCEPT
        -A ufw-before-input -p icmp --icmp-type echo-request -j ACCEPT

        # Allow specific ICMP types for FORWARD
        -A ufw-before-forward -p icmp --icmp-type destination-unreachable -j ACCEPT
        -A ufw-before-forward -p icmp --icmp-type time-exceeded -j ACCEPT
        -A ufw-before-forward -p icmp --icmp-type parameter-problem -j ACCEPT
        -A ufw-before-forward -p icmp --icmp-type echo-request -j ACCEPT

        # Allow DHCP client to work
        -A ufw-before-input -p udp --sport 67 --dport 68 -j ACCEPT

        # UFW Not Local Rules
        -A ufw-before-input -j ufw-not-local
        -A ufw-not-local -m addrtype --dst-type LOCAL -j RETURN
        -A ufw-not-local -m addrtype --dst-type MULTICAST -j RETURN
        -A ufw-not-local -m addrtype --dst-type BROADCAST -j RETURN
        -A ufw-not-local -m limit --limit 3/min --limit-burst 10 -j ufw-logging-deny
        -A ufw-not-local -j DROP

        # Allow MULTICAST mDNS for service discovery
        -A ufw-before-input -p udp -d 224.0.0.251 --dport 5353 -j ACCEPT

        # Allow MULTICAST UPnP for service discovery
        -A ufw-before-input -p udp -d 239.255.255.250 --dport 1900 -j ACCEPT

        # Forwarding rules for OpenVPN
        # Allow traffic from VPN clients to the external network
        -A ufw-before-forward -i tun0 -o eth0 -j ACCEPT
        # Allow return traffic from the external network to VPN clients
        -A ufw-before-forward -i eth0 -o tun0 -m state --state RELATED,ESTABLISHED -j ACCEPT

        # END UFW BEFORE RULES
        COMMIT

        # NAT Table Rules
        *nat
        :POSTROUTING ACCEPT [0:0]

        # Enable NAT for VPN subnet through eth0
        -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE

        COMMIT

    ```

    Explanation of Key Sections:

         Filter Table (*filter):
             Loopback Traffic: Allows all traffic on the loopback interface (lo), ensuring local processes can communicate without restrictions.
             Established Connections: Permits traffic for established and related connections, maintaining ongoing sessions.
             ICMP Traffic: Allows necessary ICMP types for network diagnostics and functionality.
             Forwarding Rules:
                 -A ufw-before-forward -i tun0 -o eth0 -j ACCEPT: Allows VPN clients (tun0) to send traffic to the external network (eth0).
                 -A ufw-before-forward -i eth0 -o tun0 -m state --state RELATED,ESTABLISHED -j ACCEPT: Allows return traffic from the external network to VPN clients.

         NAT Table (*nat):
             POSTROUTING Chain:
                 -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE: Implements NAT for the VPN subnet (10.8.0.0/24) when accessing the external network (eth0). This allows VPN clients to share the server's public IP for internet access.

3.  **Re-enable UFW and Apply New Rules:**

    - After resetting, add the necessary rules (e.g., for SSH, HTTP, HTTPS, etc.).
    - Enable UFW again:

    ```bash
    ufw enable
    ```

    **WARNING:** Always verify that your SSH rule is configured before enabling UFW to avoid accidental lockout.

## Step 12: Configure Firewall Rules

To allow VPN traffic, you need to configure your firewall to open the necessary port and to restrict other traffic as per your requirement.

1. Allow UDP traffic on port 1194 (default OpenVPN port):

   ```bash
   ufw allow 1194/udp
   ```

2. Allow traffic to be forwarded:

   ```bash
    ufw allow OpenSSH
    # WARNING: This command modifies system settings. Make sure IP forwarding is needed before proceeding.
    echo 1 > /proc/sys/net/ipv4/ip_forward
    # WARNING: This command modifies system settings. Make sure IP forwarding is needed before proceeding.
    # IP forwarding allows your server to act as a router, which is essential for allowing VPN clients to access external networks.

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
   # here somne examples we added in our case.
   ufw deny from 125.25.108.63
   ufw deny from 91.92.255.200
   ufw deny from 101.108.133.28
   ```

## Step 13: Enable and Start OpenVPN Service

Now that the configuration file is in place and the firewall rules are configured, you can start and enable the OpenVPN service.

1. Enable the OpenVPN service to start on boot:

   ```bash

   # WARNING: Ensure all configurations are correct before enabling OpenVPN to start on boot. Mistakes might lead to loss of access.
   systemctl enable openvpn-server@server  # WARNING: Ensure all configurations are correct before enabling OpenVPN to start on boot.
   ```

2. Start the OpenVPN service:

   ```bash
   # WARNING: Starting the VPN service might interrupt existing network connections. Be cautious if working remotely.
   systemctl start openvpn-server@server
   ```

3. Check the status to ensure the service is running:

   ```bash
   systemctl status openvpn-server@server
   ```

   You should see the service in the **active (running)** state.

## Security Best Practices

- **Use Strong Passphrases:** Always use strong passphrases for the CA and server keys to enhance security.
- **Restrict Access to OpenVPN Server:** Use a firewall to restrict access to the OpenVPN server to trusted IP addresses.
- **Keep Software Updated:** Regularly update OpenVPN, Easy-RSA, and other related software to protect against known vulnerabilities.

These steps complete the basic configuration of an OpenVPN server and client. Let me know if you need additional steps, further explanations, or help troubleshooting any part of the setup!

[Continue to client](CLIENT.md)

[Back to INDEX](README.md)
