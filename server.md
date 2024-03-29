# Runbook for a server installation

## Base

**Note: All commands must be executed on the server as root (unless stated otherwise)!**

        export MY_SRV_ROOT=/srv

        export MY_DOMAIN=mydomain.tld
        export MY_USER_NAME=myuser

        export MY_EMAIL_USER=me@gmail.com
        export MY_EMAIL_PW=mypassword
        export MY_EMAIL_SMTP_HOST=smtp.gmail.com
        export MY_EMAIL_SMTP_PORT=587

        mkdir -p ${MY_SRV_ROOT}
        bash -c "$(curl -fsSL https://raw.github.com/x86dev/dotfiles/master/bin/dotfiles)" && source ~/.bashrc

## Edit host name

* Edit the following files:

        /etc/hostname
        /etc/hosts

* Reboot

## APT Stuff (for Debian and derivates)

        apt-get update && apt-get upgrade
        apt-get install -y apt-listchanges apt-transport-https net-tools ntp ntpdate ca-certificates curl htop podman podman-compose tmux git vim etckeeper mc fail2ban unattended-upgrades

## Backports (for Debian and derivates)

        sh -c "echo deb http://ftp.de.debian.org/debian jessie-backports main > /etc/apt/sources.list.d/jessie-backports.list"

## Automatic updates (for Debian and derivates)

        vi /etc/apt/apt.conf.d/50unattended-upgrades
        dpkg-reconfigure tzdata
        dpkg-reconfigure -plow unattended-upgrades

## SSH + Adding regular user for logging in

* On server, do:

        adduser ${MY_USER_NAME}
        su ${MY_USER_NAME}
        bash -c "$(curl -fsSL https://raw.github.com/x86dev/dotfiles/master/bin/dotfiles)" && source ~/.bashrc

* On your local box, do:

        ssh-keygen -t rsa
        ssh-copy-id -i id_file.pub ${MY_USER_NAME}@${MY_DOMAIN}

* On server, do:
  
        sed -i 's|[#]*Port .*|Port 2222|g' /etc/ssh/sshd_config
        sed -i 's|[#]*PasswordAuthentication yes|PasswordAuthentication no|g' /etc/ssh/sshd_config

* **First verify (important -- you could lock yourself out otherwise):** /usr/sbin/sshd -D -f /etc/ssh/sshd_config
* The just-added user needs to be used in order to log in via SSH; after logging in a ```sudo su -``` can be
  used to switch to the root user
* **Then**: ```service ssh restart```

## Podman

* adduser podman
* mkdir -p ${MY_SRV_ROOT}/containers/user/podman/storage
* chmod 0711 ${MY_SRV_ROOT}/podman/containers ${MY_SRV_ROOT}/podman/containers/user
* chmod -R 0700 ${MY_SRV_ROOT}/podman/containers/user/podman
* chown -R podman:podman ${MY_SRV_ROOT}/podman/containers/user/podman
* mkdir -p /home/podman/.config/containers/
* /home/podman/.config/containers/storage.conf:
`[storage]
driver = "overlay"
rootless_storage_path = "${MY_SRV_ROOT}/podman/containers/user/podman/storage"`
* chown podman:podman -R /home/podman/.config/containers/
* systemctl enable --user podman
* systemctl start --user podman
* systemctl --user enable --now podman.sock
* systemctl --user enable --now docker.sock
* loginctl enable-linger podman
* echo "net.ipv4.ip_unprivileged_port_start=80" >> /etc/sysctl.d/podman-privileged-ports.conf
* Edit /etc/containers/registries.conf:
`[registries.search]
registries = ['ghcr.io', 'docker.io']`

* Run as user: # su - podman (*not* the same as su podman)
* Test: podman run -it -p 80:80 docker.io/library/httpd

## Cron

ln -s ${MY_SRV_ROOT}/cron/daily /etc/cron.daily/
ln -s ${MY_SRV_ROOT}/monthly /etc/cron.monthly/

## Postfix

* Install with:

        apt-get install -y postfix mailutils libsasl2-2 ca-certificates libsasl2-modules

* Configure:

        cat <<EOT >> /etc/postfix/main.cf
        relayhost = [${MY_EMAIL_SMTP_HOST}]:${MY_EMAIL_SMTP_PORT}
        smtp_sasl_auth_enable = yes
        smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
        smtp_sasl_security_options = noanonymous
        smtp_tls_CAfile = /etc/ssl/certs/ca-certificates.crt
        smtp_use_tls = yes
        EOT

* Set authentication:

        cat <<EOT > /etc/postfix/sasl_passwd
        [${MY_EMAIL_SMTP_HOST}]:${MY_EMAIL_SMTP_PORT}    ${MY_EMAIL_USER}:${MY_EMAIL_PW}
        EOT

