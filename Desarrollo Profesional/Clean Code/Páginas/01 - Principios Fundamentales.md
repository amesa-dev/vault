# 🧹 Clean Code 01 — Principios Fundamentales

[[Desarrollo Profesional/Clean Code/Clean Code|⬅️ Volver a Clean Code]] | [[Desarrollo Profesional/Clean Code/Páginas/02 - SOLID|02 →]]

> [!abstract] Introducción
> Clean Code empieza por lo más visible: los nombres, las funciones y los comentarios. Son la primera línea de contacto entre el código y quien lo lee. Un nombre honesto, una función que hace exactamente lo que su nombre dice, y un comentario que existe solo cuando es imprescindible — esas tres cosas por sí solas eliminan el 80% de la deuda técnica de comunicación.

## ¿De qué vamos a hablar?

Los principios que determinan si el código es fácil de leer: nombres que revelan intención, funciones pequeñas y con propósito único, y comentarios que solo aparecen cuando el código no puede hablar por sí solo.

### Conceptos que vamos a cubrir
- Nombres que revelan intención
- Funciones: pequeñas, un propósito, sin efectos secundarios
- Comentarios: cuándo escribirlos, cuándo no
- La regla del chico scout: siempre deja el código un poco mejor

---

## El Concepto

### Nombres que Revelan Intención

El nombre debe responder la pregunta: ¿para qué existe esto? Un nombre que necesita un comentario para explicarse es un nombre malo:

```python
# ❌ Nombres sin intención
d = 0   # días desde modificación
lst = []  # lista de usuarios activos
def proc(u, f):  # procesar usuario con flag
    if f == 1:
        lst.append(u)

# ✅ Nombres que hablan
dias_desde_modificacion = 0
usuarios_activos = []

def activar_usuario(usuario: "Usuario") -> None:
    usuarios_activos.append(usuario)

# ❌ Nombres que desinforman
nombres_de_usuarios = {"Juan": 1, "Ana": 2}  # es un dict, no una lista

# ✅ Nombres precisos
id_por_nombre_usuario: dict[str, int] = {"Juan": 1, "Ana": 2}
```

**Reglas prácticas:**
- Usa nombres pronunciables: `fecha_creacion`, no `fchCr`
- Usa nombres buscables: `MAX_INTENTOS_CONEXION = 5`, no `5` literal en el código
- Evita la codificación de tipos: `nombre`, no `strNombre` ni `nombre_str`
- Para funciones: usa verbos (`calcular_total`, `obtener_usuario`, `validar_email`)
- Para booleanos: usa formas que se leen como preguntas (`es_activo`, `tiene_permisos`, `puede_editar`)
- Para clases y entidades: usa sustantivos (`Pedido`, `CalculadoraImpuestos`, `GestorConexiones`)

```python
# Nombres de funciones que revelan el nivel de abstracción
def publicar_articulo(articulo: "Articulo") -> None:  # alto nivel — concepto de negocio
    validar_articulo(articulo)
    guardar_en_base_de_datos(articulo)
    enviar_a_indice_busqueda(articulo)
    notificar_suscriptores(articulo)

# Cada subfunción está al mismo nivel de abstracción
def validar_articulo(articulo: "Articulo") -> None:
    if not articulo.titulo:
        raise ValueError("El artículo necesita un título")
    if len(articulo.contenido) < 100:
        raise ValueError("El artículo es demasiado corto")
```

### Funciones: Pequeñas, Un Propósito

Las funciones deben hacer **una sola cosa**. Si puedes describir lo que hace una función con más de una frase conectada por "y" o "además", hace demasiado:

```python
# ❌ Función que hace demasiado
def procesar_pago(pedido_id: int, tarjeta: dict, email: str) -> bool:
    # Obtener pedido
    conn = get_db_connection()
    pedido = conn.execute("SELECT * FROM pedidos WHERE id = ?", (pedido_id,)).fetchone()

    # Validar tarjeta
    if not tarjeta.get("numero") or len(tarjeta["numero"]) != 16:
        return False

    # Cobrar
    response = requests.post("https://api.pagos.com/cobrar", json={
        "tarjeta": tarjeta["numero"],
        "importe": pedido["total"]
    })
    if response.status_code != 200:
        return False

    # Actualizar BD y enviar email
    conn.execute("UPDATE pedidos SET estado = 'pagado' WHERE id = ?", (pedido_id,))
    conn.commit()
    smtplib.SMTP("smtp.gmail.com").sendmail("no-reply@...", email, "Pago recibido")
    return True

# ✅ Responsabilidades separadas
def procesar_pago(pedido_id: int, datos_pago: "DatosPago", notificador: "Notificador") -> None:
    pedido = obtener_pedido(pedido_id)
    validar_datos_pago(datos_pago)
    ejecutar_cobro(pedido, datos_pago)
    marcar_pedido_como_pagado(pedido_id)
    notificador.notificar_pago_recibido(pedido.cliente_id)

def validar_datos_pago(datos: "DatosPago") -> None:
    if not datos.numero_tarjeta.isdigit() or len(datos.numero_tarjeta) != 16:
        raise ValueError("Número de tarjeta inválido")

def ejecutar_cobro(pedido: "Pedido", datos: "DatosPago") -> None:
    respuesta = pasarela_pagos.cobrar(datos.numero_tarjeta, pedido.total)
    if not respuesta.exitoso:
        raise ErrorPago(respuesta.mensaje)
```

