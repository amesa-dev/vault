# 🚀 CI/CD 01 — Pipelines CI/CD

[[Desarrollo Profesional/CI-CD e IaC/CI-CD e IaC|⬅️ Volver a CI/CD e IaC]] | [[Desarrollo Profesional/CI-CD e IaC/Páginas/02 - IaC y GitOps|02 →]]

> [!abstract] Introducción
> CI/CD son tres siglas que se mezclan: Integración Continua, Entrega Continua y Despliegue Continuo. No son lo mismo, y la diferencia importa. Esta página separa los conceptos, muestra cómo se materializa un pipeline en GitHub Actions, y cubre las estrategias de despliegue (blue-green, canary, rolling) que permiten poner código en producción muchas veces al día con riesgo controlado.

## ¿De qué vamos a hablar?

El flujo automatizado desde un commit hasta producción, y cómo desplegar sin que cada release sea un evento de riesgo.

### Conceptos que vamos a cubrir
- CI vs Continuous Delivery vs Continuous Deployment
- Anatomía de un pipeline: stages y gates
- GitHub Actions: workflows, jobs, steps
- Estrategias de despliegue: rolling, blue-green, canary
- Métricas DORA: cómo se mide un buen pipeline

---

## El Concepto

### CI vs CD vs CD

Las tres etapas, cada una construye sobre la anterior:

- **Continuous Integration (CI)**: cada cambio se integra a la rama principal con frecuencia (al menos diaria), y cada integración dispara un build automático + tests. El objetivo es **detectar conflictos y regresiones pronto**, cuando son baratos de arreglar. La práctica clave: ramas de vida corta y merges frecuentes, no ramas que viven semanas.
- **Continuous Delivery**: además de testear, el pipeline **empaqueta un artefacto desplegable** y lo deja listo para producción. El despliegue final es un botón que alguien pulsa cuando decide. *Siempre estás a un clic de desplegar.*
- **Continuous Deployment**: el paso final. Si todos los tests y gates pasan, el código va **automáticamente a producción** sin intervención humana. Requiere mucha confianza en la suite de tests y en los mecanismos de rollback.

La progresión natural de madurez es CI → Delivery → Deployment. No saltes al despliegue continuo sin tener primero tests sólidos ([[Desarrollo Profesional/Estrategia de Testing/Estrategia de Testing|Estrategia de Testing]]) y observabilidad ([[Desarrollo Profesional/Prometheus/Prometheus|Prometheus]]/[[Desarrollo Profesional/Grafana/Grafana|Grafana]]) para detectar y revertir problemas rápido.

### Anatomía de un Pipeline

Un pipeline es una secuencia de **stages**, cada uno con uno o más jobs, y **gates** (condiciones que deben cumplirse para avanzar):

```
[Commit/PR]
   │
   ▼
[Lint + formato] ──→ [Tests unitarios] ──→ [Build artefacto/imagen]
   │                                              │
   ▼                                              ▼
[Análisis de seguridad]                    [Tests de integración]
(SAST, escaneo deps)                              │
   │                                              ▼
   └──────────── gate: ¿todo verde? ──────→ [Push a registry]
                                                  │
                                                  ▼
                                          [Deploy a staging]
                                                  │
                                          [Tests e2e / smoke]
                                                  │
                                          gate: ¿aprobación? (Delivery)
                                                  │
                                                  ▼
                                          [Deploy a producción]
```

Principios: **falla rápido** (pon lo barato y lo que más falla primero — lint y unit tests antes que los e2e lentos), **paraleliza** lo independiente, y haz cada stage **reproducible** (mismo input → mismo output, sin depender del estado de la máquina).

### GitHub Actions

El modelo de GitHub Actions: un **workflow** (fichero YAML en `.github/workflows/`) se dispara por un evento (`push`, `pull_request`, schedule), contiene **jobs** (que corren en paralelo por defecto, en máquinas aisladas), y cada job tiene **steps** (comandos o acciones reutilizables).

```yaml
# .github/workflows/ci.yml
name: CI
on:
  pull_request:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install -e ".[dev]"
      - run: ruff check .          # lint rápido primero
      - run: pytest --cov          # tests con cobertura

  build-and-push:
    needs: test                    # gate: solo si 'test' pasó
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write              # para auth sin secretos (OIDC) con la nube
    steps:
      - uses: actions/checkout@v4
      - name: Build & push imagen
        run: |
          docker build -t europe-docker.pkg.dev/$PROYECTO/app:${{ github.sha }} .
          docker push europe-docker.pkg.dev/$PROYECTO/app:${{ github.sha }}
```

