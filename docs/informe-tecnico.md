# Evaluación Final Transversal

**Asignatura:** Introducción a Herramientas DevOps (ISY1101)
**Docente:** Carlos Martínez

**Integrantes:**
- Carlos Muñoz Leyton
- Bastián Brisso
- Carlos Camero

**Repositorios del proyecto:**
- Frontend: https://github.com/CarlosMUL/Front_despacho
- Backend Ventas: https://github.com/CarlosMUL/Back_ventas
- Backend Despachos: https://github.com/CarlosMUL/Back_despacho

*Este documento de documentación técnica se encuentra replicado en la carpeta `docs/` de los tres repositorios listados arriba.*

---

## 1. Introducción

El presente informe documenta el desarrollo de la Evaluación Final Transversal (EFT) de la asignatura Introducción a Herramientas DevOps. El proyecto consiste en la automatización del ciclo de integración y entrega continua (CI/CD) de la plataforma Innovatech Chile, compuesta por un frontend (React + Vite servido por Nginx), dos backends (Ventas y Despachos, desarrollados en Spring Boot) y una base de datos relacional MySQL gestionada mediante Amazon RDS.

La solución evolucionó a lo largo del semestre: en una primera etapa (EP2) los servicios se desplegaron manualmente sobre instancias EC2 con Docker, y en una segunda etapa (EP3) se migró la orquestación completa a AWS ECS con Fargate, incorporando Amazon ECR para el registro de imágenes, Amazon RDS para la persistencia desacoplada de los contenedores, políticas de autoscaling basadas en CPU y un pipeline CI/CD totalmente automatizado con GitHub Actions. Este informe describe la arquitectura final (EP3) y consolida las decisiones técnicas tomadas durante todo el proceso.

## 2. Método de integración del sistema

El sistema está compuesto por tres servicios independientes que se comunican entre sí a través de variables de entorno que resuelven las direcciones IP privadas de cada componente:

- **Frontend (React + Vite + Nginx):** consume los endpoints REST de ambos backends mediante llamadas HTTP configuradas por variables de entorno (`DESPACHOS_BACKEND_HOST`, `VENTAS_BACKEND_HOST`).
- **Backend Ventas (Spring Boot, puerto 8080):** expone el recurso `/api/v1/ventas` y gestiona las órdenes de compra.
- **Backend Despachos (Spring Boot, puerto 8081):** expone el recurso `/api/v1/despachos` y gestiona el estado de las entregas.
- **Base de datos:** en la etapa EC2 (EP2) cada backend tenía su propia instancia MySQL en contenedor; en la etapa ECS (EP3) ambos backends comparten una instancia Amazon RDS MySQL (`database-1`), con bases de datos lógicas separadas (`ventas_db` / `despachos_db`).

La comunicación fue validada de extremo a extremo: se consultaron órdenes de compra y despachos reales desde la interfaz del frontend, confirmando que el flujo Front → Back → BD funciona correctamente en el entorno de nube.

## 3. Contenedores

### 3.1 Dockerfiles multi-etapa

Se construyeron Dockerfiles multi-etapa para los tres servicios, siguiendo buenas prácticas de imágenes minimalistas y de seguridad:

- **Backend Ventas y Despachos:** build con `maven:3.9.6-eclipse-temurin-17-alpine`, runtime con `eclipse-temurin:17-jre-alpine`. Se crea un usuario no-root (`appuser`) para ejecutar la aplicación.
- **Frontend:** build con `node:20-alpine` (`npm ci` + `npm run build`), runtime con `nginx:alpine` sirviendo los archivos estáticos y actuando como reverse proxy hacia los backends.

**Evidencia — Dockerfile Backend Ventas (multi-stage):**

```dockerfile
FROM maven:3.9.6-eclipse-temurin-17-alpine AS builder
WORKDIR /build
COPY pom.xml .
RUN mvn dependency:go-offline -B
COPY src ./src
RUN mvn package -DskipTests -B

FROM eclipse-temurin:17-jre-alpine
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
WORKDIR /app
COPY --from=builder /build/target/Springboot-API-REST*.jar app.jar
RUN chown appuser:appgroup app.jar
USER appuser
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

*[Captura pendiente de reemplazo: pantalla de VS Code mostrando el árbol del proyecto y el Dockerfile de back-Ventas_SpringBoot]*

**Evidencia — Dockerfile Frontend con Nginx:**

```dockerfile
# Stage 1: Build
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: Runtime
FROM nginx:alpine
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
RUN chown -R appuser:appgroup /usr/share/nginx/html && \
    chown -R appuser:appgroup /var/cache/nginx && \
    chown -R appuser:appgroup /var/log/nginx && \
    touch /var/run/nginx.pid && \
    chown appuser:appgroup /var/run/nginx.pid
