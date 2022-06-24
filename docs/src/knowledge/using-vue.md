# Strapi Deployment to VPS

## Create a VPS

Requirements:
- Ubuntu 18.04 x64
- 2GB RAM
::: tip
  The \$5/mo plan is currently unsupported as Strapi will not build with 1G of RAM. At the moment, deploying the Strapi Admin interface requires more than 1g of RAM. Therefore, a minimum standard Droplet of **\$10/mo** or larger instance is needed.
  :::
- Choose a `datacenter` region nearest your audience, for example, `New York`.
::: tip
  We recommend you `add your SSH key` for better security.
  :::

## Setup production server and install Node.js
Conntect to the server via Terminal:
```bash
ssh root@ip-address
```

### Install Node.js

- Create a `.npm-global` directory and set the path to this directory for `node_modules`

```bash
cd ~
mkdir ~/.npm-global
npm config set prefix '~/.npm-global'
```

- Create (or modify) a `~/.profile` file and add this line:

```bash
sudo nano ~/.profile
```

Add these lines.

```ini
# set PATH so global node modules install without permission issues
export PATH=~/.npm-global/bin:$PATH
```

- Lastly, update your system variables:

```bash
source ~/.profile
```

You are now ready to continue to the next section.

### Install and Configure Git versioning on your server

