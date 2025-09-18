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

- Adjust `/etc/gitlab/gitlab.rb`:

```
nginx['custom_gitlab_server_config'] = "location /.well-known/acme-challenge/ {\n root /var/opt/gitlab/nginx/www; \n}\n"
```

- Reconfigure Gitlab:

```bash
sudo gitlab-ctl reconfigure
```

- Issue a certificate:

```bash
certbot certonly \
    --webroot --webroot-path /var/opt/gitlab/nginx/www/ \
    --no-eff-email \
    --key-type ecdsa \
    --reuse-key \
    --deploy-hook "sudo gitlab-ctl restart nginx" \
    -d git.yourdomain.tld


certbot certonly \
    --webroot --webroot-path /var/opt/gitlab/nginx/www/ \
    --no-eff-email \
    --key-type ecdsa \
    --reuse-key \
    --deploy-hook "sudo gitlab-ctl restart nginx" \
    -d registry.yourdomain.tld
```

- Further adjust `/etc/gitlab/gitlab.rb`:

```
gitlab_rails['gitlab_email_from'] = 'gitlab.system@metropolis.nexus'

gitlab_rails['smtp_enable'] = true
gitlab_rails['smtp_address'] = "mail.metropolis.nexus"
gitlab_rails['smtp_port'] = 465
gitlab_rails['smtp_user_name'] = "gitlab.system@metropolis.nexus"
gitlab_rails['smtp_password'] = "REDACTED"                                      
gitlab_rails['smtp_domain'] = "mail.metropolis.nexus"
gitlab_rails['smtp_authentication'] = "plain"
gitlab_rails['smtp_enable_starttls_auto'] = false
gitlab_rails['smtp_tls'] = true

gitlab_rails['gitlab_email_from'] = 'gitlab.system@metropolis.nexus'
gitlab_rails['gitlab_email_display_name'] = 'Metropolis GitLab'

gitlab_rails['gitlab_default_color_mode'] = 3

gitlab_rails['gitlab_default_projects_features_issues'] = true
gitlab_rails['gitlab_default_projects_features_merge_requests'] = true
gitlab_rails['gitlab_default_projects_features_wiki'] = false
gitlab_rails['gitlab_default_projects_features_snippets'] = false
gitlab_rails['gitlab_default_projects_features_builds'] = false
gitlab_rails['gitlab_default_projects_features_container_registry'] = false

gitlab_rails['content_security_policy'] = {
 'enabled' => true,
 'directives' => {
   'child_src' => 'self',
   'default_src' => 'none',
   'frame_src' => 'self'
 }
}

gitlab_rails['allowed_hosts'] = ['git.metropolis.nexus']

gitlab_rails['omniauth_allow_single_sign_on'] = ['openid_connect']
gitlab_rails['omniauth_sync_email_from_provider'] = 'openid_connect'
gitlab_rails['omniauth_sync_profile_from_provider'] = ['openid_connect']
gitlab_rails['omniauth_sync_profile_attributes'] = ['email']
gitlab_rails['omniauth_auto_sign_in_with_provider'] = 'openid_connect'
gitlab_rails['omniauth_block_auto_created_users'] = false
gitlab_rails['omniauth_auto_link_user'] = ['openid_connect']
gitlab_rails['omniauth_allow_bypass_two_factor'] = ['']
gitlab_rails['omniauth_providers'] = [
  {
    name: 'openid_connect',
    label: 'Authentik',
    args: {
      name: 'openid_connect',
      scope: ['openid','profile','email'],
      response_type: 'code',
      issuer: 'https://auth.metropolis.nexus/application/o/gitlab/',
      discovery: true,
      client_auth_method: 'query',
      uid_field: 'preferred_username',
      send_scope_to_token_endpoint: 'true',
      pkce: true,
      client_options: {
        identifier: 'REDACTED',                                
        secret: 'REDACTED',                                                                                                                        
        redirect_uri: 'https://git.metropolis.nexus/users/auth/openid_connect/callback'
      }
    }  
  }    
]

registry_external_url 'https://registry.metropolis.nexus'

letsencrypt['enable'] = false
nginx['redirect_http_to_https'] = true
nginx['ssl_ciphers'] = "ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256"
nginx['ssl_prefer_server_ciphers'] = "on"
nginx['ssl_protocols'] = "TLSv1.2 TLSv1.3"
nginx['ssl_session_cache'] = "builtin:1000 shared:SSL:10m"
nginx['ssl_session_tickets'] = "off"
nginx['ssl_session_timeout'] = "5m"
nginx['hsts_max_age'] = 31536000
nginx['hsts_include_subdomains'] = true
nginx['gzip_enabled'] = false

gitlab_rails['packages_enabled'] = true

spamcheck['enable'] = true
```

- Setup symlink for Certbot's certificates:

```bash
sudo mkdir -p /etc/gitlab/ssl/backup
sudo chmod 755 /etc/gitlab/ssl/backup
sudo mv /etc/gitlab/ssl/git* /etc/gitlab/ssl/backup

sudo ln -s /etc/letsencrypt/live/git.yourdomain.tld/fullchain.pem /etc/gitlab/ssl/git.yourdomain.tld.crt
sudo ln -s /etc/letsencrypt/live/git.yourdomain.tld/privkey.pem /etc/gitlab/ssl/git.yourdomain.tld.key

sudo ln -s /etc/letsencrypt/live/registry.yourdomain.tld/fullchain.pem /etc/gitlab/ssl/registry.yourdomain.tld.crt
sudo ln -s /etc/letsencrypt/live/registry.yourdomain.tld/privkey.pem /etc/gitlab/ssl/registry.yourdomain.tld.key

sudo gitlab-ctl reconfigure
```

## Configure GitLab

- Login
- Change the admin password
- Disable sign-up 

### Settings
- Settings -> General -> Visibility and access controls
    - RSA SSH Keys -> Are Forbidden
    - DSA SSH Keys -> Are Forbidden
    - ECDSA SSH Keys -> Are Forbiden
    - ED25519 SSH Keys -> Must be at least 256 bits
    - ECDSA_SK SSH keys -> Are Forbidden
    - ED25519_SK SSH Keys -> Must be at least 256 bits

- Settings -> General -> Account and limit
    - Uncheck "Gravatar enabled"
    - Uncheck "Require expiration date"
    - Uncheck "Prompt users to upload SSH keys"

- Settings -> General -> Sign-up Restrictions
    - Email confirmation settings -> Hard

- Settings -> General -> Sign-in restrictions
    - Uncheck "Allow password authentication for Git over HTTP(S)"
    - Check "Enforce two-factor authentication"
    - Check "Require administrators to enable 2FA"

- Settings -> General -> Customer experience improvement and third-party offers
    - Check "Do not display content for customer experience improvement and offers from third parties"

- Reporting -> Abuse reports -> Add email

## User Config
- Account -> Two Factor Authentication -> Enable 2FA
- Emails -> Add personal email
- Profile
    - Change primary email to personal email
    - Public email -> Select personal email
    - Add additional information
- GPG Keys -> Add GPG key
- Preferences
    - Appearance -> Auto
    - Navigation theme -> Blue
    - Syntax highlighting theme -> Dark
