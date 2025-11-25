# Proceso de Change Management

## 1. Introducción

Este documento define el proceso formal de gestión de cambios para los microservicios del ecommerce. El objetivo es asegurar que todos los cambios sean rastreados, documentados y desplegados de manera controlada.

## 2. Tipos de Cambios

### 2.1 Cambios por Tipo de Commit

Utilizamos **Conventional Commits** para categorizar los cambios:

| Tipo | Descripción | Impacto en Versión |
|------|-------------|-------------------|
| `feat:` o `feature:` | Nueva funcionalidad | Incrementa MINOR (1.2.3 → 1.3.0) |
| `fix:` o `bugfix:` | Corrección de bug | Incrementa PATCH (1.2.3 → 1.2.4) |
| `BREAKING CHANGE:` | Cambio incompatible | Incrementa MAJOR (1.2.3 → 2.0.0) |
| `chore:` | Tareas de mantenimiento | No cambia versión |
| `docs:` | Documentación | No cambia versión |
| `refactor:` | Refactorización | No cambia versión |
| `test:` | Pruebas | No cambia versión |
| `style:` | Formato de código | No cambia versión |
| `perf:` | Mejoras de rendimiento | Incrementa PATCH |
| `ci:` | Cambios en CI/CD | No cambia versión |
| `build:` | Cambios en build | No cambia versión |

### 2.2 Ejemplos de Commits

```bash
# Nueva funcionalidad
feat: agregar endpoint de búsqueda de productos

# Corrección de bug
fix: corregir validación de email en registro

# Cambio incompatible (requiere BREAKING CHANGE: en el cuerpo del commit)
feat: cambiar estructura de respuesta de API
BREAKING CHANGE: El campo 'price' ahora es 'amount' y es un objeto

# Tarea de mantenimiento
chore: actualizar dependencias de Maven

# Documentación
docs: actualizar README con instrucciones de despliegue
```

## 3. Flujo de Cambios

### 3.1 Desarrollo (Dev)

1. **Crear branch desde `dev` o `develop`**
   ```bash
   git checkout dev
   git pull origin dev
   git checkout -b feat/nueva-funcionalidad
   ```

2. **Hacer cambios y commits con prefijos convencionales**
   ```bash
   git commit -m "feat: agregar endpoint de búsqueda"
   ```

3. **Push y crear Pull Request**
   - El PR debe tener descripción clara
   - Los tests deben pasar
   - Code review requerido

4. **Merge a `dev`**
   - Despliegue automático a ambiente de desarrollo
   - Tests E2E y de performance se ejecutan

### 3.2 Stage

1. **Merge desde `dev` a `stage`**
   ```bash
   git checkout stage
   git merge dev
   git push origin stage
   ```

2. **Despliegue automático**
   - Pipeline ejecuta build, tests, deploy
   - Tests E2E y Locust se ejecutan
   - Validación manual opcional

### 3.3 Producción (Main)

1. **Merge desde `stage` a `main`**
   ```bash
   git checkout main
   git merge stage
   git push origin main
   ```

2. **Despliegue automático con versionado semántico**
   - El sistema calcula automáticamente la nueva versión basado en commits
   - Se genera release notes automáticamente
   - Se crea GitHub Release con la nueva versión
   - Despliegue a producción

## 4. Versionado Semántico Automático

### 4.1 Implementación

El versionado semántico se implementa mediante un **script personalizado en GitHub Actions** (no se usa la herramienta `semantic-release` de npm). El script se ejecuta en el job `calculate-version` del pipeline de producción.

**Ubicación**: `.github/workflows/ci-cd-prod.yml` → Job `calculate-version`

### 4.2 Cómo Funciona

El script personalizado:

1. **Obtiene el último tag** de Git (formato: `prod-X.Y.Z` o `vX.Y.Z`)
2. **Analiza los commits** desde el último tag hasta HEAD
3. **Determina el tipo de cambio** basándose en los prefijos de commits:
   - **MAJOR (2.0.0)**: Si hay `BREAKING CHANGE:` en algún commit
   - **MINOR (1.3.0)**: Si hay commits `feat:` o `feature:`
   - **PATCH (1.2.4)**: Si hay commits `fix:` o `bugfix:`
4. **Calcula la nueva versión** incrementando el número correspondiente
5. **Genera el IMAGE_TAG** con formato `prod-X.Y.Z`
6. **Pasa la versión** a los siguientes jobs del pipeline

### 4.3 Ejemplo

**Versión actual:** `1.2.3`

**Commits nuevos:**
```
feat: agregar filtro de búsqueda
fix: corregir bug en validación
docs: actualizar README
```

**Nueva versión:** `1.3.0` (incrementa MINOR por el `feat:`)

## 5. Release Notes Automáticos

Los release notes se generan automáticamente incluyendo:

- **Cambios**: Lista de commits con prefijos convencionales
- **Información de despliegue**: Tag de imagen, namespace, servicio, fecha
- **Métricas opcionales**: Cobertura de código, resultados de tests

## 6. Aprobaciones y Revisión

### 6.1 Code Review

- Todos los PRs requieren al menos 1 aprobación
- Los revisores deben verificar:
  - Prefijos de commit correctos
  - Tests pasando
  - Documentación actualizada si es necesario

### 6.2 Despliegue a Producción

- Merge a `main` dispara despliegue automático
- No requiere aprobación manual (CI/CD automatizado)
- Rollback disponible si es necesario (ver [PLANES_DE_ROLLBACK.md](./PLANES_DE_ROLLBACK.md))

## 7. Monitoreo Post-Despliegue

Después de cada despliegue:

1. Verificar health checks
2. Monitorear logs
3. Verificar métricas (CPU, memoria, errores)
4. Validar funcionalidad crítica

## 8. Comunicación de Cambios

- Los releases se publican automáticamente en GitHub
- Notificaciones opcionales a equipos relevantes
- Documentación actualizada en este repositorio

## 9. Excepciones y Casos Especiales

### 9.1 Hotfixes

Para correcciones urgentes en producción:

1. Crear branch desde `main`
2. Aplicar fix con commit `fix:`
3. Merge directo a `main` (bypass stage si es crítico)
4. Merge back a `stage` y `dev`

### 9.2 Rollback

Ver [PLANES_DE_ROLLBACK.md](./PLANES_DE_ROLLBACK.md) para procedimientos detallados.

## 10. Herramientas y Automatización

- **GitHub Actions**: CI/CD pipelines
- **Script Personalizado de Versionado**: Versionado semántico automático (implementado en bash dentro de GitHub Actions)
- **Conventional Commits**: Estándar de commits
- **GitHub Releases**: Gestión de releases

### 10.1 Detalles Técnicos del Versionado

El script de versionado semántico:

- **Lenguaje**: Bash script
- **Ubicación**: Job `calculate-version` en `.github/workflows/ci-cd-prod.yml`
- **Input**: Commits desde el último tag
- **Output**: Variables `version` y `image_tag` para uso en otros jobs
- **Formato de tags**: `prod-X.Y.Z` (ej: `prod-1.2.3`)

**Nota**: No se usa `semantic-release` (herramienta de npm) porque:
- Los servicios son Java/Maven, no Node.js
- El script personalizado es más simple y directo
- No requiere dependencias adicionales



