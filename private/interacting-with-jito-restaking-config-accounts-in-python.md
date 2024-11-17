# Interacting with Jito Restaking Config Accounts in Python

## Introduction

The Jito Restaking protocol is a cutting-edge framework designed to optimize staking operations. As part of my development efforts, I’ve been building a Python SDK to interact with the protocol, focusing on interacting with Restaking protocol.

In this blog, I'll walk through how to use Python to fetch Restaking Config accounts, highlighting the tools and techniques involved.

## Restaking Config Accounts

Restaking Config accounts are the cornerstone of the Jito Restaking protocol. These accounts store critical configuration data that governs the protocol's operations. When examining the protocol's [repo], we find the following Rust structure representing the Config account:

```rust
/// The global configuration account for the restaking program. Manages
/// program-wide settings and state.
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

These fields are essential for the protocol's configuration, from managing administrators to defining operational parameters.

[repo]: https://github.com/jito-foundation/restaking/blob/4c37d76102496edd784bb25436cb9c4340f0df01/restaking_core/src/config.rs#L12-L37

## Python SDK

Currently, there is no Python SDK for interacting with the Jito Restaking protocol. Therefore, we’ll build a basic library to interact with the Config account.

### Config Class

The Config class models the configuration account and provides methods for deserialization and address discovery:

```python
class Config:
    """
    The global configuration account for the restaking program. Manages program-wide settings and state.

    ...

    Attributes
    ----------
    admin : Pubkey
        The configuration admin

    vault_program : Pubkey
        The vault program

    ncn_count : int
        The number of NCN managed by the program

    operator_count : int
        The number of operators managed by the program
    
    epoch_length : int
        The length of an epoch in slots

    bump : int
        The bump seed for the PDA


    Methods
    -------
    deserialize(data: bytes)
        Deserialize the account data to Config struct

    seeds():
        Returns the seeds for the PDA

    find_program_address(program_id: Pubkey):
        Find the program address for the Config account
    """

    discriminator: typing.ClassVar = 1

    admin: Pubkey
    vault_program:Pubkey

    ncn_count: int
    operator_count: int
    epoch_length: int
    bump: int

    # Initialize a Config instance with required attributes
    def __init__(self, admin: Pubkey, vault_program: Pubkey, ncn_count: int, operator_count: int, epoch_length: int, bump: int):
        self.admin = admin
        self.vault_program = vault_program
        self.ncn_count = ncn_count
        self.operator_count = operator_count
        self.epoch_length = epoch_length
        self.bump = bump
```



### Deserialization

To decode data from the blockchain, we need to manually map the byte data into the Config structure:

```python
    @staticmethod
    def deserialize(data: bytes) -> "Config":
        """Deserializes bytes into a Config instance."""
        
        # Define offsets for each field
        offset = 0
        offset += 8

        # Admin
        admin = Pubkey.from_bytes(data[offset:offset + 32])
        offset += 32

        # Vault program
        vault_program = Pubkey.from_bytes(data[offset:offset + 32])
        offset += 32

        # NCN count
        ncn_count = int.from_bytes(data[offset:offset + 8], byteorder='little')
        offset += 8
        
        # Operator count
        operator_count = int.from_bytes(data[offset:offset + 8], byteorder='little')
        offset += 8

        # Epoch length
        epoch_length = int.from_bytes(data[offset:offset + 8], byteorder='little')
        offset += 8

        # Bump
        bump = int.from_bytes(data[offset:offset + 1])

        # Return a new Config instance with the deserialized data
        return Config(
            admin=admin,
            vault_program=vault_program,
            ncn_count=ncn_count,
            operator_count=operator_count,
            epoch_length=epoch_length,
            bump=bump
        )
```

### Find program address and seeds

To locate the Config account on-chain, we use the Program Derived Address (PDA) mechanism:

```python
    @staticmethod
    def seeds() -> typing.List[bytes]:
        """Return the seeds used for generating PDA."""
        return [b"config"]
    
    @staticmethod
    def find_program_address(program_id: Pubkey) -> typing.Tuple[Pubkey, int, typing.List[bytes]]:
        """Finds the program-derived address (PDA) for the given seeds and program ID."""
        seeds = Config.seeds()
        
        # Compute PDA and bump using seeds (requires solders Pubkey functionality)
        pda, bump = Pubkey.find_program_address(seeds, program_id)
        
        return pda, bump, seeds
