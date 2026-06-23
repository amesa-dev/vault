# 🔐 Seguridad 02 — Autenticación y Autorización

[[Desarrollo Profesional/Seguridad Aplicada/Seguridad Aplicada|⬅️ Volver a Seguridad Aplicada]] | [[Desarrollo Profesional/Seguridad Aplicada/Páginas/01 - OWASP y Vulnerabilidades|← 01]] | [[Desarrollo Profesional/Seguridad Aplicada/Páginas/03 - Criptografía y Secretos|03 →]]

> [!abstract] Introducción
> Dos preguntas distintas que se confunden constantemente. **Autenticación** (AuthN): ¿quién eres? **Autorización** (AuthZ): ¿qué puedes hacer? Esta página explica cómo se resuelven en sistemas modernos: la diferencia entre sesiones y tokens, el protocolo OAuth2 (que es de autorización, no de login, pese a usarse para login), OpenID Connect (que sí es de autenticación), la anatomía de un JWT y sus trampas, y los modelos de autorización RBAC y ABAC.

## ¿De qué vamos a hablar?

Cómo verificar la identidad de quien llama y controlar qué puede hacer, con los protocolos estándar de la industria.

### Conceptos que vamos a cubrir
- AuthN vs AuthZ: la distinción fundamental
- Sesiones con estado vs tokens sin estado
- OAuth2: roles, flujos y qué problema resuelve
- OpenID Connect: autenticación sobre OAuth2
- JWT: anatomía, validación y errores peligrosos
- Modelos de autorización: RBAC y ABAC

---

## El Concepto

### AuthN vs AuthZ

- **Autenticación (AuthN)**: probar quién eres. Contraseña, passkey, certificado, código de un segundo factor. El resultado es una identidad confirmada.
- **Autorización (AuthZ)**: una vez sé quién eres, ¿tienes permiso para esta acción sobre este recurso? El resultado es permitir o denegar.

Mezclarlas es la raíz de los fallos de control de acceso (ver [[Desarrollo Profesional/Seguridad Aplicada/Páginas/01 - OWASP y Vulnerabilidades|OWASP 01]]). Estar autenticado no implica estar autorizado para todo.

### Sesiones vs Tokens

Dos formas de recordar que un usuario ya se autenticó:

**Sesión con estado (stateful)**: tras el login, el servidor crea una sesión en su almacén (Redis, BD) y manda al cliente una cookie con un ID de sesión opaco. En cada petición busca la sesión por ese ID.
- *Pro*: revocación instantánea (borras la sesión del almacén). Control total.
- *Contra*: el servidor guarda estado; escalar horizontalmente exige un almacén de sesiones compartido.

**Token sin estado (stateless, típicamente JWT)**: tras el login, el servidor firma un token que contiene la identidad y los claims. El cliente lo manda en cada petición; el servidor lo **verifica con la firma**, sin consultar ningún almacén.
- *Pro*: sin estado en el servidor, escala trivialmente, va bien entre servicios.
- *Contra*: **no se puede revocar fácilmente** antes de su expiración (no hay dónde "borrarlo"). Por eso los access tokens son de vida corta.

La práctica habitual combina ambos: **access token JWT de vida corta** (5-15 min) + **refresh token de vida larga** guardado de forma revocable. Cuando el access token caduca, el cliente usa el refresh para obtener uno nuevo; si hay que cortar el acceso, se revoca el refresh.

### OAuth2 — Autorización Delegada

OAuth2 resuelve un problema concreto: **dar a una aplicación acceso limitado a tus recursos en otro servicio, sin darle tu contraseña**. Cuando "inicias sesión con Google" en una app de terceros, esa app no ve tu contraseña de Google; recibe un token con permisos acotados.

Los cuatro roles:
- **Resource Owner**: tú, el usuario.
- **Client**: la aplicación que quiere acceder en tu nombre.
- **Authorization Server**: quien autentica y emite tokens (Google, Auth0, Keycloak).
- **Resource Server**: la API que guarda tus datos y acepta el token.

