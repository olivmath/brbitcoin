# 🇧🇷 BrBitcoin Python SDK


![hello-world](https://github.com/user-attachments/assets/65597643-95cf-4583-8653-26eb2deb3fc9)



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

> [!WARNING]
>  Always test with Regtest before MAINNET usage.

### 1. Wallet Management

```python
from brbitcoin import Wallet, Network

# Create random HD wallet (testnet by default)
with Wallet.create(network=Network.TESTNET) as wallet:
    print(f"New address: {wallet.address}")
    # Always store encrypted backups
    wallet.export_encrypted("wallet.bak", password=os.environ["WALLET_PASS"])

# Import from existing hex private key
with Wallet.from_private_key("beefcafe...") as wallet:
    print(f"Imported address: {wallet.address}")

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

block = get_block_by_hash("000000000019d6...", Network.MAINNET)
print(f"The Block has {len(block.txn)} transactions")

genesis = get_block_by_number(0, Network.MAINNET)
print(f"Genesis block timestamp: {genesis.timestamp}")
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

### 4. Security Practices

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

#### 4.3 Zeroization Guarantees

```python
# Keys are wiped:
# - When context manager exits
# - After signing/broadcast
# - On object destruction
with Wallet.from_private_key("c0ffee...") as wallet:
    txid = wallet.send("bc1q...", 0.001)
    # Key no longer in memory here
```

### 5. Node Management

#### 5.1 Network Configuration

```python
from brbitcoin import ClientNode

# Connect to Bitcoin Core
client = ClientNode(
    network=Network.REGTEST,
    rpc_user="user",
    rpc_password="pass",
    host="localhost",
    port=18444
)
```

#### 5.2 Node Operations

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

#### 5.3 Direct RPC Access

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
