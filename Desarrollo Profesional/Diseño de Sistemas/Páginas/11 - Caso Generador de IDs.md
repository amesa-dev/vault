# 🏛️ Caso de Estudio — Generador de IDs Únicos (Snowflake)

[[Desarrollo Profesional/Diseño de Sistemas/Diseño de Sistemas|⬅️ Volver a Diseño de Sistemas]] | [[Desarrollo Profesional/Diseño de Sistemas/Páginas/10 - Caso Autocompletado|← 10]] | [[Desarrollo Profesional/Diseño de Sistemas/Páginas/12 - Caso Web Crawler|12 →]]

> [!abstract] Introducción
> Generar identificadores únicos suena trivial —`AUTO_INCREMENT` y listo— hasta que tienes una base de datos sharded y ese contador único deja de existir. ¿Cómo generas IDs únicos, en muchos servidores a la vez, sin coordinación que los frene, y que además sean ordenables por tiempo? Este caso pequeño y autocontenido aparece dentro de muchos otros (el [[Desarrollo Profesional/Diseño de Sistemas/Páginas/06 - Caso URL Shortener|acortador]], el [[Desarrollo Profesional/Diseño de Sistemas/Páginas/08 - Caso Chat en Tiempo Real|chat]]) y la solución canónica —el ID de Snowflake de Twitter— es elegante y vale la pena conocer bit a bit.

## ¿De qué vamos a hablar?

Cómo generar IDs únicos a escala distribuida sin un contador central, con la estructura del ID de Snowflake.

### Conceptos que vamos a cubrir
- Requisitos: único, ordenable, distribuido
- Por qué el auto-incremento y el UUID no bastan
- El ID de Snowflake: la anatomía de 64 bits
- Reloj, secuencia y el problema del clock skew
- Alternativas: rango de tickets

---

## El Diseño

### 1. Requisitos

**Funcionales**: generar identificadores **únicos** a nivel global.

**No funcionales**: **distribuido** (muchos generadores en paralelo, sin un cuello de botella central), **alto rendimiento** (miles/millones por segundo), y a ser posible **ordenable por tiempo** (que un ID mayor signifique "creado después") — muy útil para ordenar mensajes, posts, etc. sin un campo extra.

### 2. Por Qué Auto-Incremento y UUID No Bastan

- **`AUTO_INCREMENT` de la BD**: único y ordenable, pero requiere una **única fuente** (un contador central). En una BD sharded no hay contador único; y centralizarlo crea un cuello de botella y un punto único de fallo. No escala horizontalmente.
- **UUID v4 (aleatorio)**: se genera localmente sin coordinación (genial para distribuir) y la probabilidad de colisión es despreciable. Pero es **grande (128 bits)** y **no ordenable por tiempo** (es aleatorio), lo que además perjudica el rendimiento de los índices B-Tree (inserciones desordenadas → fragmentación; ver [[Desarrollo Profesional/PostgreSQL/Páginas/02 - Indexación y Performance|índices]]). UUID v7 mitiga esto incrustando tiempo.

Necesitamos algo que combine lo mejor: generación local sin coordinación **y** ordenable por tiempo **y** compacto. Eso es Snowflake.

### 3. El ID de Snowflake — Anatomía de 64 Bits

Twitter diseñó un ID de **64 bits** (cabe en un entero, compacto) compuesto por partes que, juntas, garantizan unicidad sin coordinación:

```
 0 | 41 bits de timestamp       | 10 bits de máquina | 12 bits de secuencia
 ↑   ↑                            ↑                     ↑
signo  milisegundos desde una     identifica el          contador dentro del
(0)    época propia (~69 años)    nodo/máquina (1024)    mismo ms (4096/ms)
```

- **1 bit de signo**: siempre 0 (para que el número sea positivo).
- **41 bits de timestamp**: milisegundos desde una época personalizada. 41 bits ≈ 69 años de rango. Como va en los bits altos, **los IDs son ordenables por tiempo** (uno generado después tiene timestamp mayor → ID mayor).
- **10 bits de ID de máquina**: identifican cuál de hasta **1.024** nodos generó el ID. Dos máquinas distintas nunca colisionan porque esta parte difiere.
- **12 bits de secuencia**: un contador que se incrementa para IDs generados **en el mismo milisegundo en la misma máquina** → hasta **4.096 IDs por ms por máquina**. Se reinicia cada milisegundo.

La genialidad: **cada máquina genera IDs sola, sin hablar con nadie**, y la unicidad está garantizada porque (timestamp, máquina, secuencia) nunca se repite: si dos IDs comparten milisegundo y máquina, difieren en la secuencia; si comparten máquina pero no ms, difieren en el timestamp; si son máquinas distintas, difieren en el ID de máquina.

