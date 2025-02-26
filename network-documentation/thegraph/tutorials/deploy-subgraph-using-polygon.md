# Introduction

In this tutorial you will learn how to deploy a Solidity smart contract to the Polygon Mumbai testnet using HardHat, then create and deploy its subgraph to the Subgraph Studio.

![Subgraph Studio](../../../.gitbook/assets/graph.png)

The subgraph studio is where you can create a subgraph using the Studio UI, deploy a subgraph using the Graph Protocol CLI, test it in the Playground (where you can run GraphQL queries without incurring any querying fees), add metadata about the subgraph, deploy multiple versions of your subgraphs, as well as publish them to the decentralized Graph Explorer.

When you deploy a subgraph, you alone shall have access to it. But when you publish your subgraph, anyone can access and test your subgraph in the Playground. So basically, you have full control of your subgraphs.

In this tutorial, you shall learn how to create and deploy your subgraph to Subgraph Studio using a Polygon smart contract.

# Prerequisites

To successfully complete this tutorial, you will need to have a basic understanding of Polygon and the NodeJS ecosystem. You will also need to have advanced knowledge of Solidity.

# Requirements

- You will need Metamask installed in your browser. You can install it from <https://metamask.io/>
- You need to have a recent version of Node.js installed. We recommend using v14.18.1 LTS for compatibility.

# Project setup

Run the following command to install the yarn package manager and the Graph Protocol CLI globally. These are required to build and deploy your subgraph.

```text
npm i -g yarn @graphprotocol/graph-cli
```

Then run the following commands to create a new directory called `nft`, change into that directory, and then create a new yarn package inside it:

```text
mkdir nft
cd nft
yarn init --yes
```

## Setting up Hardhat

Run the following command to install `hardhat` as a dev-dependency of the project (meaning that it is a package that is only needed for local development and testing).

```text
yarn add --dev hardhat
```

Then run `npx hardhat` to initialize the Hardhat project. This will output:

```text
888    888                      888 888               888
888    888                      888 888               888
888    888                      888 888               888
8888888888  8888b.  888d888 .d88888 88888b.   8888b.  888888
888    888     "88b 888P"  d88" 888 888 "88b     "88b 888
888    888 .d888888 888    888  888 888  888 .d888888 888
888    888 888  888 888    Y88b 888 888  888 888  888 Y88b.
888    888 "Y888888 888     "Y88888 888  888 "Y888888  "Y888

Welcome to Hardhat v2.6.5

? What do you want to do? …
▸ Create a basic sample project
  Create an advanced sample project
  Create an advanced sample project that uses TypeScript
  Create an empty hardhat.config.js
  Quit
```

You can choose any of the 5 menu options above, by using the arrow keys on your keyboard. Select "Create an empty hardhat.config.js" and press Enter. This will output: `Config file created`.

You will now have a `hardhat.config.js` file created, with the below contents:

```javascript
/**
 * @type import('hardhat/config').HardhatUserConfig
 */
module.exports = {
  solidity: "0.7.3",
};
```

Update the solidity version to `0.8.0`, like shown below:

```javascript
/**
 * @type import('hardhat/config').HardhatUserConfig
 */
module.exports = {
  solidity: "0.8.0",
};
```

HardHat will use the specified version of solc (v0.8.0) to compile your smart contract.

Run the following command to install Hardhat plugins that are required for compiling and deploying a solidity smart contract:

```text
yarn add --dev @nomiclabs/hardhat-ethers ethers @nomiclabs/hardhat-waffle hardhat-abi-exporter
```

# Writing the smart contract

To simplify writing our NFT smart contract, we shall be using the OpenZeppelin contracts npm package. You can install it as a dev-dependency by running:

```text
yarn add --dev @openzeppelin/contracts
```

We shall be using three OpenZeppelin smart contracts:

- **Counters** - to keep track of an incremental tokenId for each NFT minted
- **ERC721URIStorage** - to store the NFT metadata like the tokenURI (which contains some metadata and image)
- **ERC721** - to provide some basic functions of ERC721 standard

Create a new directory called `contracts`, then create a file inside it called `nft.sol`.

Paste this Solidity code into that file:

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

