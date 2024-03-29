# Signet Playground

Syncs with the global Signet for now, but the goal is to run a private Signet network.

## Requirements

Recent Docker version with the Compose v2 plugin.

```bash
$ docker compose version
Docker Compose version v2.25.0
```

## Setup

Start Bitcoin, Electrs and MariaDB first and follow the logs.
Should only take a few minutes, the Signet datadir is just 1.4 GB at the time of writing.

```bash
$ docker compose up -d knots electrs mariadb
$ docker compose logs -f
```

Once the Electrs backend completes its sync, start the mempool containers:

```bash
$ docker compose up -d mempool-api mempool-web
```

## Testing

### Mempool Explorer

Browse the Signet chain at http://localhost:8080

### Sparrow

Connect your Sparrow Wallet to Electrs:

1. Tools > Restart in Network > Signet
2. File > Preferences > Server > Private Electrum > localhost:60601, no SSL, no Tor proxy.
