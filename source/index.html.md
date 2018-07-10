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

Band Protocol is still in its pre-alpha development phase. Any of the following
APIs can change without notice.

# Primitives

## Public-Private Keypairs

```python
{
  "vk": "6ddb22994b551f4da5818e7a257d467e9af753348194f31dddc5f9aa489d3da1"
  "sk": "db3d48411f6e41a7e6b903e9dd64c453f98a7133613464a1b1567cffe0c6b5956ddb22994b551f4da5818e7a257d467e9af753348194f31dddc5f9aa489d3da1"
}
```

Band Protocol uses the [Ed25519](https://ed25519.cr.yp.to/) cryptographic
algorithm to generate keypairs. Every message broadcasted to the blockchain
must contain a 64-byte Ed25519 signature suffix to verify the sender's
identity. To avoid potentional confusion with term abbreviation, Band Protocol
uses the term "Verify Key" (vk) to refer to Ed25519 public key and "Secret Key"
(sk) to refer to private key.

## Addresses

```python
{
  "vk": "d2a3a5fa363f2b17ca2c90eb7bde43bda4b34c9d14d13d841d687f29e95524d4"
  "addr_hex": "f3ee3163a9be5a5c48fa90e302d6c106ae794150",
  "addr_iban": "AX17 8RZD C27K Z3PF 2UH4 UDTS FXYB A4ZH USLS"
}
```

An address is a 20-byte data to specify an object on the blockchain. Within
a raw transaction, an address is represented using the big-endian raw format.
Alternatively, an address can be expressed in the [ISO-standard IBAN format]
(https://en.wikipedia.org/wiki/International_Bank_Account_Number) with 32
alphanumeric digits for account number.

`PPCC XXXX XXXX XXXX XXXX XXXX XXXX XXXX XXXX`

where `PP` is the prefix specified below, `CC` is the checksum digits, and
`XXXX...` is the base 32 encode of the actual 20-byte address. Note that
`0`, `1`, `I`, and `O` are omitted from the encoding to avoid confusion.

There are two kinds of addresses in Band Protocol:

1. Account address for storing any kinds of tokens in the blockchain. This
address can be computed by taking the first 20 bytes of SHA256 hash of the
public key. The IBAN prefix is `AX`.

2. Contract address for specifying community or product token contracts.
This address can be computed by taking the first 20 bytes of the hash of
the transaction at which the contract is created. The IBAN prefix is `BX`.
Band native token uses zero address as its address.

## Unsigned Integer Varint Encoding

```python
encode(130 : uint8_t) = 0x82
encode(1000000000000 : uint256_t) = 0x80a094a58d1d
```

1 byte unsigned integers (uint8_t) are encoded using its one byte raw format.
For larger integers, however, Band Protocol uses [base 128 varint encoding]
(https://developers.google.com/protocol-buffers/docs/encoding#varints) to
save transaction size.

## Equation

```python
equation = "((20 * SUPPLY) ^ 2)"
json_format = ["EXP", "MUL", "20", "SUPPLY", "2"]
binary_format = 0x0603071408010702
```

One of the core features of Band Protocol is bonding curve. The protocol allows
first class value for mathematical equations. The binary format is represented
as the infix expression tree format. The JSON format is similar to binary,
but with direct string representations for constants and variables.

Name | ID | Description
-------------- | -------------- | --------------
ADD | 0x01 | Add two expressions (follow by two expressions)
SUB | 0x02 | Subtract two expressions (follow by two expressions)
MUL | 0x03 | Multiply two expressions (follow by two expressions)
DIV | 0x04 | Divide the first expressions by the second (follow by two expressions)
MOD | 0x05 | Find modulus of the first expression by the second (follow by two expressions)
EXP | 0x06 | Find the result the first expression raised to the second expression-th power (follow by two expressions)
Constant | 0x07 | An integer constant (follow by uint256 varint encoding)
Variable | 0x08 | A variable (follow by uint8 variable enum)

The following are the existing variable enums

Value | Text | Description
-------------- | -------------- | --------------
0x01 | SUPPLY | The token supply of the token
0x02 | BNDUSD | The median price of BAND-USD

# RPC Server

Band Protocol runs of top of [Tendermint](https://tendermint.com/). Thus,
the way to communicate with Band Protocol nodes is similar to Tendermint.
See [Tendermint RPC documentation](https://tendermint.github.io/slate/) for
more information.

# Transaction Message Layout

This section describes the transaction format that can be broadcasted to
Band Protocol blockchain via Tendermint ABCI's `BroadcastTxAsync`,
`BroadcastTxCommit`, and `BroadcastTxSync` RPC calls. Every message consists
of three major components.

1. Message Header. This is a fixed size structure similar on every message
type. See below for details.
2. Message Body. This is different depending on the message type. See below
for details.
3. Message Signature. This is 64-byte Ed25519 signature of the whole message
, excluding this signature. The signature must correspond to the verify key
provided in the message header.

## Message Representation Formats

Band Protocol messages can be represented in two different formats:

1. _Binary format_ is the format in which messages are broadcasted to the
blockchain and stored persistently. In this format, raw bytes are represented
as raw big-endian data and numbers are represented using varint encoding.

2. _JSON format_ is the human-readable format. In this format, raw bytes are
represented as hex strings and numbers are represented using base-10 decimal
strings. Band Protocol node provides an RPC interface for third party
applications to conveniently convert messages between binary format and json
format.

## Message Header

```python
{
  "msgid": "1",
  "ts": "1514764800000"
  "vk": "6ddb22994b551f4da5818e7a257d467e9af753348194f31dddc5f9aa489d3da1",
}
```

Field Name | Type | Description
-------------- | -------------- | --------------
MsgID | uint16_t | The type of this message
Timestamp | uint64_t | The time (ms epoch) at which this message is created
VerifyKey | 32 bytes | The verify key of the message sender


## MintMsg (ID = 1)

```python
{
  "token": "BX63 AAAA AAAA AAAA AAAA AAAA AAAA AAAA AAAA",
  "value": "1000"
}
```

Mint Message allows anyone to mint any token. This message will be available
only on testnet to facilitate blockchain testing.

Field Name | Type | Description
-------------- | -------------- | --------------
TokenKey | Address | The key of the token to mint
Value | uint256_t | The number of tokens to mint

## TxMsg (ID = 2)

```python
{
  "token": "1",
  "dest": "AX17 8RZD C27K Z3PF 2UH4 UDTS FXYB A4ZH USLS",
  "value": "1000"
}
```

Tx Message allows anyone to send their token to any address. This transaction
will fail if the sender does not have enough token to send.

Field Name | Type | Description
-------------- | -------------- | --------------
TokenKey | Address | The key of the token to mint
Destination | Address | The address to send the token to
Value | uint256_t | The number of tokens to mint

## CreateMsg (ID = 3)

```python
{
  "expressions": ["ADD", "SUB", "MUL", "2", "EXP", "X", "2", "MUL", "115", "SUPPLY", "79"],
  "max_supply": "1000",
  "spread_type": "1",
  "spread_value": "10"
}
```

Create Messages allows anyone to create a community given (1) the bonding
curve equation, (2) the maximum token supply, and (3) the price spread.

Field Name | Type | Description
-------------- | -------------- | --------------
expressions | expressions | The bonding curve value-supply equation
max_supply | uint256_t | The maximum number of community tokens that can ever be minted
spread_type | uint8_t | The type of price spread (1 for constant, 2 for rational)
spread_value | uint256_t | The spread value. If spread type is 2), the actual value is divided by 1000000

## PurchaseCTMsg (ID = 4)

```python
{
  "contract_id": "BX90 E4RK QV4E YF6M XKCE 5ND8 L6VP 67FF ZBV3",
  "value": "1000",
  "band_limit": "5000"
}
```

Purchase Community Token Message allows anyone to buy community token. Note
that user must specify Band Limit in the message to prevent sudden price
movement that leads to unexpected price.

Field Name | Type | Description
-------------- | -------------- | --------------
Contract ID | Address | The address of the community token contract to purchase
Value | uint256_t | The number of tokens to purchase
Band Limit | uint256_t | The maximum number of Band tokens the sender is willing to pay

## SellCTMsg (ID = 5)

```python
{
  "contract_id": "BX90 E4RK QV4E YF6M XKCE 5ND8 L6VP 67FF ZBV3",
  "value": "1000",
  "band_limit": "5000"
}
```

Sell Community Token Message is the opposite of the above message. The price
user receives here follows the same bonding curve as the purchase, but with
price spread.

Field Name | Type | Description
-------------- | -------------- | --------------
Contract ID | Address | The address of the community token contract to sell
Value | uint256_t | The number of tokens to sell
Band Limit | uint256_t | The minimum number of Band tokens the sender is willing to receive


# ABCI Queries

Band Protocol nodes allow clients to query blockchain data through the
Tendermint ABCI query interface. Due to Tendermint constraints, data must
be passed to band using [ABCIQuery RPC](https://tendermint.github.io/slate/#abciquery).
via HEX encoding. From that, Band interpret the data as a JSON structure
and read the method on `method` field and the parameters on `params` field.
Band protocol returns JSON response to the request. Note that Tendermint will
encode the response using base-64 encoding.

## Generate Binary Message (`txgen`)

```python
{
    "jsonrpc": "2.0",
    "id": "UNIQUE_ID"
    "method": "abci_query",
    "params": {
        "data": HEX_ENCODE({
            "method": "txgen",
            "params": {
                "msgid": "1",
                "vk": "6ddb22994b551f4da5818e7a257d467e9af753348194f31dddc5f9aa489d3da1",
                "token": "BX63 AAAA AAAA AAAA AAAA AAAA AAAA AAAA AAAA",
                "value": "200"
            }
        }),
    },
}
```
> Response

```python
{
    "jsonrpc": "2.0",
    "id": "UNIQUE_ID"
    "result": {
        "response": {
            "value": BASE64_ENCODE(RAW_MESSAGE),
            "height": "10"
        }
    }
}

```

The message simplifies the process of generate binary messages that will be
passed via broadcast tx RPC calls. The query takes the JSON representation
of a messages and returns the binary representation. Note that `sk` field
is optional, and if given, this method also add the message signature to the
end.

## Get Account Balance (`balance`)

```python
{
    "jsonrpc": "2.0",
    "method": "abci_query",
    "params": {
        "data": HEX_ENCODE({
            "method": "balance",
            "params": {
                "address": "AX17 8RZD C27K Z3PF 2UH4 UDTS FXYB A4ZH USLS",
                "token": "BX63 AAAA AAAA AAAA AAAA AAAA AAAA AAAA AAAA",
            }
        }),
    },
    "id": 42
}
```
> Response

```python
{
    "jsonrpc": "2.0",
    "id": "UNIQUE_ID"
    "result": {
        "response": {
            "value": BASE64_ENCODE({
              "balance": "1000"
            }),
            "height": "10"
        }
    }
}
```

This query returns the total number of tokens that given address has for
the given token id.

## Get Community Contract Info (`community_info`)


```python
{
    "jsonrpc": "2.0",
    "method": "abci_query",
    "params": {
        "data": HEX_ENCODE({
            "method": "community_info",
            "params": {
                "contract_id": "BX90 E4RK QV4E YF6M XKCE 5ND8 L6VP 67FF ZBV3",
            }
        }),
    },
    "id": 42
}
```
> Response

```python
{
    "jsonrpc": "2.0",
    "id": "UNIQUE_ID"
    "result": {
        "response": {
            "value": BASE64_ENCODE({
              "equation": "((2 * X) ^ 2)",
              "supply": "1000"
            }),
            "height": "10"
        }
    }
}
```

This query returns the current status of the given community token contract.