```python
class Snowflake:
    def __init__(self, machine_id: int, epoca: int = 1288834974657) -> None:
        self.machine_id = machine_id          # 0..1023, asignado por máquina
        self.epoca = epoca
        self.ultimo_ms = -1
        self.secuencia = 0

    def generar(self) -> int:
        ahora = self._ahora_ms()
        if ahora == self.ultimo_ms:
            self.secuencia = (self.secuencia + 1) & 0xFFF   # 12 bits: 0..4095
            if self.secuencia == 0:                          # se agotó este ms
                ahora = self._esperar_siguiente_ms(self.ultimo_ms)
        else:
            self.secuencia = 0
        self.ultimo_ms = ahora
        return (((ahora - self.epoca) << 22)                 # timestamp en bits altos
                | (self.machine_id << 12)                    # máquina
                | self.secuencia)                            # secuencia
```

### 4. El Problema del Clock Skew

El talón de Aquiles: Snowflake depende del **reloj de la máquina**. Si el reloj **retrocede** (por un ajuste de NTP), podrías generar un timestamp menor que el anterior y, potencialmente, un ID duplicado o desordenado. Defensas:
- **Detectar el retroceso**: si `ahora < ultimo_ms`, rechazar generar (o esperar a alcanzar el último timestamp) en vez de emitir un ID dudoso.
- Usar relojes monótonos donde sea posible y NTP bien configurado (sin saltos bruscos).

Es el eco de [[Desarrollo Profesional/Sistemas Distribuidos/Páginas/01 - Fundamentos y CAP|que el tiempo en distribuido no es de fiar]]: incluso un esquema que usa el reloj debe protegerse de él.

### 5. Alternativa: Rango de Tickets

Otra opción más simple: un **servidor de tickets** central reparte **rangos** (en vez de IDs uno a uno). Cada generador pide un bloque ("del 1.000.001 al 1.002.000") y los va consumiendo localmente; cuando se le acaban, pide otro. Reduce la presión sobre el central (una petición por bloque, no por ID), pero sigue habiendo un punto central (replícalo) y los IDs no codifican tiempo. Es lo que usan algunos sistemas (Flickr lo hizo con dos servidores de tickets para HA).

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Generar IDs únicos a escala distribuida no es trivial: `AUTO_INCREMENT` necesita un **contador central** (cuello de botella, no shardeable); **UUID v4** se genera local pero es grande y **no ordenable por tiempo** (y fragmenta índices).
> - **Snowflake (64 bits)** combina lo mejor: `[1 signo | 41 timestamp | 10 máquina | 12 secuencia]`. Cada máquina genera sola, sin coordinación, y la unicidad está garantizada por la combinación irrepetible de las tres partes.
> - Al ir el **timestamp en los bits altos**, los IDs son **ordenables por tiempo** — útil para ordenar posts/mensajes sin campo extra. 1024 máquinas × 4096 IDs/ms cada una.
> - **Clock skew** es su debilidad: si el reloj retrocede, rechaza generar o espera, para no emitir IDs duplicados/desordenados. El tiempo en distribuido no es de fiar.
> - Alternativa: **servidor de tickets** que reparte **rangos** (un round-trip por bloque, no por ID); más simple pero con punto central y sin tiempo codificado.

### Para llevar a la práctica
- [ ] Implementa el generador Snowflake de arriba y verifica que dos instancias con `machine_id` distinto nunca colisionan.
- [ ] Calcula cuántos IDs por segundo puede emitir el sistema entero (máquinas × IDs/ms × 1000).
- [ ] Razona qué pasaría si el reloj retrocede 5ms y cómo lo proteges.
- [ ] Compara el comportamiento de un índice B-Tree insertando UUIDs v4 vs IDs Snowflake (orden).

### Recursos
- 📖 *System Design Interview, Vol. 1* — Alex Xu (capítulo 7: Design a Unique ID Generator in Distributed Systems)
- 🌐 github.com/twitter-archive/snowflake — el proyecto original
- 📄 RFC 9562 — UUID v7 (UUIDs ordenables por tiempo)
- 📄 Conexión con [[Desarrollo Profesional/PostgreSQL/Páginas/02 - Indexación y Performance|índices]] y [[Desarrollo Profesional/Diseño de Sistemas/Páginas/03 - Almacenamiento Replicación y Sharding|sharding]]

---
`#diseño-de-sistemas` `#snowflake` `#id-generation` `#caso-estudio`
