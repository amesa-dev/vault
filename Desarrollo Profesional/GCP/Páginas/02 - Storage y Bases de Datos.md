# ☁️ GCP 02 — Storage y Bases de Datos

[[Desarrollo Profesional/GCP/GCP|⬅️ Volver a GCP]] | [[Desarrollo Profesional/GCP/Páginas/01 - Compute|← 01]] | [[Desarrollo Profesional/GCP/Páginas/03 - Networking e IAM|03 →]]

> [!abstract] Introducción
> GCP ofrece un espectro completo de soluciones de almacenamiento: desde objetos no estructurados (GCS) hasta bases de datos relacionales gestionadas (Cloud SQL), bases de datos de escala planetaria (Spanner) y almacenes de datos en memoria (Memorystore/Redis). La decisión de cuál usar determina la consistencia, la latencia y el coste de tu sistema.

## ¿De qué vamos a hablar?

Las opciones de almacenamiento de GCP — cuándo usar cada una y cómo interactuar con ellas desde Python.

### Conceptos que vamos a cubrir
- Cloud Storage (GCS): almacenamiento de objetos
- Cloud SQL: PostgreSQL/MySQL gestionado
- Cloud Spanner: escala horizontal relacional
- Firestore: NoSQL de documentos
- Memorystore: Redis gestionado

---

## El Concepto

### Cloud Storage (GCS) — Almacenamiento de Objetos

El almacén de objetos de GCP. Para archivos, imágenes, backups, artefactos de CI, datos de ML, logs archivados. No es un sistema de ficheros — no hay jerarquía real, solo buckets y objetos con rutas como nombres.

```python
from google.cloud import storage

client = storage.Client()

# Subir un archivo
bucket = client.bucket("mi-bucket")
blob = bucket.blob("imagenes/foto.jpg")
blob.upload_from_filename("foto.jpg")

# Subir desde memoria
import io
datos = b"contenido del archivo"
blob.upload_from_file(io.BytesIO(datos), content_type="application/octet-stream")

# Descargar
blob.download_to_filename("foto_descargada.jpg")
contenido = blob.download_as_bytes()

# URL firmada — acceso temporal sin autenticación
url_firmada = blob.generate_signed_url(
    version="v4",
    expiration=3600,  # 1 hora
    method="GET"
)

# Listar objetos
for blob in bucket.list_blobs(prefix="imagenes/"):
    print(f"{blob.name} — {blob.size} bytes")
```

```bash
# gsutil — CLI de GCS
gsutil cp archivo.txt gs://mi-bucket/
gsutil -m cp -r directorio/ gs://mi-bucket/directorio/  # copia paralela
gsutil ls gs://mi-bucket/**
gsutil du -sh gs://mi-bucket/  # tamaño total

# Clases de almacenamiento (coste vs acceso):
# Standard   — acceso frecuente, coste alto/GB, sin mínimo de tiempo
# Nearline   — acceso mensual o menos, coste bajo/GB, mínimo 30 días
# Coldline   — acceso anual o menos, muy bajo/GB, mínimo 90 días
# Archive    — acceso rarísimo, mínimo coste/GB, mínimo 365 días
```

### Cloud SQL — PostgreSQL/MySQL Gestionado

La opción más simple para bases de datos relacionales. Google gestiona backups, actualizaciones, failover y réplicas de lectura:

```bash
# Crear instancia de PostgreSQL
gcloud sql instances create mi-postgres \
  --database-version=POSTGRES_15 \
  --tier=db-n1-standard-2 \
  --region=europe-west1 \
  --availability-type=REGIONAL \  # HA con failover automático
  --backup-start-time=02:00 \
  --enable-point-in-time-recovery

# Crear base de datos y usuario
gcloud sql databases create mi_base --instance=mi-postgres
gcloud sql users create mi_usuario --instance=mi-postgres --password=secreto

# Conexión desde local (Cloud SQL Proxy)
cloud-sql-proxy mi-proyecto:europe-west1:mi-postgres --port=5432
# Luego: psql -h 127.0.0.1 -p 5432 -U mi_usuario mi_base

# Conexión desde Cloud Run/GKE — usar el conector nativo
```

```python
# Conectar desde una aplicación Python
import sqlalchemy
from google.cloud.sql.connector import Connector

def create_engine_with_connector():
    connector = Connector()

    def get_conn():
        return connector.connect(
            "mi-proyecto:europe-west1:mi-postgres",
            "pg8000",  # o asyncpg para async
            user="mi_usuario",
            password="secreto",
            db="mi_base"
        )

    engine = sqlalchemy.create_engine(
        "postgresql+pg8000://",
        creator=get_conn,
    )
    return engine
```

### Firestore — NoSQL de Documentos

Base de datos NoSQL orientada a documentos con sincronización en tiempo real. Ideal para datos no estructurados, configuraciones, perfiles de usuario en apps móviles/web:

```python
from google.cloud import firestore

db = firestore.Client()

# Escribir documento
doc_ref = db.collection("usuarios").document("usuario-123")
doc_ref.set({
    "nombre": "Ana García",
    "email": "ana@ejemplo.com",
    "activo": True,
    "creado_en": firestore.SERVER_TIMESTAMP
})

# Leer
doc = doc_ref.get()
if doc.exists:
    print(doc.to_dict())

# Actualizar campos específicos
doc_ref.update({"activo": False})

# Consultas
usuarios_activos = db.collection("usuarios").where("activo", "==", True).stream()
for doc in usuarios_activos:
    print(doc.id, doc.to_dict())

# Transacción
@firestore.transactional
def transferir(transaction, origen_ref, destino_ref, cantidad):
    origen = origen_ref.get(transaction=transaction)
    if origen.get("saldo") < cantidad:
        raise ValueError("Saldo insuficiente")
    transaction.update(origen_ref, {"saldo": firestore.Increment(-cantidad)})
    transaction.update(destino_ref, {"saldo": firestore.Increment(cantidad)})
```

### Memorystore — Redis Gestionado

Redis gestionado en GCP. Para caché, sesiones, contadores, colas de trabajo:

```bash
# Crear instancia de Redis
gcloud redis instances create mi-redis \
  --size=1 \
  --region=europe-west1 \
  --redis-version=redis_7_0
```

```python
import redis

r = redis.Redis(host="10.0.0.3", port=6379)  # IP interna de Memorystore

# Caché
r.setex("usuario:123", 3600, '{"nombre": "Ana"}')  # TTL de 1 hora
datos = r.get("usuario:123")

# Contador
r.incr("visitas:pagina:home")
visitas = int(r.get("visitas:pagina:home") or 0)

# Lista como cola
r.rpush("cola:emails", email_serializado)
email = r.lpop("cola:emails")
```

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - **GCS**: almacenamiento de objetos ilimitado. Usa clases de almacenamiento según la frecuencia de acceso (Standard → Archive).
> - **Cloud SQL**: PostgreSQL/MySQL gestionado. Para la mayoría de aplicaciones, es la opción correcta. Usa el conector nativo de Google para autenticación segura.
> - **Firestore**: NoSQL de documentos. Bien para datos semi-estructurados, apps móviles, configuraciones. No para relaciones complejas ni JOIN-heavy queries.
> - **Memorystore**: Redis gestionado. Para caché, sesiones y operaciones de alta velocidad que no necesitan persistencia duradera.

### Recursos
- 🌐 cloud.google.com/storage/docs
- 🌐 cloud.google.com/sql/docs/postgres
- 🌐 firebase.google.com/docs/firestore

---
`#gcp` `#storage` `#cloud-sql` `#firestore` `#gcs`
