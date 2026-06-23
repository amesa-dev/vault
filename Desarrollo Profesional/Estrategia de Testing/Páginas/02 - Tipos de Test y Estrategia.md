# 🧪 Testing 02 — Tipos de Test y Estrategia

[[Desarrollo Profesional/Estrategia de Testing/Estrategia de Testing|⬅️ Volver a Estrategia de Testing]] | [[Desarrollo Profesional/Estrategia de Testing/Páginas/01 - Pirámide y TDD|← 01]] | [[Desarrollo Profesional/Estrategia de Testing/Páginas/03 - TDD en Profundidad y Kata|03 →]]

> [!abstract] Introducción
> La página anterior cubrió la base (unitarios, TDD, mocks). Esta sube por la pirámide y más allá: los tests de integración (con BD real), los end-to-end, el contract testing entre servicios, y técnicas que mucha gente no conoce —property-based testing y mutation testing— que encuentran bugs que los tests escritos a mano no ven. Cierra con el tema más malinterpretado de todos: la cobertura de código, qué mide realmente y por qué un 100% puede ser una mentira.

## ¿De qué vamos a hablar?

Los niveles superiores de testing y las técnicas avanzadas, más cómo leer las métricas de calidad sin engañarte.

### Conceptos que vamos a cubrir
- Tests de integración: con dependencias reales (Testcontainers)
- End-to-end: cuándo valen la pena y cuándo no
- Contract testing entre servicios
- Property-based testing: el ordenador genera los casos
- Mutation testing y la verdad sobre la cobertura

---

## El Concepto

### Tests de Integración

Un test de integración verifica que tu código funciona con sus dependencias **reales** (o equivalentes fieles), no mockeadas. El caso más común: tu capa de persistencia contra una base de datos real. Esto detecta lo que los unitarios con mocks no pueden: SQL mal escrito, migraciones rotas, problemas de serialización, transacciones mal gestionadas.

La trampa histórica era que montar una BD para tests era lento y frágil. La solución moderna es **Testcontainers**: levanta la dependencia real en un contenedor efímero ([[Desarrollo Profesional/Docker y Contenedores/Docker y Contenedores|Docker]]) durante el test y la destruye al acabar.

```python
import pytest
from testcontainers.postgres import PostgresContainer

@pytest.fixture(scope="session")
def db_url():
    # Levanta un PostgreSQL real en un contenedor, solo para los tests
    with PostgresContainer("postgres:16") as pg:
        yield pg.get_connection_url()
    # Al salir, el contenedor se destruye: entorno limpio y reproducible

def test_guardar_y_recuperar_pedido(db_url):
    repo = RepositorioPostgres(db_url)           # implementación REAL, no mock
    repo.guardar(Pedido(id="42", estado="confirmado"))
    recuperado = repo.obtener("42")
    assert recuperado.estado == "confirmado"     # prueba el SQL de verdad
```

Esto da confianza real sobre la capa de infraestructura sin depender de una BD compartida y sucia. Es más lento que un unitario, por eso van en el nivel medio de la pirámide: los justos.

### End-to-End

Los e2e prueban el sistema completo como un usuario: a través de la API pública o pilotando un navegador (Playwright, Cypress). Dan la confianza más alta —"esto funciona de verdad de punta a punta"— pero tienen el peor coste:

- **Lentos**: levantan todo el sistema, navegan, esperan respuestas. Minutos, no milisegundos.
- **Frágiles (flaky)**: fallan por timing, datos de entorno, una animación de la UI… motivos ajenos al bug que deberían detectar. Un e2e flaky que falla 1 de cada 10 veces erosiona la confianza en toda la suite.
- **Difíciles de depurar**: cuando uno falla, el fallo puede estar en cualquiera de las muchas piezas.

Por eso: **pocos, y solo para los flujos críticos de negocio** (registro, checkout, login). No intentes cubrir cada caso con e2e — empuja esa lógica hacia abajo en la pirámide.

### Contract Testing

El hueco entre unitarios (un servicio aislado, con mocks) y e2e (todo junto, lento): ¿cómo garantizas que dos servicios encajan sin levantarlos juntos? Con **contract testing** (Pact). El consumidor declara qué espera del proveedor (el "contrato"), y en el CI del proveedor se verifica que sigue cumpliéndolo.

```
Consumidor (servicio A)              Proveedor (servicio B)
"espero GET /pedidos/42 →     ───→   en el CI de B se verifica que
 {id, estado, total}"                 B realmente responde eso.
        (genera un pacto)             Si B cambia y rompe el contrato,
                                      su CI falla ANTES de desplegar.
```

Detecta roturas de integración en CI, rápido y sin orquestar todo el sistema. Especialmente valioso en arquitecturas de microservicios donde los equipos despliegan independientemente (enlaza con [[Desarrollo Profesional/Diseño de APIs/Páginas/02 - gRPC GraphQL y Alternativas|Diseño de APIs 02]]).

