# Signet Playground

A dockerized, self-contained Bitcoin [Signet network](https://en.bitcoin.it/wiki/Signet) to study how it works.
It consists of:

* Signet node
* Signet miner
* [Electrs](https://github.com/romanz/electrs) server
* [Mempool](https://github.com/mempool/mempool) explorer
* [Faucet](https://github.com/BcnBitcoinOnly/bbo-faucet) website

## Requirements

Recent Docker version with the Compose v2 plugin (i.e. `docker compose` instead of `docker-compose`).

## Setup

Start Knots, Electrs, MariaDB, Redis and the Faucet website.
Since Knots is booting in custom signet mode Knots and Electrs will finish their setup almost immediately, as there is no blockchain to download.

```bash
$ docker compose up -d knots electrs mariadb redis faucet
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
3. Wait until the blockchain has at least 100 blocks, then get some sats from the faucet at http://localhost:8123

### bitcoin-cli

Interact directly with the node via the command line by running the `bitcoin-cli` of the `knots` container:

```shell
$ docker compose exec knots bitcoin-cli -getinfo
Chain: signet
Blocks: 101
Headers: 101
Verification progress: 100.0000%
Difficulty: 0.001126515290698186

Network: in 1, out 0, total 1
Version: 260100
Time offset (s): 0
Proxies: n/a
Min tx relay fee rate (BTC/kvB): 0.00001000

Wallet: BBO
Keypool size: 1000
Transaction fee rate (-paytxfee) (BTC/kvB): 0.00000000

Balance: 50.00000000

Warnings: (none)
```


## Clean Up

Stop and remove all containers and data volumes with the usual Docker Compose command:

```bash
$ docker compose down -v
```


## Electrs `signet_magic` parameter

The magic bytes for a signet blockchain are derived from the `signetchallenge` parameter, so they change on each signet.
They are printed in one of the first log lines of Knots when it starts.

```shell
$ docker compose logs knots | less
knots-1  | 2024-07-19T20:52:32Z Feeding 38539 bytes of environment data into RNG
knots-1  | 2024-07-19T20:52:32Z Signet with challenge 51202527a1e6ac01f40d62fd6d325739414c3eb1fe94abe372d1989eadf58e895a2d
knots-1  | 2024-07-19T20:52:32Z Bitcoin Knots version v27.1.knots20240621 (release build)
knots-1  | 2024-07-19T20:52:32Z Signet derived magic (message start): 313e47cc
...
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

If we didn't have a third descriptor for the mining rewards, addresses retrieved manually with `bitcoin-cli getnewaddress` would
eventually be reused by the mining script.

The master private key must be known in advance because the `signetchallenge` parameter of Bitcoin Knots has to be hardcoded.
The value we chose for the challenge is the Taproot scriptPubKey corresponding to the address `86h/1h/0h/2/0`, as the block 0 has a special coinbase that cannot be spent.


# References

## Guides

* [BIP-325: Signet](https://bips.xyz/325)
* [Mempool Docker installation](https://github.com/mempool/mempool/blob/master/docker/README.md)
* [Custom Signet Tutorial](https://en.bitcoin.it/wiki/Signet#Custom_Signet)
* [How to import an xpriv to a descriptor wallet in bitcoin core?](https://bitcointalk.org/index.php?topic=5483885.msg63602317#msg63602317)
* [Signet mining is not possible when using descriptor wallet?](https://github.com/bitcoin/bitcoin/issues/28911)

## Docker images

* [1maa/bitcoin:v27.1.knots20240621](https://github.com/BcnBitcoinOnly/docker-knots/blob/master/Dockerfile)
* [1maa/bbo-faucet:latest](https://github.com/BcnBitcoinOnly/bbo-faucet/blob/master/Dockerfile)
* [1maa/signet-miner:latest](https://github.com/1ma/dockertronics/blob/master/bitcoin/signet-miner/Dockerfile)
* [1maa/electrs:latest](https://github.com/1ma/dockertronics/blob/master/electrs/Dockerfile)
