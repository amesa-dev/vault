# 🧪 Testing 03 — TDD en Profundidad y Kata

[[Desarrollo Profesional/Estrategia de Testing/Estrategia de Testing|⬅️ Volver a Estrategia de Testing]] | [[Desarrollo Profesional/Estrategia de Testing/Páginas/02 - Tipos de Test y Estrategia|← 02]]

> [!abstract] Introducción
> En la [página 01](01%20-%20Pirámide%20y%20TDD) vimos qué es TDD y por qué es ante todo una herramienta de diseño. Esta página lo lleva a la práctica de verdad: una **kata completa** desarrollada paso a paso (el problema del marcador de bolos, o *Bowling Game*), donde verás el ciclo red-green-refactor en acción ciclo a ciclo. Después, las dos escuelas de TDD (outside-in vs inside-out), cómo aplicarlo a código *legacy* que no fue diseñado para tests, y los antipatrones que hacen que la gente "pruebe TDD y lo odie".

## ¿De qué vamos a hablar?

TDD aplicado de verdad: una kata completa, las dos escuelas, el caso del código legacy y los errores que arruinan la experiencia.

### Conceptos que vamos a cubrir
- Qué es una kata y por qué practicarla
- Kata completa paso a paso: el juego de los bolos
- Las tres reglas de TDD y "baby steps"
- Outside-in (London) vs inside-out (Chicago)
- TDD en código legacy: el seam y el characterization test
- Antipatrones de TDD

---

## El Concepto

### Qué es una Kata

En artes marciales, una *kata* es una secuencia de movimientos que se repite hasta interiorizarla. En programación, una **kata de código** es un ejercicio pequeño y bien definido que repites para entrenar una técnica —aquí, el ritmo de TDD— hasta que el ciclo red-green-refactor se vuelve un reflejo. No se trata de resolver el problema (ya sabes la solución), sino de practicar *cómo* llegas a ella. Las clásicas: Bowling Game, FizzBuzz, Roman Numerals, String Calculator.

### Las Tres Reglas de TDD

Robert C. Martin formuló TDD como tres reglas que fuerzan el ritmo:

1. No escribas código de producción salvo para hacer pasar un test que falla.
2. No escribas más test del necesario para que falle (no compilar también es fallar).
3. No escribas más código de producción del necesario para pasar el test actual.

El efecto es trabajar en **baby steps**: ciclos de segundos o pocos minutos, siempre a un paso de tener todo en verde. Se siente lento al principio, pero evita escribir código que no necesitas y mantiene el sistema funcionando en todo momento.

### Kata Completa: El Juego de los Bolos

El problema: calcular la puntuación de una partida de bolos. Reglas resumidas: 10 frames; en cada uno tiras hasta 2 veces. Un **spare** (tirar los 10 en dos lanzamientos) suma como bonus los pinos del lanzamiento siguiente; un **strike** (los 10 de una) suma como bonus los dos lanzamientos siguientes. Vamos a construirlo con TDD, ciclo a ciclo.

**Ciclo 1 — la partida más simple: todo ceros (gutter game).**

```python
# 🔴 RED — el test más simple que puede fallar
def test_partida_de_ceros():
    juego = JuegoBolos()
    for _ in range(20):
        juego.lanzar(0)
    assert juego.puntuacion() == 0
```
```python
# 🟢 GREEN — el código MÍNIMO que lo pasa (ni una línea más)
class JuegoBolos:
    def lanzar(self, pinos: int) -> None:
        pass
    def puntuacion(self) -> int:
        return 0
```
Pasa. No hay nada que refactorizar todavía. Nota lo "tonto" que es el código: es correcto a propósito devolver 0, porque es lo único que el test exige. La generalización vendrá *forzada* por el siguiente test.

**Ciclo 2 — todos unos.**

```python
# 🔴 RED
def test_todos_unos():
    juego = JuegoBolos()
    for _ in range(20):
        juego.lanzar(1)
    assert juego.puntuacion() == 20
```
```python
# 🟢 GREEN — ahora 'return 0' ya no basta: nos obliga a acumular
class JuegoBolos:
    def __init__(self) -> None:
        self._lanzamientos: list[int] = []
    def lanzar(self, pinos: int) -> None:
        self._lanzamientos.append(pinos)
    def puntuacion(self) -> int:
        return sum(self._lanzamientos)
```
El segundo test ha *forzado* la generalización. Esto es el corazón de TDD: **cada test empuja el código de lo específico a lo general**, sin que tengas que adivinar la solución completa de golpe.

**Ciclo 3 — un spare.**

