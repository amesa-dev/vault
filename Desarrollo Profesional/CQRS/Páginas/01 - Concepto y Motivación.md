# 🔀 CQRS 01 — Concepto y Motivación

[[Desarrollo Profesional/CQRS/CQRS|⬅️ Volver a CQRS]] | [[Desarrollo Profesional/CQRS/Páginas/02 - Implementación en Python|02 →]]

> [!abstract] Introducción
> Casi todos arrancamos con un único modelo que sirve para todo: las mismas clases y las mismas tablas que usamos para guardar un pedido las usamos para mostrarlo en un listado. Funciona… hasta que las necesidades de leer y de escribir empiezan a tirar en direcciones opuestas. CQRS pone nombre a esa tensión y propone una solución: dos modelos, uno para mandar (commands) y otro para preguntar (queries).

## ¿De qué vamos a hablar?

De dónde viene CQRS, qué problema resuelve de verdad y cómo decidir hasta qué nivel adoptarlo sin pasarte de complejidad.

### Conceptos que vamos a cubrir
- CQS, el principio del que nace CQRS
- Qué es un command y qué es una query (y por qué no deben mezclarse)
- Por qué separar lectura y escritura desbloquea optimizaciones
- Los niveles de adopción: desde separar métodos hasta separar bases de datos

---

## El Concepto

### De CQS a CQRS

Todo empieza con **CQS** (Command Query Separation), un principio de diseño de Bertrand Meyer: *cada método o bien hace algo (command) o bien responde algo (query), pero no las dos cosas*. Una query no debe tener efectos secundarios; un command no debe devolver datos de negocio.

```python
# MAL: el método muta Y devuelve estado de negocio (mezcla command y query)
def retirar(self, importe: float) -> float:
    self.saldo -= importe
    return self.saldo          # ¿es una consulta o una orden? ambas → confuso

# BIEN: un command que ordena (sin devolver estado de dominio)...
def retirar(self, importe: float) -> None:
    if importe > self.saldo:
        raise SaldoInsuficiente()
    self.saldo -= importe

# ...y una query separada que solo lee
def consultar_saldo(self) -> float:
    return self.saldo
```

**CQRS** lleva ese principio del nivel del método al nivel de la **arquitectura**: en lugar de un único modelo que sirve para leer y escribir, tienes dos modelos distintos, cada uno optimizado para su trabajo.

### Command vs Query

| | **Command** | **Query** |
|---|---|---|
| Intención | Cambiar el estado | Obtener información |
| Nombre | Imperativo: `ConfirmarPedido` | Interrogativo: `ObtenerResumenPedido` |
| Devuelve | Nada (o solo un id/ack) | Datos (un DTO de lectura) |
| Reglas de negocio | Sí, muchas (validaciones, invariantes) | No, solo proyecta datos |
| Modelo que usa | El modelo de dominio rico (Aggregates) | Proyecciones planas optimizadas |
| Idempotencia | Importa mucho (ver [[Desarrollo Profesional/Diseño de APIs/Páginas/01 - REST y Diseño de Recursos|idempotencia en APIs]]) | Naturalmente seguras |

Un **command** es una intención de cambio en pasado-futuro ("quiero que se confirme el pedido"); puede ser rechazado si viola una regla. Una **query** nunca cambia nada: si la repites mil veces, el sistema queda igual.

### El Problema que Resuelve

Leer y escribir tienen necesidades opuestas que un modelo único no puede satisfacer bien a la vez:

```
        ESCRITURA                          LECTURA
   ┌──────────────────┐            ┌──────────────────────┐
   │ Pocas operaciones │            │ Muchísimas más        │
   │ Reglas complejas  │            │ Cero reglas           │
   │ Modelo normalizado│   ≠        │ Datos desnormalizados │
   │ Consistencia fuerte│           │ Tolera estar "casi al día" │
   │ Transaccional      │           │ Cacheable             │
   └──────────────────┘            └──────────────────────┘
```

Con un solo modelo acabas con entidades llenas de getters para la UI que ensucian el dominio, o con queries que tienen que navegar Aggregates cargando cosas que no necesitan. CQRS rompe el compromiso: el lado de escritura puede ser un modelo de dominio rico (ver [[Desarrollo Profesional/DDD/Páginas/02 - Bloques Tácticos|bloques tácticos de DDD]]) y el de lectura puede ser una vista plana, una proyección en Redis o un SQL directo a una tabla desnormalizada.

### Los Niveles de Adopción

CQRS no es todo-o-nada. Hay un gradiente, y la mayoría de proyectos se quedan (con razón) en los primeros niveles:

1. **Separar los métodos** (CQS puro). Misma clase, mismas tablas, pero los commands no devuelven datos y las queries no mutan. Gratis y siempre recomendable.
2. **Separar los modelos en código**. Objetos `Command`/`Query` y sus *handlers* dedicados, sobre la **misma base de datos**. Es el "CQRS" que usa el 90% de la gente. Lo implementamos en la [[Desarrollo Profesional/CQRS/Páginas/02 - Implementación en Python|página 02]].
3. **Separar las bases de datos**. Un almacén optimizado para escribir y otro (o varios) para leer, sincronizados por eventos. Aparece la **consistencia eventual**. Lo vemos en la [[Desarrollo Profesional/CQRS/Páginas/03 - Read Models Eventos y Escalado|página 03]].

> [!tip] La regla de oro
> Sube de nivel solo cuando el dolor lo justifique. El nivel 2 te da casi todo el beneficio de claridad con muy poco coste. El nivel 3 resuelve problemas de **escala** muy concretos, pero te trae consistencia eventual y complejidad operativa: no lo adoptes "por si acaso".

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - **CQS** dice que un método o manda o pregunta, nunca ambas. **CQRS** sube esa idea al nivel de arquitectura: dos modelos distintos para leer y escribir.
> - Un **command** expresa intención de cambio, lleva las reglas de negocio y no devuelve datos de dominio; una **query** solo proyecta datos y nunca muta nada.
> - Separar resuelve una tensión real: escritura y lectura tienen necesidades opuestas (normalización vs desnormalización, consistencia fuerte vs cacheabilidad).
> - Hay **niveles**: separar métodos (gratis), separar modelos en código (el más común), separar bases de datos (solo para escala, trae consistencia eventual).

### Para llevar a la práctica
- [ ] Revisa una clase tuya y marca qué métodos mezclan command y query; sepáralos
- [ ] Coge un caso de uso y exprésalo como un objeto `Command` con nombre imperativo y otro `Query` con nombre interrogativo
- [ ] Decide honestamente en qué nivel de CQRS estás y si subir te resolvería un dolor real

### Recursos
- 🌐 martinfowler.com/bliki/CQRS.html — la introducción canónica y sus advertencias
- 📄 Greg Young — *CQRS Documents* (el material original que popularizó el patrón)
- 📖 *Implementing Domain-Driven Design* — Vaughn Vernon (capítulo de arquitectura)

---
`#cqrs` `#cqs` `#arquitectura` `#commands` `#queries`
