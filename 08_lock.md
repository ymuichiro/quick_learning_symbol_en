# 8.Lock

The Symbol blockchain has two types of LockTransactions: Hash Lock Transaction and Secret Lock Transaction.  

## 8.1 Hash Lock

Hash Lock Transaction allows a transaction that will be announced later as it is stored in every node's partial cache with a hash value till the transaction is announced, while it waits to be fully signed and the transaction can be locked without processing it on the API node.
It does not lock the tokens owned by the account, it is the transaction subject to the hash value that is locked.
The cost of a Hash Lock Transaction is 10 XYM and the maximum validity period is approximately 48 hours. The locked funds will be refunded to the account when the Hash Lock Transaction is fully signed.  


### Creation of an Aggregate Bonded Transaction.

```js
bob = sym.Account.generateNewAccount(networkType);

tx1 = sym.TransferTransaction.create(
    undefined,
    bob.address,  //Send to Bob
    [ //1XYM
      new sym.Mosaic(
        new sym.NamespaceId("symbol.xym"),
        sym.UInt64.fromUint(1000000)
      )
    ],
    sym.EmptyMessage, //mptyMessage
    networkType
);

tx2 = sym.TransferTransaction.create(
    undefined,
    alice.address,  //Send to Alice
    [],
    sym.PlainMessage.create('thank you!'), //Message
    networkType
);

aggregateArray = [
    tx1.toAggregate(alice.publicAccount), //Sent from Alice
    tx2.toAggregate(bob.publicAccount), //Sent from  Bob
]

//Aggregate Bonded Transaction
aggregateTx = sym.AggregateTransaction.createBonded(
    sym.Deadline.create(epochAdjustment),
    aggregateArray,
    networkType,
    [],
).setMaxFeeForAggregate(100, 1);

//Signature
signedAggregateTx = alice.sign(aggregateTx, generationHash);
```

Specify the public key of the sender's account when two transactions, tx1 and tx2, are arrayed in AggregateArray.
Get the public key in advance via the API with reference to the chapter on Account.
Arrayed transactions are verified for integrity in that order during block approval.
For example, it is possible to send an NFT from Alice to Bob at tx1 and then send the NFT which is sent Alice to Bob at tx1, from Bob to Carol at tx2, but notifying Aggregate transaction in the order tx2,tx1 will result in an error.
In addition, if there is even one inconsistent transaction in the Aggregate transaction, the entire Aggregate transaction will be an error and will not be approved into the chain.

### Creation, signing and announcement of Hash Lock Transaction
```js
//Creation of Hash Lock TX
hashLockTx = sym.HashLockTransaction.create(
  sym.Deadline.create(epochAdjustment),
    new sym.Mosaic(new sym.NamespaceId("symbol.xym"),sym.UInt64.fromUint(10 * 1000000)), //10xym by default
    sym.UInt64.fromUint(480), // Lock expiry date
    signedAggregateTx,// Register this hash value
    networkType
).setMaxFee(100);

//Signature
signedLockTx = alice.sign(hashLockTx, generationHash);

//Announcing Hash Lock TX
await txRepo.announce(signedLockTx).toPromise();
```

### Announcement of Aggregate Bonded Transaction

After checking with e.g. Explorer, announce the Bonded Transaction to the network.
```js
await txRepo.announceAggregateBonded(signedAggregateTx).toPromise();
```


### Co-signature
Co-sign the locked transaction with the specified account (Bob).

```js
txInfo = await txRepo.getTransaction(signedAggregateTx.hash,sym.TransactionGroup.Partial).toPromise();
cosignatureTx = sym.CosignatureTransaction.create(txInfo);
signedCosTx = bob.signCosignatureTransaction(cosignatureTx);
await txRepo.announceAggregateBondedCosignature(signedCosTx).toPromise();
```

### Note
Hash Lock Transactions can be created and announced by anyone, not just the account that initially creates and signs the transaction, but makes sure that the Aggregate Transaction includes a transaction for whom the account is the signer.
Dummy transactions with no mosaic transmission & no message are fine (they say this is a specification to affect performance).


## 8.2 Secret Lock・Secret Proof

The secret lock creates a common password in advance and locks the designated mosaic.
That allows the recipient to receive the locked mosaic if they can prove possession of the password within the expiry date.