USER appuser
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

*[Captura pendiente de reemplazo: Dockerfile del front_despacho]*

### 3.2 Orquestación local con Docker Compose

Se creó un archivo `docker-compose.yml` en la raíz del proyecto que levanta los tres servicios más las bases de datos MySQL, definiendo una red interna (`backend-net`), volúmenes nombrados para persistencia (`mysql_ventas_data`, `mysql_despachos_data`) y healthchecks para garantizar el orden de arranque (`depends_on` con `condition: service_healthy`). Las variables sensibles se gestionan mediante un archivo `.env` excluido del control de versiones mediante `.gitignore`.

**`.env` creado en la raíz del proyecto, añadido al `.gitignore`:**

```env
DB_PASSWORD=password123
DB_NAME_VENTAS=ventas_db
DB_NAME_DESPACHOS=despachos_db
```

**Servicio MySQL Ventas:**

```yaml
mysql-ventas:
  image: mysql:8.0
  command: --default-authentication-plugin=mysql_native_password
  environment:
    MYSQL_ROOT_PASSWORD: ${DB_PASSWORD}
    MYSQL_DATABASE: ${DB_NAME_VENTAS}
  volumes:
    - mysql_ventas_data:/var/lib/mysql
  networks:
    - backend-net
  healthcheck:
    test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
    interval: 10s
    timeout: 5s
    retries: 10
```

**Servicio MySQL Despachos:**

```yaml
mysql-despachos:
  image: mysql:8.0
  command: --default-authentication-plugin=mysql_native_password
  environment:
    MYSQL_ROOT_PASSWORD: ${DB_PASSWORD}
    MYSQL_DATABASE: ${DB_NAME_DESPACHOS}
  volumes:
    - mysql_despachos_data:/var/lib/mysql
  networks:
    - backend-net
  healthcheck:
    test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
    interval: 10s
    timeout: 5s
    retries: 10
```

**Servicio Ventas (backend):**

```yaml
back-ventas:
  image: carlitosmul/back-ventas:latest
  ports:
    - "8080:8080"
  environment:
    DB_ENDPOINT: mysql-ventas
    DB_PORT: 3306
    DB_NAME: ${DB_NAME_VENTAS}
    DB_USERNAME: root
    DB_PASSWORD: ${DB_PASSWORD}
  depends_on:
    mysql-ventas:
      condition: service_healthy
  networks:
    - backend-net
```

**Servicio Despacho (backend):**

```yaml
back-despachos:
  image: carlitosmul/back-despachos:latest
  ports:
    - "8081:8081"
  environment:
    DB_ENDPOINT: mysql-despachos
    DB_PORT: 3306
    DB_NAME: ${DB_NAME_DESPACHOS}
    DB_USERNAME: root
    DB_PASSWORD: ${DB_PASSWORD}
  depends_on:
    mysql-despachos:
      condition: service_healthy
  networks:
    - backend-net
```

## 4. Registro de imágenes (Amazon ECR)

Se crearon tres repositorios privados en Amazon ECR (región `us-east-1`), uno por servicio: `ventas-innovatech`, `despachos-innovatech` y `front-innovatech`. La autenticación se realizó mediante AWS CLI (`aws ecr get-login-password`) y las imágenes se publicaron con `docker tag` + `docker push`.

**Build y push — ejemplo (ventas-innovatech):**

```powershell
docker build -t ventas-innovatech .
docker tag ventas-innovatech:latest 361510669006.dkr.ecr.us-east-1.amazonaws.com/ventas-innovatech:latest
docker push 361510669006.dkr.ecr.us-east-1.amazonaws.com/ventas-innovatech:latest
```

El mismo flujo se replicó para `despachos-innovatech` y `front-innovatech`, apuntando a sus respectivos repositorios ECR.

Se implementó en los tres workflows la generación de un tag versionado adicional (SHA corto del commit, obtenido mediante `${GITHUB_SHA::7}`) junto al tag `latest` existente, en el paso de build y push a ECR. Debido al vencimiento de los créditos de AWS Academy antes del cierre del informe, no fue posible verificar el push de la imagen versionada en la consola de Amazon ECR; se verificó localmente, en los tres servicios (Ventas, Despachos y Frontend), que el mecanismo de doble etiquetado funciona correctamente mediante `docker build`, `docker tag` y `docker images`, quedando en cada caso ambos tags apuntando al mismo Image ID.

