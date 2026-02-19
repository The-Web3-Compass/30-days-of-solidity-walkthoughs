# Contrato Admin-Only: Tu Primer Sistema de Control de Acceso

¡Bienvenido de nuevo! Hoy estamos abordando algo absolutamente crítico en el desarrollo de smart contracts: **control de acceso**.

Piénsalo - no querrías que cualquiera pudiera pausar tu contrato, acuñar tokens ilimitados, o drenar la tesorería, ¿verdad? Ahí es donde entra el concepto de funciones "solo para propietario".

## El Problema Que Estamos Resolviendo

Imagina que estás construyendo un cofre del tesoro digital en la blockchain. Quieres:
- Solo TÚ (el propietario) puedas agregar tesoro al cofre
- Solo TÚ apruebes quién puede retirar tesoro
- Pero CUALQUIERA que apruebes debería poder retirar su cantidad aprobada

Sin control de acceso apropiado, cualquiera podría llamar a tus funciones de administrador y causar estragos. Vamos a construir un sistema que prevenga eso.

---

## Configurando la Propiedad

Primero, necesitamos establecer quién es el propietario. Esto sucede cuando se despliega el contrato:

```solidity
address public owner;

constructor() {
    owner = msg.sender;
}
```

**¿Qué está pasando aquí?**

- `msg.sender` es una variable especial que representa a quien sea que esté llamando a la función
- En el constructor, `msg.sender` es quien desplegó el contrato
- Guardamos esa dirección como el `owner`

Esta es una configuración única. Una vez desplegado, el propietario está bloqueado. (¡A menos que agregues una función para transferir la propiedad, lo cual exploraremos más adelante!)

---

## La Magia de los Modificadores

Aquí es donde las cosas se ponen interesantes. En lugar de escribir la misma verificación de permisos en cada función de administrador, creamos un **modificador** - una pieza de código reutilizable que actúa como un guardián:

```solidity
modifier onlyOwner() {
    require(msg.sender == owner, "Acceso denegado: Solo el propietario puede realizar esta acción");
    _;
}
```

**Desglosando esto:**

1. **`modifier onlyOwner()`** - Estamos definiendo un nuevo modificador llamado `onlyOwner`
2. **`require(msg.sender == owner, ...)`** - Esto verifica si el llamador es el propietario. Si no, la transacción revierte con un mensaje de error
3. **`_;`** - Este es un marcador de posición que dice "ejecuta el código de la función real aquí"

Piensa en el `_;` como un marcador. El modificador se ejecuta primero, verifica permisos, y si todo está bien, ejecuta la función en la ubicación del `_;`.

**¿Por qué es esto poderoso?** En lugar de copiar la misma declaración `require` en 10 funciones diferentes, solo agregas `onlyOwner` a cada una. Limpio, mantenible, y menos propenso a errores.

---

## Usando el Modificador: Agregando Tesoro

Ahora veamos el modificador en acción:

```solidity
uint256 public treasureAmount;

function addTreasure(uint256 amount) public onlyOwner {
    treasureAmount += amount;
}
```

**¿Qué sucede cuando alguien llama a esta función?**

1. El modificador `onlyOwner` se ejecuta primero
2. Verifica: "¿Es `msg.sender` igual a `owner`?"
3. Si SÍ → Continúa al cuerpo de la función y agrega tesoro
4. Si NO → Revierte la transacción con "Acceso denegado..."

Sin el modificador, cualquiera podría llamar a `addTreasure(1000000)` e inflar tu cofre del tesoro. Con él, solo el propietario puede.

---

## Aprobando Retiros

Aquí hay un caso de uso más sofisticado - el propietario puede aprobar cantidades específicas para personas específicas:

```solidity
mapping(address => uint256) public withdrawalAllowance;

function approveWithdrawal(address recipient, uint256 amount) public onlyOwner {
    require(amount <= treasureAmount, "No hay suficiente tesoro disponible");
    withdrawalAllowance[recipient] = amount;
}
```

**El patrón:**

