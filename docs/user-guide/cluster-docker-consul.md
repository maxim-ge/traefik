# Clustering / High Availability on Docker Swarm with Consul

This guide explains how to use Træfik in high availability mode in a Docker Swarm and with Let's Encrypt.

Why do we need Træfik in cluster mode? Running multiple instances should work out of the box?

If you want to use Let's Encrypt with Træfik, sharing configuration or TLS certificates between many Træfik instances, you need Træfik cluster/HA.

Ok, could we mount a shared volume used by all my instances? Yes, you can, but it will not work.
When you use Let's Encrypt, you need to store certificates, but not only.
When Træfik generates a new certificate, it configures a challenge and once Let's Encrypt will verify the ownership of the domain, it will ping back the challenge.
If the challenge is not knowing by other Træfik instances, the validation will fail.

For more information about challenge: [Automatic Certificate Management Environment (ACME)](https://github.com/ietf-wg-acme/acme/blob/master/draft-ietf-acme-acme.md#tls-with-server-name-indication-tls-sni)

## Prerequisites

You will need a working Docker Swarm cluster.

## Træfik configuration

In this guide, we will not use a TOML configuration file, but only command line flag.
With that, we can use the base image without mounting configuration file or building custom image.

What Træfik should do:

- Listen to 80 and 443
- Redirect HTTP traffic to HTTPS
- Generate SSL certificate when a domain is added
- Listen to Docker Swarm event

### EntryPoints configuration

TL;DR:

```shell    
$ traefik \
    --entrypoints=Name:http Address::80 Redirect.EntryPoint:https \
    --entrypoints=Name:https Address::443 TLS \
    --defaultentrypoints=http,https
```

To listen to different ports, we need to create an entry point for each.

The CLI syntax is `--entrypoints=Name:a_name Address:an_ip_or_empty:a_port options`.
If you want to redirect traffic from one entry point to another, it's the option `Redirect.EntryPoint:entrypoint_name`.

By default, we don't want to configure all our services to listen on http and https, we add a default entry point configuration: `--defaultentrypoints=http,https`.

### Let's Encrypt configuration

TL;DR:

```shell
$ traefik \
    --acme \
    --acme.storage=/etc/traefik/acme/acme.json \
    --acme.entryPoint=https \
    --acme.email=contact@mydomain.ca
```

Let's Encrypt needs 3 parameters: an entry point to listen to, a storage for certificates, and an email for the registration.

To enable Let's Encrypt support, you need to add `--acme` flag.

Now, Træfik needs to know where to store the certificates, we can choose between a key in a Key-Value store, or a file path: `--acme.storage=my/key` or `--acme.storage=/path/to/acme.json`.

For your email and the entry point, it's `--acme.entryPoint` and `--acme.email` flags.

### Docker configuration

TL;DR:

```shell
$ traefik \
    --docker \
    --docker.swarmmode \
    --docker.domain=mydomain.ca \
    --docker.watch
```

To enable docker and swarm-mode support, you need to add `--docker` and `--docker.swarmmode` flags.
To watch docker events, add `--docker.watch`.

### Full docker-compose file

```yaml
version: "3"
services:
  traefik:
    image: traefik:1.5
    command:
      - "--web"
      - "--entrypoints=Name:http Address::80 Redirect.EntryPoint:https"
      - "--entrypoints=Name:https Address::443 TLS"
      - "--defaultentrypoints=http,https"
      - "--acme"
      - "--acme.storage=/etc/traefik/acme/acme.json"
      - "--acme.entryPoint=https"
      - "--acme.OnHostRule=true"
      - "--acme.onDemand=false"
      - "--acme.email=contact@mydomain.ca"
      - "--docker"
      - "--docker.swarmmode"
      - "--docker.domain=mydomain.ca"
      - "--docker.watch"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    networks:
      - webgateway
      - traefik
    ports:
      - target: 80
        published: 80
        mode: host
      - target: 443
        published: 443
        mode: host
      - target: 8080
        published: 8080
        mode: host
    deploy:
      mode: global
      placement:
        constraints:
          - node.role == manager
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure
networks:
  webgateway:
    driver: overlay
    external: true
  traefik:
    driver: overlay
```

## Migrate configuration to Consul

We created a special Træfik command to help configuring your Key Value store from a Træfik TOML configuration file and/or CLI flags.

## Deploy a Træfik cluster

The best way we found is to have an initializer service.
This service will push the config to Consul via the `storeconfig` sub-command.

This service will retry until finishing without error because Consul may not be ready when the service tries to push the configuration.

The initializer in a docker-compose file will be:

```yaml
  traefik_init:
    image: traefik:1.5
    command:
      - "storeconfig"
      - "--web"
      [...]
      - "--consul"
      - "--consul.endpoint=consul:8500"
      - "--consul.prefix=traefik"
    networks:
      - traefik
    deploy:
      restart_policy:
        condition: on-failure
    depends_on:
      - consul
```

And now, the Træfik part will only have the Consul configuration.

```yaml
  traefik:
    image: traefik:1.5
    depends_on:
      - traefik_init
      - consul
    command:
      - "--consul"
      - "--consul.endpoint=consul:8500"
      - "--consul.prefix=traefik"
    [...]
```

!!! note
    For Træfik <1.5.0 add `acme.storage=traefik/acme/account` because Træfik is not reading it from Consul.

If you have some update to do, update the initializer service and re-deploy it.
The new configuration will be stored in Consul, and you need to restart the Træfik node: `docker service update --force traefik_traefik`.

## Full docker-compose file

```yaml
version: "3.4"
services:
  traefik_init:
    image: traefik:1.5
    command:
      - "storeconfig"
      - "--web"
      - "--entrypoints=Name:http Address::80 Redirect.EntryPoint:https"
      - "--entrypoints=Name:https Address::443 TLS"
      - "--defaultentrypoints=http,https"
      - "--acme"
      - "--acme.storage=traefik/acme/account"
      - "--acme.entryPoint=https"
      - "--acme.OnHostRule=true"
      - "--acme.onDemand=false"
      - "--acme.email=contact@jmaitrehenry.ca"
      - "--docker"
      - "--docker.swarmmode"
      - "--docker.domain=jmaitrehenry.ca"
      - "--docker.watch"
      - "--consul"
      - "--consul.endpoint=consul:8500"
      - "--consul.prefix=traefik"
    networks:
      - traefik
    deploy:
      restart_policy:
        condition: on-failure
    depends_on:
      - consul
  traefik:
    image: traefik:1.5
    depends_on:
      - traefik_init
      - consul
    command:
      - "--consul"
      - "--consul.endpoint=consul:8500"
      - "--consul.prefix=traefik"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    networks:
      - webgateway
      - traefik
    ports:
      - target: 80
        published: 80
        mode: host
      - target: 443
        published: 443
        mode: host
      - target: 8080
        published: 8080
        mode: host
    deploy:
      mode: global
      placement:
        constraints:
          - node.role == manager
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure
  consul:
    image: consul
    command: agent -server -bootstrap-expect=1
    volumes:
      - consul-data:/consul/data
    environment:
      - CONSUL_LOCAL_CONFIG={"datacenter":"us_east2","server":true}
      - CONSUL_BIND_INTERFACE=eth0
      - CONSUL_CLIENT_INTERFACE=eth0
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.role == manager
      restart_policy:
        condition: on-failure
    networks:
      - traefik

networks:
  webgateway:
    driver: overlay
    external: true
  traefik:
    driver: overlay

volumes:
  consul-data:
      driver: [not local]
```
