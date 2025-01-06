# Setup a web server on your homelab without exposing its IP to the internet

This guide will walk you through setting up a web server on your homelab and making it accessible to the internet securely using an Oracle VPS and Tailscale.

---

## Step 1: Setting up a local webserver on your homelab

### Step 1.1: Installing Podman

Choose the distribution running on your homelab server and follow the corresponding installation instructions:

#### Arch Linux
```bash
sudo pacman -S podman
```

#### Debian/Ubuntu
```bash
# Add the repository
. /etc/os-release
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/xUbuntu_${VERSION_ID}/Release.key | gpg --dearmor | sudo tee /etc/apt/keyrings/devel_kubic_libcontainers_stable.gpg > /dev/null
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/devel_kubic_libcontainers_stable.gpg] https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/xUbuntu_${VERSION_ID}/ /" | sudo tee /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list > /dev/null

# Install Podman
sudo apt-get update
sudo apt-get -y install podman
```

#### NixOS
Add to your configuration.nix:
```nix
{
  virtualisation.podman = {
    enable = true;
    dockerCompat = true;  # Optional: enables Docker CLI compatibility
  };
}
```
Then run:
```bash
sudo nixos-rebuild switch
```

### Step 1.2: Creating a test container

1. Create a new directory for your test site:
```bash
mkdir -p ~/nginx-test
```

2. Create a test HTML file:
```bash
cat << EOF > ~/nginx-test/index.html
<!DOCTYPE html>
<html>
<head>
    <title>Homelab Test</title>
</head>
<body>
    <h1>Hello from homelab!</h1>
</body>
</html>
EOF
```

### Step 1.3: Configure Firewall

1. Install and enable UFW:
```bash
sudo apt install ufw
sudo ufw enable
```

2. Configure firewall rules:
```bash
sudo ufw allow 22/tcp
sudo ufw allow 42080/tcp
```

3. Verify rules:
```bash
sudo ufw status
```

### Step 1.4: Running the container

1. Start the Nginx container:
```bash
podman run -d \
  --name nginx-test \
  -p 42080:80 \
  -v ~/nginx-test:/usr/share/nginx/html:ro \
  docker.io/library/nginx:alpine
```

2. Verify the container is running:
```bash
podman ps
```

3. Test the webserver:
```bash
curl localhost:42080
```
You should see your HTML content as the response.

4. (Optional) Test in browser:
Open `http://localhost:42080` in your browser to see the page.

---

## Step 2: Setting up Oracle VPS

### Step 2.1: Creating an Oracle Cloud Account

1. Sign up for a free account at https://cloud.oracle.com
2. Complete the verification process

### Step 2.2: Creating a Virtual Machine Instance

1. Navigate to: Navigation menu > Computing > Instances
2. Click "Create Instance"
3. Configure the instance:
   - Name your instance
   - Placement: Keep "Always Free Eligible" selected
   - Image: Click "Edit" and select "Canonical Ubuntu 24.04"
   - Shape: Leave default (VM.Standard.A1.Flex)

### Step 2.3: Network Configuration

1. Create a new Virtual Cloud Network (VCN):
   - A VCN is a virtual network in Oracle Cloud that provides isolated and secure network environment
   - Click "Create new VCN"
   - Name your VCN
   - Select "Create VCN with Internet Connectivity"

2. Create a new subnet:
   - Click "Create new public subnet"
   - This will automatically configure the routing and security rules

### Step 2.4: SSH Key Configuration

1. Under "Add SSH keys", select "Generate a key pair"
2. Download BOTH keys:
   - Private key (id_rsa)
   - Public key (id_rsa.pub)
3. Save these files securely - they are required for server access

### Step 2.5: Review and Create

1. Review all settings
2. Click "Create" to launch your instance
3. Wait for the instance to change status to "Running"

### Step 2.6: Connecting to Your Instance

1. Locate your instance's public IP address:
   - Find it on the Instance Details page
   - Or use: "Public IP address" field

