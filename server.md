# Runbook for a server installation

## Base

        mkdir -p /srv
        

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
        apt-get --purge remove exim4-base

## Backports (for Debian 12 Bookworm)

        sh -c "echo deb http://deb.debian.org/debian bookworm-backports main > /etc/apt/sources.list.d/debian-backports.list"

## Automatic updates (for Debian and derivates)

        vi /etc/apt/apt.conf.d/50unattended-upgrades
        dpkg-reconfigure tzdata
        dpkg-reconfigure -plow unattended-upgrades

## SSH + Adding regular user for logging in

* On server, do:

        adduser ${MY_USER_NAME}
        usermod -aG sudo ${MY_USER_NAME}
        su ${MY_USER_NAME}
        bash -c "$(curl -fsSL https://raw.github.com/x86dev/dotfiles/master/bin/dotfiles)" && source ~/.bashrc

* On your local box, do:

        ssh-keygen -t rsa
        ssh-copy-id -i id_file.pub ${MY_USER_NAME}@${MY_DOMAIN}

* On server, do:
  
        sed -i 's|[#]*Port .*|Port 2222|g' /etc/ssh/sshd_config
        sed -i 's|[#]*PasswordAuthentication yes|PasswordAuthentication no|g' /etc/ssh/sshd_config
        sed -i 's|[#]*PermitRootLogin yes|PermitRootLogin no|g' /etc/ssh/sshd_config

* **First verify (important -- you could lock yourself out otherwise):**
  * On server (separate SSH daemon at port 2222): `/usr/sbin/sshd -D -f /etc/ssh/sshd_config`
  * Login on client via: `ssh -i id_file -p 2222 ${MY_USER_NAME}@${MY_DOMAIN}`
* The just-added user needs to be used in order to log in via SSH; after logging in a ```sudo su -``` can be
  used to switch to the root user
* **Then**: ```service ssh restart```

## Docker

 ```
 for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do sudo apt-get remove $pkg; done
 sudo install -m 0755 -d /etc/apt/keyrings
 sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
 sudo chmod a+r /etc/apt/keyrings/docker.asc
 echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
 apt-get update
 apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
 groupadd docker
 usermod -aG docker ${MY_USER_NAME}
 systemctl stop docker.service
 systemctl stop containerd.service
 mv /var/lib/docker ${MY_SRV_ROOT}/
 cat > /etc/docker/daemon.json <<EOF
{
    "data-root" : "${MY_SRV_ROOT}/docker" 
}
EOF
 systemctl enable docker.service
 systemctl enable containerd.service
 systemctl start docker
 docker info -f '{{ .DockerRootDir}}'
 ```
 
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

## (Reverse) Proxy

    https://github.com/jwilder/nginx-proxy

    Configs:
        https://mozilla.github.io/server-side-tls/ssl-config-generator/

* Preparation:

        mkdir -p ${MY_SRV_ROOT}/nginx-proxy/vhost.d
        mkdir -p ${MY_SRV_ROOT}/nginx-proxy/html

## Let's Encrypt Companion (for TLS certificates)

     https://github.com/JrCs/docker-letsencrypt-nginx-proxy-companion

* Preparation:

        mkdir -p ${MY_SRV_ROOT}/letsencrypt/certs

## NextCloud

* See: https://github.com/nextcloud/docker

* Create volumes:

        docker volume create --name nextcloud-data
        docker volume create --name nextcloud-db-data
        docker volume create --name nextcloud-apps
        docker volume create --name nextcloud-config
        docker volume create --name nextcloud-themes
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

## Gitea

* Setup:
        docker volume create gitea-repo

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
    wget --no-check-certificate -p http://myservice.${MY_DOMAIN}
    docker cp nginx-proxy:/etc/nginx/conf.d/default.conf /tmp && less /tmp/default.conf

    supervisord:
        supervisorctl reload

    nginx:
        config test: nginx -t && service nginx reload
        404: nginx -g "pid /tmp/n.pid; daemon off; error_log /var/log/nginx/error.log debug;"

    ttrss:
        docker exec -it ttrss-data psql -U postgres