El flujo recomendado hoy es **Authorization Code + PKCE**:

```
1. La app redirige al usuario al Authorization Server (con un code_challenge).
2. El usuario se autentica y consiente los permisos (scopes) pedidos.
3. El servidor redirige de vuelta con un "authorization code" de un solo uso.
4. La app intercambia ese code (+ code_verifier de PKCE) por un access token.
5. La app llama al Resource Server con el access token en Authorization: Bearer.
```

**PKCE** (Proof Key for Code Exchange) evita que un atacante que intercepte el code lo canjee: la app genera un secreto (`code_verifier`), manda su hash (`code_challenge`) en el paso 1, y debe presentar el secreto original en el paso 4. Es obligatorio para apps públicas (SPA, móvil) y recomendado para todas.

> [!warning] OAuth2 no es autenticación
> OAuth2 solo dice "este token tiene permiso para X". No fue diseñado para decir "este usuario es Andrés". Usar el access token de OAuth2 como prueba de identidad es un error clásico que ha causado vulnerabilidades reales. Para autenticación, usa OpenID Connect.

### OpenID Connect (OIDC)

OIDC es una capa fina **sobre** OAuth2 que sí estandariza la autenticación. Añade un **ID Token** (un JWT) que contiene claims sobre la identidad del usuario (`sub` = identificador, `email`, `name`) y está pensado para que el cliente sepa *quién* se autenticó. Regla simple: **OAuth2 para autorización (acceder a una API), OIDC para autenticación (login)**. "Login con Google/Microsoft" usa OIDC.

### JWT — Anatomía y Trampas

Un JSON Web Token tiene tres partes en Base64URL separadas por puntos: `header.payload.signature`.

```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9   ← header  {"alg":"RS256","typ":"JWT"}
.eyJzdWIiOiIxMjMiLCJyb2xlIjoiYWRtaW4ifQ ← payload {"sub":"123","role":"admin","exp":...}
.SflKxwRJSMeKKF2QT4fwpMeJf36...          ← signature (firma sobre header+payload)
```

Lo crucial: el **payload no está cifrado**, solo firmado. Cualquiera puede leerlo (es Base64, no encriptación). **Nunca metas secretos en un JWT.** La firma garantiza *integridad* (que nadie lo alteró), no *confidencialidad*.

```python
import jwt   # PyJWT
from datetime import datetime, timezone, timedelta

CLAVE_PRIVADA = ...   # para firmar (RS256: asimétrica)
CLAVE_PUBLICA = ...   # para verificar

def emitir_token(usuario_id: str, rol: str) -> str:
    ahora = datetime.now(timezone.utc)
    payload = {
        "sub": usuario_id,
        "role": rol,
        "iat": ahora,
        "exp": ahora + timedelta(minutes=15),   # vida corta
        "iss": "https://auth.miempresa.com",     # emisor
        "aud": "api.miempresa.com",               # audiencia
    }
    return jwt.encode(payload, CLAVE_PRIVADA, algorithm="RS256")

def verificar_token(token: str) -> dict:
    return jwt.decode(
        token,
        CLAVE_PUBLICA,
        algorithms=["RS256"],            # ← FIJA el algoritmo, ver abajo
        issuer="https://auth.miempresa.com",
        audience="api.miempresa.com",
        options={"require": ["exp", "iss", "aud", "sub"]},
    )   # lanza excepción si la firma, exp, iss o aud no cuadran
```

Errores peligrosos con JWT:
- **`alg: none`**: algunas librerías mal configuradas aceptan tokens "sin firma". Un atacante pone `alg:none` y el token pasa sin verificar. **Fija siempre los algoritmos permitidos** en la verificación (`algorithms=["RS256"]`), nunca confíes en el `alg` del header.
- **Confusión RS256/HS256**: si verificas con un algoritmo simétrico (HS256) usando la *clave pública* como secreto, un atacante puede firmar tokens con esa clave pública (que es… pública). Fija el algoritmo esperado.
- **No validar `exp`, `iss`, `aud`**: un token válido para otro servicio o ya caducado no debe aceptarse. Valídalos explícitamente.
- **Meterlos en `localStorage`**: accesible por JavaScript → robable vía XSS. Prefiere cookies `HttpOnly` o memoria.

