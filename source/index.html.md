---
title: API Reference

language_tabs:
  - example

toc_footers:
  - <a href='https://bandprotocol.com'>BAND Protocol Website</a>

search: true
---

# Introduction

Welcome to Band Protocol Core API. This document aims to explain the native
blockchain message protocol that ones can use to communicate directly with
Band Protocol nodes via [JSON RPC](https://www.jsonrpc.org/specification)
without passing through another layer of client-side libraries.

Band Protocol is still in pre-alpha development. Any of the following APIs
can change without notice.

# Primitives

## Public-Private Keypairs

```json
{
  "vk": "6ddb22994b551f4da5818e7a257d467e9af753348194f31dddc5f9aa489d3da1"
  "sk": "db3d48411f6e41a7e6b903e9dd64c453f98a7133613464a1b1567cffe0c6b5956ddb22994b551f4da5818e7a257d467e9af753348194f31dddc5f9aa489d3da1"
}
```

Band Protocol uses the [Ed25519](https://ed25519.cr.yp.to/) cryptographic
algorithm to generate keypairs. Every message broadcasted to the blockchain
must be suffixed with 64 bytes Ed25519 signature to verify the sender's
identity. To avoid potentional confusion on abbreviation, Band Protocol uses
the term "Verify Key" (vk) to refer to Ed25519 public key and "Secret Key" (sk)
to refer to private key.

## Addresses

```json
{
  "vk": "6ddb22994b551f4da5818e7a257d467e9af753348194f31dddc5f9aa489d3da1"
  "addr_raw": "26daad0c2f2dae34db0f5f6f79e56486055732bc",
  "addr_iban": "AX76 E5PL 4DBR FYZD KY2R M7ZZ V3ME S2CX QNX6"
}
```

An address is a 20-byte data to specify an object on the blockchain. Within
a raw transaction, an address is represented using big-endian raw format.
Alternatively, an address can be expressed in the [ISO-standard IBAN format]
(https://en.wikipedia.org/wiki/International_Bank_Account_Number) with 32
alphanumeric digits for account number.

`PPCC XXXX XXXX XXXX XXXX XXXX XXXX XXXX XXXX`

where `PP` is the prefix specified below, `CC` is the checksum digits, and
`XXXX...` is the base 32 encode of the actual 20-byte address. Note that
`0`, `1`, `I`, and `O` are omitted from the encoding to avoid confusion.

There are two kinds of addresses in Band Protocol:

1. Account address use for storing any kinds of tokens in the blockchain. This
address can be computed by taking the first 20 bytes of SHA256 hash of the
public key. The IBAN prefix is `AX`.

2. Contract address use for specifying community or product token contracts.
This address can be computed by taking the first 20 bytes of the hash of
the transaction at which the contract is created. The IBAN prefix is `BX`.

## Unsigned Integer Varint Encoding

```python
encode(130 : uint8_t) = 0x82
encode(1000000000000 : uint256_t) = 0x80a094a58d1d
```

1 byte unsigned integers (uint8_t) are encoded using its one byte raw format.
For larger integers, however, Band Protocol uses [base 128 varint encoding]
(https://developers.google.com/protocol-buffers/docs/encoding#varints) to
save transaction size.

# RPC Server

Band Protocol runs of top of [Tendermint](https://tendermint.com/). Thus,
the way to communicate with Band Protocol nodes is similar to Tendermint.
See [Tendermint RPC documentation](https://tendermint.github.io/slate/) for
more information.

# Transaction Message Layout

This section describes the transaction format that can be broadcasted to
Band Protocol blockchain via `BroadcastTxAsync`, `BroadcastTxCommit`, or
`BroadcastTxSync` RPC calls. Every message consists of three major components.

1. Message Header. This is a fixed size structure similar on every message
type. See below for details.
2. Message Body. This is different depending on the message type. See below
for details.
3. Message Signature. This is 64-byte Ed25519 signature of the whole message
, excluding this signature. The signature must correspond to the verify key
provided in the message header.

## Message Header

Field Name | Type | Description
-------------- | -------------- | --------------
MsgID | uint16_t | The type of this message
Timestamp | uint64_t | The time (ms epoch) at which this message is created
VerifyKey | 32 bytes | The verify key of the message sender

## MintMsg

Mint Message allows anyone to mint any token. This message will be available
only on testnet to facilitate blockchain testing.

Field Name | Type | Description
-------------- | -------------- | --------------
TokenKey | 20 bytes | The key of the token to mint
Value | uint256_t | The amount of token to mint

## TxMsg

Tx Message allows anyone to send their token to any address. This transaction
will fail if the sender does not have enough token to send.

Field Name | Type | Description
-------------- | -------------- | --------------
TokenKey | 20 bytes | The key of the token to mint
Destination | 20 bytes | The address to send the token to
Value | uint256_t | The amount of token to mint

## CreateMsg

Create Msg allows anyone to create a community token contract with continuous
bonding curve.

Field Name | Type | Description
-------------- | -------------- | --------------
Curve | equation | The continuous bonding curve to create

TODO: Explain how equation is encoded

## PurchaseMsg

Purchase Msg allows anyone to buy community token from the contract. The price
is determined by the bonding curve and the current token supply.


Field Name | Type | Description
-------------- | -------------- | --------------
ContractID | 20 bytes | The address of the contract to purchase token
Value | uint256_t | The number of community token to purchase
Limit | uint256_t | The max amount of BAND the sender is willing to pay


# ABCI Queries

TODO
