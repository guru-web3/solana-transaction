# README for Displaying Activity List

This README provides a detailed explanation of how to fetch, parse, and display activities related to NFTs and SPL tokens on the Solana blockchain along with submitted Backend transaction submitted to blockchain. It includes example code snippets and references to additional resources for further reading.

## Displaying Activity List

### Steps for Fetching Activities

### Fetch Transaction History

Use Solana's RPC method [`getsignaturesforaddress`](https://solana.com/docs/rpc/http/getsignaturesforaddress) to get a list of confirmed transaction signatures.
#### 1. Import Required Modules:
- Import the necessary classes from the [Solana Web3.js library](https://solana-labs.github.io/solana-web3.js/).
#### 2. Create a Connection:
- Establish a connection to the Solana mainnet using the class `Connection` [new Connection()](https://solana-labs.github.io/solana-web3.js/classes/Connection.html).
#### 3. Fetch Signatures:
- Use the [getSignaturesForAddress](https://solana.com/docs/rpc/http/getsignaturesforaddress) method to fetch the transaction signatures for the specified wallet address.
#### 4. Return Signatures:
- Map the fetched signatures to a format suitable for further processing.

```typescript
import { Connection, PublicKey } from "@solana/web3.js";

async function fetchActivities(walletAddress: string) {
    const connection = new Connection('https://api.mainnet-beta.solana.com');
    const signatures = await connection.getSignaturesForAddress(
        new PublicKey(walletAddress)
    );
    return signatures.map(signature => {
        // Additional parsing logic here
        return signature;
    });
}

const walletAddress = "Your_Solana_Wallet_Address";
fetchActivities(walletAddress).then(activityLog => console.log(activityLog));
```


### Parse Transactions
To convert raw signatures into a user-readable format, we need to parse each transaction and extract relevant details.

#### 1. Fetch and Parse Transactions:
- For each fetched signature, retrieve the corresponding transaction using the [getParsedTransaction](https://www.quicknode.com/docs/solana/getParsedTransaction) method.
#### 2. Extract Transaction Details:
- Extract relevant details such as fee, balances, account keys, and instructions from each transaction.
#### 3. Return Parsed Transactions:
- Return the parsed transactions in a structured format.

```typescript
import { Connection, PublicKey, ParsedTransactionWithMeta } from "@solana/web3.js";

async function fetchAndParseActivities(walletAddress: string) {
    const connection = new Connection('https://api.mainnet-beta.solana.com');
    const signatures = await connection.getSignaturesForAddress(
        new PublicKey(walletAddress)
    );

    const activities = await Promise.all(signatures.map(async (signature) => {
        const transaction = await connection.getParsedTransaction(signature.signature);
        return parseTransaction(transaction);
    }));

    return activities;
}

function parseTransaction(transaction: ParsedTransactionWithMeta) {
    const { meta, transaction: { message } } = transaction;
    const { postBalances, preBalances } = meta;
    const { accountKeys, instructions } = message;

    return {
        signature: transaction.transaction.signatures[0],
        fee: meta.fee,
        postBalances,
        preBalances,
        accountKeys: accountKeys.map(key => key.toBase58()),
        instructions: instructions.map(instruction => ({
            programId: instruction.programId.toBase58(),
            data: instruction.data,
            keys: instruction.keys.map(key => key.pubkey.toBase58())
        }))
    };
}

const walletAddress = "Your_Solana_Wallet_Address";
fetchAndParseActivities(walletAddress).then(activityLog => console.log(activityLog));
```

### Display Activities
To display the activities, we need to map the parsed transactions to an activity log format that users can easily understand.
#### 1. Fetch Signatures and Transactions:
- Fetch the transaction signatures and corresponding parsed transactions for the specified wallet address.
#### 2. Format Transactions:
- Format the fetched transactions into a user-readable activity log.
#### 3. Display Activities:
- Log the formatted activities to the console or display them in the UI.

### `formatTransactionToActivity` function
#### 1. The function `formatTransactionToActivity` takes an object as a parameter. This object contains the following properties:
- `transactions`: An array of parsed transactions.
- `signaturesInfo`: An array of confirmed signature information.
- `chainId`: The ID of the blockchain network.
- `blockExplorerUrl`: The URL of the block explorer.
- `selectedAddress`: The wallet address for which activities are being fetched.
#### 2: Initialize the Activity Log
- Initialize an array `finalTxs` to store the formatted activities. Loop through each signature information and corresponding transaction.
#### 3: Create the Base Activity Object
- For each transaction, create a base activity object with common properties such as slot, status, updatedAt, signature, and blockExplorerUrl.
#### 4: Identify Relevant Instructions
Identify the relevant instructions within the transaction. This involves checking for specific instruction types such as [create](https://solana.com/developers/cookbook/tokens/create-token-account), [transfer](https://solana.com/developers/cookbook/tokens/transfer-tokens), and [burn](https://solana.com/developers/cookbook/tokens/burn-tokens).
#### 5: Parse Relevant Instructions
Parse the relevant instructions to extract details such as source, destination, amount, and type of transaction. Update the activity object with these details.

``` typescript
import { Connection, PublicKey, ParsedTransactionWithMeta, ConfirmedSignatureInfo } from "@solana/web3.js";

async function displayActivities(walletAddress: string) {
    const transactions = fetchAndParseActivities(walletAddress)

    const formattedActivities = formatTransactionToActivity({
        transactions,
        signaturesInfo: signatures,
        chainId: 'mainnet-beta',
        blockExplorerUrl: 'https://explorer.solana.com',
        selectedAddress: walletAddress,
    });

    console.log(formattedActivities);
}

const formatTransactionToActivity = (params: {
    transactions: ParsedTransactionWithMeta[];
    signaturesInfo: ConfirmedSignatureInfo[];
    chainId: string;
    blockExplorerUrl: string;
    selectedAddress: string;
}) => {
    const { transactions, signaturesInfo, chainId, blockExplorerUrl, selectedAddress } = params;
    const finalTxs = signaturesInfo.map((info, index) => {
        const tx = transactions[index];
        const finalObject: SolanaTransactionActivity = {
            slot: info.slot.toString(),
            status: tx?.meta?.err ? TransactionStatus.failed : TransactionStatus.confirmed,
            updatedAt: info.blockTime * 1000,
            signature: info.signature,
            txReceipt: info.signature,
            blockExplorerUrl: `${blockExplorerUrl}/tx/${info.signature}?cluster=${chainId}`,
            chainId,
            network: chainId,
            rawDate: new Date(info.blockTime * 1000).toISOString(),
            action: 'unknown',
            type: "unknown",
            decimal: 9,
        };

        if (!tx?.meta) return finalObject;

        let interestedTransactionInstructionIdx = -1;
        const instructionLength = tx.transaction.message.instructions.length;

        if (instructionLength > 1 && instructionLength <= 3) {
            const createInstructionIdx = tx.transaction.message.instructions.findIndex((inst) => {
                if (inst.programId.equals(ASSOCIATED_TOKEN_PROGRAM_ID)) {
                    return (inst as unknown as ParsedInstruction).parsed?.type === "create";
                }
                return false;
            });
            if (createInstructionIdx >= 0) {
                const transferIdx = tx.transaction.message.instructions.findIndex((inst) => {
                    return ["transfer", "transferChecked"].includes((inst as unknown as ParsedInstruction).parsed?.type);
                });
                interestedTransactionInstructionIdx = transferIdx;
            } else {
                const burnIdx = tx.transaction.message.instructions.findIndex((inst) => {
                    return ["burn", "burnChecked"].includes((inst as unknown as ParsedInstruction).parsed?.type);
                });
                interestedTransactionInstructionIdx = burnIdx;
            }
        }

        const interestedTransactionType = ["transfer", "transferChecked", "burn", "burnChecked"];

        if (tx.transaction.message.instructions.length === 1 || interestedTransactionInstructionIdx >= 0) {
            if (tx.transaction.message.instructions.length === 1) interestedTransactionInstructionIdx = 0;
            const inst: ParsedInstruction = tx.transaction.message.instructions[interestedTransactionInstructionIdx] as unknown as ParsedInstruction;
            if (inst.parsed && interestedTransactionType.includes(inst.parsed.type)) {
                if (inst.program === "spl-token") {
                    const source = inst.parsed.info.authority;
                    if (tx.meta.postTokenBalances?.length <= 1) {
                        finalObject.from = source;
                        finalObject.to = source;
                    } else {
                        finalObject.from = source;
                        finalObject.to = tx.meta.postTokenBalances[0].owner === source ? tx.meta.postTokenBalances[1].owner : tx.meta.postTokenBalances[0].owner;
                    }

                    const { mint } = ["burn", "burnChecked"].includes(inst.parsed.type) ? inst.parsed.info : tx.meta.postTokenBalances[0];
                    const amount = ["burnChecked", "transferChecked"].includes(inst.parsed.type)
                        ? inst.parsed.info.tokenAmount.amount
                        : inst.parsed.info.amount;
                    const decimals = ["burnChecked", "transferChecked"].includes(inst.parsed.type)
                        ? inst.parsed.info.tokenAmount.decimals
                        : inst.parsed.info.decimals;
                    finalObject.cryptoAmount = amount;
                    finalObject.cryptoCurrency = "-";
                    finalObject.fee = tx.meta.fee;
                    finalObject.type = inst.parsed.type;
                    finalObject.send = finalObject.from === selectedAddress;
                    finalObject.action = finalObject.send ? 'send' : 'receive';
                    finalObject.decimal = decimals;
                    finalObject.totalAmountString = (amount / Math.pow(10, decimals)).toString();
                    finalObject.logoURI = "";
                    finalObject.mintAddress = mint;
                } else if (inst.program === "system") {
                    finalObject.from = inst.parsed.info.source;
                    finalObject.to = inst.parsed.info.destination;
                    finalObject.cryptoAmount = inst.parsed.info.lamports;
                    finalObject.cryptoCurrency = "SOL";
                    finalObject.fee = tx.meta.fee;
                    finalObject.type = inst.parsed.type;
                    finalObject.send = inst.parsed.info.source === selectedAddress;
                    finalObject.action = finalObject.send ? 'send' : 'receive';
                    finalObject.decimal = 9;
                    finalObject.totalAmountString = (inst.parsed.info.lamports / Math.pow(10, 9)).toString();
                }
            }
        }
        return finalObject;
    });
    return finalTxs;
};

const walletAddress = "Your_Solana_Wallet_Address";
displayActivities(walletAddress);
```

## Handling Incoming Backend Transactions
### Fetch Incoming Backend Transactions
This method fetches incoming backend transactions and stores them in the state.
#### 1. Fetch Transactions from Backend:
- Use the `getBackendTransactions` method to fetch transactions from the backend.
#### 2. Format Transactions:
- Format the fetched transactions using the `formatBackendTxToActivity` helper function.
#### 3. Update State:
- Update the state with the formatted transactions.

``` typescript
async fetchIncomingBackendTransaction(address: string) {
  try {
    const data = await this.getWalletOrders<FetchedTransaction>(address);
    if (data.length > 0) {
      const fmtData = data.map((item) => formatBackendTxToActivity(item, address));
      this.updateState({ incomingBackendTransactions: fmtData }, address);
    } else {
      this.updateState({ incomingBackendTransactions: [] }, address);
    }
  } catch (error) {
    log.error("unable to fetch wallet orders", error);
  }
}
```

## Merge Transactions from Chain and Backend
This method updates the display activities with the latest data from the blockchain and merges them with the transactions stored in the backend.

#### 1. Fetch Latest Signatures:
- Fetch the latest transaction signatures from the blockchain.
#### 2. Filter Confirmed Transactions:
- Filter out transactions that are already confirmed.
#### 3. Fetch and Parse Transactions:
- Fetch and parse the transactions corresponding to the filtered signatures.
#### 4. Format Transactions:
- Format the parsed transactions into a user-readable activity log.
#### 5. Merge Transactions:
- Merge the formatted transactions with the transactions stored in the backend.
#### 6. Update State:
- Update the state with the merged transactions.

```typescript
async updateDisplayActivities(newActivities?: { [keyof: string]: SolanaTransactionActivity }): Promise<SolanaTransactionActivity[]> {
  const address = this.state.selectedAddress;
  const { chainId, blockExplorerUrl } = this.getProviderConfig();
  const connection = this.getConnection();

  // Get latest signature from blockchain for main Account
  const signatureInfo = await connection.getSignaturesForAddress(new PublicKey(address), { limit: this.config.TX_LIMIT || 40 });

  // Filter out local's signature that is confirmed
  const displayActivities: { [keyof: string]: SolanaTransactionActivity } = newActivities || this.getAddressState(address)?.displayActivities || {};
  const filteredSignaturesInfo = signatureInfo.filter((info) => {
    const activity = displayActivities[info.signature];
    if (activity) {
      return activity.status !== TransactionStatus.confirmed;
    }
    return true;
  });

  // get parsed confirmed transactions and format it to local activity display
  let incomingBlockchainTransactions: ParsedTransactionWithMeta[] = [];
  if (filteredSignaturesInfo.length > 0) {
    incomingBlockchainTransactions = await connection.parseTransaction(
      filteredSignaturesInfo.map((s) => s.signature),
      { maxSupportedTransactionVersion: 0 }
    );
  }

  const incomingBlockchainActivities = formatTransactionToActivity({
    transactions: incomingBlockchainTransactions,
    signaturesInfo: filteredSignaturesInfo,
    chainId,
    blockExplorerUrl,
    selectedAddress: this.state.selectedAddress,
  });

  // patch backend and merge new activities with local's
  incomingBlockchainActivities.forEach((item) => {
    const activity = displayActivities[item.signature];
    // new incoming transaction from blockchain
    if (!activity) {
      displayActivities[item.signature] = item;
    } else if (item.status !== activity.status) {
      activity.status = item.status;
      if (activity.id) {
        this.patchPastTx({ id: activity.id.toString(), status: activity.status, updated_at: new Date().toISOString() }, address);
        this.updateIncomingTransaction(activity.status, activity.id);
      }
    }
  });

  this.updateState({ displayActivities }, address);
  return Object.values(displayActivities);
}
```