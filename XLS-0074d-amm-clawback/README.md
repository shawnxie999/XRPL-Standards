<pre>
Title:       <b>AMMClawback</b>
Type:        <b>draft</b>

Author:      <a href="mailto:shawnxie@ripple.com">Shawn Xie</a>
             <a href="mailto:yqian@ripple.com">Yinyi Qian</a>
Affiliation: <a href="https://ripple.com">Ripple</a>
</pre>

#  AMMClawback

## Abstract

The AMMClawback amendment empowers token issuers on the XRP Ledger to regulate tokens in an effort to prevent misuse by blacklisted accounts. This document proposes enhancements to the interactions between frozen assets and AMM pools, as well as new clawback mechanism to enable issuer claw back from wallets who have deposited into AMM pools. This proposal will ensure that blacklisted token holders cannot transfer unauthorized funds and will significantly improve use cases such as stablecoins, where ensuring regulatory compliance is essential.

## 1. Overview
This proposal introduces new improvements on how AMM interacts with frozen asset and clawback.

### 1.1. AMM and Frozen Asset
Tokens in frozen trustlines (either global or individual) are currently able to interact with AMM pools, which leaves opportunities for blacklisted accounts to maliciously move funds elsewhere through the AMM. To prevent these behaviors, this proposal introduces the following:
* Prohibit a wallet from depositing new tokens (single-sided and double-sided) into an AMM pool if at least one of the tokens the wallet owns has been frozen (either individually or globally) by the token issuer 
* Prohibit wallets from transfering `LPTokens` to another address if one of the tokens in the pool has been frozen (either individually or globally) by the token issuer

### 1.2. AMM and Clawback
Currently, accounts that have enabled clawback cannot create AMM pools. This proposal introduces the following:
* Allow an account to create AMM pool even if one of the tokens has enabled `lsfAllowTrustLineClawback`. However, token issuers will not be allowed to claw back from the AMM account using the `Clawback` transaction.
* Introduce a new `AMMClawback` transaction to allow token issuers to exclusively claw back from AMM pools that has one of their tokens, according to the current proportion in the pool.


## 2. Specification
### 2.1. AMM and Frozen Asset 
#### 2.1.1. Prohibiting depositing new tokens
#### 2.1.2. Prohibiting transfering LPTokens that are frozen

The document introduces a new defintion: 

> __A holder's LPToken is frozen__ if the holder has been frozen (either globally or individually) by the issuer of one of the assets in the AMM pool.

Currently, the ledger permits the creation of offers involving frozen LPTokens and allows for direct and cross-currency payments using these tokens. Additionally, holders of frozen LPTokens can still transfer them, potentially enabling redemption by other accounts.
This document proposes breaking changes to the offers and payment engine to prevent the transfer of frozen LPTokens.

##### 2.1.2.1.1. Offers
This proposal introduces a new change to the `OfferCreate` transaction to prevent the creation of offers involving frozen LPTokens. Specifically: 
* If the `TakerGets` field specifies a frozen LPToken, the `OfferCreate` transaction will return a `tecFROZEN` error.

Moreover, any existing offers with `TakerGets` set to a frozen LPToken __can no longer be consumed and will be considered as an _unfunded_ offer that will be implicitly cancelled by new Offers that cross it.__

##### 2.1.2.1.2. Rippling  
We propose modifying the rippling step of the payment engine to include a new failure condition: __if the sender attempts to send LPTokens and one or more assets in the associated pool are frozen, the rippling step will fail.__

###### Example
Let's consider the following example:

```
                              +-------+
   +------------------------- | Carol | ------------------------+ 
   |                          +-------+                         | 
   |                              |                             |
   |                              |                             |
   | USD (frozen)                 | USD                         | USD 
   |                              |                             |
   |                              |                             |
+-------+       LPT        +-------------+           LPT    +-------+
| Alice | ---------------> | AMM ACCOUNT | ---------------> |  Bob  |
+-------+                  +-------------+                  +-------+
                                    ^
                                    |
                                    |
                                    v
                                +-------+ 
                                |  XRP  | 
                      AMM Pool  +-------+     
                                |  USD  |       
                                +-------+                                        
```

1. Carol issues USD to Alice and Bob
2. An AMM pool is created with assets XRP and Carol's USD 
3. Both Alice and Bob deposit into the AMM pool, getting back some LPTokens
4. Carol freezes Alice's USD trustline
5. Alice tries to send Bob some LPToken

Currently, at Step 5, Alice can successfully transfer LPTokens to Bob through the AMM account as the gateway. This proposal introduces a change of behavior: _Alice fails to send LPToken to Bob because her USD trustline is frozen._

##### 2.1.2.1.3. Order Book  
We propose modifying the order book step of the payment engine to include a failure condition: __if an offer's `TakerGets` field specifies a frozen LPToken, the offer will be considered unfunded and any attempt to consume it will result in a step failure.__

###### Example
Let's consider the following example:

```

                                +-------+
   +----------------------------| Carol | -------------------------+---------------------+ 
   |                            +-------+                          |                     |
   |                                |                              |                     |
   |                                |                              |                     |
   | USD (frozen)                   | USD (frozen)                 | USD                 | USD
   |                                |                              |                     |
   |                             +-----+                           |                     |
   |                             | Bob |                           |                     |
   |                             +-----+                           |                     |
   |                                 ^                             |                     |
   |                                 |                             |                     |
   |                                 |                             |                     |
   |                                 v                             |                     |                                       
+-------+         XRP            +--------------+      LPT     +----------+    LPT    +-------+
| Alice | ---------------------> | Bob's Offer  | -----------> | AMM ACCT | --------> | David |
+-------+     SendMax: 100 XRP   +--------------+              +----------+           +-------+
              Amount: 100 LPT    | Buy: 100 XRP |       
                                 +--------------+
                                 | Sell: 100 LPT|
                                 +--------------+
                                                                         
```

1. Carol issues USD to Alice, Bob and David
2. An AMM pool is created with assets XRP and Carol's USD 
3. Alice, Bob and David deposit into the AMM pool, getting back some LPTokens
4. Carol freezes Bob's USD trustline
5. Alice tries to send David LPToken by consuming Bob's offer

Currently, at Step 5, Alice can successfully transfer LPTokens to David through Bob's order book (which sells LPToken in exchange for XRP). This proposal introduces a change of behavior: _Alice fails to send LPToken to David because Bob's offer is now considered to be unfunded and cannot be consumed._


### 2.2. AMM and Clawback
#### 2.2.1. Allow creation of AMM pool when tokens have enabled clawback
#### 2.2.2. New transaction to claw back from AMM pools

