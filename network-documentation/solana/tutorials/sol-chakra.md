# Introduction

Many existing tutorials about Solana focus on writing _programs_ (commonly known as smart contracts on other blockchains), minting tokens or NFTs - however a decentralized app is incomplete without an interface and needs to be hosted on the web to reach a wider audience. In this tutorial, we will explain how to develop a frontend interface for Solana programs and how to make beautiful web apps with the help of Chakra UI.

The code for this tutorial is available in the repository [https://github.com/kanav99/solana-boilerplate](https://github.com/kanav99/solana-boilerplate). Each part of the tutorial refers to the repository, which is mentioned at the end of the part. If you are comfortable with using git on the commandline, you can `git checkout <commit hash>` or just click on the link to the code to refer to the code at that checkpoint.

# Prerequisites

- Have a basic understanding of Solana backend and have gone through the helloworld example [here](https://github.com/solana-labs/example-helloworld).

- A basic understanding of React.js is required.

# Requirements

For this tutorial, we need the following software installed -

- Node.js version 14.18.1 or higher
- The Rust toolchain, which can be found at [https://rustup.rs](https://rustup.rs)
- The Solana CLI, which can be found at [https://docs.solana.com/cli/install-solana-cli-tools](https://docs.solana.com/cli/install-solana-cli-tools)
- A Solana Wallet (like Phantom, Solflare, etc.), check out the Wallet guide [here](https://docs.solana.com/wallet-guide).

# Create an empty Chakra App

To get started, we will create an empty Chakra UI app using `create-react-app`, which is a program used to set up a React development environment and generate a starting point from templates. Run the following commands in your terminal:

```
npx create-react-app solana-boilerplate --template @chakra-ui
cd solana-boilerplate
```

- This creates a scaffolding for a simple web app with the main file of the application, `src/App.js`, containing a welcome message and a rotating Logo. We don't need that, so let's remove the existing contents of App.js and replace it with the following:

```jsx
import React from "react";
import {
  ChakraProvider,
  Box,
  Text,
  VStack,
  Grid,
  theme,
} from "@chakra-ui/react";
import { ColorModeSwitcher } from "./ColorModeSwitcher";

function App() {
  return (
    <ChakraProvider theme={theme}>
      <Box textAlign="center" fontSize="xl">
        <Grid minH="100vh" p={3}>
          <ColorModeSwitcher justifySelf="flex-end" />
          <VStack spacing={8}>
            <Text>Hello world!</Text>
          </VStack>
        </Grid>
      </Box>
    </ChakraProvider>
  );
}

export default App;
```

To start the development server, use the command `npm start` in your terminal. Once it has loaded you will be able to visit the running app in your browser at `http://localhost:3000` - you'll see an empty page with a convenient colour mode switching button with a "Hello world!" floating in the middle.

If this is your first time seeing a Chakra UI app, here is a refresher. All chakra components must be wrapped between `ChakraProvider`, which controls the theming of all children components. `Box` is equivalent to `div` tag of HTML. `VStack` is a vertical stack of elements spaced evenly.

# Basic interaction with the Solana Network

To interact with the Solana Networks (mainnet, devnet, local etc.), we use the Solana [JSON RPC API](https://docs.solana.com/developing/clients/jsonrpc-api). Instead of making raw jRPC calls, we will use the package `@solana/web3.js`, which interacts with the API.

- We begin by installing the package to the app

```
npm install --save @solana/web3.js
```

- Import the package in App.js

```jsx
import * as web3 from "@solana/web3.js";
```

- We are not using wallets as of now (more on that in subsequent sections), so we will make a makeshift wallet; if a private key does not exist in local storage, generate a new one. So, we add this code to the global scope

```jsx
// Making a connection with Solana devnet
const connection = new web3.Connection(
  web3.clusterApiUrl("devnet"),
  "confirmed"
);

// Access a localstorage item pvkey
const pvkey = localStorage.getItem("pvkey");
var wallet;
if (pvkey === null) {
  // if nothing is found
  // generate a new wallet keypair and store the private key in localstorage
  wallet = web3.Keypair.generate();
  localStorage.setItem("pvkey", wallet.secretKey);
} else {
  // if existing wallet is found, parse the wallet
  let arr = new Uint8Array(pvkey.replace(/, +/g, ",").split(",").map(Number));
  // and create a wallet object
  wallet = web3.Keypair.fromSecretKey(arr);
}
```

Keep in mind - this makeshift wallet is not to be used in production. We will replace this with actual wallets in the coming sections; the current code is just for understanding the structure. Now we have set up a connection with the network and have a wallet ready as well. Let us display some basic properties of the wallet - namely, public key and balance. The public key can be retrieved by `wallet.publicKey.toBase58()`, but the balance must be fetched from the network.

In the App.js file, to retrieve balance on-load, we use `useEffect` to interact with the network. We maintain a state variable to store the account info. Inside the App function,

```jsx
const [account, setAccount] = useState(null);

useEffect(() => {
  async function init() {
    // get account info from the network
    let acc = await connection.getAccountInfo(wallet.publicKey);
    setAccount(acc);
  }
  init();
}, []);
```

We have fetched the account details. This object contains the balance in lamport units, which is equivalent to 1/1000000000 of a SOL. Now to display the public key and balance, we change the rendering code to -

```jsx
<Text>Wallet Public Key: {wallet.publicKey.toBase58()}</Text>
<Text>
  Balance:{' '}
  {account ? (account.lamports / web3.LAMPORTS_PER_SOL) + ' SOL' : 'Loading..'}
</Text>
```

You can also use `connection.getBalance` to get only balance as well.

All the code till now is present in the [`d7ecab7`](https://github.com/kanav99/solana-boilerplate/tree/d7ecab78045b74447df797d989a70247f633c4c4) commit of the final repository.

# Getting Airdrops

To work with the Solana network, we need some SOLs. To get them on the mainnet, we need to buy them from any exchange and transfer them to the public key displayed on the page. However, we are on devnet, so we can just get them for free through airdrops. We can do it from the CLI, but let's make an easy button to get them on click.

Import `useCallback` from react, `Button` and `toast` (we will use [toasts](https://chakra-ui.com/docs/feedback/toast) for beautiful erroring) from chakra. First, define the callback that gets an airdrop and updates account object.

```jsx
const toast = useToast();
const [airdropProcessing, setAirdropProcessing] = useState(false);

const getAirdrop = useCallback(async () => {
  setAirdropProcessing(true);
  try {
    var airdropSignature = await connection.requestAirdrop(
      wallet.publicKey,
      web3.LAMPORTS_PER_SOL
    );
    await connection.confirmTransaction(airdropSignature);
  } catch (error) {
    toast({ title: "Airdrop failed", description: error });
  }
  let acc = await connection.getAccountInfo(wallet.publicKey);
  setAccount(acc);
  setAirdropProcessing(false);
}, [toast]);
```

And add a button in the rendering part -

```jsx
<Button onClick={getAirdrop} isLoading={airdropProcessing}>
  Get Airdrop of 1 SOL
</Button>
```

Get code on commit [`83fa3e5`](https://github.com/kanav99/solana-boilerplate/tree/83fa3e5877bf1a5b21a0d1c47054d2ca5158978b).

# Get Transaction history

Suppose we want to get the ten most recent transactions whenever we load the page. We can create a new react state `transactions`, and update the code inside the `init` function.

```jsx
const [transactions, setTransactions] = useState(null);
...
  async function init() {
    let acc = await connection.getAccountInfo(wallet.publicKey);
    setAccount(acc);
    let transactions = await connection.getConfirmedSignaturesForAddress2(
      wallet.publicKey,
      {
        limit: 10,
      }
    );
    setTransactions(transactions);
  }
```

and add this loop inside the rendering part.

```jsx
<Heading>Transactions</Heading>;
{
  transactions && (
    <VStack>
      {transactions.map((v, i, arr) => (
        <HStack key={"transaction-" + i}>
          <Text>Signature: </Text>
          <Code>{v.signature}</Code>
        </HStack>
      ))}
    </VStack>
  );
}
```

But we also want to call this `init` function after we airdrop, so that the transaction that added the SOLs also gets logged. So we can move this init function out of `useEffect`, but inside the `App` functional component. Hence, just before the function `getAirdrop` ends, we can call `init` and update the account balance and transactions.

You might be seeing a problem over here - the transaction, balance, or account status, in general, is not real-time. This is what we will fix in the next section. For the code till now, refer to the commit [`f50143c`](https://github.com/kanav99/solana-boilerplate/tree/f50143cb9d9c1956a57f1bff55b09cebcfdbf1fc).

# Realtime account updates using polling and custom React Hooks

Ever wondered how functions like `useToast` (in chakra), `useWallet` (in most web3 frameworks) work? These are all custom hooks. Right now, in our code, we have all of the code for frontend, wallet info, getting airdrop is all in a single function `App`. The account info is not real-time. The code in the `App` functional component should not have access to `setAccounnt` or `setTransactions`; it should just receive account info and a list of transactions and show it as a list. We will fix all of it using React custom hooks. Using hooks, we can localize a part of the application's state inside a separate function that internally manages and updates the value of those states, returning only the value of the real-time state. If you do not understand any part of what is coming, bear with me until the code for the hook is complete.

All custom hooks should start with "use", so let us call our hook `useSolanaAccount`. We know for sure that the hook should have the state of account and transactions and should return both.

```jsx
function useSolanaAccount() {
  const [account, setAccount] = useState(null);
  const [transactions, setTransactions] = useState(null);
  // updating logic here
  return { account, transactions };
}
```

Now the empty state is not that useful; let us update it. We want the state to be updated every 1 second. So, it would suffice to run the same old `init` function every 1 second. So, let us move that function in here.

```jsx
function useSolanaAccount() {
  const [account, setAccount] = useState(null);
  const [transactions, setTransactions] = useState(null);

  async function init() {
    let acc = await connection.getAccountInfo(wallet.publicKey);
    setAccount(acc);
    let transactions = await connection.getConfirmedSignaturesForAddress2(
      wallet.publicKey,
      {
        limit: 10,
      }
    );
    setTransactions(transactions);
  }

  // more code here

  return { account, transactions };
}
```

Now, to run `init` every 1 second, we will use the `useEffect` hook to set an interval using `setInterval`. You may ask why inside `useEffect`? That's because we only want to set the interval once - writing it directly inside the function body will call it every time the state updates, causing it to run the `init` function multiple times (i.e., the number of times state changed) every second (too much math? you can remember it as golden rule - don't run any fetch-y code directly inside the react functional component or hook). The final hook looks like this -

```jsx
function useSolanaAccount() {
  const [account, setAccount] = useState(null);
  const [transactions, setTransactions] = useState(null);

  async function init() {
    let acc = await connection.getAccountInfo(wallet.publicKey);
    setAccount(acc);
    let transactions = await connection.getConfirmedSignaturesForAddress2(
      wallet.publicKey,
      {
        limit: 10,
      }
    );
    setTransactions(transactions);
  }

  useEffect(() => {
    setInterval(init, 1000);
  }, []);

  return { account, transactions };
}
```

We can now remove the definition, and all calls of `init` from `App` and change the state inside it to

```jsx
...
const { account, transactions } = useSolanaAccount();
const toast = useToast();
const [airdropProcessing, setAirdropProcessing] = useState(false);

...
```

Voila! No more `setAccount` and `setTransactions` inside the App component. No more manually updating the state after every change. Try sending some SOLs from the CLI to this account and see it update in real-time! As an exercise to the reader, try using [`connection.onAccountChange`](https://solana-labs.github.io/solana-web3.js/classes/Connection.html#onAccountChange) instead of setInterval for getting updates :). Code until now is present in the commit [`c598b22`](https://github.com/kanav99/solana-boilerplate/tree/c598b220eac2070cfa4b6fc146d8cdf071ee1f40).

# Using an actual wallet

Now, before spending any of their precious SOLs, we want our users to feel safe about how we handle wallets. It is always good to leave the wallet logic and how the private key is stored to existing well-known projects. Solana has many wallet options like Solflare, Sollet, Phantom etc. We will make our application compatible with all of these using [`solana/wallet-adapter`](https://github.com/solana-labs/wallet-adapter) package(s). The goal of this section would be to remove this particular piece of code -

```jsx
const connection = new web3.Connection(
  web3.clusterApiUrl("devnet"),
  "confirmed"
);

const pvkey = localStorage.getItem("pvkey");
var wallet;
if (pvkey === null) {
  wallet = web3.Keypair.generate();
  localStorage.setItem("pvkey", wallet.secretKey);
} else {
  let arr = new Uint8Array(pvkey.replace(/, +/g, ",").split(",").map(Number));
  wallet = web3.Keypair.fromSecretKey(arr);
}
```

... and replace it with something safer. The current implementation enables the person who hosts this application access to every user's wallet, which is not right.

- Start by installing the required packages

```bash
npm i @solana/wallet-adapter-wallets \
      @solana/wallet-adapter-base \
      @solana/wallet-adapter-react \
      @solana/wallet-adapter-react-ui
```

- Add the required imports just after the previous imports end. Note that the last import is done using `require` and would need to be after all the `import`s at all the times.

```jsx
import {
  ConnectionProvider,
  WalletProvider,
  useConnection,
  useWallet,
} from "@solana/wallet-adapter-react";
import {
  getPhantomWallet,
  getSolflareWallet,
  getSolletWallet,
  getSolletExtensionWallet,
} from "@solana/wallet-adapter-wallets";
import {
  WalletModalProvider,
  WalletMultiButton,
} from "@solana/wallet-adapter-react-ui";
require("@solana/wallet-adapter-react-ui/styles.css");
```

- Inside the `App` component body, add

```jsx
const network = "devnet";
const endpoint = web3.clusterApiUrl(network);
const wallets = useMemo(
  () => [
    getPhantomWallet(),
    getSolflareWallet(),
    getSolletWallet({ network }),
    getSolletExtensionWallet({ network }),
  ],
  [network]
);
```

Here, we define three constants. First is `network`, just the string which tells which network we are on. The `endpoint` is a string containing the selected network's RPC URL (i.e. Devnet). The third constant, `wallets`, is an array of wallet descriptors that we want our app to work with. The `useMemo` is a function that memoizes the value of the function output (passed as the first argument) and only changes when a list of state variables change their value (this dependency array is passed as the second argument). We currently will work with Phantom, Solflare and Sollet.

To make all of the children components of `App` able to access the network and wallet, we have to wrap all sub-components in `ConnectionProvider` and `WalletProvider`. Then, the children elements can use `useWallet` and `useConnection` to access the wallet and network. After wrapping the components in these, the rendering of App should look something like this -

```jsx
return (
  <ChakraProvider theme={theme}>
    <ConnectionProvider endpoint={endpoint}>
      <WalletProvider wallets={wallets} autoConnect>
        <WalletModalProvider>{/* All the same elements */}</WalletModalProvider>
      </WalletProvider>
    </ConnectionProvider>
  </ChakraProvider>
);
```

Let us now delete our old definition of `connection`, `wallet` and `pvkey` - let a wallet provide that. Also, we will have to refactor our initial code such that we have a new `Home` functional modal which contains all the design and logic, and `App` should only have these providers. So, the App becomes -

```jsx
function App() {
  const network = "devnet";
  const endpoint = web3.clusterApiUrl(network);
  const wallets = useMemo(
    () => [
      getPhantomWallet(),
      getSolflareWallet(),
      getSolletWallet({ network }),
      getSolletExtensionWallet({ network }),
    ],
    [network]
  );

  return (
    <ChakraProvider theme={theme}>
      <ConnectionProvider endpoint={endpoint}>
        <WalletProvider wallets={wallets} autoConnect>
          <WalletModalProvider>
            <Home></Home>
          </WalletModalProvider>
        </WalletProvider>
      </ConnectionProvider>
    </ChakraProvider>
  );
}
```

... and move all the contents to Home. Inside Home, we have to take care of some things. First, the `connection` object now comes from `useConnection`. Second, the `publicKey` should now come from `useWallet`. Third, the `publicKey` we now get might be empty because the user might not have connected the wallet yet. In that case, we need to ask the user to connect their wallet with the application.

So, after the changes in the logic, the Home component should look like:

```jsx
function Home() {
  const { connection } = useConnection();
  const { publicKey } = useWallet();
  const { account, transactions } = useSolanaAccount();
  const toast = useToast();
  const [airdropProcessing, setAirdropProcessing] = useState(false);

  const getAirdrop = useCallback(async () => {
    setAirdropProcessing(true);
    try {
      var airdropSignature = await connection.requestAirdrop(
        publicKey,
        web3.LAMPORTS_PER_SOL
      );
      await connection.confirmTransaction(airdropSignature);
    } catch (error) {
      console.log(error);
      toast({ title: "Airdrop failed", description: "unknown error" });
    }
    setAirdropProcessing(false);
  }, [toast, publicKey, connection]);

  return (
    <Box textAlign="center" fontSize="xl">
      <Grid minH="100vh" p={3}>
        <ColorModeSwitcher justifySelf="flex-end" />
        {publicKey && (
          <VStack spacing={8}>
            <Text>Wallet Public Key: {publicKey.toBase58()}</Text>
            <Text>
              Balance:{" "}
              {account
                ? account.lamports / web3.LAMPORTS_PER_SOL + " SOL"
                : "Loading.."}
            </Text>
            <Button onClick={getAirdrop} isLoading={airdropProcessing}>
              Get Airdrop of 1 SOL
            </Button>
            <Heading>Transactions</Heading>
            {transactions && (
              <VStack>
                {transactions.map((v, i, arr) => (
                  <HStack key={"transaction-" + i}>
                    <Text>Signature: </Text>
                    <Code>{v.signature}</Code>
                  </HStack>
                ))}
              </VStack>
            )}
          </VStack>
        )}
        {!publicKey && <WalletMultiButton />}
      </Grid>
    </Box>
  );
}
```

The third part that needs to be changed before we can conclude our wallet adaptation is the `useSolanaAccount` hook:

```jsx
function useSolanaAccount() {
  const [account, setAccount] = useState(null);
  const [transactions, setTransactions] = useState(null);
  const { connection } = useConnection();
  const { publicKey } = useWallet();

  const init = useCallback(async () => {
    if (publicKey) {
      let acc = await connection.getAccountInfo(publicKey);
      setAccount(acc);
      let transactions = await connection.getConfirmedSignaturesForAddress2(
        publicKey,
        {
          limit: 10,
        }
      );
      setTransactions(transactions);
    }
  }, [publicKey, connection]);

  useEffect(() => {
    if (publicKey) {
      setInterval(init, 1000);
    }
  }, [init, publicKey]);

  return { account, transactions };
}
```

Notice that we have converted the `init` function to `useCallback` - that is because we pass this function in the `setInterval`; at that time, the publicKey is empty. So now, when the user changes the publicKey, `init` callback changes; consequently, `useEffect` is called, and a new interval is set then.

There is still one issue in this, which I am leaving as an exercise - the user may change their public key in between. In that case, we have to remove the interval created using `setInterval` using `clearInterval` and then create a new interval.

The code till now is present in the commit [`92c4c88`](https://github.com/kanav99/solana-boilerplate/tree/92c4c885a4a190e3edafcfdbe33309adfab38f8b).

# Sending transactions - Greeting yourself

Until this point, we only read from the blockchain; we have not written anything on it actively (airdrops happen due to a public program). In this part, we will learn to "write" on the blockchain by interacting with the programs deployed on it. Now, we will make a button component that says hi to a person's account, and the account maintains a counter of the number of accounts that greeted it.

For this, let us use the greeter code from the [example-helloworld](https://github.com/solana-labs/example-helloworld) repository. If you have not gone through this example already, I recommend you to. Generally, in Solana Programs, we generate a "program account" that is "owned" by the program but is "related" to your account. Anyone can create an account, not only the owner; you can pay for the space it needs and create a program account. The public key for this program account is generated using your public key and a constant seed, using the function `PublicKey.createWithSeed`. This is basically a one-to-one mapping between your public key and the program account public key, given the seed is fixed. Now, anyone with your public key can generate your program account's public key and tell the program to greet this account (or just increase the counter for this account).

- Clone the repository https://github.com/solana-labs/example-helloworld.
- While being inside the repository, run `npm run build:program-rust` to build the program.
- Deploy the program to devnet using `solana program deploy --url https://api.devnet.solana.com dist/program/helloworld.so`. This should print out a program ID. Please take a note of it. For me, it was FGbjtxeYmT5jUP7aNavo9k9mQ3rGQ815WdvwWndR7FF9, so I will use this in the following example.

Now, let us not cram all the code into a single file. Create a new file, `Greet.jsx`, with this starter code.

```jsx
import React from "react";
import { HStack, Button, Text } from "@chakra-ui/react";

export function Greet() {
  return (
    <HStack>
      <Text>Total greetings: {"0"}</Text>
      <Button>Greet Yourself</Button>
    </HStack>
  );
}
```

And inside the App.js file, add the import -

```jsx
import { Greet } from "./Greet";
```

And just below our airdrop button, add this newly made component -

```jsx
...
  <Button onClick={getAirdrop} isLoading={airdropProcessing}>
    Get Airdrop of 1 SOL
  </Button>
  <Greet />
...
```

Now in this Greet component, we want two things to happen:

1. Get the current number of greetings sent to you.
2. Greet yourself.

We will create an interface to our Rust backend. We will follow what is given in the helloworld example. The code will look very similar to the code present [here](https://github.com/solana-labs/example-helloworld/blob/master/src/client/hello_world.ts).

- Store the program id and a fixed seed as a global constant.

```jsx
const programId = new PublicKey("FGbjtxeYmT5jUP7aNavo9k9mQ3rGQ815WdvwWndR7FF9");
const GREETING_SEED = "hello";
```

- To have a similar interface as in the Rust struct [`GreetingAccount`](https://github.com/solana-labs/example-helloworld/blob/master/src/program-rust/src/lib.rs#L13-L16), we create a similar global class in Javascript.

```jsx
class GreetingAccount {
  counter = 0;
  constructor(fields) {
    if (fields) {
      this.counter = fields.counter;
    }
  }
}
```

- Now, accounts in Solana only store raw bytes. As we serialize/deserialize in the Rust code using borsh, we will do the same here using the borsh package. Deserializing using borsh requires a schema that tells the deserializing logic about the size of different fields. We create that schema next, along with the total size of the serialized class object (we need the size when we create a new greeting account and pay only for the size each greeting account needs).

```jsx
const GreetingSchema = new Map([
  [GreetingAccount, { kind: "struct", fields: [["counter", "u32"]] }],
]);

const GREETING_SIZE = borsh.serialize(
  GreetingSchema,
  new GreetingAccount()
).length;
```

- Whenever we fetch the account info from the blockchain, we get it as a form of [`AccountInfo`](https://solana-labs.github.io/solana-web3.js/modules.html#AccountInfo). To directly fetch the counter from it, we define a straightforward function, which gets the data from AccountInfo, deserializes it, and returns the counter subobject.

```jsx
function counterFromAccountInfo(accountInfo) {
  const data = borsh.deserialize(
    GreetingSchema,
    GreetingAccount,
    accountInfo.data
  );
  return data.counter;
}
```

- We now start writing out `Greet` component by adding the wallet and connection descriptors, along with a state variable containing the current counter of greetings and change the text that shows the counter.

```jsx
const wallet = useWallet();
const { connection } = useConnection();
const [counter, setCounter] = useState(null);
...
<Text>Total greetings: {counter === null ? 'Loading..' : counter}</Text>
```

- First, we will write the logic that fetches the current number of greetings sent to you. What we need to do is, first on page load, get the current number of greetings sent to you, and then add a listener on changes made to the account for any new greetings (this time, we will use `onAccountChange` instead of polling). To get the current number of greetings, we will use `connection.getAccountInfo` on the generated program account inside `useEffect` to do it on load.

```jsx
const greetedPubkey = await PublicKey.createWithSeed(
  wallet.publicKey,
  GREETING_SEED,
  programId
);

const currentAccountInfo = await connection.getAccountInfo(
  greetedPubkey,
  "confirmed"
);
```

If we do not have created the program account yet, we will set the counter to zero. Otherwise, we will use the `counterFromAccountInfo` to get the counter.

```jsx
if (currentAccountInfo === null) {
  setCounter(0);
} else {
  setCounter(counterFromAccountInfo(currentAccountInfo));
}
```

One time job is done. However, we want to change the counter every time you or someone else greets you. So we use `connection.onAccountChange` to add a listener.

```jsx
connection.onAccountChange(
  greetedPubkey,
  (accountInfo, _) => {
    setCounter(counterFromAccountInfo(accountInfo));
  },
  "confirmed"
);
```

So, the final `useEffect` should look like this -

```jsx
useEffect(() => {
  async function addListener() {
    if (wallet.publicKey) {
      const greetedPubkey = await PublicKey.createWithSeed(
        wallet.publicKey,
        GREETING_SEED,
        programId
      );
      const currentAccountInfo = await connection.getAccountInfo(
        greetedPubkey,
        "confirmed"
      );
      if (currentAccountInfo === null) {
        setCounter(0);
      } else {
        setCounter(counterFromAccountInfo(currentAccountInfo));
      }
      connection.onAccountChange(
        greetedPubkey,
        (accountInfo, _) => {
          setCounter(counterFromAccountInfo(accountInfo));
        },
        "confirmed"
      );
    }
  }
  addListener();
}, [connection, wallet.publicKey]);
```

- Second thing that we want to implement is to greet ourselves when we click the button. For this, we create a new callback and pass it as the onClick for the button.

```jsx
const greet = useCallback(async () => {
  // code goes here ..
});
...
<Button onClick={greet}>Greet Yourself</Button>
```

The logic for this should be pretty straightforward:

1. Generate program account public key using `PublicKey.createWithSeed`, if the account does not exist, pay for creating the account with the required storage space.
2. Send the program the public key to greet (in this case, our own program key).

For the first part, the code is given below. See that we create a Transaction and add an `Instruction` to the `SystemProgram`, which tells the `SystemProgram` to create a new account with the correct required space. This should help you realize that creating a new account is a Solana Program in itself!

```jsx
const greetedPubkey = await PublicKey.createWithSeed(
  wallet.publicKey,
  GREETING_SEED,
  programId
);

const greetedAccount = await connection.getAccountInfo(greetedPubkey);
if (greetedAccount === null) {
  const lamports = await connection.getMinimumBalanceForRentExemption(
    GREETING_SIZE
  );

  const transaction = new Transaction().add(
    SystemProgram.createAccountWithSeed({
      fromPubkey: wallet.publicKey,
      basePubkey: wallet.publicKey,
      seed: GREETING_SEED,
      newAccountPubkey: greetedPubkey,
      lamports,
      space: GREETING_SIZE,
      programId,
    })
  );

  const signature = await wallet.sendTransaction(transaction, connection);
  await connection.confirmTransaction(signature, "processed");
}
```

For the second part of the logic, we send an instruction to our deployed program and send it the key of the generated program key. As our program only accepts a single type of instruction, that is "greet", and it does not need any arguments, we do not need to send any data.

```jsx
const instruction = new TransactionInstruction({
  keys: [{ pubkey: greetedPubkey, isSigner: false, isWritable: true }],
  programId,
  data: Buffer.alloc(0),
});

const signature = await wallet.sendTransaction(
  new Transaction().add(instruction),
  connection
);

await connection.confirmTransaction(signature, "processed");
```

Finally, your callback should look like this -

```jsx
const greet = useCallback(async () => {
  const greetedPubkey = await PublicKey.createWithSeed(
    wallet.publicKey,
    GREETING_SEED,
    programId
  );

  const greetedAccount = await connection.getAccountInfo(greetedPubkey);
  if (greetedAccount === null) {
    const lamports = await connection.getMinimumBalanceForRentExemption(
      GREETING_SIZE
    );

    const transaction = new Transaction().add(
      SystemProgram.createAccountWithSeed({
        fromPubkey: wallet.publicKey,
        basePubkey: wallet.publicKey,
        seed: GREETING_SEED,
        newAccountPubkey: greetedPubkey,
        lamports,
        space: GREETING_SIZE,
        programId,
      })
    );

    const signature = await wallet.sendTransaction(transaction, connection);
    await connection.confirmTransaction(signature, "processed");
  }

  const instruction = new TransactionInstruction({
    keys: [{ pubkey: greetedPubkey, isSigner: false, isWritable: true }],
    programId,
    data: Buffer.alloc(0),
  });

  const signature = await wallet.sendTransaction(
    new Transaction().add(instruction),
    connection
  );

  await connection.confirmTransaction(signature, "processed");
}, [connection, wallet]);
```

Yay! You can now greet yourself! ... Doesn't seem exciting? Let us greet others in the next section :). Code till now is available in the commit [`4d5cad9`](https://github.com/kanav99/solana-boilerplate/tree/4d5cad9f3bfe4a49754de5aa6077b89f5223603e).

Possible improvements - As `PublicKey.createWithSeed` might be an expensive operation, try memoizing using `useMemo`? Set the button in loading state while the transaction is being sent?

# Greeting others

Now, we might want to greet others as greeting ourselves is not that cool. For greeting others, we will need a public key for that account; then we can send them a greeting. Our goal of the section is to send greetings to others by adding a textbox to our application where you can write the public key of your friend and then click a button to send a greeting their way.

First, as we will reuse the same logic from the previous code, we should move the greeting code into a function that accepts a public key (the one to greet) created by `useCallback` so that we can reuse it. What I did was - Added an argument, which is the base58 of the public key of the recipient, to the old `greet` callback and replaced `wallet.publicKey` to `PublicKey(<argument>)`. Now we can create a separate callback called `greetYourself` where we send our own `wallet.publicKey.toBase58()` and pass this callback to the "Greet Yourself" button.

```jsx
const greet = useCallback(
  async (publicKey) => {
    const recipient = new PublicKey(publicKey);
    const greetedPubkey = await PublicKey.createWithSeed(
      recipient,
      GREETING_SEED,
      programId
    );

    const greetedAccount = await connection.getAccountInfo(greetedPubkey);
    if (greetedAccount === null) {
      const lamports = await connection.getMinimumBalanceForRentExemption(
        GREETING_SIZE
      );

      const transaction = new Transaction().add(
        SystemProgram.createAccountWithSeed({
          fromPubkey: recipient,
          basePubkey: recipient,
          seed: GREETING_SEED,
          newAccountPubkey: greetedPubkey,
          lamports,
          space: GREETING_SIZE,
          programId,
        })
      );

      const signature = await wallet.sendTransaction(transaction, connection);
      await connection.confirmTransaction(signature, "processed");
    }

    const instruction = new TransactionInstruction({
      keys: [{ pubkey: greetedPubkey, isSigner: false, isWritable: true }],
      programId,
      data: Buffer.alloc(0),
    });

    const signature = await wallet.sendTransaction(
      new Transaction().add(instruction),
      connection
    );

    await connection.confirmTransaction(signature, "processed");
  },
  [connection, wallet]
);

const greetYourself = useCallback(async () => {
  await greet(wallet.publicKey.toBase58());
}, [greet, wallet.publicKey]);
```

Now what remains is the ReactJS code used to render the label and input:

```jsx
const [recipient, setRecipient] = useState("");
return (
  <VStack width="full">
    <HStack>
      <Text>Total greetings: {counter === null ? "Loading.." : counter}</Text>
      <Button onClick={greetYourself}>Greet Yourself</Button>
    </HStack>
    <HStack>
      <Text width="full">Greet Someone: </Text>
      <Input
        value={recipient}
        onChange={(e) => setRecipient(e.target.value)}
      ></Input>
      <Button
        onClick={() => {
          greet(recipient);
        }}
      >
        Greet
      </Button>
    </HStack>
  </VStack>
);
```

Code till here is present in the commit [`746ff70`](https://github.com/kanav99/solana-boilerplate/tree/746ff7095bb45af822fa3be5bd999a3d1ce29dca).

# Organizing things

The logic is good enough now, but the page looks like everything is crammed into a single page. Let us clean this up. We will use Chakra UI tabs on the homepage to split the page into two pages - Home and Transaction History. So, in the Home, we change the rendering code to

```jsx
return (
  <Box textAlign="center" fontSize="xl">
    <Grid minH="100vh" p={3}>
      <Tabs variant="soft-rounded" colorScheme="green">
        <TabList width="full">
          <HStack justify="space-between" width="full">
            <HStack>
              <Tab>Home</Tab>
              <Tab>Transaction History</Tab>
            </HStack>
            <ColorModeSwitcher justifySelf="flex-end" />
          </HStack>
        </TabList>
        <TabPanels>
          <TabPanel>
            {publicKey && (
              <VStack spacing={8}>
                <Text>Wallet Public Key: {publicKey.toBase58()}</Text>
                <Text>
                  Balance:{" "}
                  {account
                    ? account.lamports / web3.LAMPORTS_PER_SOL + " SOL"
                    : "Loading.."}
                </Text>
                <Button onClick={getAirdrop} isLoading={airdropProcessing}>
                  Get Airdrop of 1 SOL
                </Button>
                <Greet />
              </VStack>
            )}
            {!publicKey && <WalletMultiButton />}
          </TabPanel>
          <TabPanel>
            {publicKey && (
              <VStack spacing={8}>
                <Heading>Transactions</Heading>
                {transactions && (
                  <VStack>
                    {transactions.map((v, i, arr) => (
                      <HStack key={"transaction-" + i}>
                        <Text>Signature: </Text>
                        <Code>{v.signature}</Code>
                      </HStack>
                    ))}
                  </VStack>
                )}
              </VStack>
            )}
            {!publicKey && <WalletMultiButton />}
          </TabPanel>
        </TabPanels>
      </Tabs>
    </Grid>
  </Box>
);
```

See how we have split the code into two panels - the second one contains transaction history, and the first one contains the rest of the code. We shifted the colour mode switcher to the end of the tablist. We can add a wallet disconnect button there as well, if the wallet is connected.

```jsx
<HStack justify="space-between" width="full">
  <HStack>
    <Tab>Home</Tab>
    <Tab>Transaction History</Tab>
  </HStack>
  <HStack>
    {publicKey && <WalletDisconnectButton bg="green" />}
    <ColorModeSwitcher justifySelf="flex-end" />
  </HStack>
</HStack>
```

In case the wallet is not connected, instead of just throwing the connect button, let us create a separate component that looks better.

```jsx
function WalletNotConnected() {
  return (
    <VStack height="70vh" justify="space-around">
      <VStack>
        <Text fontSize="2xl">
          {" "}
          Looks like your wallet is not connnected. Connect a wallet to get started!
        </Text>
        <WalletMultiButton />
      </VStack>
    </VStack>
  );
}
```

Let us use `SimpleGrid` to separate the wallet properties and greeting code and use read-only form input to show data beautifully.

```jsx
<SimpleGrid columns={2} spacing={10}>
  <VStack spacing={8} borderRadius={10} borderWidth={2} p={10}>
    <FormControl id="pubkey">
      <FormLabel>Wallet Public Key</FormLabel>
      <Input type="text" value={publicKey.toBase58()} readOnly />
    </FormControl>
    <FormControl id="balance">
      <FormLabel>Balance</FormLabel>
      <Input
        type="text"
        value={
          account
            ? account.lamports / web3.LAMPORTS_PER_SOL + " SOL"
            : "Loading.."
        }
        readOnly
      />
    </FormControl>
    <Button onClick={getAirdrop} isLoading={airdropProcessing}>
      Get Airdrop of 1 SOL
    </Button>
  </VStack>
  <VStack>
    <Greet />
  </VStack>
</SimpleGrid>
```

After this change, do similar changes to the `Greet` component.

```jsx
return (
  <>
    <VStack width="full" spacing={8} borderRadius={10} borderWidth={2} p={10}>
      <FormControl id="greetings">
        <FormLabel>No. of greetings recieved</FormLabel>
        <Input
          type="text"
          value={counter === null ? "Loading.." : counter}
          readOnly
        />
      </FormControl>
      <HStack>
        <Button onClick={greetYourself}>Greet Yourself</Button>
      </HStack>
    </VStack>
    <VStack width="full" spacing={8} borderRadius={10} borderWidth={2} p={10}>
      <FormControl id="send">
        <FormLabel>Send greeting to public key</FormLabel>
        <Input
          value={recipient}
          onChange={(e) => setRecipient(e.target.value)}
        ></Input>
      </FormControl>
      <Button
        onClick={() => {
          greet(recipient);
        }}
      >
        Greet
      </Button>
    </VStack>
  </>
);
```

The code till here is available in the commit [`e56e2d5`](https://github.com/kanav99/solana-boilerplate/tree/e56e2d5ee69df21e9a3cfd4f2e6dcc2dd42436e5), or just the current master branch.

# Conclusion

Congratulations on completing this tutorial! What we developed and understood was how we can use Chakra UI and Solana to make beautiful and fast dApps. Now, you should be able to use this knowledge to power your big idea and easily use Solana to make blazing fast applications while using the ease of Chakra to develop the frontend with confidence. There is still much that can be improved in this simple application, but I will leave that to the imagination of the reader.

# About the author

This tutorial was created by Kanav Gupta. He can be found on [Github](https://github.com/kanav99) and on their [website](https://kanavgupta.xyz).

# References

- [Solana example-helloworld repository](https://github.com/solana-labs/example-helloworld)
- [Chakra UI Documentation](https://chakra-ui.com/docs/getting-started)