* Protect authentication:

        chmod 400 /etc/postfix/sasl_passwd
        postmap /etc/postfix/sasl_passwd

* Restart:

        /etc/init.d/postfix reload

## Logwatch (for weekly reports)

* Install:

        apt-get install -y logwatch

* Configure:

        mkdir -p /var/cache/logwatch
        cp /usr/share/logwatch/default.conf/logwatch.conf /etc/logwatch/conf/
        sed -i 's|Range = yesterday|Range = -7|g' /etc/logwatch/conf/logwatch.conf

        cat <<EOT > /etc/cron.weekly/00logwatch
            /usr/sbin/logwatch --output mail --mailto ${MY_EMAIL_USER} --detail high
            EOT

* Restart:

        /etc/init.d/cron restart

## Install Docker

### Kernel extras to enable docker aufs support

        apt-get -y install linux-image-extra-$(uname -r)

### Add Docker PPA and install latest version

        apt-key adv --keyserver hkp://p80.pool.sks-keyservers.net:80 --recv-keys 58118E89F3A912897C070ADBF76221572C52609D
        sh -c "echo deb https://apt.dockerproject.org/repo debian-jessie main > /etc/apt/sources.list.d/docker.list"
        apt-get update
        apt-get install -y apt-transport-https init-system-helpers docker-engine
        service docker start

### Install docker-compose

    sh -c "curl -L https://github.com/docker/compose/releases/download/1.8.2/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose"
    chmod +x /usr/local/bin/docker-compose
    sh -c "curl -L https://raw.githubusercontent.com/docker/compose/1.8.2/contrib/completion/bash/docker-compose > /etc/bash_completion.d/docker-compose"

## (Reverse) Proxy

    https://github.com/jwilder/nginx-proxy

    Configs:
        https://mozilla.github.io/server-side-tls/ssl-config-generator/

* Preparation:

        mkdir -p ${MY_SRV_ROOT}/nginx-proxy/vhost.d
        mkdir -p ${MY_SRV_ROOT}/nginx-proxy/html

* Run proxy:

        docker run -d -p 80:80 -p 443:443 \
            --restart=always \
            -v ${MY_SRV_ROOT}/letsencrypt/certs:/etc/nginx/certs:ro \
            -v /var/run/docker.sock:/tmp/docker.sock:ro \
            -v ${MY_SRV_ROOT}/nginx-proxy/vhost.d:/etc/nginx/vhost.d \
            -v ${MY_SRV_ROOT}/nginx-proxy/html:/usr/share/nginx/html \
            --label com.github.jrcs.letsencrypt_nginx_proxy_companion.nginx_proxy \
            --name nginx-proxy \
            jwilder/nginx-proxy

## Let's Encrypt Companion (for TLS certificates)

     https://github.com/JrCs/docker-letsencrypt-nginx-proxy-companion

* Preparation:

        mkdir -p ${MY_SRV_ROOT}/letsencrypt/certs

* Run:

        docker run -d \
                --restart=always \
                -v ${MY_SRV_ROOT}/letsencrypt/certs:/etc/nginx/certs:rw \
                -v /var/run/docker.sock:/var/run/docker.sock:ro \
                --volumes-from nginx-proxy \
                --name letsencrypt-nginx-proxy-companion \
                jrcs/letsencrypt-nginx-proxy-companion

## NextCloud

* See: https://github.com/nextcloud/docker

* Create volumes:

        docker volume create --name nextcloud-data
        docker volume create --name nextcloud-db-data
        docker volume create --name nextcloud-apps
        docker volume create --name nextcloud-config
        docker volume create --name nextcloud-themes

* Run NextCloud database:

        docker run -d \
            --name nextcloud-db \
            -v nextcloud-db-data:/var/lib/mysql \
            -e MYSQL_ROOT_PASSWORD=${MY_MYSQL_ROOT_PASSWORD} \
            -e MYSQL_DATABASE=nextcloud \
            -e MYSQL_USER=${MY_NEXTCLOUD_DB_USER} \
            -e MYSQL_PASSWORD=${MY_NEXTCLOUD_DB_PASSWORD} \
            mariadb:10

