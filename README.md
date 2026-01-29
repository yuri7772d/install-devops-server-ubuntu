- docker.io
- portainer
- caddy
- registry
- gitlab
- gitlab-runne
  ---
# setup ubuntu


```bash

sudo apt update && sudo apt upgrade -y

```

อนุญาตการเชื่อมต่อที่จำเป็น
```bash
sudo ufw allow OpenSSH     # Port 22 สำหรับ SSH
sudo ufw allow 80/tcp       # Port 80 สำหรับ HTTP
sudo ufw allow 443/tcp      # Port 443 สำหรับ HTTPS

sudo ufw allow 2222/tcp     # สำหรับ GitLab SSH
sudo ufw allow 5000/tcp     # สำหรับ Docker Registry
sudo ufw allow 8080/tcp     # สำหรับ Registry UI
sudo ufw allow 9443/tcp     # สำหรับ Portainer

```
---
# docker

## install เเบบ ธรรมดาง่าย

```bash
apt install docker.io -y

```

## เเบบ full servion  (เเนะนำ)
- Uninstall เวอร์ชั่นเก่าออก
```bash

for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt-get remove $pkg; done
```
- อัปเดต Package Index ติดตั้ง Package ที่จำเป็น และ เพิ่ม Docker’s official GPG key:
```bash

sudo apt-get update
sudo apt-get install ca-certificates curl gnupg lsb-release
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

- ตั้งค่า Repository:
```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

```

- ติดตั้ง Docker Engine:
```bash
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y

```

ตรวจสอบการทำงาน Service Docker และให้ทำงานอัตโนมัติทุกครั้งที่ Reboot
```bash
sudo systemctl status docker
sudo systemctl enable docker

```

---




# portainer

```bash
docker volume create portainer_data

docker run -d -p 9443:9443 --name portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce:latest

```
---

# caddy

```bash
sudo mkdir -p /opt/caddy

sudo nano /opt/caddy/Caddyfile

```

ตัวอย่าง Caddyfile

```bash
{
	# สั่งให้ Caddy ใช้ CA ภายในสำหรับทุกโดเมนในไฟล์นี้
	cert_issuer internal
	# auto_https on  # เปิดใช้งาน HTTPS อัตโนมัติ (เป็นค่าเริ่มต้นอยู่แล้ว)
}

portainer.vecskill.ovec {
	tls internal
	encode zstd gzip
	reverse_proxy https://portainer:9443 {
		transport http {
			tls_insecure_skip_verify # จำเป็นเพราะ Certificate ของ Portainer เป็น Self-signed
		}
	}
}

registry.vecskill.ovec {
	tls internal
	encode zstd gzip
	header Docker-Distribution-Api-Version "registry/2.0" # Header เฉพาะของ Registry
	reverse_proxy http://registry:5000 {
		header_up X-Forwarded-Proto {scheme}
	}
}

registry-ui.vecskill.ovec {
	tls internal
	encode zstd gzip
	reverse_proxy http://registry-ui:80 {
		header_up X-Forwarded-Proto {scheme}
	}
}

gitlab.vecskill.ovec {
	tls internal
	encode zstd gzip
	header Strict-Transport-Security "max-age=31536000;"
	log {
		output file /var/log/caddy/gitlab_access.log
	}
	reverse_proxy http://gitlab:80 {
		header_up X-Forwarded-Proto {scheme}
	}
}


```

docker compose

```bash
services:
  caddy:
    image: caddy:latest # ใช้เวอร์ชันล่าสุด หรือระบุเวอร์ชัน 2.x.x
    container_name: caddy
    restart: unless-stopped
    networks:
      - app_net
    ports:
      - "80:80"       # สำหรับ Redirect HTTP -> HTTPS
      - "443:443"     # HTTPS หลัก
      - "443:443/udp" # HTTP/3 (ถ้าต้องการ)
    volumes:
      # จุดสำคัญ: Mount Caddyfile จาก Host เข้าไปเป็น Read-Only
      - /opt/caddy/Caddyfile:/etc/caddy/Caddyfile:ro 
      # Volume สำหรับเก็บ Certificate และข้อมูลอื่นๆ
      - caddy_data:/data 
      # Volume สำหรับเก็บ Config และ Root CA
      - caddy_config:/config 
      # Volume สำหรับเก็บ Log (สร้างโฟลเดอร์ /var/log/caddy บน Host ก่อน)
      - /var/log/caddy:/var/log/caddy 
    environment:
      - TZ=Asia/Bangkok # ตั้งค่า Timezone (แนะนำ)

