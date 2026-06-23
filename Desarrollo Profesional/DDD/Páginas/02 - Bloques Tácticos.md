# 🧩 DDD 02 — Bloques Tácticos

[[Desarrollo Profesional/DDD/DDD|⬅️ Volver a DDD]] | [[Desarrollo Profesional/DDD/Páginas/01 - Estrategia|← 01]] | [[Desarrollo Profesional/DDD/Páginas/03 - Patrones Avanzados|03 →]]

> [!abstract] Introducción
> Los bloques tácticos son los patrones de construcción del código dentro de un Bounded Context. Son los objetos con los que modelas el dominio: Entities (identidad a lo largo del tiempo), Value Objects (inmutabilidad y igualdad por valor), Aggregates (consistencia transaccional), Repositories (persistencia como abstracción) y Domain Services (lógica que no pertenece a ninguna entidad).

## ¿De qué vamos a hablar?

Los cinco bloques tácticos de DDD con implementaciones en Python: cuándo usar cada uno, cómo implementarlo y qué errores evitar.

### Conceptos que vamos a cubrir
- Entity: identidad continua a través del tiempo
- Value Object: inmutable, sin identidad propia
- Aggregate: frontera de consistencia transaccional
- Repository: persistencia como puerto del dominio
- Domain Service: lógica que no pertenece a ninguna entidad

---

## El Concepto

### Entity — Identidad a lo Largo del Tiempo

Una Entity se define por su identidad, no por sus atributos. Puede cambiar sus atributos y seguir siendo la misma entidad:

```python
from dataclasses import dataclass, field
from uuid import UUID, uuid4
from datetime import datetime, timezone

@dataclass
class Pedido:
    id: UUID
    cliente_id: UUID
    lineas: list["LineaPedido"] = field(default_factory=list)
    estado: str = "borrador"
    creado_en: datetime = field(default_factory=lambda: datetime.now(timezone.utc))

    def __eq__(self, otro: object) -> bool:
        if not isinstance(otro, Pedido):
            return False
        return self.id == otro.id  # la igualdad se basa en la identidad

    def __hash__(self) -> int:
        return hash(self.id)

    def agregar_linea(self, producto_id: UUID, cantidad: int, precio: float) -> None:
        if self.estado != "borrador":
            raise ValueError("Solo se pueden modificar pedidos en borrador")
        linea = LineaPedido(producto_id=producto_id, cantidad=cantidad, precio=precio)
        self.lineas.append(linea)

    def confirmar(self) -> None:
        if not self.lineas:
            raise ValueError("No se puede confirmar un pedido vacío")
        self.estado = "confirmado"

# Factory — para crear con identidad generada
def crear_pedido(cliente_id: UUID) -> Pedido:
    return Pedido(id=uuid4(), cliente_id=cliente_id)
```

### Value Object — Sin Identidad, con Valor

Un Value Object se define por sus atributos. Dos Value Objects con los mismos atributos son iguales. Son **inmutables** — si necesitas cambiar uno, creas uno nuevo:

```python
from dataclasses import dataclass
from decimal import Decimal

@dataclass(frozen=True)  # frozen=True hace la clase inmutable
class Dinero:
    cantidad: Decimal
    moneda: str

    def __post_init__(self) -> None:
        if self.cantidad < 0:
            raise ValueError("La cantidad no puede ser negativa")
        if len(self.moneda) != 3:
            raise ValueError("Moneda debe ser código ISO de 3 letras")

    def sumar(self, otro: "Dinero") -> "Dinero":
        if self.moneda != otro.moneda:
            raise ValueError(f"No se pueden sumar {self.moneda} y {otro.moneda}")
        return Dinero(cantidad=self.cantidad + otro.cantidad, moneda=self.moneda)

    def multiplicar(self, factor: Decimal) -> "Dinero":
        return Dinero(cantidad=self.cantidad * factor, moneda=self.moneda)

@dataclass(frozen=True)
class Direccion:
    calle: str
    ciudad: str
    codigo_postal: str
    pais: str

    def __post_init__(self) -> None:
        if not self.codigo_postal.isalnum():
            raise ValueError("Código postal inválido")

# Uso
precio_a = Dinero(Decimal("10.00"), "EUR")
precio_b = Dinero(Decimal("5.00"), "EUR")
total = precio_a.sumar(precio_b)  # Dinero(Decimal("15.00"), "EUR")

# La igualdad es por valor, no por identidad
assert Dinero(Decimal("10.00"), "EUR") == Dinero(Decimal("10.00"), "EUR")  # ✅

@dataclass(frozen=True)
class LineaPedido:
    producto_id: UUID
    cantidad: int
    precio_unitario: Dinero

    @property
    def subtotal(self) -> Dinero:
        return self.precio_unitario.multiplicar(Decimal(self.cantidad))
```

### Aggregate — Frontera de Consistencia

Un Aggregate es un grupo de objetos que se tratan como una unidad para los cambios de datos. El Aggregate Root es la entidad raíz — es el único punto de entrada para modificar el aggregate. Garantiza que todos los invariantes (reglas de negocio) se mantienen:

