# 🧹 Clean Code 03 — Code Smells y Refactoring

[[Desarrollo Profesional/Clean Code/Clean Code|⬅️ Volver a Clean Code]] | [[Desarrollo Profesional/Clean Code/Páginas/02 - SOLID|← 02]]

> [!abstract] Introducción
> Los code smells son síntomas — no son bugs, pero indican problemas de diseño que eventualmente producirán bugs o harán el código imposible de mantener. Martin Fowler catalogó más de 20 en su libro *Refactoring*. Conocerlos te permite reconocer el problema antes de que se manifieste. El refactoring es el proceso disciplinado de mejorar el diseño sin cambiar el comportamiento.

## ¿De qué vamos a hablar?

Los code smells más comunes con ejemplos en Python y las técnicas de refactoring para eliminarlos.

### Conceptos que vamos a cubrir
- Los code smells más frecuentes: Long Method, God Class, Data Clumps, Feature Envy
- Técnicas de refactoring: Extract Method, Replace Conditional with Polymorphism
- El proceso de refactoring seguro: tests primero

---

## El Concepto

### Code Smell 1: Long Method (Función Larga)

Una función que hace demasiado. La guía heurística de Martin Fowler: si necesitas un comentario para explicar un bloque dentro de la función, extráelo a una función con un nombre que sea ese comentario:

```python
# ❌ Long Method — demasiados niveles de abstracción mezclados
def procesar_solicitud_credito(solicitud: dict) -> dict:
    # Validar datos personales
    if not solicitud.get("nombre"):
        return {"error": "nombre requerido"}
    if not solicitud.get("dni") or len(solicitud["dni"]) != 9:
        return {"error": "DNI inválido"}
    edad = (datetime.now() - solicitud["fecha_nacimiento"]).days // 365
    if edad < 18:
        return {"error": "debe ser mayor de edad"}

    # Calcular score crediticio
    score = 500
    if solicitud["ingresos_anuales"] > 30000:
        score += 100
    if solicitud["historial_pagos"] == "excelente":
        score += 200
    elif solicitud["historial_pagos"] == "bueno":
        score += 100
    if solicitud.get("deuda_actual", 0) > solicitud["ingresos_anuales"] * 0.5:
        score -= 200

    # Determinar resultado
    if score >= 700:
        limite = solicitud["ingresos_anuales"] * 0.3
        return {"aprobado": True, "limite": limite, "score": score}
    return {"aprobado": False, "score": score}

# ✅ Extract Method — cada sección se convierte en una función
def procesar_solicitud_credito(solicitud: dict) -> dict:
    try:
        datos = validar_datos_personales(solicitud)
        score = calcular_score_crediticio(solicitud)
        return determinar_resultado(score, solicitud["ingresos_anuales"])
    except ValueError as e:
        return {"error": str(e)}

def validar_datos_personales(solicitud: dict) -> None:
    if not solicitud.get("nombre"):
        raise ValueError("nombre requerido")
    if not solicitud.get("dni") or len(solicitud["dni"]) != 9:
        raise ValueError("DNI inválido")
    edad = (datetime.now() - solicitud["fecha_nacimiento"]).days // 365
    if edad < 18:
        raise ValueError("debe ser mayor de edad")

def calcular_score_crediticio(solicitud: dict) -> int:
    score = 500
    score += calcular_bonus_ingresos(solicitud["ingresos_anuales"])
    score += calcular_bonus_historial(solicitud["historial_pagos"])
    score += calcular_penalizacion_deuda(solicitud)
    return score

def determinar_resultado(score: int, ingresos: float) -> dict:
    if score >= 700:
        return {"aprobado": True, "limite": ingresos * 0.3, "score": score}
    return {"aprobado": False, "score": score}
```

### Code Smell 2: God Class (Clase Dios)

Una clase que sabe demasiado y hace demasiado. Se reconoce porque tiene muchos métodos sin relación entre sí y muchos atributos:

```python
# ❌ God Class — UserManager hace todo
class UserManager:
    def crear_usuario(self, ...): ...
    def autenticar(self, ...): ...
    def enviar_email_bienvenida(self, ...): ...
    def calcular_descuento_usuario(self, ...): ...
    def exportar_csv_usuarios(self, ...): ...
    def generar_informe_actividad(self, ...): ...
    def sincronizar_con_crm(self, ...): ...

# ✅ Responsabilidades separadas
class RepositorioUsuario:
    def crear(self, ...): ...
    def obtener(self, ...): ...

class ServicioAutenticacion:
    def autenticar(self, ...): ...
    def generar_token(self, ...): ...

class NotificadorUsuario:
    def enviar_bienvenida(self, ...): ...

class ExportadorUsuarios:
    def exportar_csv(self, ...): ...
```

### Code Smell 3: Data Clumps (Grupos de Datos)

Cuando el mismo grupo de variables siempre aparece junto, es señal de que deberían ser un objeto:

```python
# ❌ Data Clump — calle, ciudad, código, país siempre juntos
def registrar_envio(nombre: str, calle: str, ciudad: str, codigo_postal: str, pais: str) -> None:
    pass

def calcular_coste_envio(calle: str, ciudad: str, codigo_postal: str, pais: str) -> float:
    pass

# ✅ Introduce Parameter Object
@dataclass(frozen=True)
class Direccion:
    calle: str
    ciudad: str
    codigo_postal: str
    pais: str

    def __post_init__(self) -> None:
        if not self.codigo_postal:
            raise ValueError("Código postal requerido")

def registrar_envio(nombre: str, destino: Direccion) -> None:
    pass

def calcular_coste_envio(destino: Direccion) -> float:
    pass
```

### Code Smell 4: Feature Envy (Envidia de Características)

Un método que usa más datos de otra clase que de la suya propia. Es señal de que ese método debería estar en la otra clase:

```python
# ❌ Feature Envy — calcular_descuento usa casi solo datos de Pedido
class CalculadorPrecio:
    def calcular_descuento(self, pedido: "Pedido") -> float:
        if pedido.cliente.es_premium and pedido.total > 100:  # datos de Pedido
            return pedido.total * 0.15
        elif pedido.num_items > 10:   # datos de Pedido
            return pedido.total * 0.05
        return 0.0

# ✅ Mover el método a donde están los datos
class Pedido:
    def calcular_descuento(self) -> float:  # ahora usa sus propios datos
        if self.cliente.es_premium and self.total > 100:
            return self.total * 0.15
        elif self.num_items > 10:
            return self.total * 0.05
        return 0.0
```

### Code Smell 5: Primitive Obsession

Usar tipos primitivos donde debería haber Value Objects:

```python
# ❌ Primitive Obsession — email y teléfono son solo strings
def crear_contacto(email: str, telefono: str) -> None:
    if "@" not in email:
        raise ValueError("Email inválido")
    if not telefono.startswith("+"):
        raise ValueError("Teléfono inválido")

# El mismo código de validación se repite en cada lugar
def enviar_sms(telefono: str) -> None:
    if not telefono.startswith("+"):  # duplicado
        raise ValueError("Teléfono inválido")

# ✅ Introduce Value Object — validación encapsulada una sola vez
@dataclass(frozen=True)
class Email:
    valor: str

    def __post_init__(self) -> None:
        if "@" not in self.valor or "." not in self.valor.split("@")[1]:
            raise ValueError(f"Email inválido: {self.valor!r}")

@dataclass(frozen=True)
class Telefono:
    valor: str

    def __post_init__(self) -> None:
        if not self.valor.startswith("+") or not self.valor[1:].isdigit():
            raise ValueError(f"Teléfono inválido: {self.valor!r}")

def crear_contacto(email: Email, telefono: Telefono) -> None:
    pass  # la validación ya ocurrió al crear Email y Telefono
```

### El Proceso de Refactoring Seguro

El refactoring sin tests es trapecismo sin red. El proceso seguro:

```
1. Asegúrate de que existe un test que falla si rompes el comportamiento
2. Haz el mínimo cambio de diseño (un refactor a la vez)
3. Comprueba que todos los tests pasan
4. Repite
```

```python
# Ejemplo: Extract Method de forma segura
# Paso 1 — tienes un test que verifica el comportamiento actual
def test_calcular_descuento_cliente_premium():
    pedido = crear_pedido_premium(total=150.0)
    resultado = calcular_descuento(pedido)
    assert resultado == 22.5  # 15% de 150

# Paso 2 — extrae la lógica a una función
# La función original ahora llama a la nueva
def calcular_descuento(pedido: "Pedido") -> float:
    return _descuento_por_tipo_cliente(pedido)  # mismo comportamiento

def _descuento_por_tipo_cliente(pedido: "Pedido") -> float:
    if pedido.cliente.es_premium and pedido.total > 100:
        return pedido.total * 0.15
    return 0.0

# Paso 3 — el test pasa con la nueva estructura
# Paso 4 — repite con el siguiente smell
```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Los **code smells** son síntomas de diseño pobre, no bugs. Los más frecuentes: Long Method, God Class, Data Clumps, Feature Envy, Primitive Obsession.
> - **Extract Method** es la técnica de refactoring más usada: cuando necesitas un comentario para entender un bloque, ese bloque debería ser una función cuyo nombre sea ese comentario.
> - **Feature Envy** se detecta cuando un método usa más datos de otra clase que de la propia — mover el método a la otra clase casi siempre mejora el diseño.
> - El refactoring seguro requiere tests que fallen si rompes el comportamiento. Sin tests, estás reescribiendo, no refactorizando.

### Para llevar a la práctica
- [ ] Encuentra la función más larga de tu proyecto y aplica Extract Method para llevarla a menos de 20 líneas
- [ ] Identifica un Data Clump (un grupo de variables que siempre aparecen juntas) e introduce un Value Object
- [ ] Busca un método que use más datos de otra clase que de la propia (Feature Envy) y muévelo

### Recursos
- 📖 *Refactoring* — Martin Fowler (2ª edición, 2018) — el catálogo definitivo
- 📖 *Working Effectively with Legacy Code* — Michael Feathers — refactoring sin tests
- 🌐 refactoring.guru/refactoring/smells — catálogo visual de code smells
- 🌐 refactoring.guru/refactoring/techniques — técnicas de refactoring con ejemplos

---
`#clean-code` `#code-smells` `#refactoring` `#extract-method`
