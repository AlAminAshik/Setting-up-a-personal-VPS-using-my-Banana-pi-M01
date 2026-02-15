# Setting-up-a-personal-VPS-using-my-Banana-pi-M01
This repo contains all the processes and steps required to connected a domain from dianahost with server as my bananpi M01. I will expose my Banana Pi M1 (Armbian Lite) to the public internet and point my domain to it so it behaves like a VPS.

#Notes: Since my ISP uses CG-NAT, I cannot directly connect my domain with bananapi and I require use cloudfare as a tunnel. If only I had public IPv4, it wouldnt be an issue.

The steps are:

**** Dianahost (Domain provider):
    *  Go to cloudfare website.
    *  add your domain that is hoster on another platform.
    *  Press onboard a domain, and it will give 2 nameservers.
    *  Go to dianahost domain list and change all the nameservers to the new nameservers.
    *  wait for some time, and you can verify whether the domain is live from the cloudfare website.
    *  Done, DNS is now controlled by cloudfare.

****  Bananapi (terminal)

**  Installing webserver  
    *  install webserver "sudo apt install nginx
    *  Install cloudflared "wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-arm"
    *  move to local bin "sudo mv cloudflared-linux-arm /usr/local/bin/cloudflared"
    *  add super user executable permission "/usr/local/bin/cloudflared"
    *  check if properly installed "cloudflared --version"
    
**  Authenticate tunnel from bananpi with cloudfare
    *  get the url "cloudflared tunnel login"
    *  Open the url on the computer where cloudfare is logged in and authorize.
    *  If authorize is successfull, the terminal output on bananapi will show success.

**  Get tunnel ID and edit contents
    *  get the Tunnel ID  "cloudflared tunnel create bananapi-tunnel"
    *  create a tunnel configuraton file "nano ~/.cloudflared/config.yml"
    *  Add the following lines:
          tunnel: YOUR-TUNNEL-ID
          credentials-file: /root/.cloudflared/YOUR-TUNNEL-ID.json
          ingress:
            - hostname: yourdomain.com
              service: http://localhost:80
            - service: http_status:404
    *  Make sure to change both the "Your-tunnel-ID" and "yourdomain.com" without http.
    *  Route DNS automatically "cloudflared tunnel route dns bananapi-tunnel alaminn.com"
"
    * if you get error regarding "failed to create record.." go to cloudfare domain DNS > Records section, and delete existing root record that has your domain name directly, e.g. "name = alaminn.com"

** Run the Tunnel on bananapi:
    *  cloudfare tunnel run bananapi-tunnel.
    *  Done, now it can be accessed your website from using mobile data.
