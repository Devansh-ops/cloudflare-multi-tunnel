# Cloudflare Multi-Tunnel Setup Guide

This guide details how to set up two Cloudflare tunnels from a single Linux machine, each connecting to a different Cloudflare account. It's specifically designed for Linux and has been tested on Ubuntu Server 24 LTS.

## Prerequisites
Ensure Cloudflare is installed on your system by executing the following commands:

```bash
sudo mkdir -p --mode=0755 /usr/share/keyrings
curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg | sudo tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null
echo "deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/cloudflared.list
sudo apt-get update && sudo apt-get install cloudflared
```

## Configuration Steps

### Step 1: Authenticate Cloudflare Accounts
1. Execute `cloudflared tunnel login` to start the login process. This command might automatically download or prompt you to manually download the `cert.pem` file.
2. Check either `~/.cloudflared` or `/etc/cloudflared` for the downloaded file. In this guide, `~/.cloudflared` will be used.
3. Rename the `cert.pem` file for the first tunnel as `cert1.pem`:
    ```bash
    mv ~/.cloudflared/cert.pem ~/.cloudflared/cert1.pem
    ```
4. Repeat the login process for the second tunnel and rename the new `cert.pem` file as `cert2.pem`.

### Step 2: Create the Tunnels
Create each tunnel using the following command:
```bash
cloudflared tunnel create tunnel-test
```
**Note**: Keep a note of the tunnel UUID and the path to the `.json` credentials file for later use.

**Note**: Just in case, you can use `sudo cloudflared --config /etc/cloudflared/config2.yml tunnel list` to get a list of tunnels and their UUID's (ID column)

### Step 3: Prepare Configuration Files
Create a YAML configuration file for each tunnel. Here’s an example for the first tunnel (`config1.yml`):

```yml
tunnel: tunnel-1
credentials-file: /home/chpl/.cloudflared/0786db99-744a-41b9-9d1a-efb949256b70.json
origincert: /home/chpl/.cloudflared/cert1.pem
logfile: /var/log/cloudflared1.log
loglevel: info
autoupdate-freq: 24h
no-autoupdate: false
ingress:
  - service: http://localhost:80
warp-routing:
  enabled: false
```

### Step 4: Deploy Configuration Files
Deploy the configuration files to the appropriate directory:
```bash
sudo mkdir -p /etc/cloudflared
sudo cp ~/.cloudflared/config1.yml /etc/cloudflared/config1.yml
sudo cp ~/.cloudflared/config2.yml /etc/cloudflared/config2.yml
```

### Step 5: Register and Start the Tunnel Services

Create and register the basic Cloudflare update services in `/etc/systemd/system/`.

Set up each tunnel service by specifying the configuration file for each.

**Example Tunnel Service (`cloudflared-tunnel-1.service`)**:
```plaintext
[Unit]
Description=cloudflared
After=network-online.target
Wants=network-online.target

[Service]
TimeoutStartSec=0
Type=notify
ExecStart=/usr/bin/cloudflared --no-autoupdate --config /etc/cloudflared/config1.yml tunnel run
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

Start and enable the service to run at boot:
```bash
sudo systemctl start cloudflared-tunnel-1.service
sudo systemctl enable cloudflared-tunnel-1.service
```

### Step 6: Update DNS CNAME Records
Direct your DNS CNAME record to point to the new tunnel:
```bash
sudo cloudflared --config /etc/cloudflared/config2.yml tunnel route dns tunnel-1 devansh.jkcart.com
```

### Managing DNS Records
Deleting DNS records associated with Cloudflare tunnels must currently be done through the API or Dashboard. [Refer to this GitHub issue for more details.](https://github.com/cloudflare/cloudflared/issues/328)
