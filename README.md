# **Connecting cryptocurrencies and dollars using Crypto Index **

 

The Crypto Index by itself is a benchmark for investors. Furthermore, when combined with timestamps and signed by a trusted index publisher, the indices can also be used on-chain to provide the following characteristics: 

- **Less volatile pricing ("pay in dollars")**. The amount of transferred cryptocurrency can depend on the Crypto Index, and thus the transferred amount is measured in another cryptocurrency or fiat money.
- **Non-interactive payment**. The sender and receiver no longer need to negotiate about the real-time exchange rate. They just need to setup the script or contract as well as agree on a predetermined third party, who publishes the index. The actual payment is executed later depending on the real-time index.
- **Non-custodial.** Funds are not held by any third party; the trusted third party, *i.e.* the index publisher, can be even *completely unaware* of the reference to him. 
- **Reasonable security.** Roughly as secure as a human could do by watching multiple exchanges directly.

 

## **Background**

Cryptocurrencies are widely questioned and criticized for the large volatility, which limits the usage of cryptocurrencies as a daily payment method. For example, a user pays in BCH to buy a pizza priced in dollars, and later on the seller may find that the exchange rate of BCH to dollars has changed significantly when he wants to redeem. In this scenario, it would be desirable if there is a method to automatically determine the value of cryptocurrencies in dollars. Such a method is not only useful for pricing, but also enables various financial derivatives that depend on the value of cryptocurrencies in each other and in fiat money.



Unfortunately, so far it remains impossible to decide the value of cryptocurrencies on chain. The inherent difficulty comes from the decentralized governance -- there is no central bank nor pegging to any other currency. It is not difficult to collect information from exchanges and make decisions offchain, but an on-chain smart contract or transaction cannot autonomously access such information, for there is no reliable way to introduce offchain information in a trustless manner. 

 

To solve the problems, the Crypto Index was introduced to provide a transparent and timely benchmark of various cryptocurrencies traded globally. The quality and truthfulness of Crypto Index is guaranteed by a neutral third party who digitally signs the broadcasted index and timestamp. Although this proposal does not completely solve the problem of introducing offchain data reliably, it still sounds quite reasonable to trust a neutral third party without conflict of interests, at least no worse than trusting the information posted by a single exchange. Furthermore, if there are multiple mutually independent neutral parties, it would be much more convincing to trust the majority of them. 



  

## **Use Case 1: "Pay in dollars"** 