2. Set correct permissions for your private key:
```bash
chmod 600 ~/path/to/private_key
```

3. Connect via SSH:
```bash
ssh -i ~/path/to/private_key ubuntu@your_public_ip
```

Note: Replace `~/path/to/private_key` with the actual path to your downloaded private key, and `your_public_ip` with the IP address from step 1.

### Step 2.7: Network Security Configuration

1. Navigate to VCN details page:
   - Open the VCN used by your instance in a new tab
   - The following components need configuration:

2. Internet Gateway:
   - Verify existence or create new
   - Status should be "Available"

3. Subnet Configuration:
   - Check subnet status is "Available"
   - Verify it's in the same availability domain as your instance

4. Route Tables:
   - Verify Default Route Table has rule:
     - Destination: 0.0.0.0/0
     - Target: Your Internet Gateway
   - Add if missing

5. Security Lists:
   - Add Ingress Rules:
     ```
     HTTP (80):
     Source: 0.0.0.0/0
     Protocol: TCP
     Destination Port: 80

     HTTPS (443):
     Source: 0.0.0.0/0
     Protocol: TCP
     Destination Port: 443
     ```

---

## Step 3: Installing Tailscale

### Step 3.1: Create Tailscale Account
1. Sign up at https://login.tailscale.com

### Step 3.2: Install Tailscale

Go to https://login.tailscale.com/admin/machines/new for installation commands:
- Select your OS to get a customized installation command with auth key
- Commands will look like: `sudo tailscale up --authkey=tskey-auth-xxxxx-xxxxxxxxxxxxx`

#### On Homelab:

##### Ubuntu/Debian:
```bash
curl -fsSL https://pkgs.tailscale.com/stable/ubuntu/jammy.noarmor.gpg | sudo tee /usr/share/keyrings/tailscale-archive-keyring.gpg >/dev/null
curl -fsSL https://pkgs.tailscale.com/stable/ubuntu/jammy.tailscale-keyring.list | sudo tee /etc/apt/sources.list.d/tailscale.list
sudo apt-get update
sudo apt-get install tailscale
sudo tailscale up
```

##### Arch Linux:
```bash
sudo pacman -S tailscale
sudo systemctl enable --now tailscaled
sudo tailscale up
```

##### NixOS:
Add to configuration.nix:
```nix
{
  services.tailscale.enable = true;
}
```
Then:
```bash
sudo nixos-rebuild switch
sudo tailscale up
```

#### On Oracle VPS:
```bash
curl -fsSL https://tailscale.com/install.sh | sudo bash
sudo tailscale up
```

#### On Personal PC:
Download from: https://tailscale.com/download

### Step 3.3: Verify Connection and Test

1. Get Tailscale IPs for all devices:
```bash
tailscale ip
```
Note down these IPs:
- Homelab: `100.x.y.z`
- Oracle VPS: `100.x.y.z`
- Personal PC: `100.x.y.z`

2. Test connection to homelab webserver:
```bash
# From Oracle VPS or Personal PC
curl 100.x.y.z:42080
```
Replace `100.x.y.z` with your homelab's Tailscale IP. You should see the "Hello from homelab!" message.

## Step 4: Setting up Nginx Reverse Proxy

