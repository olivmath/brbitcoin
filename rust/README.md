# 🇧🇷 BrBitcoin Rust SDK

![hello-world](https://github.com/user-attachments/assets/65597643-95cf-4583-8653-26eb2deb3fc9)

## Design Principles

- **Automatic Zeroization**: Sensitive data wiped from memory using context managers
- **Hierarchical Security**: BIP32/BIP39/BIP44 compliant HD wallets with encrypted backups
- **Network Agnostic**: Unified API for Regtest/Testnet/Mainnet operations
- **Full RPC Access**: Direct Bitcoin Core JSON-RPC integration
- **Type Safety**: Comprehensive type hints for better developer experience

## Features

- 🔐 Secure key management with memory zeroization
- 💳 HD wallet support (BIP32, BIP39, BIP44, BIP84)
- 📡 Multiple network backends (Bitcoin Core, Electrum, Custom)
- 📦 PSBT (Partially Signed Bitcoin Transaction) support
- ⚡️ Async-first architecture for network operations
- 🔄 UTXO management with automatic coin selection
- 📊 Blockchain data inspection utilities
- 🛠️ Low-level Bitcoin script builder

## 📦 Installation

### Via Cargo (Recommended)

```bash
cargo add brbitcoin
```

### Via toml

```toml
[dependencies]
brbitcoin = "0.1"
```

## 🚀 Quick Start

> [!WARNING]
> Always test with REGTEST before MAINNET usage.

### 1. Wallet Management

```rust
use brbitcoin::{Wallet, Network};
use std::env;

// Create random HD wallet (regtest by default)
let wallet = Wallet::create();
println!("Regtest new address: {}", wallet.address());

// Import from existing key
let private_key = "123456789abcdef...";
let wallet = Wallet::from_private_key(private_key, Network::Testnet);
println!("Testnet address: {}", wallet.address());

// Create from BIP39 mnemonic
let mnemonic = "absorb lecture valley scissors giant evolve planet rotate siren chaos";
let wallet = Wallet::from_mnemonic(mnemonic, Network::Mainnet);
println!("Mainnet address: {}", wallet.address());
```

### 2. Blockchain Interaction

#### 2.1 Address Information

```rust
use brbitcoin::{Wallet, Address, get_address_info};

let address = Address::from("bc1q...");

if let Ok(info) = get_address_info(address, Network::Mainnet) {
    println!("Balance: {} satoshis", info.balance);
    println!("UTXOs: {}", info.utxos.len());
} else {
    println!("Failed to retrieve address info.");
}

let wallet = Wallet::new(Network::Testnet);
if let Ok(balance) = wallet.balance() {
    println!("Wallet balance: {} sats", balance);
} else {
    println!("Failed to retrieve wallet balance.");
}
```

#### 2.2 Transaction Inspection

```rust
use brbitcoin::{Wallet, get_transaction};

if let Ok(tx) = get_transaction("aabb...", Network::Regtest) {
    println!("Confirmations: {}", tx.confirmations);
    for output in tx.outputs {
        println!("Output value: {}", output.value);
    }
} else {
    println!("Failed to retrieve transaction.");
}

let wallet = Wallet::from_private_key("beef...", Network::Regtest);
if let Ok(utxos) = wallet.utxos() {
    for utxo in utxos {
        println!("UTXO: {}:{} - {} sats", utxo.txid, utxo.vout, utxo.value);
    }
} else {
    println!("Failed to retrieve UTXOs.");
}
```

#### 2.3 Block Exploration

```rust
use brbitcoin::get_block;

if let Ok(block) = get_block::<Hash>("000000000019d6...", Network::Mainnet) {
    println!("Block height: {}", block.height);
} else {
    println!("Failed to retrieve block information.");
}

if let Ok(genesis) = get_block::<u32>(0, Network::Mainnet) {
    println!("Genesis timestamp: {}", genesis.timestamp);
} else {
    println!("Failed to retrieve genesis block information.");
}
```

### 3. Transaction Construction

#### 3.1 High-Level Transaction Building (Recommended)

```rust
use brbitcoin::{Wallet, Fee};

const receiver: &str = "tb123...";
const amount: u64 = 100_000; // 0.001 BTC

let wallet = Wallet::new();
let txid = wallet.send(receiver, amount).unwrap();

println!("Broadcasted TX ID: {}", txid);
```

#### 3.2 Mid-Level Transaction Control

```rust
use brbitcoin::{Wallet, Transaction};

let receiver = "tb123...";
let amount = 100_000; // Satoshis
let fee = 500; // Satoshis

let wallet = Wallet::new();
if let Ok(utxos) = wallet.utxos() {
    let txid = Transaction::new(wallet.network())
        .add_input(&utxos[0])
        .add_output(receiver, amount)
        .fee(fee)
        .sign(&wallet)
        .broadcast();

    if let Ok(txid) = txid {
        println!("Transação transmitida! TX ID: {}", txid);
    } else {
        eprintln!("Erro ao transmitir transação: {:?}", txid.err());
    }
} else {
    eprintln!("Erro ao obter UTXOs: {:?}", wallet.utxos().err());
}
```

#### 3.3 Low-Level Transaction Scripting

```rust
use brbitcoin::{Wallet, Script, Transaction};

// Create a P2SH lock script
let lock_script = Script::new()
    .push_op_dup()
    .push_op_hash160()
    .push_bytes(&pubkey_hash)
    .push_op_equal_verify()
    .push_op_check_sig();

let wallet = Wallet::new();
if let Ok(utxos) = wallet.utxos() {
    let amount: u64 = 10_000; // 0.0001 BTC

    let txid = Transaction::new(Network::Regtest)
        .add_input(&utxos[0])
        .add_output_script(&lock_script, amount)
        .sign(&wallet)
        .broadcast();

    if let Ok(txid) = txid {
        println!("Transação transmitida! TX ID: {}", txid);
    } else {
        eprintln!("Erro ao transmitir transação: {:?}", txid.err());
    }
} else {
    eprintln!("Erro ao obter UTXOs: {:?}", wallet.utxos().err());
}
```

### 4. Taproot Transactions (BIP340/341/342)

#### 4.1 Generating Taproot Address

```rust
use brbitcoin::{Wallet, TaprootBuilder, Script};

let wallet = Wallet::new(Network::Mainnet).unwrap();
let internal_key = wallet.taproot_internal_key();

let script = Script::new()
    .push_op_hash160()
    .push_bytes(b"my_hash160")
    .push_op_equal();

if let Ok(taproot) = TaprootBuilder::new(internal_key)
    .add_leaf_script(&script)
    .finalize() {
    println!("Taproot Address: {}", taproot.address);
    println!("Control Block: {}", hex::encode(taproot.control_block));
}
```

#### 4.2 Sending to Taproot Address

```rust
let wallet = Wallet::new().unwrap();
const RECEIVER_TAPROOT: &str = "bc1p...";

if let Ok(txid) = Transaction::new(Network::Regtest)
    .add_input(&wallet.utxos().unwrap()[0])
    .add_output_taproot(RECEIVER_TAPROOT, 100_000) // 0.0001 BTC
    .set_change(&wallet.address())
    .estimate_fee()
    .sign(&wallet)
    .broadcast() {
    println!("Taproot TX broadcasted: {}", txid);
} else {
    eprintln!("Erro ao transmitir transação Taproot: {:?}", txid.err());
}
```

#### 4.3 Spending from Taproot (Key Path)

```rust
use brbitcoin::{Wallet, Transaction, Network};

// Spending using Schnorr signature
let wallet = Wallet::from_taproot_internal_key("internal_key_hex");
let utxo = wallet.utxos().unwrap()[0];

let txid = Transaction::new(Network::Mainnet)
    .add_taproot_input(&utxo)
    .add_output("bc1q...", 9000) // 0.009 BTC in satoshis
    .set_change(&wallet.taproot_address())
    .estimate_fee()
    .sign_taproot(&wallet)
    .broadcast()
    .unwrap();

println!("Key path spend TX: {}", txid);
```

#### 4.4 Spending from Taproot (Script Path)

```rust
use brbitcoin::{Wallet, Transaction, Network, Script, TaprootScriptSolution, hash160};

// Reveal script and provide solution
let preimage = b"secret123";
let script = Script::new()
    .push_op_hash160()
    .push_bytes(&hash160(preimage))
    .push_op_equal();

let wallet = Wallet::new().unwrap();
let solution = TaprootScriptSolution {
    script,
    solution_ops: vec![Script::op_push_bytes(), preimage],
};

let txid = Transaction::new(Network::Regtest)
    .add_taproot_input(&wallet.utxos().unwrap()[0], &solution)
    .add_output("bc1q...", 9500) // 0.0095 BTC in satoshis
    .sign_taproot(&wallet)
    .broadcast()
    .unwrap();

println!("Script path spend TX: {}", txid);
```

#### 4.5 Taproot Benefits

- Privacy: All spends look identical on-chain
- Efficiency: Smaller witness size vs traditional multisig
- Flexibility: Combine multiple spending conditions
- Standard: BIP340 (Schnorr), BIP341 (Taproot), BIP342 (Tapscript)

### 5. Security Practices

#### 5.1 Encrypted Private Key Backup

```rust
from brbitcoin import Wallet
import os

PATH = "wallet.json"
PASS = os.environ["WALLET_PASS"]

with Wallet.create() as wallet:
    wallet.export_encrypted(path=PATH, password=PASS)
```

#### 5.2 Restore from Encrypted backup

```rust
from brbitcoin import Wallet


PATH = "wallet.json"
PASS = os.environ["WALLET_PASS"]

with Wallet.from_encrypted(path=PATH, password=PASS) as wallet:
    print(f"Recovered address: {w.address}")
```

#### 5.3 Zeroization Guarantees

```rust
# Keys are wiped:
# - When context manager exits
# - After signing/broadcast
# - On object destruction
with Wallet.from_private_key("c0ffee...") as wallet:
    txid = wallet.send("bc1q...", 0.001)
    # Key no longer in memory here
```

### 6. Node Management

#### 6.1 Network Configuration

```rust
from brbitcoin import NodeClient

# Connect to Bitcoin Core
client = NodeClient(
    network=Network.REGTEST,
    rpc_user="user",
    rpc_password="pass",
    host="localhost",
    port=18443
)
```

#### 6.2 Node Operations

```rust
# Get blockchain info
info = client.get_blockchain_info()
print(f"Blocks: {info.blocks}, Difficulty: {info.difficulty}")

# Generate regtest blocks
if client.network == Network.REGTEST:
    blocks = client.generate_to_address(10, "bcrt1q...")
    print(f"Mined block: {blocks[-1]}")

# Get fee estimates
fees = electrum_client.estimate_fee(targets=[1, 3, 6])
print(f"1-block fee: {fees[1]} BTC/kvB")
```

#### 6.3 Direct RPC Access

```rust
# Raw RPC commands
mempool = client.rpc("getmempoolinfo")
print(f"Mempool size: {mempool['size']}")

# Batch requests
results = client.batch_rpc([
    ("getblockcount", []),
    ("getblockhash", [0]),
    ("getblockheader", ["000000000019d6..."])
])
print(f"Block count: {results[0]}")
```

#### 6.4 Bitcoin Core RPC Command Reference (Partial)

| Category       | Command                | Description                   | Example Usage                                                      |
| -------------- | ---------------------- | ----------------------------- | ------------------------------------------------------------------ |
| **Blockchain** | `getblockchaininfo`    | Returns blockchain state      | `getblockchaininfo`                                                |
|                | `getblock`             | Get block data by hash/height | `getblock "blockhash" 2`                                           |
|                | `gettxoutsetinfo`      | UTXO set statistics           | `gettxoutsetinfo`                                                  |
| **Wallet**     | `listtransactions`     | Wallet transaction history    | `listtransactions "*" 10 0`                                        |
|                | `sendtoaddress`        | Send to Bitcoin address       | `sendtoaddress "addr" 0.01`                                        |
|                | `backupwallet`         | Backup wallet.dat             | `backupwallet "/path/backup.dat"`                                  |
| **Network**    | `getnetworkinfo`       | Network connections/version   | `getnetworkinfo`                                                   |
|                | `addnode`              | Manage peer connections       | `addnode "ip:port" "add"`                                          |
| **Mining**     | `getblocktemplate`     | Get mining template           | `getblocktemplate {"rules":["segwit"]}`                            |
|                | `submitblock`          | Submit mined block            | `submitblock "hexdata"`                                            |
| **Utility**    | `validateaddress`      | Validate address              | `validateaddress "bc1q..."`                                        |
|                | `estimatesmartfee`     | Estimate transaction fee      | `estimatesmartfee 6`                                               |
| **Raw Tx**     | `createrawtransaction` | Create raw transaction        | `createrawtransaction '[{"txid":"...","vout":0}]' '{"addr":0.01}'` |
|                | `signrawtransaction`   | Sign raw transaction          | `signrawtransaction "hex"`                                         |
| **Control**    | `stop`                 | Shut down node                | `stop`                                                             |
|                | `uptime`               | Node uptime                   | `uptime`                                                           |

### 7. Hierarchical Deterministic (HD) Wallets

#### 7.1 Creating HD Wallets (BIP32/BIP44 compliant)

```rust
from brbitcoin import Wallet, Network

with Wallet.create_hd() as hd_wallet:
    first_address = hd_wallet.derive_address(0)
    second_address = hd_wallet.derive_address(1)
    hundredth_address = hd_wallet.derive_address(99)

    print(f"Master xpub: {hd_wallet.xpub}")
    print(f"Derivation path: {hd_wallet.derivation_path}")
    print(f"First Address: {first_address}")
    print(f"Second Address: {second_address}")
    print(f"Hundredth Address : {hundredth_address}")
```

#### 7.2 Advanced Derivation Paths

```rust
# Custom derivation schemes
with Wallet.create_hd(
    purpose=84,  # BIP84 (SegWit)
    network=Network.MAINNET,
    account_index=3,
) as segwit_wallet:
    print(f"Native SegWit address: {segwit_wallet.address}")

# Custom derivation path
with Wallet.create_hd(
    network=Network.MAINNET,
    path="m/44'/0'/1'",
) as hd_wallet:
    print(f"Custom path derivation address: {hd_wallet.derive_address(2)}")
```

#### 7.3 Hardware Wallet Integration

```rust
with Wallet.from_hardware_device(
    device_type="ledger",
    network=Network.MAINNET
) as hw_wallet:
    txid = hw_wallet.send("bc1q...", 0.01)
    print(f"Broadcasted TX ID: {txid}")
```

#### 7.4 HD Wallet Supported Standards

| Standard | Purpose                     | Example Path        |
| -------- | --------------------------- | ------------------- |
| BIP32    | Hierarchical Key Derivation | m/0'/1              |
| BIP39    | Mnemonic Phrase Generation  | 24-word seed        |
| BIP44    | Multi-Account Structure     | m/44'/0'/0'/0/0     |
| BIP84    | Native SegWit (Bech32)      | m/84'/0'/0'/0/0     |
| BIP174   | PSBT (Partially Signed Tx)  | PSBT format support |

---

## [License: MIT](../LICENSE)
