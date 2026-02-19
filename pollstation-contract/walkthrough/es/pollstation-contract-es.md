# Contrato PollStation: Dominando Arrays y Mappings

Bienvenido a una lección fundamental sobre estructuras de datos en Solidity: **arrays y mappings**. Hoy, estás construyendo un sistema de votación que demuestra cómo almacenar y gestionar colecciones de datos en la cadena.

## Por Qué Importan las Estructuras de Datos

**El desafío:** Necesitas almacenar múltiples piezas de datos relacionados:
- Lista de candidatos
- Conteos de votos para cada candidato
- Quién ha votado
- Historial de votación

**Sin estructuras de datos apropiadas:**
```solidity
// ¡Esto no escala!
string public candidate1;
string public candidate2;
string public candidate3;
uint256 public votes1;
uint256 public votes2;
uint256 public votes3;
```

**Problemas:**
- Número fijo de candidatos
- No se puede iterar sobre candidatos
- Código duplicado en todas partes
- Imposible de mantener

**La solución:** ¡Arrays y mappings!

### Declarando la Lista de Candidatos y el Seguimiento de Votos

```
string[] public candidateNames;
mapping(string => uint256) voteCount;
```

#### Arrays – Almacenando una Lista de Candidatos

string[] es un array de strings. Almacena múltiples valores en una lista ordenada.

#### Mappings – Almacenando Conteos de Votos

mapping(string => uint256) vincula el nombre de un candidato con su conteo de votos para búsquedas instantáneas.

### Agregando Candidatos – La Función addCandidateNames()

```
function addCandidateNames(string memory _candidateNames) public {
    candidateNames.push(_candidateNames);
    voteCount[_candidateNames] = 0;
}
```

### Recuperando la Lista de Candidatos

```
function getcandidateNames() public view returns (string[] memory) {
    return candidateNames;
}
```

### Votando por un Candidato – La Función vote()

```
function vote(string memory _candidateNames) public {
    voteCount[_candidateNames] += 1;
}
```

### Verificando los Votos de un Candidato

```
function getVote(string memory _candidateNames) public view returns (uint256) {
    return voteCount[_candidateNames];
}
```

---

## Arrays vs Mappings: Cuándo Usar Cada Uno

### Arrays - Listas Ordenadas

**Usar cuando:**
- Necesitas iterar sobre elementos
- El orden importa
- Necesitas conocer la longitud
- Quieres acceder por índice

**Casos de uso de ejemplo:**
- Lista de candidatos
- Historial de transacciones
- Tablas de clasificación
- Listas de titulares de tokens

**Limitaciones:**
- Costoso buscar (O(n))
- Costoso eliminar del medio
- Los costos de gas aumentan con el tamaño

### Mappings - Pares Clave-Valor

**Usar cuando:**
- Necesitas búsquedas rápidas (O(1))
- Tienes una clave única
- El orden no importa
- No necesitas iterar

**Casos de uso de ejemplo:**
- Conteos de votos por candidato
- Balances por dirección
- Registros de propiedad
- Listas de control de acceso

**Limitaciones:**
- No se puede iterar sobre las claves
- No se puede obtener la longitud
- Todas las claves existen (devuelven valor predeterminado)

---

## Consideraciones de Seguridad

### 1. Doble Votación

**El problema:** ¡Nada impide que alguien vote múltiples veces!

```solidity
// VULNERABLE
function vote(string memory _candidateNames) public {
    voteCount[_candidateNames] += 1;
}
```

**La solución:**
```solidity
mapping(address => bool) public hasVoted;

function vote(string memory _candidateNames) public {
    require(!hasVoted[msg.sender], "Ya has votado");
    hasVoted[msg.sender] = true;
    voteCount[_candidateNames] += 1;
}
```

### 2. Candidatos Inválidos

**El problema:** ¡Se puede votar por candidatos que no existen!

```solidity
// VULNERABLE
vote("PersonaAleatoria"); // Crea un conteo de votos para candidato inexistente
```

**La solución:**
```solidity
mapping(string => bool) public isCandidate;

function addCandidateNames(string memory _candidateNames) public {
    candidateNames.push(_candidateNames);
    isCandidate[_candidateNames] = true;
    voteCount[_candidateNames] = 0;
}

function vote(string memory _candidateNames) public {
    require(isCandidate[_candidateNames], "Candidato inválido");
    require(!hasVoted[msg.sender], "Ya has votado");
    hasVoted[msg.sender] = true;
    voteCount[_candidateNames] += 1;
}
```

