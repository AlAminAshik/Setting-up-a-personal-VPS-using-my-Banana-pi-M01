# Setting-up-a-personal-VPS-using-my-Banana-pi-M1
This repo contains all the processes and steps required to connected a domain from dianahost with server as my bananpi M1. I will expose my Banana Pi M1 (Armbian Lite) to the public internet and point my domain to it so it behaves like a VPS.

**Pre-requisite**
*   The bananpi M1 has only dual core processor and 1GB RAM, not enough to run a full linux, so I used a CLI version of Armbian OS, that is very light.
*   OS (minimal) installed from Armbiarn website: https://www.armbian.com/bananapi/
*   The board has no on-board wifi chip so it has be constantly connected to ethernet port.
*   I already have a domain (alaminn.com) purchased from dianahost, and I will connect the server with this domain.
*   Additionally, I used a 16GB memory card as the storage of this server.


#Notes: Since my ISP uses CG-NAT (Bangladesh sigh!), I cannot directly connect my domain with bananapi and I require use cloudfare as a tunnel. If only I had public IPv4, it wouldnt be an issue. but guess what? cloudflare provides free SSL, DDoS protection, cache reserve and more for free. I see this as an absolute win!



## The steps are:

**Setting up Domain (I bought it from Dianahost):**
*   Go to cloudfare website.
*   add your domain that is hosted on another platform(dianahost).
*   Press onboard a domain, and it will give 2 nameservers.
*   Go to dianahost domain list and change all the nameservers to the new nameservers.
*   wait for some time, and you can verify whether the domain is live from the cloudfare website.
*   Done, DNS is now controlled by cloudfare.

**Setting up server on Bananapi (terminal)**

**Installing webserver:*
*   Install the OS on a microSD card using your computer and load onto the bananapi.
*   Connect mouse, keyboard, and monitor to the bananapi. Setup and login.
*   Install webserver by typing the following on the terminal "sudo apt install nginx"; this will also install "nginx-common";
*   check if nginx is properly installed by typing "nginx"; this may output several failed attempts but it means it is working.
*   Install cloudflared "wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-arm"
*   move to local bin "sudo mv cloudflared-linux-arm /usr/local/bin/cloudflared"
*   add super user executable permission "/usr/local/bin/cloudflared"
*   check if properly installed "cloudflared --version"
    
**Authenticate tunnel from bananpi with cloudfare:*
*   Once cloudflared in installed get the url "cloudflared tunnel login"
*   Open the url on the computer where cloudfare is logged in and authorize.
*   If authorize is successfull, the terminal output on bananapi will show success.

**Get tunnel ID and edit contents:*
*   get the Tunnel ID  "cloudflared tunnel create bananapi-tunnel"
*   create a tunnel configuraton file "nano ~/.cloudflared/config.yml"
*   Add the following lines:
          tunnel: YOUR-TUNNEL-ID
          credentials-file: /root/.cloudflared/YOUR-TUNNEL-ID.json
          ingress:
            - hostname: yourdomain.com
              service: http://localhost:80
            - service: http_status:404
*   Make sure to change both the "Your-tunnel-ID" and "yourdomain.com" without http.
*   Route DNS automatically "cloudflared tunnel route dns bananapi-tunnel alaminn.com", cloudflared will now automatically create the DNS record. **if you get error in this step, (1) login to cloudflare (2) go the DNS record of your domain (3) Go to records (4) and delete existing root domain record where 'name = @' and 'name = alaminn.com'

*   if you get error regarding "failed to create record.." go to cloudfare domain DNS > Records section, and delete existing root record that has your domain name directly, e.g. "name = alaminn.com"

**Run the Tunnel on bananapi:*
*   Run the tunnel: "cloudfare tunnel run bananapi-tunnel"
*   To stop the server, press ctrl+c.
*   Done, now it can be accessed your website from using mobile data. You will see an welcome nginx screen.

**Initialize Tunnel with startup:*
*   install a service "sudo cloudflared service install"
*   start the service "sudo systemctl start cloudflared"
*   enable the service "sudo systemctl enable cloudflared"
*   Check the status if running "systemctl status cloudflared". Now the tunnel will run automatically when the bananapi boots up. DONE.
*   The bananapi takes about 2-3min to start everything after it is powered up.

## Wordpress installation
**Install pre-requisites:*
*    I will now install wordpress, since I already have an existing portfolio website made using wordpress.
*    Install PHP and its modules: "sudo apt install php-fpm php-mysql php-curl php-gd php-mbstring php-xml php-zip php-intl"
*    Install database management system: "sudo apt install mariadb-server"
*    secure database: "sudo mysql_secure_installation". Set root password → YES, Remove anonymous users → YES, Disallow root remote login → YES, Remove test database → YES, Reload privileges → YES
*    Remember the root password you entered, this will be used to access the database.
*    Now create a new database by using mariadb. Enter "sudo mysql -u root -p", enter root password and write the following lines:
        * CREATE DATABASE mywordpress;
        * CREATE USER 'wpuser'@'localhost' IDENTIFIED BY 'StrongPassword123!';
        * GRANT ALL PRIVILEGES ON mywordpress.* TO 'wpuser'@'localhost';
        * FLUSH PRIVILEGES;
        * EXIT;
