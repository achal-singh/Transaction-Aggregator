# Transaction Aggregator

A TypeScript application that fetches and processes Ethereum wallet transactions using the Alchemy SDK. The application retrieves both incoming and outgoing transactions, processes them in batches, and exports the data to CSV file(s).

## Features

- Fetches both incoming and outgoing transactions for any Ethereum address
- Handles pagination for large transaction histories
- Fetches the gas fee of every transaction by calling the getTransactionReceipt API, this operation is managed by the Receipt Queue and [**Receipt Worker**](./src/services/receiptWorker.ts).
- CSV files are generated by the CSV Queue and [**CSV Worker**](./src/services/csvWorker.ts).
- Comprehensive test coverage using Jest.

## Prerequisites

- Node.js (v18 or higher)
- npm (v6 or higher)
- An Alchemy API key (login to get one [Alchemy](https://auth.alchemy.com/))

## Setup

1. Clone the repository:

```bash
git clone https://github.com/achal-singh/Transaction-Aggregator.git
cd Transaction-Aggregator
```

2. Install dependencies:

```bash
npm install
```

3. Create a `.env` file in the root directory and add your Alchemy API key:

```bash
ALCHEMY_API_KEY=your_api_key_here
```

## Project Structure

```
├── src/
│   ├── config/         # Configuration settings
│   ├── services/       # Core business logic
│   ├── types/          # TypeScript type definitions
│   ├── utils/          # Utility functions
│   └── index.ts        # Entry point
├── src/__tests__/      # Test files
├── csv/               # Output directory for CSV files
├── jest.config.js     # Jest configuration
└── tsconfig.json      # TypeScript configuration
```

## Usage

To fetch transactions for an Ethereum address:

```bash
npm run start -- --address=0xYourEthereumAddress
```

The application will:

1. Fetch all transactions for the given address.
2. Process them in batches of 1000, sub-batches of 300.
3. Generate a CSV file once all the receipt jobs are completed.

## Operation Notes

1. Make sure a **`redis`** instance is running on your local machine/server.
2. Configure all the values given in [.env.sample](/.env.sample) file and copy paste them in a **`.env`** file.
3. After running the `npm run start -- --address=0xYourEthereumAddress` command, if redis and .env values are configured correctly then the logs should like the following two cases:
   1. [**Ideal Case**](./assets/ideal.png) - Workers complete jobs within a decent time (5-10s per job for BATCH_SIZE = 50).
   2. [**Receipt Errors**](./assets/receipt_error.png) - Workers experience receipt failure(s) from Alchemy side, which are overcome automatically by re-fetching.
4. CSV generation is started only **_after all the receipt-jobs have been completed_**.

## How It Works

### 1. Initialization

- The application starts by validating the provided Ethereum address
- Creates an instance of `TransactionService` which initializes:
  - The Alchemy SDK for blockchain interactions
  - A `QueueService` that sets up two separate queues:
    1. A Receipt Queue for processing transaction receipts
    2. A CSV Queue for handling file generation

### 2. Transaction Fetching

- The service fetches both incoming and outgoing transactions linked to the wallet address using Alchemy's `getAssetTransfers` function call
- Handles pagination automatically if there are more than 1000 transactions using the `pageKey` in the response
- Transactions are stored in memory in two batches: `incomingTxs` and `outgoingTxs`
- Once a batch of transactions is collected, they are added to the Receipt Queue for processing

### 3. Receipt Processing - using [**BullMQ**](https://docs.bullmq.io/)

- Multiple Receipt Workers (default: 3, **`MAX_RECEIPT_WORKERS`** in [**config**](./src/config/index.ts)) process transaction receipts in parallel
- Each worker:
  - Takes a batch of transactions (configured by **`BATCH_SIZE`** in [**config**](./src/config/index.ts))
  - Fetches transaction receipts from Alchemy
  - Calculates gas fees and formats transaction data
  - Has built-in retry mechanism for failed receipt fetches
- The [**QueueService**](./src/services/queueService.ts) tracks job completion and aggregates processed transactions.

### 4. CSV Generation

- Once all receipt-jobs are completed, the [**CSV Worker**](./src/services/csvWorker.ts) is triggered
- The CSV Worker:
  - Takes all processed transactions.
  - Generates a CSV file with complete transaction data.
  - Names the file as: `address_<address>_batch_<number>.csv`
- CSV generation is handled in a separate queue to:
  - Prevent blocking receipt processing.
  - Handle large datasets efficiently.
  - Provide independent error handling and retries.

### 5. Error Handling & Recovery

- Both queues implement:
  - Automatic retries with exponential backoff
  - Job deduplication
  - Graceful shutdown handling
- Failed jobs are preserved for debugging
- The system can recover from:
  - Redis connection issues
  - Alchemy API failures
  - File system errors

## Running Tests

```bash
# Run all tests
npm test

# Run tests in watch mode
npm run test:watch

# Generate test coverage report
npm run test:coverage
```

## Dependencies

- `alchemy-sdk`: For interacting with the Alchemy API
- `bullmq`: Redis based job/message queue/broker.
- `ethers`: For Ethereum-related utilities
- `@json2csv/plainjs`: For CSV generation
- `jest`: For testing
