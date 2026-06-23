# 🐳 Docker 02 — Dockerfiles y Buenas Prácticas

[[Desarrollo Profesional/Docker y Contenedores/Docker y Contenedores|⬅️ Volver a Docker y Contenedores]] | [[Desarrollo Profesional/Docker y Contenedores/Páginas/01 - Fundamentos e Imágenes|← 01]]

> [!abstract] Introducción
> Un Dockerfile mal escrito produce imágenes de 1,5 GB que tardan minutos en construirse, exponen secretos y corren como root. Uno bien escrito produce imágenes de 80 MB, se reconstruyen en segundos gracias a la caché y son seguras por defecto. La diferencia está en un puñado de técnicas: aprovechar el orden de las capas, usar builds multi-stage, elegir imágenes base mínimas y aplicar el principio de mínimo privilegio. Esta página las cubre todas, más Docker Compose para el desarrollo local.

## ¿De qué vamos a hablar?

Cómo escribir Dockerfiles que produzcan imágenes pequeñas, rápidas de construir y seguras.

### Conceptos que vamos a cubrir
- Aprovechar la caché de capas con el orden correcto
- Multi-stage builds: separar construcción de ejecución
- Imágenes base mínimas: slim, alpine, distroless
- Seguridad: usuario no-root, secretos, .dockerignore
- Docker Compose para desarrollo local

---

## El Concepto

### El Orden de las Capas y la Caché

Como cada instrucción es una capa cacheada ([página 01](01%20-%20Fundamentos%20e%20Imágenes)), y **cuando una capa cambia se invalidan todas las posteriores**, el orden determina cuánto reaprovechas entre builds. El error clásico:

```dockerfile
# ❌ MAL — copiar todo antes de instalar deps invalida la caché en cada cambio de código
FROM python:3.12-slim
WORKDIR /app
COPY . .                          # cualquier cambio en tu código...
RUN pip install -r requirements.txt   # ...obliga a reinstalar TODAS las deps
CMD ["python", "app.py"]
```

```dockerfile
# ✅ BIEN — copiar deps primero: la capa de pip solo se rehace si cambian las deps
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .           # solo este fichero
RUN pip install --no-cache-dir -r requirements.txt   # capa cacheada salvo que cambien deps
COPY . .                          # tu código cambia a menudo, pero va al final
CMD ["python", "app.py"]
```

La regla: **de lo que menos cambia a lo que más cambia.** Las dependencias cambian rara vez; tu código, en cada commit. Poniendo las deps antes, el `pip install` (lo lento) se cachea y solo se rehace cuando de verdad cambian las dependencias.

### Multi-Stage Builds

El problema: para *construir* una app necesitas compiladores, dependencias de desarrollo, herramientas… pero para *ejecutarla* no. Si todo eso queda en la imagen final, pesa de más y amplía la superficie de ataque. Los **multi-stage builds** usan varias `FROM`: una etapa construye, otra (limpia) solo copia el resultado.

```dockerfile
# --- Etapa 1: build (tiene todo el toolchain) ---
FROM python:3.12 AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

# --- Etapa 2: runtime (mínima, solo lo necesario para correr) ---
FROM python:3.12-slim AS runtime
WORKDIR /app
# Copiamos SOLO las dependencias ya instaladas de la etapa builder
COPY --from=builder /root/.local /root/.local
COPY . .
ENV PATH=/root/.local/bin:$PATH
CMD ["python", "app.py"]
# La imagen final no contiene el toolchain de build → más pequeña y segura
```

El caso más espectacular es con lenguajes compilados (Go, Rust): la etapa de build usa la imagen con el compilador (cientos de MB) y la final solo lleva el binario estático sobre una base mínima — imágenes de pocos MB.

### Imágenes Base Mínimas

Cuanto más pequeña la base, menos peso, menos tiempo de descarga y **menos CVEs** (cada paquete instalado es superficie de ataque potencial). De mayor a menor:

| Base | Tamaño aprox. | Notas |
|------|---------------|-------|
| `python:3.12` | ~1 GB | Debian completo, todo incluido. Para desarrollo. |
| `python:3.12-slim` | ~150 MB | Debian sin extras. **Buen defecto** para producción. |
| `python:3.12-alpine` | ~50 MB | Basada en Alpine (musl libc). Muy pequeña, pero `musl` puede dar problemas con paquetes que esperan glibc (ej. algunos wheels de C). |
| `gcr.io/distroless/python3` | ~50 MB | **Distroless**: sin shell, sin gestor de paquetes, solo el runtime. Máxima seguridad (no hay shell que explotar), pero difícil de depurar. |

