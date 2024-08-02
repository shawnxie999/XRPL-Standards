<pre>
Title:       <b>AMMClawback</b>
Type:        <b>draft</b>

Author:      <a href="mailto:shawnxie@ripple.com">Shawn Xie</a>
             <a href="mailto:yqian@ripple.com">Yinyi Qian</a>
Affiliation: <a href="https://ripple.com">Ripple</a>
</pre>

#  AMMClawback

## Abstract

The AMMClawback amendment enables token issuers to regulate tokens and prevent misuse by blacklisted accounts. This proposal builds upon this foundation by introducing two key enhancements: improved interactions between frozen assets and AMM pools to prevent unauthorized transfers, and a new issuer-initiated clawback mechanism. This mechanism allows issuers to claw back tokens from wallets that have deposited into AMM pools, ensuring regulatory compliance. These enhancements will effectively prevent blacklisted token holders from transferring unauthorized funds, significantly benefiting use cases like stablecoins, where regulatory compliance is crucial.

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
#### 2.1.1. Prohibiting the Deposit of New Tokens

This proposal introduces changes to the behavior of the `AMMDeposit` transaction when tokens in trustlines interact with Automated Market Maker (AMM) pools. 

Assume we have created an Automated Market Maker (AMM) pool with two assets: A and B. Currently, asset A is frozen (either individually or globally). The following table outlines whether specific scenarios are allowed or prohibited for the current behavior and proposed behavior:

| Scenario                | Current Behavior  | Proposed Behavior     |
|-------------------------|-------------------|-----------------------|
| Double-Asset Deposit    | Prohibited        | Prohibited            |
| Only Deposit A (frozen) | Prohibited        | Prohibited            |
| Only Deposit B          | Allowed           | Prohibited            |  

As illustrated in the table above, the primary change is that when one asset in the AMM pool is frozen, depositing the other asset is no longer allowed. This means that deposits are prohibited for the non-frozen asset when its paired asset is frozen.

#### 2.1.2. Prohibiting the Transfer of Frozen LPTokens

The document introduces a new defintion: 

> __A holder's LPToken is frozen__ if the holder has been frozen (either globally or individually) by the issuer of one of the assets in the AMM pool.

Currently, the ledger permits the creation of offers involving frozen LPTokens and allows for direct and cross-currency payments using these tokens. Additionally, holders of frozen LPTokens can still transfer them, potentially enabling redemption by other accounts.
This document proposes breaking changes to the offers and payment engine to prevent the transfer of frozen LPTokens.

##### 2.1.2.1. Offers
This proposal introduces a new change to the `OfferCreate` transaction to prevent the creation of offers that involve sending frozen LPTokens. Specifically: 
* If the sell amount (`TakerGets`) field specifies a frozen LPToken, the `OfferCreate` transaction will return a `tecFROZEN` error.

Moreover, any existing offers with `TakerGets` set to a frozen LPToken __can no longer be consumed and will be considered as an _unfunded_ offer that will be implicitly cancelled by new Offers that cross it.__

##### 2.1.2.2. Rippling  
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

##### 2.1.2.3. Order Book  
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
#### 2.2.1. Allowing AMM Pool Creation with Clawback-Enabled Tokens
Currently, when clawback is enabled for the issuer account by setting `lsfAllowTrustLineClawback` flag, `AMMCreate` is prohibited against this issuer. After the AMMClawback amendment, `AMMCreate` is allowed for clawback-enabled issuer. But the issuer can not clawback from the AMM account using `Clawback` transaction. `AMMClawback` transaction is needed for the issuer to clawback from an AMM account.  

##### Example: Illustrating the AMMClawback Amendment

Suppose an issuer has enabled clawback by setting the `lsfAllowTrustLineClawback` flag through an `AccountSet` transaction. Additionally, two trustlines have been established between the holder and the issuer for currencies A and B 

