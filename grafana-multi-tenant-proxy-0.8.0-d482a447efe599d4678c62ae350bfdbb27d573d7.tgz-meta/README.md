# Grafana - Multi tenant proxy

This project has been developed to make easy to deploy a Grafana app like [Loki](https://github.com/grafana/loki) in a multi-tenant way.

There is a lot of understanding about how it works.

- [https://github.com/grafana/loki/issues/701](https://github.com/grafana/loki/issues/701)
- [https://github.com/grafana/loki/issues/525](https://github.com/grafana/loki/issues/525)

This work was forked from https://github.com/k8spin/loki-multi-tenant-proxy/ and is almost based on [this issue comment](https://github.com/grafana/loki/issues/701#issuecomment-506504372).

It should work with all GrafanaLabs apps (Loki, Mimir, Tempo, Pyroscope...) but it is primarily meant to work with Loki. So the documentations and examples will mostly be based on [Loki](https://github.com/grafana/loki).

## What is it?

It is a basic golang proxy. It does basic auth, logs the requests and serves as a loki reverse proxy. I think this is the very core functionallity needed to manage a multi-tenant service.

Actually, [Grafana loki](https://github.com/grafana/loki) does not check the auth of any request. The multi-tenant mechanism is based in a request header: `X-Scope-OrgID`. So, if you have untrusted tenants, you have to ensure a tenant uses it's own tenant-id/org-id and does not use any id of other tenants.

Note *(temporary hard coded behavior)*:  
This proxy also accepts an OAuth token and associates it to the `read` user specified in the auth-config file in order to set the request header `X-Scope-OrgID` with the OrgID related to the `read` user.  
This way, we can validate OAuth token support in grafana-multi-tenant-proxy without changing the current behavior around OrgID management.

### Requirements

To use this proxy, you have to configure your server (ie [Loki](https://github.com/grafana/loki)) with `auth_enabled: true` as described in the [offical docs](https://github.com/grafana/loki/blob/v0.3.0/docs/operations.md#multi-tenancy).

Then put the proxy in front of your server instances, configure the auth proxy configuration, and run it.

### Run it

```bash
$ grafana-multi-tenant-proxy run --target-server http://localhost:3500 --port 3501 --auth-config ./my-auth-config.yaml
```

Where:

- `--port`: Port used to expose this proxy.
- `--target-server`: URL of your server instance.
- `--auth-config`: Authentication configuration file path.
- `--keep-orgid`: Don't change OrgID header.

#### Configure the proxy

The auth configuration is very simple. Just create a yaml file `my-auth-config.yaml` with the following structure:

```golang
// Authn Contains a list of users
type Authn struct {
	Users []User `yaml:"users"`
}

// User Identifies a user including the tenant
type User struct {
	Username string `yaml:"username"`
	Password string `yaml:"password"`
	OrgID    string `yaml:"orgid"`
}
```

Here is an example of a valid multi-user configuration:

```yaml
users:
  - username: User-a
    password: pass-a
    orgid: tenant-a
  - username: User-b
    password: pass-b
    orgid: tenant-b
```

A tenant can contains multiple users. But a user is tied to a simple tenant.

#### Configure the Grafana Loki clients, promtail

The default promtail configuration does not have any auth definition, so, after deploy this proxy you have to configure the promtail client configuration to point to this reverse proxy instead of pointing to the original grafana loki server.

Then, dont forget to setup your credential configuration. A simple multi-tenant promtail configuration should looks like:

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0
client:
  url: http://grafana-multi-tenant-proxy:3501/api/prom/push
  basic_auth:
    username: User-a
    password: pass-a
scrape_configs:
  - job_name: logs
    static_configs:
      - targets:
          - localhost
        labels:
          job: logs
          __path__: /var/logs/*
```

Note the `client` configuration. The original (single tenant) configuration was something similar to:

```yaml
client:
  url: http://loki-server:3500/api/prom/push
```

## Build it

If you want to build it from this repository, follow the instructions bellow:

```bash
$ docker run -it --entrypoint /bin/bash --rm golang:latest
root@6985c5523ed0:/go# git clone https://github.com/giantswarm/grafana-multi-tenant-proxy.git
Cloning into 'grafana-multi-tenant-proxy'...
remote: Enumerating objects: 88, done.
remote: Counting objects: 100% (88/88), done.
remote: Compressing objects: 100% (64/64), done.
remote: Total 88 (delta 26), reused 78 (delta 20), pack-reused 0
Unpacking objects: 100% (88/88), done
root@6985c5523ed0:/go# cd grafana-multi-tenant-proxy/cmd/grafana-multi-tenant-proxy/
root@6985c5523ed0:/go# go build
go: finding github.com/urfave/cli v1.21.0
go: finding gopkg.in/yaml.v2 v2.2.2
go: finding github.com/BurntSushi/toml v0.3.1
go: finding gopkg.in/check.v1 v0.0.0-20161208181325-20d25e280405
go: downloading github.com/urfave/cli v1.21.0
go: downloading gopkg.in/yaml.v2 v2.2.2
go: extracting github.com/urfave/cli v1.21.0
go: extracting gopkg.in/yaml.v2 v2.2.2
root@6985c5523ed0:/go# ./grafana-multi-tenant-proxy
NAME:
   Grafana Multitenant Proxy - Makes Grafana Labs applications multi tenant

USAGE:
   grafana-multi-tenant-proxy [global options] command [command options] [arguments...]

VERSION:
   dev

AUTHOR:
   Ángel Barrera - @angelbarrera92

COMMANDS:
   run      Runs the Grafana multi tenant proxy
   help, h  Shows a list of commands or help for one command

GLOBAL OPTIONS:
   --help, -h     show help
   --version, -v  print the version
```

### Build the container image

If you want to build a container image with this proxy, simply run:

```bash
$ docker build -t grafana-multi-tenant-proxy:local -f build/package/Dockerfile .
```

After built, just run it:

```bash
$ docker run --rm grafana-multi-tenant-proxy:local
```