*    keep in mind, your database credentials are:
        * database name: mywordpress
        * user name: wpuser
        * database password: StrongPassword123!

**Install wordpress:*
*    On root terminal, go to directory: "cd /var/www/". Here the wordpress files will be downloaded.
*    make a directory named wordpress "sudo mkdir wordpress"
*    enter the directory "cd wordpress"
*    Download wordpress: "sudo wget https://wordpress.org/latest.tar.gz"
*    Extract the zip file: "sudo tar -xzf latest.tar.gz"
*    Adjust permission: "sudo chown -R www-data:www-data /var/www/wordpress"
*    Adjust permission: "sudo chmod -R 755 /var/www/wordpress"

**Setup server for wordpress:*
*    Before configuring find the php version: "php -v"
*    create a configuration file for alaminn.com: "sudo nano /etc/nginx/sites-available/alaminn.com". My php version was 8.4 so I used php8.4-fpm.sock;
`server {
    listen 80;
    server_name alaminn.com;
    root /var/www/wordpress/wordpress;
    index index.php index.html index.htm;
    location / {
        try_files $uri $uri/ /index.php?$args;
    }
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.4-fpm.sock;
    }
    location ~ /\.ht {
    deny all;
    }
}`
*   Enable the site from available: "sudo ln -s /etc/nginx/sites-available/alaminn.com /etc/nginx/sites-enabled/"
*   remove the default enabled site: "sudo rm /etc/nginx/sites-enabled/default". DONE
*   Test the server: "sudo nginx -t". This will output 2 success.
*   Finally, reload the server "sudo systemctl reload nginx".
*   Preferably give final permission check: "sudo chown -R www-data:www-data /var/www/wordpress"
*   Check the website from your browser: https://alaminn.com

**Taking manual wordpress backup:*
*   I already had a exsisting website made using wordpress, so I copied all the resources to this new server.
*   Keep the database name, password, and url exactly as the wp-config.php file.
*   Edit the wp-config.php file using "sudo nano cd /var/www/wordpress/wordpress/wp-config.php
*   If website do not load, maybe there is an issue with http and https, check both database and wp-config file


## Optimization
*   I converted all images into webp format.
*   Cloudfare gives free cache reserve which is very helpfull.
*   Removed unnecessary plugings and themes.
*   Home page need to be as light as possible.


Very important:
* I faced a issue where
     > the site url and home url from wordpress dashboard setting/mysql table could not be both https. the siteurl can be https but not the home url, otherwise I cannot login to wp-admin. changing both to http worked, but in that case website is unstable, pictures could not load and many issues prevailed since originally the website is https from cloudfare so there is a improper https detection. Even reinstalling mysql and wordpress did not solve. This is solved by:
     > solved by adding a condition on wp-config.php file. add this condition before the line /* That's all, stop editing! Happy publishing. */
         define('FORCE_SSL_ADMIN', true);
         if (strpos($_SERVER['HTTP_X_FORWARDED_PROTO'], 'https') !== false) {
             $_SERVER['HTTPS'] = 'on';
         } else {
             $_SERVER['HTTPS'] = 'off';
         }

## Basic uses of MariaDB
*   installing mariaDB: sudo apt install mariadb-server -y
*   secure database: sudo mysql_secure_installation. Set root password → YES, Remove anonymous users → YES, Disallow root remote login → YES, Remove test database → YES, Reload privileges → YES
*   opening mariadb: sudo mysql -u root -p
*   available databases: SHOW databases;

   *   creating a database:
        * CREATE DATABASE mywordpress;
        * CREATE USER 'wpuser'@'localhost' IDENTIFIED BY 'StrongPassword123!';
        * GRANT ALL PRIVILEGES ON mywordpress.* TO 'wpuser'@'localhost';
        * FLUSH PRIVILEGES;
        * EXIT;
     
   * showing tables: SHOW tables;
   * showing table content (i.e wordpress): SELECT option_name, option_value FROM wp_options WHERE option_name IN ('siteurl','home');
   * updating table content (i.e wordpress): UPDATE wpxz_options SET option_value='http://alaminn.com' WHERE option_name IN ('siteurl','home');
   * Vieweing all the users: SELECT User, Host FROM mysql.user;
   * removing a user: DROP USER 'wpuser'@'localhost';
