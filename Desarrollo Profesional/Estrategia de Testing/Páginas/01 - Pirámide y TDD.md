# 🧪 Testing 01 — Pirámide y TDD

[[Desarrollo Profesional/Estrategia de Testing/Estrategia de Testing|⬅️ Volver a Estrategia de Testing]] | [[Desarrollo Profesional/Estrategia de Testing/Páginas/02 - Tipos de Test y Estrategia|02 →]] | [[Desarrollo Profesional/Estrategia de Testing/Páginas/03 - TDD en Profundidad y Kata|03 →]]

> [!abstract] Introducción
> Antes de escribir tests hay que decidir qué tipo de tests escribir y en qué proporción. La pirámide de tests es el modelo clásico para ese reparto, y aunque tiene matices modernos, sigue siendo la mejor heurística. Esta página cubre la pirámide, el ciclo de TDD (que es más una técnica de diseño que de testing), qué hace que un test sea bueno o malo, y el uso —y abuso— de los mocks.

## ¿De qué vamos a hablar?

Cómo repartir el esfuerzo de testing y cómo escribir tests que protejan comportamiento sin frenar el desarrollo.

### Conceptos que vamos a cubrir
- La pirámide de tests y por qué tiene esa forma
- TDD: red-green-refactor como herramienta de diseño
- Qué hace bueno a un test: testear comportamiento, no implementación
- Los dobles de test: mock, stub, fake, spy
- El abuso de mocks y los tests frágiles

---

## El Concepto

### La Pirámide de Tests

La pirámide (Mike Cohn) propone una distribución del esfuerzo de testing según el nivel:

```
            /\
           /e2e\          Pocos: lentos, frágiles, caros. Validan
          /------\        flujos críticos de punta a punta.
         /integr. \       Algunos: prueban que las piezas encajan
        /----------\      (BD real, servicios reales).
       /  unitarios  \    Muchos: rápidos, aislados, estables.
      /--------------\     La base de tu confianza.
```

La forma no es arbitraria; refleja un equilibrio de **coste vs confianza**:
- **Unitarios** (base): prueban una unidad aislada (función, clase) sin dependencias externas. Milisegundos, deterministas, fáciles de localizar cuando fallan. Muchos.
- **Integración** (medio): prueban que varias piezas funcionan juntas (tu código + una BD real, dos servicios). Más lentos, pero detectan lo que los unitarios no: errores de SQL, serialización, configuración.
- **End-to-end** (cima): prueban el sistema completo como lo usaría un usuario (a través de la UI o la API pública). Dan la máxima confianza pero son lentos, frágiles (fallan por motivos ajenos al bug) y caros de mantener. Pocos, solo para los flujos más críticos.

> [!tip] El antipatrón del cono de helado
> Si tienes muchos e2e y pocos unitarios (la pirámide invertida, *ice cream cone*), tu suite es lenta, inestable y difícil de depurar: un fallo no te dice qué se rompió. Empuja la lógica hacia tests unitarios y reserva los e2e para los caminos felices más importantes. Una variante moderna es el **trofeo de tests** (Kent C. Dodds), que da más peso a los de integración — válido, pero el principio se mantiene: **cuantos más arriba, menos.**

### TDD — Red, Green, Refactor

Test-Driven Development invierte el orden habitual: **escribes el test antes que el código**. El ciclo:

1. **🔴 Red**: escribe un test que falle (describe el comportamiento que aún no existe).
2. **🟢 Green**: escribe el código *mínimo* para que pase (sin elegancia, solo que pase).
3. **🔵 Refactor**: limpia el código con la red de seguridad del test pasando.

```python
# 🔴 Red — primero el test, describe QUÉ quiero
def test_descuento_aplica_porcentaje():
    carrito = Carrito(total=Decimal("100"))
    carrito.aplicar_descuento(porcentaje=10)
    assert carrito.total == Decimal("90")
# (falla: aplicar_descuento no existe aún)

# 🟢 Green — el código mínimo para pasar
def aplicar_descuento(self, porcentaje: int) -> None:
    self.total *= (Decimal("100") - porcentaje) / Decimal("100")

# 🔵 Refactor — mejora el diseño con el test cubriéndote
```

El valor de TDD no es solo "tener tests": es una **herramienta de diseño**. Escribir el test primero te obliga a pensar la *interfaz* desde el punto de vista de quien la usa, antes de enredarte en la implementación. Tiende a producir código más desacoplado y testeable (porque si es difícil de testear, lo notas inmediatamente). No es obligatorio para todo, pero es especialmente potente en lógica de dominio con reglas claras.

### Qué Hace Bueno a un Test

El principio más importante: **testea el comportamiento observable, no la implementación**. Un test debe verificar *qué* hace el código (su contrato), no *cómo* lo hace por dentro. Si un test se rompe cuando refactorizas sin cambiar el comportamiento, es un mal test.

