# **Matrix Synapse Home Server; Plus Element & SIWE**

## **Introduction**

This document is meant to provide a guide for installing a Matrix Synapse home server (hs) along side a web hosted Element client and using Sign-In With Ethereum (SIWE/siwe) as an OpenID Connect (OIDC/oidc) provider.

For the purposes of my own familiarity, these instructions were used with a Debian 11/Bullseye server. It may be updated in the future to be usable purely with Docker containers to be more universal accross all host platforms.

**Table of Contents**

1. [Overview](#overview)
1. [Secure the Server](#secure-the-server)
1. [Install Secondary Dependencies](#install-secondary-dependencies)
1. [Install Primary Components](#install-primary-components)

---
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
    - \<your_domain>.\<tld>
- Synapse Home Server
    - matrix.\<your_domain>.\<tld>
- Element Client
    - element.\<your_domain>.\<tld>
- SIWE
    - siwe.\<your_domain>.\<tld>

Secondeary Open-Source Dependencies:

- [NGINX](https://www.nginx.com/)
- Let's Encrypt
- PostgreSQL
- Docker

---
## **Secure the Server**

This step is **optional** but please follow these steps if your planning to use this in production. 

1. SSH into your Debian Server.
    ```
    ssh root@<your_domain>.<tld>
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
---    
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
    sudo certbot --nginx -d <your_domain>.<tld>
    ```
1. Confirm that NGINX is working by navigating to \<your_domain>.\<tld> in your browser and you should see HTTPS enabled in the top left corner next to the search bar.
1. For PostgreSQL we will modify the default security settings to use SHA-256 instead of the default MD5 encryption.
    - Not entirely necessary but a good extra safety measure if you ever setup the DB to be accessed remotely.
    - Edit the `/etc/postgresql/<version_number>/main/postgresql.conf` file by uncommenting the following line, it should be located just above the SSL section.
    ```
    password_encryption = scram-sha-256     # scram-sha-256 or md5
    ```
1. Restart PostgreSQL.
    ```
    sudo systemctl restart postgresql
    ```
---
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
    SIWEOIDC_BASE_URL: "https://siwe.<your_domain>.<tld>/"
    ```
1. Edit you `/etc/hosts` file to include the following line.
    - For my Debian system this must done instead from the `/etc/cloud/templates/hosts.debian.tmpl` file if the change is to be made persistant.
    ```
    127.0.0.1 siwe-oidc
    ```
1. Move the cloned repository to a more appropriate working directory.
    ```
    sudo mv siwe-oidc /opt/
    ```
1. Start the container. This step can take several minutes.
    - This step can take several minutes the first time so feel free to open another terminal window and continue on while the containers are installed.
    ```
    sudo docker compose -f /opt/siwe-oidc/docker-compose.yml up -d
    ```
1. Create an NGINX configuration file for SIWE OIDC provider.
    - Replace \<your_domain>.\<tld> with the one you registered.
    - Note the commented out blocks. As part of the OIDC client registration process, covered later on in these instructions, their are two REST API endpoints that allow clients to use your new provider.
    - To prevent potential abuse we can limit who can register by replacing `XXX.XXX.XXX.XXX` with the public IP of the client.
        - If you end up using your provider for more clients you can add those to the list as long as the appear before `deny all;`
    - By default I have left this disabled. Uncomment each block once you have edited the `allow` addresses. You can do this at any time, just remember to run `sudo systemctl reload nginx` after making those changes.
    ```
    sudo printf \
    "server {
        server_name siwe.<your_domain>.<tld>;
    
        location / {
    
            #location = /register {
            #    allow XXX.XXX.XXX.XXX;
            #    deny all;
            #
            #    proxy_pass http://localhost:8000;
            #}
    
            #location = /.well-known/openid-configuration {
            #    allow XXX.XXX.XXX.XXX;
            #    deny all;
            #
            #    proxy_pass http://localhost:8000;
            #}
    
            proxy_pass http://localhost:8000;
    
        }
    }" > /etc/nginx/sites-available/siwe.<your_domain>.<tld>
    ```
1. Enable the configuration by creating the symbolic link and install the SSL certificates using Let's Encrypt.
    ```
    sudo ln -s /etc/nginx/sites-available/siwe.<your_domain>.<tld> \
    /etc/nginx/sites-enabled/ && \
    sudo certbot --nginx -d siwe.<your_domain>.<tld> && \
    sudo systemctl reload nginx
    ```
1. You should now be able to see your SIWE page when navigating to `siwe.<your_domain>.<tld>` in your browser.

### **2. Synapse**

Official installation instructions [here](https://matrix-org.github.io/synapse/latest/setup/installation.html#matrixorg-packages).

1. Update and install.
    ```
    sudo apt update && sudo apt install matrix-synapse-py3
    ``` 
1. Begin [configuring PostgreSQL](https://matrix-org.github.io/synapse/latest/postgres.html) switching to the postgres user.
    ```
    sudo -u postgres bash
    ```
1. Create the synapse role and synapse database the synapse homeserver will use.
    - You will be prompted to create a password for the new role, remember or write it down for later.
    ```
    createuser --pwprompt synapse && \
    createdb --encoding=UTF8 \
        --locale=C \
        --template=template0 \
        --owner=synapse \
        synapse
    ```
1. Edit the `/etc/postgresql/<version_number>/main/pg_hba.conf` file to allow your new role to access the new database.
    - This configuration will only allow for connections from the same host. Enabling remote connections will not be covered here, but note that you could run PostgreSQL and the Synapse home server on different hosts.
    ```
    # IPv4 local connections:
    host    all             all             127.0.0.1/32            scram-sha-256
    # The above line is present by default, add the new line below it but above the IPv6 connections
    host    synapse         synapse         ::1/128                 scram-sha-256

    # IPv6 local connections:
    ```
1. Edit your `/etc/matrix-synapse/homeserver.yaml` file to switch from the default SQLite to connect to the new `synapse` database. While doing this step we'll also enable other home servers to access our public rooms.
    ```
    #database:
    #  name: sqlite3
    #  args:
    #    database: /var/lib/matrix-synapse/homeserver.db
    database:
      name: psycopg2
      args:
      user: synapse
      password: <your_password>
      database: synapse
      host: localhost
      cp_min: 5
      cp_max: 10
    allow_public_rooms_over_federation: true
    ```
1. Restart PostgreSQL and Matrix
    ```
    sudo systemctl restart postgresql && \
    sudo systemctl restart matrix-synapse
    ```
1. To test that the Synapse home server connected successfully to the synapse database connect to it, you will be asked for the password.
    ```
    psql -h localhost -U synapse -W -d synapse
    ```
    - Then list all tables in the synapse database with `\dt`.
    - Once you've confirmed the tables are present, there should be a few dozen of them, exit the console by using `\q` twice. 
1. Create the NGINX configuration file for the Synapse home server.
    ```
    sudo printf \
    "server {
        server_name matrix.<your_domain>.<tld>;

        # For the federation port
        # Uncomment these lines after NGINX installs the SSL certificates
    #    listen 8448 ssl http2 default_server;
    #    listen [::]:8448 ssl http2 default_server;

        location / {
            # note: do not add a path (even a single /) after the port in `proxy_pass`,
            # otherwise nginx will canonicalise the URI and cause signature verification
            # errors.
            proxy_pass http://localhost:8008;
            proxy_set_header X-Forwarded-For $remote_addr;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header Host $host;

            # Nginx by default only allows file uploads up to 1M in size
            # Increase client_max_body_size to match max_upload_size defined in homeserver.yaml
            client_max_body_size 50M;
        }
    }" > /etc/nginx/sites-available/matrix.<your_domain>.<tld>
    ```
1. Enable the configuration by creating the symbolic link and install the SSL certificates using Let's Encrypt.
    ```
    sudo ln -s /etc/nginx/sites-available/matrix.<your_domain>.<tld> \
    /etc/nginx/sites-enabled/ && \
    sudo certbot --nginx -d matrix.<your_domain>.<tld> && \
    ```
    - Then go back and uncomment the two lines listening on ports 8448 to enable communication with other home servers (federation).
1. Reload NGINX with the new configuration file.
    ```
    sudo systemctl reload nginx
    ```
1. You should now be able to navigate to matrix.\<your_domain>.\<tld> in your browser and see the default landing page with HTTPS enabled. 
    - Additionally you can also try connecting to it with the Element desktop, mobile, or web apps if you edit the default home server it's configured for and replace it with your own matrix.\<your_domain>.\<tld> domain.

### **3. Element**

You can also host your own Element web client. This is beneficial for congestion reasons on the official Element web client hosted at [app.element.io](https://app.element.io/#/welcome). You can also customize it in various ways by hosting your own.

1. Locate the [latest release](https://github.com/vector-im/element-web/releases/latest) tar.gz file and download it to the `/var/www` directory with the following commands.
    ```
    curl -L https://github.com/vector-im/element-web/releases/download/v1.11.5/element-v1.11.5.tar.gz | \
    tar zx && \
    sudo mv element-* /var/www/element
    ```
1. Copy the sample config file to a new one.
    ```
    cp /var/www/element/config.sample.json /var/www/element/config.json
    ```
1. Edit the following lines in your new `config.json` file.
    - There are many more settings you can explore, but for now these two are all we need.
    ```
    "default_server_config": {
        "m.homeserver": {
            "base_url": "https://matrix.<your_domain>.<tld>",
            "server_name": "matrix.<your_domain>.<tld>"
        },
    ...
    ```
1. Create the NGINX configuration file for the Element web client.
    ```
    sudo printf \
    "server {
        server_name element.<your_domain>.<tld>;

        root /var/www/element;
        index index.html;

        location / {
            try_files $uri $uri/ =404;
        }

    }" > /etc/nginx/sites-available/element.<your_domain>.<tld>
    ```
1. Enable the configuration by creating the symbolic link and install the SSL certificates using Let's Encrypt.
    ```
    sudo ln -s /etc/nginx/sites-available/element.<your_domain>.<tld> \
    /etc/nginx/sites-enabled/ && \
    sudo certbot --nginx -d element.<your_domain>.<tld> && \
    sudo systemctl reload nginx
    ```
1. You should now be able to navigate to https://element.\<your_domain>.\<tld> in your browser and see the default landing page with HTTPS enabled.

There are no accounts yet created, and new registrations are disabled by default. You could enable this setting but we will do that in the next step using SIWE.

Alternatively, if you want to enable registrations temporarily to create your first admin user you can choose to do so, connect other OIDC providers, or make use of the [other options available](https://matrix-org.github.io/synapse/latest/usage/configuration/user_authentication/index.html).

### **4. Synapse & SIWE**

Now that all of the infrastructure is set up we can enable signing in with SIWE.

1. This is where we will make use of the two REST API endpoints mentioned previously. The first is to get the configuration information of the OIDC provider so that our client knows how to connect.
    - Not all of the information is required so this step is for your own knowledge.
    ```
    curl https://siwe.<domain>.<tld>/.well-known/openid-configuration | jq -r
    ```
1. The second API endpoint is the actual registration process where you will receive output specific for your Matrix Synapse home server.
    ```
    curl -X POST https://siwe.<domain>.<tld>/register \
        -H 'Content-Type: application/json' \
        -d '{"redirect_uris": ["https://matrix.<domain>.<tld>/_synapse/client/oidc/callback"]}' | \
        jq -r
    ```
1. Add the following to the end of your `/etc/matrix-synapse/homeserver.yaml` file, replacing all '<>' fields with your specific information.
    ```
    oidc_providers:
      - idp_id: siwe
        idp_name: Sign-In With Ethereum
    #    idp_icon: "" # For later
        issuer: "https://siwe.<your_domain>.<tld>/"
        authorization_endpoint: "https://siwe.<your_domain>.<tld>/authorize"
        token_endpoint: "https://siwe.<your_domain>.<tld>/token"
        userinfo_endpoint: "https://siwe.<your_domain>.<tld>/userinfo"
        jwks_uri: "https://siwe.<your_domain>.<tld>/jwk"
        registration_endpoint: "https://siwe.<your_domain>.<tld>/register"
        op_policy_uri: "https://siwe.<your_domain>.<tld>/legal/privacy-policy.pdf"
        op_tos_uri: "https://siwe.<your_domain>.<tld>/legal/terms-of-use.pdf"
        client_id: "<your_client_id>"
        client_secret: "<your_client_secret>"
        registration_access_token: "<your_registration_access_token>"
        registration_client_uri: "https://oidc.signinwithethereum.org/client/<your_client_id>"
        redirect_uris: ["https://matrix.labadappt.com/_synapse/client/oidc/callback"]
    ```
1. Restart the Matrix Synapse home server.
    ```
    sudo systemctl restart matrix-synapse
    ```
1. If you received no errors you should now be able to sign-in using SIWE to create your first admin user.
    - NOTE: Only MetaMask works with this configuration. Some API keys need to be added for the other options to function.
    
### **5. Matrix Appservice Discord Bridge**
