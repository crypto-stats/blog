---
date: 2022-08-26
title: Crunching the Calldata
tagline: The next step for Ethereum rollups to cut transaction costs
author: David Mihal
authorLink: https://davidmihal.com
---

# Crunching the Calldata

Since 2020, Ethereum's scaling roadmap has been oriented around "rollups": independent execution environments that use proofs (either zero-knowledge proofs or optimistic fraud proofs) to inherit the security of Ethereum.

After years of development, rollups have finally been deployed and are gaining adoption. The flagship [Arbitrum](https://arbitrum.io/) optimistic rollup has been live for nearly a year, with [over $2.7 billion worth of assets deposited into the bridge](https://l2beat.com/), while [Optimism](https://optimism.io/) tails closely behind. Application-specific zero-knowledge rollups like Loopring and dYdX have also seen substantial adoption, and many competing general-purpose zero-knowledge rollups are set to launch within the upcoming months.

Despite the rapid scaling advancements in the rollup space, some have expressed concerns that these fees are _still too high_.

<Tweet tweetId="1464261341921923074" options={{ conversation: 'none' }} />

Indeed, transaction fees on Arbitrum and Optimism are still substantially higher than "low-fee" chains like Solana and Polygon.

So what's holding these rollups back?

## Understanding rollup economics

To understand transaction fees, we first need to break down the various costs that a blockchain transaction incurs:

**Execution**

This is the cost required for all nodes in a network to execute a transaction and validate that the outcome is valid (eg: you actually own the tokens you're trying to transfer).

**Storage/state**

This is the cost to update a blockchain's "database" with new values (eg: after a token transfer, the sender's balance is decreased and recipient's balance is increased).

**Data availability**

In order to ensure blockchains remain trustless and verifiable by all, a blockchain must ensure that all relevant data about a transaction is publicly shared with all network participants.

Essentially, this is the assurance that everybody in the world can see your transaction. Without this assurance, various attacks are possible (known as data-withholding attacks).

As we'll see, data availability is one of the key bottlenecks of blockchains today...

### Rollups: shifting execution off-chain

The major advancements of rollups comes from the ability to move the execution and storage of a blockchain "off-chain", to a limited group of nodes. Instead of each Ethereum node in the network needing to execute all transactions and store every update, we can just delegate this task to the rollup operators.

_Wait, trusting a small group of operators? Isn't that centralized?_

Great question!

Rollups aim to inherit the same security of Ethereum using various proof types. Optimistic rollups allow for a single honest entity to submit a "fraud proof" and earn a reward for a misbehaving sequencer, while ZK rollups use zero-knowledge proofs (translation: fancy cryptography) to prove that the layer-2 chain has updated correctly.

### The data-availability tradeoff

Moving execution off of the main chain allows for significantly decreased costs for execution and state storage. However, rollups still must post their data to the layer-1 chain to ensure data availability.

Essentially, rollups pay cheap, layer-2 costs for execution and storage, but must still pay layer-1 fees to post their data.

This can be seen on the "Advanced TxInfo" tab on any transaction in the ArbiScan block explorer. The transaction fee is broken down into the calldata costs to post to L1, the computation used on L2, and the L2 storage. And in almost all transactions, the L1 calldata will be the primary driver of fees.

Simply put: posting data to Layer-1 is the primary bottleneck for fees on rollups.

### The future of data availability

While data-availability is the bottleneck for rollups today, it's expected that this will be alleviated over time.

Ethereum upgrades like [Proto-Danksharding](https://www.eip4844.com/) and eventually full [Danksharding](https://notes.ethereum.org/@dankrad/new_sharding) will substantially lower the cost of posting data to Ethereum. Additionally, projects like Celestia aim to provide independent chains purpose-built to provide cheap data availability.

Over the long-run, systems like Danksharding and Celestia will make data availability cheap and abundant, shifting the bottleneck back to execution. However, these solutions will take time to reach maturity: Celestia is still months away from their mainnet launch, and it will likely be over a year until Ethereum can add data-availability upgrades like Proto-Danksharding.

## Calldata compression

Data compression is a field that's older than computers themselves! Invented in 1838, Morse code is the earliest known example of data compression. However, the use of computers accelerated research into data compression, with algorithms like Huffman coding being invented in the 1950s.

Given that rollups have cheap execution but expensive data-availability costs, it's no surprise that these teams have been integrating data-compression algorithms into their protocols. Optimism has integrated the Zlib compression algorithm into their rollup ([read more about their algorithm selection process](https://medium.com/ethereum-optimism/the-road-to-sub-dollar-transactions-part-2-compression-edition-6bb2890e3e92)), while Arbitrum's upcoming Nitro upgrade [uses the brotli compression algorithm](https://research.arbitrum.io/t/compression-in-nitro/20).

_Note: This experiment may have been rushed out ahead of the Nitro launch, to allow experimentation on uncompressed Arbitrum calldata_ 🙂

Data compression algorithms are certainly useful tools that help reduce these calldata costs. However, compressing blockchain transactions is a difficult task: data compression works by finding common patterns and shortening them. However, transactions are full of addresses, hashes, and signatures, which are essentially "random data" to these compression algorithms.

Real reductions in calldata costs will come from developers becoming mindful about how to minimize calldata in their applications. The sky-high gas prices of 2020-2021 forced developers to optimize their code to minimize execution and state storage.

<Tweet tweetId="1498852420327129090" />

As we transition to an L2-world, in which calldata goes from the cheapest resource to the most-expensive resource, developers must again learn these new optimizations.

## Experiment: how much can we compress a simple token transfer

Let's now run an experiment on Arbitrum: how much can we compress the calldata needed for a simple token-transfer? And how much do these optimizations lower transaction fees?

We've also built a simple UI, so you can try this experiment yourself:

_(Note, your wallet must have [Arbitrum Dai](http://arbiscan.io/token/0xda10009cbd5d07dd0cecc66161fc93d7c9000da1), [Arbitrum USDC](http://arbiscan.io/token/0xff970a61a04b1ca14834a43f5de4533ebddb5cc8), [Arbitrum testnet Dai](https://testnet.arbiscan.io/token/0x5364Dc963c402aAF150700f38a8ef52C1D7D7F14), or [Arbitrum testnet UNI](https://testnet.arbiscan.io/token/0x049251A7175071316e089D0616d8B6aaCD2c93b8))_

<LightXfer />

### Experiment design & control case

To run our experiment, we're going to build a simple smart contract that will transfer a token from the transaction sender, to any given address.

This sample smart contract _does_ require the user to send an `approve()` transaction before they can send our actual test transactions. Due to this limitation, it's unlikely that any user would _actually_ want to use this system for token transfers. However, the cost-saving techniques used in this experiment can be applied to other contracts (for example an optimized Uniswap router).

To start this experiment, we'll send a "control" transaction to get a baseline cost. This transaction calls a simple Solidity function, passing the token address, recipient address, and amount of tokens to transfer.

[Our test transaction](https://arbiscan.io/tx/0x7a4aac77d287f37e9263495227e768efe5df19622b35428f7d14833e1b096755) used 576,051 ArbiGas, for a total fee of $0.43.

### Simple calldata compression

Looking at the calldata used for control case, we can see there's a lot of unnecessary data we can strip out.

First, let's remove all the zeros, which are simply added as padding. Zeros _are_ cheaper than non-zero bytes, but they still incur a cost, so let's remove them.

There's also a 4-byte function signature at the beginning, which is an identifier for which Solidity function we're trying to call. We can remove this data, and have our code simply infer what action we're trying to take.

These two optimizations let us reduce the bytecode from 100 bytes down to 43 bytes. [Our test transaction](https://arbiscan.io/tx/0x4b6b1462acc680745c2416bed606c9ef3a0f43888903f794d17e934ab3e419ae) used 494,485 (a 14% decrease), and costs $0.37.

### Deterministic "helper" contract

The majority of our data is now made up of the two addresses in our calldata: one for the address of the token that we'll be transferring, the other for the recipient of the transfer.

However, we can imagine that most users are transferring the same few tokens (WETH, Dai, USDC). One way we can remove the _entire_ token address from our calldata is to deploy a special "helper" contract for that token. We now send our transaction to this helper, entirely avoiding the need to include the token address.

This lets us reduce our data bytecode down to 23 bytes. [Our test transaction](https://arbiscan.io/tx/0x3aa17d92aabe49a34e4edb9df63d5bc5d353a67f39ff891889dafcde1fc801bf) used 457,546 (a 21% decrease from the control), and costs $0.34.


### Address lookup-table

Our last stage used a "helper contract" to remove one address from our calldata, however there's still one other address in our calldata.

Is there another technique that can be more consistently used to "compress" addresses?

Thankfully, Arbitrum has a built-in contract called the "[Address Table Registry](https://developer.offchainlabs.com/docs/Special_Features)", which we can use to shorten our calldata.

This contract is essentially a "phone-book" that maps 20-byte Ethereum addresses to simple integers. Imagine your friend has a traditional phonebook: instead of reading your whole phone number to them, you could just say "I'm the 4th phone number on page 200 of the phonebook", and let them look up your number.

So what we can do is make a contract that will accept an "address index" in place of a full address, and internally look-up the full address.

By replacing both the "token" and "recipient" addresses, we can reduce the calldata down to 9 bytes. [Our test transaction](https://arbiscan.io/tx/0x24f5a6f6c45a60852d45f3d3ac4fcd9ac177ad9e91eab8bd4f8f6756ccad7dc9) used 428,347 (a 26% decrease from the control), and costs $0.32.

### Now combine it all!

Finally, let's combine all our techniques into one:

* Remove padding and function selector
* Use deterministic helper contracts to remove common addresses
* Use the Arbitrum address table to shorten other addresses

All together, our calldata size is now just 6 bytes! [The final test transaction](https://arbiscan.io/tx/0xe1d30e1ae15bb49356b6b48cc001fe5b8b0847ce9f5c6311f13f0c79addd960d) used 426,529 (also a 26% decrease from the control, marginally lower than the previous test case), and costs $0.32.

### Other techniques: Lossy compression

All compression techniques we've covered have been examples of "lossless compression", where the compressed output contains all the same data as the original input.

But just like photo and video files use "lossy compression" algorithms to remove unnecessary information, we can also remove data that's unnecessary in most cases.

The primary examples of this would be shortening numbers to remove unnecessary precision. For example, ERC-20 tokens typically maintain 18 decimal places of precision, yet most users typically only care about up to ~4 decimal places. We could build a contract that will, by default, accept numbers with 8 decimal places and multiply by 10^10, with a secondary function for users that require more precision.

Similarly, dates are typically represented as "the number of seconds since January 1, 1970" (also known as Unix time). Contracts can reduce the size of this integer by accepting times as minutes, hours or days instead, and can set their own "epoch" of, for example, January 1, 2015. 

## Takeaways 

In summary: calldata has gone from the cheapest resource on Ethereum L1, to the most expensive resource on Ethereum rollups. Data-availability technologies like Proto-Danksharding and Celestia will eventually alleviate this bottleneck, but neither have launched, and it will likely be years before data-availability becomes cheap and abundant.

Therefore, blockchain developers need be mindful of the amount of calldata their transactions require, as these have significant effects on end-user transaction fees.

This post outlines a number of techniques that can be used to reduce calldata, however I expect more techniques to come out as "optimizooors" turn their focus to layer 2.

-------

Thoughts or questions? Reach out to me at [@dmihal](https://twitter.com/dmihal) on Twitter.

L2Fees.info is part of the [CryptoFees](https://cryptofees.info/) family of websites, that aims to provide data-driven lenses to help understand the crypto ecosystem. Follow [@CryptoFeesInfo](https://cryptofees.info/) on Twitter, or come [join our community on Telegram](https://t.me/cryptofees).
