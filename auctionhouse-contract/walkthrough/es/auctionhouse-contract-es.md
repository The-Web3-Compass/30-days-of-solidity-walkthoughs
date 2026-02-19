# Contrato AuctionHouse: Construyendo Tu Primer Sistema de Subastas

¡Bienvenido a uno de los contratos más emocionantes que construirás! Hoy estamos creando un sistema de subastas completo - piensa en eBay, pero en la blockchain. Los usuarios pueden hacer ofertas, competir por artículos, y el mejor postor gana.

## ¿Por Qué Subastas en la Blockchain?

Las subastas en línea tradicionales tienen problemas:
- **Problemas de confianza** - ¿Puede la plataforma manipular las ofertas?
- **Transparencia** - ¿Puedes verificar que la oferta más alta es real?
- **Centralización** - Una compañía controla todo

Las subastas en blockchain resuelven esto. Cada oferta es pública, verificable e inmutable. Nadie puede hacer trampa. Las reglas son aplicadas por código, no por una compañía que podría tener conflictos de interés.

## Lo Que Estamos Construyendo

Un sistema de subastas donde:
- El propietario crea una subasta para un artículo con un límite de tiempo
- Cualquiera puede hacer ofertas durante el período de subasta
- Los usuarios pueden aumentar sus propias ofertas
- Después de que expira el tiempo, la subasta puede finalizarse
- El mejor postor gana

¡Vamos a construirlo paso a paso!

---

## Configurando el Estado de la Subasta

Primero, necesitamos rastrear toda la información sobre nuestra subasta:

```solidity
address public owner;           // Quién creó la subasta
string public item;             // Qué se está subastando
uint public auctionEndTime;     // Cuándo termina la oferta
address private highestBidder;  // Ganador actual
uint private highestBid;        // Monto de la oferta ganadora
bool public ended;              // ¿Se ha finalizado la subasta?
mapping(address => uint) public bids;  // Oferta actual de cada uno
address[] public bidders;       // Lista de todos los postores
```

**¿Por qué estas variables?**

- **`owner`** - La persona que desplegó el contrato y está subastando el artículo
- **`item`** - Una descripción como "NFT Raro" o "Guitarra Vintage"
- **`auctionEndTime`** - Una marca de tiempo. Después de esto, no se permiten más ofertas
- **`highestBidder` & `highestBid`** - Rastrean quién está ganando
- **`ended`** - Previene que la subasta termine múltiples veces
- **`bids` mapping** - Cada dirección tiene su monto de oferta actual
- **`bidders` array** - Útil para iterar a través de todos los participantes después

Nota que `highestBidder` y `highestBid` son `private`. No queremos que la gente espíe quién está ganando hasta que termine la subasta (aunque aún podrían averiguarlo desde eventos o llamando a `getWinner()` después de que termine).

---

## El Constructor: Iniciando la Subasta

Cuando despliegas este contrato, configuras la subasta:

```solidity
constructor(string memory _item, uint _biddingTime) {
    owner = msg.sender;
    item = _item;
    auctionEndTime = block.timestamp + _biddingTime;
}
```

**Cómo funciona:**

- **`msg.sender`** - Tú (el desplegador) te conviertes en el propietario
- **`_item`** - Especificas qué se está subastando
- **`_biddingTime`** - Cuántos segundos dura la subasta (ej., 86400 para 24 horas)

**`block.timestamp`** es una variable especial que te da el tiempo actual de la blockchain en segundos desde la época Unix (1 de enero de 1970). Al agregar `_biddingTime`, establecemos una fecha límite.

**Ejemplo:** Si despliegas en la marca de tiempo 1700000000 con `_biddingTime = 3600` (1 hora), la subasta termina en la marca de tiempo 1700003600.

---

## Haciendo una Oferta: El Corazón de la Subasta

Aquí es donde sucede la acción:

```solidity
function bid(uint amount) external {
    require(block.timestamp < auctionEndTime, "La subasta ya ha terminado.");
    require(amount > 0, "El monto de la oferta debe ser mayor que cero.");
    require(amount > bids[msg.sender], "La nueva oferta debe ser mayor que tu oferta actual.");

    if (bids[msg.sender] == 0) {
        bidders.push(msg.sender);
    }

    bids[msg.sender] = amount;

    if (amount > highestBid) {
        highestBid = amount;
        highestBidder = msg.sender;
    }
}
```

**Vamos a recorrer esto paso a paso:**

### Paso 1: Verificaciones de Validación

```solidity
require(block.timestamp < auctionEndTime, "La subasta ya ha terminado.");
```
Verifica si todavía estamos dentro del período de oferta. Si se acabó el tiempo, rechaza la oferta.

```solidity
require(amount > 0, "El monto de la oferta debe ser mayor que cero.");
```
No se permiten ofertas de cero. ¡Tienes que ofertar algo realmente!

```solidity
require(amount > bids[msg.sender], "La nueva oferta debe ser mayor que tu oferta actual.");
```
Si ya has ofertado, tu nueva oferta debe ser mayor. Esto previene spam y asegura que las ofertas solo suban.

### Paso 2: Rastrear Nuevos Postores

```solidity
if (bids[msg.sender] == 0) {
    bidders.push(msg.sender);
}
```
Si esta es tu primera oferta (oferta actual es 0), te agrega a la lista de postores. De esta manera podemos rastrear a todos los participantes.

### Paso 3: Registrar la Oferta

```solidity
bids[msg.sender] = amount;
```
Actualiza tu monto de oferta. Esto sobrescribe tu oferta anterior.

