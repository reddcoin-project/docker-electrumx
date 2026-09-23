reddcoincore/electrumx
======================

[![Build Status]][builds]
[![gh_last_release_svg]][gh_last_release_url]
[![Docker Image Size]][docker-hub]
[![Docker Pulls Count]][docker-hub]

[Build Status]: https://github.com/reddcoin-project/docker-electrumx/actions/workflows/on-tag.yml/badge.svg
[builds]: https://github.com/reddcoin-project/docker-electrumx/actions/workflows/on-tag.yml
[gh_last_release_svg]: https://img.shields.io/github/v/release/reddcoin-project/docker-electrumx?sort=semver
[gh_last_release_url]: https://github.com/reddcoin-project/docker-electrumx/releases/latest
[Docker Image Size]: https://img.shields.io/docker/image-size/reddcoincore/electrumx
[Docker Pulls Count]: https://img.shields.io/docker/pulls/reddcoincore/electrumx.svg?style=flat
[docker-hub]: https://hub.docker.com/r/reddcoincore/electrumx

> Run a Reddcoin Electrum server with one command

This repo packages the [Reddcoin fork of ElectrumX][reddcoin-electrumx] into a Docker image for `linux/amd64` and `linux/arm64`. It serves Electrum clients such as [Redd-Electrum] and needs a fully synced [`reddcoind`] node with RPC enabled — the [`reddcoincore/reddcoind`][reddcoind-image] image works well for this.

[reddcoin-electrumx]: https://github.com/reddcoin-project/electrumx
[Redd-Electrum]: https://github.com/reddcoin-project/electrum-redd
[`reddcoind`]: https://github.com/reddcoin-project/reddcoin
[reddcoind-image]: https://hub.docker.com/r/reddcoincore/reddcoind

> The work here is based on [lukechilds/docker-electrumx](https://github.com/lukechilds/docker-electrumx).


## Tags

> **NOTE:** For an always up-to-date list see: https://hub.docker.com/r/reddcoincore/electrumx/tags

* `latest` — most recent build from `master`
* `v<version>` (e.g. `v1.20.1`) — built from the matching tag of [reddcoin-project/electrumx][reddcoin-electrumx], with a matching [GitHub Release](https://github.com/reddcoin-project/docker-electrumx/releases)


## Usage

### Pull

```shell
docker pull reddcoincore/electrumx
```

### Run

```shell
docker run -d \
  --name reddcoin-electrumx \
  -v /path/to/electrumx:/data \
  -e COIN=Reddcoin \
  -e DAEMON_URL=http://rpcuser:rpcpassword@reddcoind-host:45443 \
  -p 50002:50002 \
  reddcoincore/electrumx
```

> **IMPORTANT:** The image defaults to `COIN=Bitcoin` (inherited from upstream). Always set `COIN=Reddcoin` (or `COIN=ReddcoinTestnet` with `NET=testnet` for testnet).

`DAEMON_URL` must point at the RPC interface of your `reddcoind` node (the default Reddcoin mainnet RPC port is `45443`). Make sure `reddcoind` allows RPC connections from the ElectrumX container (`rpcallowip`).

On first start ElectrumX indexes the whole Reddcoin chain into `/data`, which takes a while. Clients can connect once it has caught up with the node.

### Docker Compose

Running `reddcoind` and ElectrumX side by side on a private network:

```yaml
services:
  reddcoin-server:
    container_name: reddcoin-server
    image: reddcoincore/reddcoind:v4.22.9
    restart: on-failure
    volumes:
      - ./data/reddcoin:/home/reddcoind/.reddcoin
    environment:
      - RPC_SERVER=1
      - RPC_USERNAME=rpcuser
      - RPC_PASSWORD=change-me
      - RPC_PORT=45443
      - RPC_ALLOW_IP=172.28.0.0/16
    stop_grace_period: 5m
    networks:
      - reddnet

  reddcoin-electrumx:
    container_name: reddcoin-electrumx
    image: reddcoincore/electrumx:latest
    restart: always
    volumes:
      - ./data/electrumx:/data
    environment:
      - COIN=Reddcoin
      - DAEMON_URL=http://rpcuser:change-me@reddcoin-server:45443
      - REPORT_SERVICES=ssl://electrum.example.com:50002,wss://electrum.example.com:50004
    ports:
      - "50002:50002"
      - "50004:50004"
    networks:
      - reddnet

networks:
  reddnet:
    ipam:
      config:
        - subnet: 172.28.0.0/16
```

Set `REPORT_SERVICES` to the public hostname of your server so it can be announced to peers and clients.

### Configuration

The image sets these defaults, all of which can be overridden with `-e`:

| Variable            | Default                                                      |
|---------------------|--------------------------------------------------------------|
| `COIN`              | `Bitcoin` — **set this to `Reddcoin`**                       |
| `DB_DIRECTORY`      | `/data`                                                      |
| `SERVICES`          | `tcp://:50001,ssl://:50002,wss://:50004,rpc://0.0.0.0:8000`  |
| `SSL_CERTFILE`      | `/data/electrumx.crt`                                        |
| `SSL_KEYFILE`       | `/data/electrumx.key`                                        |
| `EVENT_LOOP_POLICY` | `uvloop`                                                     |
| `ALLOW_ROOT`        | `1`                                                          |

All ElectrumX environment variables are documented here: https://electrumx-spesmilo.readthedocs.io/en/latest/environment.html

### SSL certificate

If there's an SSL certificate/key (`electrumx.crt`/`electrumx.key`) in the `/data` volume it'll be used. If not, a self-signed one will be generated on first start.

### Ports

| Port    | Service                    | Notes                                                        |
|---------|----------------------------|--------------------------------------------------------------|
| `50001` | TCP (unencrypted)          | Exposing it with `-p 50001:50001` is strongly discouraged    |
| `50002` | SSL                        | The main port for Electrum clients                           |
| `50004` | WebSocket (WSS)            | For browser-based wallets                                    |
| `8000`  | RPC                        | Only expose to localhost: `-p 127.0.0.1:8000:8000`           |

To use the RPC interface from inside the container there's no need to expose the port:

```shell
docker exec reddcoin-electrumx electrumx_rpc getinfo
```

### Version

To run a specific version, use its tag:

```shell
docker run \
  -v /path/to/electrumx:/data \
  -e COIN=Reddcoin \
  -e DAEMON_URL=http://rpcuser:rpcpassword@reddcoind-host:45443 \
  -p 50002:50002 \
  reddcoincore/electrumx:v1.20.1
```


## Building

```shell
docker build --build-arg VERSION=1.20.1 -t reddcoincore/electrumx:v1.20.1 .
```

`VERSION` is a branch or tag of [reddcoin-project/electrumx][reddcoin-electrumx].

### Releasing

1. Bump `ARG VERSION` in the [`Dockerfile`](Dockerfile) and push to `master` — [this](.github/workflows/on-master-push.yml) builds and publishes `latest`.
2. Tag the commit with the same version and push the tag:

   ```shell
   git tag -a v1.20.1 -m "ElectrumX 1.20.1"
   git push origin v1.20.1
   ```

   [This](.github/workflows/on-tag.yml) checks the tag matches the `Dockerfile`, publishes `reddcoincore/electrumx:v1.20.1` and creates the GitHub Release.


## License

MIT © Luke Childs, Reddcoin Core Developers