```

## Restaking Client

The RestakingClient simplifies RPC communication and integrates `Config` account operations:

```python
class RestakingClient:
    http_client: Client
    restaking_program_id: Pubkey
    vault_program_id: Pubkey


    def __init__(self, url: str, restaking_program_id: Pubkey, vault_program_id: Pubkey):
        self.http_client = Client(url)
        self.restaking_program_id = restaking_program_id
        self.vault_program_id = vault_program_id

    def get_restaking_config(self) -> typing.Optional[Config]:
        config_account_pubkey, _, _ = Config.find_program_address(self.restaking_program_id)

        try:
            response = self.http_client.get_account_info(config_account_pubkey)

            if response.value is None:
                print("Account data not found.")
                return None

            data = response.value.data
            decoded_data = bytes(data)

            config = Config.deserialize(decoded_data)

            return config

        except Exception as e:
            print("An error occurred:", e)
            return None
```

## Test

Testing ensures that deserialization and address discovery function as expected:

```python
from solders.pubkey import Pubkey

from restakingpy.accounts.restaking.config import Config

def test_pubkey_restaking_config():
    expected_pubkey = Pubkey.from_string("4vvKh3Ws4vGzgXRVdo8SdL4jePXDvCqKVmi21BCBGwvn")

    program_id = Pubkey.from_string("RestkWeAVL8fRGgzhfeoqFhsqKRchg6aa1XrcH96z4Q")

    pubkey, _, _ = Config.find_program_address(program_id)

    assert pubkey == expected_pubkey

def test_deserialize_restaking_config():
    config_bytes = bytes([1, 0, 0, 0, 0, 0, 0, 0, 96, 84, 246, 179, 232, 158, 88, 207, 173, 182, 195, 222, 227, 162, 211, 191, 89, 225, 141, 138, 7, 53, 82, 181, 198, 31, 36, 130, 214, 78, 95, 38, 7, 82, 151, 3, 233, 209, 72, 36, 13, 237, 19, 215, 83, 88, 206, 101, 40, 120, 109, 65, 221, 187, 195, 114, 118, 11, 178, 161, 116, 80, 255, 125, 3, 0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 0, 0, 0, 0, 0, 128, 151, 6, 0, 0, 0, 0, 0, 255, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0])

    assert config_bytes[0] == Config.discriminator

    config = Config.deserialize(config_bytes)
    
    assert config.admin == Pubkey.from_string("7V3HKHNgxwxiMLjcgvwPCBey7yy4WJrHUH4JVFmewu1P")
    assert config.vault_program == Pubkey.from_string("Vau1t6sLNxnzB7ZDsef8TLbPLfyZMYXH8WTNqUdm9g8")
    assert config.ncn_count == 3
    assert config.operator_count == 1
    assert config.epoch_length == 432000
    assert config.bump == 255
```

## Example

Here’s an example script to fetch and print the Config account data:

```python
from solders.pubkey import Pubkey

from restakingpy.restaking_client import RestakingClient

RPC_URL = "https://api.devnet.solana.com"
RESTAKING_PROGRAM_ID = "RestkWeAVL8fRGgzhfeoqFhsqKRchg6aa1XrcH96z4Q"
VAULT_PROGRAM_ID = "Vau1t6sLNxnzB7ZDsef8TLbPLfyZMYXH8WTNqUdm9g8"

def main():
    # Replace with your actual config account public key
    restaking_program_id = Pubkey.from_string(RESTAKING_PROGRAM_ID)
    vault_program_id = Pubkey.from_string(VAULT_PROGRAM_ID)

    # Initialize the RPC client
    client = RestakingClient(RPC_URL, restaking_program_id, vault_program_id)

    # Fetch and print the config account
    config_account = client.get_restaking_config()
    if config_account:
        print("Config Account:", config_account)
    else:
        print("Failed to retrieve config account.")

if __name__ == "__main__":
    main()
```

Run the script:

```sh
python examples/get_config.py
```

## Conclusion

Next steps could include adding update functionality or integrating other Restaking accounts. 
