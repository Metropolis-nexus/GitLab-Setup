# GitLab-Setup

## Install GitLab

Install GitLab Omnibus as per upstream documentation. Don't setup account pinning in the CAA record for now. Since RHEL 10 support isn't out yet, we can use the RHEL 9 repo [here](https://packages.gitlab.com/install/repositories/gitlab/gitlab-ee/config_file.repo?os=rhel&dist=10&source=script):

## Setup certbot

```bash
subscription-manager repos --enable codeready-builder-for-rhel-10-$(arch)-rpms
sudo dnf install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-10.noarch.rpm
sudo dnf install -y certbot
sudo certbot register
```

- Pin the account in the CAA record

```bash
certbot certonly \
    --webroot --webroot-path /var/opt/gitlab/nginx/www/ \
    --no-eff-email \
    --key-type ecdsa \
    --reuse-key \
    --deploy-hook "sudo gitlab-ctl restart nginx" \
    -d git.yourdomain.tld
```

- Adjust `/etc/gitlab/gitlab.rb`:

```
letsencrypt['enable'] = false
nginx['redirect_http_to_https'] = true
nginx['ssl_ciphers'] = "ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256"
nginx['ssl_prefer_server_ciphers'] = "on"
nginx['ssl_protocols'] = "TLSv1.2 TLSv1.3"
nginx['ssl_session_cache'] = "shared:SSL:10m"
nginx['ssl_session_tickets'] = "off"
nginx['hsts_max_age'] = 31536000
nginx['hsts_include_subdomains'] = true
nginx['gzip_enabled'] = false
```

- Setup symlink for Certbot's certificates:

```bash
sudo mkdir -p /etc/gitlab/ssl/backup
sudo chmod 755 /etc/gitlab/ssl/backup
sudo mv /etc/gitlab/ssl/git* /etc/gitlab/ssl/backup
sudo ln -s /etc/letsencrypt/live/git.yourdomain.tld/fullchain.pem /etc/gitlab/ssl/git.yourdomain.tld.crt
sudo ln -s /etc/letsencrypt/live/git.yourdomain.tld/privkey.pem /etc/gitlab/ssl/git.yourdomain.tld.key
sudo gitlab-ctl reconfigure
```

## Configure GitLab

- Login
- Change the admin password
- Disable sign-up 
