# ttn-private-network
Manual and files to setup a private TheThingNetwork (TTN)

Based on the guides [Setting up a Private Routing Environment](https://www.thethingsnetwork.org/article/setting-up-a-private-routing-environment) and [Deploying a Private Routing Environment With Docker-Compose](https://www.thethingsnetwork.org/article/deploying-a-private-routing-environment-with-docker-compose)
**Please read through the guides** at least once to get a feeling for what is setup here.

# Setup

## Prerequisites

* Install and start [Docker](https://www.docker.com/products/docker).
* Install [Docker-Compose](https://www.docker.com/products/docker-compose)

Checkout the repository. The docker-compose environment is in the folter `ttn`. All commands are executed from this directory. Configuration files and container state will endup in subdirectories and are mounted inside the container to `/etc/ttn/<component>/`

## Setup Certificates

Before starting up all the containers we have to generate certificates and keys.

```
docker-compose run --rm ttn discovery gen-keypair --config /etc/ttn/discovery/ttn.yml
# This will be valid for localhost, discovery and discovery.local.thethings.network.
docker-compose run --rm ttn discovery gen-cert localhost discovery discovery.local.thethings.network --config /etc/ttn/discovery/ttn.yml --key-dir /etc/ttn/discovery

docker-compose run --rm ttn router gen-keypair --config /etc/ttn/router/ttn.yml
# This will be valid for localhost, router and router.local.thethings.network (the last one comes from the configuration).
docker-compose run --rm ttn router gen-cert localhost router --config /etc/ttn/router/ttn.yml --key-dir /etc/ttn/router

docker-compose run --rm ttn broker gen-keypair --config /etc/ttn/broker/ttn.yml
# This will be valid for localhost, broker and broker.local.thethings.network.
docker-compose run --rm ttn broker gen-cert localhost broker --config /etc/ttn/broker/ttn.yml --key-dir /etc/ttn/broker

docker-compose run --rm ttn networkserver gen-keypair --config /etc/ttn/networkserver/ttn.yml
# This will be valid for localhost, networkserver and networkserver.local.thethings.network.
docker-compose run --rm ttn networkserver gen-cert localhost networkserver networkserver.local.thethings.network --config /etc/ttn/networkserver/ttn.yml --key-dir /etc/ttn/networkserver

cat ./discovery/server.cert > ./handler/ca.cert
docker-compose run --rm ttn handler gen-keypair --config /etc/ttn/handler/ttn.yml
# This will be valid for localhost, handler and handler.local.thethings.network.
docker-compose run --rm ttn handler gen-cert localhost handler --config /etc/ttn/handler/ttn.yml --key-dir /etc/ttn/handler
```
	
The discovery server's new certificate is needed by the Router, Broker, Handler, Bridge and by ttnctl, so we add this to the trusted certificates:

```
cat discovery/server.cert > router/ca.cert
cat discovery/server.cert > broker/ca.cert
cat discovery/server.cert > handler/ca.cert
cat discovery/server.cert > bridge/ca.cert
cat networkserver/server.cert > broker/networkserver.cert
cat discovery/server.cert > ~/.ttnctl/ca.cert
```

	
## Setup Tokens

The tokens need to be generated and added to the ttn.yml file.

```
# tokens for broker
# -> ./broker/ttn.yml #DISCOVERY_TOKEN_FOR_BROKER
docker-compose run --rm  ttn discovery authorize broker mynetwork-broker --config /etc/ttn/discovery/ttn.yml
# -> ./broker/ttn.yml #NETWORKSERVER_TOKEN_FOR_BROKER
docker-compose run --rm  ttn networkserver authorize mynetwork-broker --config /etc/ttn/networkserver/ttn.yml	
		
# token for router -> ./router/ttn.yml #DISCOVERY_TOKEN_FOR_ROUTER
docker-compose run --rm  ttn discovery authorize router mynetwork-router --config /etc/ttn/discovery/ttn.yml

# token for the handler  -> ./handler/ttn.yml #DISCOVERY_TOKEN_FOR_HANDLER
docker-compose run --rm  ttn discovery authorize handler mynetwork-handler --config /etc/ttn/discovery/ttn.yml
```

## Startup the discovery

```
docker-compose up -d discovery
```

Just as in [this guide](https://www.thethingsnetwork.org/article/setting-up-a-private-routing-environment) we have to register the device address prefix:

> Now we will tell the discovery server which `DevAddr` prefix we will handle with this Broker. In this case, we use a prefix that TTN reserved for private networks: `26000000/20`. This prefix allows you to issue 4096 distinct addresses to devices in your private network, which should be more than enough (remember that device addresses are not unique; you can easily give 100 devices the exact same address). If you need a larger address space, you should file a request with The Things Network Foundation. If you are setting up a large network, you might have to apply for your own NetID from the LoRa Alliance, which will give you your own prefix.

```
docker-compose run --rm broker broker register-prefix 26000000/20 --config /etc/ttn/broker/ttn.yml
```

## Startup everything

```
docker-compose up -d
```

Check if everything is running:

```
docker ps -a
```

For stopped containers you can check the logs with `docker logs <container_name>`

For further Questions or Issues please use the [forum](https://www.thethingsnetwork.org/forum/t/setting-up-a-private-routing-environment/4445) or the `#private-backend` channel on Slack. Feel free to also open github issues in this repository. This is a community-supported guide, so please help each other out.

## ttnctl


ttnctl also needs to have the discovery server certificate, for that we have to create a `ca.cert` file in `$XDG_DATA_HOME/ttnctl`, `$XDG_CACHE_HOME/ttnctl` or `~/.ttnctl/`

```
cat discovery/server.cert >> ./ttnctl/.ttnctl/ca.cert
```

Now you can register your application to your private Handler:

```
docker-compose run --rm ttnctl applications register
```

And manage your devices in the same way as you would do on the public community network.

# Security

## Redis
Redis should be run in a local context. For a production environment you should remove the `redis` ports from the `docker-compose.yml`

```
 ports:
      - "6379:6379" # Note: you should not expose this port in production environments
```

The redis password is configured in the `.env` file and used to configure all TTN services. Change the password for your environment!

## Mosquitto (MQTT)

Add `MQTT_USERNAME` and `MQTT_PASSWORD` to `.env` to configure the services.

First create the passwords file using the `mosquitto_passwd` tool inside the mosquitto docker container:

```
docker-compose run --rm mosquitto mosquitto_passwd -b /mosquitto/config/passwords <username> <password>
```

You should now have the username and the encrypted password in `mosquitto/config/passwords`

Now update the Mosquitto config to support authentication in `mosquitto/config/mosquitto.conf`:
```
allow_anonymous false
password_file /mosquitto/config/passwords
```

Restart the docker container: `docker-compose restart mosquitto`

You can also add users for all other MQTT clients, app level authentication is not enabled yet (looking for hints how to realize that).