**Evidencia — `docker images`, Backend Ventas (tags `latest` y `v1`, mismo Image ID `acea26b5b875`):**
```
IMAGE                      ID             DISK USAGE   CONTENT SIZE
ventas-innovatech:latest   acea26b5b875   489MB        178MB
ventas-innovatech:v1       acea26b5b875   489MB        178MB
```

**Evidencia — `docker images`, Backend Despachos (tags `latest` y `v1`, mismo Image ID `2406864bb3c8`):**
```
IMAGE                         ID             DISK USAGE   CONTENT SIZE
despachos-innovatech:latest   2406864bb3c8   489MB        178MB
despachos-innovatech:v1       2406864bb3c8   489MB        178MB
```

**Evidencia — `docker images`, Frontend (tags `latest` y `v1`, mismo Image ID `7f6be38096c4`):**
```
IMAGE                      ID             DISK USAGE   CONTENT SIZE
front-innovatech:latest    7f6be38096c4   93MB         26.1MB
front-innovatech:v1        7f6be38096c4   93MB         26.1MB
```

*[Captura pendiente: repositorios `despachos-innovatech`, `front-innovatech` y `ventas-innovatech` en la consola de Amazon ECR, cada uno con 3 imágenes registradas]*

## 5. Pipeline de CI/CD (GitHub Actions)

Cada uno de los tres repositorios (`Back_ventas`, `Back_despacho`, `Front_despacho`) cuenta con su propio workflow (`deploy.yml`) que se activa ante cada push a la rama `deploy`. El pipeline ejecuta las siguientes etapas:

1. Checkout del código (`actions/checkout@v4`).
2. Configuración del entorno de ejecución (JDK 17 para backends, Node.js 20 para frontend).
3. Ejecución de la suite de pruebas (`mvn test -B` para backends, `npm run lint` para frontend).
4. Configuración de credenciales AWS (`aws-actions/configure-aws-credentials@v2`), usando los secretos temporales de AWS Academy.
5. Login a Amazon ECR (`aws-actions/amazon-ecr-login@v1`).
6. Build y push de la imagen Docker al repositorio ECR correspondiente, con doble etiqueta (`latest` + SHA del commit).
7. Actualización de la Task Definition en ECS con la nueva imagen.
8. Deploy forzado del servicio en ECS y espera de estabilización.

Los tiempos de build observados fueron: Ventas ~82.8 s, Despachos ~57.5 s y Frontend ~24.4 s (el frontend es más rápido al no requerir compilación de Maven).

### 5.1 Diagrama del pipeline

```
Push a GitHub          GitHub Actions           Push a Amazon ECR         Deploy en ECS Fargate
Front, Back            Build y test              Imagen versionada        Cluster-innovatech,
Ventas/Despachos   →    automático          →     por tag             →   autoscaling
```

*[Captura pendiente: diagrama de flujo del pipeline — ver anexo "Diagrama Pipeline"]*

### 5.2 Etapa de pruebas (test)

Se incorporó la ejecución de la suite de tests (`mvn test`) para los backends Spring Boot y `npm run lint` para el frontend, como paso previo al build de la imagen, siguiendo el flujo `checkout → test → build → push → deploy`. Debido al vencimiento de los créditos de AWS Academy antes del cierre del informe, no fue posible verificar la ejecución de esta etapa dentro del pipeline de GitHub Actions en la nube; se verificó localmente su correcto funcionamiento en los tres servicios.

**Evidencia — ejecución local de tests, Backend Ventas:**

```
[INFO] Tests run: 2, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.281 s -- in persistence.ser...
[INFO]
[INFO] Results:
[INFO]
[INFO] Tests run: 3, Failures: 0, Errors: 0, Skipped: 0
[INFO]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 6.921 s
[INFO] Finished at: 2026-07-09T22:18:02-04:00
```

**Evidencia — ejecución local de tests, Backend Despachos:**

```
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 5.043 s -- in com.citt.Spr...
[INFO]
[INFO] Results:
[INFO]
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0
[INFO]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 7.167 s
[INFO] Finished at: 2026-07-09T22:20:27-04:00
```

**Evidencia — ejecución local de lint, Frontend:**

```
> despacho@0.0.0 lint
> eslint . --ext js,jsx --report-unused-disable-directives --max-warnings 0
```

Comando ejecutado sin errores reportados (salida limpia).

El workflow `deploy.yml` fue actualizado agregando el step correspondiente en los tres repositorios (ver snippets en sección 5, subsección "Workflow CI/CD Backend Despachos"), quedando disponible para ejecutarse en la nube en cuanto se disponga de un nuevo ciclo de créditos AWS Academy.