import "@openzeppelin/contracts/utils/Counters.sol";
import "@openzeppelin/contracts/token/ERC721/extensions/ERC721URIStorage.sol";
import "@openzeppelin/contracts/token/ERC721/ERC721.sol";

contract FunNFT is ERC721URIStorage {
    using Counters for Counters.Counter;
    Counters.Counter private _tokenIds;

    struct Collection {
      uint tokenId;
      uint price;
      bool forSale;
    }

    mapping(uint => Collection) collections;

    constructor() ERC721("FunNFT Collection", "FUN") {}

    function mint(string memory tokenURI, uint price) public returns (uint tokenId) {
        // increment the counter, to get the next tokenId.
        _tokenIds.increment();
        tokenId = _tokenIds.current();

        // mint the new NFT and then set its tokenURI.
        _mint(msg.sender, tokenId);
        _setTokenURI(tokenId, tokenURI);

        // add it to the collection.
        Collection memory collection = Collection(
            tokenId,
            price,
            false
        );
        collections[tokenId] = collection;
    }

    function buy(uint tokenId) public payable {
        // verify that the NFT exists.
        require(_exists(tokenId));

        // verify that owner of the token is not the one calling this function.
        require(msg.sender != ownerOf(tokenId));

        Collection memory collection = collections[tokenId];

        // verify that the price sent is greater than equal to NFT's price.
        require(msg.value >= collection.price);

        // verify that the NFT is available to buy
        require(collection.forSale);

        // transfer the NFT from the owner to the buyer.
        address owner = ownerOf(tokenId);
        payable(owner).transfer(msg.value);
        _transfer(owner, msg.sender, tokenId);

        // update the mapping.
        collection.forSale = false;
        collections[tokenId] = collection;
    }

    function toggleForSale(uint tokenId) public {
        // verify that the NFT exists.
        require(_exists(tokenId));

        // verify that owner of the token is the one calling this function.
        require(msg.sender == ownerOf(tokenId));

        // get the NFT from mapping and store it in memory.
        Collection memory collection = collections[tokenId];

        collection.forSale = !collection.forSale;

        // update the NFT in the mapping.
        collections[tokenId] = collection;
    }
}
```

# Compiling the smart contract

By default, Hardhat will not export the ABI after compiling the solidity smart contract. But having the ABI makes it easier to create a subgraph for it.

You can export the ABI by updating your `hardhat.config.js` file:

```javascript
// Load the Hardhat plugins
require('@nomiclabs/hardhat-waffle');
require('hardhat-abi-exporter');

module.exports = {
  solidity: '0.8.0',
  abiExporter: {
    path: './abi/',
    clear: true,
  },
};
```

- **path**: The path to your ABI export directory (relative to the HardHat project's root). The directory will be created if it does not exist.
- **clear**: A boolean value that determines whether to delete old files from the specified path on compilation.

You can read about the full set of options supported by Hardhat ABI Exporter [here](https://hardhat.org/plugins/hardhat-abi-exporter.html#usage).

Run `npx hardhat compile` to compile your smart contract. Assuming there are no errors or warnings, this will output:

```text
Compiling 1 file with 0.8.0
Compilation finished successfully
```

The `nft.sol` smart contract has now been compiled successfully. The ABI will be stored in `abi/contracts/nft.sol/FunNFT.json`.

# Deploying the smart contract

Add the constants `MUMBAI_PRIVATE_KEY` and `DATAHUB_API_KEY` as well as a **networks** entry in your `hardhat.config.js` file:

```javascript
require('@nomiclabs/hardhat-waffle');
require('hardhat-abi-exporter');

// Replace this private key with your Mumbai wallet private key
const MUMBAI_PRIVATE_KEY = 'YOUR_MUMBAI_PRIVATE_KEY';

// Replace this with your Datahub api key
const DATAHUB_API_KEY = 'YOUR_DATAHUB_API_KEY';

