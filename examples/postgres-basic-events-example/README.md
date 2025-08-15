# Example Postgres Events Processor

## Overview

1. aptos-indexer-processors-sdk provides libraries/utilities (gRPC txn stream, step-based pipeline builder, Postgres helpers, server/metrics).
2. postgres-basic-events-example is a sample app showing how to use those components to index events into Postgres.

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

3) Grant all schema privileges (so the app can create tables and run migrations):
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

3) Grant all schema privileges:
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
To code, read Rust Documentation: https://doc.rust-lang.org/stable/

# 🚀 Volume & Fee Calculation for 1 Day, 7 Days, and 30 Days

## 📋 Processing Sequence:

1. �� Transaction Stream → Real-time blockchain events
- You can log the Transaction Stream in code to know exactly transaction, event structure on terminal to filter value you want
2. ⏰ Time Filtering → Only process transactions within the last 1, 7, or 30 days
   - 🔍 Filter transactions using their timestamp
   - 🛠️ Tool: Convert timestamp to date: https://www.unixtimestamp.com/
3. 🎯 Event Detection → Identify swap events from DEX protocols
   - 🏗️ Each DEX has different event structures
   - 🔎 Find TYPE_SWAP_EVENT for each DEX on AptosScan
   - 🔑 Key Difference: TYPE_SWAP_EVENT varies by DEX

   📝 Examples:
   - �� CELLANA: "0x4bf51972879e3b95c4781a5cdcb9e1ee24ef483e7d22f2d903626f126df62bd1::liquidity_pool::SwapEvent"
   - �� THALA: "0x7730cd28ee1cdc9e999336cbc430f99e7c44397c0aa77516f6f23a78559bb5::pool::SwapEvent"
   
   💡 Use TYPE_SWAP_EVENT to determine which DEX the event belongs to

4. ⚙️ DEX Processing → Extract volumes and fees per pool per DEX
   - 🎯 Once you identify the DEX, filter the event to extract required values
   - �� Required Data: AMOUNT_IN, AMOUNT_OUT, FROM_TOKEN, TO_TOKEN, FEE, and DECIMAL of each coin
   - 🔍 Find decimal places for each coin on AptosScan

   💰 Example: Calculating Volume & Fee for APT/USDT Pair on CELLANA DEX
   
   1. �� Filter Event Data: Extract AMOUNT_IN, AMOUNT_OUT, FROM_TOKEN, TO_TOKEN, and FEE from the CELLANA event
   
   2. 🪙 Determine Coin Types: Identify which coin is APT and which is USDT using FROM_TOKEN and TO_TOKEN
      - FROM_TOKEN and TO_TOKEN have COIN_TYPE values like:
        - �� APT_COIN_TYPE: "0x1::aptos_coin::AptosCoin"
        - �� USDT_COIN_TYPE: "0x357b0b74bc833e95a115ad22604854d6b0fca151cecd94111770e5d6ffc9dc2b"
      - 🔍 Search on AptosScan to find the coin type for that DEX
   
   3. 🧮 Calculate Volume: Determine transaction direction:
      
      �� Case 1: User swaps APT for USDT
      - FROM_TOKEN = APT_COIN_TYPE
      - TO_TOKEN = USDT_COIN_TYPE
      - AMOUNT_IN = 3 APT
      - AMOUNT_OUT = 12 USDT
      - �� User swapped 3 APT for 12 USDT
      
      �� Case 2: User swaps USDT for APT
      - FROM_TOKEN = USDT_COIN_TYPE
      - TO_TOKEN = APT_COIN_TYPE
      - AMOUNT_IN = 12 USDT
      - AMOUNT_OUT = 3 APT
      - �� User swapped 12 USDT for 3 APT

      �� Volume Calculation:
      - �� APT Volume: Add APT amounts from all events
      - 💚 USDT Volume: Add USDT amounts from all events
      
      ⚠️ Important: AMOUNT_IN includes the transaction fee, so calculate AMOUNT_IN - FEE to get the exact swap amount
      
      �� Fee Calculation: 
      - 🔍 Filter swap_fee_bps for each event
      - 📈 Totalize fees for each DEX to calculate 1, 7, and 30-day fees
      - ⚠️ Note: Some DEXs don't have swap_fee_bps - check carefully

   �� Note: Each DEX has different structures. The example above shows CELLANA's structure. Other DEXs like THALA and SUSHI have different structures - research the TYPE_EVENT for each DEX on AptosScan.