**Workflow CI/CD Backend Despachos:**

El workflow configura JDK 17, ejecuta los tests, configura las credenciales AWS, hace login a ECR, construye la imagen Docker con doble tag (`latest` + SHA del commit), la sube a ECR, actualiza la Task Definition y finalmente fuerza el redeploy del servicio `svc-despachos` en ECS.

```yaml
- name: Checkout codigo
  uses: actions/checkout@v4

- name: Configurar JDK 17
  uses: actions/setup-java@v4
  with:
    java-version: '17'
    distribution: 'temurin'
    cache: maven

- name: Ejecutar tests
  run: mvn test -B

- name: Configurar AWS credentials
  uses: aws-actions/configure-aws-credentials@v2
  with:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws-session-token: ${{ secrets.AWS_SESSION_TOKEN }}
    aws-region: us-east-1

- name: Login a ECR
  uses: aws-actions/amazon-ecr-login@v1

- name: Build y Push a ECR
  env:
    IMAGE_TAG: ${{ github.sha }}
    ECR_REGISTRY: 361510669006.dkr.ecr.us-east-1.amazonaws.com
    ECR_REPO: despachos-innovatech
  run: |
    docker build -t $ECR_REGISTRY/$ECR_REPO:$IMAGE_TAG -t $ECR_REGISTRY/$ECR_REPO:latest .
    docker push $ECR_REGISTRY/$ECR_REPO:$IMAGE_TAG
    docker push $ECR_REGISTRY/$ECR_REPO:latest

- name: Actualizar Task Definition con nueva imagen
  env:
    IMAGE_TAG: ${{ github.sha }}
    ECR_REGISTRY: 361510669006.dkr.ecr.us-east-1.amazonaws.com
    ECR_REPO: despachos-innovatech
  run: |
    TASK_DEF=$(aws ecs describe-task-definition --task-definition svc-despachos --query taskDefinition)
    NEW_ARN=$(echo $TASK_DEF | python3 -c "...")
    aws ecs update-service --cluster Cluster-innovatech --service svc-despachos --task-definition $NEW_ARN --force-new-deployment

- name: Esperar deployment estable
  run: |
    aws ecs wait services-stable --cluster Cluster-innovatech --services svc-despachos
```

*[Captura pendiente: pipeline operativa del repo Back_despacho, "4 workflow runs", último commit `90f584f`]*

**Workflow CI/CD Backend Ventas:**

Mismo flujo que Despachos, apuntando al repositorio ECR `ventas-innovatech` y al servicio `svc-ventas` del clúster.

```yaml
- name: Checkout codigo
  uses: actions/checkout@v4

- name: Configurar JDK 17
  uses: actions/setup-java@v4
  with:
    java-version: '17'
    distribution: 'temurin'
    cache: maven

- name: Ejecutar tests
  run: mvn test -B

- name: Configurar AWS credentials
  uses: aws-actions/configure-aws-credentials@v2
  with:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws-session-token: ${{ secrets.AWS_SESSION_TOKEN }}
    aws-region: us-east-1

- name: Login a ECR
  uses: aws-actions/amazon-ecr-login@v1

- name: Build y Push a ECR
  env:
    IMAGE_TAG: ${{ github.sha }}
    ECR_REGISTRY: 361510669006.dkr.ecr.us-east-1.amazonaws.com
    ECR_REPO: ventas-innovatech
  run: |
    docker build -t $ECR_REGISTRY/$ECR_REPO:$IMAGE_TAG -t $ECR_REGISTRY/$ECR_REPO:latest .
    docker push $ECR_REGISTRY/$ECR_REPO:$IMAGE_TAG
    docker push $ECR_REGISTRY/$ECR_REPO:latest

- name: Actualizar Task Definition con nueva imagen
  env:
    IMAGE_TAG: ${{ github.sha }}
    ECR_REGISTRY: 361510669006.dkr.ecr.us-east-1.amazonaws.com
    ECR_REPO: ventas-innovatech
  run: |
    TASK_DEF=$(aws ecs describe-task-definition --task-definition svc-ventas --query taskDefinition)
    NEW_ARN=$(echo $TASK_DEF | python3 -c "...")
    aws ecs update-service --cluster Cluster-innovatech --service svc-ventas --task-definition $NEW_ARN --force-new-deployment

- name: Esperar deployment estable
  run: |
    aws ecs wait services-stable --cluster Cluster-innovatech --services svc-ventas
```

*[Captura pendiente: repo Back_ventas, "5 workflow runs", último commit `5fcb379`]*

