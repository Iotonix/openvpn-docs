# OpenVPN Ubuntu server 22.04 Installation Guide

[Back to INDEX](README.md)

# OpenVPN Ubuntu Server 22.04 Installation Guide

[Back to INDEX](README.md)

This guide provides step-by-step instructions for installing and configuring an OpenVPN server on Ubuntu 22.04 LTS.

## Prerequisites

- A server running Ubuntu 22.04 LTS with root or sudo access.
- A basic understanding of Linux command-line operations.
- A registered domain name or a static public IP address for your server. (Optional, but recommended for easier access)

## Step 1: Update System Packages

1. Open a terminal window on your Ubuntu server.
2. Update the package list and upgrade installed packages to their latest versions:
   ```bash
   sudo apt update
   sudo apt upgrade -y
   ```

## Step 2: install OpenVPN

```bash
sudo apt install -y openvpn
# and observe the status after the installated has succeeded
sudo sudo systemctl status openvpn
```

![OpenVPN Status](./images/OpenVpnStatus.png)

## Step 3: Generate SSL/TLS Certificates

1.  **Install EasyRSA:** EasyRSA is a tool that simplifies the process of generating certificates. Install it using the following command:

    ```bash
    sudo apt install easyrsa -y
    ```

2.  **Navigate to the EasyRSA directory:**

    ```bash
    cd /usr/share/easyrsa/3
    ```

3.  **Initialize the Public Key Infrastructure (PKI):**

    ```bash
    ./easyrsa init-pki
    ```

4.  **Create a Certificate Authority (CA):**

    ```bash
    ./easyrsa build-ca nopass
    ```

    (This command creates a CA certificate without a passphrase. For production environments, it's highly recommended to use a passphrase.)

5.  **Generate a server key and certificate:**

    ```bash
    ./easyrsa gen-req server nopass
    ./easyrsa sign-req server server
    ```

6.  **Generate a client key and certificate:**

    ```bash
    ./easyrsa gen-req client nopass
    ./easyrsa sign-req client client
    ```

7.  **Generate Diffie-Hellman parameters:** (for secure key exchange)

    ```bash
    ./easyrsa gen-dh
    ```

8.  **Copy the generated files:** Copy the following files to your OpenVPN configuration directory (usually `/etc/openvpn`):

    ```bash
    sudo cp pki/ca.crt pki/issued/server.crt pki/private/server.key pki/dh.pem /etc/openvpn
    sudo cp pki/issued/client.crt pki/private/client.key /etc/openvpn
    ```

**Important Notes:**

- **Security:** In a production environment, **always** use a strong passphrase when generating your CA and server keys.
- **File Locations:** Make sure to keep track of where you store your certificates and keys.
- **Client Certificates:** You'll need to distribute the `client.crt` and `client.key` files to each client device that needs to connect to the VPN.

This completes the certificate generation process. You can now move on to configuring your OpenVPN server.
