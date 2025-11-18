# Estrategia de Branching - Proyecto Final IngeSoft V

## Resumen

Este proyecto utiliza una **estrategia de branching simplificada** basada en GitFlow, adaptada para microservicios con CI/CD automatizado. La estrategia permite separar claramente los ambientes de desarrollo, staging y producción, mientras mantiene un flujo de trabajo ágil y automatizado.

---

## Estructura de Ramas

### Ramas Principales

```
main/master (Producción)
    ↑
stage (Staging)
    ↑
dev/develop (Desarrollo)
```

### Descripción de Ramas

#### 1. **`dev` / `develop`** - Desarrollo
- **Propósito**: Desarrollo activo y pruebas iniciales
- **Tag de Imagen Docker**: `dev-latest`
- **Namespace Kubernetes**: `ecommerce-dev`
- **Pipeline**: Build → Test → Deploy
- **Pruebas Ejecutadas**:
  - Pruebas unitarias
  - Pruebas de integración
  - Pruebas E2E (no se ejecutan)
  - Pruebas de rendimiento (no se ejecutan)
  - Release Notes (no se generan)

**Uso**: Para desarrollo diario, pruebas rápidas y validación de cambios antes de promocionar a staging.

---

#### 2. **`stage`** - Staging
- **Propósito**: Ambiente de pre-producción para validación completa
- **Tag de Imagen Docker**: `stage-latest`
- **Namespace Kubernetes**: `ecommerce-dev`
- **Pipeline**: Build → Test → Deploy → E2E Tests → Performance Tests
- **Pruebas Ejecutadas**:
  - Pruebas unitarias
  - Pruebas de integración
  - Pruebas E2E (Postman/Newman)
  - Pruebas de rendimiento (Locust)
  - Release Notes (no se generan)

**Uso**: Para validar que todos los cambios funcionan correctamente en un ambiente similar a producción antes de promocionar a producción.

---

#### 3. **`main` / `master`** - Producción
- **Propósito**: Ambiente de producción estable
- **Tag de Imagen Docker**: `prod-0.1.0` (versión semántica)
- **Namespace Kubernetes**: `ecommerce-dev`
- **Pipeline**: Build → Test → Deploy → E2E Tests → Performance Tests → Release Notes
- **Pruebas Ejecutadas**:
  - Pruebas unitarias
  - Pruebas de integración
  - Pruebas E2E (Postman/Newman)
  - Pruebas de rendimiento (Locust)
  - Release Notes automáticos (GitHub Releases)

**Uso**: Solo para código que ha sido completamente validado en staging y está listo para producción.

---

## Flujo de Trabajo

### Flujo Normal de Desarrollo

```
1. Crear feature branch desde dev/develop
   git checkout -b feature/nueva-funcionalidad develop

2. Desarrollar y hacer commits
   git commit -m "feat: agregar nueva funcionalidad"

3. Push a feature branch
   git push origin feature/nueva-funcionalidad

4. Crear Pull Request a dev/develop
   → Revisión de código
   → Merge a dev/develop

5. Push a dev/develop dispara pipeline
   → Build, test, deploy a Kubernetes

6. Cuando está listo, merge dev → stage
   → Pipeline ejecuta todas las pruebas

7. Cuando está validado, merge stage → main
   → Pipeline completo + Release Notes
```

### Promoción entre Ambientes

```
┌─────────────────────────────────────────────────────────┐
│                    Flujo de Promoción                    │
└─────────────────────────────────────────────────────────┘

dev/develop ──merge──> stage ──merge──> main/master
   │                      │                  │
   │                      │                  │
   ▼                      ▼                  ▼
dev-latest          stage-latest        prod-0.1.0
   │                      │                  │
   │                      │                  │
   ▼                      ▼                  ▼
Build + Test      + E2E + Perf        + Release Notes
```

---

## Sistema de Etiquetado (Tags)

### Tags de Imágenes Docker

