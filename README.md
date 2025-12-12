- docker.io
- portainer
- caddy
- registry
- gitlab
- gitlab-runner
### docker

```bash
apt install docker.io

```
---

### portainer

```bash
docker volume create portainer_data

docker run -d -p 9443:9443 --name portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce:latest

```
---

### caddy

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


dns.vecskill.ovec {
	tls internal # บอกให้ใช้ Certificate จาก Internal CA
	encode zstd gzip
	reverse_proxy http://technitium-dns:5380
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

# ของ ubuntu server 
sudo cp /opt/caddy/caddy-root.crt /usr/local/share/ca-certificates/caddy-root.crt

sudo update-ca-certificates

```
---

### registry

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
---


### gitlab
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

password

```bash
หรือ ไปเอาใน container
cat /etc/gitlab/initial_root_password


```
---

### gitlab-runner 
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
services:
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
   gitlab runner register

   # ใส่ host เป็น http://gitlab จะดึง repo ผ่าน docker network
   http://gitlab
   # ใส่ token ไปเอามาจาก gitlab setting -> ci/cd -> runner -> create runner 

   # executer 
   docker

   # defuatl image
   node:22
     
```
- เเก้ config ของ runner ไห้ไช้ docker network
  ```bash
   #เข้ามาใน 
  nano /opt/gitlab-runner/config.toml

   #เพิ่ม
   network_mode = "app_net"
  ```
  - ตัวอย่าง config
    
  ```bash
   concurrent = 1
   check_interval = 0
   shutdown_timeout = 0

[session_server]
  session_timeout = 1800

[[runners]]
  name = "fd6608ff37c6"
  url = "http://gitlab"

  id = 39
  token = "glrt-U0ivaesXzrajgl3imRU72W86MQpwOjIKdDozCnU6MQ8.01.170y1358m"
  token_obtained_at = 2025-12-12T07:41:56Z
  token_expires_at = 0001-01-01T00:00:00Z
  executor = "docker"
  [runners.cache]
    MaxUploadedArchiveSize = 0
    [runners.cache.s3]
    [runners.cache.gcs]
    [runners.cache.azure]
  [runners.docker]
    host = "unix:///var/run/docker.sock"
    tls_verify = false
    image = "node:22"
    privileged = false
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false
    shm_size = 0
    network_mtu = 0
    volumes = ["/cache","/var/run/docker.sock:/var/run/docker.sock"]
    network_mode = "app_net"
	
  ```
---