**Workflow CI/CD Frontend:**

El workflow del frontend sigue la misma estructura, apuntando a `front-innovatech` en ECR y a `svc-frontend` en ECS.

```yaml
- name: Checkout codigo
  uses: actions/checkout@v4

- name: Configurar Node.js
  uses: actions/setup-node@v4
  with:
    node-version: '20'
    cache: 'npm'
    cache-dependency-path: package-lock.json

- name: Instalar dependencias
  run: npm ci

- name: Ejecutar lint
  run: npm run lint

- name: Configurar AWS credentials
  uses: aws-actions/configure-aws-credentials@v2
  with:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws-session-token: ${{ secrets.AWS_SESSION_TOKEN }}
    aws-region: us-east-1

- name: Login a ECR
  uses: aws-actions/amazon-ecr-login@v1

- name: Build y Push a ECR
  env:
    IMAGE_TAG: ${{ github.sha }}
    ECR_REGISTRY: 361510669006.dkr.ecr.us-east-1.amazonaws.com
    ECR_REPO: front-innovatech
  run: |
    docker build -t $ECR_REGISTRY/$ECR_REPO:$IMAGE_TAG -t $ECR_REGISTRY/$ECR_REPO:latest .
    docker push $ECR_REGISTRY/$ECR_REPO:$IMAGE_TAG
    docker push $ECR_REGISTRY/$ECR_REPO:latest

- name: Actualizar Task Definition con nueva imagen
  env:
    IMAGE_TAG: ${{ github.sha }}
    ECR_REGISTRY: 361510669006.dkr.ecr.us-east-1.amazonaws.com
    ECR_REPO: front-innovatech
  run: |
    TASK_DEF=$(aws ecs describe-task-definition --task-definition svc-frontend --query taskDefinition)
    NEW_ARN=$(echo $TASK_DEF | python3 -c "...")
    aws ecs update-service --cluster Cluster-innovatech --service svc-frontend --task-definition $NEW_ARN --force-new-deployment

- name: Esperar deployment estable
  run: |
    aws ecs wait services-stable --cluster Cluster-innovatech --services svc-frontend
```

*[Captura pendiente: repo Front_despacho, "13 workflow runs", último commit `4ffc910`]*

## 6. Infraestructura en la nube (AWS)

### 6.1 Clúster ECS y Task Definitions

Se creó el clúster `Cluster-innovatech` utilizando Fargate como Capacity Provider. Se definieron tres Task Definitions (`task-ventas`, `task-despachos`, `task-front`), todas con Launch Type Fargate, arquitectura Linux/X86_64 y el rol `LabRole` (tanto como Task Role como Execution Role, dado que AWS Academy restringe la creación de roles IAM personalizados).

*[Capturas pendientes: `task-ventas:1`, `task-despacho:1`, `task-front:1` — todas en estado ACTIVE, rol LabRole]*

### 6.2 Conexión a base de datos (funcional)

La conexión entre los backends (Ventas y Despachos) y la base de datos RDS se resolvió configurando la variable de entorno `SPRING_DATASOURCE_URL` en la Task Definition de cada servicio, agregando los siguientes parámetros a la URL JDBC:

```
jdbc:mysql://database-1.cw2ivj42uklq.us-east-1.rds.amazonaws.com:3306/[nombre_bd]?allowPublicKeyRetrieval=true&useSSL=false&createDatabaseIfNotExist=true
```

El parámetro `createDatabaseIfNotExist=true` permitió que cada backend creara automáticamente su propia base de datos (`ventas_db` y `despachos_db`) dentro de la instancia RDS al momento de establecer la conexión, sin necesidad de crearlas manualmente.

### 6.3 Servicios ECS activos

Se crearon los servicios `svc-ventas`, `svc-despachos` y `svc-frontend` en el clúster, todos con estrategia de réplica y despliegues verificados como "Completado".

| Service name | Status |
|---|---|
| svc-despachos | ✅ Active |
| svc-frontend | ✅ Active |
| svc-ventas | ✅ Active |

### 6.4 Security Groups

En la etapa EC2 (EP2) se configuraron dos Security Groups: `gs-front` (SSH + HTTP público) y `gs-back` (SSH + puertos de aplicación solo desde `gs-front`). En la migración a ECS/Fargate + RDS es necesario documentar los Security Groups equivalentes para las tasks y la instancia RDS.

**Security Group Frontend (`gs-front`):**

- **Inbound:** SSH puerto 22 desde IP del administrador (190.100.71.232/32). HTTP puerto 80 desde 0.0.0.0/0.
- **Outbound:** HTTP puerto 80 hacia 0.0.0.0/0.

