Jito Foundation recently published the [Jito Restaking CLI v1](https://github.com/jito-foundation/restaking/releases/tag/v1.0.0-cli), so I will explore some of basic features in this blog.

## What is Jito Restaking?

Jito Restaking is Solana's innovative restaking protocol that enables users to earn additional rewards by using their staked assets to secure multiple networks simultaneously. 
Following the success of restaking protocols on other blockchains like Ethereum's EigenLayer, Jito brings this capital-efficient technology to the Solana ecosystem.

At its core, Jito Restaking is a multi-asset network capable of leveraging staked base assets such as JitoSOL or any other Solana Program Library (SPL) token. 
The protocol allows these assets to be used as collateral for securing additional applications and services, known as "Node Consensus Network" (NCN).

Key components of Jito Restaking include:

- **Vaults**: These hold staked assets and issue Vault Receipt Tokens (VRTs)
- **VRTs**: Liquid tokens representing staked assets that can be used throughout Solana's DeFi ecosystem
- **Operators**: Entities that participate in consensus networks
- **Node Consensus Networks (NCNs)**: Decentralized networks that leverage the staked assets

The benefits of Jito Restaking include:

- **Enhanced rewards**: Earn additional yield beyond traditional staking rewards
- **Capital efficiency**: Use your assets in multiple protocols simultaneously
- **Maintained liquidity**: Continue using your assets in DeFi through VRTs
- **Flexible infrastructure**: By restaking assets in a Jito vault, participants receive Vault Receipt Tokens (VRTs) that can then be used in DeFi applications, providing additional liquidity and utility without sacrificing staking rewards

What makes Jito Restaking particularly powerful is its flexibility. Unlike other restaking implementations that limit collateral to specific tokens, Jito's implementation allows users to secure services using any chosen crypto asset, creating a more inclusive ecosystem.

## Jito Restaking CLI

To interact with the Jito Restaking Protocol, Jito provides the Jito Restaking CLI, a comprehensive command-line interface that enables users to interact with the protocol directly from their terminal. 

With the CLI, users can manage every aspect of their restaking experience, including vault creation, token minting, delegation management, and monitoring account statuses.

## Key Features

### Complete Protocol Management
- **Full Protocol Support**: Manage NCN, Operator, and Vault operations all from a single tool
- **Token Operations**: Easily mint and burn VRT (Vault Restaking Tokens)
- **Delegation Control**: Delegate to operators and manage your stake with simple commands

### Enhanced User Experience
- **JSON Output**: Generate structured JSON output for all account information, making integration with other tools seamless
- **Transaction Preview**: Use the `--print-tx` flag to inspect transactions before sending them
- **Detailed Monitoring**: Monitor account statuses with comprehensive data outputs

### Security-First Approach
- **Ledger Hardware Wallet Integration**: Connect and sign transactions directly with your Ledger hardware wallet for enhanced security
- **Transaction Inspection**: Preview all transactions before they're sent to ensure they match your expectations

## Command Structure Overview

The Jito Restaking CLI has a well-organized command structure that follows the architecture of the underlying protocol. Commands are divided into two main categories:

### 1. Restaking Commands (`restaking`)
These commands interact with the core restaking program and manage:

- **Configuration**: Global protocol settings
- **NCNs (Node Consensus Networks)**: Decentralized networks that secure applications
- **Operators**: Entities that run validator software for NCNs

### 2. Vault Commands (`vault`)
These commands interact with the vault program and handle:

- **Vault Management**: Create and configure vaults
- **VRT Tokens**: Mint, burn, and manage Vault Receipt Tokens
- **Delegation**: Delegate assets to operators
- **Withdrawal**: Process withdrawals from the vault

Each command follows a consistent pattern:
```
jito-restaking-cli [OPTIONS] <MODULE> <SUBMODULE> <COMMAND> [ARGS]
```

For example:
```bash
jito-restaking-cli --rpc-url <RPC_URL> vault vault initialize <TOKEN_MINT> <DEPOSIT_FEE_BPS> <WITHDRAWAL_FEE_BPS> <REWARD_FEE_BPS> <DECIMALS> <INITIALIZE_TOKEN_AMOUNT>
```

This consistent structure makes the CLI intuitive for users familiar with command-line tools.

## Enhanced Features and Usability

The CLI comes with several features that enhance its usability and integration capabilities:

### JSON Output Options

For automation and integration with other tools, the CLI supports JSON output in different formats:

```bash
# Standard JSON output (filtering out reserved fields)
jito-restaking-cli --rpc-url <RPC_URL> restaking operator get <OPERATOR_ADDRESS> --print-json

# Complete JSON output including reserved fields
jito-restaking-cli --rpc-url <RPC_URL> restaking operator get <OPERATOR_ADDRESS> --print-json-with-reserves
```

This makes it easy to parse data programmatically and integrate the CLI into scripts and other tools.

### Transaction Inspection

Before executing transactions, you can preview them to ensure they're correct:

```bash
jito-restaking-cli --rpc-url <RPC_URL> restaking ncn initialize --print-tx
```

The output shows the transaction structure and instruction data, allowing you to verify the transaction before it's processed.

### Ledger Hardware Wallet Support

For enhanced security, the CLI integrates with Ledger hardware wallets:

```bash
# Use Ledger for signing transactions
jito-restaking-cli vault vault set-admin --old-admin-keypair <OLD_ADMIN_KEYPAIR> --new-admin-keypair usb://ledger?key=0 <VAULT>
```

This feature ensures your private keys remain secure on your hardware device instead of being stored as local keypair files.

## Getting Started

Getting started with Jito Restaking CLI is straightforward:

```bash
# Install the CLI
cargo install jito-restaking-cli

# Verify installation
jito-restaking-cli --help
```

### Quick Example Workflow

Here's a quick look at a basic workflow to create a vault and mint VRT:

```bash
# Initialize a vault
jito-restaking-cli --rpc-url <RPC_URL> vault vault initialize <TOKEN_MINT> 100 100 100 9 1000000000

# Create VRT metadata
jito-restaking-cli --rpc-url <RPC_URL> vault vault create-token-metadata <VAULT> "Jito VRT Token" "JVRT" "https://metadata-url.com"

# Mint some VRT
jito-restaking-cli --rpc-url <RPC_URL> vault vault mint-vrt <VAULT> 5000000000 4900000000
```

### Vault Management Lifecycle

The vault management lifecycle is a crucial process in the Jito Restaking ecosystem. The CLI provides comprehensive commands for each phase:

#### 1. Vault Update Process

Vaults must be updated once per epoch to function properly. The CLI handles this with a three-step process:

```bash
# Step 1: Initialize the update state tracker
jito-restaking-cli --rpc-url <RPC_URL> vault vault initialize-vault-update-state-tracker <VAULT>

# Step 2: Crank the update state tracker for each operator
jito-restaking-cli --rpc-url <RPC_URL> vault vault crank-vault-update-state-tracker <VAULT> <OPERATOR>

# Step 3: Close the update state tracker
jito-restaking-cli --rpc-url <RPC_URL> vault vault close-vault-update-state-tracker <VAULT>
```

#### 2. Withdrawal Process

The withdrawal process also follows a specific sequence:

```bash
# Step 1: Enqueue a withdrawal
jito-restaking-cli --rpc-url <RPC_URL> vault vault enqueue-withdrawal <VAULT> <AMOUNT>

# Step 2: After cooldown period, burn the withdrawal ticket
jito-restaking-cli --rpc-url <RPC_URL> vault vault burn-withdrawal-ticket <VAULT>
```

#### 3. Fee Management

Vault administrators can adjust various fees as needed:

```bash
# Set vault fees
jito-restaking-cli --rpc-url <RPC_URL> vault vault set-fees --deposit-fee-bps 50 --withdrawal-fee-bps 100 --reward-fee-bps 200 <VAULT>

# Set operator fees
jito-restaking-cli --rpc-url <RPC_URL> restaking operator operator-set-fees <OPERATOR> <OPERATOR_FEE_BPS>
```

---


## Conclusion
The Jito Restaking CLI represents a significant advancement in making Solana's restaking ecosystem more accessible to developers and users.
I encourage you to try out the CLI yourself and explore the possibilities it offers. 
As the ecosystem evolves, we can expect to see even more features and improvements to this valuable tool, further enhancing the restaking experience on Solana.


## References
- [Jito Restaking on Solana: The New Chapter](https://www.jito.network/blog/restaking-on-solana-the-new-chapter/)
- [Jito Restaking CLI on crates.io](https://crates.io/crates/jito-restaking-cli)