This section describes how Alice locks the 1XYM and Bob unlocks it to receive it.

First, create a Bob account to interact with Alice.
Bob needs to announce the transaction to unlock the lock, so please receive about 10XYM from the FAUCET.

```js
bob = sym.Account.generateNewAccount(networkType);
console.log(bob.address);

//FAUCET URL outlet
console.log("https://testnet.symbol.tools/?recipient=" + bob.address.plain() +"&amount=10");
```

### Secret Lock

Create a common pass for locking and unlocking.

```js
sha3_256 = require('/node_modules/js-sha3').sha3_256;

random = sym.Crypto.randomBytes(20);
hash = sha3_256.create();
secret = hash.update(random).hex(); //Lock keyword
proof = random.toString('hex'); //Unlock keyword
console.log("secret:" + secret);
console.log("proof:" + proof);
```

###### Sample outlet
```js
> secret:f260bfb53478f163ee61ee3e5fb7cfcaf7f0b663bc9dd4c537b958d4ce00e240
  proof:7944496ac0f572173c2549baf9ac18f893aab6d0
```

Creating, signing and announcing transaction
```js
lockTx = sym.SecretLockTransaction.create(
    sym.Deadline.create(epochAdjustment),
    new sym.Mosaic(
      new sym.NamespaceId("symbol.xym"),
      sym.UInt64.fromUint(1000000) //1XYM
    ), //Mosaic to lock
    sym.UInt64.fromUint(480), //Locking period (number of blocks)
    sym.LockHashAlgorithm.Op_Sha3_256, //Algorithm used for lock keyword generation
    secret, //Lock keyword
    bob.address, //Forwarding address on unlock:Bob
    networkType
).setMaxFee(100);

signedLockTx = alice.sign(lockTx,generationHash);
await txRepo.announce(signedLockTx).toPromise();
```

The LockHashAlgorithm is as follows
```js
{0: 'Op_Sha3_256', 1: 'Op_Hash_160', 2: 'Op_Hash_256'}
```

At the time of locking, the unlocking destination is specified, which means  only Bob can unlock it.
The maximum lock period is 365 days (counting number of blocks in days).

Confirm approved transactions.
```js
slRepo = repo.createSecretLockRepository();
res = await slRepo.search({secret:secret}).toPromise();
console.log(res.data[0]);
```
###### Sample outlet
```js
> SecretLockInfo
    amount: UInt64 {lower: 1000000, higher: 0}
    compositeHash: "770F65CB0CC0CA17370DE961B2AA5B48B8D86D6DB422171AB00DF34D19DEE2F1"
    endHeight: UInt64 {lower: 323495, higher: 0}
    hashAlgorithm: 0
    mosaicId: MosaicId {id: Id}
    ownerAddress: Address {address: 'TBXUTAX6O6EUVPB6X7OBNX6UUXBMPPAFX7KE5TQ', networkType: 152}
    recipientAddress: Address {address: 'TBTWKXCNROT65CJHEBPL7F6DRHX7UKSUPD7EUGA', networkType: 152}
    recordId: "6260A1D3205E94BEA3D9E3E9"
    secret: "F260BFB53478F163EE61EE3E5FB7CFCAF7F0B663BC9DD4C537B958D4CE00E240"
    status: 0
    version: 1
```
It shows that Alice who lock the transaction is recorded in ownerAddress and the Bob is recorded in recipientAddress.
The information about the secret is published and Bob informs the network of the corresponding proof.


### Secret Proof

Unlock with using the unlock keywords.
Bob must have obtained the unlock keywords in advance.


```js
proofTx = sym.SecretProofTransaction.create(
    sym.Deadline.create(epochAdjustment),
    sym.LockHashAlgorithm.Op_Sha3_256, //Algorithm used for lock keyword generation
    secret, //Lock keyword
    bob.address, //Deactivated accounts (receiving accounts)
    proof, //Un lock keyword
    networkType
).setMaxFee(100);

signedProofTx = bob.sign(proofTx,generationHash);
await txRepo.announce(signedProofTx).toPromise();
```

