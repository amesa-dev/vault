# 🐳 Docker 01 — Fundamentos e Imágenes

[[Desarrollo Profesional/Docker y Contenedores/Docker y Contenedores|⬅️ Volver a Docker y Contenedores]] | [[Desarrollo Profesional/Docker y Contenedores/Páginas/02 - Dockerfiles y Buenas Prácticas|02 →]]

> [!abstract] Introducción
> La mayoría de la gente usa contenedores durante años sin saber qué son. Y eso lleva a errores de modelo mental: tratarlos como máquinas virtuales pequeñas, esperar que persistan datos, sorprenderse de que un contenedor "se pare" cuando su proceso principal termina. Esta página deshace esos malentendidos explicando el contenedor desde su base: un proceso de Linux normal, aislado por namespaces y limitado por cgroups, ejecutando una imagen formada por capas inmutables.

## ¿De qué vamos a hablar?

Qué es realmente un contenedor a nivel de sistema operativo, y cómo se estructuran y distribuyen las imágenes.

### Conceptos que vamos a cubrir
- Contenedor vs máquina virtual: la diferencia esencial
- Namespaces: el aislamiento
- cgroups: la limitación de recursos
- Imágenes por capas y el union filesystem
- Registries, tags y el ciclo de vida

---

## El Concepto

### Contenedor vs Máquina Virtual

La diferencia fundamental está en qué se virtualiza:

```
   Máquina Virtual                      Contenedor
┌─────────────────────┐         ┌─────────────────────┐
│  App A   │  App B    │         │  App A   │  App B    │
│ Bins/Lib │ Bins/Lib  │         │ Bins/Lib │ Bins/Lib  │
│ SO huésp.│ SO huésp. │         └──────────┴───────────┘
├──────────┴───────────┤         │  Container runtime   │
│      Hypervisor      │         ├──────────────────────┤
├──────────────────────┤         │   Kernel del host    │  ← compartido
│   SO anfitrión       │         ├──────────────────────┤
├──────────────────────┤         │   SO anfitrión       │
│      Hardware        │         │      Hardware        │
└──────────────────────┘         └──────────────────────┘
```

- Una **VM** virtualiza el hardware: cada VM lleva su propio sistema operativo completo (kernel incluido) sobre un hypervisor. Aislamiento fuerte, pero pesada (GBs, arranque en minutos).
- Un **contenedor** comparte el kernel del host. No hay sistema operativo huésped: solo el proceso de tu app y sus dependencias. Es **un proceso normal de Linux** al que el kernel le da una vista aislada del sistema. Ligero (MBs, arranque en milisegundos).

La consecuencia clave: **un contenedor no "contiene" un SO**; contiene tu aplicación y las librerías que necesita, ejecutándose sobre el kernel compartido. Por eso no puedes correr un contenedor Linux nativamente sobre un kernel Windows (Docker Desktop usa una VM Linux ligera por debajo).

### Namespaces — El Aislamiento

Un **namespace** de Linux hace que un proceso vea una versión aislada de algún recurso del sistema, como si fuera el único. Es lo que crea la ilusión de "estar solo en una máquina":

- **PID namespace**: el proceso ve su propio árbol de procesos; dentro del contenedor, tu app es el PID 1 y no ve los procesos del host.
- **Network namespace**: su propia pila de red (interfaces, IPs, tablas de rutas).
- **Mount namespace**: su propia vista del sistema de ficheros.
- **UTS namespace**: su propio hostname.
- **User namespace**: mapea usuarios; puedes ser root *dentro* del contenedor sin serlo en el host (clave para seguridad).
- **IPC namespace**: aísla la comunicación entre procesos.

Un contenedor es, literalmente, **un proceso lanzado dentro de un conjunto de namespaces**. No hay magia más allá de eso.

### cgroups — La Limitación de Recursos

Los namespaces aíslan *lo que el proceso ve*; los **cgroups** (control groups) limitan *cuántos recursos puede consumir*: CPU, memoria, I/O, número de procesos. Sin cgroups, un contenedor podría acaparar toda la RAM del host y tumbar a los demás.

```bash
# Limitar un contenedor a 512 MB de RAM y media CPU
docker run --memory=512m --cpus=0.5 mi-app
# Si el proceso supera los 512 MB, el kernel lo mata (OOMKill).
```

Esto es exactamente lo que hay detrás de los `requests` y `limits` de Kubernetes ([[Desarrollo Profesional/Kubernetes/Páginas/04 - Configuración y Operación|Config y Operación]]): se traducen en cgroups. Entender esto explica por qué un contenedor "se reinicia solo" (lo mató el OOMKiller por superar su límite de memoria).

### Imágenes por Capas

