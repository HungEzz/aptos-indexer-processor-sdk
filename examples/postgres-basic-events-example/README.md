# Example Postgres Events Processor

## Overview

1. **aptos-indexer-processors-sdk** provides libraries/utilities (gRPC txn stream, step-based pipeline builder, Postgres helpers, server/metrics).
2. **postgres-basic-events-example** is a sample app showing how to use those components to index events into Postgres.

## How to Run the Project

### 1) Get an Authorization Token
Follow the Aptos docs to obtain an API key: https://aptos.dev/build/indexer/txn-stream/aptos-hosted-txn-stream#authorization-via-api-key

### 2) Install Rust
Rustup installs both Rust and Cargo.

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

### 3) Install PostgreSQL and Create the Database

#### Linux (Ubuntu/Debian)

1) Install and start PostgreSQL:
```bash
sudo apt update
sudo apt install postgresql-16
sudo service postgresql start
```

2) Create database and user, and grant database privileges:
```bash
sudo -u postgres psql -c "CREATE DATABASE postchain WITH TEMPLATE = template0 LC_COLLATE = 'C.UTF-8' LC_CTYPE = 'C.UTF-8' ENCODING 'UTF8';" \
  -c "CREATE ROLE postchain LOGIN ENCRYPTED PASSWORD 'postchain';" \
  -c "GRANT ALL PRIVILEGES ON DATABASE postchain TO postchain;"
```

3) Grant schema privileges (so the app can create tables and run migrations):
```bash
sudo -u postgres psql -d postchain -c "GRANT ALL PRIVILEGES ON SCHEMA public TO postchain;"
sudo -u postgres psql -d postchain -c "GRANT CREATE ON SCHEMA public TO postchain;"
sudo -u postgres psql -d postchain -c "ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON TABLES TO postchain;"
sudo -u postgres psql -d postchain -c "ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON SEQUENCES TO postchain;"
```

> You can change the database name, username, and password as needed.

#### macOS (Homebrew)

1) Install and start PostgreSQL:
```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
brew install postgresql@16
brew services start postgresql@16
export PATH="$(brew --prefix postgresql@16)/bin:$PATH"
```

2) Create database and user, and grant database privileges:
```bash
psql postgres -c "CREATE DATABASE postchain WITH TEMPLATE = template0 LC_COLLATE = 'C.UTF-8' LC_CTYPE = 'C.UTF-8' ENCODING 'UTF8';" \
  -c "CREATE ROLE postchain LOGIN ENCRYPTED PASSWORD 'postchain';" \
  -c "GRANT ALL PRIVILEGES ON DATABASE postchain TO postchain;"
```

3) Grant schema privileges:
```bash
psql -d postchain -c "GRANT ALL PRIVILEGES ON SCHEMA public TO postchain;"
psql -d postchain -c "GRANT CREATE ON SCHEMA public TO postchain;"
psql -d postchain -c "ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON TABLES TO postchain;"
psql -d postchain -c "ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON SEQUENCES TO postchain;"
```

> You can change the database name, username, and password as needed.

### 4) Create config.yaml
In this example folder, create a `config.yaml` similar to:

```yaml
# This is a template yaml for the processor
health_check_port: 8085
server_config:
  transaction_stream_config:
    indexer_grpc_data_service_address: "https://grpc.mainnet.aptoslabs.com:443"
    auth_token: "AUTH_TOKEN"
    request_name_header: "events-processor"
    starting_version: 0
  postgres_config:
    connection_string: postgresql://postchain:postchain@localhost:5432/postchain
```

Update these fields:
- `auth_token`: your API key from the Developer Portal
- `connection_string`: your Postgres DSN, e.g. `postgresql://username:password@localhost:5432/database_name`

### 5) Optional config.yaml tweaks
- Start at a specific ledger version:
```yaml
starting_version: <Starting Version>
```
- Stop at an ending version:
```yaml
request_ending_version: <Ending Version>
```
- Use a different network:
```yaml
# Devnet
indexer_grpc_data_service_address: "https://grpc.devnet.aptoslabs.com:443"

# Testnet
indexer_grpc_data_service_address: "https://grpc.testnet.aptoslabs.com:443"

# Mainnet
indexer_grpc_data_service_address: "https://grpc.mainnet.aptoslabs.com:443"
```

### 6) Run the processor
```bash
cd examples/postgres-basic-events-example
cargo run --release -- -c config.yaml
```

You should see the processor start indexing Aptos blockchain events!

Example output:
```json
{"timestamp":"2024-08-15T01:06:35.169217Z","level":"INFO","message":"[Transaction Stream] Received transactions from GRPC.","stream_address":"https://grpc.testnet.aptoslabs.com/","connection_id":"5575cb8c-61fb-498f-aaae-868d1e8773ac","start_version":0,"end_version":4999,"start_txn_timestamp_iso":"1970-01-01T00:00:00.000000000Z","end_txn_timestamp_iso":"2022-09-09T01:49:02.023089000Z","num_of_transactions":5000,"size_in_bytes":5708539,"duration_in_secs":0.310734,"tps":16078,"bytes_per_sec":18371143.80788713,"filename":"/Users/reneetso/.cargo/git/checkouts/aptos-indexer-processor-sdk-2f3940a333c8389d/e1e1bdd/rust/transaction-stream/src/transaction_stream.rs","line_number":400,"threadName":"tokio-runtime-worker","threadId":"ThreadId(6)"}
{"timestamp":"2024-08-15T01:06:35.257756Z","level":"INFO","message":"Events version [0, 4999] stored successfully","filename":"src/processors/events/events_storer.rs","line_number":75,"threadName":"tokio-runtime-worker","threadId":"ThreadId(10)"}
{"timestamp":"2024-08-15T01:06:35.257801Z","level":"INFO","message":"Finished processing events from versions [0, 4999]","filename":"src/processors/events/events_processor.rs","line_number":90,"threadName":"tokio-runtime-worker","threadId":"ThreadId(17)"}
```

For more details, see the Quickstart: https://aptos.dev/build/indexer/indexer-sdk/quickstart