### Property-Based Testing

En vez de escribir casos concretos a mano, declaras una **propiedad que siempre debe cumplirse** y el framework (Hypothesis en Python) **genera cientos de entradas** —incluyendo los casos límite raros que no se te habrían ocurrido— intentando romperla. Cuando encuentra un fallo, lo *reduce* al ejemplo mínimo que lo reproduce.

```python
from hypothesis import given, strategies as st

# Propiedad: serializar y deserializar devuelve el original (round-trip)
@given(st.dictionaries(st.text(), st.integers()))
def test_serializacion_round_trip(datos):
    assert deserializar(serializar(datos)) == datos
# Hypothesis prueba con dicts vacíos, claves unicode raras, enteros
# enormes, etc. Si algo rompe el round-trip, te da el caso mínimo.
```

Es potentísimo para lógica con invariantes claras: parsers, serialización, operaciones matemáticas, estructuras de datos. Encuentra los bugs de los bordes que los tests de ejemplo siempre olvidan.

### Mutation Testing y la Verdad sobre la Cobertura

La **cobertura de código** mide qué % de líneas ejecutan tus tests. Es útil para encontrar zonas **sin testear**, pero tiene un fallo enorme: **mide que el código se ejecuta, no que se verifica.** Un test sin un solo `assert` puede dar 100% de cobertura y no probar nada.

```python
# Este test da cobertura del 100% de `calcular`... y no comprueba NADA
def test_inutil():
    calcular(5, 3)   # se ejecuta (cuenta para cobertura), pero ningún assert
```

El **mutation testing** (mutmut, Cosmic Ray en Python) mide la calidad real: introduce pequeños cambios ("mutantes") en tu código —cambia un `+` por un `-`, un `<` por un `<=`, un `True` por `False`— y comprueba si **algún test falla**. Si los tests siguen pasando con el código mutado, ese test no estaba verificando esa lógica de verdad: el mutante "sobrevive".

```
Código original:  if edad >= 18:
Mutante:          if edad >  18:   ← ¿algún test detecta la diferencia?
  - Si un test falla → el mutante "muere" → buena cobertura de ese caso.
  - Si todos pasan  → el mutante "sobrevive" → falta un test para el borde 18.
```

El **mutation score** (% de mutantes detectados) es una medida mucho más honesta que la cobertura de líneas. Es lento (ejecuta la suite muchas veces), así que se usa puntualmente sobre la lógica crítica, no en cada commit.

> [!tip] La cobertura es un suelo, no un techo
> Persigue cobertura para encontrar lo no testeado, pero no la conviertas en objetivo ("hay que llegar al 90%"): la gente escribe tests sin asserts para cumplir la métrica (Ley de Goodhart: "cuando una medida se vuelve objetivo, deja de ser buena medida"). Lo que importa es que los tests **fallen cuando el comportamiento se rompe** — y eso lo mide el mutation testing, no la cobertura.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - **Tests de integración**: con dependencias reales (BD), no mocks. **Testcontainers** levanta la dependencia en un contenedor efímero — detectan SQL roto, migraciones, serialización, lo que los unitarios no ven.
> - **End-to-end**: máxima confianza pero lentos, frágiles y difíciles de depurar. **Pocos, solo flujos críticos** (checkout, login). No cubras todo con e2e.
> - **Contract testing** (Pact) cubre el hueco entre unitarios y e2e: verifica que dos servicios encajan sin levantarlos juntos, detectando roturas en CI. Clave en microservicios.
> - **Property-based testing** (Hypothesis): declaras una propiedad y el framework genera cientos de entradas buscando romperla, encontrando los casos límite que no escribirías a mano.
> - **La cobertura mide ejecución, no verificación**: un test sin asserts da 100%. El **mutation testing** mide la calidad real (¿falla algún test si muto el código?). La cobertura es un suelo para encontrar huecos, no un objetivo.

### Para llevar a la práctica
- [ ] Añade un test de integración con Testcontainers para tu capa de persistencia más crítica.
- [ ] Cuenta tus e2e: ¿cubren solo los flujos críticos o intentan cubrirlo todo? Recorta y empuja lógica hacia abajo.
- [ ] Prueba Hypothesis en una función con invariantes claras (un parser, una serialización). Verás bugs de borde.
- [ ] Corre mutation testing (mutmut) sobre un módulo con alta cobertura. Te sorprenderá cuántos mutantes sobreviven.

### Recursos
- 🌐 testcontainers.com — dependencias reales y efímeras en tests
- 🌐 hypothesis.readthedocs.io — property-based testing en Python
- 🌐 mutmut.readthedocs.io — mutation testing en Python
- 🌐 docs.pact.io — contract testing
- 📖 *Unit Testing Principles, Practices, and Patterns* — Vladimir Khorikov

---
`#testing` `#integracion` `#e2e` `#mutation-testing` `#cobertura`
