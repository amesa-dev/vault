# 🧩 DDD 01 — Diseño Estratégico

[[Desarrollo Profesional/DDD/DDD|⬅️ Volver a DDD]] | [[Desarrollo Profesional/DDD/Páginas/02 - Bloques Tácticos|02 →]]

> [!abstract] Introducción
> El diseño estratégico de DDD es el nivel más alto de abstracción. Responde a las preguntas: ¿Cómo dividimos un sistema complejo? ¿Cómo hace que todo el equipo hable el mismo idioma? ¿Cómo mapeamos las relaciones entre las distintas partes del negocio? Es el trabajo más valioso de DDD — y el más invisible al código.

## ¿De qué vamos a hablar?

Los tres pilares del diseño estratégico: el Ubiquitous Language (el idioma común), los Bounded Contexts (las fronteras del modelo) y el Context Mapping (cómo se relacionan esas fronteras entre sí).

### Conceptos que vamos a cubrir
- Ubiquitous Language: hablar el idioma del negocio
- Bounded Context: un modelo, una frontera
- Context Map: relaciones entre contextos
- Patrones de integración: Shared Kernel, ACL, Open Host Service

---

## El Concepto

### Ubiquitous Language — El Idioma Compartido

El Ubiquitous Language (lenguaje ubicuo) es un vocabulario compartido entre desarrolladores y expertos del negocio. No es jerga técnica ni jerga de negocio — es la intersección de ambas, acordada y usada consistentemente en conversaciones, documentación y **código**.

```python
# MAL — el código usa terminología técnica que el negocio no reconoce
class UserRecord:  # "Record" es terminología de BD, no de negocio
    def update_status_flag(self, flag: int) -> None: ...  # "flag" no significa nada

# BIEN — el código refleja el lenguaje del negocio
class Cliente:
    def activar_cuenta(self) -> None: ...
    def suspender_por_impago(self) -> None: ...

class Pedido:
    def confirmar(self) -> None: ...
    def cancelar(self, motivo: str) -> None: ...
    def marcar_como_enviado(self, numero_seguimiento: str) -> None: ...
```

El Ubiquitous Language evoluciona. Cuando el negocio descubre que "cancelar" significa cosas distintas en distintos contextos, eso es una señal de que los Bounded Contexts no están bien definidos.

### Bounded Context — Un Modelo, Una Frontera

Un Bounded Context es el límite dentro del cual un modelo de dominio es válido y consistente. El mismo concepto ("Producto") puede tener significados diferentes en distintos contextos:

```
┌──────────────────────────┐   ┌──────────────────────────┐
│   Contexto: Catálogo     │   │   Contexto: Almacén      │
│                          │   │                          │
│   Producto:              │   │   Producto:              │
│   - nombre               │   │   - sku                  │
│   - descripción          │   │   - ubicación_estante    │
│   - imágenes             │   │   - cantidad_disponible  │
│   - precio_base          │   │   - peso                 │
│   - categorías           │   │   - dimensiones          │
└──────────────────────────┘   └──────────────────────────┘
```

El mismo "Producto" tiene atributos radicalmente distintos según el contexto. Intentar crear un único modelo que sirva a ambos contextos es la causa de la mayoría de los sistemas de software inflados y difíciles de mantener.

```python
# Cada contexto tiene su propio modelo
# catalogo/domain/producto.py
from dataclasses import dataclass

@dataclass
class Producto:  # Producto en el contexto de Catálogo
    id: str
    nombre: str
    descripcion: str
    precio_base: float
    categorias: list[str]

    def publicar(self) -> None:
        if not self.nombre or not self.descripcion:
            raise ValueError("Producto incompleto para publicar")

# almacen/domain/item_inventario.py
@dataclass
class ItemInventario:  # "Producto" en el contexto de Almacén — diferente clase, diferente nombre
    sku: str
    ubicacion: str
    cantidad: int
    peso_kg: float

    def reservar(self, cantidad: int) -> None:
        if cantidad > self.cantidad:
            raise ValueError(f"Stock insuficiente: {self.cantidad} disponibles")
        self.cantidad -= cantidad
```

