# install-devops-server-ubuntu
## install gitlab-runner
```
apt-get install -y curl ca-certificates
curl -fsSL https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh | sudo bash
apt-get install -y gitlab-runner
gitlab-runner --version
