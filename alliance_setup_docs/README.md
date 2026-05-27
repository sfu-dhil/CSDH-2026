# Example Alliance Setup

These instructions will setup you up with:
- 1 Fedora instance
- 2 Volumes (50GB for OS, 350GB for persistent storage)
- Network open to HTTP, HTTPS, and SSH
- 1 Floating IP

Along with Docker running in swarm mode with these containers:
- [Traefik](https://hub.docker.com/_/traefik) for ingress into other containers
- [Jenkins](https://github.com/sfu-dhil/jenkins/pkgs/container/jenkins)
- [Grafana](https://hub.docker.com/r/grafana/grafana), [Prometheus](https://hub.docker.com/r/prom/prometheus), [Node Exporter](https://hub.docker.com/r/prom/node-exporter), [cAdvisor](https://github.com/google/cadvisor/pkgs/container/cadvisor) for system monitoring
- [Anubis](https://github.com/techarohq/anubis/pkgs/container/anubis) for simple anti-bot/AI scrapping
- [Postfix](https://hub.docker.com/r/boky/postfix) for a simple send mail SMTP server
- [Umami](https://github.com/umami-software/umami/pkgs/container/umami) for log level web traffic tracking (along with [Postgres](https://hub.docker.com/_/postgres))
> Note the jenkins is a slightly modified from default with plugins and DinD (Docker in Docker) support for running jobs.

Assuming you are starting with:
- No Floating IPs, Volumes, or Instances
- You have a wildcard or multiple domains to point at the alliance server applications

## Arbutus Cloud Setup

### Create Floating IP

1. Go to `Network` -> `Floating IPs`
1. Click `Allocate IP To Project`
1. Fill in the form with
	1. Pool: `Public-Network`
1. Click `Allocate IP`


### Open HTTP (80), HTTPS (443), and SSH (22) ports

1. Go to `Networks` -> `Security Groups` -> `Manage Rules` (for default security group)
1. Ensure that the following exist:
	1. `Egress` + `IPv4` + `Any` + `Any` + `0.0.0.0/0`
	1. `Egress` + `IPv6` + `Any` + `Any` + `::/0`
	1. `Ingress` + `IPv4` + `TCP` + `22 (SSH)` + `0.0.0.0/0`
	1. `Ingress` + `IPv4` + `TCP` + `80 (HTTP)` + `0.0.0.0/0`
	1. `Ingress` + `IPv4` + `TCP` + `443 (HTTPS)` + `0.0.0.0/0`
>Note: this allows anyone to access http/https. It also easily allows all developers to SSH regardless of network (home, work, etc) BUT can be a security concern if any SSH keys are lost/misused/cracked.

### Create default fedora KeyPair

1. Go to `Compute` -> `Key Pairs`
1. Click `Create Key Pair`
	1. Key Pair Name: `alliance`
	1. Key Type: `SSH Key`
1. Click `Create Key Pair`

Store the automatically generated `alliance.pem` in a secure place
Also use it locally by storing it in your home .ssh directory (`~/.ssh/alliance.pem`) for later ssh into the `fedora` user
>Note: make sure to `chmod 400 ~/.ssh/alliance.pem` if you have permissions issues with it


### Create Instance

1. Go to `Compute` -> `Instances`
1. Click `Launch Instance`
	1. Details
		1. Instance Name: `alliance-instance`
		1. *leave other fields as default*
	1. Source
		1. Select Boot Source: `Image`
		1. Create New Volume: `Yes`
		1. Volume Size (GB): `50`
		1. Delete Volume on Instance Delete: `No`
		1. Select `Fedora-42-x64-2025-08-1` (or latest fedora) into Allocated
	1. Flavor
		1. Select `p8-12gb` into Allocated (or whatever resources you have access too)
	1. Networks
		1. Select the workspace's network into Allocated
	1. Key Pair
		1. Select `alliance` into Allocated
	1. Leave other sections as default
1. Click `Launch Instance` to complete the form


### Create Persistence Storage Volume

1. Go to `Volumes` -> `Volumes`
1. Click `Create Volume`
	1. Volume Name: `alliance_persistence`
	1. Size (GiB): `350`
	1. *leave other fields as default*
1. Click `Create Volume` to complete the form

Attach the Persistence Volume to Instance

>Note: may need to wait until the instance if fully setup (may take a few minutes)

1. Go to `Volumes` -> `Volumes`
1. Click the `alliance_persistence` row's dropdown `▼` under the *Actions* column -> `Manage Attachments`
	1. Attach to Instance: `alliance-instance`
1. Click `Attach Volume`

### Associate IP With Project

1. Go to `Networks` -> `Floating IPs`
1. Find the Floating IP in the list and click `Associate`
1. Fill in the form with
	1. Port to be associated: Select the `alliance-instance` instance option
1. Click `Associate`
1. Wait for Floating IP to be associated with the instance


## DNS settings

Go to `Network` -> `Floating IPs` and copy the IP Address for the Generated Floating IP

DNS settings should be `A` records pointing to the floating IP Address.

> Example: `alliance-jenkins.dhil.lib.sfu.ca` and `grassrootschinesehistory.ca` point to `<YOUR FLOATING POINT IP ADDRESS>` for the DHIL.


## Server Setup

Once the instance is fully up and running

### Access the server

```shell
ssh -i ~/.ssh/alliance.pem fedora@<YOUR FLOATING POINT IP ADDRESS>
```

### Update package manager

```shell
sudo dnf -y update
sudo systemctl daemon-reload

sudo dnf -y install dnf-plugins-core
sudo dnf-3 config-manager --add-repo https://download.docker.com/linux/fedora/docker-ce.repo
```


### Install docker swarm deps

```shell
sudo dnf -y install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin nano acl htop
```

### Start docker and enable autostart

```shell
sudo systemctl enable --now docker
```

### Add fedora user to docker group


```shell
sudo usermod -aG docker fedora
```
> NOTE: Docker group permissions should be enough to manage dockerized projects without using sudo in general

### Formatting the Persistence Volume

*SKIP IF THE VOLUME WAS PREVIOUSLY IN USE*

```shell
sudo fdisk /dev/vdb
	# enter the follow sequence for prompts
	n
	p
	1
	<default> #just hit enter
	<default> #just hit enter
	w
sudo mkfs -t ext4 /dev/vdb1
```

### Mounting the Persistence Volume

Get the UUID from running `sudo blkid` and looking for `/dev/vdb1`

> Example: `/dev/vdb1: UUID="f440547d-c97c-4d03-9531-872bfcdd6660" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="606c1ffa-01"`

```shell
sudo nano /etc/fstab
    # add the following line to the end Using the proper UUID
	UUID=<UUID> /mnt                    auto    defaults,nofail 0 3
```

### Reboot server

Reboot from arbutus cloud to ensure that user, group, and mounting changes take effect. `Compute` -> `Instances` -> `alliance-instance`'s actions dropdown -> `Hard Reboot Instance`


### SSH back in to instance

```shell
ssh -i ~/.ssh/alliance.pem fedora@<YOUR FLOATING POINT IP ADDRESS>
```

### Setup docker swarm

```shell
docker swarm init
```

### Setup public network (used by traefik & other private services)

```shell
docker network create --scope=swarm --attachable --driver=overlay traefik-public
docker network create --scope=swarm --attachable --driver=overlay postgres-private
docker network create --scope=swarm --attachable --driver=overlay monitoring-private
docker network create --scope=swarm --attachable --driver=overlay smtp-private
```

### Setup the folder ACL permissions

```shell
sudo mkdir -p /mnt/docker-swarm-deploy /mnt/docker-swarm-persistence
sudo chown root:docker /mnt /mnt/docker-swarm-deploy /mnt/docker-swarm-persistence

sudo setfacl -R -m group::rwx /mnt /mnt/docker-swarm-deploy /mnt/docker-swarm-persistence
sudo setfacl -R -m other::r-- /mnt /mnt/docker-swarm-deploy /mnt/docker-swarm-persistence
sudo setfacl -m default:user::rwx /mnt /mnt/docker-swarm-deploy /mnt/docker-swarm-persistence
sudo setfacl -m default:group::rwx /mnt /mnt/docker-swarm-deploy /mnt/docker-swarm-persistence
sudo setfacl -m default:group:docker:rwx /mnt /mnt/docker-swarm-deploy /mnt/docker-swarm-persistence
sudo setfacl -m default:mask::rwx /mnt /mnt/docker-swarm-deploy /mnt/docker-swarm-persistence
sudo setfacl -m default:other::r-- /mnt /mnt/docker-swarm-deploy /mnt/docker-swarm-persistence
sudo chown deploy:docker /mnt /mnt/docker-swarm-deploy /mnt/docker-swarm-persistence
```

```shell
mkdir -p /mnt/docker-swarm-persistence/postgres-data/18/docker
mkdir -p /mnt/docker-swarm-persistence/traefik
mkdir -p /mnt/docker-swarm-persistence/grafana-data
mkdir -p /mnt/docker-swarm-persistence/prometheus-data
mkdir -p /mnt/docker-swarm-persistence/jenkins-home

touch /mnt/docker-swarm-persistence/traefik/acme.json
chmod 600 /mnt/docker-swarm-persistence/traefik/acme.json

# set proper folder permissions for various services
sudo chown 999:999 -R /mnt/docker-swarm-persistence/postgres-data
sudo chown 472:472 /mnt/docker-swarm-persistence/grafana-data
sudo chown 65534:65534 /mnt/docker-swarm-persistence/prometheus-data
sudo chown 1000:1000 /mnt/docker-swarm-persistence/jenkins-home
```

### Create your secrets

On your local machine in the `alliance_setup_docs` folder

```shell
echo $(openssl rand -hex 32) | tr -d '\n' > secrets/grafana_admin_password
echo $(openssl rand -hex 32) | tr -d '\n' > secrets/postgres_root_password
echo $(openssl rand -hex 32) | tr -d '\n' > secrets/umami_admin_password
echo $(openssl rand -hex 32) | tr -d '\n' > secrets/umami_db_password
```

### Fix all docker compose configs with your network values

On your local machine in the `alliance_setup_docs` folder

In `anubis.yaml`:
- Update `services.anti_bot.environment.REDIRECT_DOMAINS` with your wildcard or jenkins domain
- Update `services.anti_bot.environment.PUBLIC_URL` and `services.anti_bot.deploy.labels -> traefik.http.routers.anubis.rule` with your publicly accessible anubis domain

In `jenkins.yaml`:
- Update `services.app.user` with your instance's Docker group id (`getent group docker`)
- Update `services.app.deploy.labels -> traefik.http.routers.jenkins.rule` with the jenkins domain

In `monitoring.yaml`:
- Update `services.grafana.environment.GF_SECURITY_ADMIN_EMAIL` with your desired admin email address
- Update `services.grafana.deploy.labels -> traefik.http.routers.grafana.rule` with your publicly accessible grafana domain

In `smtp.yaml`:
- Update `services.mail.environment.ALLOWED_SENDER_DOMAINS` with you the root domains that will send mail (ex: jenkins domain)

In `traefik.yaml`:
- Update `services.proxy.command -> --certificatesresolvers.letsencrypt.acme.email` with your desired admin email address

In `umami.yaml`:
- Update `services.tracking.environment.DATABASE_URL` with the password from `secrets/umami_db_password`
- Update `services.tracking.deploy.labels -> traefik.http.routers.umami.rule` with your publicly accessible umami domain

In `config/umami.yaml`:
- Update `http.middlewares.umami-tracking-middleware.plugin.umami-feeder.umamiPassword` with the password from `secrets/umami_db_password`


### Copy config code to server `/mnt/docker-swarm-deploy` directory

On your local machine in the `alliance_setup_docs` folder

```shell
rsync --chown=:docker --chmod=771  -pgr ./ <YOUR FLOATING POINT IP ADDRESS>:/mnt/docker-swarm-deploy --delete --exclude-from='.rsync.ignore'
```

### deploy all the services on the instance

```shell
docker stack deploy -c /mnt/docker-swarm-deploy/postgres.yaml postgres
docker stack deploy -c /mnt/docker-swarm-deploy/traefik.yaml traefik
docker stack deploy -c /mnt/docker-swarm-deploy/monitoring.yaml monitoring
docker stack deploy -c /mnt/docker-swarm-deploy/smtp.yaml smtp
docker stack deploy -c /mnt/docker-swarm-deploy/umami.yaml umami
docker stack deploy -c /mnt/docker-swarm-deploy/anubis.yaml anubis
docker stack deploy -c /mnt/docker-swarm-deploy/jenkins.yaml jenkins
```

### Setup Umami Postgres User

```shell
docker exec -it $(docker ps -q -f name=postgres_db) bash

    PGPASSWORD="$(cat /run/secrets/postgres_root_password)" psql --username=root

        CREATE DATABASE umami;

        CREATE USER umami WITH PASSWORD '<REPLACE WITH secrets/umami_db_password>';
        ALTER DATABASE umami OWNER TO umami;
        GRANT ALL PRIVILEGES ON DATABASE umami TO umami;

    exit
```