### Context Map — Relaciones Entre Contextos

Un Context Map describe cómo se relacionan los distintos Bounded Contexts. No es solo un diagrama — captura las relaciones organizacionales y técnicas.

**Patrones de integración:**

**Shared Kernel** — dos contextos comparten una pequeña parte del modelo. Alto acoplamiento, cambios requieren coordinación. Solo cuando el equipo es el mismo o hay coordinación muy estrecha.

**Customer/Supplier** — el contexto downstream (cliente) depende del upstream (proveedor). El proveedor define la API; el cliente la consume. El cliente tiene poder de influir en la hoja de ruta del proveedor.

**Anti-Corruption Layer (ACL)** — el contexto downstream crea una capa de traducción para proteger su modelo del lenguaje del contexto externo. Crucial cuando integras con sistemas legacy o APIs externas cuyo modelo no quieres que contamine el tuyo.

```python
# Anti-Corruption Layer — traducción del modelo externo al interno
# Escenario: sistema de pagos externo usa su propia terminología

# Modelo externo del proveedor de pagos
class ExternalPaymentResponse:
    txn_id: str
    txn_status: str  # "APPROVED", "DECLINED", "PENDING"
    amt: float
    currency_code: str

# ACL — traduce al lenguaje de nuestro dominio
from enum import Enum

class EstadoPago(Enum):
    APROBADO = "aprobado"
    RECHAZADO = "rechazado"
    PENDIENTE = "pendiente"

@dataclass
class Pago:  # Nuestro modelo interno
    id: str
    estado: EstadoPago
    importe: float
    moneda: str

class AdaptadorPagoExterno:
    """ACL: traduce el modelo del proveedor al modelo de nuestro dominio"""

    _ESTADOS = {
        "APPROVED": EstadoPago.APROBADO,
        "DECLINED": EstadoPago.RECHAZADO,
        "PENDING":  EstadoPago.PENDIENTE,
    }

    def traducir(self, respuesta: ExternalPaymentResponse) -> Pago:
        estado = self._ESTADOS.get(respuesta.txn_status)
        if estado is None:
            raise ValueError(f"Estado desconocido: {respuesta.txn_status}")

        return Pago(
            id=respuesta.txn_id,
            estado=estado,
            importe=respuesta.amt,
            moneda=respuesta.currency_code,
        )
```

**Open Host Service / Published Language** — el contexto upstream publica una API formal (REST, eventos) que otros pueden consumir. Es la forma estándar de que los microservicios se comuniquen.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - El **Ubiquitous Language** es el vocabulario compartido entre negocio y técnicos. El código debe reflejar ese vocabulario exactamente — si el negocio dice "confirmar pedido", el método se llama `confirmar()`, no `update_status()`.
> - Un **Bounded Context** es la frontera dentro de la cual un modelo es consistente. Distintos contextos pueden usar la misma palabra ("Producto") para conceptos con atributos completamente distintos — y eso está bien.
> - El **Context Map** captura las relaciones entre contextos: quién manda sobre el modelo (upstream/downstream) y cómo se traduce entre ellos.
> - La **Anti-Corruption Layer** protege tu modelo del vocabulario y la estructura de sistemas externos — nunca dejes que un modelo legacy o una API externa contamine tu dominio.

### Para llevar a la práctica
- [ ] Dibuja el Context Map de tu proyecto actual — identifica los bounded contexts y sus relaciones
- [ ] Identifica un lugar donde el mismo concepto tiene significados distintos según el contexto
- [ ] Crea un ACL para la integración con algún sistema externo que uses

### Recursos
- 📖 *Domain-Driven Design* — Eric Evans (el libro original, "el azul")
- 📖 *Implementing Domain-Driven Design* — Vaughn Vernon (más práctico)
- 🌐 martinfowler.com/bliki/BoundedContext.html
- 🌐 martinfowler.com/bliki/UbiquitousLanguage.html

---
`#ddd` `#estrategia` `#bounded-context` `#ubiquitous-language`
