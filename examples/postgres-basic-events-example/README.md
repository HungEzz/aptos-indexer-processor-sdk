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
 
# �� Volume Calculation for 1 Day, 7 Days, and 30 Days

### Processing Sequence:

1. Transaction Stream → Real-time blockchain events
2. Time Filtering → Only process transactions within the last 1, 7, or 30 days
   - To filter transaction within the last 1, 7, or 30 days we depend on the timestamp of the transaction
   This is a tool to covert timestamp to Date : https://www.unixtimestamp.com/
3. Event Detection → Identify swap events from 5 DEX 
   - Each DEX may have different event structures
   - You can find the TYPE_SWAP_EVENT of each DEX on aptosscan
   - Key Difference: TYPE_SWAP_EVENT varies by DEX

   Examples:
   - CELLANA: "0x4bf51972879e3b95c4781a5cdcb9e1ee24ef483e7d22f2d903626f126df62bd1::liquidity_pool::SwapEvent"
   - THALA: "0x7730cd28ee1cdc9e999336cbc430f99e7c44397c0aa77516f6f23a78559bb5::pool::SwapEvent"
   
   Depend on the TYPE_SWAP_EVENT to determine which DEX the event belongs to

4. DEX Processing → Extract volumes per pool per DEX
   - Once you determine the DEX of an event, filter that event to identify the values needed for volume calculation
   - Required Data: AMOUNT_IN, AMOUNT_OUT, FROM_TOKEN, TO_TOKEN, FEE, and DECIMAL of each event
   - You can find the decimal places of each coin for that DEX on AptosScan

   Example: Calculating Volume for APT/USDT Pair on CELLANA DEX
   
   1. Filter Event Data: Extract AMOUNT_IN, AMOUNT_OUT, FROM_TOKEN, TO_TOKEN, and FEE from the CELLANA event
   
   2. Determine Coin Types: Identify which coin is APT and which is USDT using FROM_TOKEN and TO_TOKEN from the event
      - FROM_TOKEN and TO_TOKEN have COIN_TYPE values like:
        - APT_COIN_TYPE: "0x1::aptos_coin::AptosCoin"
        - USDT_COIN_TYPE: "0x357b0b74bc833e95a115ad22604854d6b0fca151cecd94111770e5d6ffc9dc2b"
      - You must know the coin type of the coin you want to calculate - search on AptosScan to find the coin type for that DEX
   
   3. Calculate Volume: When you know the transaction direction:
      - AMOUNT_IN = amount of FROM_TOKEN
      - AMOUNT_OUT = amount of TO_TOKEN
      
      Example: Event has transaction direction:
      - FROM_TOKEN = APT_COIN_TYPE
      - TO_TOKEN = USDT_COIN_TYPE
      - AMOUNT_IN = 3
      - AMOUNT_OUT = 12
      
      In this event, the user swapped 3 APT for 12 USDT
      
      - APT Volume: Add the APT amount from each event
      - USDT Volume: Calculate USDT volume similarly
      
      Important: AMOUNT_IN of the event includes the fee to execute the transaction, so we need to calculate AMOUNT_IN - FEE to get the exact amount of coin the user actually swapped

   ⚠️ Note: Each DEX has different structures. The example above shows CELLANA's structure. Other DEXs like THALA and SUSHI will have different structures, so you need to research the TYPE_EVENT for each DEX. You can find the DEXs you need on AptosScan and examine their event structures

5. Volume Aggregation → Sum all pools per DEX
6. Database Accumulation → Add to existing rolling totals (24h, 7d, or 30d)
7. Cross-DEX Aggregation → Create "Aptos protocol" total from all DEXs
8. Final Storage → Persist all data with automatic cleanup

### Key Implementation Points:

- Time Windows: Use different filtering functions (is_within_24h(), is_within_7d(), is_within_30d())
- DEX Detection: Match event.type_str against DEX-specific constants
- Volume Calculation: Always account for fees when calculating net swap amounts
- Data Normalization: Convert raw amounts using proper decimal places for each token type
- Rolling Aggregation: Maintain running totals that automatically reset after the time period expires