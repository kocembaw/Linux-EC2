# EC2 Instance

[![AWS](https://img.shields.io/badge/AWS-232F3E?style=for-the-badge&logo=amazonaws&logoColor=white)](https://aws.amazon.com/ec2/)
[![Nginx](https://img.shields.io/badge/Nginx-009639?style=for-the-badge&logo=nginx&logoColor=white)](https://nginx.org/)
[![Ubuntu](https://img.shields.io/badge/Ubuntu-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)](https://ubuntu.com/)

A hands-on cloud computing project demonstrating how to launch an AWS EC2 instance, connect to it via SSH, and deploy a static website served by Nginx — with an optional CI/CD pipeline that auto-deploys changes on every push.

## Table of Contents
- [What It Is](#what-it-is)
- [Caveats and Limitations](#caveats-and-limitations)
- [Preview in Action](#preview-in-action)
- [Requirements](#requirements)
- [How to Build and Run](#how-to-build-and-run)
- [Project Structure](#project-structure)
- [Advanced Usage: CI/CD Pipeline](#advanced-usage-cicd-pipeline)
  - [Custom Domain and HTTPS](#custom-domain-and-https)
- [Acknowledgements](#acknowledgements)
- [License](#license)

## What It Is

This repository documents a complete workflow for provisioning a [Linux server on AWS EC2](https://docs.aws.amazon.com/ec2/) and deploying a simple static website to it. This project was implemented according to the [roadmap.sh EC2 Instance project guide](https://roadmap.sh/projects/ec2-instance).

The instance runs [Ubuntu Server](https://ubuntu.com/server) and serves the site through [Nginx](https://nginx.org/), reachable over the instance's public IPv4 address (and optionally a custom domain with HTTPS).

## Caveats and Limitations

- **CodeDeploy not used:** Accounts created after July 15, 2025 on AWS's **Free Plan** don't have access to AWS CodeDeploy. The CI/CD pipeline in this project uses **CodeBuild + AWS Systems Manager (Run Command)** instead, which stays fully within Free Plan limits.
- **IP changes on stop/start:** Without an Elastic IP, the instance's public IP changes if it's stopped and restarted.

## Preview in Action

Once deployed, requesting the site returns the static page served by Nginx:

```bash
$ curl http://<PUBLIC_IP>
<!DOCTYPE html>
<html lang="pl">
  <head><title>Moja strona na AWS EC2</title></head>
  <body><h1>Działa</h1></body>
</html>
```

## Requirements

- An [AWS account](https://aws.amazon.com/) 
- An SSH client `ssh`
- *(Optional, for stretch goals)* a registered domain name.

## How to Build and Run

1. **Launch an EC2 instance** — Ubuntu Server AMI, `t2.micro`, default VPC/subnet, security group open on ports `22` and `80`, with a key pair for SSH access.
2. **Connect via SSH**:
   ```bash
   ssh -i your-key.pem ubuntu@<PUBLIC_IP>
   ```
3. **Update packages and install Nginx**:
   ```bash
   sudo apt update && sudo apt upgrade -y
   sudo apt install nginx -y
   sudo systemctl enable --now nginx
   ```
4. **Deploy the static site**:
   ```bash
   sudo nano /var/www/html/index.html
   ```
5. **Visit the site** at `http://<PUBLIC_IP>` in your browser.

## Project Structure

```text
.
├── index.html      # The static website served by Nginx
├── buildspec.yml    # CodeBuild instructions for the CI/CD pipeline
└── README.md         # Project documentation
```

## Advanced Usage: CI/CD Pipeline

Changes pushed to this repository can be deployed automatically using **GitHub → CodePipeline → CodeBuild → AWS Systems Manager → EC2**, without relying on AWS CodeDeploy:

```yaml
# buildspec.yml (excerpt)
version: 0.2
phases:
  build:
    commands:
      - aws ssm send-command --targets "Key=tag:Name,Values=moj-projekt-website" \
          --document-name "AWS-RunShellScript" \
          --parameters commands='["cd /home/ubuntu/moj-projekt-website","sudo git pull","sudo rsync -a --exclude buildspec.yml --exclude .git ./ /var/www/html/","sudo systemctl restart nginx"]'
```

Every `git push` to the `main` branch triggers CodePipeline, which runs this CodeBuild step to pull the latest code onto the instance and reload Nginx — no manual `scp`/SSH required.

### Custom Domain and HTTPS

The site can optionally be served under a custom domain (via Route 53 or another DNS provider) with a free TLS certificate from [Let's Encrypt](https://letsencrypt.org/):

```bash
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
```

## Acknowledgements

- Project idea and requirements provided by [roadmap.sh DevOps Projects](https://roadmap.sh/projects/ec2-instance).
- Background reading on EC2 instance types: [Up and Running with AWS EC2](https://kamranahmed.info/posts/up-and-running-with-aws-ec2).

## License

Distributed under the [MIT License](LICENSE).
