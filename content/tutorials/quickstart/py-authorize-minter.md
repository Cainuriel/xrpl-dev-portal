---
html: py-authorize-minter.html
parent: send-payments-using-python.html
blurb: Authorize another account to mint NFTs for you.
labels:
  - Accounts
  - Quickstart
  - XRP
  - NFTs, NFTokens
---

# Assign an Authorized Minter Using Python

You can assign another account permission to mint NFTs for you.

This example shows how to:

1. Authorize an account to create NFTs for your account.
2. Mint an NFT for another account, when authorized.

[![Token Test Harness](img/quickstart-py30.png)](img/quickstart-py30.png)

# Usage

You can download the [Quickstart Samples](https://github.com/XRPLF/xrpl-dev-portal/tree/master/content/_code-samples/quickstart/py/){.github-code-download} archive to try the sample in your own browser.

## Get Accounts

1. Open and run `py-authorize-minter.md`.
2. Get accounts.
    1. If you have existing test account seeds:
        1. Paste a seed in the **Standby Seed** field.
        2. Click **Get Standby Account**.
        3. Click **Get Standby Account Info**.
        4. Paste a seed in the **Operational Seed** field.
        5. Click **Get Operational Account**.
        6. Click **Get Operational Account** info.
    2. If you do not have existing test accounts:
        1. Click **Get Standby Account**.
        2. Click **Get Standby Account Info**.
        3. Click **Get Operational Account**.
        4. Click **Get Operational Account Info**.
        
## Authorize an Account to Create NFTs

To authorize another account to create NFTs for your account (for example, allow the operational account to mint NFTs for the standby account):

1. Copy the **Operational Account** value.
2. Paste the **Operational Account** value in the standby **Authorized Minter** field.
3. Click **Set Minter**.

[![Authorized Minter](img/quickstart-py31.png)](img/quickstart-py31.png)

## Mint an NFT for Another Account

This example uses the Operational account, which was authorized in the previous step, to mint a token on behalf of the Standby account.

To mint a non-fungible token for another account:

1. Set the **Flags** field. For testing purposes, we recommend setting the value to _8_. 
2. Enter the **NFT URI**. This is a URI that points to the data or metadata associated with the NFT. You can use the sample URI provided if you do not have one of your own.
3. Enter the **Transfer Fee**, a percentage of the proceeds that the original creator receives from future sales of the NFT. This is a value of 0-50000 inclusive, allowing transfer rates between 0.000% and 50.000% in increments of 0.001%. If you do not set the **Flags** field to allow the NFT to be transferrable, set this field to 0.
4. Enter a **Taxon** for the NFT. If you don't have a use for the field, set it to _0_.
4. Copy the **Standby Account** value.
5. Paste the **Standby Account** value in the Operational account **Issuer** field.
6. Click the Operational account **Mint Other** button.

[![Minted NFT for Another Account](img/quickstart-py32.png)](img/quickstart-py32.png)

Once the item is minted, the authorized minter can sell the NFT normally. The proceeds go to the authorized minter, less the transfer fee. The minter and the issuer can settle up on a division of the price separately.

## Create a Sell Offer

To create a NFT sell offer:

1. On the Operational account side, enter the **Amount** of the sell offer in drops (millionths of an XRP), for example 100000000 (100 XRP).
2. Set the **Flags** field to _1_.
3. Enter the **NFT ID** of the minted NFT you want to sell.
4. Optionally, enter a number of seconds until **Expiration**.
5. Click **Create Sell Offer**.

The important piece of information in the response is the NFT Offer Index, labeled as `nft_offer_index`, which is used to accept the sell offer.

[![NFT Sell Offer](img/quickstart-py33.png)](img/quickstart-py33.png)

## Accept Sell Offer

Once a sell offer is available, you can create a new account to accept the offer and buy the NFT.

To accept an available sell offer:

1. Clear the **Standby Seed** field.
2. Click **Get Standby Account**.
3. Click **Get Standby Account Info**.
4. Enter the **NFT Offer Index** (labeled as `nft_offer_index` in the NFT offer results. This is different from the `nft_id`).
5. Click **Accept Sell Offer**.

[![Transaction Results](img/quickstart-py34.png)](img/quickstart-py34.png)

The Buyer account was debited the 100 XRP price plus 10 drops as the transaction cost. The Seller (Authorized Minter) account is credited 90 XRP. the Issuer and the Seller can divide the proceeds per their agreement in a separate transaction. The original Standby account receives a transfer fee of 10 XRP.

[![Transaction Results](img/quickstart-py35.png)](img/quickstart-py35.png)

# Code Walkthrough

You can download the [Quickstart Samples](https://github.com/XRPLF/xrpl-dev-portal/tree/master/content/_code-samples/quickstart/js/quickstart.zip){.github-code-download} archive to try each of the samples.

## mod6.py

`mod6.py` contains the new business logic for this example, the `set_minter` and `mint_other` methods.

Import dependencies and define the `testnet_url` global variable.

```python
import xrpl 
import json
from xrpl.clients import JsonRpcClient
from xrpl.wallet import Wallet
testnet_url = "https://s.altnet.rippletest.net:51234"
```

### Set Minter

This function sets the authorized minter for an account. Each account can have 0 or 1 authorized minter that can mint NFTs in its stead.

Get the wallet of the account granting permission and instantiate a client.

```python
def set_minter(seed, minter):
    """set_minter"""
    granter_wallet=Wallet(seed, sequence = 16237283)
    client=JsonRpcClient(testnet_url)
```

Define the AccountSet transaction that grants permission to another account to mint tokens on behalf of the granter account.

```python
    set_minter_tx=xrpl.models.transactions.AccountSet(
        account=granter_wallet.classic_address,
        nftoken_minter=minter,
        set_flag=xrpl.models.transactions.AccountSetFlag.ASF_AUTHORIZED_NFTOKEN_MINTER
    )    
```

Sign the transaction

```python
    signed_tx=xrpl.transaction.safe_sign_and_autofill_transaction(
        set_minter_tx, granter_wallet, client)
```

Submit the transaction and return the results.

```python
    reply=""
    try:
        response=xrpl.transaction.send_reliable_submission(signed_tx,client)
        reply=response.result
    except xrpl.transaction.XRPLReliableSubmissionException as e:
        reply=f"Submit failed: {e}"
    return reply
```

### mint_other

Get the minter wallet and instantiate a client connection to the XRP ledger.

``` python
def mint_other(seed, uri, flags, transfer_fee, taxon, issuer):
    """mint_other"""
    minter_wallet=Wallet(seed, sequence=16237283)
    client=JsonRpcClient(testnet_url)
```

Define the `NFTokenMint` transaction. The new parameter in this method is the _issuer_ field, identifying the account on whose behalf the token is being minted.

```python
    mint_other_tx=xrpl.models.transactions.NFTokenMint(
        account=minter_wallet.classic_address,
        uri=xrpl.utils.str_to_hex(uri),
        flags=int(flags),
        transfer_fee=int(transfer_fee),
        nftoken_taxon=int(taxon),
        issuer=issuer
    )
```

Sign and fill the transaction.

```python
    signed_tx=xrpl.transaction.safe_sign_and_autofill_transaction(
        mint_other_tx, minter_wallet, client)
```

Submit the transaction and report the results.

```python
    reply=""
    try:
        response=xrpl.transaction.send_reliable_submission(signed_tx,client)
        reply=response.result
    except xrpl.transaction.XRPLReliableSubmissionException as e:
        reply=f"Submit failed: {e}"
    return reply
```