Una **imagen** es una plantilla de solo lectura desde la que se crean contenedores. Su característica definitoria: está construida en **capas apiladas e inmutables**. Cada instrucción del Dockerfile (`RUN`, `COPY`, `ADD`) crea una capa nueva que registra los cambios respecto a la anterior:

```
┌────────────────────────────┐
│ CMD ["python","app.py"]     │  ← metadatos (capa 0 bytes)
├────────────────────────────┤
│ COPY . /app                 │  ← capa: tu código
├────────────────────────────┤
│ RUN pip install -r req.txt  │  ← capa: dependencias
├────────────────────────────┤
│ COPY requirements.txt .     │  ← capa: el fichero de deps
├────────────────────────────┤
│ FROM python:3.12-slim       │  ← capa base
└────────────────────────────┘
```

Dos propiedades que esto habilita:
- **Caché de capas**: si una capa no cambia, Docker la reutiliza del build anterior. Por eso el **orden importa** (lo verás en la [página 02](02%20-%20Dockerfiles%20y%20Buenas%20Prácticas)): pon lo que cambia poco (dependencias) antes de lo que cambia mucho (tu código).
- **Compartición**: varias imágenes que parten de `python:3.12-slim` comparten esa capa en disco y en la red. Solo se descarga/almacena una vez.

Cuando ejecutas un contenedor, Docker añade una **capa de escritura** fina encima de las capas de solo lectura (copy-on-write). **Todo lo que escribe el contenedor vive ahí y se pierde al borrarlo** — por eso los contenedores son efímeros y los datos persistentes van en *volúmenes* montados desde el host. Esto es la base de la filosofía de contenedores inmutables y *stateless*.

### Registries, Tags y Ciclo de Vida

Las imágenes se distribuyen a través de **registries** (Docker Hub, GCP Artifact Registry, GHCR). Una imagen se identifica por `registry/repositorio:tag`:

```bash
docker build -t europe-docker.pkg.dev/proyecto/api:1.4.2 .   # construir
docker push europe-docker.pkg.dev/proyecto/api:1.4.2          # subir al registry
docker pull europe-docker.pkg.dev/proyecto/api:1.4.2          # descargar
docker run europe-docker.pkg.dev/proyecto/api:1.4.2           # ejecutar
```

> [!warning] `latest` no es una versión
> El tag `latest` es solo un nombre por defecto, no significa "la última versión": apunta a lo que alguien etiquetó como `latest` por última vez, y puede cambiar bajo tus pies. **Etiqueta siempre por versión inmutable** (el SHA del commit, un semver). Para garantías criptográficas, referencia por **digest** (`@sha256:...`), que identifica el contenido exacto y no puede mutar.

El ciclo de vida de un contenedor está atado a su **proceso principal** (PID 1): el contenedor vive mientras ese proceso viva. Cuando termina (o crashea), el contenedor se para. No es un servidor que "está encendido": es un proceso que se ejecuta.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - Un contenedor **no es una VM ligera**: es un **proceso de Linux normal** que comparte el kernel del host, aislado por namespaces y limitado por cgroups. Las VMs virtualizan hardware (kernel propio); los contenedores comparten kernel.
> - **Namespaces** aíslan lo que el proceso ve (PID, red, ficheros, hostname, usuarios). **cgroups** limitan lo que consume (CPU, memoria) — son los `requests`/`limits` de Kubernetes por debajo.
> - Una **imagen** son capas inmutables apiladas; cada instrucción del Dockerfile crea una. Esto habilita **caché** (reutilizar capas que no cambian) y **compartición** (la capa base se guarda una vez).
> - El contenedor añade una capa de escritura efímera: **lo que escribe se pierde al borrarlo**. Los datos persistentes van en volúmenes. Contenedores = efímeros y stateless.
> - Identifica imágenes por versión inmutable (SHA/semver) o **digest**, nunca confíes en `latest`. El contenedor vive mientras viva su proceso PID 1.

### Para llevar a la práctica
- [ ] Ejecuta `docker run --memory=100m` con una app que consuma más y observa el OOMKill — verás los cgroups en acción.
- [ ] Mira las capas de una imagen tuya con `docker history <imagen>`. ¿Cuáles son las más pesadas?
- [ ] Revisa si en algún sitio dependes de `latest`. Cámbialo a un tag inmutable.
- [ ] Comprueba que tus contenedores no guardan datos importantes en su filesystem (se perderían). ¿Usan volúmenes?

### Recursos
- 🌐 docs.docker.com/get-started/docker-overview — fundamentos oficiales
- 🌐 docs.docker.com/build/cache — cómo funciona la caché de capas
- 📄 "What even is a container?" — Julia Evans (jvns.ca), explicación visual de namespaces/cgroups
- 📖 *Docker Deep Dive* — Nigel Poulton

---
`#docker` `#contenedores` `#namespaces` `#cgroups` `#imagenes`
