# 🧹 Clean Code 02 — Principios SOLID

[[Desarrollo Profesional/Clean Code/Clean Code|⬅️ Volver a Clean Code]] | [[Desarrollo Profesional/Clean Code/Páginas/01 - Principios Fundamentales|← 01]] | [[Desarrollo Profesional/Clean Code/Páginas/03 - Code Smells y Refactoring|03 →]]

> [!abstract] Introducción
> Los principios SOLID son cinco guías de diseño orientado a objetos formuladas por Robert C. Martin. No son reglas absolutas sino herramientas para razonar sobre el diseño: ¿por qué este código es difícil de cambiar? ¿por qué este test es frágil? SOLID da vocabulario para responder esas preguntas. Los ejemplos aquí son en Python.

## ¿De qué vamos a hablar?

Los cinco principios SOLID con ejemplos prácticos que muestran el problema que resuelven y cómo se aplican en Python.

### Conceptos que vamos a cubrir
- SRP: Single Responsibility Principle
- OCP: Open/Closed Principle
- LSP: Liskov Substitution Principle
- ISP: Interface Segregation Principle
- DIP: Dependency Inversion Principle

---

## El Concepto

### SRP — Una Razón para Cambiar

Una clase debe tener una sola razón para cambiar. Una "razón para cambiar" es un stakeholder o actor del sistema — si dos actores distintos pueden pedir cambios en la misma clase, esa clase tiene más de una responsabilidad:

```python
# ❌ Violación de SRP — Report tiene responsabilidades múltiples
class Reporte:
    def __init__(self, datos: list[dict]) -> None:
        self.datos = datos

    def calcular_total(self) -> float:  # lógica de negocio
        return sum(d["importe"] for d in self.datos)

    def formatear_html(self) -> str:   # presentación — razón de cambio 2
        return f"<table>{''.join(f'<tr><td>{d}</td></tr>' for d in self.datos)}</table>"

    def guardar_en_fichero(self, ruta: str) -> None:  # persistencia — razón de cambio 3
        with open(ruta, "w") as f:
            f.write(self.formatear_html())

# ✅ Responsabilidades separadas
@dataclass
class DatosReporte:
    lineas: list[dict]

    def calcular_total(self) -> float:   # solo lógica de negocio
        return sum(l["importe"] for l in self.lineas)

class FormateadorHtmlReporte:            # solo presentación
    def formatear(self, datos: DatosReporte) -> str:
        filas = "".join(f"<tr><td>{l}</td></tr>" for l in datos.lineas)
        return f"<table>{filas}</table>"

class ExportadorReporte:                 # solo persistencia
    def guardar(self, contenido: str, ruta: str) -> None:
        with open(ruta, "w") as f:
            f.write(contenido)
```

### OCP — Abierto para Extensión, Cerrado para Modificación

El código debe poder añadir nuevos comportamientos sin modificar el código existente. Esto se consigue dependiendo de abstracciones (interfaces/Protocols) en lugar de implementaciones concretas:

```python
# ❌ Violación de OCP — añadir un nuevo descuento requiere modificar la clase
class CalculadorDescuento:
    def calcular(self, tipo: str, precio: float) -> float:
        if tipo == "estudiante":
            return precio * 0.1
        elif tipo == "jubilado":
            return precio * 0.2
        elif tipo == "empleado":
            return precio * 0.3
        # Añadir "socio" requiere modificar esta función ← violación OCP
        return 0.0

# ✅ OCP — nuevos descuentos sin modificar código existente
from typing import Protocol

class PoliticaDescuento(Protocol):
    def calcular_descuento(self, precio: float) -> float: ...

@dataclass
class DescuentoEstudiante:
    def calcular_descuento(self, precio: float) -> float:
        return precio * 0.1

@dataclass
class DescuentoJubilado:
    def calcular_descuento(self, precio: float) -> float:
        return precio * 0.2

@dataclass
class DescuentoSocio:  # nuevo — sin tocar el código anterior
    porcentaje: float

    def calcular_descuento(self, precio: float) -> float:
        return precio * self.porcentaje

class CalculadorPrecio:
    def calcular(self, precio: float, descuento: PoliticaDescuento) -> float:
        return precio - descuento.calcular_descuento(precio)
```

### LSP — Los Subtipos Deben Sustituir a sus Bases

Si `B` es un subtipo de `A`, debes poder usar `B` en cualquier lugar donde se espera `A` sin que el programa deje de funcionar correctamente. Viola LSP cuando una subclase cambia el contrato de la clase base:

```python
# ❌ Violación de LSP — cuadrado viola las postcondiciones de rectángulo
class Rectangulo:
    def __init__(self, ancho: float, alto: float) -> None:
        self._ancho = ancho
        self._alto = alto

    @property
    def ancho(self) -> float:
        return self._ancho

    @ancho.setter
    def ancho(self, valor: float) -> None:
        self._ancho = valor

    @property
    def alto(self) -> float:
        return self._alto

    @alto.setter
    def alto(self, valor: float) -> None:
        self._alto = valor

    def area(self) -> float:
        return self._ancho * self._alto

class Cuadrado(Rectangulo):  # parece razonable matemáticamente...
    def __init__(self, lado: float) -> None:
        super().__init__(lado, lado)

    # Para mantener la invariante "lado == lado", cualquier setter fuerza ambos
    @Rectangulo.ancho.setter
    def ancho(self, valor: float) -> None:
        self._ancho = valor
        self._alto = valor  # ← rompe el contrato que Rectangulo garantiza

    @Rectangulo.alto.setter
    def alto(self, valor: float) -> None:
        self._ancho = valor
        self._alto = valor

# Test que falla con Cuadrado donde se espera Rectangulo
def test_rectangulo(r: Rectangulo) -> None:
    r.ancho = 5
    r.alto = 3
    assert r.area() == 15  # Pasa para Rectangulo (5*3); falla para Cuadrado (3*3=9)

# ✅ Mejor diseño — no herencia, sino tipos separados
@dataclass
class Rectangulo:
    ancho: float
    alto: float

    def area(self) -> float:
        return self.ancho * self.alto

@dataclass
class Cuadrado:
    lado: float

    def area(self) -> float:
        return self.lado ** 2

# Ambos implementan el mismo Protocol si necesitamos polimorfismo
class Figura(Protocol):
    def area(self) -> float: ...
```

