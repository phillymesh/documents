# phillymesh.net Kanban

*This is based off of tomesh's instructions located [here](https://github.com/tomesh/documents/blob/master/service_setup/wekan.md).

We are self-hosting [Wekan](https://github.com/wekan/wekan) to coordinate tasks. This page describes how the Wekan server is set up.

## Set Up Wekan Server

1. Spin up a Virtual Machine (e.g. [Vultr VPS](https://www.vultr.com/)) running **Debian 8 Jessie x64**.

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

1. Install Wekan with the [auto-install script](https://github.com/wekan/wekan-autoinstall/blob/master/autoinstall_wekan.sh):

	```
	# wget https://github.com/wekan/wekan-autoinstall/blob/master/autoinstall_wekan.sh
	# chmod +x autoinstall_wekan.sh
	# ./autoinstall_wekan.sh
	```

	Confirm Wekan is running on **http://YOUR_SERVER_IP:8080** if you have the port open. If you feel like risking it, continue to where we create the reverse proxy with nginx.

1. Point the `wekan.phillymesh.net` DNS A (and AAAA for IPv6) record to `YOUR_SERVER_IP`.

## Set up HTTPS with nginx and letsencrypt

1. Follow [these instructions](https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-14-04) to set up nginx and letsencrypt.

### Configure nginx

1. Create **/etc/nginx/sites-available/wekan.phillymesh.net**:

    ```
    server {
        listen 80;
        server_name wekan.phillymesh.net;

        location ~ /.well-known {
          allow all;
          root /usr/share/nginx/html;
        }
    }
    ```

1. Symlink **/etc/nginx/sites-available/wekan.phillymesh.net** to **/etc/nginx/sites-enabled**.

1. Redirect unexpected access to website by overwriting `/etc/nginx/sites-available/default` with:

  ```
  server {
    server_name .phillymesh.net;
    return 301 https://phillymesh.net;
  }
  ```

1. Reload nginx `service nginx reload`.

### Configure letsencrypt

1. Generate a Deffie-Hellman **.pem** and add the path to `ssl_dhparam` in nginx configurations.

1. Run the **letsencrypt-auto** script:

        ```
        # certbot-auto certonly --agree-tos --renew-by-default --email hello@phillymesh.net -a webroot --webroot-path=/usr/share/nginx/html -d wekan.phillymesh.net
        ```

1. After the cert is created, generate dhparem.pem if it doesn't exit
        ```
        openssl dhparam -out /etc/ssl/certs/dhparam.pem;
        ```

1. Lastly let's change our config to get it SSL forced by editing `/etc/nginx/sites-available/wekan.phillymesh.net` and pasting:

    ```
    upstream backend {
        server 127.0.0.1:80;
    }

    server {
        listen 80;
        server_name wekan.phillymesh.net;

        # Add headers to serve security related headers
        add_header X-Content-Type-Options nosniff;
        add_header X-Frame-Options "SAMEORIGIN";
        add_header X-XSS-Protection "1; mode=block";
        add_header X-Robots-Tag none;
        add_header X-Download-Options noopen;
        add_header X-Permitted-Cross-Domain-Policies none;

        location ~ /.well-known {
            allow all;
            root /usr/share/nginx/html;
        }

        location / {
            proxy_pass http://127.0.0.1:8080/;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            proxy_set_header Host $http_host;

            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forward-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forward-Proto http;
            proxy_set_header X-Nginx-Proxy true;

            proxy_redirect off;
        }
    }

    server {
        listen 443 ssl;
        server_name wekan.phillymesh.net;

        ssl_certificate /etc/letsencrypt/live/wekan.phillymesh.net/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/wekan.phillymesh.net/privkey.pem;
        ssl_trusted_certificate /etc/letsencrypt/live/wekan.phillymesh.net/fullchain.pem;

        ssl_session_timeout 1d;
        ssl_session_cache shared:SSL:50m;

        # Diffie-Hellman parameter for DHE ciphersuites, recommended 2048 bits
        # Generate with: openssl dhparam -out /etc/nginx/dhparam.pem 4048
        ssl_dhparam /etc/ssl/certs/dhparam.pem;

        # Mozilla "Intermediate configuration" copied from https://mozilla.github.io/server-side-tls/ssl-config-generator/
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
        ssl_ciphers 'ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-DSS-AES128-GCM-SHA256:kEDH+AESGCM:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA:ECDHE-ECDSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-DSS-AES128-SHA256:DHE-RSA-AES256-SHA256:DHE-DSS-AES256-SHA:DHE-RSA-AES256-SHA:ECDHE-RSA-DES-CBC3-SHA:ECDHE-ECDSA-DES-CBC3-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:AES:CAMELLIA:DES-CBC3-SHA:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!MD5:!PSK:!aECDH:!EDH-DSS-DES-CBC3-SHA:!EDH-RSA-DES-CBC3-SHA:!KRB5-DES-CBC3-SHA';
        ssl_prefer_server_ciphers on;

        # HSTS (ngx_http_headers_module is required) (15768000 seconds = 6 months)
        add_header Strict-Transport-Security max-age=15768000;

        # OCSP Stapling: fetch OCSP records from URL in ssl_certificate and cache them
        ssl_stapling on;
        ssl_stapling_verify on;

        # Add headers to serve security related headers
        add_header X-Content-Type-Options nosniff;
        add_header X-Frame-Options "SAMEORIGIN";
        add_header X-XSS-Protection "1; mode=block";
        add_header X-Robots-Tag none;
        add_header X-Download-Options noopen;
        add_header X-Permitted-Cross-Domain-Policies none;

        location ~ /.well-known {
            allow all;
            root /usr/share/nginx/html;
        }

        location / {
            proxy_pass http://127.0.0.1:80/;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            proxy_set_header Host $http_host;

            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forward-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forward-Proto http;
            proxy_set_header X-Nginx-Proxy true;

            proxy_redirect off;
        }
    }
    ```

1. Reload nginx `service nginx reload`.

1. Add the following to **crontab**:

	```
	30 2 * * 1 /opt/letsencrypt/letsencrypt-auto renew >> /var/log/le-renew.log
	35 2 * * 1 /bin/systemctl reload nginx
	```

## Configure Automated Emails

1. Stop the Wekan server `forever stop WEKAN_PROCESS_NO`.

1. In **/etc/init.d/wekan**, change `MAIL_URL` to mail server SMTP settings and add `MAIL_FROM` using our Zoho email service:

	```
	export MAIL_URL='smtp://noreply@phillymesh.net:PASSWORD@smtp.zoho.com:587/'
	export MAIL_FROM='noreply@phillymesh.net'
	```

1. Start Wekan server again `/etc/init.d/wekan start`.
