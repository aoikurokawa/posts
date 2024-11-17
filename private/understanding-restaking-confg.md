# Understanding Restaking Config Accounts

## Introduction
Jito Restaking is a next-generation restaking platform for Solana and SVM environments. I am trying to explain how each account works in Restaking protocol.
In this time, I am explaining Config account.

## Structure of Config account

When we look at [repo], you can see like:

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Pod, Zeroable, AccountDeserialize, ShankAccount)]
#[repr(C)]
pub struct Config {
    /// The configuration admin
    pub admin: Pubkey,

    /// The vault program
    pub vault_program: Pubkey,

    /// The number of NCN managed by the program
    ncn_count: PodU64,

    /// The number of operators managed by the program
    operator_count: PodU64,

    /// The length of an epoch in slots
    epoch_length: PodU64,

    /// The bump seed for the PDA
    pub bump: u8,

    /// Reserved space
    reserved: [u8; 263],
}
```

### Admin

### Vault Program

### NCN Count

### Epoch Length 

### Bump



[repo]: https://github.com/jito-foundation/restaking/blob/4c37d76102496edd784bb25436cb9c4340f0df01/restaking_core/src/config.rs#L12-L37