module.exports = {
  solidity: '0.8.0',
  abiExporter: {
    path: './abi/',
    clear: true,
  },
  networks: {
    mumbai: {
      url: `https://matic-mumbai--jsonrpc.datahub.figment.io/apikey/${DATAHUB_API_KEY}`,
      accounts: [`0x${MUMBAI_PRIVATE_KEY}`],
    },
  },
};
```

## Getting your Datahub API Key

To get your `DATAHUB_API_KEY`, head over to <https://datahub.figment.io/sign_up> and create a free account.

![Datahub Signup](../../../.gitbook/assets/datahub_signup.png)

Click on the verification link you shall receive in your email. After email verification, you will be redirected to <https://datahub.figment.io/>.

Click on the `Polygon` button to view the Polygon Services Dashboard at <https://datahub.figment.io/services/polygon>. You can copy your API key to the clipboard by clicking on the **copy** button.

![Datahub Polygon API Key](../../../.gitbook/assets/datahub_polygon_api.png)

## Getting your Metamask private key

You need to add a connection to the Polygon Mumbai testnet to your Metamask by following the instructions below (remember to replace `<YOUR_DATAHUB_API_KEY>` in the JSON-RPC URL with your actual DataHub API key):

- Login in to Metamask
- Click on the `Network` drop-down
- Select `Custom RPC`
- Fill in the details:
  - **Network Name**: Polygon Mumbai Testnet
  - **New RPC URL**: `https://matic-mumbai--jsonrpc.datahub.figment.io/apikey/<YOUR_DATAHUB_API_KEY>`
  - **ChainID**: `80001`
  - **Symbol**: `MATIC`
  - **Explorer**: `https://mumbai.polygonscan.com/`

  You will also need some Mumbai MATIC tokens in your Metamask wallet. Follow the instructions below:

  - Go to <https://faucet.polygon.technology/>, and paste your Metamask wallet address.
  - Select Mumbai Network and MATIC token.
  - You shall receive 0.1 MATIC tokens once your transaction is mined

To get your `MUMBAI_PRIVATE_KEY`, open your browser and open Metamask. Select **Polygon Mumbai Testnet** network. Click on Account details, and click on `Export Private Key`.

**Never ever share your private key with anyone!**

## Deploying your smart contract

Create a subdirectory inside the HardHat project called `scripts`, and paste the following code into a new file `deploy.js` inside the `scripts` directory:

```javascript
async function main() {
  const [deployer] = await ethers.getSigners();

  console.log("Deploying contracts with the account:", deployer.address);
  console.log("Account balance:", (await deployer.getBalance()).toString());

  const FunNFT = await ethers.getContractFactory("FunNFT");
  const funNFT = await FunNFT.deploy();

  console.log("Contract address:", funNFT.address);
}

main()
  .then(() => process.exit(0))
  .catch((error) => {
    console.error(error);
    process.exit(1);
  });
```

To deploy the smart contract to the Mumbai testnet using HardHat, run the command:

```text
npx hardhat run scripts/deploy.js --network mumbai
```

This will output:

```text
Deploying contracts with the account: <YOUR MUMBAI WALLET ADDRESS>
Account balance: <YOUR MUMBAI WALLET BALANCE>
Contract address: <YOUR MUMBAI SMART CONTRACT ADDRESS>
```

Congratulations! You just deployed your smart contract successfully to Mumbai testnet!

You will need the contract address when creating the subgraph, so remember to copy it.

# Creating the subgraph

Run the following command to create the subgraph by downloading the contract ABI from the Mumbai testnet. A new directory called `funnft` will be created, and all node dependencies will be installed automatically.

```text
graph init --contract-name FunNFT --index-events --studio \
  --from-contract <YOUR MUMBAI SMART CONTRACT ADDRESS> \
  --abi abi/contracts/nft.sol/FunNFT.json \
  --network mumbai \
  funnft
```

Replace `<YOUR MUMBAI SMART CONTRACT ADDRESS>` with your deployed FunNFT smart contract address from above.

If you are a Windows user, replace `\` characters in the above command with `^` character.

This will output:

```text
✔ Subgraph slug · funnft
✔ Directory to create the subgraph in · funnft
✔ Ethereum network · mumbai
✔ Contract address · <YOUR MUMBAI SMART CONTRACT ADDRESS>
✔ ABI file (path) · abi/contracts/nft.sol/FunNFT.json
✔ Contract Name · FunNFT
———
  Generate subgraph from ABI
  Write subgraph to directory