Buenas prácticas en Actions:
- **No pongas secretos en el YAML**: usa `secrets` del repo, y mejor aún **OIDC** (`id-token: write`) para autenticarte con GCP/AWS sin guardar credenciales de larga vida.
- **Fija versiones de las actions** (idealmente por hash, no por tag móvil) — son código de terceros que corre con tus permisos (cadena de suministro, ver [[Desarrollo Profesional/Seguridad Aplicada/Páginas/03 - Criptografía y Secretos|Seguridad 03]]).
- **Etiqueta las imágenes por SHA del commit**, no solo `latest`, para que cada despliegue sea trazable y reversible.
- **Mínimos permisos**: declara `permissions` explícitos por job.

### Estrategias de Despliegue

Cómo sustituir la versión vieja por la nueva sin tirar el servicio:

- **Rolling update** (el defecto en Kubernetes): reemplaza las réplicas poco a poco (unas nuevas suben, unas viejas bajan). Sin downtime, sin coste extra de infra, pero conviven ambas versiones un rato (cuidado con cambios de esquema incompatibles).
- **Blue-Green**: levantas el entorno nuevo (green) completo junto al viejo (blue), lo pruebas, y cambias todo el tráfico de golpe. Rollback instantáneo (vuelves a blue). Coste: el doble de infra durante la transición.
- **Canary**: mandas un % pequeño del tráfico (5%) a la versión nueva, observas métricas de error/latencia, y si va bien aumentas gradualmente hasta el 100%. Limita el radio de impacto de un fallo. Es la estrategia más segura para servicios críticos, y se automatiza con métricas (canary analysis).

```
Rolling:   [v1 v1 v1 v1] → [v2 v1 v1 v1] → [v2 v2 v1 v1] → ... → [v2 v2 v2 v2]
Blue-Green: blue(v1)=100% tráfico   green(v2) listo →  switch →  green(v2)=100%
Canary:    v2 recibe 5% → 25% → 50% → 100%   (abortando si las métricas empeoran)
```

> [!tip] El despliegue seguro empieza en el diseño
> Para desplegar sin downtime, los cambios deben ser **backward-compatible**: migraciones de BD en dos fases (añadir columna → desplegar código que la use → más tarde quitar la vieja), nunca un cambio que rompa la versión anterior mientras conviven. Esto se llama *expand-contract* o migraciones expand/contract.

### Métricas DORA

El estudio *DevOps Research and Assessment* identificó cuatro métricas que correlacionan con equipos de alto rendimiento:

1. **Deployment Frequency**: cada cuánto despliegas (los mejores: a demanda, varias veces al día).
2. **Lead Time for Changes**: del commit a producción (los mejores: menos de un día).
3. **Change Failure Rate**: % de despliegues que causan un fallo (los mejores: 0-15%).
4. **Time to Restore**: cuánto tardas en recuperarte de un incidente (los mejores: menos de una hora).

La conclusión contraintuitiva del estudio: **velocidad y estabilidad no se oponen**. Los equipos que despliegan más a menudo también fallan menos y se recuperan antes, porque sus cambios son pequeños, automatizados y reversibles.

---

## 🔑 Resumen

> [!summary] Puntos Clave
> - **CI** (integrar y testear seguido), **Continuous Delivery** (dejar un artefacto listo para desplegar a un clic) y **Continuous Deployment** (desplegar automáticamente si pasa todo) son etapas de madurez, no sinónimos.
> - Un pipeline es stages + gates. Pon lo barato y lo que más falla primero (lint, unit), paraleliza, y haz cada stage reproducible.
> - **GitHub Actions**: workflows → jobs → steps. Usa OIDC en vez de secretos para la nube, fija versiones de las actions, etiqueta imágenes por SHA, mínimos permisos.
> - Estrategias: **rolling** (gradual, sin coste extra), **blue-green** (switch instantáneo, doble infra), **canary** (% creciente con observación, el más seguro). Requieren cambios backward-compatible (expand-contract).
> - **DORA**: frecuencia de despliegue, lead time, change failure rate y time to restore. Velocidad y estabilidad van juntas cuando los cambios son pequeños y reversibles.

### Para llevar a la práctica
- [ ] Revisa tu pipeline: ¿los tests rápidos corren antes que los lentos? ¿hay pasos secuenciales que podrían paralelizarse?
- [ ] Comprueba cómo te autenticas con la nube desde CI. Si usas claves de servicio guardadas como secreto, migra a OIDC.
- [ ] Identifica qué estrategia de despliegue usas hoy y si tus migraciones de BD son backward-compatible.
- [ ] Mide tus cuatro métricas DORA, aunque sea a ojo. ¿Cuál es tu mayor cuello de botella?

### Recursos
- 📖 *Accelerate* — Forsgren, Humble, Kim (la base científica de DORA)
- 📖 *Continuous Delivery* — Jez Humble, David Farley
- 🌐 docs.github.com/actions — documentación de GitHub Actions
- 🌐 dora.dev — las métricas DORA y el reporte anual State of DevOps

---
`#cicd` `#github-actions` `#despliegue` `#dora`
