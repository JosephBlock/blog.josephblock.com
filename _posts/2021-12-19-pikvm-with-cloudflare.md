---
title:  "PiKVM with Cloudflare Tunnel and Access"
date:   2021-12-24 13:00:00 -0600
categories:
  - tutorials
tags:
  - pikvm
  - cloudflare
---

Have multiple PiKVM in your environment?  What about if your network is behind a carrier grade NAT? Don't want to open a port on your firewall? You can use Cloudflare's Tunnel to access your PiKVM from anywhere and get around any limitiations that your network may have.

## Setup
1. Log into PiKvm shell
2. Switch to root `su -`
3. Enable read-write `rw`
4. Update pacman `pacman -Syy`
5. Install Go `pacman -S go`

    ![pacman-go](/assets/tutorials/pikvm-cloudflare/image15.png)

6. Create a new certificate signed by Cloudflare [OpenSSL#Generate_an_RSA_private_key](https://wiki.archlinux.org/title/OpenSSL#Generate_an_RSA_private_key)
    ```
    openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out private.key
    openssl req -new -sha256 -key private.key -out crt.req
    ```
   ![OpenSSL#Generate_an_RSA_private_key](/assets/tutorials/pikvm-cloudflare/image20.png)

7. In SSL/TLS go to Origin server and click create certificate
     ![SSL/TLS](/assets/tutorials/pikvm-cloudflare/image26.png)
8. Choose Use my private key and csr and paste everything in crt.req
    ![csr](/assets/tutorials/pikvm-cloudflare/image6.png)
9. Click create and save the certificate on the server as public.crt
    ![ls](/assets/tutorials/pikvm-cloudflare/image11.png)
   
10. Update the PiKVM certificates [(Lets Encrypt and PiKVM)](https://github.com/pikvm/pikvm/issues/226#issuecomment-820549786)
    - Copy certs created to `/etc/kvmd/nginx/ssl/` (either rename old certs or use new names)
    ![copy certs](/assets/tutorials/pikvm-cloudflare/image27.png)
    - Update file permissions 
        ```
        cd /etc/kvmd/nginx/ssl/
        chown root:kvmd-nginx *
        chmod 440 private.key
        chmod 444 public.crt
        ```
        ![chmod](/assets/tutorials/pikvm-cloudflare/image29.png)
    - If you are using a custom certficate, update the `/etc/kvmd/nginx/ssl.conf` file
        ```
        sed -i 's/server.crt/public.crt/' /etc/kvmd/nginx/ssl.conf
        sed -i 's/server.key/private.key/' /etc/kvmd/nginx/ssl.conf
        sed -i 's/TLSv1.1 TLSv1//' /etc/kvmd/nginx/ssl.conf
        ```
        ![nginx config](/assets/tutorials/pikvm-cloudflare/image22.png)
    - restart kvmd-nginx 
    ```
    systemctl restart kvmd-nginx
    ```
    - Reload the PiKVM web page, certificate should now be signed by Cloudflare
    
        ![CF Signed Cert](/assets/tutorials/pikvm-cloudflare/image16.png)
11. Switch back to standard user `exit`
12. Build cloudflared, it's going to take a few minutes
    ```
    git clone https://aur.archlinux.org/cloudflared.git
    cd cloudflared/
    makepkg
    ```
    ![make cloudflared](/assets/tutorials/pikvm-cloudflare/image28.png)
13. Switch to root and install cloudflared 
    ```
    pacman -U <date>-armv7h.pkg.tar.xz
    ```
    ![pacman install](/assets/tutorials/pikvm-cloudflare/image19.png)
14. Setup tunnel [Cloudflare Tunnel Guide](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/install-and-setup/tunnel-guide)
    ```
    cloudflared tunnel login
    cloudflared tunnel create <name of tunnel to create>
    cloudflared tunnel route dns <tunnel id> <hostname>
    ```
    - You can see that the record is updated in the DNS settings

    ![cf login](/assets/tutorials/pikvm-cloudflare/image13.png)
    ![cf tunnel create](/assets/tutorials/pikvm-cloudflare/image8.png)
    ![cf tunnel route](/assets/tutorials/pikvm-cloudflare/image14.png)
    ![cf verify](/assets/tutorials/pikvm-cloudflare/image2.png)
15. Create config file at `/root/.cloudflared/config.yml`  
    ```
    url: https://localhost:443  
    tunnel: <tunnel id>  
    credentials-file: /root/.cloudflared/<tunnel id>.json  
    origin-server-name: <hostname>
    ```
    ![tunnel config](/assets/tutorials/pikvm-cloudflare/image17.png)
16. Test tunnel
    ```
    cloudflared tunnel run <tunnel id>
    ```
    - In any browser, go to hostname that was setup, PiKVM login page should now be accessible
    - `ctrl - c` to stop test

        ![tunnel test](/assets/tutorials/pikvm-cloudflare/image23.png)
        ![webpage](/assets/tutorials/pikvm-cloudflare/image1.png)
17. Once test is successful, update the service config file
    ```
    mv /etc/cloudflared/config.yml /etc/cloudflared/config.yml.original
    mv /root/.cloudflared/config.yml /etc/cloudflared/config.yml
    ```
    ![backup](/assets/tutorials/pikvm-cloudflare/image24.png)
    ![move tunnel config](/assets/tutorials/pikvm-cloudflare/image3.png)

18. Enable tunnel auto start 
    ```
    cloudflared service install
    systemctl enable cloudflared
    ```
19. Set PiKvm back to readonly `ro`

## To secure the setup even further, setup Cloudflare Access
1. Sign in or create an account for [Cloudflare for Teams](https://cloudflare.com/teams) and create a new team. Cloudflare provides 50 users free per team.
2. Select Access -> Application and click `Add an application`
    1. Select Self-hosted
        ![self hosted](/assets/tutorials/pikvm-cloudflare/image5.png)
    3. Enter an application name
    3. In application domain select the base domain and input the chosen subdomain in the first field
    4. Select configured identity providers, One-time pin is easiest to setup, but any option can be used if configured. Enable instant auth if only one provider is used
        ![configure application](/assets/tutorials/pikvm-cloudflare/image7.png)
    5. On next page, create the access policy  
            ![access policy](/assets/tutorials/pikvm-cloudflare/image18.png)
        - If you do not have a group, create one in the My Team section and add yourself as a member by email address.
            ![groups](/assets/tutorials/pikvm-cloudflare/image21.png)
    6. On the last setup page click Add application
        ![access successfully configured](/assets/tutorials/pikvm-cloudflare/image25.png)
    7. Access the hostname and you should now have a Cloudflare Access prompt
        ![access prompt](/assets/tutorials/pikvm-cloudflare/image4.png)
        ![setup complete](/assets/tutorials/pikvm-cloudflare/image9.png)