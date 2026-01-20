---
layout: post
title: "Bitcoin Transactions: A Dev's Look Under the Hood"
date: 2026-01-15
tags: [bitcoin, blockchain, cryptocurrency, backend]
---

I went down a rabbit hole last month trying to understand how Bitcoin transactions actually work. Not the "blockchain is a distributed ledger" stuff you read everywhere - the actual nuts and bolts of what happens when you send BTC.

## It's Not What You Think

First misconception I had: Bitcoin doesn't work like a bank account with a balance. There's no database entry saying "Kinshuk has 0.5 BTC". Instead, it's all about UTXOs.

## UTXOs - The Building Blocks

UTXO = Unspent Transaction Output

Think of it like cash. When someone sends you Bitcoin, you get a "note" (UTXO) of a specific amount. To spend it, you have to use the whole note and get change back.

Example:
- Alice has a 1 BTC UTXO
- She wants to send Bob 0.3 BTC
- Transaction: Input = 1 BTC UTXO, Outputs = 0.3 BTC to Bob + 0.69 BTC back to Alice (0.01 BTC goes to miners as fee)

The original 1 BTC UTXO is now "spent" and two new UTXOs exist.

## Transaction Structure

A raw Bitcoin transaction looks something like this (simplified):

```
{
  "version": 1,
  "inputs": [
    {
      "txid": "previous_transaction_hash",
      "vout": 0,
      "scriptSig": "signature + public_key"
    }
  ],
  "outputs": [
    {
      "value": 30000000,  // satoshis (0.3 BTC)
      "scriptPubKey": "OP_DUP OP_HASH160 <pubKeyHash> OP_EQUALVERIFY OP_CHECKSIG"
    }
  ]
}
```

The `scriptSig` proves you own the input. The `scriptPubKey` defines the conditions to spend the output.

## The Script Part is Wild

Bitcoin has its own scripting language. It's stack-based and intentionally limited (no loops - prevents infinite execution).

Standard P2PKH (Pay to Public Key Hash) script:

```
scriptPubKey: OP_DUP OP_HASH160 <pubKeyHash> OP_EQUALVERIFY OP_CHECKSIG
scriptSig: <signature> <publicKey>
```

When you spend, Bitcoin concatenates these and executes:
1. Push signature and public key to stack
2. Duplicate the public key
3. Hash it
4. Compare with the pubKeyHash in the output
5. Verify the signature

If the stack ends with `true`, transaction is valid.

## Why This Matters for Devs

If you're building anything that interacts with Bitcoin:

**1. You need to track UTXOs**

Can't just query a balance. You need to know which UTXOs belong to an address and their amounts.

**2. Fee calculation is tricky**

Fees are based on transaction size in bytes, not amount sent. More inputs = bigger transaction = higher fee.

**3. Change addresses matter**

For privacy, wallets often send change to a new address. This is why your wallet shows transactions you don't recognize - it's your own change.

## Tools I Used to Learn

- **Bitcoin Core** - run your own node, poke around with `bitcoin-cli`
- **BlockCypher API** - good for testing without running a node
- **LearnMeABitcoin.com** - best visual explanations I found
- **Mastering Bitcoin (Antonopoulos)** - the bible, but dense

## Simple Code Example

Here's how you might decode a raw transaction in Python:

```python
from bitcoin.core import CTransaction
from bitcoin.core import x  # hex decoder

raw_tx = "0100000001..."  # your raw tx hex
tx = CTransaction.deserialize(x(raw_tx))

print(f"Inputs: {len(tx.vin)}")
print(f"Outputs: {len(tx.vout)}")

for i, output in enumerate(tx.vout):
    print(f"Output {i}: {output.nValue} satoshis")
```

You'll need `python-bitcoinlib` for this.

## The Rabbit Hole Goes Deeper

I didn't even touch on:
- SegWit and how it changes transaction structure
- Taproot and Schnorr signatures
- Lightning Network (layer 2)
- Multi-sig transactions

Maybe future posts if anyone's interested.

## Key Takeaways

1. Bitcoin uses UTXOs, not account balances
2. Transactions consume old UTXOs and create new ones
3. Script language controls spending conditions
4. Transaction size (not value) determines fees

Understanding this stuff helped me debug a payment integration at work way faster than I would have otherwise. Worth the weekend I spent on it.

---

*Building something with Bitcoin? I'd love to hear about it - [007kinshuk@gmail.com](mailto:007kinshuk@gmail.com)*
