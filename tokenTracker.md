# NFT and Token Tracker for Solana

This documentation provides an overview of how our application tracks NFTs and SPL tokens on the Solana blockchain, along with how to display a list of activities related to these assets. The solution uses key Solana infrastructure such as QuickNode and Solana Registry while focusing on token and NFT ownership tracking.

## Objective

1. **Track NFTs**: Identify and filter verified NFTs from an owner's account using the Solana blockchain.
2. **Track SPL Tokens**: Manage and display SPL tokens while ensuring that custom tokens and unverified tokens are handled properly.


## 1. Tracking NFTs

### High-Level Process:

#### Fetch Token Account/Owner Information:

- Use the function `getParsedTokenAccountsByOwner` to get all tokens owned by a specific user.

#### Metadata Mapping:

- Filter out the tokens that correspond to NFTs by accessing the metadata of each token.

#### Verification Using QuickNode API:

- Query the QuickNode API to identify verified and unverified NFTs.

#### Filter Unverified NFTs:

- Ensure that the unverified NFTs are excluded from the final display list, leaving only the verified ones.

### Example Code:

# Token Fetching and Management
1. Token Fetching Workflow:
2. Token List via Registry:
3. Fetch a token list using the Solana SPL token registry (npm @solana/spl-token-registry).

### Unknown Tokens:
Handle tokens that are not in the registry but are present in the user's account by categorizing them as custom tokens.

### Custom Token Management:
Allow users to add custom tokens which are fetched and validated separately.

### Example Code:

```typescript
import { Connection, PublicKey } from "@solana/web3.js";
import { TOKEN_PROGRAM_ID } from "@solana/spl-token";

// Function to fetch and filter NFTs
async function fetchAndFilterNFTs(walletAddress: string) {
    const connection = new Connection('https://api.mainnet-beta.solana.com');
    const tokenAccounts = await connection.getParsedTokenAccountsByOwner(
        new PublicKey(walletAddress),
        { programId: TOKEN_PROGRAM_ID }
    );

    const nftTokens = tokenAccounts.value.filter(account => {
        const tokenAmount = account.account.data.parsed.info.tokenAmount;
        return tokenAmount.decimals === 0 && tokenAmount.uiAmount === 1;
    });

    const verifiedNfts = await quickNodeApi.getVerifiedNfts(nftTokens);
    return verifiedNfts;
}

const walletAddress = "Your_Solana_Wallet_Address";
const verifiedNfts = await fetchAndFilterNFTs(walletAddress);
console.log(verifiedNfts);
```

### Example Code:
```typescript
import { TokenListProvider } from '@solana/spl-token-registry';

// Function to fetch known and unknown tokens
async function fetchTokens(walletAddress: string) {
    const connection = new Connection('https://api.mainnet-beta.solana.com');
    const tokenListProvider = new TokenListProvider();
    const tokenList = await tokenListProvider.resolve();
    const knownTokens = tokenList.filterByChainId(101); // Fetch Solana Mainnet tokens

    const tokenAccounts = await connection.getParsedTokenAccountsByOwner(
        new PublicKey(walletAddress),
        { programId: TOKEN_PROGRAM_ID }
    );

    const userTokens = tokenAccounts.value.map(account => ({
        mintAddress: account.account.data.parsed.info.mint,
        tokenAddress: account.pubkey.toBase58(),
        balance: account.account.data.parsed.info.tokenAmount.uiAmount,
    }));

    const knownTokenAddresses = knownTokens.map(token => token.address);
    const unknownTokens = userTokens.filter(token => !knownTokenAddresses.includes(token.mintAddress));

    return { knownTokens, unknownTokens };
}

const walletAddress = "Your_Solana_Wallet_Address";
const { knownTokens, unknownTokens } = await fetchTokens(walletAddress);
console.log("Known Tokens:", knownTokens);
console.log("Unknown Tokens:", unknownTokens);
// filter out unknown nfts from quick node nfts
```