### Step 4.1: Install Nginx
SSH to your Oracle VPS (if you haven't already) and install Nginx:
```bash
sudo apt update
sudo apt install nginx
```

### Step 4.2: Configure Nginx

1. Create new site configuration:
```bash
sudo nano /etc/nginx/sites-available/homelab
```

2. Add the following configuration:
```nginx
server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://TAILSCALE_IP:42080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```
Replace `TAILSCALE_IP` with your homelab's Tailscale IP.

3. Enable the site:
```bash
sudo ln -s /etc/nginx/sites-available/homelab /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl restart nginx
```

### Step 4.3: Configure NAT Rules

1. Install iptables-persistent:
```bash
sudo apt install iptables-persistent
```

2. Add NAT rules:
```bash
sudo iptables -t nat -A POSTROUTING -p tcp -d TAILSCALE_IP --dport 42080 -j MASQUERADE
sudo iptables -t nat -A PREROUTING -p tcp --dport 80 -j DNAT --to-destination TAILSCALE_IP:42080
```
Replace `TAILSCALE_IP` with your homelab's Tailscale IP.

3. Make rules persistent:
```bash
sudo netfilter-persistent save
```

### Step 4.4: Configure Firewall

1. Install UFW:
```bash
sudo apt install ufw
```

2. Configure rules:
```bash
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
```

3. Enable firewall:
```bash
sudo ufw enable
sudo ufw status
```

## Step 5: Testing the Setup

1. From any device, open a web browser and enter the Oracle VPS's public IP address:
```
http://YOUR_VPS_PUBLIC_IP
```

2. You should see the "Hello from homelab!" page that was previously only accessible locally.

Note: If the page doesn't load, verify:
- Nginx is running: `sudo systemctl status nginx`
- UFW rules are correct: `sudo ufw status`
- VPS security list includes port 80
- NAT rules are active: `sudo iptables -t nat -L`


## Step 6: Adding Domain and SSL (Optional)

### Step 6.1: Configure Domain
1. Create account at NoIP.org
2. Go to Dynamic DNS -> No-IP Hostnames
2. Create DDNS hostname (e.g., xxxx-homelab.ddns.net) pointing to VPS public IP

### Step 6.2: Update Nginx Configuration
1. Edit site configuration:
```bash
sudo nano /etc/nginx/sites-available/homelab
```

2. Update server block:
```nginx
server {
    listen 80;
    server_name xxxx-homelab.ddns.net;

    location / {
        proxy_pass http://TAILSCALE_IP:42080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Step 6.3: Install and Configure SSL
```bash
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d xxxx-homelab.ddns.net
```

Certbot will automatically modify the Nginx configuration to handle SSL.

### Step 6.4: Configure HTTPS

1. Check the main site configuration on your VPS:
```bash
sudo cat /etc/nginx/sites-available/homelab
```

2. It should look something like this, replace it if necessary:
```nginx
server {
    server_name xxxx-homelab.ddns.net;
    listen 443 ssl;
    
    # SSL configuration
    ssl_certificate /etc/letsencrypt/live/xxxx-homelab.ddns.net/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/xxxx-homelab.ddns.net/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    location / {
        proxy_pass http://TAILSCALE_IP:42080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

### Step 6.5: Add HTTP to HTTPS Redirect

1. Create redirect configuration:
```bash
sudo nano /etc/nginx/sites-available/http-redirect
```

2. Add redirect rules:
```nginx
server {
    listen 80 default_server;
    server_name _;  # Catches all server names
    return 301 https://$host$request_uri;
}
```

3. Enable configuration:
```bash
sudo ln -s /etc/nginx/sites-available/http-redirect /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

## Step 7: Final Configuration

### Step 7.1: Automating Container Startup

1. Create podman-compose.yml:
```bash
mkdir ~/nginx-compose
cd ~/nginx-compose
nano podman-compose.yml
```

```yaml
version: '3'
services:
  web:
    image: docker.io/library/nginx:alpine
    ports:
      - "42080:80"
    volumes:
      - ~/nginx-test:/usr/share/nginx/html:ro
    restart: always
```

2. Create systemd service:
```bash
mkdir -p ~/.config/systemd/user/
podman-compose generate-systemd web > ~/.config/systemd/user/podman-compose-web.service
systemctl --user enable podman-compose-web.service
systemctl --user start podman-compose-web.service
loginctl enable-linger $USER
```

### Step 7.2: Website Maintenance
- To edit your website content, you modify files in directory: `~/nginx-test/index.html`
- To change the directory of the website content: Change volume path in podman-compose.yml
- After changes to the setup, we need to restart our pod with nginx: `systemctl --user restart podman-compose-web.service`
- You don't need to restart anything after just changing the website content in your folder, like editing your index.html. Those changes should be visible instantly after saving the file