- **Pre-Amendment Behavior:**  
    - If the holder attempts to create an AMM pool (`AMMCreate`) using the pair of trustline assets A and B associated with the issuer, the transaction will fail with a `tecNO_PERMISSION` error.

- **Post-Amendment Behavior:**  
    - After the AMMClawback amendment, if the holder submits an `AMMCreate` transaction to create an AMM pool with assets A and B, the transaction will be successful.
    - However, if the issuer wants to clawback assets from the AMM account, they must use the `AMMClawback` transaction instead of the regular `Clawback` transaction.  

This change allows for the creation of AMM pools with clawback-enabled issuers while introducing a new transaction type (`AMMClawback`) for issuers to clawback assets from AMM accounts.


#### 2.2.2. Introducing a New Transaction for Clawback from AMM Pools
This proposal introduces a new transaction type `AMMClawback` to allow asset issuers to claw back their assets from the AMM pool.  

Issuers can only claw back issued tokens in the AMM pool only if the `lsfAllowTrustLineClawback` flag is enabled. Attempting to do so without this flag set will result in an error code `tecNO_PERMISSION`.  

By designating the AMM account and holder account, this transaction will:  
- Claw back all LPTokens held by the specified holder account that are associated with the issuer from the specified AMM account.
- Initiate a two-asset withdrawal on the current proportion from the AMM account, resulting in:
  - The issuer's asset being returned to the issuer's account.
  - The non-issuer asset being transferred back to the holder's account

##### 2.2.2.1. Fields for AMMClawback transaction  

| Field Name          | Required?        |  JSON Type    | Internal Type     |
|---------------------|:----------------:|:-------------:|:-----------------:|
| `TransactionType`   |:heavy_check_mark:|`string`       |   `UINT16`        |  

`TransactionType` specifies the new transaction type `AMMClawback`. The integer value is 31. The recommended name is `ttAMM_CLAWBACK`.

---

| Field Name          | Required?        |  JSON Type    | Internal Type     |
|---------------------|:----------------:|:-------------:|:-----------------:|
| `Account`           |:heavy_check_mark:|`string`       |   `ACCOUNT ID`    |  

`Account` designates the issuer of the asset being clawed back, and must match the account submitting the transaction.

---  
| Field Name          | Required?        |  JSON Type    | Internal Type     |
|---------------------|:----------------:|:-------------:|:-----------------:|
| `Holder`            |:heavy_check_mark:|`string`       |   `ACCOUNT ID`    |  

`Holder` specifies the holder account of the LP Token to be clawed back.

---  

| Field Name          | Required?        |  JSON Type          | Internal Type     |
|---------------------|:----------------:|:-------------------:|:-----------------:|
| `AMMAccount`        |:heavy_check_mark:|`string`             |   `ACCOUNT ID`    |  

`AMMAccount` specifies the AMM account from which the transaction will withdraw assets after clawing back `Holder`'s LPtokens.

---  


##### 2.2.2.2. AMMClawback transaction example

```
{
  "TransactionType": "AMMClawback",
  "Account": "rPdYxU9dNkbzC5Y2h4jLbVJ3rMRrk7WVRL",
  "Holder": "rvYAfWj5gh67oV6fW32ZzP3Aw4Eubs59B",
  "AMMAccount": "rp2MaZMQDpgAHwWbADaQMrmf4AD5JsPQUR",
  "Flags": 1,
  "Fee": 10
}
```

- Upon execution, this transaction enables the issuer `rPdYxU9dNkbzC5Y2h4jLbVJ3rMRrk7WVRL` to claw back all LPTokens from holder `rvYAfWj5gh67oV6fW32ZzP3Aw4Eubs59B` associated with AMM account `rp2MaZMQDpgAHwWbADaQMrmf4AD5JsPQUR`.
- The transaction will result in the withdrawal of two corresponding assets from the AMM account on the current proportion:  
  - The asset issued by the `Account` will be returned to the issuer.
  - The other asset will be transferred back to the holder's wallet.
