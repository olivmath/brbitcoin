# 🇧🇷 BrBitcoin Python SDK

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

### Via Poetry (Recommended)

```bash
poetry add brbitcoin
```

### Via pip

```bash
pip install brbitcoin
```

## 🚀 Quick Start

> [!WARNING]
> Always test with REGTEST before MAINNET usage.

### 1. Wallet Management

```python
from brbitcoin import Wallet, Network
import os

# Create random HD wallet (testnet by default)
with Wallet.create() as wallet:
    print(f"Testnet new address: {wallet.address}")
    wallet.export_encrypted("wallet.json", password=os.environ["WALLET_PASS"])

# Import from existing key
with Wallet.from_private_key("beefcafe...", network=Network.REGTEST) as wallet:
    print(f"Regtest address: {wallet.address}")

# Create from BIP39 mnemonic
mnemonic = "absorb lecture valley scissors giant evolve planet rotate siren chaos"
with Wallet.from_mnemonic(mnemonic, network=Network.MAINNET) as wallet:
    print(f"Mainnet address: {wallet.address}")
```

### 2. Blockchain Interaction

#### 2.1 Address Information

```python
from brbitcoin import Wallet, get_address_info

info = get_address_info("bc1q...", Network.MAINNET)
print(f"Balance: {info.balance} satoshis")
print(f"UTXOs: {len(info.utxos)}")

with Wallet(network=Network.TESTNET) as wallet:
    print(f"Wallet balance: {wallet.balance} sats")
```

#### 2.2 Transaction Inspection

```python
from brbitcoin import Wallet, get_transaction

tx = get_transaction("aabb...", Network.REGTEST)
print(f"Confirmations: {tx.confirmations}")
for output in tx.outputs:
    print(f"Output value: {output.value}")

with Wallet.from_private_key("beef...") as wallet:
    utxos = wallet.utxos()
    for utxo in utxos:
        print(f"UTXO: {utxo.txid}:{utxo.vout} - {utxo.value} sats")
```

#### 2.3 Block Exploration

```python
from brbitcoin import get_block

# By hash
block = get_block("000000000019d6...", Network.MAINNET)
print(f"Block height: {block.height}")

# By number
genesis = get_block(0, Network.MAINNET)
print(f"Genesis timestamp: {genesis.timestamp}")
```

### 3. Transaction Building

#### 3.1 High-Level (Recommended)

```python
from brbitcoin import Wallet, Fee

RECEIVER = "tb123..."
AMOUNT = 0.001 # BTC

with Wallet(network=Network.REGTEST) as wallet:
    txid = wallet.send(to=RECEIVER, amount=AMOUNT)

    print(f"Broadcasted TX ID: {txid}")
```

#### 3.2 Mid-Level Control

```python
from brbitcoin import Wallet, Transaction, to_btc

RECEIVER = "tb123..."
AMOUNT = 100_000 # Satoshis == 0.001 BTC
FEE = 500 # Satoshi == 0.0000005 BTC

with Wallet.from_private_key("fff...") as wallet:
    utxos = wallet.utxos()

    txid = (
        Transaction(network=wallet.network)
        .add_input(utxos[0])
        .add_output(RECEIVER, to_btc(AMOUNT))
        .fee(to_btc(FEE))
        # .estimate_fee()
        .sign(wallet)
        .broadcast()
    )
    print(f"Broadcasted TX ID: {txid}")
```

#### 3.3 Low-Level Scripting

```python
from brbitcoin import Wallet, Script, Transaction

# Create a P2SH lock script
lock_script = (
    Script()
    .push_op_dup()
    .push_op_hash_160()
    .push_bytes(pubkey_hash)
    .push_op_equal_verify()
    .push_op_check_sig()
)

with Wallet(network=Network.REGTEST) as wallet:
    inputs = wallet.utxos()
    AMOUNT = 0.0001 # BTC
    txid = (
        Transaction(network=Network.REGTEST)
        .add_input(inputs[0])
        .add_output_script(lock_script, AMOUNT)
        .sign(wallet)
        .broadcast()
    )

    print(f"Broadcasted TX ID: {txid}")
```

### 4. Taproot Transactions (BIP340/341/342)

#### 4.1 Generating Taproot Address

```python
from brbitcoin import Wallet, TaprootBuilder, Script, Op

# Generate internal key
with Wallet.create(network=Network.MAINNET) as wallet:
    internal_key = wallet.taproot_internal_key()

    # Build Taproot script tree
    script = Script().push_op_hash_160().push_bytes(b"my_hash160").push_op_equal()
    taproot = TaprootBuilder(internal_key).add_leaf_script(script).finalize()

    print(f"Taproot Address: {taproot.address}")
    print(f"Control Block: {taproot.control_block.hex()}")
```

#### 4.2 Sending to Taproot Address

```python
with Wallet(network=Network.REGTEST) as sender:
    receiver_taproot = "bc1p..."

    txid = (
        Transaction(network=Network.REGTEST)
        .add_input(sender.utxos()[0])
        .add_output_taproot(receiver_taproot, 0.01)  # 0.01 BTC
        .set_change(sender.address)
        .estimate_fee()
        .sign(sender)
        .broadcast()
    )
    print(f"Taproot TX broadcasted: {txid}")
```

#### 4.3 Spending from Taproot (Key Path)

```python
# Spending using Schnorr signature
with Wallet.from_taproot_internal_key("internal_key_hex") as wallet:
    utxo = wallet.utxos()[0]

    tx = (
        Transaction(network=Network.MAINNET)
        .add_taproot_input(utxo)
        .add_output("bc1q...", 0.009)
        .set_change(wallet.taproot_address)
        .estimate_fee()
        .sign_taproot(wallet)
        .broadcast()
    )
    print(f"Key path spend TX: {tx.txid}")
```

#### 4.4 Spending from Taproot (Script Path)

```python
from brbitcoin import TaprootScriptSolution, Script

# Reveal script and provide solution

preimage = b"secret123"
script = (
    Script()
    .push_op_hash160()
    .push_bytes(hash160(preimage))
    .push_op_equal()
)

with Wallet(network=Network.REGTEST) as spender:
    solution = TaprootScriptSolution(
        script=script,
        solution_ops=[Script.op_push_bytes, preimage]
    )

    tx = (
        Transaction(network=Network.REGTEST)
        .add_taproot_input(utxo, solution=solution)
        .add_output("bc1q...", 0.0095)
        .sign_taproot(spender)
        .broadcast()
    )
    print(f"Script path spend TX: {tx.txid}")
```

#### 4.5 Taproot Benefits

- Privacy: All spends look identical on-chain
- Efficiency: Smaller witness size vs traditional multisig
- Flexibility: Combine multiple spending conditions
- Standard: BIP340 (Schnorr), BIP341 (Taproot), BIP342 (Tapscript)

### 5. Security Practices

#### 5.1 Encrypted Private Key Backup

```python
from brbitcoin import Wallet
import os

PATH = "wallet.json"
PASS = os.environ["WALLET_PASS"]

with Wallet.create() as wallet:
    wallet.export_encrypted(path=PATH, password=PASS)
```

#### 5.2 Restore from Encrypted backup

```python
from brbitcoin import Wallet


PATH = "wallet.json"
PASS = os.environ["WALLET_PASS"]

with Wallet.from_encrypted(path=PATH, password=PASS) as wallet:
    print(f"Recovered address: {w.address}")
```

#### 5.3 Zeroization Guarantees

```python
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

```python
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

```python
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

```python
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

```python
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

```python
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

```python
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

# [License: MIT](../LICENSE)
