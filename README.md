# Signet Playground

A dockerized, self-contained Bitcoin [Signet network](https://en.bitcoin.it/wiki/Signet) to study how it works.

## Requirements

Recent Docker version with the Compose v2 plugin.

```bash
$ docker compose version
Docker Compose version v2.25.0
```

## Setup

Start Knots, Electrs and MariaDB.
Since Knots is booting in custom signet mode Knots and Electrs will finish their setup almost immediately, as there is no blockchain to download.

```bash
$ docker compose up -d knots electrs mariadb
```

Now run a one-time process that will store in Knots' wallet the private key that is used for the signet challenge:

```bash
$ docker compose run --rm wallet-setup
{
  "name": "BBO"
}
```

With the private key stored in Knots you can now start the miner and mempool processes:

```bash
$ docker compose up -d miner mempool-api mempool-web
```

## Testing

### Mempool Explorer

Browse the Signet chain at http://localhost:8080

### Sparrow

Connect your Sparrow Wallet to Electrs:

1. Tools > Restart in Network > Signet
2. File > Preferences > Server > Private Electrum > localhost:60601, no SSL, no Tor proxy.


## Clean Up

Stop and remove all containers and data volumes with the usual Docker Compose command:

```bash
$ docker compose down -v
```


# References

## Guides

* [BIP-325: Signet](https://bips.xyz/325)
* [Mempool Docker installation](https://github.com/mempool/mempool/blob/master/docker/README.md)
* [Custom Signet Tutorial](https://en.bitcoin.it/wiki/Signet#Custom_Signet)

## Docker images

* [1maa/bitcoin:v26.1.knots20240325](https://github.com/BcnBitcoinOnly/docker-knots/blob/master/Dockerfile)
* [1maa/signet-miner:latest](https://github.com/1ma/dockertronics/blob/master/bitcoin/signet-miner/Dockerfile)
* [1maa/electrs:latest](https://github.com/1ma/dockertronics/blob/master/electrs/Dockerfile)
