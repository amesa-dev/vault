# 🧩 DDD (Domain-Driven Design)

[[Desarrollo Profesional/Inicio Profesional|⬅️ Volver a Desarrollo Profesional]]

> [!abstract] Domain-Driven Design (DDD)
> Es un enfoque de desarrollo de software centrado en el dominio del negocio. Propone modelar el software basándose en la realidad y reglas del negocio mediante una colaboración estrecha entre expertos del dominio y desarrolladores.

---

## 🔑 Conceptos Estratégicos

### 1. Ubiquitous Language (Lenguaje Ubicuo)
Un lenguaje común y compartido por todo el equipo (técnicos y expertos de negocio). Se utiliza en la conversación diaria, en la documentación y directamente en el código (nombres de clases, variables, métodos).

### 2. Bounded Context (Contexto Acotado)
Una delimitación conceptual dentro de la cual un modelo de dominio particular es aplicable y consistente. Evita la ambigüedad (por ejemplo, el concepto de "Usuario" puede significar cosas distintas en el contexto de "Facturación" y en el de "Soporte Técnico").

---

## 🏗️ Conceptos Tácticos (Bloques de Construcción)

### 1. Entity (Entidad)
Un objeto que se define no por sus atributos, sino por una identidad única y continua a lo largo del tiempo.
* *Ejemplo:* Un `Usuario` con un `ID` único. Sus datos (nombre, email) pueden cambiar, pero sigue siendo la misma entidad.

### 2. Value Object (Objeto de Valor)
Un objeto que no tiene identidad propia y se define únicamente por el valor de sus propiedades. Son inmutables.
* *Ejemplo:* Una `Dirección` (calle, ciudad, país) o una `Moneda` (monto, divisa). Si dos direcciones tienen los mismos datos, se consideran iguales.

### 3. Aggregate (Agregado)
Un conjunto de objetos asociados (Entidades y Objetos de Valor) que se tratan como una unidad lógica para los cambios de datos. Cada agregado tiene una **raíz del agregado** (*Aggregate Root*) a través de la cual se accede y se garantiza la consistencia de las reglas de negocio.

### 4. Repository (Repositorio)
Interfaz que define las operaciones de persistencia para guardar y recuperar raíces de agregados, abstrayendo los detalles de la infraestructura de base de datos.

### 5. Domain Event (Evento de Dominio)
Algo que ocurrió en el dominio y que es de interés para el negocio. Se nombran en pasado.
* *Ejemplo:* `PedidoCreado`, `PagoRechazado`.

---
`#ddd` `#arquitectura` `#diseno` `#apuntes`
