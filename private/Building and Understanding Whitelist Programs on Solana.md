## Introduction

In the fast-paced world of blockchain and decentralized applications, ensuring fair access to exclusive events like NFT minting, token sales, or staking programs has become increasingly important. This is where whitelist programs come into play. A whitelist acts as a pre-approval system, allowing only selected wallet addresses to participate in specific activities. By implementing a whitelist, projects can mitigate common challenges like bot attacks, gas wars, and overloaded transactions, creating a smoother and fairer user experience.
In this blog, we’ll explore:

- Step-by-step guidance on implementing a whitelist using Solana programs

By the end, you’ll have a clear understanding of how to create a robust whitelist program on Solana, complete with practical examples and code snippets. Let’s get started!

## Implement a Whitelist Program on Solana

### Structure

To implement a whitelist program on Solana, we will design it with a modular structure that includes three core components: Core, Program, and SDK.

#### 1. Core: Define the accounts

The Core is where we define the data structures (accounts) that store essential information for the whitelist program. On Solana, accounts are used to hold on-chain data, and their structure is crucial for program functionality.

We have 2 main accounts in whitelist program.

- Whitelist
- WhitelistEntry

##### Whitelist

```rust
/// The "base" whitelist account upon which all whitelist entry account addresses are derived
#[derive(Debug, Clone, Copy, PartialEq, Eq, Pod, Zeroable, AccountDeserialize, ShankAccount)]
#[repr(C)]
pub struct Whitelist {
    // The account that created this whitelist
    pub admin: Pubkey,
}
```

##### WhitelistEntry

```rust
/// a PDA derived from the address of the account to add and the base whitelist
/// defined in create_whitelist::Whitelist
///
/// Checking if an account address X is whitelisted in whitelist Y
/// involves checking if a WhitelistEntry exists whose address is derived from X and Y
#[derive(Debug, Clone, Copy, PartialEq, Eq, Pod, Zeroable, AccountDeserialize, ShankAccount)]
#[repr(C)]
pub struct WhitelistEntry {
    /// The base whitelist account that this entry is derived from
    pub parent: Pubkey,

    /// The address that this entry whitelists
    pub whitelisted: Pubkey,

    /// Rate limiting
    pub rate_limiting: PodU64,
}
```


#### 2. Program: Define the instructions

The Program component handles the business logic and defines instructions (functions) that operate on the Core accounts. Instructions dictate what actions can be performed, such as adding users to the whitelist, removing users, or verifying whitelisted status.

We have 4 main instructions:

- InitializeWhitelist
- AddToWhitelist
- CheckWhitelisted
- RemoveFromWhitelist

##### InitializeWhitelist

```rust
use jito_bytemuck::{AccountDeserialize, Discriminator};
use jito_jsm_core::{
    create_account,
    loader::{load_signer, load_system_account, load_system_program},
};
use ncn_portal_core::whitelist::Whitelist;
use ncn_portal_sdk::error::NcnPortalError;
use solana_program::{
    account_info::AccountInfo, entrypoint::ProgramResult, msg, program_error::ProgramError,
    pubkey::Pubkey, rent::Rent, sysvar::Sysvar,
};

pub fn process_initialize_whitelist(
    program_id: &Pubkey,
    accounts: &[AccountInfo],
) -> ProgramResult {
    let [whitelist, admin, system_program] = accounts else {
        return Err(ProgramError::NotEnoughAccountKeys);
    };

    load_system_account(whitelist, true)?;
    load_signer(admin, true)?;
    load_system_program(system_program)?;

    // The whitelist account shall be at the canonical PDA
    let (whitelist_pubkey, whitelist_bump, mut whitelist_seeds) =
        Whitelist::find_program_address(program_id, admin.key);
    whitelist_seeds.push(vec![whitelist_bump]);
    if whitelist_pubkey.ne(whitelist.key) {
        msg!("Whitelist account is not at the correct PDA");
        return Err(ProgramError::InvalidAccountData);
    }

    msg!("Initializing whitelist at address {}", whitelist.key);
    create_account(
        admin,
        whitelist,
        system_program,
        program_id,
        &Rent::get()?,
        8_u64
            .checked_add(size_of::<Whitelist>() as u64)
            .ok_or(NcnPortalError::ArithmeticOverflow)?,
        &whitelist_seeds,
    )?;

    let mut whitelist_data = whitelist.try_borrow_mut_data()?;
    whitelist_data[0] = Whitelist::DISCRIMINATOR;
    let whitelist = Whitelist::try_from_slice_unchecked_mut(&mut whitelist_data)?;
    *whitelist = Whitelist::new(*admin.key);

    Ok(())
}
```

