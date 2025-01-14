- Sui v0.9.0

# Install

This codebase requires installation of both the Sui CLI ant the Sui source code. The latter has to be present in the parent folder of the repository.

- Install the [Sui CLI](https://docs.sui.io/build/install)
- `git clone https://github.com/MystenLabs/sui.git --branch devnet` to download the current sui develop branch source code

# Built and Test

1. `$ sui move build` to build the available move modules
2. `$ sui move test` to run the move tests
3. `./bin/publish.sh` to publish the modules on localnet or devnet

# OriginByte

A new approach to NFTs.

Origin-Byte is an ecosystem of tools, standards and smart contracts designed to make life easier for Web3 Game Developers and NFT creators. From simple artwork to complex gaming assets, we want to help you reach the public, and provide on-chain market infrastructure.


This repository currently contains the first version of OriginByte NFT and collection framework. It comprises four modules, of which two are generic and two domain-specific:

Generic modules:
- `nft.move` as generic NFT module
- `collection.move` as generic collection module
- `slingshot.move` as generic slingshot launchpad module

Domain-specific modules:
- `std_nft.move` as a standard NFT metadata implementation
- `std_collection.move` as a standard Collection metadata implementation
- `fixed_price.move` as fixed price launchpad configuration

## Minting an NFT Collection

Conceptually, NFTs are organized in NFT collections. To mint an NFT, projects must first create the NFT collection object, where metadata and configurations about the project will be stored. The NFT collection objects are meant to be owned by the project owners, who maintain control over the collection and its NFTs while the collection is mutable. At any point in time, the collection owner can decide to make the collection immutable which involves freezing the collection object and its associated NFTs. However, not all fields of the Collection are frozen:

- The field current_supply will still mutate every time an NFT is minted or burned
- Collection owners will still be able to push and pop tags onto the field tags

Once the collection object is created via the domain-specific collection module, you can use the domain-specific NFT module to mint the NFTs. All domain-specific NFT objects will be guaranteed to have the field collection_id if they are implemented with the generic module. This field acts as a pointer to the collection object and allows us to build a permissioning behaviour.

When minting an NFT, you need to pass on a mutable reference to the Collection object. This means that only the collection owner can perform the initial mint, unless it is a shared-object in which case anybody can or anyone can. This collection ownership pattern is useful since many projects will want to transfer NFTs to a Launchpad as part of the initial mint process. Therefore we expose three methods:

- `mint_and_transfer` to mint to an address
- `mint_to_launchpad` to mint to a `Slingshot<T, Config>` object


Let us now describe the `Collection` and `CollectionMeta` objects, instantiated by the `collection` and `std_collection` modules, as well as the `NftOwned` and `NftMeta` objects, instantiated by the `nft` and `std_nft` modules.

### Collection

The collection object, `Collection<phantom T, Meta>`, has the following data model:

| Field            | Type          | Description |
| ---------------- | ------------- | ----------- |
| `id`             | `UID`         | The UID of the collection object |
| `name`           | `String`      | The name of the collection |
| `symbol`         | `String`      | The symbol/ticker of the collection |
| `current_supply` | `u64`         | The current number of instantiated NFT objects for this collection |
| `total_supply`   | `u64`         | The maximum number of NFT objects instantiated at any given time for this collection |
| `initial_price`  | `u64`         | Initial mint price in Sui |
| `receiver`       | `address`     | Address that receives the mint price in Sui |
| `tags`           | `Tags`        | A set of strings that categorize the domain in which the NFT operates |
| `is_mutable`     | `bool`        | A configuration field that dictates whether NFTs are mutable |
| `metadata`       | `Meta`        | A generic type representing the metadata object embedded in the NFT collection |

Where `Tags` is a struct with the field `enumerations` as a `VecMap<u64, String>` being the set of strings representing the domains of the NFT (e.g. Art, Gaming Asset, Tickets, Loyalty Points, etc.)

The collection object has the following functions that mutate state:

- `create` which mints a collection object and returns it
- `init_args` returns `InitCollection` struct from the inputs which acts as input to `create` function
- `increase_supply` which increments `current_supply` by one 
- `decrease_supply` which decreases `current_supply` by one
- `burn` which burns the collection object if `current_supply` is zero

and the following modifier functions:
- `rename`
- `change_symbol`
- `change_total_supply`
- `change_initial_price`
- `change_receiver`
- `freeze_collection` (irreversible)
- `push_tag`

and the following getter functions:
- `name`
- `symbol`
- `current_supply`
- `total_supply`
- `tags`
- `metadata`
- `metadata_mut` returns a mutable reference to metadata object
- `initial_price`
- `receiver`
- `is_mutable`
- `id`
- `id_ref`


### Standard Collection Metadata

The standard collection metadata object, `CollectionMeta`, has the following data model:

| Field             | Type              | Description |
| ----------------- | ----------------- | ----------- |
| `id`              | `UID`             | The UID of the standard collection metadata object |
| `royalty_fee_bps` | `u64`             | The royalty fees creators accumulate on the sale of NFTs * |
| `creators`        | `vector<Creator>` | A vector containing the information of the creators |
| `data`            | `String`          | An open string field to add any arbitrary data |

* `royalty_fee_bps` is currently not being utilized but will be used in the standard launchpad module.

Where `Creators` is a struct with the following fields:
- `id` representing the address of the creator
- `verified` which is a bool value that represents if the creator address has been verified via signed transaction (this functionality is still not implemented)
- `share_of_royalty` as the percentage share that the creator has over `royalty_fee_bps`.

The collection metadata object has the following functions that mutate state:

- `mint_and_transfer` mints a collection object, it's corresponding metadata object, and transfers it to a recipient
- `mint_and_share` mints a collection object, it's corresponding metadata object, and makes it a shared object
- `add_creator` pushes a `Creator` to the `creators` field
- `remove_creator` pops a `Creator` from the `creators` field
- `change_royalty` changes the field `royalty_fee_bps`
- `burn` which burns the collection object and subsequently the metadata object

and the following getter functions:
- `royalty`
- `creators`
- `data`


### NFT

Generic NFT object, `NftOwned<phantom T, Meta>`, has the following data model:

| Field             | Type              | Description |
| ----------------- | ----------------- | ----------- |
| `id`              | `UID`             | The UID of the generic NFT object |
| `collection_id`   | `ID`              | A pointer to the collection object |
| `metadata`        | `u64`             |  A generic type representing the metadata object embedded in the NFT |

The generic NFT object has the following init and drop functions:
- `create_owned` to create the NFT (called by the Domain-specific module)
- `destroy_owned` to destroy the NFT (called by the Domain-specific module) 

the following modifier functions:
- `owned_metadata_mut`

and the following getter functions:

- `owned_metadata`
- `uid_ref`
- `id`
- `id_ref`
- `collection_id`

### NFT Standard Metadata

Standard NFT metadata object, `NftMeta`, has the following data model:

| Field             | Type              | Description |
| ----------------- | ----------------- | ----------- |
| `id`              | `UID`             | The UID of the NFT metadata object |
| `name`            | `String`          | Name of the NFT object |
| `index`           | `u64`             | The index of the NFT in relation to the whole collection |
| `url`             | `Url`             | The URL of the NFT |
| `attributes`      | `Attributes`      | Attributes of a given NFT |


Where `Attributes` is a struct with the field `keys`, the attribute keys represented as string vector, in other words the set of traits (e.g. Hat, Color of T-shirt, Fur type, etc.) and NFT has, and `values` of such traits (e.g. Straw Hat, White T-shirt, Blue Fur, etc.).

The standard NFT metadata object has the following init and drop functions:
- `mint_and_transfer` which mint and NFT and transfer it to a recipient. Currently, the only way to mint the NFT is if the call is made by the Collection owner (or by anyone if the collection is a shared object). This last option allows for anyone to mint their desired metadata which is suboptimal.

For the time being we are allowing for this so that marketplaces can allow users to mint devnet NFT mints from temporary collections. However, we plan to deprecate this as soon as we deploy the launchpad module, which will allow collection owners to mint and transfer the NFT to a launchpad object which will configure the primary market sale strategy.
- `burn` which burns the NFT

and the following getter functions:

- `name`
- `index`
- `url`
- `attributes`

### Slingshot Launchpad

The Slingshot object, `Slingshot<phantom T, Config>`, has the following data model:

| Field             | Type              | Description |
| ----------------- | ----------------- | ----------- |
| `id`              | `UID`             | The UID of the Slingshot object |
| `collection_id`   | `ID`              | The ID of the NFT Collection object |
| `live`            | `bool`            | Boolean indicating if the sale is live |
| `admin`           | `address`         | The address of the administrator |
| `receiver`        | `address`         | The address of the receiver of funds |
| `nfts`            | `vector<ID>`      | Vector of all IDs owned by the slingshot |
| `config`          | `Config`          | Config object |

It has the following init and drop functions:
- `create` to create the Slingshot launchpad (called by the Domain-specific module)
- `delete` to destroy the Slingshot launchpad (called by the Domain-specific module) 

the following transfer functions:
- `transfer_back` which allows the administrator to transfer back nfts from the Slingshot to a recipient address

the following modifier functions:
- `add_nft` (called by the NFT contract when an NFT is transferred to the Slingshot)
- `pop_nft` (called by the Domain-specific launchpad module when NFTs are transferred out of the Slingshot)
- `sale_on` to start the sale
- `sale_off` to pause or stop the sale
- `config_mut` returns a mutable reference to the upstream module

and the following getter functions:

- `collection_id`
- `collection_id_ref`
- `live`
- `config`
- `receiver`
- `admin`
- `nfts`


### Slingshot: Fixed Price Launchpad

The Fixed Price Launchpad config, `LaunchpadConfig`, has the following data model:

| Field             | Type              | Description |
| ----------------- | ----------------- | ----------- |
| `id`              | `UID`             | The UID of the Launchpad configuration object |
| `price`           | `u64`             | The fixed price of each NFT in the collection |

It has the following init and drop functions:
- `create` to create the Launchpad
- `delete` to destroy the Launchpad

the following transfer functions:
- `buy_nft_certificate` when a user wants to buy an NFT, they must first buy the NFT certificate which contains the ID of the NFT they can claim
- `claim_nft` after buying the NFT certificate, the user can then call this endpoint to redeem the allocated NFT

the following modifier functions:
- `new_price` (permissioned endpoint for administrator to call and change the sale price)

and the following getter functions:
- `price`