Confirm the approval result.
```js
txInfo = await txRepo.getTransaction(signedProofTx.hash,sym.TransactionGroup.Confirmed).toPromise();
console.log(txInfo);
```
###### Sample outlet
```js
> SecretProofTransaction
  > deadline: Deadline {adjustedValue: 12669305546}
    hashAlgorithm: 0
    maxFee: UInt64 {lower: 20700, higher: 0}
    networkType: 152
    payloadSize: 207
    proof: "A6431E74005585779AD5343E2AC5E9DC4FB1C69E"
    recipientAddress: Address {address: 'TBTWKXCNROT65CJHEBPL7F6DRHX7UKSUPD7EUGA', networkType: 152}
    secret: "4C116F32D986371D6BCC44CE64C970B6567686E79850E4A4112AF869580B7C3C"
    signature: "951F440860E8F24F6F3AB8EC670A3D448B12D75AB954012D9DB70030E31DA00B965003D88B7B94381761234D5A66BE989B5A8009BB234716CA3E5847C33F7005"
    signer: PublicAccount {publicKey: '9DC9AE081DF2E76554084DFBCCF2BC992042AA81E8893F26F8504FCED3692CFB', address: Address}
  > transactionInfo: TransactionInfo
        hash: "85044FF702A6966AB13D05DBE4AC4C3A13520C7381F32540429987C207B2056B"
        height: UInt64 {lower: 323805, higher: 0}
        id: "6260CC7F60EE2B0EA10CCEDA"
        merkleComponentHash: "85044FF702A6966AB13D05DBE4AC4C3A13520C7381F32540429987C207B2056B"
    type: 16978
```

The SecretProofTransaction does not contain information about  the amount of mosaic received. Check the amount in the receipt created when the block is generated.
Search for receipts addressed to Bob with receipt type:LockHash_Completed.


```js
receiptRepo = repo.createReceiptRepository();

receiptInfo = await receiptRepo.searchReceipts({
    receiptType:sym.ReceiptTypeLockHash_Completed,
    targetAddress:bob.address
}).toPromise();
console.log(receiptInfo.data);
```
###### Sample outlet
```js
> data: Array(1)
  >  0: TransactionStatement
        height: UInt64 {lower: 323805, higher: 0}
     >  receipts: Array(1)
          > 0: BalanceChangeReceipt
                amount: UInt64 {lower: 1000000, higher: 0}
            > mosaicId: MosaicId
                  id: Id {lower: 760461000, higher: 981735131}
              targetAddress: Address {address: 'TBTWKXCNROT65CJHEBPL7F6DRHX7UKSUPD7EUGA', networkType: 152}
              type: 8786
```

ReceiptType is as follows

```js
{4685: 'Mosaic_Rental_Fee', 4942: 'Namespace_Rental_Fee', 8515: 'Harvest_Fee', 8776: 'LockHash_Completed', 8786: 'LockSecret_Completed', 9032: 'LockHash_Expired', 9042: 'LockSecret_Expired', 12616: 'LockHash_Created', 12626: 'LockSecret_Created', 16717: 'Mosaic_Expired', 16718: 'Namespace_Expired', 16974: 'Namespace_Deleted', 20803: 'Inflation', 57667: 'Transaction_Group', 61763: 'Address_Alias_Resolution', 62019: 'Mosaic_Alias_Resolution'}

8786: 'LockSecret_Completed' :LockSecret is completed
9042: 'LockSecret_Expired'　：LockSecret is expired
```

## 8.3 Tips for use


### Paying the transaction fee instead

Generally blockchains require transaction fees for sending transactions. Therefore, users who want to use blockchains need to get some native currency of the chain for fees (e.g. Symbol's native currency XYM) from the exchange in advance.
If the user is a company, the way it is managed might be an issue from an operational point of view.
If it uses Aggregate Transaction, service providers can cover hash lock and transaction fees on behalf of users.

### Scheduled transactions

Secret locks are refunded to the  account which created the transaction after a specified number of blocks.
Using this, when the service provider charges the cost of the lock for the Secret Lock account, the amount of tokens owned by the user for the lock will increase after the expiry date has passed.
On the other hand, announcing a secret proof transaction before the deadline has passed is treated as a cancellation as the transaction is completed and the appropriation is returned to the service provider.

### Atomic swap
Secret locks can be used to exchange token mosaics with other chains.
Please note that this is not to be mistaken for a Hash Lock, as other chains refer to it as a hash time lock contract (HTLC).
