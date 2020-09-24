---
title: Signing a Transaction
---

Signing a transaction for Flow is a multi-step process that can involve one or more accounts, each of which signs for a different purpose.

## Signer Roles

- *Proposer:* the account that specifies a [proposal key](#proposal-key).	
- *Payer:* the account paying for the transaction fees.	
- *Authorizers:* zero or more accounts authorizing the transaction to mutate their state.	

## Proposal Key

Each transaction must declare a proposal key, which can be an account key from any Flow account. The account that owns the proposal key is referred to as the _proposer_.

A proposal key definition declares the address, key ID, and up-to-date sequence number for the account key.

```javascript
{	
  // other transaction fields
  // ...
  "proposalKey": {
    "address": "0x01",
    "keyId": 7,
    "sequenceNumber": 42
  }
}	
```	

## Sequence Numbers	

Flow uses sequence numbers to ensure that each transaction executes at most once. 
This prevents many unwanted situations such as 
[transaction replay attacks](https://en.wikipedia.org/wiki/Replay_attack).	

Sequence numbers work similarly to transaction nonces in Ethereum, 
but with several key differences:

- **Each key in an account has a dedicated sequence number** associated with it. 
Unlike Ethereum, there is no sequence number for the entire account.	
- When creating a transaction, only the **proposer must specify a sequence number**. 
Payers and authorizers are not required to.	

The transaction proposer is only required to specify a sequence number for a 
single account key, even it if signs with multiple keys. 
This key is referred to as the _proposal key_.	

Each time an account key is used as a proposal key, its sequence number is incremented by 1. 
The sequence number is updated after execution, 
even if the transaction fails (reverts) during execution.

A transaction is rejected if its proposal key does not specify a sequence number 
equal to the sequence number stored on the account _at execution time._	

**Example**	

After the below transaction is executed, 
the sequence number for `Key 7` on `Account 0x01` will increase to 43.

```javascript
{	
  // other transaction fields	
  // ...	
  "proposalKey": {
    "address": "0x01",
    "keyId": 7,
    "sequenceNumber": 42
  },
  "payer": "0x02",
  "authorizers": [ "0x01" ],
}
```

## Anatomy of a Transaction

Due to the existence of weighted keys and split signing roles, 
Flow transactions sometimes need to be signed multiple times by one or more parties. 
That is, multiple unique signatures may be needed to authorize a single transaction.

A transaction can contain two types of signatures: payload signatures and envelope signatures.

![Transaction Anatomy](transaction-anatomy.png)

### Payload

The transaction _payload_ is the innermost portion of a transaction and 
contains the data that uniquely identifies the operations applied by the transaction. 
In Flow, two transactions with the same payload will never be executed more than once.

The transaction _proposer_ and _authorizer_ are only required to sign the transaction payload. 
These signatures are the **payload signatures**.

### Authorization Envelope

The transaction _authorization envelope_ contains both the transaction payload and 
the payload signatures.

The transaction _payer_ is required to sign the authorization envelope. 
These signatures are the **envelope signatures**.

> ❗ Special case: if an account is both the _payer_ and either a _proposer_ or _authorizer_, it is only required to sign the envelope. 

### Payment Envelope

The outermost portion of the transaction, which contains the payload and envelope signatures, 
is referred to as the _payment envelope_.

### Payer Signs Last

The payer must sign the portion of the transaction that contains the payload signatures, 
which means that the payer must always sign last. This allows the payer to ensure that 
they are signing a valid transaction with all of the required payload signatures.

### Signature Structure

A transaction signature is a composite structure containing three fields:

- Address
- Key ID
- Signature Data

The _address_ and _key ID_ fields declare the account key that generated the signature, 
which is required in order to verify the signature against the correct public key.

## Common Signing Scenarios

Below are several scenarios in which different signature combinations are 
required to authorize a transaction.

### Single party, single signature

The simplest Flow transaction declares a single account as the proposer, 
payer and authorizer. In this case, the account can sign the transaction 
with a single signature.

This scenario is only possible if the signature is generated by a key
with full signing weight.

| Account   | Key ID | Weight |
|-----------|--------|--------|
| `0x01`    | 1      | 1.0    |

```javascript
{	
  "payload": {
    "proposalKey": {
      "address": "0x01",
      "keyId": 1,
      "sequenceNumber": 42
    },
    "payer": "0x01",
    "authorizers": [ "0x01" ]
  },
  "payloadSignatures": [], // 0x01 is the payer, so only needs to sign envelope
  "envelopeSignatures": [
    {
      "address": "0x01",
      "keyId": 1,
      "sig": "0xabc123"
    }
  ]
}
```

### Single party, multiple signatures

A transaction that declares a single account as the proposer, 
payer and authorizer may still specify multiple signatures if the account 
uses weighted keys to achieve multi-sig functionality.

| Account   | Key ID | Weight |
|-----------|--------|--------|
| `0x01`    | 1      | 0.5    |
| `0x01`    | 2      | 0.5    |

```javascript
{	
  "payload": {
    "proposalKey": {
      "address": "0x01",
      "keyId": 1,
      "sequenceNumber": 42
    },
    "payer": "0x01",
    "authorizers": [ "0x01" ]
  },
  "payloadSignatures": [], // 0x01 is the payer, so only needs to sign envelope
  "envelopeSignatures": [
    {
      "address": "0x01",
      "keyId": 1,
      "sig": "0xabc123"
    },
    {
      "address": "0x01",
      "keyId": 2,
      "sig": "0xdef456"
    }
  ]
}
```

### Multiple parties

A transaction that declares different accounts for each signing role will 
require at least one signature from each account.

| Account   | Key ID | Weight |
|-----------|--------|--------|
| `0x01`    | 1      | 1.0    |
| `0x02`    | 1      | 1.0    |

```javascript
{	
  "payload": {
    "proposalKey": {
      "address": "0x01",
      "keyId": 1,
      "sequenceNumber": 42
    },
    "payer": "0x02",
    "authorizers": [ "0x01" ]
  },
  "payloadSignatures": [
    {
      "address": "0x01", // 0x01 is not payer, so only signs payload
      "keyId": 1,
      "sig": "0xabc123"
    }
  ],
  "envelopeSignatures": [
    {
      "address": "0x02",
      "keyId": 1,
      "sig": "0xdef456"
    },
  ]
}
```

### Multiple parties, multiple signatures

A transaction that declares different accounts for each signing role 
may require more than one signature per account if those accounts 
use weighted keys to achieve multi-sig functionality.

| Account   | Key ID | Weight |
|-----------|--------|--------|
| `0x01`    | 1      | 0.5    |
| `0x01`    | 2      | 0.5    |
| `0x02`    | 1      | 0.5    |
| `0x02`    | 2      | 0.5    |

```javascript
{	
  "payload": {
    "proposalKey": {
      "address": "0x01",
      "keyId": 1,
      "sequenceNumber": 42
    },
    "payer": "0x02",
    "authorizers": [ "0x01" ]
  },
  "payloadSignatures": [
    {
      "address": "0x01", // 0x01 is not payer, so only signs payload
      "keyId": 1,
      "sig": "0xabc123"
    },
        {
      "address": "0x01", // 0x01 is not payer, so only signs payload
      "keyId": 2,
      "sig": "0x123abc"
    }
  ],
  "envelopeSignatures": [
    {
      "address": "0x02",
      "keyId": 1,
      "sig": "0xdef456"
    },
    {
      "address": "0x02",
      "keyId": 2,
      "sig": "0x456def"
    },
  ]
}
```