* Run NextCloud:

         docker run -d \
         --name nextcloud \
         --link nextcloud-db:db \
         --restart=always \
         -p 8888:80 \
         -e UID=1000 \
         -e GID=1000 \
         -e UPLOAD_MAX_SIZE=10G \
         -e APC_SHM_SIZE=128M \
         -e OPCACHE_MEM_SIZE=128 \
         -e CRON_PERIOD=15m \
         -e TZ=Berlin/UTC \
         -e MYSQL_DATABASE=nextcloud \
         -e MYSQL_USER=nextcloud \
         -e MYSQL_PASSWORD=${MY_NEXTCLOUD_DB_PASSWORD} \
         -e DB_HOST=db_nextcloud \
         -e VIRTUAL_HOST=nextcloud.${MY_DOMAIN} \
         -e VIRTUAL_PORT=8888 \
         -e LETSENCRYPT_HOST=nextcloud.${MY_DOMAIN} \
         -e LETSENCRYPT_EMAIL=webmaster@${MY_DOMAIN} \
         -v nextcloud-config:/var/www/html/config \
         -v nextcloud-data:/var/www/html/data \
         -v nextcloud-themes:/var/www/html/themes \
         nextcloud


## Tiny Tiny RSS (TTRSS)

    https://github.com/x86dev/docker-ttrss

* Let's Encrypt Challenge:

        echo 'location "/.well-known/acme-challenge" {
            default_type "text/plain";
            root ${MY_SRV_ROOT}/letsencrypt/acme-challenge;
        }' > ${MY_SRV_ROOT}/nginx-proxy/vhost.d/ttrss.${MY_DOMAIN}

* Run TTRSS:

        docker run -d -it \
            --restart=always \
            -e VIRTUAL_HOST=ttrss.${MY_DOMAIN} \
            -e VIRTUAL_PORT=8080 \
            -e TTRSS_SELF_URL=https://ttrss.${MY_DOMAIN} \
            -e TTRSS_URL=ttrss.${MY_DOMAIN} \
            -e TTRSS_PROTO=http \
            -e LETSENCRYPT_HOST=ttrss.${MY_DOMAIN} \
            -e LETSENCRYPT_EMAIL=webmaster@${MY_DOMAIN} \
            --link ttrss-data:db \
            --name ttrss \
            x86dev/docker-ttrss

## VPN (IPSEC)

* Setup:

        export MY_VPN_IPSEC_PSK=your_ipsec_pre_shared_key
        export MY_VPN_USER=your_vpn_username
        export MY_VPN_PASSWORD=your_vpn_password

* Load kernel module:

        modprobe af_key

* Run:

        docker run \
            --name ipsec-vpn-server \
            -e VPN_IPSEC_PSK=${MY_VPN_IPSEC_PSK} \
            -e VPN_USER=${MY_VPN_USER} \
            -e VPN_PASSWORD=${MY_VPN_PASSWORD} \
            --restart=always \
            -p 500:500/udp \
            -p 4500:4500/udp \
            -v /lib/modules:/lib/modules:ro \
            -d --privileged \
            hwdsl2/ipsec-vpn-server

* Check status

        docker exec -it ipsec-vpn-server ipsec status

## Gitea

* Setup:
        docker volume create gitea-repo

* Run:
        docker run -d -it \
            --name gitea \
            --restart=always \
            -v gitea-repo:/data \
            -e VIRTUAL_HOST=git.${MY_DOMAIN} \
            -e VIRTUAL_PORT=3000 \
            -e LETSENCRYPT_HOST=git.${MY_DOMAIN} \
            -e LETSENCRYPT_EMAIL=webmaster@${MY_DOMAIN} \
            gitea/gitea

* Configuration:
        vi /var/lib/docker/volumes/gitea-repo/_data/gitea/conf/app.ini
        docker restart gitea

---


## Certificates
    docker exec letsencrypt-nginx-proxy-companion /app/force_renew

## Always restart containers (on reboot / failures)
    docker ps | cut -f1 -d' ' | tail -n +2 | xargs sudo docker update --restart=always

## Backup / Restore
    docker run -v nextcloud-data:/volume -v /tmp:/backup loomchild/volume-backup backup nextcloud-data
    
## Cleanup (make sure you known what you're doing!)
    docker rm -v $(docker ps --filter status=exited -q 2>/dev/null) 2>/dev/null
    docker rmi $(docker images --filter dangling=true -q 2>/dev/null) 2>/dev/null

## Rescue
    ssh user@remote "dd if=/dev/sdX | gzip -1 -" | pv | dd of=image.gz
    /usr/bin/apt-clone clone --with-dpkg-repack /var/backups/ &> /dev/null

## Debugging
    docker run --rm -it <images> /bin/bash
    wget --no-check-certificate -p http://pydio.${MY_DOMAIN}
    docker cp nginx-proxy:/etc/nginx/conf.d/default.conf /tmp && less /tmp/default.conf

    supervisord:
        supervisorctl reload

    nginx:
        config test: nginx -t && service nginx reload
        404: nginx -g "pid /tmp/n.pid; daemon off; error_log /var/log/nginx/error.log debug;"

    ttrss:
        docker exec -it ttrss-data psql -U postgres