### Paso 4: Verificar si Eres el Nuevo Líder

```solidity
if (amount > highestBid) {
    highestBid = amount;
    highestBidder = msg.sender;
}
```
¡Si tu oferta supera la actual más alta, te conviertes en el nuevo líder!

**Nota importante:** Esta es una subasta simplificada. En una implementación real, querrías manejar transferencias reales de ETH, reembolsos para usuarios superados, y más. Pero esto enseña la lógica central maravillosamente.

---

## Finalizando la Subasta

Una vez que se acaba el tiempo, alguien necesita finalizar la subasta:

```solidity
function endAuction() external {
    require(block.timestamp >= auctionEndTime, "La subasta aún no ha terminado.");
    require(!ended, "El final de la subasta ya fue llamado.");
    ended = true;
}
```

**¿Por qué estas verificaciones?**

1. **Verificación de tiempo** - No se puede terminar temprano. Los postores necesitan el período completo
2. **Verificación de ya terminado** - Previene llamar a esta función múltiples veces

**¿Quién puede llamar a esto?** ¡Cualquiera! Nota que no hay modificador `onlyOwner`. Una vez que se acaba el tiempo, cualquiera puede finalizarla. Este es un buen diseño - no quieres que la subasta se quede atascada si el propietario desaparece.

Establecer `ended = true` es importante porque previene casos extremos raros y marca claramente la subasta como completa.

---

## Viendo al Ganador

Después de que termina la subasta, cualquiera puede verificar quién ganó:

```solidity
function getWinner() external view returns (address, uint) {
    require(ended, "La subasta aún no ha terminado.");
    return (highestBidder, highestBid);
}
```

**Puntos clave:**

- **Función `view`** - No modifica el estado, solo lee datos. ¡Sin costo de gas cuando se llama externamente!
- **Devuelve dos valores** - La dirección del ganador y su oferta ganadora
- **Requiere que la subasta haya terminado** - No se puede espiar al ganador mientras la oferta aún está en curso

Esta función devuelve una tupla (múltiples valores). En JavaScript, la desestructurarías así:
```javascript
const [winner, winningBid] = await contract.getWinner();
```

---

## Escenarios del Mundo Real

**Escenario 1: Tres Postores**

1. Alice oferta 100 → Ella es la mejor postora
2. Bob oferta 150 → Bob toma la delantera
3. Alice oferta 200 → Alice recupera la delantera
4. El tiempo expira → Alice gana con 200

**Escenario 2: Oferta de Último Segundo**

1. Charlie oferta 100 al inicio
2. No pasa nada durante horas...
3. Diana oferta 500 con 10 segundos restantes
4. El tiempo expira → Diana gana (sin tiempo para contra-oferta)

Esto se llama "sniping" y es común en subastas. Algunas plataformas agregan mecanismos "anti-snipe" que extienden el tiempo si llegan ofertas cerca del final.

---

## ¿Qué Falta? (Y Por Qué)

Este es un contrato de enseñanza, así que simplificamos algunas cosas:

**1. Sin manejo real de ETH** - En producción, usarías `payable` y `msg.value` para manejar dinero real

**2. Sin reembolsos** - Cuando son superados, los usuarios deberían recibir su ETH de vuelta automáticamente

**3. Sin incremento mínimo de oferta** - Las subastas reales a menudo requieren que las ofertas aumenten al menos X%

**4. Sin precio de reserva** - Muchas subastas tienen un precio mínimo que el artículo debe alcanzar

**5. Sin patrón de retiro** - Los ganadores deberían poder reclamar el artículo, los vendedores deberían recibir el pago

¡Cubriremos estos patrones en contratos futuros!

---

## Conceptos Clave Que Has Aprendido

**1. Lógica basada en tiempo** - Usar `block.timestamp` para fechas límite

**2. Gestión de estado** - Rastrear estado complejo de subasta a través de múltiples variables

**3. Patrones de validación** - Múltiples declaraciones `require()` para hacer cumplir reglas

**4. Público vs. Privado** - Ocultar `highestBidder` hasta el momento apropiado

**5. Funciones view** - Leer datos sin costos de gas

---

## Consideraciones de Seguridad

**Manipulación de marca de tiempo del bloque:** Los mineros pueden manipular `block.timestamp` por unos pocos segundos. Para subastas de alto valor, esto podría importar. Considera usar números de bloque en su lugar, o acepta el pequeño riesgo.

**Front-running:** En blockchains públicas, otros pueden ver tu oferta antes de que se mine y enviar una oferta más alta con más gas para ser minada primero. Este es un problema inherente de blockchain.

**Denegación de Servicio:** Si reembolsas automáticamente a usuarios superados, un contrato malicioso podría rechazar el reembolso y romper tu subasta. Usa el patrón de retiro en su lugar (los usuarios retiran fondos en lugar de que tú los envíes).

---

## Desafíate a Ti Mismo

Extiende este contrato:

1. **Agrega manejo de ETH** - Haz `bid()` pagable y rastrea dinero real
2. **Implementa reembolsos** - Crea una función `withdraw()` para usuarios superados
3. **Agrega un precio de reserva** - La subasta solo tiene éxito si la oferta más alta cumple el mínimo
4. **Crea múltiples subastas** - Usa structs y mappings para ejecutar muchas subastas simultáneamente
5. **Agrega eventos** - Emite `BidPlaced`, `AuctionEnded`, etc. para integración de frontend

¡Acabas de construir la base de mercados descentralizados. Este mismo patrón impulsa subastas de NFT, ventas de nombres de dominio, y más. ¡Bien hecho!
