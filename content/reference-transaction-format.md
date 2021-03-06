# Transactions Overview

A _Transaction_ is the only way to modify the CSC Ledger. Transactions are only valid if signed, submitted, and accepted into a validated ledger version following the [consensus process](https://casinocoin.org/build/casinocoin-ledger-consensus-process/). Some ledger rules also generate _[pseudo-transactions](#pseudo-transactions)_, which aren't signed or submitted, but still must be accepted by consensus. Transactions that fail are also included in ledgers because they modify balances of CSC to pay for the anti-spam [transaction cost](concept-transaction-cost.html).

* [Authorizing Transactions](#authorizing-transactions)
* [Common Fields of All Transactions](#common-fields)
* [Transaction Types](#transaction-types)
* [Reliable Transaction Submission](#reliable-transaction-submission)
* [Transaction Results - How to find and interpret transaction results](#transaction-results)
* [Full Transaction Response List - Complete table of all error codes](#full-transaction-response-list)

## Authorizing Transactions

In the decentralized CSC Ledger, a digital signature proves that a transaction is authorized to do a specific set of actions. Only signed transactions can be submitted to the network and included in a validated ledger. A signed transaction is immutable: its contents cannot change, and the signature is not valid for any other transaction. <!-- STYLE_OVERRIDE: is authorized to -->

A transaction can be authorized by any of the following types of signatures:

* A single signature from the master secret key that is mathematically associated with the sending address. You can disable or enable the master key using an [AccountSet transaction][].
* A single signature that matches a regular key associated with the address. You can add, remove, or replace a regular key using a [SetRegularKey transaction][].
* A [multi-signature](#multi-signing) that matches a list of signers owned by the address. You can add, remove, or replace a list of signers using a [SignerListSet transaction][].

Any signature type can authorize any type of transaction, with the following exceptions:

* Only the master key can [disable the master key](#accountset-flags).
* Only the master key can [permanently give up the ability to freeze](concept-freeze.html#no-freeze).
* You can never remove the last method of signing transactions from an address.

## Signing and Submitting Transactions

Sending a transaction to the CSC Ledger involves several steps:

1. Create an [unsigned transaction in JSON format](#unsigned-transaction-format).
2. Use one or more signatures to [authorize the transaction](#authorizing-transactions).
3. Submit a transaction to a `casinocoind` server. If the transaction is properly formed, the server provisionally applies the transaction to its current version of the ledger and relays the transaction to other members of the peer-to-peer network.
4. The [consensus process](https://casinocoin.org/build/casinocoin-ledger-consensus-process/) determines which provisional transactions get included in the next validated ledger.
5. The `casinocoind` servers apply those transactions to the previous ledger in a canonical order and share their results.
6. If enough [trusted validators](tutorial-casinocoind-setup.html#reasons-to-run-a-validator) created the exact same ledger, that ledger is declared _validated_ and the [results of the transactions](#transaction-results) in that ledger are immutable.

### Unsigned Transaction Format

Here is an example of an unsigned [Payment transaction][] in JSON:

```
{
  "TransactionType" : "Payment",
  "Account" : "cDarPNJEpCnpBZSfmcquydockkePkjPGA2",
  "Destination" : "ca5nK24KXen9AHvsdFTKHSANinZseWnPcX",
  "Amount" : {
     "currency" : "USD",
     "value" : "1",
     "issuer" : "cDarPNJEpCnpBZSfmcquydockkePkjPGA2"
  },
  "Fee": "12",
  "Flags": 2147483648,
  "Sequence": 2,
}
```

The CSC Ledger only relays and executes a transaction if the transaction object has been authorized by the sending address (in the `Account`) field. For transactions authorized by only a single signature, you have two options:

1. Convert it to a binary blob and sign it offline. This is preferable, since it means that the account secret used for signing the transaction is never transmitted over any network connection.
    * You can use [CasinocoinAPI](reference-casinocoinapi.html#sign) for offline signing.
2. Have a `casinocoind` server sign the transaction for you. The [sign command](reference-casinocoind.html#sign) takes a JSON-format transaction and secret and returns the signed binary transaction format ready for submission. (Transmitting your account secret is dangerous, so you should only do this from within a trusted and encrypted connection, or through a local connection, and only to a server you control.)
    * As a shortcut, you can use the [submit command](reference-casinocoind.html#submit) with a `tx_json` object to sign and submit a transaction all at once. This is only recommended for testing and development purposes.

In either case, signing a transaction generates a binary blob that can be submitted to the network. This means using `casinocoind`'s [submit command](reference-casinocoind.html#submit). Here is an example of the same transaction, as a signed blob, being submitted with the WebSocket API:

```
{
  "id": 2,
  "command": "submit",
  "tx_blob" : "120000240000000461D4838D7EA4C6800000000000000000000000000055534400000000004B4E9C06F24296074F7BC48F92A97916C6DC5EA968400000000000000F732103AB40A0490F9B7ED8DF29D246BF2D6269820A0EE7742ACDD457BEA7C7D0931EDB74483046022100982064CDD3F052D22788DB30B52EEA8956A32A51375E72274E417328EBA31E480221008F522C9DB4B0F31E695AA013843958A10DE8F6BA7D6759BEE645F71A7EB240BE81144B4E9C06F24296074F7BC48F92A97916C6DC5EA983143E9D4A2B8AA0780F682D136F7A56D6724EF53754"
}
```

After a transaction has been submitted, you can check its status using the API, for example using the [tx command](reference-casinocoind.html#tx).

**Caution:** The success of a transaction is not final unless the transaction appears in a **validated** ledger with the result code `tesSUCCESS`. See also: [Finality of Results](#finality-of-results).

Example response from the `tx` command:

```
{
  "id": 6,
  "status": "success",
  "type": "response",
  "result": {
    "Account": "cDarPNJEpCnpBZSfmcquydockkePkjPGA2",
    "Amount": {
      "currency": "USD",
      "issuer": "cDarPNJEpCnpBZSfmcquydockkePkjPGA2",
      "value": "1"
    },
    "Destination": "ca5nK24KXen9AHvsdFTKHSANinZseWnPcX",
    "Fee": "10",
    "Flags": 2147483648,
    "Sequence": 2,
    "SigningPubKey": "03AB40A0490F9B7ED8DF29D246BF2D6269820A0EE7742ACDD457BEA7C7D0931EDB",
    "TransactionType": "Payment",
    "TxnSignature": "3045022100D64A32A506B86E880480CCB846EFA3F9665C9B11FDCA35D7124F53C486CC1D0402206EC8663308D91C928D1FDA498C3A2F8DD105211B9D90F4ECFD75172BAE733340",
    "date": 455224610,
    "hash": "33EA42FC7A06F062A7B843AF4DC7C0AB00D6644DFDF4C5D354A87C035813D321",
    "inLedger": 7013674,
    "ledger_index": 7013674,
    "meta": {
      "AffectedNodes": [
        {
          "ModifiedNode": {
            "FinalFields": {
              "Account": "cDarPNJEpCnpBZSfmcquydockkePkjPGA2",
              "Balance": "99999980",
              "Flags": 0,
              "OwnerCount": 0,
              "Sequence": 3
            },
            "LedgerEntryType": "AccountRoot",
            "LedgerIndex": "13F1A95D7AAB7108D5CE7EEAF504B2894B8C674E6D68499076441C4837282BF8",
            "PreviousFields": {
              "Balance": "99999990",
              "Sequence": 2
            },
            "PreviousTxnID": "7BF105CFE4EFE78ADB63FE4E03A851440551FE189FD4B51CAAD9279C9F534F0E",
            "PreviousTxnLgrSeq": 6979192
          }
        },
        {
          "ModifiedNode": {
            "FinalFields": {
              "Balance": {
                "currency": "USD",
                "issuer": "rrrrrrrrrrrrrrrrrrrrBZbvji",
                "value": "2"
              },
              "Flags": 65536,
              "HighLimit": {
                "currency": "USD",
                "issuer": "cDarPNJEpCnpBZSfmcquydockkePkjPGA2",
                "value": "0"
              },
              "HighNode": "0000000000000000",
              "LowLimit": {
                "currency": "USD",
                "issuer": "ca5nK24KXen9AHvsdFTKHSANinZseWnPcX",
                "value": "100"
              },
              "LowNode": "0000000000000000"
            },
            "LedgerEntryType": "CasinocoinState",
            "LedgerIndex": "96D2F43BA7AE7193EC59E5E7DDB26A9D786AB1F7C580E030E7D2FF5233DA01E9",
            "PreviousFields": {
              "Balance": {
                "currency": "USD",
                "issuer": "rrrrrrrrrrrrrrrrrrrrBZbvji",
                "value": "1"
              }
            },
            "PreviousTxnID": "7BF105CFE4EFE78ADB63FE4E03A851440551FE189FD4B51CAAD9279C9F534F0E",
            "PreviousTxnLgrSeq": 6979192
          }
        }
      ],
      "TransactionIndex": 0,
      "TransactionResult": "tesSUCCESS"
    },
    "validated": true
  }
}
```


### Multi-Signing

Multi-signing in the CSC Ledger is the act of [authorizing transactions](#authorizing-transactions) for the CSC Ledger by using a combination of multiple secret keys. You can have any combination of authorization methods enabled for your address, including multi-signing, a master key, and a [regular key](#setregularkey). (The only requirement is that _at least one_ method must be enabled.)

The [SignerListSet transaction][] defines which addresses can authorize transactions from your address. You can include up to 8 addresses in a SignerList. You can control how many signatures are needed, in which combinations, by using the quorum and weight values of the SignerList.

To successfully submit a multi-signed transaction, you must do all of the following:

* The address sending the transaction (specified in the `Account` field) must own a [`SignerList` in the ledger](reference-ledger-format.html#signerlist).
* The transaction must include the `SigningPubKey` field as an empty string.
* The transaction must include a [`Signers` field](#signers-field) containing an array of signatures.
* The signatures present in the `Signers` array must match signers defined in the SignerList.
* For the provided signatures, the total `weight` associated with those signers must be equal or greater than the `quorum` for the SignerList.
* The [transaction cost](concept-transaction-cost.html) (specified in the `Fee` field) must be at least (N+1) times the normal transaction cost, where N is the number of signatures provided.
* All fields of the transaction must be defined before collecting signatures. You cannot [auto-fill](#auto-fillable-fields) any fields.
* If presented in binary form, the `Signers` array must be sorted based on the numeric value of the signer addresses, with the lowest value first. (If submitted as JSON, the [`submit_multisigned` command](reference-casinocoind.html#submit-multisigned) handles this automatically.)

For more information, see [How to Multi-Sign](tutorial-multisign.html).


### Reliable Transaction Submission

Reliably submitting transactions is the process of achieving both of the following:

* Idempotency - A transaction should be processed once and only once, or not at all.
* Verifiability - Applications can determine the final result of a transaction.

To have both qualities when submitting a transaction, an application should:

1. Construct and sign the transaction first, including a [`LastLedgerSequence`](#lastledgersequence) parameter that gives the transaction a limited lifespan.
2. Persist details of the transaction before submitting.
3. Submit the transaction.
4. Confirm that the transaction was either included in a validated ledger, or that it has expired due to `LastLedgerSequence`.
5. If a transaction fails or expires, you can modify and resubmit it.

Main article: [Reliable Transaction Submission](tutorial-reliable-transaction-submission.html)

### Identifying Transactions

The `"hash"` is the unique value that identifies a particular transaction. The server provides the hash in the response when you submit the transaction; you can also look up a transaction in an account's transaction history with the [account_tx command](reference-casinocoind.html#account-tx).

The transaction hash can be used as a "proof of payment" since anyone can [look up the transaction by its hash](#looking-up-transaction-results) to verify its final status.

## Common Fields

[Internal Type]: https://github.com/casinocoin/casinocoind/blob/master/src/casinocoin/protocol/impl/SField.cpp

Every transaction type has the same set of fundamental fields. Field names are case-sensitive. The common fields for all transactions are:

| Field                  | JSON Type        | [Internal Type][] | Description  |
|:-----------------------|:-----------------|:------------------|:-------------|
| Account                | String           | Account           | The unique address of the account that initiated the transaction. |
| [AccountTxnID][]       | String           | Hash256           | _(Optional)_ Hash value identifying another transaction. This transaction is only valid if the sending account's previously-sent transaction matches the provided hash. |
| [Fee][]                | String           | Amount            | (Required, but [auto-fillable](#auto-fillable-fields)) Integer amount of CSC, in drops, to be destroyed as a cost for distributing this transaction to the network. |
| [Flags][]              | Unsigned Integer | UInt32            | _(Optional)_ Set of bit-flags for this transaction. |
| [LastLedgerSequence][] | Number           | UInt32            | (Optional, but strongly recommended) Highest ledger sequence number that a transaction can appear in. |
| [Memos][]              | Array of Objects | Array             | _(Optional)_ Additional arbitrary information used to identify this transaction. |
| PreviousTxnID | String | Hash256 | Use `AccountTxnID` instead. |
| [Sequence][]           | Unsigned Integer | UInt32            | (Required, but [auto-fillable](#auto-fillable-fields)) The sequence number, relative to the initiating account, of this transaction. A transaction is only valid if the `Sequence` number is exactly 1 greater than the last-valided transaction from the same account. |
| SigningPubKey          | String           | PubKey            | (Automatically added when signing) Hex representation of the public key that corresponds to the private key used to sign this transaction. If an empty string, indicates a multi-signature is present in the `Signers` field instead. |
| [Signers][]            | Array            | Array             | _(Optional)_ Array of objects that represent a [multi-signature](#multi-signing) which authorizes this transaction. |
| SourceTag              | Unsigned Integer | UInt32            | _(Optional)_ Arbitrary integer used to identify the reason for this payment, or a sender on whose behalf this transaction is made. Conventionally, a refund should specify the initial payment's `SourceTag` as the refund payment's `DestinationTag`. |
| TransactionType        | String           | UInt16            | The type of transaction. Valid types include: `Payment`, `OfferCreate`, `OfferCancel`, `TrustSet`, `AccountSet`, `SetRegularKey`, `SignerListSet`, `EscrowCreate`, `EscrowFinish`, `EscrowCancel`, `PaymentChannelCreate`, `PaymentChannelFund`, and `PaymentChannelClaim`. |
| TxnSignature           | String           | VariableLength    | (Automatically added when signing) The signature that verifies this transaction as originating from the account it says it is from. |

[AccountTxnID]: #accounttxnid
[Fee]: #transaction-cost
[Flags]: #flags
[LastLedgerSequence]: #lastledgersequence
[Memos]: #memos
[Sequence]: #canceling-or-skipping-a-transaction
[Signers]: #signers-field

### Auto-fillable Fields

Some fields can be automatically filled in before the transaction is signed, either by a `casinocoind` server or by the library used for offline signing. Both [casinocoin-libjs](https://github.com/casinocoin/casinocoin-libjs) and `casinocoind` can automatically provide the following values:

* `Fee` - Automatically fill in the [transaction cost](concept-transaction-cost.html) based on the network. (*Note:* `casinocoind`'s [sign command](reference-casinocoind.html#sign) supports limits on how high the filled-in-value is, using the `fee_mult_max` parameter.)
* `Sequence` - Automatically use the next sequence number for the account sending the transaction.

For a production system, we recommend *not* leaving these fields to be filled by the server. For example, if transaction costs become high due to a temporary spike in network load, you may want to wait for the cost to decrease before sending some transactions, instead of paying the temporarily-high cost.

The [`Paths` field](#paths) of the [Payment](#payment) transaction type can also be automatically filled in.


### Transaction Cost

To protect the CSC Ledger from being disrupted by spam and denial-of-service attacks, each transaction must destroy a small amount of CSC. This _[transaction cost](concept-transaction-cost.html)_ is designed to increase along with the load on the network, making it very expensive to deliberately or inadvertently overload the network.

The `Fee` field specifies an amount, in [drops of CSC][Currency Amount], to destroy as the cost for relaying this transaction. If the transaction is included in a validated ledger (whether or not it achieves its intended purpose), then the amount of CSC specified in the `Fee` parameter is destroyed forever. You can [look up the transaction cost](concept-transaction-cost.html#querying-the-transaction-cost) in advance, or [let `casinocoind` set it automatically](concept-transaction-cost.html#automatically-specifying-the-transaction-cost) when you sign a transaction.

**Note:** [Multi-signed transactions](#multi-signing) require additional fees to relay to the network.


### Canceling or Skipping a Transaction

An important and intentional feature of the CSC Ledger is that a transaction is final as soon as it has been incorporated in a validated ledger.

However, if a transaction has not yet been included in a validated ledger, you can effectively cancel it by rendering it invalid. Typically, this means sending another transaction with the same `Sequence` value from the same account. If you do not want the replacement transaction to do anything, send an [AccountSet](#accountset) transaction with no options.

For example, if you try to submit 3 transactions with sequence numbers 11, 12, and 13, but transaction 11 gets lost somehow or does not have a high enough [transaction cost](#transaction-cost) to be propagated to the network, then you can cancel transaction 11 by submitting an AccountSet transaction with no options and sequence number 11. This does nothing (except destroying the transaction cost for the new transaction 11), but it allows transactions 12 and 13 to become valid.

This approach is preferable to renumbering and resubmitting transactions 12 and 13, because it prevents transactions from being effectively duplicated under different sequence numbers.

In this way, an AccountSet transaction with no options is the canonical "[no-op](http://en.wikipedia.org/wiki/NOP)" transaction.

### LastLedgerSequence

We strongly recommend that you specify the `LastLedgerSequence` parameter on every transaction. Provide a value of about 3 higher than [the most recent ledger index](reference-casinocoind.html#ledger) to ensure that your transaction is either validated or rejected within a matter of seconds.

Without the `LastLedgerSequence` parameter, a transaction can become stuck in an undesirable state where it is neither validated nor rejected for a long time. Specifically, if the load-based [transaction cost](#transaction-cost) of the network increases after you send a transaction, your transaction may not get propagated enough to be included in a validated ledger, but you would have to pay the (increased) transaction cost to [send another transaction canceling it](#canceling-or-skipping-a-transaction). Later, if the transaction cost decreases again, the transaction can become included in a future ledger. The `LastLedgerSequence` places a hard upper limit on how long the transaction can wait to be validated or rejected.

### AccountTxnID

The `AccountTxnID` field lets you chain your transactions together, so that a current transaction is not valid unless the previous one (by Sequence Number) is also valid and matches the transaction you expected.

One situation in which this is useful is if you have a primary system for submitting transactions and a passive backup system. If the passive backup system becomes disconnected from the primary, but the primary is not fully dead, and they both begin operating at the same time, you could potentially have serious problems like some transactions sending twice and others not at all. Chaining your transactions together with `AccountTxnID` ensures that, even if both systems are active, only one of them can submit valid transactions at a time.

To use AccountTxnID, you must first set the [asfAccountTxnID](#accountset-flags) flag, so that the ledger keeps track of the ID for the account's previous transaction.

### Memos

The `Memos` field includes arbitrary messaging data with the transaction. It is presented as an array of objects. Each object has only one field, `Memo`, which in turn contains another object with *one or more* of the following fields:

| Field      | Type   | [Internal Type][] | Description                        |
|:-----------|:-------|:------------------|:-----------------------------------|
| MemoData   | String | VariableLength    | Arbitrary hex value, conventionally containing the content of the memo. |
| MemoFormat | String | VariableLength    | Hex value representing characters allowed in URLs. Conventionally containing information on how the memo is encoded, for example as a [MIME type](http://www.iana.org/assignments/media-types/media-types.xhtml). |
| MemoType   | String | VariableLength    | Hex value representing characters allowed in URLs. Conventionally, a unique relation (according to [RFC 5988](http://tools.ietf.org/html/rfc5988#section-4)) that defines the format of this memo. |

The MemoType and MemoFormat fields should only consist of the following characters: `ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789-._~:/?#[]@!$&'()*+,;=%`

The `Memos` field is limited to no more than 1KB in size (when serialized in binary format).

Example of a transaction with a Memos field:

```
{
    "TransactionType": "Payment",
    "Account": "cMmTCjGFRWPz8S2zAUUoNVSQHxtRQD4eCx",
    "Destination": "c3kmLJN5D28dHuH8vZNUZpMC43pEHpaocV",
    "Memos": [
        {
            "Memo": {
                "MemoType": "687474703a2f2f6578616d706c652e636f6d2f6d656d6f2f67656e65726963",
                "MemoData": "72656e74"
            }
        }
    ],
    "Amount": "1"
}
```

### Signers Field

The `Signers` field contains a [multi-signature](#multi-signing), which has signatures from up to 8 key pairs, that together should authorize the transaction. The `Signers` list is an array of objects, each with one field, `Signer`. The `Signer` field has the following nested fields:

| Field         | Type   | [Internal Type][] | Description                     |
|:--------------|:-------|:------------------|:--------------------------------|
| Account       | String | AccountID         | The address associated with this signature, as it appears in the SignerList. |
| TxnSignature  | String | Blob              | A signature for this transaction, verifiable using the `SigningPubKey`. |
| SigningPubKey | String | PubKey            | The public key used to create this signature. |

The `SigningPubKey` must be a key that is associated with the `Account` address. If the referenced `Account` is a funded account in the ledger, then the SigningPubKey can be that account's current Regular Key if one is set. It could also be that account's Master Key, unless the [lsfDisableMaster](reference-ledger-format.html#accountroot-flags) flag is enabled. If the referenced `Account` address is not a funded account in the ledger, then the `SigningPubKey` must be the master key associated with that address.

Because signature verification is a compute-intensive task, multi-signed transactions cost additional CSC to relay to the network. Each signature included in the multi-signature increases the [transaction cost](concept-transaction-cost.html) required for the transaction. For example, if the current minimum transaction cost to relay a transaction to the network is `10000` drops, then a multi-signed transaction with 3 entries in the `Signers` array would need a `Fee` value of at least `40000` drops to relay.



### Flags

The `Flags` field can contain various options that affect how a transaction should behave. The options are represented as binary values that can be combined with bitwise-or operations to set multiple flags at once.

Most flags only have meaning for a specific transaction type. The same bitwise value may be reused for flags on different transaction types, so it is important to pay attention to the `TransactionType` field when setting and reading flags.

The only flag that applies globally to all transactions is as follows:

| Flag Name           | Hex Value  | Decimal Value | Description               |
|:--------------------|:-----------|:--------------|:--------------------------|
| tfFullyCanonicalSig | 0x80000000 | 2147483648    | Require a fully-canonical signature, to protect a transaction from [transaction malleability](https://wiki.ripple.com/Transaction_Malleability) exploits. |



# Transaction Types

The type of a transaction (`TransactionType` field) is the most fundamental information about a transaction. This indicates what type of operation the transaction is supposed to do.

All transactions have certain fields in common:

* [Common Fields](#common-fields)

Each transaction type has additional fields relevant to the type of action it causes:

* [AccountSet - Set options on an account](#accountset)
* [EscrowCancel - Reclaim escrowed CSC](#escrowcancel)
* [EscrowCreate - Create an escrowed CSC payment](#escrowcreate)
* [EscrowFinish - Deliver escrowed CSC to recipient](#escrowfinish)
* [OfferCancel - Withdraw a currency-exchange order](#offercancel)
* [OfferCreate - Submit an order to exchange currency](#offercreate)
* [Payment - Send funds from one account to another](#payment)
* [PaymentChannelClaim - Claim money from a payment channel](#paymentchannelclaim)
* [PaymentChannelCreate - Open a new payment channel](#paymentchannelcreate)
* [PaymentChannelFund - Add more CSC to a payment channel](#paymentchannelfund)
* [SetRegularKey - Set an account's regular key](#setregularkey)
* [SignerListSet - Set multi-signing settings](#signerlistset)
* [TrustSet - Add or modify a trust line](#trustset)

<!--{# coming soon: 
* [CheckCancel - Cancel a check](#checkcancel)
* [CheckCash - Redeem a check](#checkcash)
* [CheckCreate - Create a check](#checkcreate)
#}-->

_Pseudo-Transactions_ that are not created and submitted in the usual way, but may be added to open ledgers according to ledger rules. They still must be approved by consensus to be included in a validated ledger. Pseudo-transactions have their own unique transaction types:

* [SetFee - Adjust the minimum transaction cost or account reserve](#setfee)
* [EnableAmendment - Apply a change to transaction processing](#enableamendment)


{% include 'transactions/accountset.md' %}

{% include 'transactions/escrowcancel.md' %}

{% include 'transactions/escrowcreate.md' %}

{% include 'transactions/escrowfinish.md' %}

{% include 'transactions/offercancel.md' %}

{% include 'transactions/offercreate.md' %}

{% include 'transactions/payment.md' %}

{% include 'transactions/paymentchannelclaim.md' %}

{% include 'transactions/paymentchannelcreate.md' %}

{% include 'transactions/paymentchannelfund.md' %}

{% include 'transactions/setregularkey.md' %}

{% include 'transactions/signerlistset.md' %}

{% include 'transactions/trustset.md' %}


# Pseudo-Transactions

Pseudo-Transactions are never submitted by users, nor propagated through the network. Instead, a server may choose to inject them in a proposed ledger directly. If enough servers inject an equivalent pseudo-transaction for it to pass consensus, then it becomes included in the ledger, and appears in ledger data thereafter.

Some of the fields that are mandatory for normal transactions do not make sense for pseudo-transactions. In those cases, the pseudo-transaction has the following default values:

| Field         | Default Value                                            |
|:--------------|:---------------------------------------------------------|
| Account       | [ACCOUNT_ZERO](concept-accounts.html#special-addresses) |
| Sequence      | 0                                                        |
| Fee           | 0                                                        |
| SigningPubKey | ""                                                       |
| Signature     | ""                                                       |


{% include 'transactions/enableamendment.md' %}

{% include 'transactions/setfee.md' %}


# Transaction Results

## Immediate Response

The response from the [`submit` command](reference-casinocoind.html#submit) contains a provisional result from the `casinocoind` server indicating what happened during local processing of the transaction.

The response from `submit` contains the following fields:

| Field                   | Value          | Description                       |
|:------------------------|:---------------|:----------------------------------|
| `engine_result`          | String         | A code that categorizes the result, such as `tecPATH_DRY` |
| `engine_result_code`    | Signed Integer | A number that corresponds to the `engine_result`, although exact values are subject to change. |
| `engine_result_message` | String         | A human-readable message explaining what happened. This message is intended for developers to diagnose problems, and is subject to change without notice. |

If nothing went wrong when submitting and applying the transaction locally, the response looks like this:

```js
    "engine_result": "tesSUCCESS",
    "engine_result_code": 0,
    "engine_result_message": "The transaction was applied. Only final in a validated ledger."
```

**Note:** A successful result at this stage does not indicate that the transaction has completely succeeded; only that it was successfully applied to the provisional version of the ledger kept by the local server. Failed results at this stage are also provisional and may change. See [Finality of Results](#finality-of-results) for details.

## Looking up Transaction Results

To see the final result of a transaction, use the [`tx` command](reference-casinocoind.html#tx), [`account_tx` command](reference-casinocoind.html#account-tx), or other response from `casinocoind`. Look for `"validated": true` to indicate that this response uses a ledger version that has been validated by consensus.

| Field                  | Value   | Description                               |
|:-----------------------|:--------|:------------------------------------------|
| meta.TransactionResult | String  | A code that categorizes the result, such as `tecPATH_DRY` |
| validated              | Boolean | Whether or not this result comes from a validated ledger. If `false`, then the result is provisional. If `true`, then the result is final. |

```js
    "hash": "E08D6E9754025BA2534A78707605E0601F03ACE063687A0CA1BDDACFCD1698C7",
    "meta": {
      ...
      "TransactionResult": "tesSUCCESS"
    },
    "validated": true
```

## Result Categories

Both the `engine_result` and the `meta.TransactionResult` use standard codes to identify the results of transactions, as follows:

| Category              | Prefix              | Description                    |
|:----------------------|:--------------------|:-------------------------------|
| Local error           | [tel](#tel-codes)   | The casinocoind server had an error due to local conditions, such as high load. You may get a different response if you resubmit to a different server or at a different time. |
| Malformed transaction | [tem](#tem-codes)   | The transaction was not valid, due to improper syntax, conflicting options, a bad signature, or something else. |
| Failure               | [tef](#tef-codes)   | The transaction cannot be applied to the server's current (in-progress) ledger or any later one. It may have already been applied, or the condition of the ledger makes it impossible to apply in the future. |
| Retry                 | [ter](#ter-codes)   | The transaction could not be applied, but it might be possible to apply later. |
| Success               | [tes](#tes-success) | (Not an error) The transaction succeeded. This result only final in a validated ledger. |
| Claimed cost only     | [tec](#tec-codes)   | The transaction did not achieve its intended purpose, but the [transaction cost](concept-transaction-cost.html) was destroyed. This result is only final in a validated ledger. |

The distinction between a local error (`tel`) and a malformed transaction (`tem`) is a matter of protocol-level rules. For example, the protocol sets no limit on the maximum number of paths that can be included in a transaction. However, a server may define a finite limit of paths it can process. If two different servers are configured differently, then one of them may return a `tel` error for a transaction with many paths, while the other server could successfully process the transaction. If enough servers are able to process the transaction that it survives consensus, then it can still be included in a validated ledger.

By contrast, a `tem` error implies that no server anywhere can apply the transaction, regardless of settings. Either the transaction breaks the rules of the protocol, it is unacceptably ambiguous, or it is completely nonsensical. The only way a malformed transaction could become valid is through changes in the protocol; for example, if a new feature is adopted, then transactions using that feature could be considered malformed by servers that are running older software which predates that feature.

## Claimed Cost Justification

Although it may seem unfair to charge a [transaction cost](concept-transaction-cost.html) for a failed transaction, the `tec` class of errors exists for good reasons:

* Transactions submitted after the failed one do not have to have their Sequence values renumbered. Incorporating the failed transaction into a ledger uses up the transaction's sequence number, preserving the expected sequence.
* Distributing the transaction throughout the network increases network load. Enforcing a cost makes it harder for attackers to abuse the network with failed transactions.
* The transaction cost is generally very small in real-world value, so it should not harm users unless they are sending large quantities of transactions.

## Finality of Results

The order in which transactions apply to the consensus ledger is not final until a ledger is closed and the exact transaction set is approved by the consensus process. A transaction that succeeded initially could still fail, and a transaction that failed initially could still succeed. Additionally, a transaction that was rejected by the consensus process in one round could achieve consensus in a later round.

A validated ledger can include successful transactions (`tes` result codes) as well as failed transactions (`tec` result codes). No transaction with any other result is included in a ledger.

For any other result code, it can be difficult to determine if the result is final. The following table summarizes when a transaction's outcome is final, based on the result code from submitting the transaction:

| Error Code      | Finality                                                   |
|:----------------|:-----------------------------------------------------------|
| `tesSUCCESS`    | Final when included in a validated ledger                  |
| Any `tec` code  | Final when included in a validated ledger                  |
| Any `tem` code  | Final unless the protocol changes to make the transaction valid |
| `tefPAST_SEQ`   | Final when another transaction with the same sequence number is included in a validated ledger |
| `tefMAX_LEDGER` | Final when a validated ledger has a sequence number higher than the transaction's `LastLedgerSequence` field, and no validated ledger includes the transaction |

Any other transaction result is potentially not final. In that case, the transaction could still succeed or fail later, especially if conditions change such that the transaction is no longer prevented from applying. For example, trying to send a non-CSC currency to an account that does not exist yet would fail, but it could succeed if another transaction sends enough CSC to create the destination account. A server might even store a temporarily-failed, signed transaction and then successfully apply it later without asking first.



## Understanding Transaction Metadata

The metadata section of a transaction includes detailed information about all the changes to the shared CSC Ledger that the transaction caused. This is true of any transaction that gets included in a ledger, whether or not it is successful. Naturally, the changes are only final if the transaction is validated.

Some fields that may appear in transaction metadata include:

| Field                                 | Value               | Description    |
|:--------------------------------------|:--------------------|:---------------|
| `AffectedNodes`                       | Array               | List of [ledger objects](reference-ledger-format.html#ledger-object-types) that were created, deleted, or modified by this transaction, and specific changes to each. |
| `DeliveredAmount`                     | [Currency Amount][] | **DEPRECATED.** Replaced by `delivered_amount`. Omitted if not a partial payment. |
| `TransactionIndex`                    | Unsigned Integer    | The transaction's position within the ledger that included it. (For example, the value `2` means it was the 2nd transaction in that ledger.) |
| `TransactionResult`                   | String              | A [result code](#result-categories) indicating whether the transaction succeeded or how it failed. |
| [`delivered_amount`](#delivered-amount) | [Currency Amount][] | The [Currency Amount][] actually received by the `Destination` account. Use this field to determine how much was delivered, regardless of whether the transaction is a [partial payment](#partial-payments). |

### delivered_amount

The `Amount` of a [Payment transaction][] indicates the amount to deliver to the `Destination`, so if the transaction was successful, then the destination received that much -- **except if the transaction was a [partial payment](#partial-payments)**. (In that case, any positive amount up to `Amount` might have arrived.) Rather than choosing whether or not to trust the `Amount` field, you should use the `delivered_amount` field of the metadata to see how much actually reached its destination.

The `delivered_amount` field of transaction metadata is included in all successful Payment transactions, and is formatted like a normal currency amount. However, the delivered amount is not available for transactions that meet both of the following criteria:

* Is a partial payment, and
* Included in a validated ledger before 2014-01-20

If both conditions are true, then `delivered_amount` contains the string value `unavailable` instead of an actual amount. If this happens, you can only figure out the actual delivered amount by reading the AffectedNodes in the transaction's metadata.

See also: [Partial Payments](concept-partial-payments.html)

## Full Transaction Response List

[[Source]<br>](https://github.com/casinocoin/casinocoind/blob/develop/src/casinocoin/protocol/TER.h "Source")


### tel Codes

These codes indicate an error in the local server processing the transaction; it is possible that another server with a different configuration or load level could process the transaction successfully. They have numerical values in the range -399 to -300. The exact code for any given error is subject to change, so don't rely on it.

| Code                  | Explanation                                          |
|:----------------------|:-----------------------------------------------------|
| `telBAD_DOMAIN`        | The transaction specified a domain value (for example, the `Domain` field of an [AccountSet transaction][]) that cannot be used, probably because it is too long to store in the ledger. |
| `telBAD_PATH_COUNT`    | The transaction contains too many paths for the local server to process. |
| `telBAD_PUBLIC_KEY`   | The transaction specified a public key value (for example, as the `MessageKey` field of an [AccountSet transaction][]) that cannot be used, probably because it is too long. |
| `telCAN_NOT_QUEUE`    | The transaction did not meet the [open ledger cost](concept-transaction-cost.html), but this server did not queue this transaction because it did not meet the [queuing restrictions](concept-transaction-cost.html#queuing-restrictions). For example, a transaction returns this code when the sender already has 10 other transactions in the queue. You can try again later or sign and submit a replacement transaction with a higher transaction cost in the `Fee` field. |
| `telCAN_NOT_QUEUE_BALANCE` | The transaction did not meet the [open ledger cost](concept-transaction-cost.html) and also was not added to the transaction queue because the sum of potential CSC costs of already-queued transactions is greater than the expected balance of the account. You can try again later, or try submitting to a different server. |
| `telCAN_NOT_QUEUE_BLOCKS` | The transaction did not meet the [open ledger cost](concept-transaction-cost.html) and also was not added to the transaction queue. This transaction could not replace an existing transaction in the queue because it would block already-queued transactions from the same sender by changing authorization methods. (This includes all [SetRegularKey][] and [SignerListSet][] transactions, as well as [AccountSet][] transactions that change the RequireAuth/OptionalAuth, DisableMaster, or AccountTxnID flags.) You can try again later, or try submitting to a different server. |
| `telCAN_NOT_QUEUE_BLOCKED` | The transaction did not meet the [open ledger cost](concept-transaction-cost.html) and also was not added to the transaction queue because a transaction queued ahead of it from the same sender blocks it. (This includes all [SetRegularKey][] and [SignerListSet][] transactions, as well as [AccountSet][] transactions that change the RequireAuth/OptionalAuth, DisableMaster, or AccountTxnID flags.) You can try again later, or try submitting to a different server. |
| `telCAN_NOT_QUEUE_FEE` | The transaction did not meet the [open ledger cost](concept-transaction-cost.html) and also was not added to the transaction queue. This code occurs when a transaction with the same sender and sequence number already exists in the queue and the new one does not pay a large enough transaction cost to replace the existing transaction. To replace a transaction in the queue, the new transaction must have a `Fee` value that is at least 25% more, as measured in [fee levels](concept-transaction-cost.html#fee-levels). You can increase the `Fee` and try again, send this with a higher `Sequence` number so it doesn't replace an existing transaction, or try sending to another server. |
| `telCAN_NOT_QUEUE_FULL` | The transaction did not meet the [open ledger cost](concept-transaction-cost.html) and the server did not queue this transaction because this server's transaction queue is full. You could increase the `Fee` and try again, try again later, or try submitting to a different server. The new transaction must have a higher transaction cost, as measured in [fee levels](concept-transaction-cost.html#fee-levels), than the transaction in the queue with the smallest transaction cost. |
| `telFAILED_PROCESSING` | An unspecified error occurred when processing the transaction. |
| `telINSUF_FEE_P`       | The `Fee` from the transaction is not high enough to meet the server's current [transaction cost](concept-transaction-cost.html) requirement, which is derived from its load level. |
| `telLOCAL_ERROR`        | Unspecified local error.                             |
| `telNO_DST`_`PARTIAL`   | The transaction is an CSC payment that would fund a new account, but the [tfPartialPayment flag](#partial-payments) was enabled. This is disallowed. |

### tem Codes

These codes indicate that the transaction was malformed, and cannot succeed according to the CSC Ledger protocol. They have numerical values in the range -299 to -200. The exact code for any given error is subject to change, so don't rely on it.

| Code                         | Explanation                                   |
|:-----------------------------|:----------------------------------------------|
| `temBAD_AMOUNT`               | An amount specified by the transaction (for example the destination `Amount` or `SendMax` values of a [Payment](#payment)) was invalid, possibly because it was a negative number. |
| `temBAD_AUTH_MASTER`         | The key used to sign this transaction does not match the master key for the account sending it, and the account does not have a [Regular Key](#setregularkey) set. |
| `temBAD_CURRENCY`             | The transaction improperly specified a currency field. See [Specifying Currency Amounts][Currency Amount] for the correct format. |
| `temBAD_EXPIRATION`           | The transaction improperly specified an expiration value, for example as part of an [OfferCreate transaction][]. Alternatively, the transaction did not specify a required expiration value, for example as part of an [EscrowCreate transaction][]. |
| `temBAD_FEE`                  | The transaction improperly specified its `Fee` value, for example by listing a non-CSC currency or some negative amount of CSC. |
| `temBAD_ISSUER`               | The transaction improperly specified the `issuer` field of some currency included in the request. |
| `temBAD_LIMIT`                | The [TrustSet transaction][] improperly specified the `LimitAmount` value of a trustline. |
| `temBAD_OFFER`                | The [OfferCreate transaction][] specifies an invalid offer, such as offering to trade CSC for itself, or offering a negative amount. |
| `temBAD_PATH`                 | The [Payment](#payment) transaction specifies one or more [Paths](#paths) improperly, for example including an issuer for CSC, or specifying an account differently. |
| `temBAD_PATH_LOOP`           | One of the [Paths](#paths) in the [Payment](#payment) transaction was flagged as a loop, so it cannot be processed in a bounded amount of time. |
| `temBAD_SEND_CSC_LIMIT`     | The [Payment](#payment) transaction used the [tfLimitQuality](#limit-quality) flag in a direct CSC-to-CSC payment, even though CSC-to-CSC payments do not involve any conversions. |
| `temBAD_SEND_CSC_MAX`       | The [Payment](#payment) transaction included a `SendMax` field in a direct CSC-to-CSC payment, even though sending CSC should never require SendMax. (CSC is only valid in SendMax if the destination `Amount` is not CSC.) |
| `temBAD_SEND_CSC_NO_DIRECT` | The [Payment](#payment) transaction used the [tfNoDirectCasinocoin](#payment-flags) flag for a direct CSC-to-CSC payment, even though CSC-to-CSC payments are always direct. |
| `temBAD_SEND_CSC_PARTIAL`   | The [Payment](#payment) transaction used the [tfPartialPayment](#partial-payments) flag for a direct CSC-to-CSC payment, even though CSC-to-CSC payments should always deliver the full amount. |
| `temBAD_SEND_CSC_PATHS`     | The [Payment](#payment) transaction included `Paths` while sending CSC, even though CSC-to-CSC payments should always be direct. |
| `temBAD_SEQUENCE`             | The transaction is references a sequence number that is higher than its own `Sequence` number, for example trying to cancel an offer that would have to be placed after the transaction that cancels it. |
| `temBAD_SIGNATURE`            | The signature to authorize this transaction is either missing, or formed in a way that is not a properly-formed signature. (See [tecNO_PERMISSION](#tec-codes) for the case where the signature is properly formed, but not authorized for this account.) |
| `temBAD_SRC_ACCOUNT`         | The `Account` on whose behalf this transaction is being sent (the "source account") is not a properly-formed [account](concept-accounts.html) address. |
| `temBAD_TRANSFER_RATE`       | The [`TransferRate` field of an AccountSet transaction](#transferrate) is not properly formatted or out of the acceptable range. |
| `temDST_IS_SRC`              | The [TrustSet transaction][] improperly specified the destination of the trust line (the `issuer` field of `LimitAmount`) as the `Account` sending the transaction. You cannot extend a trust line to yourself. (In the future, this code could also apply to other cases where the destination of a transaction is not allowed to be the account sending it.) |
| `temDST_NEEDED`               | The transaction improperly omitted a destination. This could be the `Destination` field of a [Payment](#payment) transaction, or the `issuer` sub-field of the `LimitAmount` field fo a `TrustSet` transaction. |
| `temINVALID`                   | The transaction is otherwise invalid. For example, the transaction ID may not be the right format, the signature may not be formed properly, or something else went wrong in understanding the transaction. |
| `temINVALID_FLAG`             | The transaction includes a [Flag](#flags) that does not exist, or includes a contradictory combination of flags. |
| `temMALFORMED`                 | Unspecified problem with the format of the transaction. |
| `temREDUNDANT`                 | The transaction would do nothing; for example, it is sending a payment directly to the sending account, or creating an offer to buy and sell the same currency from the same issuer. |
| `temREDUNDANT_SEND_MAX`      | |
| `temRIPPLE_EMPTY`             | The [Payment](#payment) transaction includes an empty `Paths` field, but paths are necessary to complete this payment. |
| `temBAD_WEIGHT`                | The [SignerListSet transaction][] includes a `SignerWeight` that is invalid, for example a zero or negative value. |
| `temBAD_SIGNER`                | The [SignerListSet transaction][] includes a signer who is invalid. For example, there may be duplicate entries, or the owner of the SignerList may also be a member. |
| `temBAD_QUORUM`                | The [SignerListSet transaction][] has an invalid `SignerQuorum` value. Either the value is not greater than zero, or it is more than the sum of all signers in the list. |
| `temUNCERTAIN`                 | Used internally only. This code should never be returned. |
| `temUNKNOWN`                   | Used internally only. This code should never be returned. |
| `temDISABLED`                  | The transaction requires logic that is disabled. Typically this means you are trying to use an [amendment](concept-amendments.html) that is not enabled for the current ledger. |


### tef Codes

These codes indicate that the transaction failed and was not included in a ledger, but the transaction could have succeeded in some theoretical ledger. Typically this means that the transaction can no longer succeed in any future ledger. They have numerical values in the range -199 to -100. The exact code for any given error is subject to change, so don't rely on it.

| Code                   | Explanation                                         |
|:-----------------------|:----------------------------------------------------|
| `tefALREADY`             | The same exact transaction has already been applied. |
| `tefBAD_ADD_AUTH`      | **DEPRECATED.**                                     |
| `tefBAD_AUTH`           | The key used to sign this account is not authorized to modify this account. (It could be authorized if the account had the same key set as the [Regular Key](#setregularkey).) |
| `tefBAD_AUTH_MASTER`   | The single signature provided to authorize this transaction does not match the master key, but no regular key is associated with this address. |
| `tefBAD_LEDGER`         | While processing the transaction, the ledger was discovered in an unexpected state. If you can reproduce this error, please [report an issue](https://github.com/casinocoin/casinocoind/issues) to get it fixed. |
| `tefBAD_QUORUM`         | The transaction was [multi-signed](#multi-signing), but the total weights of all included signatures did not meet the quorum. |
| `tefBAD_SIGNATURE`      | The transaction was [multi-signed](#multi-signing), but contained a signature for an address not part of a SignerList associated with the sending account. |
| `tefCREATED`             | **DEPRECATED.**                                     |
| `tefEXCEPTION`           | While processing the transaction, the server entered an unexpected state. This may be caused by unexpected inputs, for example if the binary data for the transaction is grossly malformed. If you can reproduce this error, please [report an issue](https://github.com/casinocoin/casinocoind/issues) to get it fixed. |
| `tefFAILURE`             | Unspecified failure in applying the transaction.    |
| `tefINTERNAL`            | When trying to apply the transaction, the server entered an unexpected state. If you can reproduce this error, please [report an issue](https://github.com/casinocoin/casinocoind/issues) to get it fixed. |
| `tefINVARIANT_FAILED`   | An invariant check failed when trying to claim the [transaction cost](concept-transaction-cost.html). Requires the [EnforceInvariants amendment](reference-amendments.html#enforceinvariants). If you can reproduce this error, please [report an issue](https://github.com/casinocoin/casinocoind/issues). |
| `tefMASTER_DISABLED`    | The transaction was signed with the account's master key, but the account has the `lsfDisableMaster` field set. |
| `tefMAX_LEDGER`         | The transaction included a [`LastLedgerSequence`](#lastledgersequence) parameter, but the current ledger's sequence number is already higher than the specified value. |
| `tefNO_AUTH_REQUIRED`  | The [TrustSet transaction][] tried to mark a trustline as authorized, but the `lsfRequireAuth` flag is not enabled for the corresponding account, so authorization is not necessary. |
| `tefNOT_MULTI_SIGNING` | The transaction was [multi-signed](#multi-signing), but the sending account has no SignerList defined. |
| `tefPAST_SEQ`           | The sequence number of the transaction is lower than the current sequence number of the account sending the transaction. |
| `tefWRONG_PRIOR`        | The transaction contained an `AccountTxnID` field (or the deprecated `PreviousTxnID` field), but the transaction specified there does not match the account's previous transaction. |

### ter Codes

These codes indicate that the transaction failed, but it could apply successfully in the future, usually if some other hypothetical transaction applies first. They have numerical values in the range -99 to -1. The exact code for any given error is subject to change, so don't rely on it.

| Code             | Explanation                                               |
|:-----------------|:----------------------------------------------------------|
| `terFUNDS_SPENT`  | **DEPRECATED.**                                           |
| `terINSUF_FEE_B` | The account sending the transaction does not have enough CSC to pay the `Fee` specified in the transaction. |
| `terLAST`          | Used internally only. This code should never be returned. |
| `terNO_ACCOUNT`   | The address sending the transaction is not funded in the ledger (yet). |
| `terNO_AUTH`      | The transaction would involve adding currency issued by an account with `lsfRequireAuth` enabled to a trust line that is not authorized. For example, you placed an offer to buy a currency you aren't authorized to hold. |
| `terNO_LINE`      | Used internally only. This code should never be returned. |
| `terno_casinocoin`    | Used internally only. This code should never be returned. |
| `terOWNERS`        | The transaction requires that account sending it has a nonzero "owners count", so the transaction cannot succeed. For example, an account cannot enable the [`lsfRequireAuth`](#accountset-flags) flag if it has any trust lines or available offers. |
| `terPRE_SEQ`      | The `Sequence` number of the current transaction is higher than the current sequence number of the account sending the transaction. |
| `terRETRY`         | Unspecified retriable error.                              |
| `terQUEUED`        | The transaction met the load-scaled [transaction cost](concept-transaction-cost.html) but did not meet the open ledger requirement, so the transaction has been queued for a future ledger. |

### tes Success

The code `tesSUCCESS` is the only code that indicates a transaction succeeded. This does not always mean it did what it was supposed to do. (For example, an [OfferCancel][] can "succeed" even if there is no offer for it to cancel.) Success uses the numerical value 0.

| Code       | Explanation                                                     |
|:-----------|:----------------------------------------------------------------|
| `tesSUCCESS` | The transaction was applied and forwarded to other servers. If this appears in a validated ledger, then the transaction's success is final. |

### tec Codes

These codes indicate that the transaction failed, but it was applied to a ledger to apply the [transaction cost](concept-transaction-cost.html). They have numerical values in the range 100 to 199. The exact codes sometimes appear in ledger data, so they do not change, but we recommend not relying on the numeric value regardless.

| Code                       | Value | Explanation                             |
|:---------------------------|:------|:----------------------------------------|
| `tecCLAIM`                 | 100   | Unspecified failure, with transaction cost destroyed. |
| `tecCRYPTOCONDITION_ERROR` | 146   | This [EscrowCreate][] or [EscrowFinish][] transaction contained a malformed or mismatched crypto-condition. |
| `tecDIR_FULL`              | 121   | The address sending the transaction cannot own any more objects in the ledger. |
| `tecDST_TAG_NEEDED`        | 143   | The [Payment](#payment) transaction omitted a destination tag, but the destination account has the `lsfRequireDestTag` flag enabled. |
| `tecFAILED_PROCESSING`     | 105   | An unspecified error occurred when processing the transaction. |
| `tecFROZEN`                | 137   | The [OfferCreate transaction][] failed because one or both of the assets involved are subject to a [global freeze](concept-freeze.html). |
| `tecINSUF_RESERVE_LINE`    | 122   | The transaction failed because the sending account does not have enough CSC to create a new trust line. (See: [Reserves](concept-reserves.html)) This error occurs when the counterparty already has a trust line in a non-default state to the sending account for the same currency. (See `tecNO_LINE_INSUF_RESERVE` for the other case.) |
| `tecINSUF_RESERVE_OFFER`   | 123   | The transaction failed because the sending account does not have enough CSC to create a new Offer. (See: [Reserves](concept-reserves.html)) |
| `tecINSUFFICIENT_RESERVE`  | 141   | The transaction would increase the [reserve requirement](concept-reserves.html) higher than the sending account's balance. [SignerListSet][], [PaymentChannelCreate][], [PaymentChannelFund][], and [EscrowCreate][] can return this error code. See [SignerLists and Reserves](reference-ledger-format.html#signerlists-and-reserves) for more information. |
| `tecINTERNAL`              | 144   | Unspecified internal error, with transaction cost applied. This error code should not normally be returned. If you can reproduce this error, please [report an issue](https://github.com/casinocoin/casinocoind/issues). |
| `tecINVARIANT_FAILED`      | 147   | An invariant check failed when trying to execute this transaction. Requires the [EnforceInvariants amendment](reference-amendments.html#enforceinvariants). If you can reproduce this error, please [report an issue](https://github.com/casinocoin/casinocoind/issues). |
| `tecNEED_MASTER_KEY`       | 142   | This transaction tried to cause changes that require the master key, such as [disabling the master key or giving up the ability to freeze balances](#accountset-flags). |
| `tecNO_ALTERNATIVE_KEY`    | 130   | The transaction tried to remove the only available method of [authorizing transactions](#authorizing-transactions). This could be a [SetRegularKey transaction][] to remove the regular key, a [SignerListSet transaction][] to delete a SignerList, or an [AccountSet transaction][] to disable the master key. (Prior to `casinocoind` 0.30.0, this was called `tecMASTER_DISABLED`.) |
| `tecNO_AUTH`               | 134   | The transaction failed because it needs to add a balance on a trust line to an account with the `lsfRequireAuth` flag enabled, and that trust line has not been authorized. If the trust line does not exist at all, `tecNO_LINE` occurs instead. |
| `tecNO_DST`                | 124   | The account on the receiving end of the transaction does not exist. This includes Payment and TrustSet transaction types. (It could be created if it received enough CSC.) |
| `tecNO_DST_INSUF_CSC`      | 125   | The account on the receiving end of the transaction does not exist, and the transaction is not sending enough CSC to create it. |
| `tecNO_ENTRY`              | 140   | Reserved for future use.                |
| `tecNO_ISSUER`             | 133   | The account specified in the `issuer` field of a currency amount does not exist. |
| `tecNO_LINE`               | 135   | The `TakerPays` field of the [OfferCreate transaction][] specifies an asset whose issuer has `lsfRequireAuth` enabled, and the account making the offer does not have a trust line for that asset. (Normally, making an offer implicitly creates a trust line if necessary, but in this case it does not bother because you cannot hold the asset without authorization.) If the trust line exists, but is not authorized, `tecNO_AUTH` occurs instead. |
| `tecNO_LINE_INSUF_RESERVE` | 126   | The transaction failed because the sending account does not have enough CSC to create a new trust line. (See: [Reserves](concept-reserves.html)) This error occurs when the counterparty does not have a trust line to this account for the same currency. (See `tecINSUF_RESERVE_LINE` for the other case.) |
| `tecNO_LINE_REDUNDANT`     | 127   | The transaction failed because it tried to set a trust line to its default state, but the trust line did not exist. |
| `tecNO_PERMISSION`         | 139   | The sender does not have permission to do this operation. For example, the [EscrowFinish transaction][] tried to release a held payment before its `FinishAfter` time, or someone tried to use [PaymentChannelFund][] on a channel the sender does not own. |
| `tecNO_REGULAR_KEY`        | 131   | The [AccountSet transaction][] tried to disable the master key, but the account does not have another way to [authorize transactions](#authorizing-transactions). If [multi-signing](#multi-signing) is enabled, this code is deprecated and `tecNO_ALTERNATIVE_KEY` is used instead. |
| `tecNO_TARGET`             | 138   | The transaction referenced an Escrow or PayChannel ledger object that doesn't exist, either because it never existed or it has already been deleted. (For example, another [EscrowFinish transaction][] has already executed the held payment.) Alternatively, the destination account has `asfDisallowCSC` set so it cannot be the destination of this [PaymentChannelCreate][] or [EscrowCreate][] transaction. |
| `tecOVERSIZE`              | 145   | This transaction could not be processed, because the server created an excessively large amount of metadata when it tried to apply the transaction. |
| `tecOWNERS`                | 132   | The transaction requires that account sending it has a nonzero "owners count", so the transaction cannot succeed. For example, an account cannot enable the [`lsfRequireAuth`](#accountset-flags) flag if it has any trust lines or available offers. |
| `tecPATH_DRY`              | 128   | The transaction failed because the provided paths did not have enough liquidity to send anything at all. This could mean that the source and destination accounts are not linked by trust lines. |
| `tecPATH_PARTIAL`          | 101   | The transaction failed because the provided paths did not have enough liquidity to send the full amount. |
| `tecUNFUNDED`              | 129   | The transaction failed because the account does not hold enough CSC to pay the amount in the transaction _and_ satisfy the additional reserve necessary to execute this transaction. (See: [Reserves](concept-reserves.html)) |
| `tecUNFUNDED_ADD`          | 102   | **DEPRECATED.**                         |
| `tecUNFUNDED_PAYMENT`      | 104   | The transaction failed because the sending account is trying to send more CSC than it holds, not counting the reserve. (See: [Reserves](concept-reserves.html)) |
| `tecUNFUNDED_OFFER`        | 103   | The [OfferCreate transaction][] failed because the account creating the offer does not have any of the `TakerGets` currency. |


{% include 'snippets/casinocoind_versions.md' %}
{% include 'snippets/tx-type-links.md' %}

[Currency Amount]: reference-casinocoind.html#specifying-currency-amounts
