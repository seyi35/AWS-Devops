# 🛠️ DevOps Project 1: Deploying a WordPress on Apache Server with Vagrant (LAMP Stack)

## 🔍 Overview  
In this project, we provision a WordPress site using **Vagrant** and **Ubuntu (focal64)**. The environment is powered by the **LAMP stack** (Linux, Apache, MySQL, PHP).

---

## ⚙️ Prerequisites  
- [Vagrant](https://www.vagrantup.com/downloads)  
- [VirtualBox](https://www.virtualbox.org/wiki/Downloads)  
- Basic knowledge of Linux terminal

---

## 🧰 Step 1: Set Up Vagrant Environment

Open your terminal, let’s create a new project folder and initialize our Ubuntu environment:

```bash
mkdir wordpress
cd wordpress
vagrant init ubuntu/focal64
```

Now at this point a vagrant file has been created ,lets tweak your `Vagrantfile`. First run `cat Vagrantfile`  and edit your script like so:

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/focal64"
  config.vm.network "private_network", ip: "192.168.56.18"
  config.vm.network "public_network"
  config.vm.boot_timeout = 300
  
  #300 seconds =5mins
  
 #Set RAM to 2gb to the VM
  config.vm.provider "virtualbox" do |vb|
    vb.memory = "2048"
  end
```
### Summary of script
***Base Image***: Ubuntu 20.04 LTS (ubuntu/focal64)

***Networking:***

  - Private network with static IP 192.168.56.18 (host-only access)

  - Public network (bridged mode for LAN access)

***Boot Timeout:*** 5 minutes (300 seconds) to allow for slower VM startups

---
Next let's bring up the VM and ssh into it. On your terminal run:
```bash
Vagrant up && ssh

```

---

## ✅ Steps (Handled by provision.sh)

### Step 2: Installing LAMP Dependencies

Now switch to root user `sudo -i` and run the following commands to update packages and install everything WordPress needs:

```bash
# Update packages
sudo apt update

# Install Apache, MySQL, PHP, and required modules
sudo apt install apache2 ghostscript libapache2-mod-php mysql-server php php-bcmath php-curl php-imagick php-intl php-json php-mbstring php-mysql php-xml php-zip -y
```

You just brought the entire LAMP stack to life.🔥


### Step 3: Download & Install WordPress

Now let’s grab the latest version of WordPress and put it in the right place:

```bash
sudo mkdir -p /srv/www
sudo chown www-data: /srv/www
curl https://wordpress.org/latest.tar.gz | sudo -u www-data tar zx -C /srv/www
```

We’re staging WordPress in `/srv/www` so it’s ready for Apache to serve.

---

### 🌐 Step 4: Configure Apache for WordPress

Set up a virtual host for WordPress so it’s properly served from Apache. First open the Apache conf file in vim. 

```bash
vim /etc/apache2/sites-available/wordpress.conf
```
Then paste the following content

```apache

<VirtualHost *:80>
    DocumentRoot /srv/www/wordpress
    <Directory /srv/www/wordpress>
        Options FollowSymLinks
        AllowOverride Limit Options FileInfo
        DirectoryIndex index.php
        Require all granted
    </Directory>
    <Directory /srv/www/wordpress/wp-content>
        Options FollowSymLinks
        Require all granted
    </Directory>
</VirtualHost>

```
Save and close the file: `Esc` `:wq`

Next enable the site and modules, and reload Apache:

```apache
# Enable the new site and modules
sudo a2ensite wordpress
sudo a2enmod rewrite
sudo a2dissite 000-default

# Apply changes
sudo systemctl reload apache2
```

And yeah, Apache is ready with WordPress! 🌐

---


### Step Step 5: Configure MySQL

Let’s set up the database and user that WordPress will use:

Connect to MySQL:
```bash
sudo mysql -u root
```
Run these commands to create a database and user for WordPress (replace admin123 with your own password):

```bash
mysql -u root -e 'CREATE DATABASE wordpress;'
mysql -u root -e 'CREATE USER wordpress@localhost IDENTIFIED BY "admin123";'
mysql -u root -e 'GRANT SELECT,INSERT,UPDATE,DELETE,CREATE,DROP,ALTER ON wordpress.* TO wordpress@localhost;'
mysql -u root -e 'FLUSH PRIVILEGES;'
```
 Exit MySQL `quit;`

You’ve just given WordPress its own playground in MySQL. 🎯

---

### 🧩 Step 6: Connect WordPress to the Database

Now, let’s wire up the `wp-config.php`:

```bash
sudo -u www-data cp /srv/www/wordpress/wp-config-sample.php /srv/www/wordpress/wp-config.php
sudo -u www-data sed -i 's/database_name_here/wordpress/' /srv/www/wordpress/wp-config.php
sudo -u www-data sed -i 's/username_here/wordpress/' /srv/www/wordpress/wp-config.php
sudo -u www-data sed -i 's/password_here/admin123/' /srv/www/wordpress/wp-config.php
```

🔐 **Important:** Replace the secret keys with real secure ones from this link:  
[WordPress Secret Keys](https://api.wordpress.org/secret-key/1.1/salt/)

Paste them into `wp-config.php` to boost your site's security. 🛡️

```bash
sudo -u www-data vim /srv/www/wordpress/wp-config.php
```

Ensure to save and close.

### Step 7: Final Setup via Browser

1. To set up on the browser we need to get the Ip Address by running `ip addr show`

2. We will be using the public Ip to access it on the web. Complete the WordPress installer:  
 - Choose a site title  
 - Create an admin username and password  
 - Enter your email  
 - And click Install! 🎉
---

## ✅ Done!

You’ve successfully automated the deployment of a full LAMP-based WordPress site using Vagrant! 🙌

🛠️ **Skills Gained**

-  VM Provisioning: Used Vagrant to create and configure an Ubuntu virtual machine.
-  LAMP Stack Setup: Installed Apache, MySQL, and PHP to run web apps.
-  WordPress Deployment: Set up WordPress manually on a local server.
-  Apache Virtual Hosts: Configured Apache to point to the correct website folder.
-  MySQL Setup: Created a secure database and user for WordPress.
-  Security: Secured WordPress with unique keys in the config file.
-  Local Environment: Learned how to build and manage local dev setups.


so yeah keep experimenting, break things, and build again - that’s the DevOps way 💥