| Rama | Tag | Descripción |
|------|-----|-------------|
| `dev` / `develop` | `dev-latest` | Tag estático, siempre apunta a la última versión de desarrollo |
| `stage` | `stage-latest` | Tag estático, siempre apunta a la última versión de staging |
| `main` / `master` | `prod-0.1.0` | Tag de versión semántica (actualmente fijo, futuro: automático) |

### Tags Adicionales

- **SHA del commit**: Cada imagen también se etiqueta con el SHA corto del commit
- **Versión fija**: `0.1.0` (actualmente, futuro: versionado semántico automático)

---

## Convenciones de Commits

### Formato: Conventional Commits

Todos los commits deben seguir el formato **Conventional Commits** para que los Release Notes se generen automáticamente:

```
<tipo>: <descripción>

[opcional: cuerpo del commit]

[opcional: footer con breaking changes]
```

### Tipos de Commits Válidos

| Tipo | Descripción | Ejemplo |
|------|-------------|---------|
| `feat:` | Nueva funcionalidad | `feat: agregar endpoint de búsqueda de productos` |
| `fix:` | Corrección de bug | `fix: corregir error en cálculo de total de orden` |
| `chore:` | Tareas de mantenimiento | `chore: actualizar dependencias de seguridad` |
| `docs:` | Documentación | `docs: actualizar README con nuevas instrucciones` |
| `refactor:` | Refactorización | `refactor: mejorar estructura de clases de servicio` |
| `test:` | Pruebas | `test: agregar pruebas unitarias para PaymentService` |
| `style:` | Formato de código | `style: corregir indentación en archivos Java` |
| `perf:` | Mejoras de rendimiento | `perf: optimizar consulta de base de datos` |
| `ci:` | CI/CD | `ci: agregar step de SonarQube al pipeline` |
| `build:` | Build system | `build: actualizar versión de Maven` |

### Breaking Changes

Para indicar un cambio incompatible (incrementa MAJOR en versionado semántico):

```
feat!: cambiar estructura de respuesta de API

BREAKING CHANGE: El campo 'price' ahora se llama 'priceUnit'
```

---

## Pipelines por Rama

### Pipeline: `dev` / `develop`

```yaml
Jobs ejecutados:
1. build-and-test
   - Compilación
   - Pruebas unitarias
   - Pruebas de integración
   - Reportes de pruebas

2. build-docker
   - Construcción de imagen
   - Tag: dev-latest
   - Push a Docker Hub

3. deploy
   - Despliegue a Kubernetes
   - Namespace: ecommerce-dev
   - Rollout restart (para tags estáticos)
```

**Duración estimada**: ~5-10 minutos

---

### Pipeline: `stage`

```yaml
Jobs ejecutados:
1. build-and-test (igual que dev)
2. build-docker (tag: stage-latest)
3. deploy
4. e2e-tests
   - Pruebas E2E con Postman/Newman
   - Validación de flujos completos
   - Reportes HTML/JSON

5. performance-tests
   - Pruebas de rendimiento con Locust
   - 50 usuarios, 2 minutos
   - Reportes HTML/CSV
```

**Duración estimada**: ~15-20 minutos

---

### Pipeline: `main` / `master`

```yaml
Jobs ejecutados:
1. build-and-test
2. build-docker (tag: prod-0.1.0)
3. deploy
4. e2e-tests
5. performance-tests
6. release-notes
   - Análisis de commits
   - Generación de Release Notes
   - Creación de GitHub Release
```

**Duración estimada**: ~20-25 minutos

---

## Protección de Ramas

### Recomendaciones (Configurar en GitHub)

#### Rama `main` / `master`
- Requerir Pull Request antes de merge
- Requerir revisión de código (al menos 1 aprobación)
- Requerir que los checks pasen antes de merge
- No permitir force push
- No permitir eliminación de la rama

#### Rama `stage`
- Requerir Pull Request antes de merge
- Requerir que los checks pasen antes de merge
- Revisión de código opcional (más ágil)