##### AddToWhitelist

```rust
use jito_bytemuck::{AccountDeserialize, Discriminator};
use jito_jsm_core::{
    create_account,
    loader::{load_signer, load_system_account, load_system_program},
};
use ncn_portal_core::{whitelist::Whitelist, whitelist_entry::WhitelistEntry};
use ncn_portal_sdk::error::NcnPortalError;
use solana_program::{
    account_info::AccountInfo, entrypoint::ProgramResult, msg, program_error::ProgramError,
    pubkey::Pubkey, rent::Rent, sysvar::Sysvar,
};

pub fn process_add_to_whitelist(
    program_id: &Pubkey,
    accounts: &[AccountInfo],
    rate_limiting: u64,
) -> ProgramResult {
    let [whitelist_info, whitelist_entry_info, whitelisted_info, admin_info, system_program] =
        accounts
    else {
        return Err(ProgramError::NotEnoughAccountKeys);
    };

    Whitelist::load(program_id, admin_info.key, whitelist_info, true)?;
    let whitelist_data = whitelist_info.data.borrow();
    let whitelist = Whitelist::try_from_slice_unchecked(&whitelist_data)?;

    whitelist.check_admin(admin_info.key)?;

    load_system_account(whitelist_entry_info, true)?;
    load_signer(admin_info, true)?;
    load_system_program(system_program)?;

    // The whitelist entry account shall be at the canonical PDA
    let (whitelist_entry_pubkey, whitelist_entry_bump, mut whitelist_entry_seeds) =
        WhitelistEntry::find_program_address(program_id, whitelist_info.key, whitelisted_info.key);
    whitelist_entry_seeds.push(vec![whitelist_entry_bump]);
    if whitelist_entry_pubkey.ne(whitelist_entry_info.key) {
        msg!("Whitelist entry account is not at the correct PDA");
        return Err(ProgramError::InvalidAccountData);
    }

    msg!(
        "Initializing whitelist entry at address {}",
        whitelist_entry_info.key
    );
    create_account(
        admin_info,
        whitelist_entry_info,
        system_program,
        program_id,
        &Rent::get()?,
        8_u64
            .checked_add(size_of::<WhitelistEntry>() as u64)
            .ok_or(NcnPortalError::ArithmeticOverflow)?,
        &whitelist_entry_seeds,
    )?;

    let mut whitelist_entry_data = whitelist_entry_info.try_borrow_mut_data()?;
    whitelist_entry_data[0] = Whitelist::DISCRIMINATOR;
    let whitelist_entry = WhitelistEntry::try_from_slice_unchecked_mut(&mut whitelist_entry_data)?;
    *whitelist_entry =
        WhitelistEntry::new(*whitelist_info.key, *whitelisted_info.key, rate_limiting);

    Ok(())
}
```

##### CheckWhitelisted

```rust

```

##### RemoveFromWhitelist

```rust

```



### 3. SDK: Define the instructions and errors.

The SDK (Software Development Kit) simplifies interaction with the Solana program by providing a set of pre-defined instructions, error handling, and helper functions. This allows developers to integrate the whitelist program into their applications seamlessly.


#### Instructions


#### Errors

