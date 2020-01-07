TTN Dashboard
=============

<!-- TOC START min:1 max:3 link:true asterisk:false update:true -->
- [Introduction](#introduction)
- [Requirements](#requirements)
  - [Linux host](#linux-host)
  - [Domain name](#domain-name)
- [Installation](#installation)
  - [Initial setup](#initial-setup)
  - [Managing Containers](#managing-containers)
  - [Adding Secure HTTP (HTTPS)](#adding-secure-http-https)
- [Data persistence](#data-persistence)
- [Configuring the dashboard -- overview](#configuring-the-dashboard----overview)
  - [Node-RED](#node-red)
    - [Getting data from The Things Network](#getting-data-from-the-things-network)
    - [Storing data in InfluxDB](#storing-data-in-influxdb)
  - [Grafana](#grafana)
- [Advanced Configuration](#advanced-configuration)
  - [Additional parameters](#additional-parameters)
  - [Monitoring your server](#monitoring-your-server)
  - [Use the NGINX reverse proxy for other containers](#use-the-nginx-reverse-proxy-for-other-containers)
<!-- TOC END -->

# Introduction
_TTN Dashboard_ provides an easy to setup environment for displaying [The Things Network](https://www.thethingsnetwork.org/) application data on a dashboard.

The setup is based on Containers orchestrated by [docker-compose](https://docs.docker.com/compose/).\
It does not require a lot of resources and can run on a small Linux cloud instance or a small ARM based device like a [Raspberry Pi](https://www.raspberrypi.org/).

The following Containers will be used:  
- [NGINX](https://www.nginx.com/) reverse proxy, composed of the following containers:
  - [nginx](https://hub.docker.com/_/nginx): the NGINX reverse proxy itself
  - [docker-gen](https://github.com/jwilder/docker-gen): a configuration generator for NGINX
  - [letsencrypt-nginx-proxy-companion](https://github.com/JrCs/docker-letsencrypt-nginx-proxy-companion): manages [Let’s Encrypt](https://letsencrypt.org/) certificates (optional)
- [Node-RED](https://nodered.org/): user friendly interface to collect data from [The Things Network](https://www.thethingsnetwork.org/), process it and store it into a database
- [InfluxDB](https://www.influxdata.com/): Time series database to store the data
- [Grafana](https://grafana.com/): provides web-based dashboard from the stored data
- [httpd](https://hub.docker.com/_/httpd): transient container providing the `htpasswd` utility to encrypt and store password for the web services

![schema](ttn-dashboard.png)

# Requirements
## Linux host
You will need a Linux host with `docker`, `docker-compose` and `git` installed.  
Minimum required versions are:
- `docker`: 17.06.0 or above
- `docker-compose`: 1.18.0 or above

However, for security reason, latest versions are recommended.

## Domain name
If you want to expose your dashboard to the Internet, you will need domain names.\
The NGINX reverse proxy uses the domain name to route the requests to either Grafana or Node-RED; you will needs two names pointing to your server -- e.g.: `ttn-grafana.example.com` and `ttn-node.example.com`.\
If you don't have your own domain you can use a free service like [Duck DNS](https://www.duckdns.org).

For local home usage (intranet) you can use a local DNS server or even the _hosts_ file of your computer.

# Installation
## Initial setup
1. Clone this repository and enter its directory:  
  ```
  git clone https://github.com/oracle/AmedeeBulle/ttn-dashboard.git
  cd ttn-dashboard
  ```
1. Copy the `.env.distr` file to `.env`  
  ```
  cp .env.distr .env
  ```
1. Customize the `.env` file to your needs, the following parameters are mandatory:  
  - Host names
    - `TTN_GRAFANA_HOST`: the hostname to access the Grafana dashboard -- e.g.: `ttn-grafana.example.com`
    - `TTN_NODE_RED_HOST`: the hostname to access the Node-RED interface -- e.g.: `ttn-node.example.com`
  - InfluxDB
    - `TTN_INFLUX_ADMIN_PASSWORD`: administration password
    - `TTN_INFLUX_PASSWORD`: password for the user with read/write access (used in Node-RED)
    - `TTN_INFLUX_READ_PASSWORD`: password for the user with read-only access (used in Grafana)
  - Grafana
    - `TTN_GRAFANA_ADMIN_PASSWORD`: password for the admin user (additional users are defined in Grafana itself)
1. Building containers.  
  The proxy containers are only available for the `x86-64`; if you run this setup on a different architecture, you will need tho build these as well.
    - `x86-64` architecture: this is the common case, only the `Node-RED` container needs to be built:
    ```
    docker-compose build
    ```
    - Non `x86-64` architecture (Raspberry Pi, ...):
    ```
    git submodule init
    git submodule update
    docker-compose -f docker-compose.yml -f docker-compose-build.yml build
    ```
1. Set a user/password to access the Node-RED instance.  
  This is important as there is no user management in Node-RED!
  ```
  docker-compose -f docker-compose.yml -f docker-compose-htpasswd.yml \
    run --rm htpasswd \
    ./bin/htpasswd -bc /htpasswd/<NODE_FQDN> <USER> <PASSWORD>
  ```
  with
    - `<NODE_FQDN>`: the Node-RED domain name (same as `TTN_NODE_RED_HOST`)
    - `<USER>`/`<PASSWORD>`: the username/password you want to use to authenticate to Node-RED
1. Start the containers:
  ```
  docker-compose up
  ```

The initial stat will take more time as the InfluxDB database needs to be initialised.

You should be able to browse to your new Node-RED/Grafana setup using the defined domain names.

## Managing Containers
To start the containers (in _detached_ mode):
```
docker-compose up -d
```

View the log (use `-f` to follow log output):
```
docker-compose logs [-f]
```

Stop (kill) the containers:
```
docker-compose kill
```

Stop and destroy the containers:
```
docker-compose down
```

## Adding Secure HTTP (HTTPS)
If you expose your setup to the Internet you should `HTTPS` to encrypt the traffic.

This will happen automatically once you enable the `letsencrypt-nginx-proxy-companion` container:

1. Set the `PROXY_EMAIL` variable in the `.env` configuration file.  
   Provide a valid email address so that Let's Encrypt can warn you about expiring certificates and allow you to recover your account.
1. Stop the containers (if they already run)
  ```
  docker-compose down
  ```
1. Enable `HTTPS` and restart the containers:
  ```
  ln -s docker-compose-https.yml docker-compose.override.yml
  docker-compose up -d && docker-compose logs -f
  ```

It might take a couple of minutes to get the Let's Encrypt certificates in place, after which you can enjoy Secure HTTP!

Note that the HTTPS configuration is only possible if your server is exposed to the Internet (either directly or through NAT) as Let's Encrypt needs to authenticate the service.

# Data persistence
Data (database, certificates, ...) is stored in docker named volumes and persists across container/system restart.

 If you really want/need to delete the volumes and the associated data, run:
 ```
 docker-compose down --volumes
 ```

# Configuring the dashboard -- overview
Detailed instruction on how to use Node-RED and Grafana is outside the scope of this project.

## Node-RED
### Getting data from The Things Network
_The Things Network Node-RED Nodes_ are already installed in the Node-RED instance. Information on how to configure these is available in the [Quickstart](https://github.com/TheThingsNetwork/nodered-app-node/blob/master/docs/quickstart.md#configuring-your-application)

### Storing data in InfluxDB

## Grafana

# Advanced Configuration

## Additional parameters
The setup can be further customised (user names, log level, ...) in the `.env`. It should be self-explanatory.

## Monitoring your server
If you monitor your server with `collectd`, you can send the data to your InfuxDB instance:

1. On the `collectd` side, ensure the `network` plugin is enabled and configure -- E.g. in `/etc/collectd.conf`:
  ```
  ...
  LoadPlugin network
  ...
  <Plugin network>
    # client setup:
    <Server "127.0.0.1" "25826">
      SecurityLevel Encrypt
      Username "MyCollectdUser"
      Password "MySecretPassword"
      ResolveInterval 14400
    </Server>
  ...
  </Plugin>
  ...
  ```
1. In the `.env` configuration file:
  ```
  # Monitoring servers with collectd:
  TTN_INFLUXDB_COLLECTD_ENABLED=true
  # Database name for collectd (by default use the initial DB)
  TTN_INFLUX_COLLECTD_DB=ttn
  # Port to bind on: do NOT bind on your public interface!
  TTN_INFLUXDB_BIND_ADDRESS=127.0.0.1:25826
  # Security level (none, sign, encrypt)
  INFLUXDB_COLLECTD_SECURITY_LEVEL=encrypt
  ```
1. Create a file named `auth_file` with the user/password pair:
  ```
  MyCollectdUser:MySecretPassword
  ```
1. Copy that file into the `ttn_influxdb` volume
```
$ # the actual volume name may vary depending on your configuration
$ docker volume ls | grep ttn_influxdb
local               ttn-dashboard_ttn_influxdb
$ # copy to the volume
$ docker cp auth_file ttn-dashboard_ttn_influxdb:/etc/collectd/
```
1. Re-start `collectd` and the containers

## Use the NGINX reverse proxy for other containers
The NGINX reverse proxy setup can be used for additional services running on your node, but they must use the same docker network.\
To achieve that, we have to use an external network which needs to be created separately:
```
docker network create proxy-net
docker-compose -f docker-compose.yml -f docker-compose-net.yml up -d
```
