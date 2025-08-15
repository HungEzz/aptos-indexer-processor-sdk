# Aptos Indexer SDK
Generally, an indexer processor follow this flow:

1. Receive a stream of Aptos transactions
2. Extract data from the transactions
3. Transform and merge the parsed data into a coherent, standardized schema
4. Store the transformed data into a database

The Aptos Indexer SDK works by modeling each processor as a graph of independent steps. Each of the steps in the flow above is written as a `Step` in the SDK, and the output of each `Step` is connected to the input of the next `Step` by a channel.

# How to use

To your `Cargo.toml` , add

```yaml
aptos-indexer-processor-sdk = { git = "https://github.com/aptos-labs/aptos-indexer-processor-sdk.git", rev = "{COMMIT_HASH}" }
aptos-indexer-processor-sdk-server-framework = { git = "https://github.com/aptos-labs/aptos-indexer-processor-sdk.git", rev = "{COMMIT_HASH}" }
```

# Get started

We’ve created a [Quickstart Guide to Aptos Indexer SDK](https://aptos.dev/build/indexer/indexer-sdk/quickstart) which gets you setup and running an events processor that indexes events on the Aptos blockchain. 

# Documentation
Full documentation can be found [here](https://aptos.dev/build/indexer/indexer-sdk/documentation)

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