(For the first use case,  we only present a very simple BCH script here. To have fun with more BCH scripts, a useful tool can be found in <https://ivy.copernet.io/bitcoincash>.)

Alice wants to send Bob the funds with value equals to $V$ dollars, but she does not know what the exchange rate will be like by the time when Bob cashes out the funds. Therefore, Alice locks up the funds perhaps with value greater than $V$ until some prefixed time $t_1$, and allows Bob to claim roughly equal to $V$ dollars in the meanwhile. After time $t_1$ Alice can redeem all the remaining funds.

### Parties:

- Alice, the sender of the funds;
- Bob, the recipient;
- Tom, the trusted index publisher.

### Setup:

Alice needs to generate the script or smart contract that handles the locked funds. She also needs the public key of Bob (`bpubkey`) and Tom (`tpubkey`)  for this setup. In the setup, Alice can specify the time window from $t_0$ to $t_1$ when Bob can claim the funds, as well as the value $V$ that she wants to transfer measured in other currency. 

Note that Alice must also specify the granularity of amount of transferred funds when using scripts in BCH, which is not necessary for smart contracts.

### Protocol Outline

When using the Crypto Index in smart contracts, it is easy to implement the verification the signature of index and timestamp, and perform arbitrary consequential operations thereafter. In what follows we discuss how to use the index in scripts.

For example, Alice wants to send Bob some funds equal to $V$ dollars at the time when $1$ coin is worth about $1$ dollar, and promises Bob that he can claim $2V$ coins if the price goes down to threshold=$0.5$ dollar per coin before time $t_1$. (Of course, Alice can create more outputs to make the payment smoother or resort to smart contracts if possible, but for simplicity we only present a conceptual example script with two outputs.) The protocol works as follows:

1. Alice determines the value $V$ in dollars that she wants to send, the time period $(t_0,t_1)$ that she allows Bob to claim. 
2. Then Alice gets the public key of Bob and the public key of an index publisher Tom that both of them trusts.
3. With above information, Alice creates a funding transaction with two P2SH outputs as below. Alice makes sure that she has saved the recovery information in case of failure.
4. The first output locks $V$ coins until time $t_1$. Bob can claim these funds at any time with his public key, and Alice can redeem the funds with her recovery key in case Bob does not claim by time $t_1$.
5. The second output also locks $V$ coins until time $t_1$, after which Alice can redeem. Before $t_1$ Bob can claim the funds only by providing his public key together with an index signed by Tom showing that the exchange rate goes below $0.5$ dollar/coin in the period $(t_0, t_1)$.
6. After the transaction is broadcasted and processed, Bob is allowed to claim $V$ coins in the first output immediately.
7. Bob watches the index published by Tom. If the index goes below the threshold, Bob can use the timestamped and signed index to claim the $V$ coins in the second output.
8. If the index never goes below the threshold by time $t_1$, then Bob cannot use the second output and Alice gets the funds back.

### Scripts:

##### Output1:

Bob‘s unlock script: `(txsig, bpubkey, 0)`, `script evaluation`:

(pushes from scriptsig: `txsig`, `bpubkey`, ` 0`) 

```
IF                       (txsig, bpubkey)
  DUP                     -skip-
  HASH160                 -skip-
  <recovery_pubkey_hash>  -skip-
  EQUALVERIFY             -skip-
  CHECKSIGVERIFY          -skip-
  <recovery_time>         -skip-
  CHECKLOCKTIMEVERIFY     -skip-
ELSE
  DUP                    (txsig, bpubkey, bpubkey)
  HASH160                (txsig, bpubkey, HASH160(bpubkey))
  <pubkeyhash>           (txsig, bpubkey, HASH160(bpubkey), pubkeyhash)
  EQUALVERIFY            (txsig, bpubkey)
  CHECKSIG               (1)
ENDIF
```

 

Alice's unlock script: `(rtxsig, recovery_pubkey, 1)`, `script evaluation`:

(pushes from scriptsig: `rtxsig`, `recovery_pubkey`, `1`) 

```
IF                       (rtxsig, recovery_pubkey)
  DUP                    (rtxsig, recovery_pubkey, recovery_pubkey)
  HASH160                (rtxsig, recovery_pubkey, HASH160(recovery_pubkey))
  <recovery_pubkey_hash> (rtxsig, recovery_pubkey, HASH160(recovery_pubkey), recovery_pubkey_hash)
  EQUALVERIFY            (rtxsig, recovery_pubkey)
  CHECKSIGVERIFY         ()
  <recovery_time>        (recovery_time)
  CHECKLOCKTIMEVERIFY    (1)
ELSE
  DUP                    -skip-
  HASH160                -skip-
  <pubkeyhash>           -skip-
  EQUALVERIFY            -skip-
  CHECKSIG               -skip-
ENDIF
```

 

##### Output2

Bob‘s unlock script: `(certsig, index, t, txsig, bpubkey, 0)`, `script evaluation`:

(pushes from scriptsig: `certsig`, `index`, `t`, `txsig`, `bpubkey`, `0`)

```
IF                       (certsig, index, t, txsig, bpubkey)
  DUP                     -skip-
  HASH160                 -skip-
  <recovery_pubkey_hash>  -skip-
  EQUALVERIFY             -skip-
  CHECKSIGVERIFY          -skip-
  <recovery_time>         -skip-
  CHECKLOCKTIMEVERIFY     -skip-
ELSE                     (certsig, index, t, txsig, bpubkey, 0)
  DUP                    (certsig, index, t, txsig, bpubkey, bpubkey)
  HASH160                (certsig, index, t, txsig, bpubkey, HASH160(bpubkey))
  <pubkeyhash>           (certsig, index, t, txsig, bpubkey, HASH160(bpubkey), pubkeyhash)
  EQUALVERIFY            (certsig, index, t, txsig, bpubkey)
  CHECKSIGVERIFY         (certsig, index, t)
  DUP                    (certsig, index, t, t)
  <t0>                   (certsig, index, t, t, t0)
  GREATTHAN              (certsig, index, t, 1)
  <1>                    (certsig, index, t, 1, 1)
  NUMEQUALVERIFY         (certsig, index, t)
  DUP                    (certsig, index, t, t)
  <t1>                   (certsig, index, t, t, t1)
  LESSTHAN               (certsig, index, t, 1)
  <1>                    (certsig, index, t, 1, 1)
  NUMEQUALVERIFY         (certsig, index, t)
  <threshold>            (certsig, index, t, threshold)
  <2>                    (certsig, index, t, threshold, 2)
  PICK                   (certsig, index, t, threshold, index)
  GREATTHAN              (certsig, index, t, 1)
  <1>                    (certsig, index, t, 1, 1)
  NUMEQUALVERIFY         (certsig, index, t)
  CAT                    (certsig, index|t)
  <tpubkey>              (certsig, index|t, tpubkey)
  CHECKDATASIG           (1)
ENDIF
```

 

Alice's unlock script: `(rtxsig, recovery_pubkey, 1)`, `script evaluation`:

(pushes from scriptsig: `rtxsig`, `recovery_pubkey`, `1`)

```
IF                       (rtxsig, recovery_pubkey)
  DUP                    (rtxsig, recovery_pubkey, recovery_pubkey)
  HASH160                (rtxsig, recovery_pubkey, HASH160(recovery_pubkey))
  <recovery_pubkey_hash> (rtxsig, recovery_pubkey, HASH160(recovery_pubkey), recovery_pubkey_hash)
  EQUALVERIFY            (rtxsig, recovery_pubkey)
  CHECKSIGVERIFY         ()
  <recovery_time>        (recovery_time)
  CHECKLOCKTIMEVERIFY    (1)
ELSE
  DUP                    -skip-
  HASH160                -skip-
  <pubkeyhash>           -skip-
  EQUALVERIFY            -skip-
  CHECKSIGVERIFY         -skip-
  DUP                    -skip-
  <t0>                   -skip-
  GREATTHAN              -skip-
  <1>                    -skip-
  NUMEQUALVERIFY         -skip-
  DUP                    -skip-
  <t1>                   -skip-
  LESSTHAN               -skip-
  <1>                    -skip-
  NUMEQUALVERIFY         -skip-
  <threshold>            -skip-
  <2>                    -skip-
  PICK                   -skip-
  GREATTHAN              -skip-
  <1>                    -skip-
  NUMEQUALVERIFY         -skip-
  CAT                    -skip-
  <tpubkey>              -skip-
  CHECKDATASIG           -skip-
ENDIF 
```

 

## Use Case 2: Futures Contracts

The index published by a trusted index provider can also be used for on-chain financial derivatives product development by providing a trustworthy reference price. Take futures for an illustration:

*Nov 1, 2018:* Alice bought $1$ lot of November BCH futures on an decentralized exchange at $\$300$/BCH, while Bob sold $1$ lot of November BCH futures at $\$300$/BCH, both parties decide to hold till expiry (i.e. *Nov 30*, *2018*). Suppose the futures contract will be cash settled on the contract expiry date, and will be settled against a trusted settlement price available at 10 AM on the expiry date.

*Nov 30, 2018 (final settlement day):* on 10 AM, a trusted third party index provider published a BCH price $\$310$/BCH. Then, the P/Ls on Alice's account should be $\$310-\$300=\$10$ (Profit) and Bob's account $\$300-\$310=-\$10$ (Loss).

The logic of an on-chain smart contract handling this futures: it simply checks the date is already *Nov 30* and the BCH price \$310/BCH is signed by a predisgnated index provider, then computes the profit and loss of Alice and Bob, and divide the deposits respectively. After the final settlement day, either Alice or Bob (or anybody else) can invoke this contract by sending a transaction with the signed BCH price in the data field.

Of course, a usable futures contract may have more complicated rules and must deal with boundary cases, *e.g.* the deposit of Bob (or Alice) is insufficient for the settlement. But the point of our futures contract example here is that the availability of a third party offchain index will allow the flurish of the on-chain financial product development.



## **Security**

The trust in the third party index publisher is critical. For example, in use case 1, Bob and Tom can collude to steal the locked funds, or Alice and Tom can collude to prevent Bob from claiming the funds. However, Tom is never able to steal the funds by himself, since the funds fully stay in the control of Alice and Bob. Furthermore, Tom can be completely unaware of the script/contract between Alice and Bob when it appears as a commitment, until when the funds are moved away.

On the other hand, if Tom helps either party to steal the funds by signing on an incorrect index, then the incorrect index will be publicly observable when the funds are moved. As a result, everyone will be aware that Tom is malfunctioning and will likely stop referencing Tom’s future indices. 

Similar consequence happens if Tom introduces a significant latency in publishing the index. In both cases Tom will lose reputation and future trust from others, which gives Tom a very strong motivation to remain honest, especially when there are multiple competing index publishers. 

 

## **Conclusion**

The Crypto Index connects the value of cryptocurrencies to each other as well as to fiat money. This connection enables various on-line applications including stable pricing method, futures contracts and more on-chain financial derivatives products. 

For security, the Crypto Index makes no change to the current blockchain protocols and provides an optional service for the vast majority of the cryptocurrency market. Conservative users can remain unchanged by simply not using the index. For those who uses the Crypto Index on-chain, it is also reasonably secure since the index provider cannot steal money from those who references him and more importantly he has strong incentives to behave honestly.





 

 

 

 

 



 

 
