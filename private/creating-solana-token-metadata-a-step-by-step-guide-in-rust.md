## Introduction
Creating and managing tokens on the Solana blockchain has become a fundamental task for developers involved in decentralized finance (DeFi), NFTs, and other blockchain applications. However, the process involves more than just minting tokens; it also requires associating them with metadata that describes key attributes like the token's name, symbol, and a reference to its digital content via a URI.

In this blog, we'll dive deep into a Rust program that accomplishes precisely this—creating a new token mint and its corresponding metadata on the Solana blockchain. We'll walk through the code step by step, explaining the purpose and functionality of each part. Whether you're new to Solana or looking to enhance your Rust skills, this guide will equip you with a solid understanding of how to create and manage token metadata using the powerful combination of Rust and Solana.

By the end of this blog, you'll have a clear grasp of how to leverage the Metaplex Token Metadata program in conjunction with the Solana Program Library (SPL) to build robust and scalable blockchain applications. Let's get started!

## Getting started

### Set up dir

Create a parent directory.

```bash
cargo new --lib vault && cd vault
```

Remove `src` directory.

```bash
rm -rf src
```

Initialize a program(smart contract) directory

```bash
cargo new --lib vault_program
```

Initialize a sdk directory

```bash
cargo new --lib vault_sdk
```

Initialize a integration test directory

```bash
cargo new --lib integration_tests
```

Folder structure like:

```txt
|.
├── integration_tests
|
├── vault_program
│   └── src
|
└── vault_sdk
    └── src
```

### Install dependencies

Write a dependencies that we are going to use in `Cargo.toml`.

```toml
[workspace]
members = [
	"integration_tests",
	"vault_program",
	"vault_sdk"
]

resolver = "2"

[workspace.dependencies]
borsh = { version = "0.10.3" }
solana-sdk = "~1.18"
solana-program = "~1.18"
spl-token = { version = "4.0.0", features = ["no-entrypoint"] }
spl-token-2022 = { version = "3.0.4", features = ["no-entrypoint"] }
vault-program = { path = "vault_program", version = "=0.0.1" }
vault-sdk = { path = "vault_sdk", version = "=0.0.1" }
```

After writing the file, make sure the program can be compiled.

### SDK

vault_sdk
├── Cargo.toml
└── src
    ├── inline_mpl_token_metadata.rs
    ├── instruction.rs
    ├── lib.rs
    └── sdk.rs


Inside SDK, we would use `borsh` and `solana-program`.

```toml
[package]
name = "vault-sdk"
version = "0.0.1"
edition = "2021"

[dependencies]
borsh = { workspace = true }
solana-program = { workspace = true }
```

We have 3 modules in SDK, `inline_token_metadata`, `instruction`, `sdk`. So we write like this in `lib.rs`.

```rust
pub mod inline_mpl_token_metadata;
pub mod instruction;
pub mod sdk;
```

