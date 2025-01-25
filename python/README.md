# 🇧🇷 BrBitcoin Python SDK

> [!WARNING]
> This SDK is under active development. Report issues on [GitHub](https://github.com/youruser/brbitcoin).

## Features

- **Automatic Zeroization**: Sensitive data wiped from memory using context managers
- **Multi-Layer Security**: Hierarchical deterministic wallets with encrypted backups
- **Network Agnostic**: Supports Regtest/Testnet/Mainnet via multiple backends
- **Full RPC Support**: Direct access to Bitcoin Core JSON-RPC API

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

### 1. Wallet Management

```python
from brbitcoin import Wallet, Network

# Create random wallet (default: testnet)
with Wallet.create(network=Network.TESTNET) as wallet:
    print(f"New wallet address: {wallet.address}")
    # Warning: Only export keys for backup purposes!
    print(f"Private key: {wallet.export_encrypted(path="wallet.json",password="pass123")}")

# Import from existing key (hex format)
with Wallet.from_private_key("beefcafe...", network=Network.REGTEST) as wallet:
    print(f"Imported wallet address: {wallet.address}")

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
print(f"UTXO count: {info.utxo}")

with Wallet.create(network=Network.TESTNET) as wallet:
    balance = wallet.balance()
    print(f"Balance: {balance} satoshis")
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

block = get_block_by_hash("000000000019d6...", Network.MAINNET)
print(f"The Block has {len(block.txn)} transactions")

genesis = get_block_by_number(0, Network.MAINNET)
print(f"Genesis block timestamp: {genesis.timestamp}")
```

### 3. Sending Bitcoin

#### 3.1 High-Level (Recommended)

```python
from brbitcoin import Wallet, Fee

RECEIVER = "tb123..."
AMOUNT = 0.001 # BTC

with Wallet(network=Network.REGTEST) as wallet:
    txid = wallet.send(to=RECEIVER, amount=AMOUNT)

    print(f"Broadcasted TX ID: {txid}")
```

#### 3.2 Mid-Level (Manual Control)

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
        .sign(wallet)
        .broadcast()
    )
    print(f"Broadcasted TX ID: {txid}")
```

#### 3.3 Low-Level (Custom Scripts)

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

### 4. Security Notes

- **Automatic Zeroization**: Private keys are wiped from memory when:
  - The context manager (`with`) is closed.
  - Methods `.sign()` or `.broadcast()` complete.
  - The `Wallet` object is destroyed.

#### 4.1 Encrypted Private Key Backup

```python
from brbitcoin import Wallet

with Wallet.create() as wallet:
    wallet.export_encrypted(path="wallet.json",password="pass123")
```

#### 4.2 Restore from Encrypted backup

```python
from brbitcoin import Wallet

with Wallet.from_encrypted("wallet.json", password="pass123") as wallet:
    print(f"Recovered address: {w.address}")
```

### 5. Node Management

#### 5.1 Network Configuration

```python
from brbitcoin import Wallet, Node, Network

client = Node(
    network=Network.REGTEST,
    rpc_user="your_user",
    rpc_password="your_pass",
    host="localhost"
    rpc_url=18444
)
```

#### 5.2 Basic Node Operations

```python
# Get node information
info = client.get_info()
print(f"Block height: {info.blocks}")
print(f"Node version: {info.version}")

# Generate blocks (Regtest only)
if client.network == Network.REGTEST:
    new_blocks = client.generate_blocks(10, address="bcrt1q...")
    print(f"Generated {len(new_blocks)} blocks")

# Get mempool status
mempool = client.get_mempool()
print(f"{len(mempool)} transactions in mempool")

# Estimate smart fee
fee_rate = client.estimate_fee(6)  # Fee for 6-block confirmation
print(f"Recommended fee rate: {fee_rate} BTC/kvB")

# Get block header
header = client.get_block_header("000000000000000000015a0...")
print(f"Block timestamp: {header.timestamp}")
```

#### 5.3 Direct RPC Commands

```python
# Get blockchain info
chain_info = client.command("getblockchaininfo")
print(f"Chain: {chain_info['chain']}, Blocks: {chain_info['blocks']}")

# Generate blocks (regtest only)
if client.network == Network.REGTEST:
    blocks = client.command("generatetoaddress", 100, "bcrt1q...")
    print(f"Mined {len(blocks)} blocks")

# Send raw transaction
raw_tx = "020000000001..."
tx_id = client.command("sendrawtransaction", raw_tx)
print(f"Broadcasted TX: {tx_id}")

full_help = client.command("help")
print("Available commands:", list(full_help.keys()))
```

#### 5.4 Bitcoin Core RPC Command Reference (Partial)

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
