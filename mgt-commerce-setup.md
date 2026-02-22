# MGT-Commerce: Magento 2 + Nginx + Varnish + phpMyAdmin Architecture
**Architecture:** "Varnish Sandwich" (SSL offloading -> Cache -> Application)
**Stack:** Debian 12, Nginx, Varnish 7.1, PHP 8.3-FPM, Magento 2

## 1. Systemd Override Fix (Varnish Port Conflict)
Debian may use a hidden override file forcing Varnish to Port 8080, conflicting with the backend.
**Command to strip the conflicting port:**
`sudo sed -i 's/-a :8080//g' /etc/systemd/system/varnish.service`
`sudo systemctl daemon-reload`

## 2. Varnish VCL Configuration (`/etc/varnish/default.vcl`)
Direct Varnish to fetch from the internal Localhost Nginx backend.
```vcl
backend default {
    .host = "127.0.0.1";
    .port = "8080";
    .first_byte_timeout = 600s;
    .connect_timeout = 5s;
    .between_bytes_timeout = 2s;
}
3. The Complete Nginx Configuration (/etc/nginx/sites-available/magento)
Nginx
# 1. SSL Frontend (Accepts public traffic)
server {
    listen 443 ssl;
    server_name _; # Wildcard to accept any Elastic IP

    ssl_certificate /etc/nginx/ssl/magento.crt;
    ssl_certificate_key /etc/nginx/ssl/magento.key;

    # Bypass Varnish for phpMyAdmin
    location /phpmyadmin {
        proxy_pass [http://127.0.0.1:8080](http://127.0.0.1:8080);
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }

    # Send all normal Magento traffic to Varnish
    location / {
        proxy_pass [http://127.0.0.1:80](http://127.0.0.1:80); 
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }
}

# 2. Magento Backend (Varnish pulls from here)
server {
    listen 8080;
    server_name _; 
    root /var/www/html/magento/pub;
    index index.php;

    # phpMyAdmin execution block
    location ^~ /phpmyadmin {
        root /usr/share;
        index index.php index.html index.htm;
        
        location ~ ^/phpmyadmin/(.+\.php)$ {
            fastcgi_pass unix:/run/php/php8.3-fpm-test-ssh.sock;
            fastcgi_index index.php;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
            include fastcgi_params;
        }
    }

    # Magento main routing
    location / {
        try_files $uri $uri/ /index.php$is_args$args;
    }

    # Magento PHP execution (with buffer expansion)
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.3-fpm-test-ssh.sock;
        
        # Critical for Magento 2 to prevent 502 Bad Gateway
        fastcgi_buffer_size 128k;
        fastcgi_buffers 4 256k;
        fastcgi_busy_buffers_size 256k;
    }
}
4. phpMyAdmin Setup
Create a symlink to serve phpMyAdmin securely through the Nginx backend:
sudo apt install phpmyadmin -y
sudo ln -s /usr/share/phpmyadmin /var/www/html/magento/pub/phpmyadmin
Access securely via: https://[IP]/phpmyadmin/

5. Elastic IP Database Updates
If the server is rebuilt or the IP changes, update the Magento Base URL to match the new IP to prevent redirect loops:

Bash
sudo mysql -u root -p
SQL
USE magento;
UPDATE core_config_data SET value = 'https://[NEW_ELASTIC_IP]/' WHERE path = 'web/unsecure/base_url';
UPDATE core_config_data SET value = 'https://[NEW_ELASTIC_IP]/' WHERE path = 'web/secure/base_url';
EXIT;
Bash
sudo -u test-ssh /usr/bin/php8.3 /var/www/html/magento/bin/magento cache:flush

---

### The Final Steps
Once you have saved that file, you just need to run those Git commands we talked about to push it to your repository. 

Since it's nearly midnight, let's get this final push done so you can wrap up your Cloud Masters work for the week. 

**Were you able to paste, save, and run the `git push` successfully?**