#### Rama `dev` / `develop`
- Permitir push directo (desarrollo ágil)
- Requerir que los checks pasen antes de merge a stage

---

## Diferencias Clave con GitFlow Clásico

### GitFlow Clásico
- `main` (producción)
- `develop` (desarrollo)
- `feature/*` (features)
- `release/*` (preparación de releases)
- `hotfix/*` (correcciones urgentes)

### Nuestra Estrategia Simplificada
- `main/master` (producción)
- `stage` (staging/pre-producción)
- `dev/develop` (desarrollo)
- `feature/*` (opcional, para features grandes)

**Ventajas de nuestra estrategia**:
- Más simple y ágil
- Menos ramas que gestionar
- Integración continua más frecuente
- Mejor para equipos pequeños
- CI/CD automatizado en cada rama

---

## Mejores Prácticas

### 1. Desarrollo de Features

```bash
# Crear feature branch desde develop
git checkout develop
git pull origin develop
git checkout -b feature/nombre-feature

# Desarrollar y hacer commits
git commit -m "feat: agregar funcionalidad X"

# Push y crear PR
git push origin feature/nombre-feature
# Crear Pull Request a develop en GitHub
```

### 2. Promoción a Staging

```bash
# Desde develop, crear PR a stage
# En GitHub: Pull Request develop → stage
# Esperar que todos los checks pasen
# Merge cuando esté listo
```

### 3. Promoción a Producción

```bash
# Desde stage, crear PR a main
# En GitHub: Pull Request stage → main
# Esperar que todos los checks pasen
# Revisar Release Notes generados
# Merge cuando esté validado
```

### 4. Hotfixes (Correcciones Urgentes)

```bash
# Opción 1: Desde main directamente (solo para emergencias)
git checkout main
git checkout -b hotfix/correccion-urgente
# Hacer cambios
git commit -m "fix: corregir error crítico"
# PR a main → stage → develop (merge hacia atrás)

# Opción 2: Desde stage (recomendado)
git checkout stage
git checkout -b hotfix/correccion-urgente
# Hacer cambios
git commit -m "fix: corregir error crítico"
# PR a stage → main → develop
```

---

## Métricas y Monitoreo

### Métricas por Rama

- **Frecuencia de commits**: Monitorear actividad en cada rama
- **Tiempo de pipeline**: Tiempo promedio de ejecución por rama
- **Tasa de éxito**: Porcentaje de pipelines exitosos
- **Tiempo de promoción**: Tiempo promedio desde dev → stage → main

---

## Estrategia de Namespace

**Importante**: Todos los ambientes (dev, stage, prod) despliegan al mismo namespace `ecommerce-dev` en Kubernetes.

**Razón**: Optimización de recursos para cuenta de estudiante de Azure.

**Diferenciación**: Se logra mediante:
- Tags diferentes de imágenes Docker
- Configuraciones diferentes por ambiente (futuro: ConfigMaps por ambiente)
- Promoción controlada mediante ramas

---

## Referencias

- [Conventional Commits](https://www.conventionalcommits.org/)
- [GitFlow Workflow](https://www.atlassian.com/git/tutorials/comparing-workflows/gitflow-workflow)
- [GitHub Flow](https://guides.github.com/introduction/flow/)
- [Semantic Versioning](https://semver.org/)

---

## Notas de Implementación

### Estado Actual
- Estrategia implementada y funcionando
- Pipelines configurados para todas las ramas
- Release Notes automáticos en main/master
- Versionado semántico automático: Pendiente (actualmente usa versión fija)

### Mejoras Futuras
- [ ] Implementar versionado semántico automático basado en commits
- [ ] Configurar protección de ramas en GitHub
- [ ] Agregar aprobaciones manuales para producción
- [ ] Implementar feature flags por ambiente
- [ ] Configurar notificaciones de fallos en pipelines

---

**Última actualización**: Enero 2025  
**Versión del documento**: 1.0

