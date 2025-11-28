# 🚀 DevOps Server Setup - GitLab Runner on Ubuntu

[![GitLab Runner](https://img.shields.io/badge/GitLab-Runner-orange)](https://docs.gitlab.com/runner/)

This guide explains how to install and configure **GitLab Runner** on Ubuntu with **Docker executor**, suitable for CI/CD pipelines.

---

## 📌 Table of Contents
- [Features](#features)
- [Prerequisites](#prerequisites)
- [1️⃣ Install GitLab Runner](#1️⃣-install-gitlab-runner)
- [2️⃣ Configure Runner](#2️⃣-configure-runner)
- [3️⃣ Restart GitLab Runner](#3️⃣-restart-gitlab-runner)
- [4️⃣ Register Runner with GitLab](#4️⃣-register-runner-with-gitlab)
- [5️⃣ Test Runner](#5️⃣-test-runner)
- [Optional: Docker Privileged Mode](#optional-docker-privileged-mode)
- [Notes](#notes)

---

## ✨ Features
- Install GitLab Runner on Ubuntu
- Configure Docker executor
- Map Docker socket for container access
- Support environment variables
- Auto-restart on server boot

---

## 🛠 Prerequisites
- Ubuntu 20.04+ or Debian-based OS
- Root or sudo privileges
- Docker installed on host
- GitLab account & project

---

## 1️⃣ Install GitLab Runner

```bash
sudo apt-get update
sudo apt-get install -y curl ca-certificates

# Add GitLab Runner repository
curl -fsSL https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh | sudo bash

# Install GitLab Runner
sudo apt-get install -y gitlab-runner

# Verify installation
gitlab-runner --version

## 1️⃣ Install GitLab Runner

```bash
 volumes = ["/cache","/var/run/docker.sock:/var/run/docker.sock"]
environment = ["DOCKER_HOST=unix:///var/run/docker.sock"]