### 3. Adición No Autorizada de Candidatos

**El problema:** ¡Cualquiera puede agregar candidatos!

**La solución:**
```solidity
address public admin;

constructor() {
    admin = msg.sender;
}

modifier onlyAdmin() {
    require(msg.sender == admin, "No eres admin");
    _;
}

function addCandidateNames(string memory _candidateNames) public onlyAdmin {
    candidateNames.push(_candidateNames);
    isCandidate[_candidateNames] = true;
    voteCount[_candidateNames] = 0;
}
```

---

## Características Avanzadas de Votación

### 1. Período de Votación

```solidity
uint256 public votingStart;
uint256 public votingEnd;

constructor(uint256 _durationInDays) {
    votingStart = block.timestamp;
    votingEnd = block.timestamp + (_durationInDays * 1 days);
}

function vote(string memory _candidateNames) public {
    require(block.timestamp >= votingStart, "La votación no ha comenzado");
    require(block.timestamp <= votingEnd, "La votación ha terminado");
    // ... resto de la lógica
}
```

### 2. Obtener Ganador

```solidity
function getWinner() public view returns (string memory winner, uint256 winningVoteCount) {
    require(block.timestamp > votingEnd, "La votación aún está activa");
    
    winningVoteCount = 0;
    for (uint i = 0; i < candidateNames.length; i++) {
        uint256 votes = voteCount[candidateNames[i]];
        if (votes > winningVoteCount) {
            winningVoteCount = votes;
            winner = candidateNames[i];
        }
    }
}
```

### 3. Historial de Votos

```solidity
mapping(address => string) public voterChoice;

function vote(string memory _candidateNames) public {
    // ... validación ...
    voterChoice[msg.sender] = _candidateNames;
    voteCount[_candidateNames] += 1;
}
```

---

## Consejos de Optimización de Gas

### 1. Usar Bytes32 en Lugar de String

```solidity
// COSTOSO
string[] public candidateNames;
mapping(string => uint256) voteCount;

// MÁS BARATO
bytes32[] public candidateNames;
mapping(bytes32 => uint256) voteCount;
```

**¿Por qué?** Los strings son dinámicos y costosos. bytes32 es de tamaño fijo y más barato.

### 2. Cachear la Longitud del Array

```solidity
// COSTOSO - lee la longitud en cada iteración
for (uint i = 0; i < candidateNames.length; i++) {
    // ...
}

// MÁS BARATO - lee la longitud una vez
uint256 length = candidateNames.length;
for (uint i = 0; i < length; i++) {
    // ...
}
```

### 3. Usar Eventos para Historial

```solidity
event Voted(address indexed voter, string candidate, uint256 timestamp);

function vote(string memory _candidateNames) public {
    // ... lógica ...
    emit Voted(msg.sender, _candidateNames, block.timestamp);
}
```

**¿Por qué?** Los eventos son mucho más baratos que el almacenamiento para datos históricos.

---

## Sistemas de Votación del Mundo Real

### Snapshot (Votación Fuera de la Cadena)
- Votos almacenados fuera de la cadena (IPFS)
- Votación sin gas
- Solo ejecución en la cadena

### Compound Governance
- Votación ponderada por tokens
- Soporte de delegación
- Ejecución con timelock

### Aragon
- Aplicaciones de votación modulares
- Múltiples estrategias de votación
- Framework de DAO

---

## Conceptos Clave Que Has Aprendido

**1. Arrays** - Listas ordenadas e iterables

**2. Mappings** - Búsquedas rápidas de clave-valor

**3. Selección de estructura de datos** - Cuándo usar cada una

**4. Patrones de seguridad** - Prevenir doble votación, validación

**5. Optimización de gas** - bytes32, caché, eventos

**6. Mecánicas de votación** - Períodos, ganadores, historial

---

## Por Qué Esto Importa

Los arrays y mappings son la base de cada smart contract:
- **Balances de tokens** - mapping(address => uint256)
- **Propiedad de NFT** - mapping(uint256 => address)
- **Lista blanca** - mapping(address => bool)
- **Historial de transacciones** - array de structs

Dominar estas estructuras de datos es esencial para el desarrollo en Solidity.

## Desafíate a Ti Mismo

Extiende este sistema de votación:
1. Agrega votación ponderada (basada en tokens)
2. Implementa votación por clasificación
3. Crea un sistema multi-encuesta
4. Agrega delegación de votos
5. Construye un mecanismo de votación cuadrática

¡Has dominado las estructuras de datos de Solidity. Este es conocimiento fundamental!
