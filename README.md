# Pi Node Docker Image

This is a fork of [PiCoreTeam/pi-node-docker](https://github.com/PiCoreTeam/pi-node-docker) upgraded for:

- **Protocol 22 → 23** (stellar-core 23.*)
- **Ubuntu 20 → 24** (base image)
- **PostgreSQL 12 → 16**
- **Multi-network support**: mainnet, testnet1 (original), testnet2

## Software Versions

- **PostgreSQL 16**
- **stellar-core** 23.*
- **horizon** 2.*
- **Supervisord** — process manager
- **stellar-archivist** — history archive management (optional)

## Usage

### 1. Choose a Network

| Flag | Network | Passphrase | History CDN |
|------|---------|------------|-------------|
| `--mainnet` | Pi Mainnet | `Pi Network` | `https://history.mainnet.minepi.com` |
| `--testnet` | Pi Testnet (original) | `Pi Testnet` | `https://history.testnet.minepi.com` |
| `--testnet2` | Pi Testnet2 | `Pi Testnet` | `https://history.temp.testnet2.minepi.com` |

The **Go Horizon API** uses these corresponding `NETWORK=` env var values:
| Docker flag | NETWORK= (for Go Horizon) |
|-------------|--------------------------|
| `--mainnet` | `pimainnet` |
| `--testnet` | `pitestnet1` |
| `--testnet2` | `pitestnet2` (or `pitestnet` for backward compat) |

### 2. Choose Ports to Expose

The software listens on several ports. At minimum, expose the horizon HTTP port (8000).

### 3. Mount a Data Volume

You **must** mount a host directory to `/opt/stellar`:

```shell
$ docker run --rm -it -p "31401:8000" -v "/path/to/data:/opt/stellar" --name pi-node pinetwork/pi-node-docker:organization_mainnet-v1.0-p23.0 --mainnet
```

### Configuring the Original Pi Testnet (testnet1)

The `testnet/` directory contains **placeholder validators** marked with `__TESTNET_VALIDATOR*__`.
You must replace these before running the image on the original Pi Testnet.

#### How to find the validator keys

**Option A — From a running Pi Testnet node's config:**
If you have access to a node already connected to the original Pi Testnet, look at its `stellar-core.cfg`:

```shell
$ grep -A4 '\[\[VALIDATORS\]\]' /opt/stellar/core/etc/stellar-core.cfg
```

Copy the `PUBLIC_KEY`, `ADDRESS`, and `HISTORY` values into:

```
pi-node-docker/testnet/core/etc/stellar-core.cfg
pi-node-docker/testnet/horizon/etc/stellar-core-captive.yml
```

**Option B — From the Pi Core Team:**
The validator configuration is published by the Pi Core Team. Check:
- https://github.com/PiCoreTeam/pi-node-docker — look at `testnet/core/etc/stellar-core.cfg`
- https://github.com/PiCoreTeam — community announcements

**Option C — From a running stellar-core node via HTTP API:**
If you have a stellar-core node running on the old testnet, query its quorum set:

```shell
$ curl http://<node-ip>:11626/info | jq .info.quorum
```

This returns the public keys of validators in the quorum set.

**Option D — From the Pi Network blockchain explorer or community:**
Look up the Pi Testnet validator set in the Pi blockchain explorer or community documentation.
Common validator key prefixes for Pi Core Team: `GDLA3KOM...`, `GDFDDPMCL...`, `GAOBNDXT...`.

#### Understanding the validator config format

Each validator entry needs three fields:

```toml
[[VALIDATORS]]
NAME="validatorN"                                          # Any descriptive name
HOME_DOMAIN="pi-core-team"                                # Domain for the validator
PUBLIC_KEY="G..."                                         # The validator's public key (starts with G)
ADDRESS="<ip>:31402"                                      # IP and peer port
HISTORY="curl -sf https://history.testnet.minepi.com/{0} -o {1}"  # Archive CDN URL
```

**Note:** Both testnets (original and testnet2) share the same passphrase `"Pi Testnet"`.
They are distinguished by different validator sets and history archive URLs.
If the old testnet has been merged into or replaced by testnet2, you only need the testnet2 configuration.

#### After filling in the validators

1. Verify the config is valid:
```shell
$ stellar-core --conf pi-node-docker/testnet/core/etc/stellar-core.cfg --ll info --checkquorum
```

2. Build and run:
```shell
$ cd pi-node-docker
$ docker build --platform linux/amd64 -t pinetwork/pi-node-docker:organization_mainnet-v1.0-p23.0 .
$ docker run --rm -it -p "31401:8000" -v "/path/to/data:/opt/stellar" --name pi-node pinetwork/pi-node-docker:organization_mainnet-v1.0-p23.0 --testnet
```

### Background vs. Interactive Containers

1. Run an interactive session first, ensuring all services start correctly.
2. You will be prompted to set a PostgreSQL password (or set `POSTGRES_PASSWORD` env var).
3. Shut down the interactive container (using Ctrl-C).
4. Start a new container using the same host directory in the background.

### Customizing Configurations

Default configurations are copied to the data directory on first launch:

```
/opt/stellar
├── core
│   └── etc
│       └── stellar-core.cfg    # stellar-core configuration
├── horizon
│   └── etc
│       ├── horizon.env         # Horizon environment variables
│       └── stellar-core-captive.yml  # Captive core configuration
├── postgresql
│   └── etc
│       ├── postgresql.conf     # PostgreSQL configuration
│       ├── pg_hba.conf         # PostgreSQL client authentication
│       └── pg_ident.conf       # PostgreSQL user mapping
├── supervisor
│   └── etc
│       └── supervisord.conf    # Supervisord configuration
├── history/
│   └── local/                  # Local history archive
└── migration_status            # Tracks executed migrations (auto-created)
```

Stop the container before editing configuration files, then restart after changes.

**WARNING:** Incorrect configuration edits can break services.

## Command Line Options

| Option                     | Description                               |
|----------------------------|-------------------------------------------|
| `--mainnet`                | Connect to Pi Network mainnet             |
| `--testnet`                | Connect to Pi Testnet (original)          |
| `--testnet2`               | Connect to Pi Testnet2                    |
| `--enable-auto-migrations` | Run database/config migrations on startup |

## Environment Variables

| Variable            | Description                                                                         |
|---------------------|-------------------------------------------------------------------------------------|
| `POSTGRES_PASSWORD` | Set PostgreSQL password (avoids interactive prompt)                                 |
| `NODE_PRIVATE_KEY`  | Set the node's private key (secret seed). Optional - auto-generated if not provided |

## Migrations

The container includes migration scripts that update database schemas, modify deprecated configuration parameters, and apply other necessary changes when upgrading. Migrations run automatically on startup when enabled with `--enable-auto-migrations`.

### How It Works

- Migration scripts are in `/migrations/` inside the container
- Scripts execute in alphanumeric order (e.g., `001_*.sh`, `002_*.sh`, ...)
- Each script runs only once — completed migrations are tracked in `/opt/stellar/migration_status`
- If a migration fails, the container stops immediately (fail-fast)
- Failed migrations will re-run on next startup
- Backups are created in `/opt/stellar/migration_backups/` before changes

### Enabling Migrations

```shell
$ docker run -d \
    -v "/path/to/data:/opt/stellar" \
    -p "31401:8000" \
    --name pi-node \
    pinetwork/pi-node-docker:organization_mainnet-v1.0-p23.0 --mainnet --enable-auto-migrations
```

### Running Migrations Manually

```shell
$ docker exec -it pi-node /migrations/migration_runner.sh
```

Or a specific migration:

```shell
$ docker exec -it pi-node /migrations/001_update_validator3.sh
```

Step-by-step documentation is in [migrations/docs](migrations/docs).

## Ports

| Port  | Service      | Description              |
|-------|--------------|--------------------------|
| 5432  | PostgreSQL   | Database access port     |
| 8000  | Horizon      | Main HTTP port           |
| 6060  | Horizon      | Admin port               |
| 31402 | stellar-core | Peer node port           |
| 11626 | stellar-core | HTTP port (internal)     |
| 11726 | captive-core | Captive core HTTP port   |
| 1570  | webfsd       | Local history server     |

### Recommended Port Mappings

| Host Port | Container Port | Service              |
|-----------|----------------|----------------------|
| 31401     | 8000           | Horizon HTTP         |
| 31402     | 31402          | stellar-core peer    |
| 31403     | 1570           | Local history server |

### Security Considerations

- **PostgreSQL (5432):** Keep protected. Write access can corrupt your view of the network.
- **Horizon HTTP (8000):** Safe to expose publicly.
- **Horizon Admin (6060):** Expose only to trusted networks.
- **stellar-core HTTP (11626):** Expose only to trusted networks. Allows admin commands.
- **stellar-core Peer (31402):** Can be exposed publicly to improve connectivity.
- **Local history (1570):** Internal only.

## Accessing and Debugging

Access a running container:

```shell
$ docker exec -it pi-node bash
```

### Managing Services

Services are managed via [supervisord](http://supervisord.org/index.html):

```shell
$ supervisorctl
horizon                          RUNNING    pid 143, uptime 0:01:12
postgresql                       RUNNING    pid 126, uptime 0:01:13
stellar-core                     RUNNING    pid 125, uptime 0:01:13
supervisor>
```

Common commands:
```shell
supervisor> restart horizon
supervisor> stop stellar-core
supervisor> tail -f horizon     # live horizon logs
supervisor> tail -f stellar-core
```

### Viewing Logs

Logs are at `/var/log/supervisor/`. Use `supervisorctl tail` for live logs.

`/tmp/stellar-core.log` — stellar-core log file.

### Accessing Databases

Two PostgreSQL databases:

- **`core`** — stellar-core data
- **`horizon`** — Horizon data

Connect with **username:** `stellar`, **password:** the one you set during init (or `POSTGRES_PASSWORD`).

## Example Commands

**Initialize mainnet (interactive):**
```shell
$ docker run -it --rm \
    -v "/path/to/data:/opt/stellar" \
    -p "31401:8000" -p "31402:31402" -p "31403:1570" \
    --name pi-node \
    pinetwork/pi-node-docker:organization_mainnet-v1.0-p23.0 --mainnet
```

**Start testnet2 in background (after init):**
```shell
$ docker run -d \
    -v "/path/to/data:/opt/stellar" \
    -p "31401:8000" -p "31402:31402" -p "31403:1570" \
    -e POSTGRES_PASSWORD=your_password \
    --name pi-node \
    pinetwork/pi-node-docker:organization_mainnet-v1.0-p23.0 --testnet2
```

## Docker Compose

Recommended `docker-compose.yml`:

```yaml
name: pi-node

services:
  mainnet:
    image: pinetwork/pi-node-docker:organization_mainnet-v1.0-p23.0
    container_name: mainnet
    env_file:
      - ./.env
    volumes:
      - ./data/stellar:/opt/stellar
      - ./data/supervisor_logs:/var/log/supervisor
      - ./data/history:/history
    ports:
      - "31401:8000"
      - "31402:31402"
      - "31403:1570"
    command: ["--mainnet --enable-auto-migrations"]
```

Create `.env`:
```shell
POSTGRES_PASSWORD=your_secure_password
NODE_PRIVATE_KEY=your_node_private_key  # Optional - auto-generated if not provided
```

Start:
```shell
$ docker compose up -d mainnet
```

## Building

```shell
$ make build
```

Builds as `pinetwork/pi-node-docker:organization_mainnet-v1.0-p23.0`.

## Changes from Upstream (PiCoreTeam/pi-node-docker)

| Area | Upstream | This fork |
|------|----------|-----------|
| Ubuntu base | 20.04 | 24.04 |
| PostgreSQL | 12 | 16 |
| stellar-core | 19.9.0 | 23.* |
| horizon | 2.30.0 | 2.* |
| Networks | mainnet only | mainnet + testnet1 + testnet2 |
| apt key method | apt-key add (deprecated) | keyrings |
| STELLAR_CORE_DATABASE_URL | present in common config | removed (deprecated in horizon) |
| Captive core storage path | not in common config | added as default |

## Troubleshooting

If you encounter issues, open an issue in the repository.