1. **Función solo para propietario** - Solo el propietario puede aprobar retiros
2. **Verificación de seguridad** - No se puede aprobar más de lo que hay en el cofre
3. **Registrar la asignación** - Almacenar cuánto puede retirar esta persona

¡Esto es similar a cómo funcionan los tokens ERC-20 con la función `approve()`!

---

## Retirando Tesoro: Acceso de Dos Niveles

Ahora la parte interesante - una función con lógica diferente para el propietario vs. usuarios regulares:

```solidity
function withdrawTreasure(uint256 amount) public {
    if (msg.sender == owner) {
        // El propietario puede retirar cualquier cosa
        require(amount <= treasureAmount, "No hay suficiente tesorería disponible para esta acción.");
        treasureAmount -= amount;
        return;
    }
    
    // Los usuarios regulares solo pueden retirar su cantidad aprobada
    require(amount <= withdrawalAllowance[msg.sender], "No tienes aprobación para esta cantidad");
    require(amount <= treasureAmount, "No hay suficiente tesoro en el cofre");
    
    withdrawalAllowance[msg.sender] -= amount;
    treasureAmount -= amount;
}
```

**Dos caminos a través de esta función:**

**Camino 1 (Propietario):**
- Verificar si hay suficiente tesoro
- Retirar cualquier cantidad
- ¡Listo!

**Camino 2 (Usuario Regular):**
- Verificar si tienen aprobación para esta cantidad
- Verificar si hay suficiente tesoro
- Deducir de su asignación
- Deducir del cofre del tesoro

Nota que no usamos el modificador `onlyOwner` aquí porque QUEREMOS que los usuarios regulares también lo llamen - solo tienen diferentes permisos.

---

## Por Qué Este Patrón Importa

El control de acceso está en todas partes en Web3:

- **Contratos de tokens** - Solo el propietario puede acuñar nuevos tokens
- **DAOs** - Solo la gobernanza puede ejecutar propuestas
- **Mercados** - Solo el propietario puede pausar el comercio
- **Contratos actualizables** - Solo el admin puede actualizar la implementación

El modificador `onlyOwner` es tu primera línea de defensa contra el acceso no autorizado.

---

## Patrones Comunes Que Verás

**1. Transferir Propiedad:**
```solidity
function transferOwnership(address newOwner) public onlyOwner {
    require(newOwner != address(0), "El nuevo propietario no puede ser la dirección cero");
    owner = newOwner;
}
```

**2. Múltiples Roles:**
```solidity
mapping(address => bool) public admins;

modifier onlyAdmin() {
    require(admins[msg.sender], "No eres un admin");
    _;
}
```

**3. Restricciones Basadas en Tiempo:**
```solidity
modifier afterDeadline() {
    require(block.timestamp > deadline, "Demasiado temprano");
    _;
}
```

---

## Consideraciones de Seguridad

**Ten cuidado con la propiedad:**
- Si pierdes la clave privada del propietario, pierdes el control del contrato para siempre
- Si la clave del propietario se ve comprometida, un atacante tiene control total
- Considera usar billeteras multi-firma para cuentas de propietario en producción

**Prueba tus modificadores:**
- Asegúrate de que realmente previenen el acceso no autorizado
- Prueba casos extremos (¿qué pasa si el propietario es address(0)?)
- Verifica que los mensajes de error sean claros

---

## Lo Que Has Aprendido

Ahora entiendes:
- Cómo establecer la propiedad del contrato
- Cómo crear modificadores reutilizables
- Cómo implementar control de acceso basado en roles
- Por qué el control de acceso es crítico para la seguridad

Este es conocimiento fundamental. Casi todos los smart contracts del mundo real usan alguna forma de control de acceso. Verás `onlyOwner` y patrones similares en todas partes.

## Desafíate a Ti Mismo

Intenta extender este contrato:
- Agrega una función `renounceOwnership()` que establezca el propietario a address(0)
- Crea un modificador `onlyApproved` para usuarios con asignación no cero
- Agrega un historial de retiros usando eventos
- Implementa un sistema de permisos multi-nivel (propietario, admin, usuario)

¡A continuación, combinaremos el control de acceso con lógica más compleja. Estás construyendo una base sólida!
