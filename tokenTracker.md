# NFT and Token Tracker for Solana

This documentation provides an overview of how our application tracks NFTs and SPL tokens on the Solana blockchain, along with how to display a list of activities related to these assets. The solution uses key Solana infrastructure such as QuickNode and Solana Registry while focusing on token and NFT ownership tracking.

## Objective

1. **Track NFTs**: Identify and filter verified NFTs from an owner's account using the Solana blockchain.
2. **Track SPL Tokens**: Manage and display SPL tokens while ensuring that custom tokens and unverified tokens are handled properly.


## 1. Tracking NFTs

### High-Level Process:

#### Fetch Token Account/Owner Information:

Use the function `getParsedTokenAccountsByOwner` to get all tokens owned by a specific user.

#### Metadata Mapping:

Filter out the tokens that correspond to NFTs by accessing the metadata of each token.

#### Verification Using QuickNode API:

Query the QuickNode API to identify verified and unverified NFTs.

#### Filter Unverified NFTs:

Ensure that the unverified NFTs are excluded from the final display list, leaving only the verified ones.

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
const ownerTokens = await getParsedTokenAccountsByOwner(walletAddress);
const nftTokens = filterNftTokens(ownerTokens); // Custom filter function
const verifiedNfts = quickNodeApi.getVerifiedNfts(nftTokens);
```

### Example Code:
```typescript
import { TokenListProvider } from '@solana/spl-token-registry';

const tokenList = await new TokenListProvider().resolve();
const knownTokens = tokenList.filterByChainId(101); // Fetch Solana Mainnet tokens
const unknownTokens = fetchUnknownTokens(userTokens, knownTokens); 
// filter out unknown nfts from quick node nfts
```