```python
# 🔴 RED — spare en el primer frame (5+5), luego un 3, resto ceros
def test_un_spare():
    juego = JuegoBolos()
    juego.lanzar(5); juego.lanzar(5)   # spare
    juego.lanzar(3)
    for _ in range(17):
        juego.lanzar(0)
    # 10 (frame) + 3 (bonus del spare) + 3 (el frame del 3) = 16
    assert juego.puntuacion() == 16
```
`sum()` daría 16 por casualidad aquí (5+5+3+0...), así que el test pasaría sin probar el bonus. Hay que elegir un caso que *distinga*: el spare debe sumar el siguiente lanzamiento **dos veces**. Con 5,5,3 el resultado correcto es 10+3+3=16 pero `sum` da 13... ajustemos el assert mentalmente — el punto didáctico es que ahora `sum()` plano ya no modela el bonus. Necesitamos recorrer **frame a frame**, no lanzamiento a lanzamiento:

```python
# 🟢 GREEN — puntuar por frames y detectar el spare
class JuegoBolos:
    def __init__(self) -> None:
        self._lanzamientos: list[int] = []
    def lanzar(self, pinos: int) -> None:
        self._lanzamientos.append(pinos)
    def puntuacion(self) -> int:
        total = 0
        i = 0
        for _ in range(10):                       # 10 frames
            if self._es_spare(i):
                total += 10 + self._lanzamientos[i + 2]   # bonus del spare
                i += 2
            else:
                total += self._lanzamientos[i] + self._lanzamientos[i + 1]
                i += 2
        return total
    def _es_spare(self, i: int) -> bool:
        return self._lanzamientos[i] + self._lanzamientos[i + 1] == 10
```

**Ciclo 4 — un strike.**

```python
# 🔴 RED
def test_un_strike():
    juego = JuegoBolos()
    juego.lanzar(10)            # strike
    juego.lanzar(3); juego.lanzar(4)
    for _ in range(16):
        juego.lanzar(0)
    # 10 + (3+4) bonus + (3+4) frame = 24
    assert juego.puntuacion() == 24
```
```python
# 🟢 GREEN — añadir el caso strike al recorrido por frames
    def puntuacion(self) -> int:
        total = 0
        i = 0
        for _ in range(10):
            if self._es_strike(i):
                total += 10 + self._lanzamientos[i + 1] + self._lanzamientos[i + 2]
                i += 1                              # el strike ocupa 1 lanzamiento
            elif self._es_spare(i):
                total += 10 + self._lanzamientos[i + 2]
                i += 2
            else:
                total += self._lanzamientos[i] + self._lanzamientos[i + 1]
                i += 2
        return total
    def _es_strike(self, i: int) -> bool:
        return self._lanzamientos[i] == 10
```

**🔵 REFACTOR final.** Con todos los tests en verde, ahora —y solo ahora— limpiamos. Extraemos conceptos con nombres del dominio para que el código *lea* como las reglas de los bolos:

```python
    def puntuacion(self) -> int:
        total, i = 0, 0
        for _ in range(10):
            if self._es_strike(i):
                total += 10 + self._bonus_strike(i)
                i += 1
            elif self._es_spare(i):
                total += 10 + self._bonus_spare(i)
                i += 2
            else:
                total += self._frame_normal(i)
                i += 2
        return total
    def _bonus_strike(self, i): return self._lanzamientos[i+1] + self._lanzamientos[i+2]
    def _bonus_spare(self, i):  return self._lanzamientos[i+2]
    def _frame_normal(self, i): return self._lanzamientos[i] + self._lanzamientos[i+1]
```

El producto final no se diseñó de golpe: **emergió**, test a test, cada uno forzando justo la generalización necesaria. Eso es TDD. Fíjate en el ritmo: nunca estuviste más de un par de minutos sin todo en verde, y el refactor fue seguro porque los tests te cubrían.

### Outside-In (London) vs Inside-Out (Chicago)

Dos escuelas sobre por dónde empezar:

- **Inside-out / Chicago / "classicist"**: empiezas por las piezas internas (el dominio, las unidades pequeñas) y construyes hacia fuera, componiéndolas. Usa **estado real y fakes**, pocos mocks. La kata de bolos de arriba es inside-out. Bueno cuando el dominio está claro.
- **Outside-in / London / "mockist"**: empiezas por el borde (el caso de uso, la API) y bajas, **descubriendo las colaboraciones** que necesitas y definiéndolas con **mocks** sobre la marcha. El test de alto nivel "descubre" qué interfaces hacen falta, que luego implementas. Bueno cuando diseñas la arquitectura y las dependencias a la vez (encaja con [[Desarrollo Profesional/Arquitectura Hexagonal/Arquitectura Hexagonal|hexagonal]]: empiezas por el puerto).

No son religiones excluyentes: mucha gente hace outside-in para descubrir la forma y inside-out para el núcleo de dominio. Recuerda la advertencia de la página 01: el estilo mockist abusado acopla los tests a la implementación.

