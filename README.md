Local Docker NGINX Setup

This project provides a step-by-step guide to setting up a local NGINX web server and reverse proxy using Docker and Docker Compose. The guide covers everything from initial system setup to configuring a self-signed SSL certificate for secure access.

## Prerequisites

Before starting, ensure you have the following:

A virtual machine (VM) running CentOS
Install OS RHEL-based (CentOS) 


## Steps

### Hypervisor Installation

1. Install Hypervisor (e.g., VirtualBox or VMware), in this project we using VirtualBox:**
Download and install your preferred hypervisor
Create a New Virtual Machine: Open VirtualBox and click "New."
Name and OS Type: Enter the name of your VM and select the appropriate OS type (Linux) and version.
Memory Size: Assign at least 1-2 GB of RAM.
Hard Disk: Choose "Create a virtual hard disk now" and allocate at least 10 GB of space.
Storage: Select "Dynamically allocated" to save disk space.
Select ISO: Under "Storage," click on the "Empty" disk, and load the RHEL-based OS (CentOS) ISO.
Start VM: Start the VM and follow the installation instructions.

### OS Installation

1. Install CentOS on the VM
   - Download the CentOS ISO.
   - Attach the ISO to the VM and start the VM.
   - Follow the installation prompts to install CentOS.
   - Set up a root password and create a user.

2. Configure SSH Access
Install OpenSSH Server
sudo dnf install openssh-server  # For Fedora, CentOS, Rocky, Alma
Start and Enable SSH Service
sudo systemctl start sshd
sudo systemctl enable sshd
Configure Firewall to Allow SSH (if firewall is enabled):
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --reload

3. Activate SELinux
Check the Current SELinux Status
sestatus
Edit the SELinux Configuration File:
sudo vi /etc/selinux/config
Look for the line that starts with SELINUX=. Change it to enforcing:
SELINUX=enforcing
Save and Exit
Press Esc, then type :wq
Reboot the System
sudo reboot
Verify SELinux is Active
sestatus

4. Port Allowed
Install firewalld (if not already installed):
sudo yum install firewalld
Start and Enable firewalld
sudo systemctl start firewalld
sudo systemctl enable firewalld
Check Active Zones:
sudo firewall-cmd --get-active-zones
Allow SSH, HTTP, and HTTPS Ports:
Allow SSH (port 22):
sudo firewall-cmd --zone=public --add-service=ssh --permanent
Allow HTTP (port 80):
sudo firewall-cmd --zone=public --add-service=http --permanent
Allow HTTPS (port 443):
sudo firewall-cmd --zone=public --add-service=https --permanent
Remove Other Ports (Optional):
sudo firewall-cmd --zone=public --remove-port=PORT_NUMBER/tcp --permanent
#Replace PORT_NUMBER with the actual port number you want to block.
Reload the Firewall:
sudo firewall-cmd --reload
Check Open Ports:
sudo firewall-cmd --zone=public --list-all

### Docker Installation

1. Update System Packages
sudo yum update -y
2. Install Required Packages
sudo yum install -y yum-utils
3. Add Docker Repository
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
4. Install Docker
sudo yum install -y docker-ce docker-ce-cli containerd.io
5. Start and Enable Docker
sudo systemctl start docker
sudo systemctl enable docker
6. Install Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/download/v2.7.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
7. Verify Docker Installation
docker --version
docker-compose --version

### Services Installation

#### Webserver
1. Create Project Directory
mkdir ~/nginx-webserver
cd ~/nginx-webserver
2. Create `docker-compose.yml` file
sudo nano docker-compose.yml
   version: '3'
   services:
     web:
       image: nginx:latest
       volumes:
         - ./html:/usr/share/nginx/html
       networks:
         - nginx_network
   networks:
      nginx_network:
        external: true 
3. Create HTML Directory and File
mkdir ~/nginx-webserver/html
sudo nano ~/nginx-webserver/html/index.html
`index.html` content:**
   <!DOCTYPE html>
   <html>
   <head>
       <title>My Dandi NGINX Container</title>
   </head>
   <body>
       <h1>Welcome to my NGINX Container!</h1>
   </body>
   </html>
4. Deploy NGINX Webserver Container
    sudo docker-compose up -d 
    #or if you’re using the Docker Compose plugin
    sudo docker compose up -d
5. Display directory structure 
    tree
6. Test Access
   - Open your browser and navigate to `http://your_vm_ip:10080’.

#### Reverse Proxy

##### Self-Signed Certificate

1. Create Directory for SSL
sudo mkdir -p /etc/nginx/ssl
cd /etc/nginx/ssl
2. Generate SSL Certificate
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout selfsigned.key -out selfsigned.crt
#Common Name: Enter your domain name (e.g., `dandi.local`).
#Other fields (Country Name, State, etc.) can be filled in as needed or left blank.
3. Display and save the Certificate
Display certificate
cat /etc/nginx/ssl/selfsigned.crt
cat /etc/nginx/ssl/selfsigned.key
Save certificate
sudo cp /etc/nginx/ssl/selfsigned.crt ~/
sudo cp /etc/nginx/ssl/selfsigned.key ~/
Change file permissions
sudo chmod 644 ~/selfsigned.key
sudo chmod 644 ~/selfsigned.crt

##### NGINX Proxy Manager Setup

1. Create Project Directory
mkdir ~/nginx-proxy-manager
cd ~/nginx-proxy-manager

2. Create `docker-compose.yml` for NGINX Proxy Manager
   version: '3'
   services:
     app:
       image: jc21/nginx-proxy-manager:latest
       container_name: nginx-proxy-manager
       ports:
         - "80:80"
         - "443:443"
         - "81:81"  # Port for the web interface
       volumes:
         - ./data:/data
         - ./letsencrypt:/etc/letsencrypt
       environment:
         DB_SQLITE_FILE: "/data/database.sqlite"
       networks:
         - nginx_network
   networks:
     nginx_network:
        external: true 

3. Start NGINX Proxy Manager
sudo docker-compose up -d


4. Access Web GUI
Navigate to `http://your_vm_ip:81`.
Login with default credentials: 
Email: `admin@example.com`
Password: `changeme`

##### Configure Proxy Host

1. Add Proxy Host
Go to `Proxy Hosts` > `Add Proxy Host`.
Domain Name: `mydomain.local`.
Forward Hostname/IP: `web`.
Forward Port: `80`.
Enable `Websockets Support`.
SSL: Select the self-signed certificate you generated.
Click `Save`.

2. Set Up Domain in `/etc/hosts`
sudo nano /etc/hosts
your_vm_ip mywebsite.local —> (10.0.2.15 dandi.local)

   Add the following line
your_vm_ip mydomain.local

3. Test Access
   - Open your browser and navigate to `https://mydomain.local`.