networks:
  app_net:
    external: true
    
volumes:
  caddy_data:
    external: true
  caddy_config:
    external: true


```

curtificate

```bash
docker cp caddy:/data/caddy/pki/authorities/local/root.crt /opt/caddy/caddy-root.crt
sudo chmod 644 /opt/caddy/caddy-root.crt

scp bncc@192.168.100.3:/opt/caddy/caddy-root.crt C:/Users/Public

# import on windo
Import-Certificate -FilePath "C:\Users\Public\caddy-root.crt" -CertStoreLocation Cert:\LocalMachine\Root

# ของ devops server 
sudo cp /opt/caddy/caddy-root.crt /usr/local/share/ca-certificates/caddy-root.crt

sudo update-ca-certificates

# ของ production server

scp bncc@192.168.100.3:/opt/caddy/caddy-root.crt /opt/caddy-root.crt

sudo cp /opt/caddy-root.crt /usr/local/share/ca-certificates/caddy-root.crt

sudo update-ca-certificates

```
---

# registry

```bash
networks:
  app_net:
    external: true

volumes:
  registry_data:
    external: true

services:
  registry:
    image: registry:2
    container_name: registry
    restart: always
    ports:
      - "5000:5000"
    environment:
      REGISTRY_STORAGE_DELETE_ENABLED: 'true'
    networks:
      - app_net
    volumes:
      - registry_data:/var/lib/registry

  registry-ui:
    image: joxit/docker-registry-ui:main
    container_name: registry-ui
    restart: always
    networks:
      - app_net
    ports:
      - "8080:80"
    environment:
      - SINGLE_REGISTRY=true
      - REGISTRY_TITLE=Docker Registry UI
      - DELETE_IMAGES=true
      - SHOW_CONTENT_DIGEST=true
      - NGINX_PROXY_PASS_URL=http://registry:5000
      - SHOW_CATALOG_NB_TAGS=true
      - CATALOG_MIN_BRANCHES=1
      - CATALOG_MAX_BRANCHES=1
      - TAGLIST_PAGE_SIZE=100
      - REGISTRY_SECURED=false
      - CATALOG_ELEMENTS_LIMIT=1000
    depends_on:
      - registry


```
ตั้งค่า Insecure Registry ใน docker deamon
- devops-server
- production-server
- dev
  
```bash

#เข้าไปใน daemon.json
nano /etc/docker/daemon.json

{
  "insecure-registries" : ["registry.vecskill.ovec", "registry-ui.vecskill.ovec","192.168.100.110:5000","192.168.100.110:8080"]
}

#restart docker
sudo systemctl restart docker

```
## pull push registry image ที่จำเป็น
```bash
# ===============================
# Docker Registry: 192.168.100.180:5000
# ===============================

# ---------- library/node ----------
docker pull node:20
docker tag node:20 192.168.100.180:5000/library/node:20
docker push 192.168.100.180:5000/library/node:20


# ---------- library/nginx ----------
docker pull nginx:latest
docker tag nginx:latest 192.168.100.180:5000/library/nginx:latest
docker push 192.168.100.180:5000/library/nginx:latest


# ---------- library/alpine ----------
docker pull alpine:latest
docker tag alpine:latest 192.168.100.180:5000/library/alpine:latest
docker push 192.168.100.180:5000/library/alpine:latest


# ---------- library/mysql ----------
docker pull mysql:8
docker tag mysql:8 192.168.100.180:5000/library/mysql:8
docker push 192.168.100.180:5000/library/mysql:8


# ---------- library/phpmyadmin ----------
docker pull phpmyadmin:latest
docker tag phpmyadmin:latest 192.168.100.180:5000/library/phpmyadmin:latest
docker push 192.168.100.180:5000/library/phpmyadmin:latest


# ---------- gitlab/gitlab-runner ----------
docker pull gitlab/gitlab-runner:latest
docker tag gitlab/gitlab-runner:latest 192.168.100.180:5000/gitlab/gitlab-runner:latest
docker push 192.168.100.180:5000/gitlab/gitlab-runner:latest


# ---------- portainer/portainer-ce ----------
docker pull portainer/portainer-ce:latest
docker tag portainer/portainer-ce:latest 192.168.100.180:5000/portainer/portainer-ce:latest
docker push 192.168.100.180:5000/portainer/portainer-ce:latest

# ---------- library/node:20-alpine ----------
docker pull node:20-alpine
docker tag node:20-alpine localhost:5000/library/node:20-alpine
docker push localhost:5000/library/node:20-alpine