✔ Create subgraph scaffold
✔ Initialize subgraph repository
✔ Install dependencies with yarn
✔ Generate ABI and schema types with yarn codegen

Subgraph funnft created in funnft

Next steps:

  1. Run `graph auth` to authenticate with your deploy key.

  2. Type `cd funnft` to enter the subgraph.

  3. Run `yarn deploy` to deploy the subgraph.

Make sure to visit the documentation on https://thegraph.com/docs/ for further information.
```

# Creating the project in Subgraph Studio

First, you will want to head over to the Subgraph Studio at <https://thegraph.com/studio/>.

![Login to Subgraph Studio](../../../.gitbook/assets/graph_connect.png)

Click on the **Connect Wallet** button. Choose a Metamask wallet to login with. Once you are authenticated, you shall see the below screen, where you can create your first subgraph.

![Create your first subgraph](../../../.gitbook/assets/graph_create_subgraph.png)

Next, you need to give your subgraph a name. Give the name as **FunNFT**. Once that's done, you will see this screen:

![Subgraph dashboard](../../../.gitbook/assets/graph_fun_nft.png)

On this screen, you can see details about the subgraph like your deploy key, the subgraph slug and status.

# Deploying the subgraph

Before you can deploy your subgraph, you need to get your deploy key from <https://thegraph.com/studio/subgraph/funnft/>. This is used to authenticate your account with the Subgraph Studio.

Run the following commands to set your deploy key.

```text
cd funnft
graph auth --studio <DEPLOY_KEY>
```

You should see the following output:

```text
Deploy key set for https://api.studio.thegraph.com/deploy/
```

Once that is done, you can run `yarn deploy` to deploy the subgraph to Subgraph Studio. When you are prompted for a version label, you can choose `v1.0.0`. You should see the following output:

```text
yarn run v1.22.15
$ graph deploy --node https://api.studio.thegraph.com/deploy/ funnft
✔ Version Label (e.g. v0.0.1) · v1.0.0
  Skip migration: Bump mapping apiVersion from 0.0.1 to 0.0.2
  Skip migration: Bump mapping apiVersion from 0.0.2 to 0.0.3
  Skip migration: Bump mapping apiVersion from 0.0.3 to 0.0.4
  Skip migration: Bump mapping apiVersion from 0.0.4 to 0.0.5
  Skip migration: Bump mapping specVersion from 0.0.1 to 0.0.2
✔ Apply migrations
✔ Load subgraph from subgraph.yaml
  Compile data source: FunNFT => build/FunNFT/FunNFT.wasm
✔ Compile subgraph
  Copy schema file build/schema.graphql
  Write subgraph file build/FunNFT/abis/FunNFT.json
  Write subgraph manifest build/subgraph.yaml
✔ Write compiled subgraph to build/
  Add file to IPFS build/schema.graphql
                .. QmbVoYXPnjucXQv8cRDVRF8BH77YMSFWCvVu9sFQEQGhWg
  Add file to IPFS build/FunNFT/abis/FunNFT.json
                .. QmNiduSnZPvFKuiyhxdp1V8FSfhxgdtNx6TRkifugbL1XA
  Add file to IPFS build/FunNFT/FunNFT.wasm
                .. QmcKpXkdPTAjofYwqXCNreKk3AnRma7qxdk5v3jAyJSE8y
✔ Upload subgraph to IPFS

Build completed: QmVyyxKVJgGae8AC6SYLzg11huUKNmjSGos81B9ghXXWfe

Deployed to https://thegraph.com/studio/subgraph/funnft

Subgraph endpoints:
Queries (HTTP):     https://api.studio.thegraph.com/query/8676/funnft/v1.0.0
Subscriptions (WS): https://api.studio.thegraph.com/query/8676/funnft/v1.0.0

Done in 14.68s.
```

You have now deployed the subgraph to your Subgraph Studio account!

# Conclusion

Congratulations on finishing this tutorial! You have learned how to deploy a smart contract to the Polygon Mumbai testnet using HardHat. You also learned how to create and deploy a subgraph for the smart contract on Subgraph Studio.

# About the Author

I'm Robin Thomas, a blockchain enthusiast with few years of experience working with various blockchain protocols. Feel free to connect with me on [GitHub](https://github.com/robin-thomas).