Inside `inline_mpl_token_metadata.rs`, I just copied from [solana program library](https://github.com/solana-labs/solana-program-library/blob/master/stake-pool/program/src/inline_mpl_token_metadata.rs).

```rust
//! Inlined MPL metadata types to avoid a direct dependency on
//! `mpl-token-metadata'

solana_program::declare_id!("metaqbxxUerdq28cj1RbAWkYQm3ybzjb6a8bt518x1s");

pub mod instruction {
    use borsh::{BorshDeserialize, BorshSerialize};
    use solana_program::{
        instruction::{AccountMeta, Instruction},
        pubkey::Pubkey,
    };

    use super::state::DataV2;

    #[derive(BorshSerialize, BorshDeserialize, PartialEq, Eq, Debug, Clone)]
    struct CreateMetadataAccountArgsV3 {
        /// Note that unique metadatas are disabled for now.
        pub data: DataV2,
        /// Whether you want your metadata to be updateable in the future.
        pub is_mutable: bool,
        /// UNUSED If this is a collection parent NFT.
        pub collection_details: Option<u8>,
    }

    #[allow(clippy::too_many_arguments)]
    pub fn create_metadata_accounts_v3(
        program_id: Pubkey,
        metadata_account: Pubkey,
        mint: Pubkey,
        mint_authority: Pubkey,
        payer: Pubkey,
        update_authority: Pubkey,
        name: String,
        symbol: String,
        uri: String,
    ) -> Instruction {
        let mut data = vec![33]; // CreateMetadataAccountV3
        data.append(
            &mut borsh::to_vec(&CreateMetadataAccountArgsV3 {
                data: DataV2 {
                    name,
                    symbol,
                    uri,
                    seller_fee_basis_points: 0,
                    creators: None,
                    collection: None,
                    uses: None,
                },
                is_mutable: true,
                collection_details: None,
            })
            .unwrap(),
        );
        Instruction {
            program_id,
            accounts: vec![
                AccountMeta::new(metadata_account, false),
                AccountMeta::new_readonly(mint, false),
                AccountMeta::new_readonly(mint_authority, true),
                AccountMeta::new(payer, true),
                AccountMeta::new_readonly(update_authority, true),
                AccountMeta::new_readonly(solana_program::system_program::ID, false),
            ],
            data,
        }
    }

    #[derive(BorshSerialize, BorshDeserialize, PartialEq, Eq, Debug, Clone)]
    pub struct UpdateMetadataAccountArgsV2 {
        pub data: Option<DataV2>,
        pub update_authority: Option<Pubkey>,
        pub primary_sale_happened: Option<bool>,
        pub is_mutable: Option<bool>,
    }
    pub fn update_metadata_accounts_v2(
        program_id: Pubkey,
        metadata_account: Pubkey,
        update_authority: Pubkey,
        new_update_authority: Option<Pubkey>,
        metadata: Option<DataV2>,
        primary_sale_happened: Option<bool>,
        is_mutable: Option<bool>,
    ) -> Instruction {
        let mut data = vec![15]; // UpdateMetadataAccountV2
        data.append(
            &mut borsh::to_vec(&UpdateMetadataAccountArgsV2 {
                data: metadata,
                update_authority: new_update_authority,
                primary_sale_happened,
                is_mutable,
            })
            .unwrap(),
        );
        Instruction {
            program_id,
            accounts: vec![
                AccountMeta::new(metadata_account, false),
                AccountMeta::new_readonly(update_authority, true),
            ],
            data,
        }
    }
}

/// PDA creation helpers
pub mod pda {
    use solana_program::pubkey::Pubkey;

    use super::ID;
    const PREFIX: &str = "metadata";
    /// Helper to find a metadata account address
    pub fn find_metadata_account(mint: &Pubkey) -> (Pubkey, u8) {
        Pubkey::find_program_address(&[PREFIX.as_bytes(), ID.as_ref(), mint.as_ref()], &ID)
    }
}

pub mod state {
    use borsh::{BorshDeserialize, BorshSerialize};
    #[repr(C)]
    #[derive(BorshSerialize, BorshDeserialize, PartialEq, Eq, Debug, Clone)]
    pub struct DataV2 {
        /// The name of the asset
        pub name: String,
        /// The symbol for the asset
        pub symbol: String,
        /// URI pointing to JSON representing the asset
        pub uri: String,
        /// Royalty basis points that goes to creators in secondary sales
        /// (0-10000)
        pub seller_fee_basis_points: u16,
        /// UNUSED Array of creators, optional
        pub creators: Option<u8>,
        /// UNUSED Collection
        pub collection: Option<u8>,
        /// UNUSED Uses
        pub uses: Option<u8>,
    }
}
```

Define the instruction in `instruction.rs`.

```rust
use borsh::{BorshDeserialize, BorshSerialize};

#[derive(Debug, BorshSerialize, BorshDeserialize)]
pub enum VaultInstruction {
    CreateTokenMetadata {
        name: String,
        symbol: String,
        uri: String,
    },
}
```

When we send the transaction, we need the interface.

```rust
use borsh::BorshSerialize;
use solana_program::{
    instruction::{AccountMeta, Instruction},
    pubkey::Pubkey,
    system_program,
};

use crate::instruction::VaultInstruction;

#[allow(clippy::too_many_arguments)]
pub fn create_token_metadata(
    program_id: &Pubkey,
    mint_account: &Pubkey,
    mint_authority: &Pubkey,
    metadata: &Pubkey,
    payer: &Pubkey,
    token_program_id: &Pubkey,
    name: String,
    symbol: String,
    uri: String,
) -> Instruction {
    let accounts = vec![
        AccountMeta::new(*mint_account, true),
        AccountMeta::new_readonly(*mint_authority, true),
        AccountMeta::new(*metadata, false),
        AccountMeta::new(*payer, true),
        AccountMeta::new_readonly(*token_program_id, false),
        AccountMeta::new_readonly(crate::inline_mpl_token_metadata::id(), false),
        AccountMeta::new_readonly(system_program::id(), false),
    ];

    Instruction {
        program_id: *program_id,
        accounts,
        data: VaultInstruction::CreateTokenMetadata { name, symbol, uri }
            .try_to_vec()
            .unwrap(),
    }
}
```

### Smart contract

```txt
vault_program/
├── Cargo.toml
└── src
    ├── create_token_metadata.rs
    └── lib.rs
```

We are going to use bunch of dependencies.

```toml
[package]
name = "vault-program"
version = "0.0.1"
edition = "2021"

[lib]
crate-type = ["cdylib", "lib"]
name = "vault_program"

[features]
no-entrypoint = []
no-idl = []
no-log-ix-name = []
cpi = ["no-entrypoint"]
default = []

[dependencies]
borsh = { workspace = true }
solana-program = { workspace = true }
spl-token = { workspace = true }
spl-token-2022 = { workspace = true }
vault-sdk = { workspace = true }
```

Inside `lib.rs`:

```rust
mod create_token_metadata;

use borsh::BorshDeserialize;
use create_token_metadata::process_create_token_metadata;
use solana_program::{
    account_info::AccountInfo, declare_id, entrypoint::ProgramResult, msg,
    program_error::ProgramError, pubkey::Pubkey,
};
use vault_sdk::instruction::VaultInstruction;

declare_id!("AE7fSUJSGxMzjNxSPpNTemrz9cr26RFue4GwoJ1cuR6f");

#[cfg(not(feature = "no-entrypoint"))]
solana_program::entrypoint!(process_instruction);

pub fn process_instruction(
    program_id: &Pubkey,
    accounts: &[AccountInfo],
    instruction_data: &[u8],
) -> ProgramResult {
    if program_id.ne(&id()) {
        return Err(ProgramError::IncorrectProgramId);
    }

    let instruction = VaultInstruction::try_from_slice(instruction_data)?;

    match instruction {
        VaultInstruction::CreateTokenMetadata { name, symbol, uri } => {
            msg!("Instruction: CreateTokenMetadata");
            process_create_token_metadata(program_id, accounts, name, symbol, uri)?;
        }
    }

    Ok(())
}
```

Inside `create_token_metadata.rs`:

```rust
use solana_program::{
    account_info::AccountInfo, entrypoint::ProgramResult, msg, program::invoke,
    program_error::ProgramError, program_pack::Pack, pubkey::Pubkey, rent::Rent,
    system_instruction, sysvar::Sysvar,
};
use vault_sdk::inline_mpl_token_metadata::instruction::create_metadata_accounts_v3;

pub fn process_create_token_metadata(
    _program_id: &Pubkey,
    accounts: &[AccountInfo],
    name: String,
    symbol: String,
    uri: String,
) -> ProgramResult {
    let [mint_account, mint_authority, metadata, payer, token_program, mpl_token_metadata_program, system_program] =
        accounts
    else {
        return Err(ProgramError::NotEnoughAccountKeys);
    };

    // First create the account for the Mint
    //
    msg!("Creating mint account...");
    msg!("Mint: {}", mint_account.key);
    invoke(
        &system_instruction::create_account(
            payer.key,
            mint_account.key,
            (Rent::get()?).minimum_balance(spl_token_2022::state::Mint::LEN),
            spl_token_2022::state::Mint::LEN as u64,
            token_program.key,
        ),
        &[
            mint_account.clone(),
            payer.clone(),
            system_program.clone(),
            token_program.clone(),
        ],
    )?;

    // Now initialize that account as a Mint (standard Mint)
    //
    msg!("Initializing mint account...");
    msg!("Mint: {}", mint_account.key);
    invoke(
        &spl_token_2022::instruction::initialize_mint2(
            token_program.key,
            mint_account.key,
            mint_authority.key,
            None,
            9,
        )?,
        &[mint_account.clone()],
    )?;

    msg!("Creating metadata account...");
    msg!("Metadata account address: {}", metadata.key);

    let new_metadata_instruction = create_metadata_accounts_v3(
        *mpl_token_metadata_program.key,
        *metadata.key,
        *mint_account.key,
        *mint_authority.key,
        *payer.key,
        *mint_authority.key,
        name,
        symbol,
        uri,
    );

    invoke(
        &new_metadata_instruction,
        &[
            metadata.clone(),
            mint_account.clone(),
            mint_authority.clone(),
            payer.clone(),
            system_program.clone(),
        ],
    )?;

    Ok(())
}
```

### Integration Tests

We just finished writing a program that initialize mint account and token metadata account.
Let's check the program is working properly.

Before writing this blog, I was curious about [LiteSVM](https://github.com/LiteSVM/litesvm). So we are going to use it.

```txt
integration_tests/
├── Cargo.toml
└── tests
    ├── fixtures
    │   └── mpl_token_metadata.so
    ├── helpers
    │   ├── mod.rs
    │   └── token.rs
    ├── tests.rs
    └── vault
        ├── create_token_metadata.rs
        └── mod.rs
```

```toml
[package]
name = "integration_tests"
version = "0.1.0"
edition = "2021"

[dependencies]

[dev-dependencies]
borsh = { workspace = true }
litesvm = "0.1.0"
solana-program = { workspace = true }
solana-sdk = { workspace = true }
spl-token = { workspace = true }
spl-token-2022 = { workspace = true }
vault-program = { workspace = true }
vault-sdk = { workspace = true }
```

Since we are going to use built-in program `mpl_token_metadata` program.
So we will download the so file from [spl repo](https://github.com/solana-labs/solana-program-library/blob/master/stake-pool/program/tests/fixtures/mpl_token_metadata.so).
Then, move the file under fixture folder.

```bash
mv {place you downloaded}/mpl_token_metadata.so integration_tests/fixtures
```

Inside `tests.rs`, define 2 modules.

```rust
mod helpers;
mod vault;
```

#### Helpers

```rust
pub mod token;
```

```rust
use borsh::BorshDeserialize;
use solana_program::pubkey::Pubkey;

#[derive(Clone, BorshDeserialize, Debug, PartialEq, Eq)]
pub struct Metadata {
    pub key: u8,
    pub update_authority: Pubkey,
    pub mint: Pubkey,
    pub name: String,
    pub symbol: String,
    pub uri: String,
    pub seller_fee_basis_points: u16,
    pub creators: Option<Vec<u8>>,
    pub primary_sale_happened: bool,
    pub is_mutable: bool,
}
```

#### Write a test code  

`mod.rs`

```rust
mod create_token_metadata;
```

`create_token_metadata.rs`

```rust
#[cfg(test)]
mod tests {
    use std::path::PathBuf;

    use borsh::BorshDeserialize;
    use litesvm::LiteSVM;
    use solana_sdk::{
        native_token::LAMPORTS_PER_SOL, signature::Keypair, signer::Signer,
        transaction::Transaction,
    };
    use vault_sdk::inline_mpl_token_metadata;

    fn read_vault_program() -> Vec<u8> {
        let mut so_path = PathBuf::from(env!("CARGO_MANIFEST_DIR"));
        so_path.push("../target/deploy/vault.so");
        println!("SO path: {:?}", so_path.to_str());
        std::fs::read(so_path).unwrap()
    }

    fn read_mpl_token_metadata_program() -> Vec<u8> {
        let mut so_path = PathBuf::from(env!("CARGO_MANIFEST_DIR"));
        so_path.push("tests/fixtures/mpl_token_metadata.so");
        println!("SO path: {:?}", so_path.to_str());
        std::fs::read(so_path).unwrap()
    }

    #[test]
    fn test_create_token_metadata_ok() {
        let mut svm = LiteSVM::new();
        svm.add_program(vault_program::id(), &read_vault_program());
        svm.add_program(
            inline_mpl_token_metadata::id(),
            &read_mpl_token_metadata_program(),
        );

        let payer_kp = Keypair::new();
        let payer_pk = payer_kp.pubkey();

        svm.airdrop(&payer_pk, 10 * LAMPORTS_PER_SOL).unwrap();

        let mint_account = Keypair::new();

        // Create token metadata
        let name = "restaking JTO";
        let symbol = "rJTO";
        let uri = "https://www.jito.network/restaking/";

        let metadata_pubkey =
            inline_mpl_token_metadata::pda::find_metadata_account(&mint_account.pubkey()).0;

        let ix = vault_sdk::sdk::create_token_metadata(
            &vault_program::id(),
            &mint_account.pubkey(),
            &payer_pk,
            &metadata_pubkey,
            &payer_pk,
            &spl_token::id(),
            name.to_string(),
            symbol.to_string(),
            uri.to_string(),
        );

        let blockhash = svm.latest_blockhash();
        let tx = Transaction::new_signed_with_payer(
            &[ix],
            Some(&payer_pk),
            &[&mint_account, &payer_kp],
            blockhash,
        );

        let tx_result = svm.send_transaction(tx);
        assert!(tx_result.is_ok());

        let token_metadata_account = svm.get_account(&metadata_pubkey).unwrap();
        let metadata = crate::helpers::token::Metadata::deserialize(
            &mut token_metadata_account.data.as_slice(),
        )
        .unwrap();

        assert!(metadata.name.starts_with(name));
        assert!(metadata.symbol.starts_with(symbol));
        assert!(metadata.uri.starts_with(uri));
    }
}
```

## Resources
- https://github.com/solana-developers/program-examples
- https://medium.com/@LeoOnSol/solana-program-test-your-local-validator-on-steroids-5b991d035441
