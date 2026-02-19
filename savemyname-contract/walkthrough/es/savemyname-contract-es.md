# Contrato SaveMyName: Entendiendo Strings y Ubicaciones de Datos

Bienvenido a una lección fundamental en Solidity: **trabajar con strings y entender las ubicaciones de datos**. Hoy, estás aprendiendo cómo almacenar y recuperar datos de texto en la blockchain - y por qué es más complejo que almacenar números.

## Por Qué los Strings Son Diferentes

**Tipos simples (números):**
```solidity
uint256 public age = 25;  // Tamaño fijo: 32 bytes
bool public active = true; // Tamaño fijo: 1 byte
address public owner;      // Tamaño fijo: 20 bytes
```

**Tipos complejos (strings):**
```solidity
string public name = "Alice";           // Tamaño variable: 5 bytes
string public bio = "Solidity dev...";  // Tamaño variable: 15+ bytes
```

**La diferencia:**
- **Números** - Tamaño fijo, almacenados directamente
- **Strings** - Tamaño variable, almacenados como arrays dinámicos
- **Strings** - Más costosos de almacenar y manipular
- **Strings** - Requieren manejo especial (palabras clave memory/storage)

## El Costo de Almacenar Strings

**Costos de gas:**
- Almacenar `uint256`: ~20,000 gas
- Almacenar string corto (5 caracteres): ~25,000 gas
- Almacenar string largo (100 caracteres): ~100,000+ gas

**¿Por qué tan costoso?** Los strings se almacenan como arrays de bytes. Cada carácter cuesta gas para almacenar.

### Almacenando un Nombre y Bio en la Blockchain

```
string name;
string bio;
```

#### ¿Qué Está Pasando Aquí?

string name; y string bio; son variables de estado almacenadas permanentemente en la blockchain.

### La Función add() – Almacenando Datos

```
function add(string memory _name, string memory _bio) public {
    name = _name;
    bio = _bio;
}
```

#### Desglosándolo

_name y _bio son parámetros de función. Usamos la palabra clave memory porque los strings son complejos y requieren almacenamiento temporal durante la ejecución de la función.

### La Función retrieve() – Obteniendo Datos de la Blockchain

```
function retrieve() public view returns (string memory, string memory) {
    return (name, bio);
}
```

#### Entendiendo las Funciones view

La palabra clave view le dice a Solidity que esta función solo lee datos y no cuesta gas cuando se llama externamente.

### Haciendo el Contrato Más Eficiente

Puedes combinarlas en una sola función, pero podría aumentar los costos de gas si modifica el estado:

```
function saveAndRetrieve(string memory _name, string memory _bio) public returns (string memory, string memory) {
    name = _name;
    bio = _bio;
    return (name, bio);
}
```

---

## Entendiendo las Ubicaciones de Datos

Solidity tiene tres ubicaciones de datos: **storage**, **memory**, y **calldata**.

### Storage - Almacenamiento Permanente en Blockchain

```solidity
string public name;  // Almacenado permanentemente en blockchain
string public bio;   // Almacenado permanentemente en blockchain
```

**Características:**
- Persiste entre llamadas de función
- Más costoso (cuesta gas escribir)
- Las variables de estado siempre están en storage
- Puede ser modificado

**Ejemplo:**
```solidity
function updateName(string memory _name) public {
    name = _name;  // Escribe en storage (¡costoso!)
}
```

### Memory - Almacenamiento Temporal de Función

```solidity
function add(string memory _name, string memory _bio) public {
    // _name y _bio existen solo durante la ejecución de la función
    name = _name;
    bio = _bio;
}
```

**Características:**
- Temporal, borrado después de que termina la función
- Más barato que storage
- Requerido para tipos complejos en parámetros de función
- Puede ser modificado

**¿Por qué usar memory?**
```solidity
// ¡Esto no compilará!
function add(string _name) public {  // Falta 'memory'
    name = _name;
}

// ¡Esto funciona!
function add(string memory _name) public {
    name = _name;
}
```

### Calldata - Entrada de Función de Solo Lectura

```solidity
function add(string calldata _name, string calldata _bio) external {
    // _name y _bio son de solo lectura
    name = _name;
    bio = _bio;
}
```

**Características:**
- Solo lectura (no puede ser modificado)
- Opción más barata
- Solo para parámetros de función externa
- Mejor para optimización de gas

**Comparación:**

| Ubicación | Modificable | Persistencia | Costo | Caso de Uso |
|----------|-----------|-------------|------|----------|
| **storage** | Sí | Permanente | Alto | Variables de estado |
| **memory** | Sí | Temporal | Medio | Variables de función |
| **calldata** | No | Temporal | Bajo | Entradas externas |

---

## Operaciones con Strings y Limitaciones

### Lo Que NO PUEDES Hacer con Strings

```solidity
// ❌ No puedes concatenar directamente
string result = "Hello" + "World";  // ¡No funciona!

// ❌ No puedes comparar directamente
if (name == "Alice") { }  // ¡No funciona!

// ❌ No puedes obtener la longitud fácilmente
uint len = name.length;  // ¡No funciona!

// ❌ No puedes acceder a caracteres por índice
string firstChar = name[0];  // ¡No funciona!
```

### Lo Que SÍ PUEDES Hacer

**1. Almacenar y recuperar:**
```solidity
string public name;

function setName(string memory _name) public {
    name = _name;
}

function getName() public view returns (string memory) {
    return name;
}
```

**2. Comparar usando keccak256:**
```solidity
function isNameAlice(string memory _name) public pure returns (bool) {
    return keccak256(abi.encodePacked(_name)) == keccak256(abi.encodePacked("Alice"));
}
```