5. �� Volume Aggregation → Sum all volumes per DEX
6. 💾 Database Accumulation → Add to existing rolling totals (24h, 7d, or 30d)
7. 🔗 Cross-DEX Aggregation → Create "Aptos protocol" total from all DEXs
8. 🎯 Final Storage → Persist all data with automatic cleanup

## �� Key Implementation Points:

- ⏰ Time Windows: Use filtering functions (is_within_24h(), is_within_7d(), is_within_30d())
- 🎯 DEX Detection: Match event.type_str against DEX-specific constants
- �� Volume Calculation: Always account for fees when calculating net swap amounts
- �� Data Normalization: Convert raw amounts using proper decimal places for each token type
- 🔄 Rolling Aggregation: Maintain running totals that automatically reset after the time period expires

## 📈 Buy/Sell Volume Calculation

After calculating volume and fees, determine buy/sell volumes for each coin on each DEX:

💡 Method: Determine transaction direction - AMOUNT_IN is sell volume, AMOUNT_OUT is buy volume

�� Example: Calculate Buy/Sell Volume for APT Coin on CELLANA DEX

For a batch of events, filter events that traded APT and determine transaction direction:

- �� EVENT 1: AMOUNT_IN is APT = 2, AMOUNT_OUT is USDT = 6
  - 👤 User sold 2 APT, bought 6 USDT
- �� EVENT 2: AMOUNT_IN is SUI = 3, AMOUNT_OUT is APT = 5
  - 👤 User sold 3 SUI, bought 5 APT
- �� EVENT 3: AMOUNT_IN is APT = 1, AMOUNT_OUT is USDC = 5.99
  - 👤 User sold 1 APT, bought 5.99 USDC

📊 Result:
- �� APT Sell Volume: 2 + 1 = 3 APT (sold)
- 🟡 APT Buy Volume: 5 APT (bought)

💡 Note: This approach tracks the actual trading direction - what users are selling vs. buying for each token.

## 📊 Volume Data for Chart

## From volume calculation for 24h, 7 days, 30 days, create time-based buckets for chart visualization as how many points you like:
- For example: 
### 🕐 24-Hour Volume Buckets 
- Chart Points: 12 points
- Time Division: 24 hours ÷ 12 = 2 hours per bucket
- Bucket Index: 0-11 (representing 2-hour intervals)

Example 24h Buckets:
- 🕐 Bucket 0: 00:00-02:00 
- 🕑 Bucket 1: 02:00-04:00 
- 🕒 Bucket 2: 04:00-06:00 
- ...
- 🕚 Bucket 11: 22:00-24:00 

### 📅 7-Day Volume Buckets
- Chart Points: 7 points
- Time Division: 7 days ÷ 7 = 1 day per bucket
- Bucket Index: 0-6 

Example 7d Buckets:
- 📅 Bucket 0: Day 1 
- 📅 Bucket 1: Day 2 
- 📅 Bucket 2: Day 3 
- 📅 Bucket 6: Day 7 

### 🗓️ 30-Day Volume Buckets
- Chart Points: 30 points
- Time Division: 30 days ÷ 30 = 1 day per bucket
- Bucket Index: 0-29 


Example 30d Buckets:
- 📅 Bucket 0: Day 1
- 📅 Bucket 1: Day 2
- 📅 Bucket 2: Day 3
- ...
- 📅 Bucket 29: Day 30

### 🔄 Bucket Processing Flow:

