-------

Para implementar **zeroização** (apagamento seguro de dados sensíveis da memória) na abordagem onde a `Wallet` gerencia a assinatura, precisamos garantir que:

1. **Chaves privadas** sejam apagadas após o uso, mesmo em caso de exceção.
2. **Dados temporários** (nonces, derivations paths) não persistam na memória.
3. A API mantenha ergonomia, sem comprometer a segurança.

---

### **Estratégia de Zeroização em Python**  
Python não oferece controle direto sobre a memória, mas podemos usar técnicas como:

#### **1. `bytearray` Sobrescrita para Chaves**  
```python
from zeroize import zeroize_buffer  # Biblioteca para zeroização segura

class Wallet:
    def __init__(self, private_key_hex: str):
        # Armazena a chave em um buffer mutável
        self._privkey = bytearray.fromhex(private_key_hex)
    
    def sign(self, tx: Transaction) -> Transaction:
        try:
            # Copia para um buffer temporário e assina
            key_copy = bytes(self._privkey)
            tx.signature = ecdsa_sign(key_copy, tx.hash())
            return tx
        finally:
            zeroize_buffer(self._privkey)  # Apaga a chave da memória
            zeroize_buffer(key_copy)       # Apaga a cópia
```

#### **2. Context Managers para Ciclo de Vida Controlado**  
```python
class SecureSigner:
    def __init__(self, private_key_hex: str):
        self._privkey = bytearray.fromhex(private_key_hex)
    
    def __enter__(self):
        return self
    
    def __exit__(self, *args):
        zeroize_buffer(self._privkey)  # Garante zeroização ao sair do contexto
    
    def sign_tx(self, tx: Transaction) -> Transaction:
        tx.sign(bytes(self._privkey))
        return tx

# Uso seguro:
with SecureSigner("my_private_key") as signer:
    signed_tx = tx.sign(signer).broadcast()
```

#### **3. Atualização da API Proposta**  
```python
class Transaction:
    def __init__(self):
        self._inputs = []
        self._outputs = []
        self._signatures = bytearray()  # Dados sensíveis em buffers mutáveis
    
    def sign(self, wallet: Wallet) -> "Transaction":
        try:
            # Assina e armazena a assinatura em buffer mutável
            sig = wallet.sign(self.hash())
            self._signatures.extend(sig)
            return self
        finally:
            wallet.clear_keys()  # Método interno para zeroização
    
    def broadcast(self) -> str:
        # Serializa e envia, depois apaga dados sensíveis
        raw_tx = self.serialize()
        network.post(raw_tx)
        zeroize_buffer(self._signatures)
        return raw_tx.txid()

    def __del__(self):
        # Destrutor como última linha de defesa (não confiável, mas útil)
        zeroize_buffer(self._signatures)
```

---

### **Problemas e Soluções na Abordagem Original**  

#### **1. Risco: Chave Persiste na Wallet**  
- **Problema**: Se a `Wallet` mantiver a chave em uma `str` ou `bytes` imutável, ela pode vazar via garbage collector.  
- **Solução**:  
  ```python
  class Wallet:
      def __init__(self, private_key_hex: str):
          self._privkey = bytearray.fromhex(private_key_hex)  # Mutável
      
      def clear_keys(self):
          zeroize_buffer(self._privkey)  # Sobrescreve o buffer
  ```

#### **2. Risco: Dados Sensíveis em Exceções**  
- **Problema**: Se `.sign()` falhar após gerar dados parciais, eles podem persistir.  
- **Solução**:  
  ```python
  def sign(self, wallet: Wallet) -> "Transaction":
      try:
          # Cria cópia temporária para assinatura
          temp_sig = bytearray()
          sig = wallet.sign(self.hash())
          temp_sig.extend(sig)
          self._signatures = temp_sig
          return self
      except Exception as e:
          zeroize_buffer(temp_sig)
          raise
  ```

---

### **Exemplo Revisado com Zeroização**  
```python
from zeroize import zeroize_buffer

class SecureWallet:
    def __init__(self, private_key_hex: str):
        self._privkey = bytearray.fromhex(private_key_hex)
    
    def sign_transaction(self, tx_hash: bytes) -> bytes:
        try:
            # Usar libsecp256k1 para evitar cópias implícitas
            from secp256k1 import PrivateKey
            privkey = PrivateKey(bytes(self._privkey), raw=True)
            sig = privkey.ecdsa_sign(tx_hash)
            return privkey.ecdsa_serialize(sig)
        finally:
            zeroize_buffer(self._privkey)
    
    def __del__(self):
        zeroize_buffer(self._privkey)

class Transaction:
    def __init__(self, network: Network):
        self.network = network
        self._inputs = []
        self._outputs = []
        self._signatures = bytearray()
    
    def sign(self, wallet: SecureWallet) -> "Transaction":
        tx_hash = self._compute_hash()
        sig = wallet.sign_transaction(tx_hash)
        self._signatures.extend(sig)
        return self
    
    def broadcast(self) -> str:
        raw_tx = self.serialize()
        # Simular envio (substitua pela lógica real)
        print(f"Broadcasting TX: {raw_tx.hex()}")
        zeroize_buffer(self._signatures)
        return "fake_txid"
    
    def __del__(self):
        zeroize_buffer(self._signatures)

# Uso Seguro
wallet = SecureWallet("abcd1234...")
tx = (
    Transaction(network=Network.REGTEST)
    .add_input(inputs[0])
    .add_output(RECEIVER, to_btc(AMOUNT))
    .set_fee_strategy(Fee.MEDIUM)
    .sign(wallet)
    .broadcast()
)
```

---

### **Checklist de Segurança**  
- [ ] Chaves armazenadas em `bytearray`, não em `str`/`bytes`.  
- [ ] `zeroize_buffer` aplicado em:  
  - Destruidores (`__del__`).  
  - Após serialização.  
  - Blocos `finally` após exceções.  
- [ ] Uso de bibliotecas criptográficas que evitam cópias implícitas (ex: `secp256k1` em vez de `ecdsa`).  
- [ ] Testes de vazamento com ferramentas como `tracemalloc` ou `gc.get_objects()`.  

Esta abordagem garante que dados sensíveis sejam apagados mesmo em cenários complexos, mantendo a fluência da API.


-------
