# NFT and Token Tracker for Solana

This documentation provides an overview of how our application tracks NFTs and SPL tokens on the Solana blockchain, along with how to display a list of activities related to these assets. The solution uses key Solana infrastructure such as QuickNode and Solana Registry while focusing on token and NFT ownership tracking.

## Objective

1. **Track NFTs**: Identify and filter verified NFTs from an owner's account using the Solana blockchain.
2. **Track SPL Tokens**: Manage and display SPL tokens while ensuring that custom tokens and unverified tokens are handled properly.


## 1. Tracking NFTs

### High-Level Process:

1. #### Fetch Token Account/Owner Information:

- Use the function `getParsedTokenAccountsByOwner`[RPC method](https://solana-labs.github.io/solana-web3.js/classes/Connection.html#getParsedTokenAccountsByOwner). to get all tokens owned by a specific user.
- This function queries the Solana blockchain to retrieve token accounts associated with a given wallet address.
ref: 

2. #### Metadata Mapping:

- Filter out the tokens that correspond to NFTs by accessing the metadata of each token.
- NFTs in solana typically have specific characteristics, such as having a token amount of 1 and no decimals.

3. #### Verification Using QuickNode API:

- Query the QuickNode API to identify verified and unverified NFTs.
- QuickNode provides an API that can be used to verify the authenticity of NFTs.

4. #### Filter Unverified NFTs:

- Ensure that the unverified NFTs are excluded from the final display list, leaving only the verified ones.
- This step ensures that only legitimate NFTs are shown to the user.

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
1. Import Required Modules:
- Import the necessary classes from the Solana Web3.js library and the SPL Token library.

2. Define fetchAndFilterNFTs Function:
- Create an asynchronous function fetchAndFilterNFTs that takes a wallet address as a parameter.
- Establish a connection to the Solana mainnet.
- Fetch the token accounts for the given wallet address using getParsedTokenAccountsByOwner.
- Filter the token accounts to identify NFTs based on their token amount and decimals.
- Use the QuickNode API to verify the NFTs.
- Return the list of verified NFTs.

3. Execute fetchAndFilterNFTs:
- Call the fetchAndFilterNFTs function with a specified wallet address and log the verified NFTs to the console.

## 2. Tracking SPL Tokens
1. #### Token Fetching Workflow:
- Fetch a list of tokens using the Solana SPL token registry `new TokenListProvider()`[query tokens](https://github.com/solana-labs/token-list?tab=readme-ov-file#query-available-tokens).
- The SPL token registry provides a list of known tokens on the Solana blockchain [Token List](https://raw.githubusercontent.com/solana-labs/token-list/refs/heads/main/src/tokens/solana.tokenlist.json).

2. #### Token List via Registry:
- Use the TokenListProvider from the SPL token registry to fetch the list of known tokens[TokenListProvider](https://github.com/solana-labs/token-list/blob/main/src/lib/tokenlist.ts)
- This step ensures that we have an up-to-date list of tokens that are recognized on the Solana blockchain.

3. #### Handle Unknown Tokens
- Handle tokens that are not in the registry but are present in the user's account by categorizing them as custom tokens.
- This step ensures that even tokens not recognized by the registry are still tracked and displayed.

4. #### Custom Token Management:
- Allow users to add custom tokens which are fetched and validated and stored separately in our database.
- This step provides flexibility for users to manage tokens that are not part of the official registry.

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
1. Import Required Modules:
- Import the necessary classes from the Solana Web3.js library and the SPL Token Registry library.

2. Define fetchTokens Function:
- Create an asynchronous function fetchTokens that takes a wallet address as a parameter.
- Establish a connection to the Solana mainnet.
- Use the TokenListProvider to fetch the list of known tokens from the SPL token registry.
- Fetch the token accounts for the given wallet address using getParsedTokenAccountsByOwner.
- Map the token accounts to extract relevant details such as mint address, token address, and balance.
- Identify unknown tokens by comparing the user's tokens with the known tokens from the registry.
- Return the list of known and unknown tokens.

3. Execute fetchTokens:
- Call the fetchTokens function with a specified wallet address and log the known and unknown tokens to the console.


## Concepts and Implementation

### QuickNode
- QuickNode is a blockchain infrastructure provider that offers APIs for interacting with various blockchains, including Solana.
- In this implementation, QuickNode is used to verify NFTs by querying their API to check the authenticity of the NFTs.

### SPL Token Package
- The SPL Token package provides utilities for interacting with SPL tokens on the Solana blockchain.
- It includes functions for fetching token accounts, parsing token data, and managing token metadata.

## Summary
By leveraging QuickNode and the SPL Token package, our application can effectively track and manage NFTs and SPL tokens on the Solana blockchain. The process involves fetching token accounts, filtering and verifying NFTs, and managing known and unknown tokens. This ensures that users have a comprehensive view of their assets while maintaining the integrity and authenticity of the tracked tokens.