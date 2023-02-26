Overview of Stellar Anchors and integration examples.

# Summary

- [1. Overview](#1-overview)
  - [1.1. Issuance](#11-issuance)
- [2. Deposits and Withdrawals](#2-deposits-and-withdrawals)
- [3. SEPs mentioned in this document](#3-seps-mentioned-in-this-document)
- [4. Integrating](#4-integrating)
  - [4.1. Fetch stellar.toml](#41-fetch-stellartoml)
  - [4.2. Get auth token](#42-get-auth-token)
  - [4.3. Trustline](#43-trustline)
  - [4.4. Deposit](#44-deposit)
  - [4.5. Withdrawal](#45-withdrawal)
  - [4.6. Direct transfer (Anchor-Anchor)](#46-direct-transfer-anchor-anchor)
- [5. Wallet CLI](#5-wallet-cli)

## 1. Overview

The Stellar blockchain allows any wallet to hold different assets, as well as
send/receive payments to/from other Stellar wallets in the blockchain.

## 1.1. Issuance

Each asset is issued ("minted") by a Stellar account, called "issuer".  
Any Stellar account can issue a token. Issuing a token is simply making a Stellar
payment of a new asset. For example:

- There's an arbitrary Stellar account with pubkey (`GAB...SAY`). Let's call it `ACC1`
- There's another arbitrary Stellar account with pubkey (`GDW...FAB`). Let's call it `ACC2`
- `ACC1` wants to become an issuer of an asset named `ASSET123`
- `ACC2` creates a trustline to `asset_code=ASSET123 asset_issuer=GAB...SAY`
- `ACC1` sends a payment of `asset_code=ASSET123 asset_issuer=GAB...SAY` to `GDW...FAB` with `amount=100`
- Now, 100 units of `ASSET123` were created and added to `GDW...FAB`
- This now makes `ACC1` the issuer of `ASSET123`, and `ACC2` a holder of `ASSET123`
- Other Stellar accounts can also create the trustline and hold `ASSET123`

# 2. Deposits and Withdrawals

In order to transfer off-chain funds in and out of the Stellar blockchain, we
have deposits and withdrawals.

The entities (usually companies) that handle `off-chain ← → on-chain` transfers
are called Anchors.

**Deposit** is when a customer is adding off-chain funds (e.g. US dollars, Euros, etc)
into a Stellar wallet. The basic flow for deposit is:

- Customer initiates the deposit from a Stellar Wallet (Lobstr, Stellarterm, Solar, Albedo, Freighter, etc)
- Stellar wallet sends a deposit request to an Anchor
- The Anchor collects the information it needs in order to process the deposit,
  this might include KYC/Bank information
- Anchor displays the deposit instructions to the user
- User makes the off-chain deposit
- Anchor detects the off-chain deposit
- Anchor sends the Stellar payment to the user, completing the deposit flow

**Withdrawal** is when a customer has funds in a Stellar wallet, and proceeds to
move those funds to an off-chain destination (e.g. bank account, bitcoin address, cash pickup, etc).
The basic flow for withdrawal is:

- Customer initiates the withdrawal from a Stellar Wallet (Lobstr, Stellarterm, Solar, Albedo, Freighter, etc)
- Stellar wallet sends a withdrawal request to an Anchor
- The Anchor collects the information it needs in order to process the withdrawal,
  this might include KYC/Bank information, plus the off-chain destination info
  (e.g. destination bank account, destination bitcoin address, etc)
- Wallet sends a Stellar payment to the Anchor
- Anchor detects the Stellar payment
- Anchor sends the off-chain funds to the off-chain destination

Stellar has protocols which the Wallet-Anchor interactions should occur for
deposits and withdrawals. Those protocols are called SEPs (Stellar Ecosystem Proposals).  
Each SEP can be found in the [Stellar git repository](https://github.com/stellar/stellar-protocol/tree/master/ecosystem).
The main focus of this document is to guide Wallets on how to manage deposits
and withdrawals using a Stellar Anchor.
The relevant SEPs for this document are listed [below](#3-seps).

Here are a few links to help you get started:

- [Stellar Introduction](https://developers.stellar.org/docs/start/introduction/)
- [Stellar Tools](https://developers.stellar.org/docs/software-and-sdks/)
- [Stellar Awesome Links](https://github.com/koltenb/awesome-stellar)
- [Javascript Tutorial](https://blog.abuiles.com/building-your-own-venmo-with-stellar/)
- [SEP-24 Instructions for Wallets](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0024.md#basic-wallet-implementation)
- [SEP-6 Instructions for Wallets](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0006.md#basic-wallet-implementation)
- [Anchor Validator](https://anchor-tests.stellar.org/)

Overview of the steps to do a deposit or withdrawal using an Anchor:

- Fetch stellar.toml
  - The Wallet fetches the Anchor's `stellar.toml` file, which contains information
    about currencies and URLs required by the SEPs
- Create trustlines to the Anchor
  - In order to hold assets issued, the Wallet must create a trustline to the asset
    on each of its Stellar accounts. Otherwise the accounts won't be able to receive the deposits.
- Create a deposit or withdrawal transaction using [SEP-6](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0006.md) or [SEP-24](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0024.md)
  - The main difference between SEP-6 and SEP-24 is that SEP-6 requires the Wallet
    to collect all required fields from the user and submit them non-interactively to the Anchor.
    In SEP-24, the Anchor interactively collects all the information it needs from the user.
  - It's up to the Wallet to decide to use SEP-6 or SEP-24 to manage transactions.

Stellar has two networks: `TESTNET` and `PUBLIC`.  
Most Anchors only support deposits and withdrawals in the `PUBLIC` network, as
it's where you can find assets like USD, BTC, etc, which can be converted
to real-world USD and BTC.
On `TESTNET` there are a few Anchors, but they're mostly testing/development versions,
as `TESTNET` is Stellar's test network, where there's no real money involved and
you can get funded test accounts for free.

## 3. SEPs mentioned in this document

- [SEP-1](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0001.md)
- [SEP-2](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0002.md)
- [SEP-6](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0006.md)
- [SEP-10](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0010.md)
- [SEP-12](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0012.md)
- [SEP-24](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0024.md)
- [SEP-31](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0031.md)

## 4. Integrating

### 4.1. Fetch stellar.toml

`stellar.toml` provides these variables, which are required for SEP-10, SEP-6, SEP-24 and SEP-31 respectively:

- `WEB_AUTH_ENDPOINT`
- `TRANSFER_SERVER`
- `TRANSFER_SERVER_SEP0024`
- `DIRECT_PAYMENT_SERVER`

Python example on how to fetch the testnet `stellar.toml`:

```python
import requests
import toml

stellar_toml = toml.loads(requests.get('https://<home_domain>/.well-known/stellar.toml').text)
```

### 4.2. Get auth token

[SEP-10](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0010.md) provides a mechanism to
prove ownership of a Stellar account and obtain a reusable JWT token which carries the ownership information.  
The token is required in many transaction endpoints ([SEP-24](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0024.md), [SEP-6](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0006.md), etc)
and allow the Wallet to deposit and withdrawal funds on the account through a Stellar Anchor.
The token is usually valid for 1 day after it's generated, and can be used in all HTTP requests while it's valid.

Python example (based on [Django Polaris code](https://github.com/stellar/django-polaris/blob/c2efcf4fc8da630ea76c19df7e9b80be671f90ef/polaris/polaris/tests/helpers.py#L15)) on how to get a new JWT token:

```python
from stellar_sdk.keypair import Keypair
from stellar_sdk.transaction_envelope import TransactionEnvelope
from stellar_sdk.network import Network

secret_key = 'SBYWIVPVH5PQPB...'

auth_url = stellar_toml['WEB_AUTH_ENDPOINT']

# get challenge transaction and sign it
client_signing_key = Keypair.from_secret(secret_key)
response = requests.get(f'{auth_url}?account={client_signing_key.public_key}')
content = json.loads(response.content)
envelope_xdr = content['transaction']
envelope_object = TransactionEnvelope.from_xdr(
    envelope_xdr, network_passphrase=Network.TESTNET_NETWORK_PASSPHRASE
)
envelope_object.sign(client_signing_key)
client_signed_envelope_xdr = envelope_object.to_xdr()

# submit the signed transaction to prove ownership of the account
response = requests.post(
    auth_url,
    json={"transaction": client_signed_envelope_xdr},
)
content = json.loads(response.content)

sep10_token = content['token']
```

### 4.3. Trustline

To deposit assets into a Stellar account, the account must first trust the asset.  
A trustline operation is required only once and the trust will last forever on the account unless removed.  
Python example on how to trust an Anchor asset:

```python
from stellar_sdk.server import Server
from stellar_sdk.transaction_builder import TransactionBuilder

secret_key = 'SBYWIVPV...'
horizon_testnet = 'https://horizon-testnet.stellar.org/'
asset_info = stellar_toml['CURRENCIES'][0]

server = Server(horizon_url=horizon_testnet)
keypair = Keypair.from_secret(secret_key)
account = server.load_account(keypair.public_key)
builder = TransactionBuilder(source_account=account,
        network_passphrase=Network.TESTNET_NETWORK_PASSPHRASE)
builder.append_change_trust_op(asset_code=asset_info['code'],
        asset_issuer=asset_info['issuer'])
envelope = builder.build()
envelope.sign(keypair)
response = server.submit_transaction(envelope)
assert response['successful']
```

### 4.4. Deposit

Deposits are a way for users to deposit real world currencies (ex: USD, EUR) into their Stellar account (usually managed by a Wallet).  
For example, a user can have EUR on a bank account outside Stellar, and deposit that as CLPX into a Stellar account.  
To deposit assets, the Wallet must create deposit transactions on our Anchor.  
To create transactions, there are two options:

- [SEP-6](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0006.md)
  - Non-interactive - Wallet must provide all required information through HTTP requests
  - Does not require opening any external web page
  - See [example](#sep-6-deposit-python-example) below
- [SEP-24](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0024.md)
  - Interactive - all information needed from the user is collected by the Anchor using web pages
  - Requires opening a popup window or iframe pointing to an URL provided by the Anchor
  - See [example](#sep-24-deposit-python-example) below

#### SEP-6 Deposit Python example:

```python
def sep6_deposit():
    data = {
        'asset_code': 'PURPLE',
        'account': 'GC75JLZ6...',

        # these fields are specific for this Anchor,
        #  they're specified on the SEP-6 /info endpoint
        'type': 'sepa',  # SEPA transfer deposit
        'first_name': 'John',
        'last_name': 'Doe',
        'email_address': 'johndoe@example.com',
        'amount': '11.0',
    }
    headers = {
        'Authorization': 'Bearer ' + sep10_token
    }
    url = stellar_toml['TRANSFER_SERVER'] + '/deposit'
    response = requests.post(url, data=data, headers=headers).json()
    return render_sep6_instructions(response)  # display instructions to user
```

#### SEP-24 Deposit Python example:

```python
def sep24_deposit():
    data = {
        'asset_code': 'PURPLE',
        'account': 'GC75JLZ6...',
    }
    headers = {
        'Authorization': 'Bearer ' + sep10_token
    }
    url = stellar_toml['TRANSFER_SERVER_SEP0024'] + '/transactions/deposit/interactive'
    response = requests.post(url, data=data, headers=headers).json()
    return render_sep24_interactive(response['url'])  # popup window on the client
```

### 4.5. Withdrawal

Withdrawals are a way for users to obtain assets from their Stellar account as real world currencies (ex: USD, EUR).  
For example, a user can have token balance in a Stellar account and withdraw that as fiat, receiving the fiat on a bank account outside Stellar.  
To withdraw assets, the Wallet must create withdrawal transactions on the Anchor.  
To create transactions, there are two options:

- [SEP-6](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0006.md)
  - Non-interactive - Wallet must provide all required information through API requests
  - Does not require opening any external web page
  - See [example](#sep-6-withdrawal-python-example) below
- [SEP-24](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0024.md)
  - Interactive - all information needed from the user is collected by the Anchor using web pages
  - Requires opening a popup window or iframe pointing to an URL provided by the Anchor
  - See [example](#sep-24-withdrawal-python-example) below

#### SEP-6 Withdrawal Python example:

```python
def sep6_withdrawal():
    data = {
        'asset_code': 'PURPLE',

        # these fields,
        #  they're specified on the SEP-6 /info endpoint
        'amount': '12.5',
        'type': 'cash',
        'dest_country': 'uk',
        'benef_first_name': 'John',
        'benef_last_name': 'Doe',
        'benef_email': 'johndoe@example.com',
    }
    headers = {
        'Authorization': 'Bearer ' + sep10_token
    }
    url = stellar_toml['TRANSFER_SERVER'] + '/withdraw'
    response = requests.post(url, data=data, headers=headers).json()
    return render_sep6_instructions(response)  # display instructions to user
```

#### SEP-24 Withdrawal Python example:

```python
def sep24_withdrawal():
    data = {
        'asset_code': 'PURPLE',
    }
    headers = {
        'Authorization': 'Bearer ' + sep10_token
    }
    url = stellar_toml['TRANSFER_SERVER_SEP0024'] + '/transactions/withdraw/interactive'
    response = requests.post(url, data=data, headers=headers).json()
    return render_sep24_interactive(response['url'])  # popup window on the client
```

### 4.6. Direct transfer (Anchor-Anchor)

A direct transfer is a Stellar payment made between two Anchor accounts.  
For example, let's say Anchor A is a bank and Anchor B is another bank.  
A person who has a bank account in Anchor A wants to send a money transfer
to another person who has a bank account in Anchor B.  
Normally, this can be accomplished via standard bank transfers like SEPA, Wire, etc.  
With [SEP-31](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0031.md),
Anchor A can send a Stellar payment to Anchor B instead of a bank transfer,
this allows the anchors to take advantage of the speed of Stellar payments and
avoid the burocracy of standard bank transfers (which can take days to take place).

- [SEP-31](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0031.md)
  - Used for anchor-anchor direct transfers
  - Non-interactive - Anchor must provide all required information through API requests
  - Does not require opening any external web page
  - See [example](#sep-31-python-example) below

#### SEP-31 Python example:

```python
def sep31_create_transaction():
    payload = {
        "amount": 15.0,
        "asset_code": "PURPLE",
        "fields": {
            "transaction": {
                "remitter_email": "test@test.test",
                "remitter_first_name": "John",
                "remitter_last_name": "Doe",
                "remitter_phone_number": "+4906921999510",
                "beneficiary_email": "test2@test.test",
                "beneficiary_first_name": "Ana",
                "beneficiary_last_name": "Doe",
                "beneficiary_bank_iban": "GB02 REVO 0099 7040 2170 16",
                "beneficiary_bank_bic": "CHASUS33",
                "beneficiary_phone_number": "+4906921999510"
            }
        }
    }
    headers = {
        'Authorization': 'Bearer ' + sep10_token
    }
    url = stellar_toml['TRANSFER_SERVER_SEP0031'] + '/transactions'
    response = requests.post(url, json=payload, headers=headers).json()
    return response
```

## 5. Wallet CLI

This repository contains a CLI Stellar wallet implementation for demonstration purposes.

### 5.1. Requirements

- Python3.6+
- pip3

### 5.2. Dependencies

```
cd wallet-cli
python3 -m pip install virtualenv
python3 -m virtualenv .venv
pip install -r requirements.txt
```

### 5.3. Running

Activate virtualenv (required only once for a terminal session):

```
source .venv/bin/activate
```

Create the database:

```
python cli.py database create
```

Get started by looking at the options:

```
python cli.py --help
```

Examples:

```
# SEP-1
python cli.py sep1 fetch_stellar_toml

# Trust
python cli.py trust change_trust

# SEP-10
python cli.py sep10 auth

# See SEP-24 options
python cli.py sep24 --help
```