**3. Convertir a bytes para manipulación:**
```solidity
function getLength(string memory _str) public pure returns (uint) {
    return bytes(_str).length;
}
```

---

## Patrones Avanzados de Strings

### 1. Usar bytes32 en Lugar de string

**Para texto corto de longitud fija:**

```solidity
// COSTOSO
string public name;  // Tamaño variable, dinámico

// MÁS BARATO
bytes32 public name;  // Tamaño fijo, máximo 32 bytes

function setName(string memory _name) public {
    require(bytes(_name).length <= 32, "Nombre demasiado largo");
    name = bytes32(bytes(_name));
}

function getName() public view returns (string memory) {
    return string(abi.encodePacked(name));
}
```

**Ahorro:** ¡Reducción del ~50% en gas para strings cortos!

### 2. Concatenación de Strings

```solidity
function concatenate(string memory a, string memory b) public pure returns (string memory) {
    return string(abi.encodePacked(a, b));
}

// Ejemplo: concatenate("Hello", "World") devuelve "HelloWorld"
```

### 3. Comparación de Strings

```solidity
function compareStrings(string memory a, string memory b) public pure returns (bool) {
    return keccak256(abi.encodePacked(a)) == keccak256(abi.encodePacked(b));
}
```

### 4. Múltiples Valores de Retorno

```solidity
function getUserInfo() public view returns (string memory, string memory, uint256) {
    return (name, bio, block.timestamp);
}

// Uso:
// (string memory userName, string memory userBio, uint256 timestamp) = getUserInfo();
```

---

## Casos de Uso del Mundo Real

### ENS (Ethereum Name Service)

**Almacena nombres de dominio como strings:**
```solidity
mapping(bytes32 => string) public names;

function setName(bytes32 node, string memory name) public {
    names[node] = name;
}
```

### Metadata de NFT

**Almacena URIs de tokens:**
```solidity
mapping(uint256 => string) private _tokenURIs;

function tokenURI(uint256 tokenId) public view returns (string memory) {
    return _tokenURIs[tokenId];
}
```

### Perfiles Sociales

**Almacena biografías de usuarios:**
```solidity
struct Profile {
    string username;
    string bio;
    string avatarURL;
}

mapping(address => Profile) public profiles;
```

---

## Consejos de Optimización de Gas

### 1. Usar Eventos para Datos Históricos

```solidity
event NameChanged(address indexed user, string newName, uint256 timestamp);

function setName(string memory _name) public {
    emit NameChanged(msg.sender, _name, block.timestamp);
    // No almacenar en estado si solo necesitas historial
}
```

**¿Por qué?** Los eventos son mucho más baratos que el almacenamiento (~2,000 gas vs ~20,000 gas).

### 2. Almacenar Hashes en Lugar de Strings Completos

```solidity
// COSTOSO
mapping(address => string) public documents;

// MÁS BARATO
mapping(address => bytes32) public documentHashes;

function storeDocument(string memory doc) public {
    documentHashes[msg.sender] = keccak256(abi.encodePacked(doc));
}

function verifyDocument(string memory doc) public view returns (bool) {
    return documentHashes[msg.sender] == keccak256(abi.encodePacked(doc));
}
```

### 3. Usar IPFS para Texto Grande

```solidity
// Almacenar hash IPFS (46 bytes) en lugar de contenido completo
mapping(address => string) public ipfsHashes;

function setContent(string memory ipfsHash) public {
    ipfsHashes[msg.sender] = ipfsHash;
    // Contenido real almacenado fuera de la cadena en IPFS
}
```

---

## Errores Comunes

### 1. Olvidar la Ubicación de Datos

```solidity
// ❌ No compilará
function setName(string _name) public {
    name = _name;
}

// ✅ Correcto
function setName(string memory _name) public {
    name = _name;
}
```

### 2. Usar storage Cuando Quieres Decir memory

```solidity
// ❌ Costoso e incorrecto
function processName(string storage _name) internal {
    // ¡Modifica la variable de estado!
}

// ✅ Correcto
function processName(string memory _name) internal pure {
    // Trabaja con copia
}
```

### 3. No Validar la Longitud del String

```solidity
// ❌ Podría almacenar strings enormes (¡costoso!)
function setBio(string memory _bio) public {
    bio = _bio;
}

// ✅ Limitar longitud
function setBio(string memory _bio) public {
    require(bytes(_bio).length <= 280, "Bio demasiado larga");
    bio = _bio;
}
```

---

## Conceptos Clave Que Has Aprendido

**1. Almacenamiento de strings** - Cómo se almacenan los datos de texto en blockchain

**2. Ubicaciones de datos** - Diferencias entre storage, memory, calldata

**3. Costos de gas** - Por qué los strings son costosos

**4. Operaciones con strings** - Comparación, concatenación, conversión

**5. Optimización** - Patrones de bytes32, eventos, IPFS

**6. Funciones view** - Lecturas gratuitas vs escrituras costosas

---

## Por Qué Esto Importa

Los strings están en todas partes en Web3:
- **ENS** - Nombres de dominio
- **NFTs** - URIs de metadata
- **Social** - Nombres de usuario, biografías
- **DeFi** - Símbolos de tokens, nombres

Entender cómo trabajar con strings eficientemente es esencial para construir aplicaciones reales.

## Desafíate a Ti Mismo

Extiende este contrato:
1. Agrega validación de longitud de string
2. Implementa concatenación de strings para nombres completos
3. Crea una verificación de unicidad de nombre de usuario
4. Construye un sistema de perfiles con múltiples campos de string
5. Agrega integración IPFS para contenido de texto grande

¡Has dominado los strings y las ubicaciones de datos. Este es conocimiento fundamental de Solidity!