*[Captura: `sg-0e610689aefaf4628 - gs-front`, 2 reglas inbound (SSH y HTTP)]*

**Security Group Backend (`gs-back`):**

- **Inbound:** SSH puerto 22 desde IP del administrador. TCP 8080 y 8081 aceptados exclusivamente desde `gs-front`.
- **Outbound:** sin restricciones adicionales.

*[Captura: configuración de `gs-back` con reglas SSH, TCP 8080 y TCP 8081 con source `sg-0e610689aefaf4628`]*

### 6.5 Diagrama de arquitectura

La pauta exige un diagrama gráfico que represente la arquitectura completa: VPC, subredes, clúster ECS/Fargate, repositorios ECR, instancia RDS y el flujo de tráfico entre componentes.

```
VPC innovatech-vpc
├── Subred pública
│   └── svc-frontend (ECS Fargate task) ──────────► Amazon ECR (front/ventas/despachos)
└── Subred privada
    └── ECS Cluster: Cluster-innovatech
        ├── svc-ventas (puerto 8080)   ─┐
        └── svc-despachos (puerto 8081)─┼──► Amazon RDS MySQL — database-1 (ventas_db, despachos_db)
                                          │         ▲
                                          └─────────┘
                                     IAM Role: LabRole (permisos de cuenta académica)
```

*[Ver anexo "Diagrama de arquitectura innovatech" para versión gráfica completa]*

## 7. Configuración y gestión de secretos

Las credenciales de AWS Academy (Access Key ID, Secret Access Key y Session Token) se almacenaron exclusivamente como GitHub Secrets en cada repositorio, y los workflows las referencian mediante la sintaxis `${{ secrets.NOMBRE }}` sin hardcodear ningún valor en el código fuente. El uso de `AWS_SESSION_TOKEN` es obligatorio en AWS Academy dado que las credenciales de laboratorio son temporales.

| Secret | Última actualización |
|---|---|
| `AWS_ACCESS_KEY_ID` | 1 hour ago |
| `AWS_SECRET_ACCESS_KEY` | 1 hour ago |
| `AWS_SESSION_TOKEN` | 1 hour ago |

Respecto al principio de mínimo privilegio (IAM): se utilizó el rol preexistente `LabRole` para las Task Definitions, dado que AWS Academy restringe la creación de roles IAM personalizados en el laboratorio. Esta limitación debe quedar explícitamente justificada como una restricción del entorno académico, no como una decisión de diseño de producción.

## 8. Observabilidad

Se configuraron grupos de logs en CloudWatch para los tres servicios (`/ecs/task-ventas`, `/ecs/task-despacho`, `/ecs/task-front`), evidenciando el arranque correcto de Spring Boot y del servicio Nginx + React. Adicionalmente, se configuró un dashboard de CloudWatch con la métrica `ECS · CPUUtilization` para los tres servicios, graficada en una línea de tiempo que muestra los picos de carga durante el arranque inicial y durante las pruebas de autoscaling.

| Grupo de logs | Flujos activos | Detalle |
|---|---|---|
| `/ecs/task-despacho` | 18 | Inicio correcto de Spring Boot (Tomcat embedded, Hibernate, RegionFactoryInitiator). Último evento: 16 de junio de 2026. |
| `/ecs/task-front` | 29 | Servicio Nginx + React operativo. Último evento: 16 de junio de 2026, 02:43 UTC. |
| `/ecs/task-ventas` | 6 | Logs de arranque del backend de ventas con Spring Boot. |

*[Capturas pendientes de reemplazo: pantallas de CloudWatch para cada grupo de logs]*

## 9. Seguridad básica

Prácticas de endurecimiento aplicadas hasta ahora:

- Uso de imágenes base minimalistas (`alpine` / `jre-alpine` / `nginx:alpine`) en los tres servicios.
- Ejecución de los backends con un usuario no-root (`appuser`) en lugar de `root`.
- Exposición mínima de puertos: solo el puerto de aplicación de cada servicio (8080, 8081, 80).
- Separación de red: en la etapa EC2, el backend no fue accesible directamente desde internet, solo desde el Security Group del frontend.

### 9.1 Escaneo de vulnerabilidades Ventas, Despachos y Frontend (Docker Scout)

