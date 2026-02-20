# Setting-up-a-personal-VPS-using-my-Banana-pi-M01
This repo contains all the processes and steps required to connected a domain from dianahost with server as my bananpi M01. I will expose my Banana Pi M1 (Armbian Lite) to the public internet and point my domain to it so it behaves like a VPS.

**Pre-requisite**
*   The bananpi M1 has only dual core processor and 1GB RAM, not enough to run a full linux, so I used a CLI version of Armbian OS, that is very light.
*   The board has no on-board wifi chip so it has be constantly connected to ethernet port.
*   I already have a domain (alaminn.com) purchased from dianahost, and I will connect the server with this domain.
*   Additionally, I used a 16GB memory card as the storage of this server.


#Notes: Since my ISP uses CG-NAT (Bangladesh sigh!), I cannot directly connect my domain with bananapi and I require use cloudfare as a tunnel. If only I had public IPv4, it wouldnt be an issue.



## The steps are:

**Setting up Dianahost (Domain provider):**
*   Go to cloudfare website.
*   add your domain that is hoster on another platform.
*   Press onboard a domain, and it will give 2 nameservers.
*   Go to dianahost domain list and change all the nameservers to the new nameservers.
*   wait for some time, and you can verify whether the domain is live from the cloudfare website.
*   Done, DNS is now controlled by cloudfare.

**setting up server on Bananapi (terminal)**

**  Installing webserver  
*   install webserver "sudo apt install nginx"; this will also install "nginx-common";
*   check if nginx is properly installed by typing "nginx"; this may output several failed attempts but it means it is working.
*   Install cloudflared "wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-arm"
*   move to local bin "sudo mv cloudflared-linux-arm /usr/local/bin/cloudflared"
*   add super user executable permission "/usr/local/bin/cloudflared"
*   check if properly installed "cloudflared --version"
    
**  Authenticate tunnel from bananpi with cloudfare
*   get the url "cloudflared tunnel login"
*   Open the url on the computer where cloudfare is logged in and authorize.
*   If authorize is successfull, the terminal output on bananapi will show success.

**  Get tunnel ID and edit contents
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
*   Route DNS automatically "cloudflared tunnel route dns bananapi-tunnel alaminn.com"
"
*   if you get error regarding "failed to create record.." go to cloudfare domain DNS > Records section, and delete existing root record that has your domain name directly, e.g. "name = alaminn.com"

** Run the Tunnel on bananapi:
*   Run the tunnel: "cloudfare tunnel run bananapi-tunnel"
*   To stp the server, press ctrl+c.
*   Done, now it can be accessed your website from using mobile data.

** Initialize Tunnel with startup
*   install a service "sudo cloudflared service install"
*   start the service "sudo systemctl start cloudflared"
*   enable the service "sudo systemctl enable cloudflared"
*   Check the status if running "systemctl status cloudflared"



## Wordpress
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

## Basics of MariaDB
*   installing mariaDB: sudo apt install mariadb-server -y
*   secure database: sudo mysql_secure_installation. Set root password → YES, Remove anonymous users → YES, Disallow root remote login → YES, Remove test database → YES, Reload privileges → YES
*   opening mariadb: sudo mysql -u root -p
*   available databases: SHOW databases;

   *   creating a database:
         CREATE DATABASE mywordpress;
         CREATE USER 'wpuser'@'localhost' IDENTIFIED BY 'StrongPassword123!';
         GRANT ALL PRIVILEGES ON mywordpress.* TO 'wpuser'@'localhost';
         FLUSH PRIVILEGES;
         EXIT;
     
   * showing tables: SHOW tables;
   * showing table content (i.e wordpress): SELECT option_name, option_value FROM wp_options WHERE option_name IN ('siteurl','home');
   * updating table content (i.e wordpress): UPDATE wpxz_options SET option_value='http://alaminn.com' WHERE option_name IN ('siteurl','home');
   * Vieweing all the users: SELECT User, Host FROM mysql.user;
   * removing a user: DROP USER 'wpuser'@'localhost';
