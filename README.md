# The Things Network (TTN) Stack Setup Guide

This repository contains configuration files and documentation for setting up a local The Things Network stack, following the [TTN Docker documentation](https://www.thethingsindustries.com/docs/enterprise/docker/).

## Prerequisites

Before starting, consider your deployment scenario:
- How will gateway devices connect to TTN?
- How will you access the TTN web UI?

## Network Setup

This example uses a simple setup with only one raspberry Pi 5 (denote as Pi):
- **Gateway**: SX1250 LoRaWAN HAT installed in Pi
- **TTN Server**: TTN opensource docker release installed in Pi
- **Local Setup**: Both gateway and sensors registered in Pi's TTN server
- **Access**: TTN console **ONLY** accessible from the Pi

## Installation Steps

1. [Configure Network Interfaces](#configure-network-interfaces)
2. [Generate Certificates](#generate-certificates)
3. [Configure TTN Docker Stack](#configure-ttn-docker-stack)
4. [Create Admin User](#create-admin-user)
5. [Run TTN Docker](#run-ttn-docker)

## Configure Network Interfaces

Once again, this step is not necessary for deploying in a machine with a static and always available IP address (or a domain).

Here we create a dummy IP for TTN server, because TLS certificate is needed and we cannot generate a certificate for `localhost`!

To ensure we have this dummy interface every time we reboot, 
we use built-in `/etc/rc.local`. 

Put the following into `/etc/rc.local` (if the file already exists, add before `exit 0`),
```bash
#!/bin/bash

# Load dummy module
modprobe dummy

# Create and configure dummy interface
ip link add dummy0 type dummy
ip link set dummy0 up

# Add primary IP
ip addr add 192.168.111.1/24 dev dummy0

# Log for debugging
logger "Dummy interface dummy0 has been configured"

exit 0
```

Then make sure it is executable,
```bash
sudo chmod +x /etc/rc.local
```

Then reboot. To verify after reboot:
```bash
ip addr show dummy0
```

## Generate Certificates

Also refer to [custom certificate documentation](https://www.thethingsindustries.com/docs/enterprise/docker/certificates/#custom-certificate-authority).

### Install Prerequisites
```bash
# Install Go (version >1.20)
# Download the arm64 tarball from https://go.dev/dl/
wget https://go.dev/dl/go1.24.5.linux-arm64.tar.gz
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf go1.24.5.linux-arm64.tar.gz

# Edit /etc/profile and add
export PATH=$PATH:/usr/local/go/bin

# Install cfssl
go install github.com/cloudflare/cfssl/cmd/...@latest

# Similarly add Go bin to PATH so we can access cfssl
export PATH=$PATH:~/go/bin
```

### Generate Certificates
1. Navigate to the `cert_gen` directory `cd cert_gen`
2. Update `cert.json` to use the static IP (`192.168.111.1`)
3. Run the certificate generation commands:
```bash

# Generate certificates
cfssl genkey -initca ca.json | cfssljson -bare ca
cfssl gencert -ca ca.pem -ca-key ca-key.pem cert.json | cfssljson -bare cert

# Rename according to yml config
mv cert-key.pem key.pem

# Permission
sudo chown 886:886 ./cert.pem ./key.pem
```

## Configure TTN Docker Stack

Also refer to the [configuration documentation](https://www.thethingsindustries.com/docs/enterprise/docker/configuration/).

### Prerequisites
- Install Docker and Docker Compose plugin from the official website
- The provided `docker-compose.yml` and `config/stack/ttn-lw-stack-docker.yml` are modified versions of the original open source configuration

### Important Configuration Changes
Notably, here are what was changed in `config/stack/ttn-lw-stack-docker.yml` compared to original version:
- Replace hostnames with the static IP address
- Re-generate HTTP cookie in the HTTP server configuration
- Use custom CA instead of Let's Encrypt

## Create Admin User

Follow the [initialization documentation](https://www.thethingsindustries.com/docs/enterprise/docker/running-the-stack/#initialization), ignoring Enterprise-related modules.

### Initialize Database
```bash
docker compose pull
docker compose run --rm stack is-db migrate
```

### Create Admin User
```bash
docker compose run --rm stack is-db create-admin-user \
  --id admin \
  --email admin@test.local
```

### Create OAuth Clients

#### CLI Client
```bash
docker compose run --rm stack is-db create-oauth-client \
  --id cli \
  --name "Command Line Interface" \
  --owner admin \
  --no-secret \
  --redirect-uri "local-callback" \
  --redirect-uri "code"
```

#### Console Client
Note that the static IP is used here.
```bash
SERVER_ADDRESS=https://192.168.111.1
ID=console
NAME=Console
CLIENT_SECRET=console
REDIRECT_URI=${SERVER_ADDRESS}/console/oauth/callback
REDIRECT_PATH=/console/oauth/callback
LOGOUT_REDIRECT_URI=${SERVER_ADDRESS}/console
LOGOUT_REDIRECT_PATH=/console

docker compose run --rm stack is-db create-oauth-client \
  --id ${ID} \
  --name "${NAME}" \
  --owner admin \
  --secret "${CLIENT_SECRET}" \
  --redirect-uri "${REDIRECT_URI}" \
  --redirect-uri "${REDIRECT_PATH}" \
  --logout-redirect-uri "${LOGOUT_REDIRECT_URI}" \
  --logout-redirect-uri "${LOGOUT_REDIRECT_PATH}"
```

## Run TTN Docker

```bash
# Start the stack
docker compose up

# Stop the stack
docker compose down
```

## Bonus: Network Configuration
This assumes your system is using `NetworkManager` to manage network.
```bash
# Check
sudo systemctl status NetworkManager
```

If not found, it is likely your OS is using `wpa_supplicant` with `dhcpcd`. Similar resources are available online.

### Connecting to WPA2 Enterprise WiFi (CLI)

Use these commands to connect to WPA2 Enterprise WiFi without a GUI:

```bash
# Variables to set:
# CONNECTION_NAME: random string for connection name
# INTERFACE: WiFi interface (wlan0, wlo1, etc.) - use 'ip link' to find
# SSID: WiFi network name
# USERNAME: your username
# PASSWORD: your password
# ANONYMOUS-IDENTITY: random string

sudo nmcli con add type wifi ifname {INTERFACE} con-name {CONNECTION_NAME} ssid {SSID}
sudo nmcli con edit id {CONNECTION_NAME}

nmcli> set ipv4.method auto
nmcli> set 802-1x.eap peap
nmcli> set 802-1x.phase2-auth mschapv2
nmcli> set 802-1x.identity {USERNAME}
nmcli> set 802-1x.password {PASSWORD}
nmcli> set 802-1x.anonymous-identity {ANONYMOUS-IDENTITY}
nmcli> set wifi-sec.key-mgmt wpa-eap
nmcli> save
nmcli> activate
```

### Set up Static Ethernet IP
```bash
sudo nmcli connection add type ethernet \
    con-name "eth0-static" \
    ifname eth0 \
    ipv4.method manual \
    ipv4.addresses 192.168.10.10/24
```
Note that with this config Pi won't be able to get network from ethernet.
But it is useful for `ssh` in headless setup.
You can run a cable between laptop and Pi. Then manually configure laptop's ethernet IP to be `192.168.10.3`.

## TODO

### HTTP-only Issue

This is WIP and could be misconfigurations.

With the prebuilt docker, TLS certificates are required. I have no success with the HTTP only configuration. 
Even with gRPC configured to listen HTTP port, it still attempts TLS connection.