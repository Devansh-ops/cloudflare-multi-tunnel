# Cloudflare Multi Tunnel

Steps to create two tunnels on a single machine, connecting to two separate cloudflare account

`Note: This is going to be linux specific, and has been tested on ubuntu server 24 LTS`

We will be using locally managed cloudflared tunnels for this

Step 0: Ensure you have cloudflare installed

```bash
sudo mkdir -p --mode=0755 /usr/share/keyrings
curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg | sudo tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null
echo "deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/cloudflared.list
sudo apt-get update && sudo apt-get install cloudflared
```

Step 1 : login to the two cloudflae accounts, and get the `cert.pem` file for both

type
```bash
cloudflared tunnel login
```

Then, it will either automatically download, or you will have to manually download the `cert.pem` file

The file will be saved in either `~/.cloudflared` or `/etc/cloudflared` check and use that. Currently, I am going to use `~/.cloudflared` as the path

Then, rename the `cert.pem` file 
I have used `cert1.pem` for the first tunnel

```bash
mv ~/.cloudflared/cert.pem ~/.cloudflared/cert1.pem
```

Now, Repeat the above steps for the second tunnel, saving the `cert.pem` file as `cert2.pem`

Step 2 : Create the tunnels

```
cloudflared tunnel create tunnel-test
```

example output:
```
Tunnel credentials written to /home/chpl/.cloudflared/41d541cd-1e82-4e96-9a25-788f4608d051.json. cloudflared chose this file based on where your origin certificate was found. Keep this file secret. To revoke these credentials, delete the tunnel.

Created tunnel tunnel-test with id 41d541cd-1e82-4e96-9a25-788f4608d051
```

Use the below command to find your tunnel UUID (column `ID`)
also then go the folder `~/.cloudflared` to find the `.json` file for the tunnel

```bash
cloudflared --config ~/.cloudflared/config1.yml tunnel list
```

Step 3 : Write the config files

Example `config1.yml` file:

```yml
# config1.yml
tunnel: tunnel-1
credentials-file: /home/chpl/.cloudflared/0786db99-744a-41b9-9d1a-efb949256b70.json

# Certificate file obtained from cloudflared tunnel login
origincert: /home/chpl/.cloudflared/cert1.pem

# Log settings
logfile: /var/log/cloudflared1.log
loglevel: info

# Automatic updates
autoupdate-freq: 24h
no-autoupdate: false

# Ingress rules
ingress:
    # can add a hostname here, but we are letting apache / nginx handle the routing
  - service: http://localhost:80 

# Additional optional settings
warp-routing:
  enabled: false
```

Step 4: Copy the config files to `/etc/cloudflared/`
1. Make the directory
```bash
sudo mkdir -p /etc/cloudflared
```
2. Copy and paste
```bash
cp ~/.cloudflared/config1.yml /etc/cloudflared/config1.yml
cp ~/.cloudflared/config2.yml /etc/cloudflared/config2.yml
```

Step 5 : Create and register the basic services

Create them in `/etc/systemd/system/` folder

default cludflare update services:

`cloudflared-update.service`
```
[Unit]
Description=Update cloudflared
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/bin/bash -c '/usr/bin/cloudflared update; code=$?; if [ $code -eq 11 ]; then systemctl restart cloudflared; exit 0; fi; exit $code'
```


`cloudflared-update.timer`
```
[Unit]
Description=Update cloudflared

[Timer]
OnCalendar=daily

[Install]
WantedBy=timers.target
```

Step 6 : Create and register the tunnel service(s)

Template tunnel service

`cloudflared-tunnel-1.service`
```
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

Step 7 : Update the CNAME record to point the host to this tunnel

example:
```bash
 sudo cloudflared --config /etc/cloudflared/config2.yml tunnel route dns 22114866-b3cd-4c85-ad64-49f46f0ab2c5 devansh.jkcart.com
```

### How to delete this dns record?
Currently, the cloudflare tunnel does not support deleting the tunnel via commands.  
You have to use either the API or Dashboard. [Linked GitHub issue asking for this feature](https://github.com/cloudflare/cloudflared/issues/328)


