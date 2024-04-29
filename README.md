# Cloudflare Multi-Tunnel Setup Guide

This guide explains how to establish two Cloudflare tunnels from a single Linux machine, each connected to a separate Cloudflare account. It is tailored for Linux, specifically tested on Ubuntu Server 24 LTS.

## Prerequisites
Before you begin, ensure Cloudflare is installed on your system by executing the following commands:

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
Use the following command to create each tunnel:
```bash
cloudflared tunnel create tunnel-test
```
**Note**: Record the tunnel UUID and path to the `.json` credentials file from the output for future use.

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
cp ~/.cloudflared/config1.yml /etc/cloudflared/config1.yml
cp ~/.cloudflared/config2.yml /etc/cloudflared/config2.yml
```

### Step 5: Setup System Services
Create and register the basic Cloudflare update services in `/etc/systemd/system/`.

**Cloudflare Update Service (`cloudflared-update.service`)**:
```plaintext
[Unit]
Description=Update cloudflared
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/bin/bash -c '/usr/bin/cloudflared update; code=$?; if [ $code -eq 11 ]; then systemctl restart cloudflared; exit 0; fi; exit $code'
```

**Cloudflare Update Timer (`cloudflared-update.timer`)**:
```plaintext
[Unit]
Description=Update cloudflared

[Timer]
OnCalendar=daily

[Install]
WantedBy=timers.target
```

### Step 6: Register the Tunnel Services
Setup each tunnel service, specifying the configuration file for each.

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

### Step 7: Update DNS CNAME Records
Direct your DNS CNAME record to point to the new tunnel:
```bash
sudo cloudflared --config /etc/cloudflared/config2.yml tunnel route dns 22114866-b3cd-4c85-ad64-49f46f0ab2c5 devansh.jkcart.com
```

### Managing DNS Records
Currently, deleting DNS records associated with Cloudflare tunnels must be done through the API or Dashboard due to limitations in the CLI tool. [Refer to this GitHub issue for more details

.](https://github.com/cloudflare/cloudflared/issues/328)

This comprehensive guide should assist you in setting up multiple Cloudflare tunnels on a single Linux machine, each configured and managed independently.