**Sin efectos secundarios ocultos:**

```python
# ❌ Efecto secundario oculto — el nombre no sugiere que modifica el usuario
def comprobar_contrasena(usuario: "Usuario", contrasena: str) -> bool:
    if usuario.contrasena_hash == hash(contrasena):
        usuario.ultimo_acceso = datetime.now()  # ← efecto secundario oculto
        return True
    return False

# ✅ Efectos secundarios explícitos
def verificar_contrasena(usuario: "Usuario", contrasena: str) -> bool:
    return usuario.contrasena_hash == hash(contrasena)

def registrar_acceso(usuario: "Usuario") -> None:
    usuario.ultimo_acceso = datetime.now()

# Se usa así — explícito
if verificar_contrasena(usuario, contrasena_ingresada):
    registrar_acceso(usuario)
    iniciar_sesion(usuario)
```

**Número de argumentos:**

```python
# ❌ Demasiados argumentos
def crear_usuario(nombre: str, email: str, edad: int, pais: str, ciudad: str, activo: bool) -> "Usuario":
    pass

# ✅ Agrupar en un objeto de datos
@dataclass
class DatosRegistroUsuario:
    nombre: str
    email: str
    edad: int
    pais: str
    ciudad: str
    activo: bool = True

def crear_usuario(datos: DatosRegistroUsuario) -> "Usuario":
    pass
```

### Comentarios: Cuándo y Cuándo No

El mejor comentario es el código que no lo necesita. Un comentario que explica *qué* hace el código es redundante — el código ya lo dice. El comentario valioso explica *por qué*:

```python
# ❌ Comentario que explica lo que ya se ve
# Verificar si el usuario es mayor de edad
if usuario.edad >= 18:

# ❌ Comentario de historial (para eso está git)
# 2024-01-15 - Andrés: corregido bug de autenticación
# 2024-02-03 - Luisa: añadido soporte para OAuth

# ✅ Comentario que explica por qué — una decisión no obvia
# IEEE 754: la aritmética de punto flotante acumula errores en sumas largas
# Usamos Decimal en lugar de float para cálculos financieros
total = Decimal("0")

# ✅ Advertencia sobre consecuencias
# CUIDADO: esta consulta hace full table scan si no hay índice en cliente_id
# Asegúrate de que exista el índice antes de correr en producción
pedidos = db.execute("SELECT * FROM pedidos WHERE cliente_id = %s", (cliente_id,))

# ✅ TODO — con explicación del por qué está pendiente
# TODO: migrar a asyncpg cuando Flask sea reemplazado por FastAPI (Q2 2025)
conn = psycopg2.connect(DATABASE_URL)
```

### La Regla del Chico Scout

> "Siempre deja el campamento más limpio de lo que lo encontraste."

Cada vez que tocas un archivo, déjalo un poco mejor. No hace falta un refactor masivo — pequeñas mejoras continuas acumulan calidad:

```python
# Encontraste esto al añadir una feature
def proc_u(u, t, f):
    if f:
        u.last_t = t

# Lo dejaste así — misma lógica, mejor código
def registrar_tiempo_acceso(usuario: "Usuario", timestamp: datetime, registrar: bool) -> None:
    if registrar:
        usuario.ultimo_acceso = timestamp
```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Los nombres deben revelar intención, no requerir comentarios para explicarse. Si necesitas un comentario para entender el nombre, cambia el nombre.
> - Las funciones hacen **una sola cosa**. Si describes lo que hace con "y" o "además", la divides. El nivel de abstracción dentro de una función debe ser uniforme.
> - Los efectos secundarios son el tipo de bug más difícil de encontrar. Si una función modifica estado, su nombre debe decirlo.
> - Los comentarios son para el *por qué*, no el *qué*. Si el código necesita un comentario para explicar *qué* hace, reescribe el código.
> - La regla del chico scout: deja siempre el código un poco mejor de como lo encontraste.

### Para llevar a la práctica
- [ ] Encuentra tres variables con nombres de una o dos letras en tu código y renómbralas
- [ ] Identifica una función que haga más de una cosa y sepárala
- [ ] Busca un comentario que explique *qué* hace el código y reemplázalo con código que se explique solo

### Recursos
- 📖 *Clean Code* — Robert C. Martin, Capítulos 2-4
- 📖 *The Pragmatic Programmer* — David Thomas y Andrew Hunt
- 🌐 refactoring.guru/refactoring — catálogo de técnicas de refactoring

---
`#clean-code` `#nombres` `#funciones` `#comentarios`
