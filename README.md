# How to set up a Mastodon instance from scratch

I have recently installed a Mastodon instance [sayansi.social](https://sayansi.social) on a Virtual Private Server. These are the steps that I've taken.

## Preliminary steps ##

### 1. Buy the domain ###

I bought the [sayansi.social](https://sayansi.social) domain on [NameCheap](https://namecheap.com). For the moment, I only bought it for one year, as I still don't know how long the instance will last. Different providers may offer the same domain at different prices, so it is better to do some research before buying.

### 2. Subscribe to email provider ###

Mastodon needs an email provider to send notifications. I subscribed to [Brevo](https://www.brevo.com/). In the account setup, I asked to validate my [sayansi.social](https://sayansi.social) domain and Brevo took care of the necessary DNS settings. I also created an authorised sender `notifications@sayansi.social` for Mastodon notifications. Finally, I created an SMTP key that will be used in the Mastodon setup.

### 3. Buy a VPS ###

I bought a VPS from [Contabo](https://contabo.com). My machine is quite minimal: 4 CPU cores, 6 GB RAM, 400 GB SSD and a 250 GB object storage. The OS is Ubuntu 24. In the Control Panel, I created a bucket out of my object storage, and I made it public: take note of the tenant ID, as this will be needed for the Mastodon setup.

## VPS configuration ##

### 1. Connect to VPS and update system packages ###

I connected to my VPS via `ssh` and I updated system packages.

```
apt update && apt upgrade -y
```

### 2. Install fail2ban to block repeated login attampets

Following the [official documentation](https://docs.joinmastodon.org/admin/prerequisites/#install-fail2ban-so-it-blocks-repeated-login-attempts), I installed `fail2ban`.

```
apt install fail2ban
```

I created file `/etc/fail2ban/jail.local`.

```
touch /etc/fail2ban/jail.local
```

I added the following content to the configuration file:

```
[DEFAULT]
destemail = your@email.here
sendername = Fail2Ban

[sshd]
enabled = true
port = 22
mode = aggressive
```

Finally, I restarted fail2ban.

```
systemctl restart fail2ban
```

### 3. Install and configure firewall ###

Following the [official documentation](https://docs.joinmastodon.org/admin/prerequisites/#install-a-firewall-and-only-allow-ssh-http-and-https-ports), I installed `iptables-persistent`. During the installation, I was asked to keep the current rules - I declined.

```
apt install -y iptables-persistent
```

I edited `/etc/iptables/rules.v4` with the following content:

```
*filter

#  Allow all loopback (lo0) traffic and drop all traffic to 127/8 that doesn't use lo0
-A INPUT -i lo -j ACCEPT
-A INPUT ! -i lo -d 127.0.0.0/8 -j REJECT

#  Accept all established inbound connections
-A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

#  Allow all outbound traffic - you can modify this to only allow certain traffic
-A OUTPUT -j ACCEPT

#  Allow HTTP and HTTPS connections from anywhere (the normal ports for websites and SSL).
-A INPUT -p tcp --dport 80 -j ACCEPT
-A INPUT -p tcp --dport 443 -j ACCEPT
#  (optional) Allow HTTP/3 connections from anywhere.
-A INPUT -p udp --dport 443 -j ACCEPT

#  Allow SSH connections
#  The -dport number should be the same port number you set in sshd_config
-A INPUT -p tcp -m state --state NEW --dport 22 -j ACCEPT

#  Allow ping
-A INPUT -p icmp -m icmp --icmp-type 8 -j ACCEPT

# Allow destination unreachable messages, especially code 4 (fragmentation required) is required or PMTUD breaks
-A INPUT -p icmp -m icmp --icmp-type 3 -j ACCEPT

#  Log iptables denied calls
-A INPUT -m limit --limit 5/min -j LOG --log-prefix "iptables denied: " --log-level 7

#  Reject all other inbound - default deny unless explicitly allowed policy
-A INPUT -j REJECT
-A FORWARD -j REJECT

COMMIT
```

Then, I loaded the rules.

```
iptables-restore < /etc/iptables/rules.v4
```

Similary, I edited `/etc/iptables/rules.v6` with the following content:

```
*filter

#  Allow all loopback (lo0) traffic and drop all traffic to 127/8 that doesn't use lo0
-A INPUT -i lo -j ACCEPT
-A INPUT ! -i lo -d ::1/128 -j REJECT

#  Accept all established inbound connections
-A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

#  Allow all outbound traffic - you can modify this to only allow certain traffic
-A OUTPUT -j ACCEPT

#  Allow HTTP and HTTPS connections from anywhere (the normal ports for websites and SSL).
-A INPUT -p tcp --dport 80 -j ACCEPT
-A INPUT -p tcp --dport 443 -j ACCEPT
#  (optional) Allow HTTP/3 connections from anywhere.
-A INPUT -p udp --dport 443 -j ACCEPT

#  Allow SSH connections
#  The -dport number should be the same port number you set in sshd_config
-A INPUT -p tcp -m state --state NEW --dport 22 -j ACCEPT

#  Allow ping
-A INPUT -p icmpv6 -j ACCEPT

#  Log iptables denied calls
-A INPUT -m limit --limit 5/min -j LOG --log-prefix "iptables denied: " --log-level 7

#  Reject all other inbound - default deny unless explicitly allowed policy
-A INPUT -j REJECT
-A FORWARD -j REJECT

COMMIT
```

And I loaded the rules.

```
ip6tables-restore < /etc/iptables/rules.v6
```

### 4. Set access via SSH keys ###

I created a SSH key pair on my local machine, protecting it with a passphrase.

```
ssh-keygen -t rsa
```

By default the private and public keys are stored in `.ssh` folder inside my home. I copied the content of the public key `id_rsa.pub` and I pasted it into `/root/.ssh/authorized_keys` inside my VPS. I then tested the connection using the key. Finally, I edited `/etc/ssh/sshd_config` in my VPS and I set to "no" the variables *ChallengeResponseAuthentication*, *PasswordAuthentication*, *UsePAM*. Then, I restarted the `ssh` service.

```
systemctl restart ssh
```

## Docker installation and reverse proxy setup ##

### 1. Install Docker ###

I installed Docker following the [official documentation](https://docs.docker.com/engine/install/ubuntu/).

```
# Remove old packages
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt-get remove $pkg; done

# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update

# Install
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Test
sudo docker run hello-world
```

### 2. Set up nginx-proxy ###

Following these [instructions](https://giorgiogilestro.notion.site/How-to-setup-a-mastodon-docker-instance-d2c052ee8807456982dd258781689bc4), I created a Docker network.

```
docker network create nginx-proxy
```
Then I created the following `docker-compose.yml` file inside a `/opt/nginx-proxy` folder.

```
services:
  nginx-proxy:
    image: nginxproxy/nginx-proxy
    container_name: nginx-proxy
    labels:
      - com.github.nginx-proxy.nginx_proxy=true
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./volumes/certs:/etc/nginx/certs
      - ./volumes/conf.d:/etc/nginx/conf.d
      - ./volumes/vhost:/etc/nginx/vhost.d
      - ./volumes/html:/usr/share/nginx/html
      - /var/run/docker.sock:/tmp/docker.sock:ro
    dns:
      - 8.8.8.8
      - 8.8.4.4
    restart: always

  nginx-proxy-acme:
    image: nginxproxy/acme-companion
    container_name: nginx-proxy-acme
    depends_on:
      - nginx-proxy
    volumes_from:
      - nginx-proxy
    volumes:
      - ./volumes/certs:/etc/nginx/certs:rw
      - ./volumes/acme:/etc/acme.sh
      - /var/run/docker.sock:/var/run/docker.sock:ro
    environment:
      - DEFAULT_EMAIL=YOUR_EMAIL_HERE_FOR_LETSENCRYPT_CERTS
      - NGINX_PROXY_CONTAINER=nginx-proxy
    restart: always

networks:
  default:
    name: nginx-proxy
    external: true
```

I then started `nginx` and its `acme` companion.

```
docker compose up -d
```

## Mastodon installation ##

### 1. Prepare for the installation ###

Following these [instructions](https://giorgiogilestro.notion.site/How-to-setup-a-mastodon-docker-instance-d2c052ee8807456982dd258781689bc4), I created the following `docker-compose.yml` file inside a `/opt/mastodon` folder.

```
# Adapted from https://github.com/mastodon/mastodon/blob/main/docker-compose.yml
---
services:
  db:
    restart: always
    image: postgres:14-alpine
    shm_size: 256mb
    networks:
      - internal_network
    healthcheck:
      test: ['CMD', 'pg_isready', '-U', 'postgres']
    volumes:
      - ./postgres14:/var/lib/postgresql/data
    environment:
      - 'POSTGRES_HOST_AUTH_METHOD=trust'

  redis:
    restart: always
    image: redis:7-alpine
    networks:
      - internal_network
    healthcheck:
      test: ['CMD', 'redis-cli', 'ping']
    volumes:
      - ./redis:/data

  es:
    restart: always
    image: elasticsearch:7.17.24
    environment:
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m -Des.enforce.bootstrap.checks=true"
      - "xpack.license.self_generated.type=basic"
      - "xpack.security.enabled=false"
      - "xpack.watcher.enabled=false"
      - "xpack.graph.enabled=false"
      - "xpack.ml.enabled=false"
      - "bootstrap.memory_lock=true"
      - "cluster.name=es-mastodon"
      - "discovery.type=single-node"
      - "thread_pool.write.queue_size=1000"
    networks:
       - external_network
       - internal_network
    healthcheck:
       test: ["CMD-SHELL", "curl --silent --fail localhost:9200/_cluster/health || exit 1"]
    volumes:
       - ./elasticsearch:/usr/share/elasticsearch/data
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536
        hard: 65536
    ports:
      - '127.0.0.1:9200:9200'

  web:
    #build: .
    image: tootsuite/mastodon:v4.3.3
    #image: ghcr.io/mastodon/mastodon:v4.3.0 this would be the reccomended image on the github docker-compose file
    restart: always
    env_file: .env.production
    #command: bash -c "rm -f /mastodon/tmp/pids/server.pid; bundle exec rails s -p 3000"
    command: bundle exec puma -C config/puma.rb
    networks:
      - external_network
      - internal_network
    healthcheck:
      # prettier-ignore
      test: ['CMD-SHELL', 'wget -q --spider --proxy=off localhost:3000/health || exit 1']
    ports:
      - '127.0.0.1:3000:3000'
    depends_on:
      - db
      - redis
      - es
    volumes:
      - ./public/system:/mastodon/public/system
    expose:
      - 3000
    environment:
      VIRTUAL_HOST: "sayansi.social"
      LETSENCRYPT_HOST: "sayansi.social"
      VIRTUAL_PATH: "/"
      VIRTUAL_PORT: 3000
      RAILS_LOG_LEVEL: warn

  streaming:
    #build: .
    image: tootsuite/mastodon-streaming:v4.3.3
    restart: always
    env_file: .env.production
#    command: node ./streaming
    command: node ./streaming/index.js
    networks:
      - external_network
      - internal_network
    healthcheck:
      # prettier-ignore
      test: ['CMD-SHELL', "curl -s --noproxy localhost localhost:4000/api/v1/streaming/health | grep -q 'OK' || exit 1"]
    ports:
      - '127.0.0.1:4000:4000'
    depends_on:
      - db
      - redis
    expose:
      - 4000
    environment:
      VIRTUAL_HOST: "sayansi.social"
      VIRTUAL_PATH: "/api/v1/streaming"
      VIRTUAL_PORT: 4000

  sidekiq:
    #build: .
    image: tootsuite/mastodon:v4.3.3
    restart: always
    env_file: .env.production
    command: bundle exec sidekiq
    environment:
      - RAILS_LOG_LEVEL=error
    depends_on:
      - db
      - redis
    networks:
      - external_network
      - internal_network
    volumes:
      - ./public/system:/mastodon/public/system
    healthcheck:
      test: ['CMD-SHELL', "ps aux | grep '[s]idekiq\ 6' || false"]

  ## Uncomment to enable federation with tor instances along with adding the following ENV variables
  ## http_hidden_proxy=http://privoxy:8118
  ## ALLOW_ACCESS_TO_HIDDEN_SERVICE=true
  # tor:
  #   image: sirboops/tor
  #   networks:
  #      - external_network
  #      - internal_network
  #
  # privoxy:
  #   image: sirboops/privoxy
  #   volumes:
  #     - ./priv-config:/opt/config
  #   networks:
  #     - external_network
  #     - internal_network

networks:
  external_network:
    name: nginx-proxy
    external: true
  internal_network:
    internal: true
```

I then edited `.env` file with the following content.

```
#.env
LETS_ENCRYPT_EMAIL=YOUR_EMAIL_HERE_FOR_LETSENCRYPT_CERTS
MASTODON_DOMAIN=sayansi.social
MASTODON_VERSION=v4.3.3
```

I created `.env.production` file.

```
touch .env.production
```

### 2. Install Mastodon ###

I launched the Mastodon setup.

```
sudo docker compose run --rm -u root -v $(pwd)/.env.production:/opt/mastodon/.env.production -e RUBYOPT=-W0 web bundle exec rake mastodon:setup
```

The wizard asks some questions concerning `postgres` and `redis`: just give the default answers. Then, it asks information about the remote storage (in my case the Contabo bucket) and the email provider (in my case Brevo).

Once the setup is completed, the file `.env.production` will be populated with various settings. For reference, these are my settings for the Contabo bucket:

```
S3_ENABLED=true
S3_BUCKET=mastodon
AWS_ACCESS_KEY_ID=THE_ACCESS_KEY_ID_OF_YOUR_OBJECT_STORAGE
AWS_SECRET_ACCESS_KEY=THE_SECRET_ACCESS_KEY_OF_YOUR_OBJECT_STORAGE 
S3_REGION=default
S3_PROTOCOL=https
S3_HOSTNAME=eu2.contabostorage.com
S3_ENDPOINT=https://eu2.contabostorage.com
S3_ALIAS_HOST=THE_TENANT_ID_OF_YOUR_BUCKET
```

These are the settings for my email provider:

```
SMTP_SERVER=smtp-relay.brevo.com
SMTP_PORT=587
SMTP_LOGIN=YOUR_BREVO_LOGIN
SMTP_PASSWORD=YOUR_SMTP_KEY
SMTP_AUTH_METHOD=plain
SMTP_OPENSSL_VERIFY_MODE=none
SMTP_ENABLE_STARTTLS=auto
SMTP_FROM_ADDRESS='Mastodon <notifications@sayansi.social>'
```

Run Mastodon.

```
docker compose up -d
```

Mastodon is now live on [sayansi.social](https://sayansi.social). Hooray! :tada:
