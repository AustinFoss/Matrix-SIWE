# **Matrix Synapse Home Server; Plus Element & SIWE**

## **Introduction**

This document is meant to provide a guide for installing a Matrix Synapse home server (hs) along side a web hosted Element client and using Sign-In With Ethereum (SIWE/siwe) as an OpenID Connect (OIDC/oidc) provider.

For the purposes of my own familiarity, these instructions were used with a Debian 11/Bullseye server. It may be updated in the future to be usable purely with Docker containers to be more universal accross all host platforms.

**Table of Contents**

1. [Overview](#overview)
1. [Secure the Server](#secure-the-server)
1. [Install Secondary Dependencies](#install-secondary-dependencies)
1. [Install Primary Components](#install-primary-components)

## **Overview**

Primary Open-Source Components:

- [SIWE](https://login.xyz/) - Docker
    - [OIDC Provider](https://docs.login.xyz/servers/oidc-provider)
- [Matrix](https://matrix.org)
    - [Protocol Specification](https://spec.matrix.org/latest/)
- [Synapse](https://matrix.org/docs/projects/server/synapse) - Matrix.org packages repository; Debian
    - [Installation Documentation](https://matrix-org.github.io/synapse/latest/setup/installation.html)
- [Element](https://element.io/) - Deployed Static Site
    - [Web hosted GitHub Repo](https://github.com/vector-im/element-web) 
- [Matrix-Appservice-Discord Bridge]() - Built From Source
    - [GitHub ]

It is expected that you have a domain registered to which you can assign the following sub-domains for use with Let's Encrypt:

- Default Home Page
    - \<your_domain>.com
- Synapse Home Server
    - matrix.\<your_domain>.com
- Element Client
    - element.\<your_domain>.com
- SIWE
    - siwe.\<your_domain>.com

Secondeary Open-Source Dependencies:

- [NGINX](https://www.nginx.com/) - Web Server
- Let's Encrypt
- PostgreSQL - Database (DB/db)
- Docker

## **Secure the Server**

This step is **optional** but please follow these steps if your planning to use this in production. 

1. SSH into your Debian Server.
    ```
    ssh root@<your_domain>.com
    ```
1. Begin by making sure everything is updated:
    ```
    sudo apt update && sudo apt upgrade
    ```
1. Create a non-root user, or use an existing one and add it to the sudo group.
    ```
    adduser <username> && \
    adduser <username> sudo
    ```
1. Setup SSH keys to connect to the server without a password.
    - Windows users using Putty: https://www.ssh.com/academy/ssh/putty/windows/puttygen
    - Linux terminal users: https://www.ssh.com/academy/ssh/keygen#creating-an-ssh-key-pair-for-user-authentication
1. Once you are able to sign in to the server with your non-root user over SSH without a password.
    - Uncomment or add the following lines to your `/etc/ssh/sshd_config` file.
        ```
        PasswordAuthentication no

        PermitRootLogin no
        ```
    - Delete the `/root/.ssh/authorized_keys` file if it was added during your VPS setup.
    - Restart SSH.
        ```
        sudo systemctl restart ssh
        ```
1. Install UFW firewall, Fail2Ban, Unattended Upgrades, jq, curl, & git.
    ```
    sudo apt install ufw fail2ban unattended-upgrades jq curl git && \
    sudo systemctl enable --now unattended-upgrades && \
    sudo systemctl enable --now fail2ban
    sudo ufw enable
    ```
1. Add SSH, HTTP, HTTPS, & 8448 rules. 8448 is used by Matrix to connect to other servers; not necessary for a private server but required for communication with other servers.
    ```
    sudo ufw allow ssh && \
    sudo ufw allow http && \ 
    sudo ufw allow https && \
    sudo ufw allow 8448
    ```
## **Install Secondary Components**

This step installs Let's Encrypt, NGINX, PostgreSQL, & Docker
1. Install all necessary signing keys.
    ```
    curl -fsSL https://nginx.org/keys/nginx_signing.key | \
    sudo gpg --dearmor -o /usr/share/keyrings/nginx.gpg && \

    curl -fsSL https://www.postgresql.org/media/keys/ACCC4CF8.asc | \
    sudo gpg --dearmor -o /usr/share/keyrings/psql.gpg && \

    curl -fsSL https://download.docker.com/linux/debian/gpg | \
    sudo gpg --dearmor -o /usr/share/keyrings/docker.gpg
    ```
1. Add all repositories to your sources list directory.
    ```
    echo deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/nginx.gpg] https://nginx.org/packages/debian bullseye nginx | \
    sudo tee /etc/apt/sources.list.d/nginx-stable.list && \

    echo deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/psql.gpg] http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main | \
    sudo tee /etc/apt/sources.list.d/psql.list && \

    echo deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker.gpg] \
    https://download.docker.com/linux/debian $(lsb_release -cs) stable | \
    sudo tee /etc/apt/sources.list.d/docker.list    
    ```
1. Update system with new repositories and install new packages.
    ```
    sudo apt update && \
    sudo apt install nginx-full postgres docker-ce docker-ce-cli containerd.io docker-compose-plugin certbot python3-certbot-nginx
    ```
1. Setup HTTPS traffic to the default NGINX landing page.
    ```
    sudo certbot --nginx -d <your_domain>.com
    ```
1. Confirm that NGINX is working by navigating to the public IP of your server in your browser and you should see HTTPS enabled in the top left corner next to the search bar.

## **Install Primary Components**

1. SIWE
1. Synapse
1. Element
1. Matrix Appservice Discord Bridge

### **1. SIWE**

SIWE provides a prebuilt docker image that gives us a very easy to use Docker image to start up our own OpenID Provider server in a container.

1. Clone the git repository. It is my own fork from [the original](https://github.com/spruceid/siwe-oidc), modified to restart on server reboot and made more obvious where you need to fill in your own domain.
    ```
    git clone https://github.com/AustinFoss/siwe-oidc.git
    ```
1. Modifiy the `SIWEOIDC_BASE_URL` variable in the `docker-compose.yml` file by filling in your own domain.
    ```
    SIWEOIDC_BASE_URL: "https://siwe.<your_domain>.com/"
    ```
1. Move the cloned repository to a more appropriate working directory.
    ```
    sudo mv siwe-oidc /opt/
    ```
1. Start the container. This step can take several minutes.
    ```
    sudo docker compose -f /opt/siwe-oidc/docker-compose.yml up -d
    ```
1. Create an NGINX configuration file for SIWE OIDC provider.
    ```
    sudo printf \
    "server {
        server_name siwe.<your_domain>.com;
    
        location / {
    
            #location = /register {
            #    allow xxx.xxx.xxx.xxx;
            #    deny all;
            #
            #    proxy_pass http://localhost:8000;
            #}
    
            #location = /.well-known/openid-configuration {
            #    allow xxx.xxx.xxx.xxx;
            #    deny all;
            #
            #    proxy_pass http://localhost:8000;
            #}
    
            proxy_pass http://localhost:8000;
    
        }
    }" > /etc/nginx/sites-available/siwe.<your_domain>.com
    ```
1. Enable the new configuration file by creating a symbolic link to it in the `/etc/nginx/sites-enabled/` directory, enable HTTPS with certbot, and restart NGINX.
    ```
    sudo ln -s /etc/nginx/sites-available/siwe.<your_domain>.com \ /etc/nginx/sites-enabled/ && \
    sudo certbot --nginx -d siwe.<your_domain>.com
    ```
1. You should now be able to see your SIWE page when navigating to `siwe.<your_domain>.com` in your browser.

### **2. Synapse**

### **3. Element**

### **4. Matrix Appservice Discord Bridge**
