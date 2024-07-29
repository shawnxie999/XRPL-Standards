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
#### 2.1.2.1. Payment Engine
In the current behavior, after the issuer freezes a trustline by setting `lsfHighFreeze`/`lsfLowFreeze` on the trustline (individual freeze) or `lsfGlobalFreeze` on the account (global freeze), the trustline cannot decrease its balance but can still increase its balance (allowing the receipt of funds). This proposal suggests a change to the behavior: __any frozen trustline can no longer increase its balance as a result of any transaction__. In other words, any receipt of funds in the frozen trustline will result in failure.

### 2.2. AMM and Clawback
#### 2.2.1. Allow creation of AMM pool when tokens have enabled clawback
#### 2.2.2. New transaction to claw back from AMM pools