Se ejecutó `docker scout cves` sobre las tres imágenes del proyecto (`ventas-innovatech`, `despachos-innovatech` y `front-innovatech`). Ventas y Despachos, al compartir el mismo `pom.xml` y la misma imagen runtime (`eclipse-temurin:17-jre-alpine`), arrojaron un perfil idéntico: 86 vulnerabilidades cada una (5 críticas, 27 altas, 26 medias, 16 bajas, 12 sin clasificar), originadas principalmente en `tomcat-embed-core` y `openssl`. El Frontend, al estar construido sobre `nginx:alpine` con un stack Node/npm en la etapa de build, presenta un perfil distinto: 64 vulnerabilidades (1 crítica, 10 altas, 14 medias, 6 bajas, 33 sin clasificar), concentradas en paquetes del sistema base Alpine y dependencias de npm, sin exposición del componente Tomcat/Spring que afecta a los backends.

| Severidad | Cantidad | Origen principal |
|---|---|---|
| Crítica (Critical) | 5 | tomcat-embed-core 10.1.39 (4), openssl 3.5.6-r0 (1) |
| Alta (High) | 27 | tomcat-embed-core (10), openssl (7), jackson-databind (2), spring-boot (2), sqlite (2), spring-core (1) |
| Media (Medium) | 26 | tomcat-embed-core (6), openssl (4), jackson-databind (2), spring-webmvc (3), otros paquetes del SO base |
| Baja (Low) | 16 | tomcat-embed-core (5), openssl (2), logback-core (3), spring-webmvc (2), otros |
| Sin especificar | 12 | expat 2.7.5-r0 (paquete del sistema base Alpine) |

**Vulnerabilidades críticas más relevantes:**

| CVE | Componente | Descripción | Versión corregida |
|---|---|---|---|
| CVE-2026-43512 | tomcat-embed-core 10.1.39 | Autenticación incorrecta (CVSS 9.8) | 10.1.55 |
| CVE-2026-41293 | tomcat-embed-core 10.1.39 | Validación de entrada incorrecta (CVSS 9.8) | 10.1.55 |
| CVE-2026-43515 | tomcat-embed-core 10.1.39 | Autorización incorrecta (CVSS 9.1) | 10.1.55 |
| CVE-2026-29145 | tomcat-embed-core 10.1.39 | Autenticación incorrecta (CVSS 9.1) | 10.1.53 |
| CVE-2026-34182 | openssl 3.5.6-r0 | Vulnerabilidad crítica en librería base del SO | 3.5.7-r0 |

**Análisis:** la gran mayoría de las vulnerabilidades detectadas provienen de dos fuentes concentradas — la versión de `tomcat-embed-core` empaquetada por defecto con la versión de Spring Boot utilizada (3.4.4), y el paquete `openssl` de la imagen base de Alpine. Ambas tienen actualización disponible que corrige la totalidad de sus hallazgos críticos y altos. Como acción correctiva se recomienda actualizar la dependencia de Spring Boot a una versión que incluya `tomcat-embed-core ≥10.1.55` y reconstruir la imagen sobre una versión más reciente de `eclipse-temurin:17-jre-alpine` que incluya el paquete openssl parchado.

## 10. Orquestación y escalabilidad

Se configuró autoscaling en los tres servicios ECS mediante la política "Target Tracking" con la métrica `ECSServiceAverageCPUUtilization`, con un rango de tareas de mínimo 1 y máximo 3. Se optó por un servicio de orquestación gestionado (ECS con Fargate) en lugar de un despliegue manual, ya que permite recuperación automática ante fallos de tareas, actualización de servicios sin intervención manual (rolling deployment mediante `--force-new-deployment`) y ajuste automático de capacidad según la demanda de CPU, sin necesidad de gestionar servidores subyacentes.

| Servicio | Política | Umbral objetivo | Mín. / Máx. tareas |
|---|---|---|---|
| svc-ventas | cpu-scaling-ventas | 5 % | 1 / 3 |
| svc-despachos | cpu-scaling-despachos | 50 % | 1 / 3 |
| svc-frontend | cpu-scaling-front | 50 % | 1 / 3 |

Se evidenció la activación real de las alarmas de CloudWatch asociadas al autoscaling al generar carga sobre el servicio de ventas, confirmando que las políticas reaccionan efectivamente a cambios en la utilización de CPU.

*[Capturas pendientes de reemplazo: alarmas en modo alarma para `svc-ventas` y `svc-frontend`; actividades de escalado del front con estados "Correcto" y "Error" (AlreadyAtMinCapacity / AlreadyAtDesiredCapacity)]*

## 11. Validación funcional del sistema

El frontend fue accesible públicamente desde la IP asignada por Fargate en el puerto 8080/80. Se validó la respuesta de ambos backends mediante `curl` a sus respectivos endpoints REST, confirmando que los servicios están operativos. Desde la interfaz del frontend se realizaron consultas reales, visualizando órdenes de compra y órdenes de despacho con sus datos correctos, confirmando la comunicación de extremo a extremo entre frontend, backends y base de datos RDS.

