# Setup Conjur Open Source Suite on RHEL with Podman
A guide to setup a minimal Conjur OSS on Podman using `podman play kube` with base policies for MySQL and AWS API keys

## Introduction
- The official [Conjur OSS quick start](https://github.com/cyberark/conjur-quickstart) uses `docker-compose` which includes several containers:
  - openssl
  - bot_app
  - database (postgres)
  - conjur
  - proxy (nginx)
  - client (conjur-cli)
- Components dropped:
  - `openssl`: used to generate certificates for the Conjur server, personal/enterprise certificate chains are used in most cases
    - This demo uses a personal self-signed certificate case
  - `bot_app`: a demo app, not needed
  - `client`: Conjur CLI, there's a new v7 python-based Conjur CLI instead (<https://github.com/cyberark/conjur-api-python3/releases>)
- Deploying on Podman
  - Why Podman?
    - RHEL 8 does not officially support Docker
    - Podman uses a socket, Docker uses a daemon
    - No more `containerd`
  - Docker Compose with Podman: Podman has added support for Docker Compose since version 3.0 (<https://www.redhat.com/sysadmin/podman-docker-compose>)
  - But there is a better way to run multiple containers as a collection on Podman using `podman play kube`
    - Uses Kubernetes manifest file to define pods on podman (<https://docs.podman.io/en/latest/markdown/podman-play-kube.1.html>)
    - All containers in the pod shares the same IP address - this means that containers in the pod can find each other using `localhost`
    - Start-on-boot of pods is done by systemd unit file - pods are managed like a service

## Software Versions
- RHEL 8.5
- Podman 3.3.1
- Conjur OSS 1.15
- Postgres 14.2
- Nginx 1.21.6

Note: The `:latest` tag is used for all container images, the latest version are stated as above at the time of writing

# 1.0 Setup host prerequisites
## 1.1 Install necessary packages
- Install `podman` and `policycoreutils-python-utils` packages
- The `policycoreutils-python-utils` is required for the `semanage fcontext` command, which is used to allow the Postgres container to access the data directory on the host
```console
yum install -y podman policycoreutils-python-utils
systemctl enable --now podman
```

## 1.2 Setup Conjur CLI
- The official docker-compose from CyberArk uses the v5 container/ruby-based Conjur CLI, there's a new v7 python-based Conjur CLI (<https://github.com/cyberark/conjur-api-python3/releases>)
- The container-based Conjur CLI will also be deprecated in June 2022 (<https://docs.conjur.org/Latest/en/Content/Tools/CLI_Help.htm>)
```console
curl -L -o conjur-cli-rhel-8.tar.gz https://github.com/cyberark/conjur-api-python3/releases/download/v7.1.0/conjur-cli-rhel-8.tar.gz
tar xvf conjur-cli-rhel-8.tar.gz
mv conjur /usr/local/bin/
```
- Clean-up
```console
rm -f conjur-cli-rhel-8.tar.gz
```

## 1.3 Prepare data directores
- There are 3 directories used by Conjur OSS for data persistence
  - Data directory
    - Conjur data is all stored in the Postgres database, as long as this directory persist, you can easily delete and redeploy the pod with the data intact
    - Mount `host:/opt/conjur/data` to `database:/var/lib/postgresql/data`
    - To allow the Postgres container access to the data directory, the SELinux type label needs to be assigned to `svirt_sandbox_file_t`
    - SELinux can also simply be disabled, but that is not preferred
  - Nginx config directory
    - The reverse proxy configuration `default.conf` is to be stored here (Ref: <https://github.com/joetanx/conjur-oss/blob/main/default.conf>)
    - Mount `host:/opt/conjur/conf` to `/etc/nginx/conf.d`
  - Nginx certificates directory
    - Stores the certificates
    - Mount `host:/opt/conjur/conf/tls` to `/etc/nginx/tls`
```console
mkdir -p /opt/conjur/{data,conf/tls}
semanage fcontext -a -t svirt_sandbox_file_t /opt/conjur/data
restorecon -v /opt/conjur/data
```
- Pull the Nginx configuration file and certificates
  - The official docker-compose from CyberArk includes an `openssl` container to generate certificate for Conjur, this demo uses a personal certificate chain
  - The certificates pulled below are for `conjur.vx` hostname and are signed by a personal CA `central.pem`
  - To generate your own certificate chain, read: <https://joetanx.github.io/self-signed-ca/>
```console
curl -L -o /opt/conjur/conf/default.conf https://github.com/joetanx/conjur-oss/raw/main/default.conf
curl -L -o /opt/conjur/conf/tls/nginx.pem https://github.com/joetanx/conjur-oss/raw/main/conjur.vx.pem
curl -L -o /opt/conjur/conf/tls/nginx.key https://github.com/joetanx/conjur-oss/raw/main/conjur.vx.key
```

# 2.0 Deploy Conjur OSS
- Download the pod manifest file:
```console
curl -L -o conjur-oss.yaml https://github.com/joetanx/conjur-oss/raw/main/conjur-oss.yaml
```

## 2.1 Prepare data key
- Secrets and API keys in Conjur are encrypted with AES-256-GCM before storing in the database
- The Conjur service has a unique 256-bit data key that is created when Conjur is first configured
- **Note**
  - It is critical to store this data key securely
  - Losing or exposing this data key means potentially losing access to or exposing the Conjur data stored in the database
- 
- Generate the data key and inject to the pod manifest for deployment
```console
export CONJUR_DATA_KEY="$(podman run --rm docker.io/cyberark/conjur data-key generate)"
echo "      value: $CONJUR_DATA_KEY" >> conjur-data-key
sed -i '/<conjur-data-key>/ r conjur-data-key' conjur-oss.yaml
sed -i '/<conjur-data-key>/d' conjur-oss.yaml
```
- Remove the data key after injecting into the pod manifest
- **Note**: Backup and store the data key securely elsewhere before removing it!!!
```console
rm -f conjur-data-key
```

## 2.2 Deploy Conjur OSS
- Deploy Conjur OSS:
```console
podman play kube conjur-oss.yaml
```
- Verify that the pod and all containers are running:
```console
podman pod ps
podman ps
```
- Test access to Conjur OSS status page (replace `conjur.vx` with your host FQDN):
```console
curl -k https://conjur.vx
```
- Clean-up
```console
rm -f conjur-oss.yaml
```

## 2.3 Generate systemd unit files to start Conjur OSS on boot
- Containers on podman are dependent on systemd to start on boot
  - The below steps generate systemd unit files for the pod and each container in the pod
  - Enabling the pod's systemd unit file will start the pod on boot and trigger the containers automatically
- Ref: <https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux_atomic_host/7/html/managing_containers/running_containers_as_systemd_services_with_podman>
```console
cd /etc/systemd/system
podman generate systemd conjur-oss -n --restart-policy=always --files
systemctl enable pod-conjur-oss
```

# 3.0 Initialize Conjur OSS
- The initialization is done by running `conjurctl account create` command in the Conjur container
- The below command creates a Conjur account named `cyberark`, you can use any account name you want
- Initialization creates the default `admin` user with a generated API key
- The `| tee admin_data` portion send the output to a file named `admin_data` as well as display it on screen (`stdout`)
```console
podman exec conjur-oss-conjur conjurctl account create cyberark | tee admin_data
```

## 3.1 Test Conjur CLI login
- Initialize the Conjur CLI and login as `admin`
- Verify login with `conjur whoami` and set the admin password
```console
conjur init -u https://conjur.vx -a cyberark
conjur login -i admin
conjur whoami
conjur user change-password -p CyberArk123!
```

# 4.0 Staging secret variables
- Integration guides in my GitHub uses the secrets set in this step
- Pre-requisites
  - Setup MySQL database according to this guide: <https://joetanx.github.io/mysql-world_db>
  - Have an AWS IAM user account with programmatic access
- Credentials are configured by `app-vars.yaml` in `world_db` and `aws_api` policies that are defined with the respective secret variables
- Download the Conjur policies
```console
curl -L -o app-vars.yaml https://github.com/joetanx/conjur-oss/raw/main/app-vars.yaml
```
- Load the policies to Conjur
```console
conjur policy load -b root -f app-vars.yaml
```
- Populate the variables
```console
conjur variable set -i world_db/username -v cityapp
conjur variable set -i world_db/password -v Cyberark1
conjur variable set -i aws_api/awsakid -v <AWS_ACCESS_KEY_ID>
conjur variable set -i aws_api/awssak -v <AWS_SECRET_ACCESS_KEY>
```
- Clean-up
```console
rm -f app-vars.yaml
```
