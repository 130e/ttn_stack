# The Things Network (TTN) Stack Setup Guide

This repository contains configuration files and documentation for setting up a local The Things Network stack, following the [TTN Docker documentation](https://www.thethingsindustries.com/docs/enterprise/docker/).

## Current Issues

- **Inflexible IP binding**: TTN binds to a single IP address. We should create forwarding rules between interfaces
- **Console account creation**: The "Create Account" button doesn't work in the console page
- **TLS requirement**: Self-signed certificates are required. HTTP (no TLS) doesn't work with the prebuilt TTN Docker image, even with gRPC configured for HTTP

## Prerequisites

Before starting, consider your deployment scenario:
- How will gateway devices connect to TTN?
- How will you access the TTN web UI?

## Network Setup

This guide uses a simple two-device setup:
- **TTN Server**: Raspberry Pi with static IP `192.168.10.2`
- **Gateway Device**: Raspberry Pi with static IP `192.168.10.4`
- **Web Access**: TTN console accessible at `https://192.168.10.2/console/`

### Alternative Setup
You can run TTN directly on the gateway Pi, but note that:
- The interface TTN binds to must have a static IP
- The interface must remain active (ethernet or WiFi must stay connected)

## Installation Steps

1. [Configure Network Interfaces](#configure-network-interfaces)
2. [Generate Certificates](#generate-certificates)
3. [Configure TTN Docker Stack](#configure-ttn-docker-stack)
4. [Create Admin User](#create-admin-user)
5. [Run TTN Docker](#run-ttn-docker)

## Configure Network Interfaces

This guide uses `dhcpcd` for network configuration.

### Install dhcpcd
```bash
sudo apt update
sudo apt install dhcpcd5
sudo systemctl enable dhcpcd
sudo systemctl start dhcpcd
```

### Configure Static IP
Edit `/etc/dhcpcd.conf` and add:
```bash
interface eth0
static ip_address=192.168.10.2/24
```

Repeat this configuration on the gateway Pi with IP `192.168.10.4`.

### Optional: Direct PC Connection
If connecting directly to a PC via ethernet cable, manually configure the PC's ethernet IP to `192.168.10.11` or something similar.

## Generate Certificates

Follow the [custom certificate documentation](https://www.thethingsindustries.com/docs/enterprise/docker/certificates/#custom-certificate-authority).

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
```

### Generate Certificates
1. Navigate to the `cert_gen` directory
2. Update `cert.json` to use your static IP (`192.168.10.2`)
3. Run the certificate generation commands:
```bash
# Add Go bin to PATH
export PATH=$PATH:~/go/bin

# Generate certificates
cfssl genkey -initca ca.json | cfssljson -bare ca
cfssl gencert -ca ca.pem -ca-key ca-key.pem cert.json | cfssljson -bare cert
```

## Configure TTN Docker Stack

Follow the [configuration documentation](https://www.thethingsindustries.com/docs/enterprise/docker/configuration/).

### Prerequisites
- Install Docker and Docker Compose plugin from the official website
- The provided `docker-compose.yml` and `config/stack/ttn-lw-stack-docker.yml` are modified versions of the original open source configuration

### Important Configuration Changes
In `config/stack/ttn-lw-stack-docker.yml`:
- Replace hostnames with static IP addresses
- Generate HTTP cookie key in the HTTP server configuration
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
```bash
SERVER_ADDRESS=https://192.168.10.2
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

## Additional Notes

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

**Note**: If `nmcli` is not installed, check if NetworkManager is running:
```bash
sudo systemctl status NetworkManager
```

If NetworkManager is running, install `nmcli`:
```bash
sudo apt install network-manager
```