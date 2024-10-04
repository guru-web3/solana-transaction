# README for Displaying Activity List

This README provides a detailed explanation of how to fetch, parse, and display activities related to NFTs and SPL tokens on the Solana blockchain. It includes example code snippets and references to additional resources for further reading.

## Displaying Activity List

### Steps for Fetching Activities

#### Fetch Transaction History

Use Solana's RPC method `getConfirmedSignaturesForAddress2` to get a list of confirmed transaction signatures.

```typescript
import { Connection, PublicKey } from "@solana/web3.js";

async function fetchActivities(walletAddress) {
    const connection = new Connection('https://api.mainnet-beta.solana.com');
    const activities = await connection.getConfirmedSignaturesForAddress2(
        new PublicKey(walletAddress)
    );
    return activities.map(signature => {
        // Additional parsing logic here
        return signature;
    });
}

const walletAddress = "Your_Solana_Wallet_Address";
const activityLog = await fetchActivities(walletAddress);
console.log(activityLog);
```

## Parse Transactions
To convert raw signatures into a user-readable format, we need to parse each transaction and extract relevant details.

```typescript
import { Connection, PublicKey, TransactionSignature } from "@solana/web3.js";

async function fetchAndParseActivities(walletAddress) {
    const connection = new Connection('https://api.mainnet-beta.solana.com');
    const signatures = await connection.getConfirmedSignaturesForAddress2(
        new PublicKey(walletAddress)
    );

    const activities = await Promise.all(signatures.map(async (signature) => {
        const transaction = await connection.getConfirmedTransaction(signature.signature);
        return parseTransaction(transaction);
    }));

    return activities;
}

function parseTransaction(transaction) {
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
const activityLog = await fetchAndParseActivities(walletAddress);
console.log(activityLog);
```

## Display Activities
Map the parsed transactions to an activity log for the user to visualize their historical interaction with SPL tokens and NFTs.

``` typescript
const signatureInfo = await connection.getSignaturesForAddress(walletAddress);
const parsedActivities = activities.map(parseTransaction);
let incomingBlockchainTransactions: ParsedTransactionWithMeta[] = [];

if (signatureInfo.length > 0) {
    incomingBlockchainTransactions = await connection.getParsedTransactions(
    signatureInfo.map((s) => s.signature),
    { maxSupportedTransactionVersion: 0 }
    );
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
      blockExplorerUrl: getSolanaTransactionLink(blockExplorerUrl, info.signature, chainId),
      chainId,
      network: CHAIN_ID_NETWORK_MAP[chainId],
      rawDate: new Date(info.blockTime * 1000).toISOString(),
      action: ACTIVITY_ACTION_UNKNOWN,
      type: "unknown",
      decimal: 9,
    };

    if (!tx?.meta) return finalObject;

    let interestedTransactionInstructionidx = -1;
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
        interestedTransactionInstructionidx = transferIdx;
      } else {
        const burnIdx = tx.transaction.message.instructions.findIndex((inst) => {
          return ["burn", "burnChecked"].includes((inst as unknown as ParsedInstruction).parsed?.type);
        });
        interestedTransactionInstructionidx = burnIdx;
      }
    }

    const interestedTransactionType = ["transfer", "transferChecked", "burn", "burnChecked"];

    if (tx.transaction.message.instructions.length === 1 || interestedTransactionInstructionidx >= 0) {
      if (tx.transaction.message.instructions.length === 1) interestedTransactionInstructionidx = 0;
      const inst: ParsedInstruction = tx.transaction.message.instructions[interestedTransactionInstructionidx] as unknown as ParsedInstruction;
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
          finalObject.action = finalObject.send ? ACTIVITY_ACTION_SEND : ACTIVITY_ACTION_RECEIVE;
          finalObject.decimal = decimals;
          finalObject.totalAmountString = cryptoAmountToUiAmount(amount, decimals);
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
          finalObject.action = finalObject.send ? ACTIVITY_ACTION_SEND : ACTIVITY_ACTION_RECEIVE;
          finalObject.decimal = 9;
          finalObject.totalAmountString = lamportToSol(inst.parsed.info.lamports);
        }
      }
    }
    return finalObject;
  });
  return finalTxs;
};

formatTransactionActivity({
    transactions: incomingBlockchainTransactions,
    signaturesInfo: filteredSignaturesInfo,
    chainId,
    blockExplorerUrl,
    selectedAddress: this.state.selectedAddress,
});
```