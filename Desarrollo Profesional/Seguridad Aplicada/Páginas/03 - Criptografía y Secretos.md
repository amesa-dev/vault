# 🔐 Seguridad 03 — Criptografía y Secretos

[[Desarrollo Profesional/Seguridad Aplicada/Seguridad Aplicada|⬅️ Volver a Seguridad Aplicada]] | [[Desarrollo Profesional/Seguridad Aplicada/Páginas/02 - Autenticación y Autorización|← 02]]

> [!abstract] Introducción
> No necesitas ser criptógrafo para escribir software seguro, pero sí necesitas saber qué herramienta usar para cada cosa y, sobre todo, qué no hacer nunca. Esta página cubre la criptografía aplicada del día a día: cómo guardar contraseñas (hashing), cómo proteger datos (cifrado simétrico y asimétrico), cómo gestionar secretos sin meterlos en el código, y cómo funciona la confianza en la red (TLS, mTLS) y en la cadena de suministro del software. La regla de oro: **no inventes tu propia criptografía; usa primitivas estándar bien.**

## ¿De qué vamos a hablar?

La criptografía práctica que un desarrollador usa, y la gestión de secretos y confianza que la rodea.

### Conceptos que vamos a cubrir
- Hashing de contraseñas: por qué bcrypt/argon2 y no SHA-256
- Cifrado simétrico vs asimétrico: cuándo cada uno
- Gestión de secretos: fuera del código, siempre
- TLS y mTLS: confianza en tránsito
- Seguridad de la cadena de suministro (supply chain)

---

## El Concepto

### Hashing de Contraseñas

Nunca guardes contraseñas en claro. Pero tampoco con un hash criptográfico normal (SHA-256, MD5): esos están diseñados para ser **rápidos**, y la rapidez es exactamente lo que ayuda a un atacante a probar miles de millones de contraseñas por segundo si roba tu base de datos. Para contraseñas se usan **funciones de derivación lentas y con sal**: **bcrypt**, **scrypt** o **Argon2** (el ganador de la Password Hashing Competition, recomendado hoy).

```python
# ❌ MAL — rápido, sin sal: vulnerable a rainbow tables y fuerza bruta
import hashlib
hash = hashlib.sha256(password.encode()).hexdigest()

# ✅ BIEN — Argon2: lento, con sal automática, resistente a GPU/ASIC
from argon2 import PasswordHasher
ph = PasswordHasher()
hash = ph.hash(password)        # incluye la sal y los parámetros en el propio hash
# ... más tarde, al validar:
try:
    ph.verify(hash, password_introducida)   # constante en tiempo
    es_valida = True
except Exception:
    es_valida = False
```

Conceptos clave:
- **Sal (salt)**: un valor aleatorio único por contraseña, almacenado junto al hash. Hace que dos usuarios con la misma contraseña tengan hashes distintos y anula las *rainbow tables*. bcrypt/argon2 la gestionan por ti.
- **Factor de coste / work factor**: ajusta cuánto cuesta calcular el hash. Súbelo con los años conforme el hardware mejora.
- **Comparación en tiempo constante**: comparar hashes con `==` puede filtrar información por timing. Las librerías de hashing comparan en tiempo constante; para otros secretos usa `hmac.compare_digest`.

### Cifrado Simétrico vs Asimétrico

- **Simétrico**: la misma clave cifra y descifra. Rápido, para grandes volúmenes de datos. El estándar es **AES**, y debes usarlo en un modo **autenticado** (AES-GCM) que garantiza confidencialidad *e* integridad. Nunca uses ECB.
- **Asimétrico**: par de claves pública/privada. Lo que cifras con la pública solo se descifra con la privada (y viceversa para firmar). Lento, para intercambiar claves o firmar. Algoritmos: RSA, o curvas elípticas (Ed25519, ECDSA).

En la práctica se combinan (**cifrado híbrido**, lo que hace TLS): se usa criptografía asimétrica para acordar una clave simétrica, y luego se cifra el grueso de los datos con la simétrica (rápida).

```python
# Cifrado simétrico autenticado con la librería `cryptography`
from cryptography.fernet import Fernet

clave = Fernet.generate_key()       # guárdala en un gestor de secretos, no en el código
f = Fernet(clave)                   # Fernet = AES-128-CBC + HMAC, autenticado
token = f.encrypt(b"dato sensible")
original = f.decrypt(token)         # falla si el token fue manipulado
```

> [!warning] No inventes criptografía
> El error más caro en cripto no es elegir mal el algoritmo: es implementarlo mal (un IV reutilizado, un modo inseguro, una comparación con timing leak). Usa librerías de alto nivel y auditadas (`cryptography` en Python, libsodium/NaCl), que toman las decisiones seguras por ti. Si te ves escribiendo XOR o gestionando IVs a mano, párate.

### Gestión de Secretos

Las claves de API, contraseñas de BD, claves de cifrado y tokens **nunca** van en el código ni en el repositorio (un `git push` accidental los expone para siempre; el historial de git recuerda). Tampoco son ideales en variables de entorno en texto plano para secretos críticos. La jerarquía, de peor a mejor:

1. ❌ Hardcodeados en el código / commiteados.
2. ⚠️ Variables de entorno (mejor, pero visibles en logs/`/proc`, sin rotación ni auditoría).
3. ✅ **Gestor de secretos dedicado**: GCP Secret Manager, AWS Secrets Manager, HashiCorp Vault. Ofrecen cifrado en reposo, control de acceso por IAM, rotación automática y auditoría de quién accedió.

```python
# Leer un secreto de GCP Secret Manager en arranque, nunca hardcodear
from google.cloud import secretmanager

def obtener_secreto(nombre: str, proyecto: str) -> str:
    cliente = secretmanager.SecretManagerServiceClient()
    ruta = f"projects/{proyecto}/secrets/{nombre}/versions/latest"
    respuesta = cliente.access_secret_version(name=ruta)
    return respuesta.payload.data.decode("UTF-8")
```

Complementos: **detección de secretos en CI** (gitleaks, trufflehog) para evitar que se cuelen al repo, y **rotación** periódica para que un secreto filtrado tenga ventana de validez corta. Si un secreto se expone, **rótalo, no lo borres del historial** y confíes — asume que ya se copió.

### TLS y mTLS

**TLS** (lo que pone la "s" en HTTPS) da tres garantías sobre la conexión: confidencialidad (cifrado), integridad (no manipulado) y **autenticación del servidor** (un certificado firmado por una CA de confianza prueba que hablas con quien crees). El cliente verifica el certificado del servidor; el servidor no sabe quién es el cliente (eso lo resuelve la capa de auth de arriba).

**mTLS** (mutual TLS) añade que **el cliente también presenta un certificado** y el servidor lo verifica. Así ambos extremos se autentican criptográficamente. Es el estándar para comunicación **servicio a servicio** dentro de una arquitectura de microservicios — un service mesh (Istio, Linkerd) lo establece de forma transparente entre pods, de modo que cada servicio prueba su identidad sin gestionar contraseñas. Encaja con el concepto de **Zero Trust**: no confíes en la red, verifica cada conexión.

### Seguridad de la Cadena de Suministro

Tu código depende de cientos de paquetes que no escribiste. Un atacante que compromete una dependencia popular (o publica una con un nombre parecido — *typosquatting*) ejecuta código en tu build y en producción. Defensas:

- **Lockfiles con hashes** (`poetry.lock`, `package-lock.json`): fijan versiones exactas y verifican integridad, para que no se cuele una versión alterada.
- **Escaneo de dependencias** (Dependabot, `pip-audit`, Snyk): avisa de CVEs conocidas en lo que usas.
- **SBOM** (Software Bill of Materials): inventario de todo lo que compone tu artefacto, para saber qué está afectado cuando aparece una vulnerabilidad (como pasó con Log4Shell).
- **Firma de artefactos** (Sigstore/cosign): firmar imágenes de contenedor y verificar la firma antes de desplegar, para garantizar que lo que corre es lo que construiste.
- **Mínimo privilegio en CI**: el pipeline tiene credenciales potentes; trátalo como producción (ver [[Desarrollo Profesional/CI-CD e IaC/CI-CD e IaC|CI/CD e IaC]]).

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - **Contraseñas**: nunca en claro ni con SHA-256 (demasiado rápido). Usa **Argon2/bcrypt/scrypt** con sal (las librerías la gestionan) y comparación en tiempo constante.
> - **Cifrado**: simétrico (AES-GCM, rápido, grandes datos) vs asimétrico (RSA/curvas, para acordar claves y firmar). TLS combina ambos (híbrido). Usa modos **autenticados**, nunca ECB.
> - **No inventes criptografía**: usa librerías de alto nivel auditadas (`cryptography`, libsodium). Los fallos están en la implementación, no en el algoritmo.
> - **Secretos**: jamás en el código/repo. Usa un **gestor de secretos** (Secret Manager, Vault) con IAM, rotación y auditoría. Escanea el repo para evitar fugas; si se filtra, rota.
> - **TLS** autentica al servidor y cifra el canal; **mTLS** autentica también al cliente — el estándar servicio-a-servicio (Zero Trust, service mesh).
> - **Cadena de suministro**: lockfiles con hashes, escaneo de dependencias (CVEs), SBOM, firma de artefactos y CI con mínimo privilegio.

### Para llevar a la práctica
- [ ] Revisa cómo hasheas contraseñas. Si es SHA/MD5, migra a Argon2/bcrypt.
- [ ] Busca secretos hardcodeados o en el historial de git (usa `gitleaks`). Rota los que encuentres.
- [ ] Activa Dependabot/`pip-audit` en tus repos para vigilar CVEs en dependencias.
- [ ] Si tienes microservicios, evalúa mTLS vía service mesh para el tráfico interno.

### Recursos
- 🌐 cryptography.io — la librería de referencia en Python (y su guía de qué usar)
- 🌐 owasp.org/www-project-cheat-sheets — "Password Storage", "Cryptographic Storage", "Secrets Management"
- 🌐 latacora.micro.blog/2018/04/03/cryptographic-right-answers.html — qué primitiva usar para cada cosa
- 🌐 slsa.dev — framework de seguridad de la cadena de suministro
- 📖 *Serious Cryptography* — Jean-Philippe Aumasson

---
`#seguridad` `#criptografia` `#secretos` `#tls` `#supply-chain`
