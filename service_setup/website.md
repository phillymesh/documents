# phillymesh.net

*This is based off of tomeshmesh's instructions located [here](https://github.com/tomesh/documents/blob/master/service_setup/wekan.md).*

We are self-hosting [phillymesh.net](https://phillymesh.net). This page describes how the server is set up

## Set Up The Virtual Server

1. Spin up a Virtual Machine (e.g. [RamNode VPS](https://ramnode.com)) running **Debian 8 Jessie x64**.

1. Secure your server and create non-root user with `sudo` by following [these instructions](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-14-04).

1. Get a root shell by logging in and as non-root user, then run `sudo -i`.

1. Enable **iptables** firewall by creating `/etc/iptables.rules`:

        ```
        *filter
        :INPUT ACCEPT [0:0]
        :FORWARD ACCEPT [0:0]
        :OUTPUT ACCEPT [46:5876]
        -A INPUT -i lo -j ACCEPT
        -A INPUT -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
        -A INPUT -p tcp -m tcp --dport 22 -j ACCEPT
        -A INPUT -p tcp -m tcp --dport 80 -j ACCEPT
        -A INPUT -p tcp -m tcp --dport 443 -j ACCEPT
        -A INPUT -j DROP
        COMMIT
        ```

1. Enable **ip6tables** by creating `/etc/ip6tables-rules`:

        ```
        *filter
        :INPUT ACCEPT [0:0]
        :FORWARD ACCEPT [0:0]
        :OUTPUT ACCEPT [0:0]
        -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
        -A INPUT -p ipv6-icmp -j ACCEPT
        -A INPUT -i lo -j ACCEPT
        -A INPUT -m conntrack --ctstate NEW -p tcp -m multiport --dports 22,80,443 -j ACCEPT
        -A INPUT -j REJECT --reject-with icmp6-port-unreachable
        -A FORWARD -j REJECT --reject-with icmp6-port-unreachable
        COMMIT
        ```

1. Reload the rules:

        ```
        # iptables-restore < /etc/iptables.rules
        # ip6tables-restore < /etc/ip6tables.rules
        ```

1. Create DNS records (A,AAAA) for phillymesh.net using the IPv4 and IPv4 addresses of this box.

## Install Apache, PHP, MariaDB

1. Make sure we have Apache 2 installed and run configtest.

```
apt-get install apache2
apache2ctl configtest
```

`configtest` will check to make sure a fully qualified domain name is configured. If not, it can be added to the apache config (https://www.digitalocean.com/community/tutorials/how-to-create-a-self-signed-ssl-certificate-for-apache-in-ubuntu-16-04)

```
nano /etc/apache2/apache2.conf
```

At the bottom of the file, add a line like the following:

```
ServerName example.com
```

1. Now, we will want to make sure mysql is uninstalled and install mariadb in its place:

```
apt-get remove mysql-server
apt-get install mariadb-server mariadb-client
```

Next, we will go through the secure installation process:

```
mysql_secure_installation
```

1. Now, to install php7 instead of php5, we have to update the sources list (http://unix.stackexchange.com/questions/252671/installing-php7-0-from-sid-on-jessie)

```
nano /etc/apt/sources.list
```

Paste the following at the bottom:

```
deb http://packages.dotdeb.org jessie all
```

After saving the file, we now need to add the gpg key from dotdeb.org, add it, and then update:

```
wget https://www.dotdeb.org/dotdeb.gpg
apt-key add dotdeb.gpg
apt-get update
```

Now we can install php7 and related packages (https://www.howtoforge.com/tutorial/install-apache-with-php-and-mysql-on-ubuntu-16-04-lamp/):

```
apt-get -y install php7.0 libapache2-mod-php7.0 php7.0-mysql php7.0-curl php7.0-gd php7.0-intl php-pear php7.0-imap php7.0-mcrypt php7.0-pspell php7.0-recode php7.0-sqlite3 php7.0-tidy php7.0-xmlrpc php7.0-xsl php7.0-mbstring php-gettext
```

1. Before restarting apache, edit the config file to make sure AllowOverride is enabled for <Directory /> and <Directory /var/www/>. Change the blocks to look like this and then save:

```
nano /etc/apache2/apache2.conf
```

```
<Directory />
        Options FollowSymLinks
        AllowOverride None
        Require all denied
</Directory>

<Directory /usr/share>
        AllowOverride None
        Require all granted
</Directory>

<Directory /var/www/>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
</Directory>

#<Directory /srv/>
#       Options Indexes FollowSymLinks
#       AllowOverride None
#       Require all granted
#</Directory>

```

Now we restart apache:

```
systemctl restart apache2
```

## Configure the Apache Virtual Host

1. Let's create a vhost file, `nano  nano /etc/apache2/sites-available/phillymesh.net.conf`

```
# Place any notes or comments you have here
# It will make any customization easier to understand in the weeks to come

# domain: phillymesh.net
# public: /var/www/phillymesh.net/

<VirtualHost *:80>

  # Admin email, Server Name (domain name) and any aliases
  ServerAdmin info@phillymesh.net
  ServerName phillymesh.net
  ServerAlias www.phillymesh.net


  # Index file and Document Root (where the public files are located)
  DirectoryIndex index.php
  DocumentRoot /var/www/phillymesh.net/public


  # Custom log file locations
  LogLevel warn
  ErrorLog  /var/www/phillymesh.net/log/error.log
  CustomLog /var/www/phillymesh.net/log/access.log combined
</VirtualHost>
```

1. Link the file to `sites-enabled` via `ln -s /etc/apache2/sites-available/phillymesh.net.conf /etc/apache2/sites-enabled`.

1. Restart apache via `systemctl restart apache2`.


## Getting a TLS cert with Let's Encrypt

1. Now, let's get the Let's Encrypt client up and running (https://www.digitalocean.com/community/tutorials/how-to-set-up-let-s-encrypt-certificates-for-multiple-apache-virtual-hosts-on-ubuntu-14-04):

```
cd /usr/local/sbin
wget https://dl.eff.org/certbot-auto
chmod a+x /usr/local/sbin/certbot-auto
```

Make sure that the A record(s) for the domain point to the ip address of the server.

1. Certificates for domains and subdomains can be generated easily:

```
certbot-auto --apache -d phillymesh.net -d www.phillymesh.net
```

All generated certificates are available in `/etc/letsencrypt/live` and Apache has automatically been updated and restarted.

1. Now, let's make sure these certificates are automatically renewed before the expire:

```
crontab -e
```

In the bottom of the crontab, paste the following line for a task that runs every monday at 2:30 AM:

```
30 2 * * 1 /usr/local/sbin/certbot-auto renew >> /var/log/le-renew.log
```

1. Finally, we need to make sure that the vhost configuration is done properly. 

```
cd /etc/apache2/sites-available
mv example.com.conf example.com-le.conf
cd ../sites-enabled
ln -s ../sites-available/example.com-le.conf example.com-le.conf
systemctl restart apache2
```

## Set up the Wordpress Database

1. We are using MariaDB, so let's dip down into the console with `mysql -uroot -p`.

2. Now we need to run a few commands to create the table, and grant it access to our urser;

```
MariaDB [(none)]> CREATE DATABASE phillymesh_net_blog
MariaDB [(none)]> GRANT ALL PRIVILEGES ON phillymesh_net_blog.* TO "phillymesh"@"127.0.0.1" IDENTIFIED BY "PASSWORDHERE";
MariaDB [(none)]> FLUSH PRIVILEGES;
```

Then quit the console with `exit`.

## Install Wordpress

1. First we need to create some directories in `/var/www`.

```
mkdir -p /var/www/phillymesh.net/{cgi-bin,log,private}
```

1. Change into the phillymesh.net directory with `cd /var/www/phillymesh.net`.

1. Download and unpack the latest wordpress release:

```
wget https://wordpress.org/latest.tar.gz
tar -xzf latest.tar.gz
mv wordpress/ public
```

1. Now let's give permission of our new site back to Apache;

```
chown -R www-data:www-data /var/www/phillymesh.net/
```

## Finish Wordpress Installation

1. Navigate to https://phillymesh.net to complete the installation, using the database details created earlier to link wordpress to our db.
