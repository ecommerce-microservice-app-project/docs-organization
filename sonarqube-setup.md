# Guía de Instalación y Configuración de SonarQube en GCP

**Fecha:** 26 de Noviembre, 2025  
**VM:** prod-monitoring-vm (GCP)  
**IP Pública:** 34.42.225.63  
**Sistema Operativo:** Ubuntu 24  
**RAM:** 16GB (inicialmente 3.8GB, ampliada durante el proceso)  
**Servicios existentes:** Grafana, Prometheus, kubectl proxy

---

## Tabla de Contenidos

1. [Contexto Inicial](#contexto-inicial)
2. [Preparación del Sistema](#preparación-del-sistema)
3. [Instalación de Docker y Docker Compose](#instalación-de-docker-y-docker-compose)
4. [Configuración Inicial de SonarQube](#configuración-inicial-de-sonarqube)
5. [Problemas Encontrados y Soluciones](#problemas-encontrados-y-soluciones)
6. [Actualización de SonarQube](#actualización-de-sonarqube)
7. [Integración con GitHub](#integración-con-github)
8. [Configuración de Pipelines](#configuración-de-pipelines)
9. [Conclusiones y Recomendaciones](#conclusiones-y-recomendaciones)

---

## Contexto Inicial

### Objetivo

Configurar SonarQube para análisis de código estático en una infraestructura de microservicios con aproximadamente 10 repositorios en GitHub (organización).

### Infraestructura Existente

- VM en GCP dedicada principalmente al monitoreo de cluster Kubernetes
- Grafana y Prometheus ya ejecutándose
- Múltiples proyectos con pipelines de CI/CD en GitHub Actions
- 3 ambientes por proyecto: dev, stage, prod

---

## Preparación del Sistema

### 1. Verificación de Recursos

#### Especificaciones Iniciales (INSUFICIENTES)

```bash
# Comando usado
lscpu | grep -E '^CPU\(s\)|^Model name'
free -h

# Resultado inicial
Model name: Intel(R) Xeon(R) CPU @ 2.20GHz
RAM total: 3.8GB
RAM usada: ~600MB (Grafana + Prometheus)
```

**Problema:** SonarQube requiere mínimo 4GB RAM, idealmente 6-8GB.

**Solución:** Ampliar la VM a 16GB RAM

```bash
gcloud compute instances stop prod-monitoring-vm
gcloud compute instances set-machine-type prod-monitoring-vm --machine-type=e2-standard-4
gcloud compute instances start prod-monitoring-vm
```

#### Especificaciones Finales (ADECUADAS)

```bash
# Resultado después de ampliar
Model name: Intel(R) Xeon(R) CPU @ 2.20GHz
RAM total: 16GB
RAM disponible: ~15GB
Disco: 20GB (31% usado)
```

### 2. Configuración del Sistema para SonarQube

SonarQube requiere ajustes específicos del kernel:

```bash
# Ajustes temporales (se pierden al reiniciar)
sudo sysctl -w vm.max_map_count=524288
sudo sysctl -w fs.file-max=131072
sudo ulimit -n 131072
sudo ulimit -u 8192

# Hacerlos permanentes
echo "vm.max_map_count=524288" | sudo tee -a /etc/sysctl.conf
echo "fs.file-max=131072" | sudo tee -a /etc/sysctl.conf
```

---

## Instalación de Docker y Docker Compose

### Instalación Completa

```bash
# Actualizar el sistema
sudo apt update

# Instalar dependencias
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common

# Agregar la clave GPG de Docker
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Agregar el repositorio de Docker
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Instalar Docker
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Agregar usuario al grupo docker
sudo usermod -aG docker $USER

# Verificar instalación
sudo docker --version
```

---

## Configuración Inicial de SonarQube

### 1. Crear Estructura de Directorios

```bash
mkdir -p ~/sonarqube
cd ~/sonarqube
```

### 2. Crear docker-compose.yml (Versión Inicial)

```yaml
version: "3"

services:
  sonarqube:
    image: sonarqube:lts-community
    container_name: sonarqube
    depends_on:
      - db
    environment:
      SONAR_JDBC_URL: jdbc:postgresql://db:5432/sonar
      SONAR_JDBC_USERNAME: sonar
      SONAR_JDBC_PASSWORD: sonar_pass_change_me
    ports:
      - "9000:9000"
    volumes:
      - sonarqube_data:/opt/sonarqube/data
      - sonarqube_extensions:/opt/sonarqube/extensions
      - sonarqube_logs:/opt/sonarqube/logs
    restart: unless-stopped

  db:
    image: postgres:13
    container_name: sonarqube_db
    environment:
      POSTGRES_USER: sonar
      POSTGRES_PASSWORD: sonar_pass_change_me
      POSTGRES_DB: sonar
    volumes:
      - postgresql_data:/var/lib/postgresql/data
    restart: unless-stopped

volumes:
  sonarqube_data:
  sonarqube_extensions:
  sonarqube_logs:
  postgresql_data:
```

### 3. Configurar Firewall en GCP

```bash
# Abrir puerto 9000 para SonarQube
gcloud compute firewall-rules create allow-sonarqube \
  --allow tcp:9000 \
  --description "Allow SonarQube access" \
  --direction INGRESS
```

### 4. Iniciar SonarQube

```bash
cd ~/sonarqube
sudo docker compose up -d

# Ver logs
sudo docker compose logs -f sonarqube
```

### 5. Primer Acceso

- **URL:** http://34.42.225.63:9000
- **Usuario por defecto:** admin
- **Contraseña por defecto:** admin
- **Acción requerida:** Cambiar contraseña en primer login

---

## Problemas Encontrados y Soluciones

### Problema 1: Versión Obsoleta de SonarQube

**Error:**

```
"You're running a version of SonarQube that is no longer active.
Please upgrade to an active version immediately."
```

**Causa:** La imagen `sonarqube:lts-community` descargaba una versión antigua sin soporte.

**Intento de Solución 1:** Actualizar directamente a versión más nueva

```bash
# Intentamos con versión específica
nano docker-compose.yml
# Cambiar a: image: sonarqube:25.11.0.114957-community

sudo docker compose pull
```

**Error Obtenido:**

```
The version of SonarQube you are trying to upgrade from is too old.
Please upgrade to the 24.12 version first.
```

**Solución Final:** Reinstalación limpia con versión actual

```bash
# Detener y eliminar todo
cd ~/sonarqube
sudo docker compose down

# Eliminar volúmenes (BORRA TODOS LOS DATOS)
sudo docker volume rm sonarqube_sonarqube_data sonarqube_sonarqube_extensions sonarqube_sonarqube_logs sonarqube_postgresql_data

# O borrar todos los volúmenes
sudo docker volume prune -f

# Editar docker-compose.yml
nano docker-compose.yml
# Cambiar a: image: sonarqube:community

# Levantar con versión nueva
sudo docker compose pull
sudo docker compose up -d
sudo docker compose logs -f sonarqube
```

**Resultado:** SonarQube versión 25.x instalada correctamente.

---

### Problema 2: Integración con GitHub - OAuth Callback Error

**Objetivo:** Integrar SonarQube con GitHub usando GitHub App para:

- Análisis automático de repositorios
- Decoración de Pull Requests
- Gestión centralizada

#### Configuración de GitHub App

**Pasos realizados:**

1. GitHub → Organization Settings → Developer settings → GitHub Apps → New GitHub App
2. Configuración:
   ```
   GitHub App name: SonarQube-ecommerce-microservice
   Homepage URL: http://34.42.225.63:9000
   Callback URL: http://34.42.225.63:9000/oauth2/callback/github
   Webhook URL: http://34.42.225.63:9000/api/alm/github/webhook
   Webhook Secret: [generado con openssl rand -base64 32]
   ```
3. Permisos configurados:
   - Contents: Read-only
   - Metadata: Read-only
   - Pull requests: Read & write
   - Checks: Read & write
4. Webhook events: Pull request, Push
5. OAuth: Request user authorization during installation

**Error Persistente:**

```
Be careful!
The redirect_uri is not associated with this application.
The application might be misconfigured or could be trying to
redirect you to a website you weren't expecting.
```

**URL que GitHub intentaba usar:**

```
http://34.42.225.63:9000/2Fprojects...
```

_(Nota: el encoding %2F indica un problema de configuración)_

#### Intentos de Solución

**Intento 1:** Verificar Callback URL en GitHub App

- Verificado: `http://34.42.225.63:9000/oauth2/callback/github`
- Checkbox marcado: Request user authorization (OAuth) during installation
- Resultado: Sin cambios

**Intento 2:** Configurar Server Base URL en SonarQube

Según documentación oficial de SonarQube:

```
Administration → Configuration → General Settings → General
Server base URL: http://34.42.225.63:9000
```

**Intento 3:** Reiniciar SonarQube

```bash
sudo docker compose restart
sudo docker compose logs -f sonarqube
```

Resultado: Sin cambios

**Intento 4:** Recrear Callback URL en GitHub

- Borrar y volver a agregar el Callback URL
- Guardar cambios
- Resultado: Mismo error

#### Documentación Consultada

URLs de callback encontradas en la documentación oficial:

- GitLab: `<URL>/oauth2/callback/gitlab`
- Bitbucket: `<URL>/oauth2/callback`
- Google: `<URL>/oauth2/callback/google`
- GitHub: `<URL>/oauth2/callback/github`

**Conclusión:** El formato del callback URL es correcto según documentación oficial de SonarQube.

#### Teorías sobre la Causa

1. **Problema de red/proxy:** La VM está detrás de un firewall de GCP
2. **HTTP vs HTTPS:** SonarQube recomienda HTTPS en producción
3. **Bug de versión:** Posible incompatibilidad entre versión de SonarQube y GitHub App API
4. **Configuración de red de Docker:** Posible problema con la red bridge de Docker

**Decisión Final:** Usar método manual con tokens en lugar de GitHub App OAuth.

---

### Problema 3: Personal Access Token no disponible en la interfaz

**Objetivo:** Usar Personal Access Token como alternativa a GitHub App.

**Problema:** La versión de SonarQube instalada solo mostraba opción de GitHub App, no de Personal Access Token.

**Ubicación esperada:**

```
Administration → DevOps Platform Integrations → GitHub → Create configuration
```

**Lo que se mostraba:**

- Solo formulario para GitHub App (Client ID, Client Secret, Private Key, Webhook Secret)
- No había dropdown ni opción para cambiar a Personal Access Token

**Causa:** Versión de SonarQube con interfaz limitada para configuración de GitHub.

**Solución:** No aplicable, se decidió usar método manual.

---

### Problema 4: Creación Automática de Proyectos sin Permisos

**Error al ejecutar pipeline:**

```
Subida de resultados: FALLA (proyecto no existe o sin permisos)
Error detallado:
- Conexión a SonarQube: OK
- Compilación del proyecto: OK
- Análisis del código: OK (archivos analizados correctamente)
- Subida de resultados: FALLA
```

**Causa:** El token generado no tenía permisos para crear proyectos automáticamente.

**Opciones de solución:**

**Opción 1:** Crear proyectos manualmente en SonarQube

```
1. Acceder a http://34.42.225.63:9000
2. Projects → Create Project → Manually
3. Project key: nombre-del-repositorio
4. Main branch: main
5. Generar token para el proyecto
```

**Opción 2:** Dar permisos de administrador al token

- No implementada por razones de seguridad
- Requiere token con permisos elevados

**Decisión:** Crear proyectos manualmente antes de ejecutar pipelines.

---

## Actualización de SonarQube

### Proceso de Actualización por Pasos (No Aplicado)

**Nota:** Este proceso fue documentado pero no aplicado, ya que se optó por reinstalación limpia.

```bash
# Paso 1: Actualizar a versión intermedia 24.12
cd ~/sonarqube
sudo docker compose down
nano docker-compose.yml
# Cambiar a: image: sonarqube:24.12-community

sudo docker compose pull
sudo docker compose up -d
sudo docker compose logs -f sonarqube

# Esperar 3-5 minutos hasta ver "SonarQube is operational"

# Paso 2: Actualizar a versión final
sudo docker compose down
nano docker-compose.yml
# Cambiar a: image: sonarqube:community

sudo docker compose pull
sudo docker compose up -d
```

### Versiones Disponibles Consultadas

```bash
curl -s https://hub.docker.com/v2/repositories/library/sonarqube/tags?page_size=100 | grep -o '"name":"[^"]*"' | head -20

Resultados:
- latest
- community
- 25.11.0.114957-community (más reciente)
- 2025.5.0-enterprise
- 2025.4-lta-enterprise
```

---

## Integración con GitHub

### Método Final: Configuración Manual

Debido a los problemas con GitHub App OAuth, se implementó el método manual.

### 1. Crear Personal Access Token en GitHub

**Para Organización:**

```
GitHub → Organization Settings → Personal access tokens → Settings
→ Fine-grained tokens → Generate new token

Configuración:
- Token name: SonarQube-Organization
- Resource owner: [tu-organización]
- Repository access: All repositories (o específicos)
- Permissions:
  * Contents: Read-only
  * Metadata: Read-only
  * Pull requests: Read and write
  * Webhooks: Read and write
```

**Para usuario personal:**

```
GitHub → Settings → Developer settings → Personal access tokens
→ Tokens (classic) → Generate new token (classic)

Permisos:
-  repo (todos los sub-permisos)
-  admin:org → read:org
-  admin:repo_hook → write:repo_hook
```

### 2. Configurar Secrets a Nivel de Organización

```
GitHub → Organization Settings → Secrets and variables → Actions
→ New organization secret

Secrets creados:
1. SONAR_TOKEN: [token generado en SonarQube para cada proyecto]
2. SONAR_HOST_URL: http://34.42.225.63:9000 (como variable, no secret)

Repository access: All repositories
```

### 3. Crear Proyectos en SonarQube

**Para cada repositorio:**

```
1. SonarQube → Projects → Create Project → Manually
2. Project display name: nombre-del-repo
3. Project key: nombre-del-repo (usar mismo nombre que repo de GitHub)
4. Main branch name: main (o master)
5. Next
6. With GitHub Actions
7. Copiar el token generado
8. Actualizar SONAR_TOKEN en GitHub si es necesario
```

---

## Configuración de Pipelines

### Estructura de Proyecto

```
Organización:
├── backend-service/
│   ├── .github/workflows/
│   │   ├── dev.yml
│   │   ├── stage.yml
│   │   └── prod.yml
│   └── sonar-project.properties
├── frontend-service/
│   ├── .github/workflows/
│   │   ├── dev.yml
│   │   ├── stage.yml
│   │   └── prod.yml
│   └── sonar-project.properties
├── api-gateway/
│   ├── .github/workflows/
│   │   ├── dev.yml
│   │   ├── stage.yml
│   │   └── prod.yml
│   └── sonar-project.properties
├── ...
└── infrastructure/
    ├── .github/workflows/
    │   └── sonarqube.yml (NUEVO)
    └── sonar-project.properties (NUEVO)
```

### Configuración de Pipeline para Proyectos de Desarrollo

**Solo modificar pipelines de DEV, no stage ni prod**

Agregar al final del archivo `.github/workflows/dev.yml`:

```yaml
sonarqube:
  name: SonarQube Analysis
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v3
      with:
        fetch-depth: 0 # Importante para análisis completo

    - name: SonarQube Scan
      uses: sonarsource/sonarqube-scan-action@master
      env:
        SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
        SONAR_HOST_URL: ${{ vars.SONAR_HOST_URL }}
```

### Archivo sonar-project.properties

Crear en la raíz de cada proyecto:

```properties
# Identificador único del proyecto (debe coincidir con el creado en SonarQube)
sonar.projectKey=nombre-del-repositorio

# Directorio de código fuente
sonar.sources=.

# URL del servidor SonarQube
sonar.host.url=http://34.42.225.63:9000

# Exclusiones (ajustar según tecnología)
sonar.exclusions=**/node_modules/**,**/dist/**,**/build/**,**/*.test.js,**/*.spec.js,**/coverage/**,**/target/**,**/__pycache__/**,**/.pytest_cache/**

# Configuraciones específicas por lenguaje (ejemplos)

# Para Java/Maven
# sonar.java.binaries=target/classes

# Para JavaScript/TypeScript
# sonar.javascript.lcov.reportPaths=coverage/lcov.info

# Para Python
# sonar.python.coverage.reportPaths=coverage.xml
```

### Pipeline para Repositorio de Infraestructura

Como este repo no tiene pipelines, crear uno nuevo:

`.github/workflows/sonarqube.yml`:

```yaml
name: SonarQube Analysis

on:
  push:
    branches:
      - main
      - dev
  pull_request:
    types: [opened, synchronize, reopened]

jobs:
  sonarqube:
    name: SonarQube Analysis
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0

      - name: SonarQube Scan
        uses: sonarsource/sonarqube-scan-action@master
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
          SONAR_HOST_URL: ${{ vars.SONAR_HOST_URL }}
```

### Estrategia de Commits

**Para cada repositorio hacer DOS commits:**

**Commit 1: Configuración**

```bash
git add .github/workflows/dev.yml sonar-project.properties
git commit -m "chore: configure SonarQube analysis"
git push
```

**Commit 2: Trigger del pipeline**

```bash
# Hacer un cambio mínimo para disparar el pipeline
echo "" >> README.md  # o cualquier cambio pequeño
git add README.md
git commit -m "chore: trigger SonarQube pipeline"
git push
```

### Visualización del Dashboard

Después del primer análisis exitoso:

```
Acceder a: http://34.42.225.63:9000

Dashboard incluye:
├── Overview
│   ├── Bugs
│   ├── Vulnerabilidades
│   ├── Code Smells
│   ├── Coverage
│   ├── Duplications
│   └── Security Hotspots
├── Issues
│   ├── Lista detallada
│   ├── Filtros por severidad
│   └── Filtros por tipo
├── Measures
│   ├── Reliability
│   ├── Security
│   ├── Maintainability
│   └── Coverage
├── Code
│   └── Navegador de código con issues inline
└── Activity
    └── Historial de análisis
```

---

## Conclusiones y Recomendaciones

### Lo que Funcionó

**Instalación de SonarQube con Docker Compose**

- Método más simple y mantenible
- Fácil de actualizar y hacer backup
- Separación de concerns con PostgreSQL

  **Ampliación de RAM de la VM**

- Crítico para el rendimiento de SonarQube
- 16GB es adecuado para SonarQube + Grafana + Prometheus

  **Configuración de Secrets a Nivel de Organización**

- Un solo lugar para gestionar tokens
- Fácil rotación de credenciales
- Todos los repositorios comparten configuración

  **Método Manual de Creación de Proyectos**

- Más control sobre la configuración
- Evita problemas de permisos
- Funciona de inmediato

### Lo que NO Funcionó

**GitHub App OAuth Integration**

- Error persistente de redirect_uri
- Múltiples intentos de configuración fallidos
- Documentación seguida correctamente
- Problema posiblemente relacionado con HTTP vs HTTPS o configuración de red

  **Actualización Incremental de Versiones**

- Versión inicial demasiado antigua
- Requería múltiples pasos de actualización
- Reinstalación limpia fue más rápida

  **Personal Access Token directo en SonarQube**

- Interfaz de la versión instalada no lo soportaba adecuadamente
- Solo mostraba opción de GitHub App

### Recomendaciones Futuras

#### 1. Configurar HTTPS

**Por qué:**

- SonarQube recomienda HTTPS en producción
- Podría resolver el problema de OAuth con GitHub
- Mejor seguridad para tokens y credenciales

**Cómo:**

```bash
# Opción A: Usar Let's Encrypt con Nginx como reverse proxy
sudo apt install nginx certbot python3-certbot-nginx

# Configurar Nginx
sudo nano /etc/nginx/sites-available/sonarqube

server {
    listen 80;
    server_name sonarqube.tudominio.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name sonarqube.tudominio.com;

    ssl_certificate /etc/letsencrypt/live/sonarqube.tudominio.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/sonarqube.tudominio.com/privkey.pem;

    location / {
        proxy_pass http://localhost:9000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# Obtener certificado
sudo certbot --nginx -d sonarqube.tudominio.com

# Opción B: Usar Cloud Load Balancer de GCP con certificado SSL
```

#### 2. Implementar Backup Automático

**Script de backup:**

```bash
#!/bin/bash
# backup-sonarqube.sh

BACKUP_DIR="/home/torres/backups/sonarqube"
DATE=$(date +%Y%m%d_%H%M%S)

mkdir -p $BACKUP_DIR

# Backup de la base de datos PostgreSQL
docker exec sonarqube_db pg_dump -U sonar sonar > $BACKUP_DIR/sonar_db_$DATE.sql

# Backup de volúmenes de Docker
docker run --rm -v sonarqube_sonarqube_data:/data -v $BACKUP_DIR:/backup ubuntu tar czf /backup/sonarqube_data_$DATE.tar.gz -C /data .
docker run --rm -v sonarqube_sonarqube_extensions:/data -v $BACKUP_DIR:/backup ubuntu tar czf /backup/sonarqube_extensions_$DATE.tar.gz -C /data .

# Mantener solo los últimos 7 backups
find $BACKUP_DIR -name "*.sql" -mtime +7 -delete
find $BACKUP_DIR -name "*.tar.gz" -mtime +7 -delete

echo "Backup completado: $DATE"
```

**Cron job:**

```bash
# Ejecutar backup diario a las 2 AM
crontab -e
0 2 * * * /home/torres/scripts/backup-sonarqube.sh >> /var/log/sonarqube-backup.log 2>&1
```

#### 3. Configurar Quality Gates Personalizados

**En SonarQube:**

```
Quality Gates → Create
Configurar umbrales:
- Coverage: > 80%
- Duplications: < 3%
- Maintainability Rating: A
- Reliability Rating: A
- Security Rating: A
```

**Aplicar a proyectos:**

```
Project Settings → Quality Gate → Seleccionar custom gate
```

#### 4. Monitoreo de SonarQube

**Integrar con Prometheus (ya instalado):**

`prometheus.yml`:

```yaml
scrape_configs:
  - job_name: "sonarqube"
    static_configs:
      - targets: ["localhost:9000"]
    metrics_path: "/api/monitoring/metrics"
```

**Dashboard en Grafana:**

- Importar dashboard de SonarQube (ID: 9139)
- Métricas clave: análisis completados, tiempo de análisis, issues por severidad

#### 5. Implementar Webhooks para Notificaciones

**En SonarQube:**

```
Administration → Configuration → Webhooks → Create

URL: https://hooks.slack.com/services/YOUR/WEBHOOK/URL
Secret: [opcional]

Events:
- Quality Gate status changed
- New issues detected
```

#### 6. Rotación Regular de Tokens

**Proceso recomendado:**

```
1. Cada 90 días, generar nuevo token en SonarQube
2. Actualizar SONAR_TOKEN en GitHub Organization Secrets
3. Verificar que todos los pipelines funcionan
4. Revocar token antiguo
5. Documentar en log de cambios
```

#### 7. Documentación Interna

**Crear wiki interno con:**

- Cómo crear un nuevo proyecto en SonarQube
- Cómo interpretar métricas del dashboard
- Cómo resolver issues comunes
- Proceso de escalamiento de problemas

#### 8. Limpieza de Proyectos Antiguos

**Política recomendada:**

```bash
# Script para identificar proyectos sin análisis recientes
# Ejecutar manualmente cada 3 meses

curl -u admin:tu-password \
  "http://34.42.225.63:9000/api/projects/search" \
  | jq '.components[] | select(.lastAnalysisDate < "2024-08-01") | .key'

# Eliminar proyectos obsoletos manualmente desde la UI
```

### Métricas de Éxito

**Después de 1 mes de uso:**

- [ ] Todos los repositorios activos tienen análisis configurado
- [ ] Al menos 80% de cobertura de código en proyectos críticos
- [ ] Reducción de code smells en 30%
- [ ] Zero bugs críticos sin resolver por más de 1 semana
- [ ] Dashboards de Grafana mostrando métricas de SonarQube

**Después de 3 meses:**

- [ ] Quality Gates integrados en proceso de merge
- [ ] PRs bloqueados automáticamente si no pasan quality gate
- [ ] Deuda técnica reducida en 50%
- [ ] Equipo familiarizado con métricas de SonarQube

### Costos de GCP

**Estimación mensual con VM actual:**

```
VM e2-standard-4 (4 vCPU, 16GB RAM):
- Costo estimado: ~$120-150 USD/mes
- Incluye: SonarQube + Grafana + Prometheus + kubectl proxy

Alternativa más económica:
- VM e2-standard-2 (2 vCPU, 8GB RAM): ~$60-80 USD/mes
- Suficiente si solo se usa SonarQube ocasionalmente
```

**Opción de reducción de costos:**

```bash
# Script para apagar VM automáticamente fuera de horario laboral
# Usar Cloud Scheduler + Cloud Functions

# Apagar a las 8 PM
gcloud compute instances stop prod-monitoring-vm --zone=us-central1-a

# Encender a las 8 AM
gcloud compute instances start prod-monitoring-vm --zone=us-central1-a

Ahorro estimado: 50-60% del costo mensual
```

### Recursos Adicionales

**Documentación Oficial:**

- SonarQube Docs: https://docs.sonarsource.com/sonarqube
- GitHub Actions Integration: https://docs.sonarsource.com/sonarqube/latest/analyzing-source-code/ci-integration/github-actions/
- Quality Gates: https://docs.sonarsource.com/sonarqube/latest/user-guide/quality-gates/

**Comunidad:**

- SonarQube Community: https://community.sonarsource.com/
- Stack Overflow tag: [sonarqube]
- GitHub Issues: https://github.com/SonarSource/sonarqube/issues

**Tutoriales Recomendados:**

- SonarQube Best Practices
- Code Coverage con diferentes frameworks
- Custom Rules para análisis específicos

---

## Apéndice A: Comandos Útiles

### Gestión de Docker Compose

```bash
# Ver estado de contenedores
sudo docker compose ps

# Ver logs en tiempo real
sudo docker compose logs -f

# Ver logs de un servicio específico
sudo docker compose logs -f sonarqube

# Reiniciar servicios
sudo docker compose restart

# Detener servicios
sudo docker compose stop

# Iniciar servicios
sudo docker compose start

# Detener y eliminar contenedores (mantiene volúmenes)
sudo docker compose down

# Detener, eliminar contenedores Y volúmenes (BORRA DATOS)
sudo docker compose down -v

# Actualizar imágenes
sudo docker compose pull

# Ver uso de recursos
sudo docker stats
```

### Gestión de Volúmenes

```bash
# Listar volúmenes
sudo docker volume ls

# Inspeccionar un volumen
sudo docker volume inspect sonarqube_sonarqube_data

# Eliminar volúmenes no utilizados
sudo docker volume prune

# Eliminar un volumen específico
sudo docker volume rm sonarqube_sonarqube_data

# Ver tamaño de volúmenes
sudo du -sh /var/lib/docker/volumes/*
```

### Diagnóstico

```bash
# Ver logs del sistema
sudo journalctl -u docker

# Ver uso de recursos del sistema
htop
free -h
df -h

# Verificar conectividad
curl http://localhost:9000/api/system/status

# Ver configuración de SonarQube
sudo docker exec sonarqube cat /opt/sonarqube/conf/sonar.properties

# Acceder a la base de datos
sudo docker exec -it sonarqube_db psql -U sonar -d sonar

# Verificar firewall de GCP
gcloud compute firewall-rules list --filter="name:sonarqube"
```

### Mantenimiento

```bash
# Limpiar recursos de Docker no utilizados
sudo docker system prune -a

# Ver espacio usado por Docker
sudo docker system df

# Reiniciar Docker daemon
sudo systemctl restart docker

# Ver versión de SonarQube instalada
curl -u admin:tu-password http://34.42.225.63:9000/api/server/version
```

---

## Apéndice B: Problemas Comunes y Soluciones

### 1. SonarQube no inicia

**Síntomas:**

- Contenedor se reinicia constantemente
- Logs muestran errores de memoria o permisos

**Soluciones:**

```bash
# Verificar configuración del kernel
sysctl vm.max_map_count
sysctl fs.file-max

# Si son incorrectos, aplicar de nuevo
sudo sysctl -w vm.max_map_count=524288
sudo sysctl -w fs.file-max=131072

# Verificar permisos de volúmenes
sudo chown -R 1000:1000 /var/lib/docker/volumes/sonarqube*

# Reiniciar contenedores
sudo docker compose restart
```

### 2. Error de conexión a base de datos

**Síntomas:**

- SonarQube no puede conectarse a PostgreSQL
- Logs muestran "Connection refused"

**Soluciones:**

```bash
# Verificar que PostgreSQL esté corriendo
sudo docker compose ps

# Ver logs de PostgreSQL
sudo docker compose logs db

# Verificar credenciales en docker-compose.yml
cat docker-compose.yml | grep -A5 "POSTGRES"

# Reiniciar solo la base de datos
sudo docker compose restart db
```

### 3. Pipeline falla con "Project not found"

**Solución:**

```
1. Ir a SonarQube: http://34.42.225.63:9000
2. Projects → Create Project → Manually
3. Usar exactamente el mismo nombre que en sonar-project.properties
4. Generar token para el proyecto
5. Actualizar SONAR_TOKEN en GitHub si es necesario
6. Re-ejecutar pipeline
```

### 4. Análisis muy lento

**Posibles causas:**

- Muchos archivos para analizar
- RAM insuficiente
- Exclusiones mal configuradas

**Soluciones:**

```properties
# Agregar más exclusiones en sonar-project.properties
sonar.exclusions=**/node_modules/**,**/dist/**,**/build/**,**/target/**,**/coverage/**,**/.git/**,**/.next/**,**/.nuxt/**

# Aumentar memoria de contenedor en docker-compose.yml
services:
  sonarqube:
    deploy:
      resources:
        limits:
          memory: 4G
```

### 5. Dashboard vacío después de análisis

**Verificar:**

```bash
# Ver logs del análisis en GitHub Actions
# Buscar líneas como:
# - "ANALYSIS SUCCESSFUL"
# - "More about the report processing"

# Verificar en SonarQube
curl -u admin:tu-password \
  "http://34.42.225.63:9000/api/ce/activity?componentKey=nombre-proyecto"

# Si status es SUCCESS pero no hay datos, verificar:
# - sonar.projectKey en sonar-project.properties
# - sonar.sources en sonar-project.properties
```

---

## Apéndice C: Checklist de Instalación

### Pre-instalación

- [ ] VM con al menos 8GB RAM (16GB recomendado)
- [ ] Ubuntu 20.04 o superior
- [ ] Acceso SSH a la VM
- [ ] Permisos de administrador
- [ ] Puerto 9000 disponible

### Instalación

- [ ] Docker instalado
- [ ] Docker Compose instalado
- [ ] Usuario agregado al grupo docker
- [ ] Configuración del kernel aplicada
- [ ] docker-compose.yml creado
- [ ] Firewall de GCP configurado
- [ ] SonarQube iniciado
- [ ] Acceso web verificado
- [ ] Contraseña cambiada

### Post-instalación

- [ ] Server base URL configurado
- [ ] Proyectos creados en SonarQube
- [ ] Tokens generados
- [ ] Secrets configurados en GitHub
- [ ] sonar-project.properties en cada repo
- [ ] Pipelines modificados
- [ ] Primer análisis exitoso
- [ ] Dashboard visible con datos
- [ ] Backup configurado (opcional)
- [ ] Monitoreo configurado (opcional)

---

## Registro de Cambios

| Fecha      | Versión SonarQube              | Cambio                            | Realizado por |
| ---------- | ------------------------------ | --------------------------------- | ------------- |
| 2025-11-26 | N/A → lts-community            | Instalación inicial               | Torres        |
| 2025-11-26 | lts-community → 25.x community | Reinstalación limpia              | Torres        |
| 2025-11-26 | 25.x                           | Configuración de proyectos manual | Torres        |

---

## Contactos y Soporte

**Administrador del sistema:** Torres  
**Ubicación de archivos:** `/home/torres/sonarqube/`  
**Documentación interna:** [Agregar enlace a wiki interna]  
**Incidencias:** [Agregar sistema de tickets]