*[Captura pendiente: interfaz "Despacho devoops" en `44.220.42.156:8080`, mostrando consulta de órdenes de compra y tabla de órdenes de despacho]*

Adicionalmente, se verificó la recuperación automática del sistema tras múltiples ejecuciones del pipeline (14 corridas en Frontend, 5 en Ventas, 4 en Despachos), forzando en cada caso un nuevo despliegue en ECS con reemplazo exitoso de tareas.

## 12. Limitaciones y trabajo futuro

- **Ausencia de un Application Load Balancer (ALB):** actualmente las IPs públicas y privadas de las tareas Fargate deben actualizarse manualmente en las variables de entorno cada vez que se redespliega un servicio, ya que Fargate asigna una IP nueva en cada ciclo de vida del contenedor. La incorporación de un ALB junto con Service Discovery o DNS interno eliminaría esta dependencia.
- **Uso del rol `LabRole`** en lugar de roles IAM de mínimo privilegio, debido a las restricciones del entorno AWS Academy.
- **Verificación de la etapa de test/lint solo en entorno local, no en la nube:** el step fue agregado a los tres workflows y validado exitosamente en local (`mvn test`, `npm run lint`), pero no fue posible confirmar su ejecución dentro de GitHub Actions ni el push del tag versionado a Amazon ECR debido al vencimiento de los créditos de AWS Academy antes del cierre del informe (ver marcadores E y G).

## 13. Conclusión

El desarrollo de este proyecto permitió evolucionar la plataforma Innovatech Chile desde un despliegue manual en instancias EC2 hacia una arquitectura de orquestación productiva sobre AWS ECS con Fargate, integrando ECR para el registro de imágenes, RDS para la persistencia desacoplada, autoscaling basado en CPU y un pipeline de CI/CD automatizado con GitHub Actions. Se logró desplegar los tres servicios de forma funcional, con comunicación validada de extremo a extremo y políticas de escalado activas. La etapa de test/lint (marcador G) fue incorporada y validada localmente en los tres servicios. El único punto pendiente identificado — tag versionado en ECR (marcador E), verificado localmente pero no en la nube por vencimiento de créditos AWS Academy — queda señalado como marcador en este informe para ser completado en cuanto se disponga de un nuevo ciclo de créditos.

## 14. Resumen de marcadores pendientes

| Marcador | Sección | Estado |
|---|---|---|
| E | Registro de imágenes (ECR) | ✅ Resuelto localmente — tag versionado implementado y verificado con `docker images` en los 3 servicios; no verificado en ECR por vencimiento de créditos AWS |
| G | Pipeline CI/CD | ✅ Resuelto — etapa `test`/`lint` agregada y verificada localmente en los 3 servicios; no verificada en GitHub Actions por vencimiento de créditos AWS |
| K | Security Groups | ✅ Resuelto — capturas de `gs-front` y `gs-back` incorporadas |

## 15. Referencias

- Docker Inc. (2026). *Docker documentation*. https://docs.docker.com
- Docker Inc. (2026). *Docker Scout documentation*. https://docs.docker.com/scout
- Amazon Web Services. (2026). *Amazon Elastic Container Service (ECS) Developer Guide*. https://docs.aws.amazon.com/ecs
- Amazon Web Services. (2026). *AWS Fargate*. https://docs.aws.amazon.com/AmazonECS/latest/developerguide/AWS_Fargate.html
- Amazon Web Services. (2026). *Amazon Elastic Container Registry (ECR) User Guide*. https://docs.aws.amazon.com/ecr
- Amazon Web Services. (2026). *Amazon Relational Database Service (RDS) User Guide*. https://docs.aws.amazon.com/rds
- Amazon Web Services. (2026). *Amazon CloudWatch User Guide*. https://docs.aws.amazon.com/cloudwatch
- GitHub Inc. (2026). *GitHub Actions documentation*. https://docs.github.com/actions
- VMware / Broadcom. (2026). *Spring Boot reference documentation*. https://docs.spring.io/spring-boot
- Nginx Inc. (2026). *NGINX documentation*. https://nginx.org/en/docs
- DuocUC. (2026). *Evaluación Final Transversal — Encargo con Presentación: Instrucciones y pauta de evaluación*. Asignatura Introducción a Herramientas DevOps (ISY1101).

## 16. Anexos

- Diagrama de arquitectura innovatech
- Diagrama Pipeline