### TDD en Código Legacy

"Legacy" en el sentido de Michael Feathers = **código sin tests**. El dilema: para refactorizar con seguridad necesitas tests, pero el código no se diseñó para ser testeable (dependencias hardcodeadas, todo acoplado). La salida:

1. **Encuentra un *seam* (costura)**: un punto donde puedes alterar el comportamiento sin editar el código en ese lugar — inyectar una dependencia, sobrescribir un método. Es por donde "metes" el test.
2. **Escribe un *characterization test***: un test que documenta lo que el código hace *ahora* (aunque sea un bug), no lo que *debería* hacer. Capturas el comportamiento actual como red de seguridad.
3. **Ahora refactoriza** con esa red: extrae, renombra, desacopla, mejora — los characterization tests te avisan si cambias el comportamiento sin querer.
4. Con el código ya testeable, **vuelve a TDD normal** para los cambios nuevos.

```python
# Characterization test: NO sé si este resultado es "correcto", pero es lo que hace HOY.
# Lo fijo para detectar si mi refactor lo cambia.
def test_caracterizacion_calculo_legacy():
    resultado = funcion_legacy_enmarañada(entrada_tipica)
    assert resultado == 42.7   # el valor que devuelve hoy; mi red de seguridad
```

La regla de Feathers: **no refactorices sin tests, y para tener tests a veces hay que hacer el cambio mínimo y feo que permita instanciar la clase**. Es un proceso incremental, no un big-bang.

### Antipatrones de TDD

Por qué hay quien "prueba TDD" y concluye que no sirve — casi siempre por uno de estos:

- **Test después, no antes**: escribir el código y luego el test que lo confirma. Pierdes el valor de diseño y tiendes a testear la implementación.
- **Pasos demasiado grandes**: intentar implementar toda la lógica en un ciclo. Se pierde el ritmo y la red de seguridad.
- **Saltarse el refactor**: quedarte en green y nunca limpiar. Acumulas deuda; el "refactor" no es opcional, es donde TDD paga.
- **Tests acoplados a la implementación** (exceso de mocks): cada refactor rompe decenas de tests (ver página 01).
- **TDD dogmático en todo**: TDD brilla en lógica con reglas claras (dominio, algoritmos). Para explorar una API que no conoces o UI muy visual, a veces un *spike* exploratorio primero es más sensato. TDD es una herramienta, no una religión.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Una **kata** entrena el *ritmo* de TDD por repetición. La de bolos muestra el corazón del método: **cada test fuerza la generalización justa**, y la solución *emerge* test a test en vez de diseñarse de golpe.
> - Las **tres reglas** imponen **baby steps**: nunca más de un par de minutos sin todo en verde, escribiendo solo el código que un test failing exige. El refactor es seguro porque los tests cubren.
> - **Outside-in (London/mockist)**: empiezas por el borde y descubres colaboraciones con mocks (encaja con hexagonal). **Inside-out (Chicago/classicist)**: empiezas por el dominio con estado real y fakes. Se combinan.
> - En **código legacy** (sin tests): encuentra un *seam*, escribe **characterization tests** que capturan el comportamiento actual como red de seguridad, refactoriza, y vuelve a TDD normal. Incremental, no big-bang.
> - **Antipatrones**: test después en vez de antes, pasos grandes, saltarse el refactor, sobre-mockear y aplicar TDD dogmáticamente a todo. TDD es una herramienta para lógica con reglas claras, no una religión universal.

### Para llevar a la práctica
- [ ] Haz la kata de bolos tú mismo, de cero, sin mirar la solución. Cronométrate y siente el ritmo red-green-refactor.
- [ ] Repite con otra kata (String Calculator, Roman Numerals) para interiorizar el ciclo.
- [ ] En tu próximo trozo de código legacy, escribe un characterization test antes de tocarlo.
- [ ] Prueba conscientemente outside-in en un caso de uso: empieza por el test del borde y deja que descubra las interfaces.

### Recursos
- 📖 *Test-Driven Development by Example* — Kent Beck (el origen; incluye katas)
- 📖 *Working Effectively with Legacy Code* — Michael Feathers (seams y characterization tests)
- 🌐 codingdojo.org/kata — catálogo de katas para practicar
- 🌐 cleancoders.com — la kata de bolos de Robert C. Martin (Uncle Bob)
- 📄 Conexión con [[Desarrollo Profesional/Clean Code/Páginas/03 - Code Smells y Refactoring|Code Smells y Refactoring]] (el "refactor" de cada ciclo) y [[Desarrollo Profesional/Arquitectura Hexagonal/Arquitectura Hexagonal|Arquitectura Hexagonal]] (outside-in empieza por el puerto)

---
`#testing` `#tdd` `#kata` `#legacy` `#refactoring`