1. 📥 Input: Swap events with timestamps
2. ⏰ Time Window Check: Filter events within the time period (24h/7d/30d)
3. 🧮 Bucket Index Calculation: Determine which bucket each event belongs to
4. 📊 Volume Aggregation: Sum volumes for each bucket
5. 💾 Array Creation: Create volume arrays with each bucket 

### 🎯 Key Features:

- 🔄 Sliding Window: Automatically adjusts time windows as time progresses
- Real-time Updates: Buckets update in real-time as new transactions 
- 🌏 Timezone Support: All calculations use GMT+7 timezone 
- 🗑️ Auto Cleanup: Old buckets automatically removed to maintain performance

# 💰 Average Price Calculation on Aptos Protocol

## 📚 Understanding VWAP (Volume Weighted Average Price)

Reference: https://medium.com/hackernoon/you-do-not-know-how-coinmarketcap-prices-coins-42c8a4063bb3

### 💡 Key Concepts:
- 🏦 No Fiat Currency: Aptos doesn't have direct fiat, so we use USDT as the base currency
- ⚡ APT as Base: APT has the highest volume, so we use APT to determine other coin prices
- 🔄 Real-time Pricing: Use the newest events to get current prices
- 📊 Oracle Enhancement: For 100% accuracy, multiply USDT result with oracle USDT price

### 🛠️ Implementation Steps:

🔄 Volume Calculation → Calculate total volumes on all DEX and volumes per DEX
💱 Price Extraction → Get latest pair asset prices you want from newest events
- For example to get the prices APT/USDT you filter each APT/USDT event
- Determine transaction direction and get APT amount / USDT amount so you can get APT/USDT price
🧮 VWAP Calculation → Apply volume-weighted formula
* Remember check the condition of formula
🔗 Cross-Coin Pricing → Use APT/USDT price to determine price other coins


## 📊 Calculate the percentage volume change for last 24h

### 🎯 Overview:

### 📋 Processing Steps:

#### Step 1: Data Collection & Time Filtering
- Collect Events: Gather all swap events from the last 48 hours
- Protocol Coverage: Include events from all DEX 

#### Step 2: Period Separation
- Current Period: Last 24 hours (0-24h from now)
- Previous Period: Previous 24 hours (24-48h from now)
- Boundary: Use timestamp calculations to separate periods

#### Step 3: Volume Calculation per Period
- Current Volume: Sum all volumes for each coin in the current 24h period
- Previous Volume: Sum all volumes for each coin in the previous 24h period
- Coin Aggregation: Calculate volumes separately for each token 

#### Step 4: Percentage Change Calculation
- Formula Application: Use the standard percentage change formula: 
((Current Volume - Previous Volume) / Previous Volume) × 100
- Result Range: Percentage can be positive (increase), negative (decrease), or zero (no change)
- Remember remove old data to maintain system performance

### �� Processing Flow:

1. �� Event Collection → Gather 48h of swap events
2. ⏰ Time Filtering → Separate into current vs previous periods
3. �� Volume Calculation → Sum volumes for each period
4. 📊 Percentage Calculation → Apply change formula
5. 💾 Data Storage → Save to database 
6. 🔄 Real-Time Updates → Process new events continuously

### 📈 Example:

#### Volume Increase:
- Current: 1,000,000 USDT
- Previous: 800,000 USDT
- Result: +25% increase

#### Volume Decrease:
- Current: 600,000 USDT
- Previous: 800,000 USDT
- Result: -25% decrease

## 📊 Calculate the percentage price change for last 24h
- Based on the average price of a coin you calculated above, determine the average price of that coin for the last 24 hours.

- Apply the same formula used for calculating the percentage volume change over the last 24 hours:
((Current price - Previous price) / Previous price) × 100

For example:
- The current price of APT is $4.20.

- The price of APT 24 hours ago was $4.00.

The percentage price change over the last 24 hours is:
((4.20 - 4.0) / 4.0) × 100 = + 5% increase
