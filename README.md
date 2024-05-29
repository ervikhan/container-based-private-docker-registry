<h2>Setup Private Docker Registry (Container Based)</h2>

- **Create Container Registry**
    
    ```bash
    docker run -d -p 5000:5000 --restart=always --name registry -e REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY=/etc/docker/registry -v /etc/registry:/etc/docker/registry registry:2
    ```
    <i>you can customized port, and mounted directory</i>

- **Create UI for Private Docker Registry**
    
    ```bash
    docker run -d -p 81:80 -e DELETE_IMAGES=true -e REGISTRY_URL=http://103.246.107.127:5000 -e REGISTRY_TITLE="Docker Registry Server" --name registry-ui  joxit/docker-registry-ui:1.5-static
    ```
    <i>you can also customized port, and mounted directory</i>

- **Create Container Nginx**

    <i>this is for reverse proxy/port pointing</i>
    - Docker File
        
        ```bash
        ARG IMAGE_NAME=ubuntu
        ARG IMAGE_TAG=20.04
        
        FROM ${IMAGE_NAME}:${IMAGE_TAG}
        
        MAINTAINER ervikhan <m.ervikhan@gmail.com>
        LABEL service="Nginx-Latest"
        LABEL base_os="Ubuntu 20.04"
        
        #install nginx service
        RUN apt update
        RUN DEBIAN_FRONTEND=noninteractive apt -y install nginx
        
        #run nginx service
        CMD service nginx restart && /bin/bash
        ```
        
    - Build Image
        
        ```bash
        docker build -t nginx:ubuntu20.04 /lokasiDockerFile
        ```
        
    - Run Nginx
        
        ```bash
        docker run -d -t --name nginx -p 80:80 -p 443:443 --restart unless-stopped nginx:ubuntu20.04
        ```
        

    - **Configuration**

    - Unlink default nginx conf
        
        ```bash
        unlink /etc/nginx/sites-enabled/default
        ```
        
    - Make new file, example
        
        ```bash
        nano /etc/nginx/sites-available/reverse-proxy.conf
        ```
        
    - Fill with
        
        ```bash
        server{
        listen 80;
        listen 443 ssl;
        server_name YOUR_EXAMPLE_DOMAIN.COM;
        
        #LOG
        access_log /var/log/nginx/registry/access.log;
        error_log  /var/log/nginx/registry/error.log;
        
        #SSL Configuration
        include /etc/nginx/sites-available/nginx-ssl.conf;
        
        location / {
            #Configure which address the request is proxied to
            proxy_pass          http://YOUR_IP:81/;
        }
        }
        ```
        
    - `/etc/nginx/sites-available/nginx-ssl.conf`
        
        ```bash
        ssl_certificate /home/ssl/dinustek.pem; #ssl_certificate
        ssl_certificate_key /home/ssl/privatekey.pem; #ss_private_key
        ssl_session_cache shared:TLSSL:10m;
        ssl_protocols  TLSv1 TLSv1.1 TLSv1.2; # optional
        ssl_ciphers HIGH:!aNULL:!eNULL:!EXPORT:!CAMELLIA:!DES:!MD5:!PSK:!RC4;
        ```
        
    - Linking
        
        ```bash
        ln -s /etc/nginx/sites-available/reverse-proxy.conf /etc/nginx/sites-enabled/reverse-proxy.conf
        ```
        
    - Test
        
        ```bash
        nginx -t
        ```
        
    - Restart service
        
        ```bash
        service nginx restart
        ```
        

- **AUTHENTICATION**

    - Install service
        
        ```bash
        apt-get install apache2 apache2-utils
        ```
        
    - Create user and passwd
        
        ```bash
        htpasswd -c /etc/registry/auth/.htpasswd <usernam>
        ```
        
    - Add some configuration on nginx
        
        ```bash
        server{
        listen 80;
        listen 443 ssl;
        server_name YOUR_DOMAIN_EXAMPLE.COM;
        
        #LOG
        access_log /var/log/nginx/registry/access.log;
        error_log  /var/log/nginx/registry/error.log;
        
        #SSL Configuration
        include /etc/nginx/sites-available/nginx-ssl.conf;
        
        location / {
                #AUTH
            auth_basic "Registry Authentication";
            auth_basic_user_file /etc/registry/auth/.htpasswd;
            add_header 'Docker-Distribution-Api-Version' 'registry/2.0' always;
        
            #Configure which address the request is proxied to
            proxy_pass          http://YOUR_IP:81/;
        }
        }
        ```
        
    - Restart NGINX
        
        ```bash
        service nginx restart
        ```
**For docker compose version is coming soon**