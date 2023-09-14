---
title: Document Storage
author: Rome Reginelli <rome@ripple.com>, Aanchal Malhotra <amalhotra@ripple.com>
affiliation: Ripple
revision: 4
core_protocol_changes_required: true
---
# Document Storage

Many use cases call for storing arbitrary, unstructured data in the ledger and retrieving it. This might be used for app configurations, identity data, oracles, user storage for play-to-earn games, provenance tracking of objects, and so on.

This proposal defines "CRUD operations" (create, retrieve, update, delete) for arbitrary documents.

- Adds one ledger entry type, Document
- Adds two new transaction types:
    - DocumentSet, which creates or updates a Document
    - DocumentDelete, which deletes a Document
- Extends the `ledger_entry` and `account_object` API methods for getting Document entries from the ledger

These changes require an amendment to the XRP Ledger.

(This standards draft draws heavily on a previous draft of the [XLS-40d: Decentralized Identity](https://github.com/XRPLF/XRPL-Standards/discussions/100) specification.)

## Rationale

Blockchains are fundamentally good at storing data in a manner that is public, highly-available, and cryptographically verifiable. (Changes require a signed transaction and data integrity is verified with cryptographic hashes.) Various blockchain use cases depend on being able to store and retrieve data from the ledger in various formats. This specification aims to support further general purpose use of the XRP Ledger by allowing users to store small chunks of data, called Documents, either directly on-ledger or with an on-ledger reference to data stored off-ledger.

In general, this spec is applicable for anything where **all of the following are true**:

- The XRP Ledger does not validate the integrity or structure of the data.
- The data is owned by one account, which can update or delete it as needed.
- The document is either small enough to store directly on the XRP Ledger, or is stored elsewhere but the XRP Ledger stores the information necessary to look up and verify it.

Non-fungible tokens provide a similar ability to store arbitrary URIs or small data blocks in the ledger. However, NFTs have some properties that are critical to their use cases but other inconvenient for the use cases Document Storage targets.

- NFTs can be traded, sold, or transferred. Documents cannot.
- Documents are mutable. NFTs cannot be changed after minting, at least under the XLS-20 standard. ([XLS-46d: Dynamic NFTs](https://github.com/XRPLF/XRPL-Standards/discussions/130) proposes mutable NFTs.)
- It is possible to directly look up a given type of Document owned by a particular user knowing nothing more than the user account and the type of document. This is not possible with NFTs, which have a sequence that is defined at minting time, so it is not possible for different accounts to use a consistent ID to store equivalent data.

The functionality defined by Document Storage is similar to [Stellar's "Manage Data" operation](https://developers.stellar.org/docs/fundamentals-and-concepts/list-of-operations#manage-data), but the amount and format of data allowed is different. Stellar allows for a 64-byte data field identified by a 64-byte identifying string; Document Storage allows up to two 256-byte data fields identified by a 4-byte identifying integer.

One example of a use case for Document Storage is non-sensitive client application settings. Wallet applications are already highly portable; as long as you know your secret keys, you can use a wallet app from any device and switch at any time. Most of the data about your XRP Ledger account is natively part of the ledger, such as your balances, trust lines, and various account settings. However, there is currently no place on-ledger to store settings that are specific to the client application, which means that switching to a different device means either exporting and re-importing certain settings, or having a separate backend service hosted by the wallet provider. With Document Storage, a client application can define a specific document number to store its settings at, which would be the same for every user, and define/retrieve settings from that Document purely on-ledger. Then, if the user connected using the same static wallet code from any device, they could instantly access their saved settings with no outside service or file needed. Taking it a step further, other apps could read these settings, or multiple separate apps could share settings in a standard format, allowing greater portability and compatibility. This allows many more dApps to have no backend other than the XRP Ledger itself.

Document Storage also has a potential for synergy with other proposed extensions, such as Hooks, which could read or modify Documents beyond what they can do with Hooks' built in state management.

Of course, since all data in the XRP Ledger is public, Document storage is not suited for storing secret, sensitive, or personal information. (Even if it is encrypted, having the data permanently publicly accessible is poor operational security.)

## DocumentSet transaction

This transaction type creates or updates a `Document` ledger entry. In addition to transaction common fields, it has:

| Field | Type | Required? | Description |
|-------|------|-----------|-------------|
| `DocumentNumber` | UInt32 | Yes | An arbitrary number to identify the document to add or modify. You can uniquely identify any Document ledger entry by the pair (`Owner`, `DocumentNumber`). By convention, values of 65535 or less are reserved for "well known" documents and values of 65536 or greater are unreserved. |
| `Data` | VL | No | Arbitrary binary data (hex-formatted in JSON). Limited to 256 bytes or less. |
| `URI` | VL | No | A URI that encodes or locates a document (hex-formatted in JSON). Only bytes that are valid in URIs are allowed. Limited to 256 bytes or less.  |

The `Data` and `URI` fields are collectively called the _data fields_ of the Document. You can specify one, both, or neither of these.

The `URI` field can only contain bytes that correspond to characters valid in URIs, specifically the ASCII values for the following characters: `ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789-._~:/?#[]@!$&'()*+,;=%`. (This is the same restriction that applies to the [`MemoType` and `MemoFormat` fields](https://xrpl.org/transaction-common-fields.html#memos-field).) It is intended, but not required, for the `URI` field to encode a reference to an off-ledger document that is too large to store on-ledger, such as an `ipfs:` URL.

The `Data` field may contain fully arbitrary data.

The transaction creates or updates (aka "upserts") the corresponding Document entry in the ledger with the matching `DocumentNumber`, adding or replacing the `Data` and `URI` fields with the ones provided. Specify an empty `Data` or `URI` field to remove it. (If the field already does not exist in the Document, this has no effect.) This transaction always updates the `PreviousTxnID` and `PreviousTxnLgrSeq` fields of a Document even if it did not make changes to the data fields, so you can use an otherwise no-op transaction to "renew" a document.

The DocumentSet transaction has no specific transaction flags.

## DocumentDelete transaction

This transaction removes a `Document` ledger entry. In addition to transaction common fields, it has the following field:

| Field | Type | Required? | Description |
|-------|------|-----------|-------------|
| `DocumentNumber` | UInt32 | Yes | An arbitrary number to identify the document to delete. |

The DocumentDelete transaction has no specific transaction flags.

## Document ledger entry

A Document ledger entry stores arbitrary data on behalf of a given account.

A Document entry counts as one item for purposes of the owner reserve, and is tracked in an owner directory. Only the owner of an object can update it, although you can use multi-signing to share that power. Each account can have (theoretically) up to 2<sup>32</sup> Document entries, each with a different `DocumentNumber`, but practically speaking the owner reserve puts a stricter limit on how many Document entries one account can afford.

This ledger entry has the following fields:

| Field | Type | Required? | Description |
|-------|------|-----------|-------------|
| `Owner` | AccountID | Yes | The account that owns this data. Only this account has permission to update it. Automatically set to the sender of the `DocumentSet` transaction that created this ledger entry. |
| `DocumentNumber` | UInt32 | Yes | An arbitrary number to identify this entry. By convention, values of 65535 or less are reserved for "well known" documents and values of 65536 or greater are unreserved. |
| `Data` | VL | No | Arbitrary binary data, hex-formatted. Limited to 256 bytes or less. |
| `URI` | VL | No | A URI that encodes or locates the document (hex-formatted in JSON). Limited to 256 bytes or less. |
| `PreviousTxnID` | UInt256 | Yes | Hash of the previous transaction to modify this entry. (Same as on other entries with this field.) |
| `PreviousTxnLgrSeq` | UInt32 | Yes | Ledger index of the ledger when this entry was most recently updated/created. (Same as other entries with this field.) |

The fields `Data` and `URI` are guaranteed to be nonzero length if present.

### Document ID format

The ledger entry ID of a Document object is the SHA-512HAlf of the following values, concatenated in order:

- The Document space key (proposed: `0x0044`)
- The AccountID of the DocumentSet transaction sender (the owner)
- The `DocumentNumber` value

## Well-Known Document Numbers

It some cases, it is helpful to have certain types of documents conventionally stored at a predetermined `DocumentNumber` so that you can look up an account's document of that type without further information. Similar to reserved ports in TCP/IP, we define a range of "reserved" document numbers and a registry (a list to be maintained on xrpl.org) mapping specific numbers to specific types of documents.

The reserved range of `DocumentNumber` values is 0 through 65535 inclusive. To register a given document number for a specific purpose, create an XLS draft and specify the meaning and format of the document type to be stored at that number. The registry will be updated when that XLS draft is accepted.

Values of 65536 or greater are unreserved and may be used for any type of document.

At a protocol level, no rules are enforced regarding the type of document stored at any number.

## API Changes

The `account_objects` method can return `Document` ledger entries. Extend the API method to allow filtering by `"type": "document"` to return only these types of documents. The same applies to the `ledger` and `ledger_data` methods, which can also retrieve arbitrary ledger entries and filter by type.

The `ledger_entry` method can retrieve a specific `Document` ledger entry, similar to how it returns other types. There are two ways to look up a given `Document`:

- Look it up by ID using the existing [Get Ledger Object By ID](https://xrpl.org/ledger_entry.html#get-ledger-object-by-id) syntax (the `index` request field)
- Look it up by the `DocumentNumber` and the account that owns it, using a new `document` request field, which is an **Object** with two nested, required sub-fields as follows:


| Field | Type | Description |
|-------|------|-------------|
| `document.owner` | String - Address | The unique address of the account that owns the document. |
| `document.number` | Number - UInt32 | The `DocumentNumber` of the document to look up. |

This is similar to how you can look up offers, escrows, tickets, and others.

## Size Considerations

At 572 bytes including the bookkeeping fields, the maximum size of a single Document ledger entry is slightly larger than most directly user-editable ledger entries (for comparison, a trust line is between 234 and 250 bytes and a payment channel can be up to 247 bytes if my math is right), but much smaller than a single NFTokenPage entry, which can be over 73,000 bytes if it stores the maximum number of NFTs. Therefore, we consider it appropriate to require a single owner reserve increment per Document object owned in the ledger.

Users can doubtlessly find various ways to store arbitrary data in the ledger, but to discourage wasteful use of resources we explicitly don't define a way of linking multiple Document entries together for storing larger amounts of data. Users can use the data fields to identify, locate, and verify documents which are stored and distributed using another system such as IPFS or even BitTorrent. For example, the `Data` field can contain a hash, the URI field can be a `magnet:` link, and so on.

## Document History

Some use cases may call for examining multiple recent values for a given Document. This is already possible using past ledger versions and transaction metadata. The `PreviousTxnID` and `PreviousTxnLgrSeq` fields can be used to directly look up the previous version of a Document the same way they can for other types of ledger entry with those fields. (Note: most ledger entry types have these fields; the big exception is RippleState, for trust lines. The other types that don't have these fields are not directly user-modifiable: Amendments, DirectoryNode, FeeSettings, LedgerHashes, and NegativeUNL.)

In the edge case where a ledger entry is modified more than once within a single ledger, you must use the transaction metadata (specifically the `FinalFields` of a `ModifiedNode` entry) to look up the intermediate state of the entry. This state exists for only a brief moment while executing the transactions to build the ledger, but it may be necessary to understand the full history of the entry.

A limitation of the `PreviousTxnID` and `PreviousTxnLgrSeq` fields is that the "thread" they create of an entry's history does not go back further than its creation. If an entry was previously deleted and later recreated with the same ID, each instance's history is separate.

It may be useful to add an API method to the server or client libraries for looking up past states of a ledger entry. This would not require an amendment.