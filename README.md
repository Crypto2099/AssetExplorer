# Cardano Wallet Asset Explorer

These documents will seek to provide information relating to building and deploying a Cardano Blockchain Asset Explorer.

## Required Tools

### Cardano Blockchain Database Access

#### Cardano DB Sync

While not necessarily the easiest method, the most robust and flexible option is to install and host your own copy of
the *[cardano-db-sync](https://github.com/input-output-hk/cardano-db-sync)* application which interfaces with a local *
cardano-node* instance to populate a PostgreSQL database with information from the blockchain.

#### Other Alternatives

##### Blockfrost API

[Blockfrost.io](https://blockfrost.io) by Five Binaries presents a RESTful API provided by long-term Cardano community
members. Has many endpoints for querying specific information about particular accounts and/or addresses and their
current UTXO status.

##### Dandelion API

[Dandelion.link](https://dandelion.link) by Gimba Labs offers a variety of APIs and endpoints for interacting with the
Cardano blockchain whether through existing libraries and packages or in-house developed libraries and packages.

## The Process

### Step 1: Identify the User Wallet

This is most easily accomplished using a Shelley-era wallet that is also registered as a staking address. Essentially
the user can provide one of their used receiving addresses. This address specifically can be queried for the presence of
native assets (FTs/NFTs) and can also be linked with other "child" addresses of the account through the link of the
staking address ID.

In the case of a staked Shelley-era wallet, multiple generated addresses (change chain, etc) can be identified through
the use of the "stake address" for the account, allowing assets existing on multiple different change addresses to be
accessed simultaneously.

### Step 2: Identify Assets Attached to the Wallet

In this step, we simply need to query the current UTXO state of the chain for all addresses associated with the address
provided by the user or (preferably) the "stake address ID" in order to compile a list of the assets.

```postgresql
select encode(ma_tx_mint.policy, 'hex')  AS policy_id,
       encode(ma_tx_mint.name, 'escape') AS asset_id,
       ma_tx_out.quantity                AS quantity,
       tx_metadata.json                  AS metadata
from ma_tx_out
         INNER JOIN ma_tx_mint ON ma_tx_mint.policy = ma_tx_out.policy AND ma_tx_mint.name = ma_tx_out.name
         LEFT JOIN tx_metadata ON tx_metadata.tx_id = ma_tx_mint.tx_id
where tx_out_id = any (
    select id
    from utxo_view
    where stake_address_id = (
        select distinct stake_address_id
        from tx_out
        where address =
              'addr1q8wftxz8n7qwwkjy9dkasnxywztpguq32dvg0hwcarj9wrrs7f0ldlv6muesavcc43yx3xa2wgw49hk4eldzcdj5pcrqa3arf5'
    )
       or payment_cred = (
        select distinct payment_cred
        from tx_out
        where address =
              'addr1q8wftxz8n7qwwkjy9dkasnxywztpguq32dvg0hwcarj9wrrs7f0ldlv6muesavcc43yx3xa2wgw49hk4eldzcdj5pcrqa3arf5'
    )
)
```

**Expected Output**

policy_id | asset_id | quantity | metadata|type
----------|----------|----------|---------|----
9bee44c5c494dee38622e6da0b3b312820f9f75dd6cc256c769db788|CryptoKnittie08258|1|{metadata_object}|NFT
d894897411707efa755a76deb66d26dfd50593f2e70863e1661e98a0|spacecoins|1000000|NULL|FT

### Step 3: Parsing Metadata

The final step in displaying the assets is parsing the metadata provided for the asset. Currently, there are several
disparate formats being used by the Cardano Community to define NFT, FT, and Serial NFT assets. The most popular format
currently in use on the chain is the "C721" specification originally used and detailed
by [Alessandro Konrad](https://github.com/alessandrokonrad) (creator of the [Spacebudz](https://spacebudz.io/) series of
NFTs on the Cardano blockchain). This is currently known
as [Draft CIP-85](https://github.com/cardano-foundation/CIPs/pull/85).

#### Draft CIP-85 (C721) Format

In this format, NFT assets are described in the metadata by following fairly closely to the EIP-721 standard. Metadata
for individual tokens is stored in a nested structure under metadata index **721** and then nested by **Policy ID** and
subsequently **Asset ID**.

```json
{
  "721": {
    "[policy_id]": {
      "[asset_id]": {
        "name": "My Cool NFT",
        "image": "ipfs://[ipfsHash]",
        "src": "ipfs://[ipfsHash]",
        "type": "video",
        "description": "This is a description of my asset!"
      }
    }
  }
}
```

#### Legacy CNFT Format

The "CNFT Format" was used originally by the team at [CNFT.io](https://cnft.io) for minting "serialized" or limited-edition
NFT assets in bulk quantities. This can be thought of as being similar to EIP-1155 standards more than 721 Tokens where
multiple tokens share a set of similar properties that are common to all tokens in the set aside from the unique
Asset ID.

```json
{
  "721": {
    "cnftFormat": "0.01",
    "asset": {
      "url": "https://ipfs.io/ipfs/",
      "ipfs": "[ipfsHash]",
      "mimeType": "image/jpeg",
      "assetHash": "[sha256assetHash]",
      "hashType": "sha256",
      "artistName": "Cool NFT Creator Dude",
      "description": "This is a description of my asset!",
      "collectionName": "Cool NFTs Series #0",
      "attributes": {
        "attribute1": "blue",
        "attribute2": [
          "banana",
          "apple"
        ]
      },
      "gallery": [
        {
          "url": "https://ipfs.io/ipfs/",
          "ipfs": "[ipfsHash]",
          "mimeType": "image/jpeg",
          "assetHash": "[sha256assetHash]",
          "hashType": "sha256",
          "media": "print"
        },
        {
          "url": "https://ipfs.io/ipfs/",
          "ipfs": "[ipfsHash]",
          "mimeType": "video/mp4",
          "assetHash": "[sha256assetHash]",
          "hashType": "sha256",
          "media": "video"
        },
        {
          "url": "https://ipfs.io/ipfs/",
          "ipfs": "[ipfsHash]",
          "mimeType": "audio/mp3",
          "assetHash": "[sha256assetHash]",
          "hashType": "sha256",
          "media": "audio"
        }
      ]
    },
    "publisher": [
      "https://tokhun.io"
    ],
    "tokens": {
      "[asset_id]001": "My Cool NFT #001",
      "[asset_id]00n": "My Cool NFT #00n"
    }
  }
}
```