Recomendación práctica: **slim** como defecto sensato; **distroless** para producción con foco en seguridad; cuidado con alpine si usas paquetes con extensiones en C.

### Seguridad del Contenedor

Varias prácticas que reducen drásticamente el riesgo:

```dockerfile
FROM python:3.12-slim

# ✅ Crear y usar un usuario NO-root: si comprometen el proceso, no es root del contenedor
RUN useradd --create-home --uid 1000 appuser
WORKDIR /app
COPY --chown=appuser:appuser . .
RUN pip install --no-cache-dir -r requirements.txt
USER appuser                      # el contenedor corre como appuser, no root

EXPOSE 8000
CMD ["python", "app.py"]
```

- **No corras como root**: por defecto los contenedores corren como root, y aunque está aislado, un escape de contenedor combinado con root es mucho peor. Crea un usuario sin privilegios y usa `USER`.
- **Nunca metas secretos en la imagen**: un `ENV API_KEY=...` o un `COPY` de un fichero con credenciales queda **grabado en una capa para siempre**, aunque lo borres en una capa posterior (las capas anteriores persisten). Inyecta secretos en runtime (variables de entorno, gestor de secretos — ver [[Desarrollo Profesional/Seguridad Aplicada/Páginas/03 - Criptografía y Secretos|Seguridad 03]]), o usa `RUN --mount=type=secret` para que no quede en capas.
- **`.dockerignore`**: excluye del contexto de build lo que no debe entrar (`.git`, `.env`, `node_modules`, `__pycache__`). Reduce el tamaño del contexto y evita filtrar secretos accidentalmente.
- **Escanea la imagen**: herramientas como Trivy o `docker scout` detectan CVEs conocidas en las capas. Intégralo en CI.
- **Fija la base por digest** para reproducibilidad: `FROM python:3.12-slim@sha256:...`.

### Docker Compose

Para desarrollo local, levantar manualmente una app + su base de datos + Redis es tedioso. **Docker Compose** describe un conjunto de servicios en un YAML y los levanta juntos:

```yaml
# compose.yaml
services:
  api:
    build: .
    ports: ["8000:8000"]
    environment:
      DATABASE_URL: postgresql://user:pass@db:5432/app
    depends_on:
      db: { condition: service_healthy }   # espera a que la BD esté lista

  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: pass
    volumes:
      - pgdata:/var/lib/postgresql/data    # volumen: los datos sobreviven
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user"]
      interval: 5s

volumes:
  pgdata:
```

`docker compose up` levanta todo, con una **red interna** donde los servicios se ven por nombre (`db`, `api`). Compose es para desarrollo y entornos simples; para producción a escala, ese rol lo cumple [[Desarrollo Profesional/Kubernetes/Kubernetes|Kubernetes]].

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - **Orden de capas**: de lo que menos cambia a lo que más. Copia `requirements.txt` e instala deps *antes* de copiar tu código, para que la capa lenta (`pip install`) se cachee.
> - **Multi-stage builds**: una etapa construye (con toolchain) y otra mínima solo copia el resultado. Imágenes más pequeñas y con menos superficie de ataque, espectacular con Go/Rust.
> - **Base mínima**: `slim` como defecto, `distroless` para máxima seguridad. Menos paquetes = menos peso y menos CVEs. Cuidado con alpine y paquetes con C.
> - **Seguridad**: corre como **usuario no-root** (`USER`), **nunca metas secretos en capas** (quedan grabados), usa `.dockerignore`, escanea con Trivy y fija la base por digest.
> - **Docker Compose** orquesta varios servicios (app + BD + cache) en local con una red interna; en producción a escala, eso lo hace Kubernetes.

### Para llevar a la práctica
- [ ] Reordena un Dockerfile tuyo para que las dependencias se instalen antes de copiar el código. Mide la mejora de tiempo de build.
- [ ] Convierte una imagen a multi-stage y compara el tamaño antes/después con `docker images`.
- [ ] Comprueba si tus contenedores corren como root (`docker run ... whoami`). Añade un usuario no-root.
- [ ] Busca secretos en tus imágenes (`docker history --no-trunc`) y añade un `.dockerignore`.

### Recursos
- 🌐 docs.docker.com/build/building/best-practices — buenas prácticas oficiales de Dockerfile
- 🌐 github.com/GoogleContainerTools/distroless — imágenes distroless
- 🌐 aquasecurity.github.io/trivy — escáner de vulnerabilidades de imágenes
- 🌐 docs.docker.com/compose — referencia de Docker Compose

---
`#docker` `#dockerfile` `#multi-stage` `#compose` `#seguridad`
