# Introduction

In this tutorial, we'll create a simple DEX (Decentralized Exchange) with which we'll understand how to code a DEX and how a DEX works in general. 

This DEX will be using the Constant Product Market-Making algorithm, and it will be based on the v1 of Uniswap. If you want more information about the Constant Product Market-Making algorithm and v1 of Uniswap you can read this [white paper](https://hackmd.io/@HaydenAdams/HJ9jLsfTz). You can also refer to this [video](https://www.youtube.com/watch?v=-6HefvM5vBg) to get an understanding of the CPMM algorithm. Before we start to write extensive code, you can find the full source code from this repository: https://github.com/vivekascoder/tez-dex


# Prerequisites

Basic knowledge of SmartPy, React, Taquito and smart contracts.

# Requirements
- [NodeJs](https://nodejs.org/en/) (v14.18.1 or higher)
- [Yarn](https://yarnpkg.com/)
- [Temple Wallet](https://templewallet.com/)


# Project Setup

Create an empty React project using `create-react-app`, after creating it install the following packages.

> NOTE: For this tutorial, I'll be using `tailwindcss` for styling the components, but this step is completely optional. You can use any UI Library/Css Framework that you want.

1. Taquito and BeconWallet

```text
yarn add @taquito/taquito @taquito/beacon-wallet 
```

2. Tailwindcss (Optional)

```text
yarn add --dev tailwindcss@npm:@tailwindcss/postcss7-compat postcss@^7 autoprefixer@^9
```

For more information on setting up tailwindcss with `create-react-app` you can read [here](https://tailwindcss.com/docs/guides/create-react-app).

After installing the dependencies make a folder called `./contracts` in the root directory where we'll keep all our smartpy code.

Deploy two FA1.2 tokens namely **🐱 Cat Token** and **LP Token** which we'll be using later in this tutorial.


# Working on the Smart Contract

## Add dex.py
We'll start working on our smart contract by creating a file called `dex.py` in the `contracts` directory where we'll keep all the code related to our DEX functionality.

Create a demo FA1.2 contract called `Token` which we'll use only for testing:

```py
import smartpy as sp
fa12 = sp.io.import_script_from_url("https://smartpy.io/templates/FA1.2.py")


class Token(fa12.FA12):
    """ Test FA1.2 Token """
    pass
```

Create another Contract in the same file and call it `Dex`, it is going to be our main DEX contract class.

```py
class Dex(sp.Contract):
    def __init__(self, _token_address, _lp_address):
        self.init(
            invariant=sp.nat(0),
            tez_pool=sp.mutez(0),
            token_pool=sp.nat(0),
            token_address=_token_address,
            lp_address=_lp_address,
        )
```

Here the storage in the `Dex` contract consist of 5 things where invariant denotes the invariant of the pool, it is equal to the product of the `tez_pool` and `token_pool`. The `tez_pool` and `token_pool` keeps track of the quantity of xtz and Cat token in our pool which will used to determine the ratio while exchanging tokens. Here `lp_address` and `token_address` keeps track of the address of our LP token and Cat Token which while be used while doing inter contract calls.


We'll now create some utility function that will be used to do the transfer, namely `transfer_tokens` which will be used to transfer a certain amount of tokens from one address to another with the help of inter contract calls and `transfer_tez` which is just used to transfer a certain amount of xtz from the pool to a specific address,

```py
def transfer_tokens(self, from_, to, amount):
    """ Utility Function to transfer FA1.2 Tokens."""
    sp.verify(amount > sp.nat(0), 'INSUFFICIENT_TOKENS[transfer_token]')
    transfer_type = sp.TRecord(
        from_=sp.TAddress,
        to_=sp.TAddress,
        value=sp.TNat
    ).layout(("from_ as from", ("to_ as to", "value")))
    transfer_data = sp.record(from_=from_, to_=to, value=amount)
    token_contract = sp.contract(
        transfer_type,
        self.data.token_address,
        "transfer"
    ).open_some()
    sp.transfer(transfer_data, sp.mutez(0), token_contract)

def transfer_tez(self, to_, amount: sp.TMutez):
    """ Utility function to transfer tezos. """
    sp.send(to_, amount, message="SENDING_TEZ")

```

While allowing other people to provide liquidity to our pool, we'll have to give them a certain amount of **LP Tokens** based on their liquidity, for that we'll have to mint LP tokens and burn them when the user wants their liquidity back. For this, we'll have two more utility functions namely `mint_lp`, and `burn_lp` which are basically used to mint and burn LP tokens.

```py
def mint_lp(self, amount):
    """Mint `amount` LP tokens to `sp.sender` account."""
    transfet_type = sp.TRecord(
        address=sp.TAddress,
        value=sp.TNat,
    )
    transfer_value = sp.record(
        address=sp.sender,
        value=amount,
    )
    contract = sp.contract(
        transfet_type,
        self.data.lp_address,
        'mint'
    ).open_some()
    sp.transfer(transfer_value, sp.mutez(0), contract)

def burn_lp(self, amount):
    """ Burn `amount` LP Tokens. """
    transfet_type = sp.TRecord(
        address=sp.TAddress,
        value=sp.TNat,
    )
    transfer_value = sp.record(
        address=sp.sender,
        value=amount,
    )
    contract = sp.contract(
        transfet_type,
        self.data.lp_address,
        'burn'
    ).open_some()
    sp.transfer(transfer_value, sp.mutez(0), contract)
```

The `mint_lp` method takes the amount of tokens to mint and then it does an inter contract call to out LP Token Contract to the `entry_point` called `mint` to mint LP tokens. In the same way, the `burn_lp` method takes the amount of LP Tokens to burn.

> NOTE: Please note that here our DEX contract is the admin of LP Token contract and that's why it can mint and burn the tokens.


## Initialize Exchange
Now we'll work on an entry point called `initialize_exchange` that will be responsible for providing the initial liquidity in the pool and govern the initial ratio between the token and xtz which will we used to govern the prices.

```py
@sp.entry_point
def initialize_enchange(self, token_amount):
    sp.if ((self.data.token_pool == sp.nat(0)) & (self.data.tez_pool == sp.mutez(0))):
        tez_amount = sp.amount
        sp.verify(token_amount > sp.nat(10), message="NOT_ENOUGH_TOKEN")
        sp.verify((tez_amount > sp.mutez(1)), message="NOT_EMOUGH_TEZ")

        self.data.tez_pool = tez_amount
        self.data.token_pool = token_amount
        self.data.invariant = token_amount * \
            sp.utils.mutez_to_nat(tez_amount)

        self.transfer_tokens(
            from_=sp.sender, 
            to=sp.self_address,
            amount=token_amount
        )
    sp.else:
        sp.failwith('ALREADY_INITIALIZED')
```

This entry point first checks whether the total quantity of tez and token is zero or not, which will limit users to only be able to call it after the initial liquidity is provided.

Then we check whether the token amount and xtz amount is more than 0, and then we update the `tez_pool` and `token_pool` along with the invariant. Then we transfer the tokens from the user to our contract using our utility function `transfer_tokens`.


## Tez to Token Swap

Now we'll code the most interesting part of our dex which is the exchange tez into tokens and vice versa. We'll start with tez to token swap, we'll create an entry point called `tez_to_token` which will exchange the amount of xtz sent i.e `sp.sender` and exchange it to Cat tokens based on the current exchange ratio.

In the CPMM algorithm, the invariant needs to be constant, which means `x.y=k` where x and y represent the amount of xtz and tokens respectively. Now if you want to swap xtz into the token, then the amount of new xtz will increase let's call it `x1`. 

And let's call the new token amount as `y1`, so according to the equation `x1.y1=k`.

This means the new amount of tokens in the pool will be `y1 = k / x1`. This is how we are calculating the `new_token_pool`.

Now, the amount of tokens that the user will receive for exchanging his `x` tez (x is in mutez) would be `token_out = y - y1`. This is how we are calculating `token_out`.

> NOTE: Please note that after exchanging certain xtz into tokens, the amount of xtz in the pool will increase and the amount of tokens will decrease and that's why while calculating the amount of tokens that the user will receive we're doing `y - y1` and not `y1 - y`.

After calculating the `tokens_out`, we're updating the new `tez_pool` and `token_pool` along with the new invariant. After that, we're sending `token_out` amount of tokens to the user using our utility function called `transfer_tokens`.

```py
@sp.entry_point
def tez_to_token(self):
    sp.verify(sp.amount > sp.mutez(0), message="NOT_ENOUGH_TEZ")
    tez_in_nat = sp.utils.mutez_to_nat(sp.amount) 
    new_tez_pool = self.data.tez_pool + sp.amount
    new_token_pool = sp.local('new_token_pool', sp.nat(0))
    new_token_pool.value = self.data.invariant / sp.utils.mutez_to_nat(new_tez_pool) 
    token_out = sp.local('token_out', sp.as_nat(self.data.token_pool - new_token_pool.value))
    self.data.tez_pool = new_tez_pool
    self.data.token_pool = new_token_pool.value
    self.data.invariant = self.data.token_pool * sp.utils.mutez_to_nat(self.data.tez_pool)

    sp.if token_out.value > sp.nat(0):
        self.transfer_tokens(
            from_=sp.self_address,
            to=sp.sender, 
            amount=token_out.value
        )
```



## Token to Tez Swap
The process of exchanging tokens into tez is also quite similar to what we've done in our tez to token swap.

```py
@sp.entry_point
def token_to_tez(self, params):
    sp.set_type(params, sp.TRecord(token_amount=sp.TNat))
    sp.verify(params.token_amount > sp.nat(0), message="NOT_ENOUGH_TOKEN")
    new_token_pool = self.data.token_pool + params.token_amount
    new_tez_pool = self.data.invariant / new_token_pool
    self.data.tez_pool = sp.utils.nat_to_mutez(new_tez_pool)
    self.data.token_pool = new_token_pool
    self.data.invariant = new_token_pool * new_tez_pool

    self.transfer_tokens(
        from_=sp.sender, 
        to=sp.self_address, 
        amount=params.token_amount
    )

    tez_out = self.data.tez_pool - sp.utils.nat_to_mutez(new_tez_pool)
    self.transfer_tez(to_=sp.sender, amount=tez_out)
```

The `self.transfer_tokens` function will transfer `token_amount` to this contract's address. `self.transfer_tez`  will transfer the swapped tezos.

We can calculate the `new_token_pool` by adding the amount of token which is sent to the contract by the user to the previous amount of token in the pool. We're calculating the new amount of xtz by dividing the invariant with the `new_token_amount`.

And then we're updating the values in the pool.

```py
new_token_pool = self.data.token_pool + params.token_amount
new_tez_pool = self.data.invariant / new_token_pool
self.data.tez_pool = sp.utils.nat_to_mutez(new_tez_pool)
self.data.token_pool = new_token_pool
self.data.invariant = new_token_pool * new_tez_pool
```

After this, we're transferring the amount of tokens i.e `params.token_amount` from the sender to our contract address. After that, we're calculating the `tez_out` which represents the amount of xtz (in mutez) which the user will receive. 

Finally, we're actually transfering `tez_out` xtz from our contract to the sender.

```py
self.transfer_tokens(
    from_=sp.sender, 
    to=sp.self_address, 
    amount=params.token_amount
)

tez_out = self.data.tez_pool - sp.utils.nat_to_mutez(new_tez_pool)
self.transfer_tez(to_=sp.sender, amount=tez_out)
```


## Invest Liquidity
We're done with exchanging the tez into tokens and vice versa, but we need to add a way in our contract with which we can allow other people to add new liquidity to the pool, and for this, we're going to add two entry points namely `invest_liquidty` and `divest_liquidity` which will allow users to add new liquidity to the pool and remove their liquidity from the pool respectively.

Before diving into the code of `invest_liquidity`, let's see what are the things that it needs to do.
1. Check if the xtz sent in the transaction while calling the entry point is more than 0.
2. Calculate the total liquidity of the pool.
3. Calculate the token amount that we'll take which is determined based on the current exchange ratio and amount of xtz sent.
4. Calculate the amount of LP tokens that we'll have to mint for the user based on their liquidity.
5. Mint the required LP Tokens.
6. Update the contract's storage.

```py
@sp.entry_point
def invest_liquidity(self):
    sp.verify(sp.amount > sp.mutez(0), message="NOT_ENOUGH_TOKEN")

    total_liquidity = sp.utils.mutez_to_nat(self.data.tez_pool) + self.data.token_pool
    token_amount = sp.utils.mutez_to_nat(sp.amount) * self.data.token_pool / sp.utils.mutez_to_nat(self.data.tez_pool)
    liquidity_minted = sp.utils.mutez_to_nat(sp.amount) * total_liquidity / sp.utils.mutez_to_nat(self.data.tez_pool)

    self.mint_lp(liquidity_minted)
    
    self.data.tez_pool += sp.amount
    self.data.token_pool += token_amount
    self.data.invariant = sp.utils.mutez_to_nat(self.data.tez_pool) * self.data.token_pool

    self.transfer_tokens(
        from_=sp.sender, 
        to=sp.self_address, 
        amount=token_amount
    )
```

First of all, we're calculating the `total_liquidity` which is the sum of the current value of `tez_pool` and `token_pool`. Then we're calculating the amount of tokens that we'll need along with xtz sent based on the current ratio between token and xtz.

```py
sp.verify(sp.amount > sp.mutez(0), message="NOT_ENOUGH_TOKEN")

total_liquidity = sp.utils.mutez_to_nat(self.data.tez_pool) + self.data.token_pool
token_amount = sp.utils.mutez_to_nat(sp.amount) * self.data.token_pool / sp.utils.mutez_to_nat(self.data.tez_pool)
```

Then we're calculating the amount of liquidity that we'll have to mint for the liquidity provider which depends on the amount of liquidity sent to the pool.

After that, we'll be minting LP tokens with our utility function `mint_lp` and then we're updating the contract's storage. 

Finally we're transferring the amount of tokens needed from the sender's address to our contract's address.

```py
liquidity_minted = sp.utils.mutez_to_nat(sp.amount) * total_liquidity / sp.utils.mutez_to_nat(self.data.tez_pool)

self.mint_lp(liquidity_minted)

self.data.tez_pool += sp.amount
self.data.token_pool += token_amount
self.data.invariant = sp.utils.mutez_to_nat(self.data.tez_pool) * self.data.token_pool

self.transfer_tokens(
    from_=sp.sender, 
    to=sp.self_address, 
    amount=token_amount
)
```

## Divest Liquidity
Now, users can invest liquidity into our pool, so let's work on `divest_liquidity` which will allow liquidity providers to get back their liquidity.

Our `divest_liquidity` entry point takes only one parameter which is the amount of LP tokens that the liquidity provider wants to burn, based on this amount we'll give them their liquidity back.

```py
@sp.entry_point
def divest_liquidity(self, lp_amount: sp.TNat):
    """ Burn LP and give back the liquidity """
    sp.verify(lp_amount > sp.nat(0), 'INVALID_AMOUNT')
    total_liquidity = sp.utils.mutez_to_nat(self.data.tez_pool) + self.data.token_pool

    tez_out = sp.utils.mutez_to_nat(self.data.tez_pool) * lp_amount / total_liquidity
    token_out = self.data.token_pool * lp_amount / total_liquidity

    self.data.tez_pool = self.data.tez_pool - sp.utils.nat_to_mutez(token_out)
    self.data.token_pool = sp.as_nat(self.data.token_pool - token_out)
    self.data.invariant = self.data.token_pool * sp.utils.mutez_to_nat(self.data.tez_pool)
    self.burn_lp(lp_amount)

    sp.if tez_out > sp.nat(0):
        self.transfer_tez(to_=sp.sender, amount=sp.utils.nat_to_mutez(tez_out))
    
    sp.if token_out > sp.nat(0):
        self.transfer_tokens(
            from_=sp.self_address,
            to=sp.sender, 
            amount=token_out
        )
    sp.else:
        sp.failwith('NOT_ENOUGH_TOKENS')
```

We'll start by verifying the amount of LP tokens (needs to be more than 0). Then we're calculating the total_liquidity of the pool.

Now based on the amount of LP tokens you are burning we're calculating the `tez_out` and `token_out` which represents the amount of xtz and tokens the liquidity provider will get back.

```py
sp.verify(lp_amount > sp.nat(0), 'INVALID_AMOUNT')
total_liquidity = sp.utils.mutez_to_nat(self.data.tez_pool) + self.data.token_pool

tez_out = sp.utils.mutez_to_nat(self.data.tez_pool) * lp_amount / total_liquidity
token_out = self.data.token_pool * lp_amount / total_liquidity
```


Now we are updating the contract's storage and using our utility functions to send back the liquidity.

```py
self.data.tez_pool = self.data.tez_pool - sp.utils.nat_to_mutez(token_out)
self.data.token_pool = sp.as_nat(self.data.token_pool - token_out)
self.data.invariant = self.data.token_pool * sp.utils.mutez_to_nat(self.data.tez_pool)
self.burn_lp(lp_amount)

sp.if tez_out > sp.nat(0):
    self.transfer_tez(to_=sp.sender, amount=sp.utils.nat_to_mutez(tez_out))

sp.if token_out > sp.nat(0):
    self.transfer_tokens(
        from_=sp.self_address,
        to=sp.sender, 
        amount=token_out
    )
sp.else:
    sp.failwith('NOT_ENOUGH_TOKENS')
```

So far we've coded all the required entry points. Now let's move to the frontend. For getting full source code, check this repository: https://github.com/vivekascoder/tez-dex


# Working on the Frontend

We'll start by importing the necessary things in the `App.jsx` file.

> NOTE: Just to keep things simple I'm writing everything in `App.jsx` but you can
> structure your code however you like.

Let's import all the necessary things that we might need in our project. Here we're using `axios` to make use of **tzkt API** to fetch balances of the users.

```jsx
import React, { useState, useEffect } from "react";
import "./index.css";

import { TezosToolkit } from "@taquito/taquito";
import { BeaconWallet } from "@taquito/beacon-wallet";

import CONFIG from "./config.js";
import axios from "axios";
```

We've a file here called `config.js` which contains the addresses and other information about our contracts.

```jsx
const config = {
  rpcUrl: "https://granadanet.smartpy.io",
  tokenAddress: "KT1HrjGaoTmdZ8Znbup6bgV2gEpFF8tt9jo5",
  tokenDecimals: 10 ** 6,
  tokenBalanceBigMapId: 87342,
  dexAddress: "KT1FFoWsyFH7rJGxogvPRRuu8MwskY6Asags",
  lpTokenAddress: "KT1AEd7ZCSdpZhNNnEwZU9sNFSkib2s7e61f",
  lpDecimals: 10 ** 9,
  lpBalanceBigMapId: 87352,
  preferredNetwork: "granadanet",
};

export default config;
```

We'll create two separate JSX Components in the same `App.jsx` file namely `Notification` which will be responsible for showing any message and `Balance` which shows the current amount of **Cat Token** and **LP Token** a user has.

```jsx
function Notification({ error, setError }) {
  return (
    <div
      className={`fixed bottom-2 left-2 right-2 p-4 text-sm bg-purple-500 text-white shadow-md z-50 
        ${error ? "block" : "hidden"}`}
    >
      <div className="flex items-center space-x-4">
        <button
          onClick={() => {
            setError("");
          }}
        >
          ❎
        </button>
        <p className="flex-1">{error}</p>
      </div>
    </div>
  );
}

function Balances({catToken, lpToken}) {
  return (
    <div className="bg-gray-100 shadow-sm flex items-center justify-center p-4 mb-20 space-x-10">
      <span className="font-semibold text-sm">🐱 Cat Token: {catToken}</span>
      <span className="font-semibold text-sm">💦 LP Token: {lpToken}</span>
    </div>
  );
}
```

Now, let's start our main `App` component and create some state variables using useState to make certain state within our application. 

1. `tezos`: Keeps track of `TezosToolkit` object.
2. `wallet`: Keeps track of the address of the user.
3. `error`: Keeps track of an error message which will be displayed in the `Notification` component.
4. `swapXtzAmount`: Keeps track of the input box that represents xtz amount to send while swapping.
5. `swapTokenAmount`:  Keeps track of the input box that represents token amount to send while swapping.
6. `liquidityXtz`: Keeps track of the amount of xtz that needs to be sent while adding liquidity to the pool.
7. `lpToBurn`: Keeps track of the amount of LP tokens that needs to be burn while calling `divest_liquidity`.
8. `lpBalance`: Keeps track of current user's LP Token balance.
9. `catBalance`: Keeps track of the amount of Cat Tokens that the user has.

```jsx
function App() {
  const [tezos, setTezos] = useState(null);
  const [wallet, setWallet] = useState(null);
  const [error, setError] = useState("");

  // For the input boxes.
  const [swapXtzAmount, setSwapXtzAmount] = useState(0);
  const [swapTokenAmount, setSwapTokenAmount] = useState(0);
  const [liquidityXtz, setLiquidityXtz] = useState(0);
  const [lpToBurn, setLpToBurn] = useState(0);

  // For showing the balnces.
  const [lpBalance, setLpBalance] = useState(0);
  const [catBalance, setCatBalance] = useState(0);

  return (<div></div>);
}
```

We'll create a utility function that can fetch the token balance of a user with the help of `tzkt` API.

```jsx
async function getBalance(userAddress, bigmapId) {
  const {data} = await axios.get(
    `https://api.granadanet.tzkt.io/v1/bigmaps/${bigmapId}/keys`
  );
  const requiredEl = data.find((el) => {
    return el.key === userAddress;
  })
  if (requiredEl) {
    return parseInt(requiredEl.value.balance);
  } else {
    return 0;
  }
}
```

We'll have one function to update the balance, so that whenever we want to update the balance, we can directly do it just by calling this method.

```jsx
async function updateBalances() {
if (tezos) {
    const cat = await getBalance(
    wallet,
    CONFIG.tokenBalanceBigMapId
    );
    setCatBalance(cat);

    const lp = await getBalance(
    wallet,
    CONFIG.lpBalanceBigMapId
    );
    setLpBalance(lp);
    console.log({cat, lp})
}
}
```

Let's make use of the `useEffect` hook to update the balance whenever `wallet` changes.

```jsx
useEffect( () => {
  async function runUseEffect () {
    await updateBalances()
    console.log('updates')
  }
  runUseEffect();
}, [wallet])
```

Let's make a function to connect to the **Temple Wallet**.

```jsx
async function connectToWallet() {
  if (!tezos) {
    const t = new TezosToolkit(CONFIG.rpcUrl);
    const wallet = new BeaconWallet({
      name: "Tez Dex",
      preferredNetwork: CONFIG.preferredNetwork,
    });
    await wallet.requestPermissions({
      network: { type: CONFIG.preferredNetwork },
    });
    t.setWalletProvider(wallet);
    const pkh = await t.wallet.pkh();
    setTezos(t);
    setWallet(pkh);
  } else {
    setError("The wallet is already connected.");
  }
}
```

When the user will click on **Swap** button we'll fire this method which will call the `tez_to_token` entry point.

```jsx
async function exchange() {
  try {
    const dexContract = await tezos.wallet.at(CONFIG.dexAddress);
    if (swapXtzAmount > 0) {
      // Swap Xtz -> Token
      console.log('Tez -> Token')
      const xtz = parseInt(swapXtzAmount * 10 ** 6);
      
      // Interacting with the entry_point
      const op = await dexContract.methods.tez_to_token().send({amount: xtz, mutez: true});
      setError(`Operation Hash: ${op.opHash}`)
      const result = await op.confirmation();
      console.log(result);
    } 
    else if(swapTokenAmount > 0) {
      console.log('Token-> Tez')
      // Swap Token -> Xtz
      const catAmount = parseInt(swapTokenAmount * CONFIG.tokenDecimals);
      const tokenContract = await tezos.wallet.at(CONFIG.tokenAddress);
      const batch = await tezos.wallet.batch()
      .withContractCall(
        tokenContract.methods.approve(
          CONFIG.dexAddress,
          catAmount,
        )
      )
      .withContractCall(
        dexContract.methods.token_to_tez(catAmount)
      )
      const batchOp = await batch.send();
      console.log("Operation hash:", batchOp.hash);
      setError(`Operation Hash: ${batchOp.hash}`)

      await updateBalances();
    } 
    else {
      setError(`Not a valid Value.`)
    }
  } catch(err) {
    setError(err.message)
  }
}
```

In the same way, when the user will click on the **Add** button of addLiquidity form we'll fire this method which will call the `invest_liquidity` entry point.

Another cool thing to note here is that we're also calculating the amount of Cat tokens that needs to be approved based on the amount of XTZ sent and the current exchange ratio.

```jsx
async function addLiquidity() {
  // Add the liquidity into the dex.
  const dexContract = await tezos.wallet.at(CONFIG.dexAddress);
  const tokenContract = await tezos.wallet.at(CONFIG.tokenAddress);
  
  const xtz = parseInt(liquidityXtz * 10 ** 6);
  const storage = await dexContract.storage();
  const tezpool = storage['tez_pool'].toNumber();
  const tokenPool = storage['token_pool'].toNumber();
  const tokenNeeded = parseInt(xtz * tokenPool / tezpool);
  
  const op = await tokenContract.methods.approve(
    CONFIG.dexAddress,
    tokenNeeded
  ).send();
  setError(`Operation Hash: ${op.opHash}`)
  const result = await op.confirmation();
  console.log(result);

  // Interacting with the entry_point
  const anotherOp = await dexContract.methods.invest_liquidity().send({amount: xtz, mutez: true});
  setError(`Operation Hash: ${anotherOp.opHash}`)
  const anotherResult = await anotherOp.confirmation();
  console.log(anotherResult);

  await updateBalances();
}
```


Now, we'll create a method that will be fired when the user will click on **Remove** button and call the `divest_liquidity` entry point with the amount of LP Tokens that needs to be burn.

```jsx
async function removeLiquidity() {
  const lp = parseInt(lpToBurn * 10 ** 6);
  const dexContract = await tezos.wallet.at(CONFIG.dexAddress);

  // Remove the liquidity from the dex based on the amount of the LP Token burn.
  const op = await dexContract.methods.divest_liquidity(lp).send();
  setError(`Operation Hash: ${op.opHash}`)
  const result = await op.confirmation();
  console.log(result);
  
  await updateBalances();
}
```

Let's start coding the actual UI, with the help of some HTML and tailwindcss utility classes. Here you can see we're conditionally rendering the `Notification` component which will display a little popup if the `error` has some value.

After that, we're also rendering the `Balance` component.
```jsx
<div className="max-w-2xl mx-auto relative min-h-screen">
  {error ? <Notification error={error} setError={setError} /> : ""}
  <nav className="bg-gray-100 shadow-sm flex items-center justify-between p-4 mb-20">
    <h1 className="text-lg font-semibold">⚔️ Tez Dex</h1>
    <div className="flex space-x-3 items-center">
      <button className={btnClass} onClick={connectToWallet}>
        {wallet
          ? `${wallet.slice(0, 5)}...${wallet.slice(32, 36)}`
          : "💳️ Connect"}
      </button>
    </div>
  </nav>

  <Balances 
    catToken={catBalance / CONFIG.tokenDecimals}
    lpToken={lpBalance / CONFIG.lpDecimals}
  />
</div>

```

After that, let's start coding some forms for the different actions that we want to call, all the forms are similar to each other.

```jsx
return (
    <div className="max-w-2xl mx-auto relative min-h-screen">
      {error ? <Notification error={error} setError={setError} /> : ""}
      <nav className="bg-gray-100 shadow-sm flex items-center justify-between p-4 mb-20">
        <h1 className="text-lg font-semibold">⚔️ Tez Dex</h1>
        <div className="flex space-x-3 items-center">
          <button className={btnClass} onClick={connectToWallet}>
            {wallet
              ? `${wallet.slice(0, 5)}...${wallet.slice(32, 36)}`
              : "💳️ Connect"}
          </button>
        </div>
      </nav>

      <Balances 
        catToken={catBalance / CONFIG.tokenDecimals}
        lpToken={lpBalance / CONFIG.lpDecimals}
      />
      <div className="m-2 p-4 bg-gray-200">
        <p className="text-xs text-gray-500">⚔️ Tez Dex / Exchange</p>
        <form
          className="space-y-4 mt-4"
          onSubmit={(e) => {
            e.preventDefault();
            exchange();
          }}
        >
          <input
            type="number"
            placeholder="Amount of XTZ"
            name="tez"
            className="block text-sm w-full"
            value={swapXtzAmount}
            onChange={(e) => {setSwapXtzAmount(e.target.value);}}
          />
          <input
            type="number"
            placeholder="Amount of Token"
            name="token"
            className="block text-sm w-full"
            value={swapTokenAmount}
            onChange={(e) => {setSwapTokenAmount(e.target.value);}}
          />
          <button
            className={btnClass}
          >
            🔃 Swap
          </button>
        </form>
      </div>

      <div className="m-2 p-4 bg-gray-200">
        <p className="text-xs text-gray-500">⚔️ Tez Dex / Add Liquidity</p>
        <form className="space-y-4 mt-4" onSubmit={(e) => {e.preventDefault(); addLiquidity();}}>
          <div className="flex space-x-2">
            <input
              type="number"
              placeholder="Amount of XTZ"
              name="tez"
              className="text-sm w-full flex-1"
              value={liquidityXtz} 
              onChange={(e) => {setLiquidityXtz(e.target.value)}}
            />
            <button className={btnClass}>💦 Add</button>
          </div>
        </form>
      </div>

      <div className="m-2 p-4 bg-gray-200">
        <p className="text-xs text-gray-500">⚔️ Tez Dex / Remove Liquidity</p>
        <form className="space-y-4 mt-4" onSubmit={(e) => {e.preventDefault(); removeLiquidity();}}>
          <div className="flex space-x-2">
            <input
              type="number"
              placeholder="Amount of LP Tokens to burn"
              name="tez"
              className="text-sm w-full flex-1"
              value={lpToBurn} 
              onChange={(e) => {setLpToBurn(e.target.value)}}
            />
            <button className={btnClass}>🔥 Remove</button>
          </div>
        </form>
      </div>

      <p className="absolute bottom-2 right-2 text-xs font-semibold">
        Coded by{" "}
        <a href="https://github.com/vivekascoder" className="text-blue-500">
          @vivekascoder
        </a>
      </p>
    </div>
  );
```

You can start the React project with the command `yarn start`. Then open your web browser to `http://localhost:3000`, you'll see something like the following image:

![Tez Dex](https://i.imgur.com/QIElN7Z_d.webp?maxwidth=760&fidelity=grand)


## Swapping Tez into token.

Click on the Swap button after entering some amount in xtz let's say 1.

![Tez To Token](https://i.imgur.com/GUGspjM.png)

You can see after exchanging 1 tez we're getting around 97 cat token based on current ratio.
![Transaction](https://i.imgur.com/4lHckUS.png)


# About The Author
I am Vivek Kumar, you can find more about me [on my GitHub profile](https://github.com/vivekascoder)!

# References
1. [Uniswap v1 white paper](https://hackmd.io/@HaydenAdams/HJ9jLsfTz)
2. [Video Explaining DEX](https://hackmd.io/@HaydenAdams/HJ9jLsfTz)
