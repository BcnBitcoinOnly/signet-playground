# Signet Playground

A dockerized Bitcoin [Signet network](https://en.bitcoin.it/wiki/Signet) for local development or studying how Signet works.

## Services

| Container        | Internal Ports (Docker network)  | External Ports (localhost) | Comments                                                      |
|------------------|----------------------------------|----------------------------|---------------------------------------------------------------|
| Bitcoin node     | 8443, 28332, 28333, 38332, 38333 | `N/A`                      | Latest Knots release, ZMQ notifications, 60 second block time |
| Signet miner     | `N/A`                            | `N/A`                      | Configured to mine a block every 60 seconds                   |
| Fulcrum          | 8000, 60601                      | 60601                      |                                                               |
| Faucet website   | 8080                             | 8123                       | [BBO faucet], self-contained PHP webserver                    |
| Mempool frontend | 2019, 8080                       | 8080                       | [Retropex fork], Caddy webserver                              |
| Mempool backend  | 8999                             | `N/A`                      | [Retropex fork]                                               |
| MariaDB          | 3306                             | `N/A`                      | Requirement of Mempool backend                                |
| Valkey           | 6379                             | `N/A`                      | Redis fork, requirement of faucet and Mempool backend         |

All internal TCP ports that aren't exposed to the host by default can be exposed by adding
or expanding a `ports` section in `compose.yml` or writing a local `compose.override.yml` file.


## Requirements

Recent Docker version with the Compose v2 plugin (i.e. `docker compose` instead of `docker-compose`).

## Setup

```bash
$ docker compose up --wait
$ docker compose rm -f wallet-setup
```

## Testing

### Mempool Explorer

Browse the Signet chain at http://localhost:8080

### Sparrow

Connect your Sparrow Wallet to the signet Fulcrum:

1. Tools > Restart in Network > Signet
2. File > Preferences > Server > Private Electrum > localhost:60601, no SSL, no Tor proxy.
3. Wait until the blockchain has at least 100 blocks, then get some sats from the faucet at http://localhost:8123

### bitcoin-cli

Interact directly with the node via the command line by running the `bitcoin-cli` of the `node` container:

```shell
$ docker compose exec node bitcoin-cli --version
Bitcoin Knots RPC client version v29.2.knots20251110
Copyright (C) 2009-2025 The Bitcoin Knots developers
Copyright (C) 2009-2025 The Bitcoin Core developers

Please contribute if you find Bitcoin Knots useful. Visit
<https://bitcoinknots.org/> for further information about the software.
The source code is available from <https://github.com/bitcoinknots/bitcoin>.

This is experimental software.
Distributed under the MIT software license, see the accompanying file COPYING
or <https://opensource.org/licenses/MIT>
```


## Clean Up

Stop and remove all containers and data volumes with the usual Docker Compose command:

```bash
$ docker compose down -v
```


## Wallet Organization

The `wallet-setup` step has been carefully designed to create a wallet with these characteristics:

* It derives Taproot addresses.
* The miner has a fresh address for each coinbase reward.
* The wallet itself can still be used as a regular wallet from the command line.

By default, the `createwallet` command populates the wallet with 2 descriptors of each single-signature type (P2PKH, Nested SegWit, P2WPKH and P2TR).
However, we pass the `blank=true` modifier to create an empty wallet with no descriptors in order to control how we set it up.

We then import 3 different Taproot descriptors based on the same master private key.
The first two are the usual pair of descriptors for receiving and change addresses.
The third one is non-standard, and is only meant to be used by the miner process.
That's why it's marked as internal and inactive.

| Descriptor use case | Derivation path | Internal | Active  |
|---------------------|-----------------|----------|---------|
| Receiving addresses | `86h/1h/0h/0/*` | `false`  | `true`  |
| Change addresses    | `86h/1h/0h/1/*` | `true`   | `true`  |
| Mining rewards      | `86h/1h/0h/2/*` | `true`   | `false` |

The mining script will send each reward to addresses from the third descriptor corresponding to the block height, so the reward for block 123 will go to address `86h/1h/0h/2/123` and so on.
Bitcoin Knots is capable of spending UTXOs belonging to any of the three descriptors, but when sending bitcoin to itself it will generally do it an address of the first descriptor.
Similarly, it will use addresses from the second descriptor as change addresses.

The tpub descriptor for the miner can be obtained with `bitcoin-cli listdescriptors` after having imported the `86h/1h/0h/2/*` tprv.
The fingerprint and derivation path hints inside square brackets at the start of the descriptor are not necessary.

If we didn't have a third descriptor for the mining rewards, addresses retrieved manually with `bitcoin-cli getnewaddress` would
eventually be reused by the mining script.

The master private key must be known in advance because the `signetchallenge` parameter of Bitcoin Knots has to be hardcoded.
The value we chose for the challenge is the Taproot scriptPubKey corresponding to the address `86h/1h/0h/2/0`, as the block 0 has a special coinbase that cannot be spent.


# References

## Guides

* [BIP-325: Signet](https://bips.xyz/325)
* [Mempool Docker installation](https://github.com/Retropex/mempool/blob/master/docker/README.md)
* [Custom Signet Tutorial](https://en.bitcoin.it/wiki/Signet#Custom_Signet)
* [How to import an xpriv to a descriptor wallet in bitcoin core?](https://bitcointalk.org/index.php?topic=5483885.msg63602317#msg63602317)
* [Signet mining is not possible when using descriptor wallet?](https://github.com/bitcoin/bitcoin/issues/28911)

## Docker images

* [1maa/bitcoin:latest](https://github.com/BcnBitcoinOnly/docker-knots/blob/master/cmake/Dockerfile)
* [1maa/bbo-faucet:latest](https://github.com/BcnBitcoinOnly/bbo-faucet/blob/master/Dockerfile)


[BBO faucet]: github.com/BcnBitcoinOnly/bbo-faucet
[Retropex fork]: https://github.com/Retropex/mempool
