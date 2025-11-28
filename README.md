# install-devops-server-ubuntu
## install gitlab-runner
  ```
   apt-get install -y curl ca-certificates
   curl -fsSL https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh | sudo bash
   apt-get install -y gitlab-runner
   gitlab-runner --version
  
- เข้าไปที่
  ```
  nano /etc/gitlab-runner/config.toml
   
  ```
  volumes = ["/cache","/var/run/docker.sock:/var/run/docker.sock"] 
  environment = ["DOCKER_HOST=unix:///var/run/docker.sock"]

1. sfgds
  ```
  sudo systemctl restart gitlab-runner
