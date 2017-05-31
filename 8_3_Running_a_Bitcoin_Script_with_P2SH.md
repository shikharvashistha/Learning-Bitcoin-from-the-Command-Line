# 8.3: Running a Bitcoin Script with P2SH

> **NOTE:** This is a draft in progress, so that I can get some feedback from early reviewers. It is not yet ready for learning.

Now that you know the theory and practice behind P2SH addresses, you're ready to turn a non-standard Bitcoin Script into an actual transaction. We'll be reusing the simple locking script from [§7.2: Running a Bitcoin Script](7_2_Running_a_Bitcoin_Script.md), `OP_ADD 99 OP_EQUAL`.

## Create a P2SH Transaction

To lock a transaction with this Script, do the following:

1. Serialize `OP_ADD 99 OP_EQUAL`:
   1. OP_ADD = 0x93 — a simple opcode translation
   2. 99 = 0x01, 0x63 — this opcode pushes one byte onto the stack, 99 (hex: 0x63)
      * No worries about endian conversion because it's only one byte
   3. OP_EQUAL = 0x87 — a simple opcode translation
   4. `<serialized99Equal>` = "93016387" 
2. Save `<serialized99Equal>` for future reference as the `redeemScript`.
   1. `<redeemScript>` = "93016387"
3. SHA-256 and RIPEMD-160 hash the serialized script.
   1. `<hashed99Equal>` = "3f58b4f7b14847a9083694b9b3b52a4cea2569ed"
4. Produce a P2SH locking script that includes the `<hashed99Equal>`.
   1. `scriptPubKey` = "a9143f58b4f7b14847a9083694b9b3b52a4cea2569ed87"

You can then create a transaction using this `scriptPubKey`, probably via an API.

## Unlock the P2SH Transaction

To unlock this transaction requires that the recipient produce a `scriptSig` that prepends two constants totalling ninety-nine to the serialized script: `1 98 <serialized99Equal>`.

### Run the First Round of Validation

The process of unlocking the P2SH transaction then begins with a first round of validation. 

Concatenate `scriptSig` and `scriptPubKey` and execute them, as normal:
```
Script: 1 98 <serialized99Equal> OP_HASH160 <hashed99Equal> OP_EQUAL
Stack: []

Script: 98 <serialized99Equal> OP_HASH160 <hashed99Equal> OP_EQUAL
Stack: [ 1 ]

Script: <serialized99Equal> OP_HASH160 <hashed99Equal> OP_EQUAL
Stack: [ 1 98 ]

Script: OP_HASH160 <hashed99Equal> OP_EQUAL
Stack: [ 1 98 <serialized99Equal> ]

Script: <hashed99Equal> OP_EQUAL
Stack: [ 1 98 <hashed99Equal> ]

Script: OP_EQUAL
Stack: [ 1 98 <hashed99Equal> <hashed99Equal> ]

Script: 
Stack: [ 1 98 True ]
```
The Script ends with a `True` on top of the stack, and so it succeeds ... even though there's other cruft below it.

However, because this was a P2SH script, the execution isn't done. 

### Run the Second Round of Validation

For the second round of validation, deserialize the `redeemScript` ("93016387" = "OP_ADD 99 OP_EQUAL"), then execute it using the items in the `scriptSig` prior to the serialized script:

```
Script: 1 98 OP_ADD 99 OP_EQUAL
Stack: [ ]

Script: 98 OP_ADD 99 OP_EQUAL
Stack: [ 1 ]

Script: OP_ADD 99 OP_EQUAL
Stack: [ 1 98 ]

Script: 99 OP_EQUAL
Stack: [ 99 ]

Script: OP_EQUAL
Stack: [ 99 99 ]

Script: 
Stack: [ True ]
```
With that second validation _also_ true, the UTXO can now be spent!

## Summary: Building a Bitcoin Script with P2SH

Once you know the technique of building P2SHes, any Script can be embedded in a Bitcoin transaction; and once you understand the technique of validating P2SHes, it's easy to run the scripts in two rounds.