### ISP — Interfaces Específicas, No Generales

Los clientes no deben depender de métodos que no usan. Una interfaz grande obliga a los implementadores a definir métodos que no necesitan:

```python
# ❌ Violación de ISP — interfaz demasiado grande
class Trabajador(Protocol):
    def trabajar(self) -> None: ...
    def comer(self) -> None: ...
    def dormir(self) -> None: ...
    def calcular_nomina(self) -> float: ...

# Un robot no come ni duerme — se ve obligado a implementar métodos vacíos
class Robot:
    def trabajar(self) -> None:
        print("Trabajando")

    def comer(self) -> None:
        pass  # Los robots no comen — interfaz contaminada

    def dormir(self) -> None:
        pass  # Los robots no duermen — interfaz contaminada

    def calcular_nomina(self) -> float:
        return 0.0  # ¿Los robots cobran? — interfaz contaminada

# ✅ ISP — interfaces pequeñas y específicas
class PuedeTrabajar(Protocol):
    def trabajar(self) -> None: ...

class PuedeAlimentarse(Protocol):
    def comer(self) -> None: ...

class TieneNomina(Protocol):
    def calcular_nomina(self) -> float: ...

class Robot:
    def trabajar(self) -> None:  # solo implementa lo que necesita
        print("Trabajando")

class EmpleadoHumano:
    def trabajar(self) -> None: ...
    def comer(self) -> None: ...
    def calcular_nomina(self) -> float: ...  # implementa todo porque todo aplica
```

### DIP — Depende de Abstracciones

Los módulos de alto nivel no deben depender de módulos de bajo nivel. Ambos deben depender de abstracciones. Las abstracciones no deben depender de los detalles:

```python
# ❌ Violación de DIP — alto nivel depende de bajo nivel
class ServicioNotificacion:
    def __init__(self) -> None:
        self._smtp = smtplib.SMTP("smtp.gmail.com")  # acoplado a Gmail

    def notificar(self, destinatario: str, mensaje: str) -> None:
        # solo puede usar SMTP
        self._smtp.sendmail("no-reply@...", destinatario, mensaje)

class GestorPedidos:
    def __init__(self) -> None:
        self._notif = ServicioNotificacion()  # acoplado a la implementación concreta

    def confirmar_pedido(self, pedido: "Pedido") -> None:
        pedido.confirmar()
        self._notif.notificar(pedido.cliente.email, "Pedido confirmado")

# ✅ DIP — ambos dependen de la abstracción
class EnviadorMensajes(Protocol):
    def enviar(self, destinatario: str, mensaje: str) -> None: ...

class EnviadorSmtp:
    def enviar(self, destinatario: str, mensaje: str) -> None:
        smtp = smtplib.SMTP("smtp.gmail.com")
        smtp.sendmail("no-reply@...", destinatario, mensaje)

class EnviadorSlack:  # alternativa — sin cambiar GestorPedidos
    def enviar(self, destinatario: str, mensaje: str) -> None:
        requests.post("https://slack.com/...", json={"text": mensaje})

class EnviadorMemoria:  # para tests — sin SMTP real
    def __init__(self) -> None:
        self.enviados: list[tuple[str, str]] = []

    def enviar(self, destinatario: str, mensaje: str) -> None:
        self.enviados.append((destinatario, mensaje))

class GestorPedidos:
    def __init__(self, notificador: EnviadorMensajes) -> None:  # depende de la abstracción
        self._notif = notificador

    def confirmar_pedido(self, pedido: "Pedido") -> None:
        pedido.confirmar()
        self._notif.enviar(pedido.cliente.email, "Pedido confirmado")
```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - **SRP**: una clase cambia por una sola razón. Si dos stakeholders distintos pueden pedir cambios, hay dos responsabilidades.
> - **OCP**: abierto para extensión (añadir comportamientos nuevos), cerrado para modificación (sin cambiar el código existente). Se consigue con `Protocol`.
> - **LSP**: sustituye cualquier subtipo por su base sin que el programa deje de funcionar. Si una subclase restringe el contrato (levanta excepciones donde la base no las lanza, o cambia las postcondiciones), viola LSP.
> - **ISP**: interfaces pequeñas y específicas. Los clientes no deben depender de métodos que no usan.
> - **DIP**: depende de abstracciones, no de implementaciones concretas. Inyecta las dependencias en lugar de crearlas dentro de la clase.

### Para llevar a la práctica
- [ ] Identifica una clase que tenga más de una razón para cambiar y separa las responsabilidades
- [ ] Convierte un `if tipo == "X"` en una jerarquía de Protocol para aplicar OCP
- [ ] Revisa dónde creas instancias de servicios concretos (BD, email) dentro de clases de alto nivel — extráelos como dependencias inyectadas

### Recursos
- 📖 *Clean Architecture* — Robert C. Martin, Parte III (SOLID en detalle)
- 🌐 blog.cleancoder.com — el blog de Uncle Bob
- 🌐 refactoring.guru/design-patterns — catálogo de patrones que aplican SOLID

---
`#clean-code` `#solid` `#srp` `#ocp` `#lsp` `#isp` `#dip`