```python
from dataclasses import dataclass, field
from uuid import UUID, uuid4
from decimal import Decimal

@dataclass
class Carrito:  # Aggregate Root
    id: UUID
    cliente_id: UUID
    items: list["ItemCarrito"] = field(default_factory=list)
    cupon_aplicado: "Cupon | None" = None

    # Invariante: máximo 50 items diferentes
    MAX_ITEMS = 50

    def agregar_producto(self, producto_id: UUID, cantidad: int, precio: Dinero) -> None:
        if len(self.items) >= self.MAX_ITEMS:
            raise ValueError(f"Carrito lleno: máximo {self.MAX_ITEMS} productos")

        item_existente = self._buscar_item(producto_id)
        if item_existente:
            # Actualizar cantidad si ya existe
            nuevo_item = ItemCarrito(
                producto_id=producto_id,
                cantidad=item_existente.cantidad + cantidad,
                precio=precio
            )
            self.items = [i if i.producto_id != producto_id else nuevo_item for i in self.items]
        else:
            self.items.append(ItemCarrito(producto_id=producto_id, cantidad=cantidad, precio=precio))

    def eliminar_producto(self, producto_id: UUID) -> None:
        self.items = [i for i in self.items if i.producto_id != producto_id]

    def aplicar_cupon(self, cupon: "Cupon") -> None:
        if self.cupon_aplicado is not None:
            raise ValueError("Ya hay un cupón aplicado")
        if not cupon.es_valido():
            raise ValueError("El cupón ha expirado")
        self.cupon_aplicado = cupon

    @property
    def total(self) -> Dinero:
        # Dinero solo conoce .sumar() (no implementa el operador +), así que
        # acumulamos con un bucle en lugar de sum()
        subtotal = Dinero(Decimal("0"), "EUR")
        for item in self.items:
            subtotal = subtotal.sumar(item.subtotal)
        if self.cupon_aplicado:
            # Aplicamos el descuento multiplicando por el factor restante.
            # No construimos un Dinero negativo (el invariante de Dinero lo prohíbe).
            factor = (Decimal("100") - self.cupon_aplicado.porcentaje) / Decimal("100")
            return subtotal.multiplicar(factor)
        return subtotal

    def _buscar_item(self, producto_id: UUID) -> "ItemCarrito | None":
        return next((i for i in self.items if i.producto_id == producto_id), None)

@dataclass(frozen=True)
class ItemCarrito:  # Value Object dentro del Aggregate
    producto_id: UUID
    cantidad: int
    precio: Dinero

    @property
    def subtotal(self) -> Dinero:
        return self.precio.multiplicar(Decimal(self.cantidad))
```

### Repository — Persistencia como Puerto

El Repository es una abstracción de la persistencia. Desde el dominio, parece una colección en memoria. El dominio define la interfaz; la infraestructura la implementa:

```python
from abc import ABC, abstractmethod
from typing import Protocol

# Puerto (en el dominio) — solo la interfaz
class RepositorioCarrito(Protocol):
    def obtener(self, id: UUID) -> Carrito | None: ...
    def guardar(self, carrito: Carrito) -> None: ...
    def eliminar(self, id: UUID) -> None: ...

# Implementación en memoria (para tests)
class RepositorioCarritoMemoria:
    def __init__(self) -> None:
        self._storage: dict[UUID, Carrito] = {}

    def obtener(self, id: UUID) -> Carrito | None:
        return self._storage.get(id)

    def guardar(self, carrito: Carrito) -> None:
        self._storage[carrito.id] = carrito

    def eliminar(self, id: UUID) -> None:
        self._storage.pop(id, None)

# Implementación PostgreSQL (en infraestructura)
# class RepositorioCarritoPostgres:
#     def __init__(self, session: AsyncSession) -> None:
#         self._session = session
#     def obtener(self, id: UUID) -> Carrito | None:
#         # mapeo de ORM a entidad de dominio
```

### Domain Service — Lógica Sin Hogar

Un Domain Service contiene lógica de dominio que no pertenece naturalmente a ninguna entidad o Value Object. Si la operación involucra múltiples Aggregates, es candidata a Domain Service:

```python
@dataclass
class ServicioTransferencia:
    """Transferencia entre cuentas — involucra dos Aggregates"""

    repositorio_cuentas: "RepositorioCuenta"

    def transferir(self, origen_id: UUID, destino_id: UUID, importe: Dinero) -> None:
        cuenta_origen = self.repositorio_cuentas.obtener(origen_id)
        cuenta_destino = self.repositorio_cuentas.obtener(destino_id)

        if cuenta_origen is None or cuenta_destino is None:
            raise ValueError("Cuenta no encontrada")

        if cuenta_origen.moneda != importe.moneda:
            raise ValueError("Moneda incompatible")

        cuenta_origen.debitar(importe)
        cuenta_destino.acreditar(importe)

        self.repositorio_cuentas.guardar(cuenta_origen)
        self.repositorio_cuentas.guardar(cuenta_destino)
```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - **Entity**: identidad única y continua. La igualdad se basa en el ID, no en los atributos. Encapsula invariantes de negocio.
> - **Value Object**: inmutable, definido por sus atributos. `@dataclass(frozen=True)` en Python. Si necesitas uno "modificado", creas uno nuevo.
> - **Aggregate**: frontera de consistencia transaccional. El Aggregate Root es el único punto de entrada. Una transacción = un Aggregate.
> - **Repository**: el dominio ve la persistencia como una colección en memoria. Define la interfaz (Protocol); la infraestructura la implementa.
> - **Domain Service**: para lógica que involucra múltiples Aggregates o que no encaja naturalmente en ninguna entidad.

### Para llevar a la práctica
- [ ] Identifica en tu proyecto qué objetos son Entities (necesitan ID) y cuáles son Value Objects (definidos por atributos)
- [ ] Encuentra un Aggregate — qué objetos deben cambiar juntos para mantener la consistencia
- [ ] Convierte un acceso directo a BD en un Repository con Protocol

### Recursos
- 📖 *Domain-Driven Design* — Eric Evans, Capítulos 5-6 (bloques tácticos)
- 📖 *Implementing Domain-Driven Design* — Vaughn Vernon, Parte II
- 🌐 martinfowler.com/bliki/ValueObject.html
- 🌐 martinfowler.com/bliki/DDD_Aggregate.html

---
`#ddd` `#tactical` `#entity` `#value-object` `#aggregate`