A convenient way to maintain your Strapi application and update it during and after initial development is to use [Git](https://git-scm.com/book/en/v2/Getting-Started-About-Version-Control). In order to use Git, you will need to have it installed on your Droplet. Droplets should have Git installed by default, so you will first check if it is installed and if it is not installed, you will need to install it.

The next step is to configure Git on your server.

#### 1. Check to see if `Git` is installed

If you see a `git version 2.x.x` then you do have `Git` installed. Check with the following command:

```bash
git --version
```

#### 2. **OPTIONAL:** Install Git.

::: tip
Only do this step if _not installed_, as above. Please follow these directions on [how to install Git on Ubuntu 18.04](https://www.digitalocean.com/community/tutorials/how-to-install-git-on-ubuntu-18-04).
:::

## Deploy from Github

### 1. Create a Repo

```bash
git init
git commit -m "first commit"
git branch -M main
git remote add origin https://github.com/your-name/your-repo.git
git push -u origin main
```

### 2. Clone Repo on VPS

You will next deploy your Strapi project to your Droplet by `cloning it from GitHub`.

From your terminal, `logged in as your non-root user` to your Droplet:

```bash
cd ~
git clone https://github.com/your-name/your-project-repo.git
```

Next, navigate to the `my-project` folder, the root for Strapi. You will now need to run `npm install` to install the packages for your project.

`Path: ./my-project/`

```bash
cd ./my-project/
npm install
NODE_ENV=production npm run build
```

Your Strapi project is now installed on your **Droplet**. You have a few more steps prior to being able to access Strapi and [create your first user](/developer-docs/latest/getting-started/quick-start.md).

You will next need to [install and configure PM2 Runtime](#install-and-configure-pm2-runtime).

## Install and configure PM2 Runtime

[PM2 Runtime](https://pm2.keymetrics.io) allows you to keep your Strapi project alive and to reload it without downtime.

Ensure you are logged in as a **non-root** user. You will install **PM2** globally:

```bash
npm install pm2@latest -g
```

### The ecosystem.config.js file

- You will need to configure an `ecosystem.config.js` file. This file will manage the **database connection variables** Strapi needs to connect to your database. The `ecosystem.config.js` will also be used by `pm2` to restart your project whenever any changes are made to files within the Strapi file system itself (such as when an update arrives from GitHub). You can read more about this file [here](https://pm2.keymetrics.io/docs/usage/pm2-doc-single-page/).

  - You will need to open your `nano` editor and then `copy/paste` the following:

```bash
cd ~
pm2 init
sudo nano ecosystem.config.js
```

- Next, replace the boilerplate content in the file, with the following:

```js
module.exports = {
  apps: [
    {
      name: 'strapi',
      cwd: 'repository-name',
      script: 'npm',
      args: 'start',
      env: {
        NODE_ENV: 'production',
      },
    },
  ],
};
```

Use the following command to start `pm2`:

```bash
cd ~
pm2 start ecosystem.config.js
```

Follow the steps below to have your app launch on system startup.

::: tip
These steps are modified from the DigitalOcean [documentation for setting up PM2](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-node-js-application-for-production-on-ubuntu-18-04#step-3-%E2%80%94-installing-pm2).
:::

- Generate and configure a startup script to launch PM2, it will generate a Startup Script to copy/paste, do so:

```bash
$ cd ~
$ pm2 startup systemd

[PM2] Init System found: systemd
[PM2] To setup the Startup Script, copy/paste the following command:
sudo env PATH=$PATH:/usr/bin /usr/lib/node_modules/pm2/bin/pm2 startup systemd -u your-name --hp /home/your-name
```

- Copy/paste the generated command:

```bash
$ sudo env PATH=$PATH:/usr/bin /usr/lib/node_modules/pm2/bin/pm2 startup systemd -u your-name --hp /home/your-name

[PM2] Init System found: systemd
Platform systemd

. . .


[PM2] [v] Command successfully executed.
+---------------------------------------+
[PM2] Freeze a process list on reboot via:
   $ pm2 save

[PM2] Remove init script via:
   $ pm2 unstartup systemd
```

- Next, `Save` the new PM2 process list and environment. Then `Start` the service with `systemctl`:

```bash
pm2 save

[PM2] Saving current process list...
[PM2] Successfully saved in /home/your-name/.pm2/dump.pm2

```

## Install NGINX
Since Nginx is available in Ubuntu’s default repositories, it is possible to install it from these repositories using the apt packaging system.

Since this may be your first interaction with the apt packaging system in this session, update the local package index so that you have access to the most recent package listings. Afterward, you can install nginx:
```bash
sudo apt update
sudo apt install nginx
```

#### Adjust the Firewall
You can check the current setting by running the following:
```bash
sudo ufw status
```
To let in HTTPS traffic allow the Nginx Full profile:
```bash
sudo ufw allow 'Nginx Full'
```
Now when you run the ufw status command it will reflect these new rules:
```bash
sudo ufw status
```
```bash
Output
Status: active

To                         Action      From
--                         ------      ----
OpenSSH                    ALLOW       Anywhere
Nginx Full                 ALLOW       Anywhere
OpenSSH (v6)               ALLOW       Anywhere (v6)
Nginx Full (v6)            ALLOW       Anywhere (v6)
```
Check with the systemd init system to make sure the service is running:
```bash
systemctl status nginx
```
In order for Nginx to serve  content, it’s necessary to create a server block with the correct directives. Instead of modifying the default configuration file directly, make a new one at /etc/nginx/sites-available/your_domain:
```bash
sudo nano /etc/nginx/sites-available/your_domain
```

Add the following configuration block, which is similar to the default, but updated for your new directory and domain name:
```bash
upstream strapi {
    server 127.0.0.1:1337;
}

server {
    # Listen HTTP
    listen 80;
    server_name api.example.com; # change here

    # Redirect HTTP to HTTPS
    return 301 https://$host$request_uri;
}

server {
    # Listen HTTPS
    listen 443 ssl;
    server_name api.example.com; # change here

    # Proxy Config
    location / {
        proxy_pass http://strapi;
        proxy_http_version 1.1;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Server $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Host $http_host;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
        proxy_pass_request_headers on;
    }
}

```


Same for sites-enabled:
```bash
sudo nano /etc/nginx/sites-enabled/your_domain
```
```bash
upstream strapi {
    server 127.0.0.1:1337;
}

server {
    # Listen HTTP
    listen 80;
    server_name api.example.com; # change here

    # Redirect HTTP to HTTPS
    return 301 https://$host$request_uri;
}

server {
    # Listen HTTPS
    listen 443 ssl;
    server_name api.example.com; # change here

    # Proxy Config
    location / {
        proxy_pass http://strapi;
        proxy_http_version 1.1;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Server $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Host $http_host;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
        proxy_pass_request_headers on;
    }
}

```

To avoid a possible hash bucket memory problem that can arise from adding additional server names, it is necessary to adjust a single value in the /etc/nginx/nginx.conf file. Open the file:
```bash
sudo nano /etc/nginx/nginx.conf
```
Find the server_names_hash_bucket_size directive and remove the # symbol to uncomment the line:
```bash
...
http {
    ...
    server_names_hash_bucket_size 64;
    ...
}
...
```
Save and close the file when you are finished.

Next, test to make sure that there are no syntax errors in any of your Nginx files:
```bash
sudo nginx -t
```
If there aren’t any problems, restart Nginx to enable your changes:
```bash
sudo systemctl restart nginx
```

## Install SSL
### Install Certbot
The first step to using Let’s Encrypt to obtain an SSL certificate is to install the Certbot software on your server.

The Certbot project recommends that most users install the software through snap, a package manager originally developed by Canonical (the company behind Ubuntu) and now available on many Linux distributions:
```bash
sudo snap install --classic certbot
```
Your output will display the current version of Certbot and successful installation:
```bash
Output

certbot 1.21.0 from Certbot Project (certbot-eff✓) installed
```

Next, create a symbolic link to the newly installed /snap/bin/certbot executable from the /usr/bin/ directory. This will ensure that the certbot command can run correctly on your server. To do this, run the following ln command. This contains the -s flag which will create a symbolic or soft link, as opposed to a hard link:
```bash
sudo ln -s /snap/bin/certbot /usr/bin/certbot
```
Certbot is now ready to use, but in order for it to configure SSL for Nginx, you need to verify some of Nginx’s configuration.

### Obtain an SSL Certificate
Certbot provides a variety of ways to obtain SSL certificates through plugins. The Nginx plugin will take care of reconfiguring Nginx and reloading the configuration whenever necessary. To use this plugin, run the following:
```bash
sudo certbot --nginx -d your_domain -d your_domain
```

This runs certbot with the --nginx plugin, using -d to specify the names you’d like the certificate to be valid for.

If this is your first time running certbot, you will be prompted to enter an email address and agree to the terms of service. After doing so, certbot will communicate with the Let’s Encrypt server to request a certificate for your domain. If successful, you will receive the following output:

```
Output

Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/your_domain/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/your_domain/privkey.pem
This certificate expires on 2022-01-27.
These files will be updated when the certificate renews.
Certbot has set up a scheduled task to automatically renew this certificate in the background.

Deploying certificate
Successfully deployed certificate for your_domain to /etc/nginx/sites-enabled/your_domain
Successfully deployed certificate for www.your_domain to /etc/nginx/sites-enabled/your_domain
Congratulations! You have successfully enabled HTTPS on https://your_domain and https://www.your_domain

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
If you like Certbot, please consider supporting our work by:
 * Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
 * Donating to EFF:                    https://eff.org/donate-le
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
```

### Verify Certbot Auto-Renewal
Let’s Encrypt’s certificates are only valid for ninety days. This is to encourage users to automate their certificate renewal process. The certbot package you installed takes care of this by adding a renew script to /etc/cron.d. This script runs twice a day and will automatically renew any certificate that’s within thirty days of expiration.

To test the renewal process, you can do a dry run with certbot:
```bash
sudo certbot renew --dry-run
```

Your `Strapi` project has been installed on a VPS using **Ubuntu 18.04**.