# ---------- library/docker:26-dind ----------
docker pull docker:26-dind
docker tag docker:26-dind localhost:5000/library/docker:26-dind
docker push localhost:5000/library/docker:26-dind

# ---------- library/docker:26 ----------
docker pull docker:26
docker tag docker:26 localhost:5000/library/docker:26
docker push localhost:5000/library/docker:26

# ---------- library/alpine-ssh:latest ----------
docker pull panubo/sshd:latest
docker tag panubo/sshd:latest localhost:5000/library/alpine-ssh:latest
docker push localhost:5000/library/alpine-ssh:latest

docker pull mysql:8.0
docker tag mysql:8.0 localhost:5000/library/mysql:8.0
docker push localhost:5000/library/mysql:8.0
```
---


# gitlab
compose

```bash
services:
  gitlab:
    image: 'gitlab/gitlab-ce:latest'
    container_name: gitlab
    restart: always
    hostname: 'gitlab.vecskill.ovec'
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'https://gitlab.vecskill.ovec'
        nginx['listen_port'] = 80
        nginx['listen_https'] = false
        letsencrypt['enable'] = false
        gitlab_rails['gitlab_shell_ssh_port'] = 2222
        # เก็บ IP จริงเมื่อมี reverse proxy
        nginx['real_ip_header'] = 'X-Forwarded-For'
        nginx['real_ip_recursive'] = 'on'
        nginx['real_ip_trusted_addresses'] = ['172.16.0.0/12','192.168.0.0/16','10.0.0.0/8']
    ports:
      # - '80:80'
      # - '443:443'
      - '2222:22'
    shm_size: "512m"
    volumes:
      - 'gitlab_config:/etc/gitlab'
      - 'gitlab_logs:/var/log/gitlab'
      - 'gitlab_data:/var/opt/gitlab'
    healthcheck:
      test: ["CMD-SHELL", "curl -fsS http://localhost/-/health || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 10
    networks:
      - app_net

networks:
  app_net:
    external: true

volumes:
  gitlab_config:
    external: true
  gitlab_logs:
    external: true
  gitlab_data:
    external: true

```

password gitlab

```bash
หรือ ไปเอาใน container
cat /etc/gitlab/initial_root_password


```
---
# gen ssh-key
 - get ที่เครื่อง devops server
 - นำ Public Key ไปใส่ที่เครื่อง Production
   
```bash
#gen ssh-key
ssh-keygen -t ed25519

#ดู public key
cat ~/.ssh/id_ed25519.pub

# วาง public key ที่นี้ ในเครื่อง production-server
nano ~/.ssh/authorized_keys

#ดู private key
cat ~/.ssh/id_ed25519

```
---
# gitlab-runner 
# native

```bash
curl -fsSL https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh | sudo bash

sudo apt-get install -y gitlab-runner

```

```bash
volumes = ["/cache","/var/run/docker.sock:/var/run/docker.sock"]
environment = ["DOCKER_HOST=unix:///var/run/docker.sock"]

# restart gitlab runner
systemctl restart gitlab-runner
```
---

# docker container
- สร้างโฟเดอร์ไว้เก็บ config
```bash
      mkdir -p /opt/gitlab-runner
```
- docker compose

```bash
services:
  gitlab-runner:
    image: gitlab/gitlab-runner:latest
    container_name: gitlab_runner
    restart: always
    networks:
      - app_net
    volumes:
      - /opt/gitlab-runner:/etc/gitlab-runner # เก็บ config.toml
      - /var/run/docker.sock:/var/run/docker.sock
      - /opt/caddy/caddy-root.crt:/usr/local/share/ca-certificates/caddy-root.crt:ro
    environment:
      - DOCKER_HOST=unix:///var/run/docker.sock

networks:
  app_net:
    external: true

```

- register -runner

```bash
   #เข้ามาใน container
   docker exec -it gitlab-runner -u root -p

   # register runner
   gitlab-runner register

   # ใส่ host เป็น http://gitlab จะดึง repo ผ่าน docker network
   http://gitlab
   # ใส่ token ไปเอามาจาก gitlab setting -> ci/cd -> runner -> create runner 

   # runner name
   docker

   # executer 
   docker

   # defuatl image
   node:22
     
```
- เเก้ config ของ runner ไห้ไช้ docker network
  
```bash

   #เข้ามาใน 
  nano /opt/gitlab-runner/config.toml


   #ใส่ไว้หลัง url= "http://gitlab"
   clone_url= "http://gitlab"

   #เพิ่ม
   network_mode = "app_net"

```
ci/cd variable
```bash
SSH_PRIVATE_KEY
```
---