### RBAC vs ABAC

Dos modelos para decidir el "¿puede hacer esto?":

- **RBAC (Role-Based Access Control)**: los permisos se agrupan en roles, y a los usuarios se les asignan roles. "Los `editor` pueden publicar artículos." Simple, auditable, suficiente para la mayoría de los casos. Es el modelo de [[Desarrollo Profesional/Kubernetes/Páginas/04 - Configuración y Operación|RBAC de Kubernetes]] y de [[Desarrollo Profesional/GCP/Páginas/03 - Networking e IAM|IAM de GCP]].
- **ABAC (Attribute-Based Access Control)**: la decisión depende de atributos del usuario, el recurso y el contexto. "Un médico puede ver una historia clínica **si** está asignado al paciente **y** es horario laboral **y** desde la red del hospital." Más expresivo y granular, pero más complejo de razonar y auditar.

```python
# RBAC simple
PERMISOS = {
    "admin":  {"leer", "escribir", "borrar"},
    "editor": {"leer", "escribir"},
    "lector": {"leer"},
}
def puede(usuario, accion: str) -> bool:
    return accion in PERMISOS.get(usuario.rol, set())

# ABAC: la política evalúa atributos en contexto
def puede_ver_historia(medico, historia, ahora) -> bool:
    return (historia.paciente_id in medico.pacientes_asignados
            and medico.de_guardia(ahora))
```

Empieza con RBAC; pasa a ABAC (o a un motor de políticas como OPA/Cedar) solo cuando las reglas de negocio realmente dependan del contexto.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - **AuthN** (¿quién eres?) ≠ **AuthZ** (¿qué puedes hacer?). Estar autenticado no te autoriza a todo.
> - **Sesiones** (estado en servidor, revocables al instante) vs **tokens JWT** (sin estado, escalan, pero no se revocan fácil). La práctica: access token corto + refresh token revocable.
> - **OAuth2** es autorización delegada (acceso a una API sin dar tu contraseña); usa Authorization Code + **PKCE**. **No** lo uses como prueba de identidad.
> - **OIDC** añade autenticación sobre OAuth2 con un **ID Token**. OAuth2 → autorizar, OIDC → login.
> - Un **JWT** está firmado, no cifrado: el payload es legible, nunca metas secretos. Al verificar, **fija el algoritmo** (`alg:none` y confusión RS256/HS256 son ataques reales) y valida `exp`/`iss`/`aud`.
> - **RBAC** (roles, simple, suficiente casi siempre) vs **ABAC** (atributos+contexto, granular pero complejo). Empieza por RBAC.

### Para llevar a la práctica
- [ ] Revisa cómo verificas JWTs: ¿fijas `algorithms=[...]`? ¿validas `exp`, `iss`, `aud`?
- [ ] Comprueba dónde guarda tu frontend los tokens. Si es `localStorage`, son robables por XSS.
- [ ] Asegúrate de que tus access tokens son de vida corta y existe un mecanismo de refresh/revocación.
- [ ] Documenta tu modelo de autorización: ¿RBAC? ¿Dónde se comprueban los permisos?

### Recursos
- 🌐 oauth.net/2 — especificaciones y guías de OAuth2
- 🌐 openid.net/connect — OpenID Connect
- 🌐 jwt.io — depurador de JWT y librerías
- 🌐 owasp.org/www-project-cheat-sheets — "Authentication" y "JSON Web Token" cheat sheets
- 📄 RFC 6749 (OAuth2), RFC 7519 (JWT), RFC 7636 (PKCE)

---
`#seguridad` `#oauth2` `#oidc` `#jwt` `#rbac`