```python
# ❌ MAL — testea la implementación (con qué métodos internos lo hace)
def test_malo():
    servicio = ServicioPedido(repo_mock)
    servicio.confirmar(pedido_id)
    repo_mock.guardar.assert_called_once()       # acoplado a CÓMO lo hace
    repo_mock._validar.assert_called()           # detalle interno

# ✅ BIEN — testea el comportamiento (el resultado observable)
def test_bueno():
    repo = RepositorioMemoria()                  # fake, no mock
    servicio = ServicioPedido(repo)
    servicio.confirmar(pedido_id)
    assert repo.obtener(pedido_id).estado == "confirmado"   # el QUÉ
```

Otras propiedades de un buen test (a veces resumidas como **F.I.R.S.T.**):
- **Fast** (rápido): si la suite tarda minutos, dejas de ejecutarla.
- **Independent** (independiente): no depende del orden ni del estado de otros tests.
- **Repeatable** (repetible): mismo resultado siempre, sin depender de la hora, la red o datos externos (los tests *flaky* destruyen la confianza).
- **Self-validating** (auto-validable): pasa o falla sin interpretación manual.
- **Timely** (oportuno): escrito cerca del código que prueba.

Y el patrón de estructura **Arrange-Act-Assert**: preparar el escenario, ejecutar la acción, verificar el resultado. Un test, una razón para fallar.

### Los Dobles de Test

Cuando una unidad depende de algo externo (BD, API, reloj), lo sustituyes por un **doble**. Hay varios tipos, y confundirlos lleva a malos tests:

| Doble | Qué hace | Cuándo |
|-------|----------|--------|
| **Dummy** | objeto de relleno que no se usa | cumplir una firma |
| **Stub** | devuelve respuestas predefinidas | controlar lo que devuelve una dependencia |
| **Fake** | implementación real pero simplificada (ej. repo en memoria) | sustituir una BD con algo funcional y rápido |
| **Spy** | registra cómo fue llamado, para verificarlo después | comprobar efectos (se envió un email) |
| **Mock** | doble programado con expectativas que se verifican | verificar interacciones concretas |

La distinción clave **stub vs mock**: un stub *responde* (testeas estado: el resultado); un mock *verifica que fue llamado de cierta forma* (testeas interacción: el comportamiento). Prefiere stubs/fakes y aserciones sobre el estado; usa mocks con moderación, solo cuando la interacción *es* lo que importa (que se publicó un evento, que se llamó al servicio de pago).

### El Abuso de Mocks

El error más común al madurar en testing: **mockearlo todo**. Una suite donde cada dependencia es un mock con `assert_called_with(...)` está acoplada a la implementación: cualquier refactor rompe decenas de tests aunque el comportamiento sea idéntico, y peor, **los tests pueden pasar mientras el sistema real está roto** (el mock devuelve lo que tú dijiste, no lo que la dependencia real haría).

Heurísticas:
- Prefiere **fakes** (un repositorio en memoria, ver [[Desarrollo Profesional/DDD/Páginas/02 - Bloques Tácticos|el RepositorioMemoria de DDD]]) sobre mocks cuando puedas.
- **Mockea solo los límites del sistema** (APIs externas, el reloj, servicios de terceros), no tus propias clases internas.
- Si necesitas mockear mucho para testear una clase, suele ser una señal de que está demasiado acoplada — el test te está diciendo algo del diseño.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - La **pirámide**: muchos unitarios (rápidos, aislados, la base de la confianza), algunos de integración (las piezas juntas, con BD/servicios reales), pocos e2e (máxima confianza pero lentos y frágiles). Evita el **cono de helado** invertido.
> - **TDD** (red-green-refactor) es ante todo una **herramienta de diseño**: escribir el test primero fuerza a pensar la interfaz desde el consumidor y produce código más desacoplado.
> - Un buen test verifica **comportamiento observable, no implementación**: si se rompe al refactorizar sin cambiar el comportamiento, es un mal test. Cumple **F.I.R.S.T.** y usa Arrange-Act-Assert.
> - **Dobles**: stub (responde, testeas estado) vs mock (verifica llamadas, testeas interacción) vs fake (implementación simplificada). Prefiere fakes y aserciones sobre estado.
> - **No mockees todo**: mockea solo los límites del sistema (APIs externas, reloj). El exceso de mocks acopla los tests a la implementación y puede pasar con el sistema roto. Mucho mock = señal de mal diseño.

### Para llevar a la práctica
- [ ] Dibuja la forma real de tu suite: ¿es una pirámide o un cono de helado? ¿dónde está el desequilibrio?
- [ ] Toma un test que use muchos mocks y reescríbelo con un fake, asertando sobre el estado resultante.
- [ ] Prueba TDD en la próxima función de lógica de dominio que escribas: test primero.
- [ ] Busca tests *flaky* (que fallan a veces): suelen depender del tiempo, el orden o la red. Arréglalos o bórralos.

### Recursos
- 📖 *Test-Driven Development by Example* — Kent Beck
- 📖 *Unit Testing Principles, Practices, and Patterns* — Vladimir Khorikov (excelente sobre mocks vs estado)
- 🌐 martinfowler.com/articles/mocksArentStubs.html — la distinción mock/stub
- 🌐 martinfowler.com/bliki/TestPyramid.html

---
`#testing` `#tdd` `#piramide` `#mocks`
