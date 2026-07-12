# Reflexión Individual — Evaluación Final Transversal

**Estudiante:** Carlos Muñoz Leyton
**Asignatura:** Introducción a Herramientas DevOps (ISY1101)
**Proyecto:** Innovatech Chile — CI/CD end-to-end sobre AWS ECS Fargate
**Equipo:** Carlos Muñoz Leyton, Bastián Brisso, Carlos Camero

---

## 1. Cómo se desarrolló el proyecto

El proyecto lo trabajamos en tres etapas a lo largo del semestre, y cada una fue construyendo sobre la anterior:

En la **EP2** partimos con lo más básico: contenedorizar los tres servicios (frontend en React, y los backends de Ventas y Despachos en Spring Boot) con Dockerfiles multietapa, y desplegarlos manualmente sobre instancias EC2 usando Docker. En esa etapa configuramos Docker Compose para poder levantar todo en local antes de subirlo a la nube, con una red interna y volúmenes para las bases de datos MySQL.

En la **EP3** dimos el salto a una arquitectura de orquestación real: migramos de EC2 manual a **Amazon ECS con Fargate**, incorporamos **Amazon ECR** para tener un registro privado de imágenes, y separamos la base de datos a una instancia de **Amazon RDS** compartida (con bases de datos lógicas separadas para cada servicio). También armamos el pipeline de CI/CD con GitHub Actions para automatizar el build, push y deploy.

En la **EFT** cerramos los puntos que quedaban pendientes de la pauta: agregamos una etapa de test explícita en el pipeline (`mvn test` para los backends, `npm run lint` para el frontend), implementamos tags versionados en las imágenes de ECR (además de `latest`), documentamos los Security Groups que faltaban, y corrimos un escaneo de vulnerabilidades con Docker Scout sobre las tres imágenes.

Un problema real que tuvimos hacia el final fue que **se nos vencieron los créditos de AWS Academy** antes de poder verificar en la nube los últimos cambios (el tag versionado en ECR y la ejecución del step de test dentro de GitHub Actions). Por eso, para esas dos partes, la evidencia que presentamos es de ejecución **local** (`docker build`, `docker tag`, `docker images`, `mvn test`), dejando documentado en el informe que el mecanismo está implementado y funciona, aunque no se pudo confirmar en el entorno cloud por esa restricción externa.

## 2. Qué decisiones técnicas se tomaron

- **AWS como proveedor cloud**, porque era el entorno provisto por Duoc a través de AWS Academy Learner Lab, y porque ECS + Fargate nos permitía desplegar contenedores sin tener que administrar servidores directamente (a diferencia de EC2, donde nosotros éramos responsables del sistema operativo, parches, etc.).
- **Fargate por sobre EC2 como Capacity Provider**, porque no queríamos gestionar instancias subyacentes — Fargate se encarga de la infraestructura y nosotros solo definimos la Task Definition.
- **Imágenes base Alpine** (`eclipse-temurin:17-jre-alpine`, `nginx:alpine`, `node:20-alpine`) para mantener las imágenes lo más livianas posible y reducir la superficie de ataque.
- **Dockerfiles multietapa**, separando la etapa de build (con Maven o npm) de la etapa de runtime, para que la imagen final no cargue con herramientas de compilación que no necesita en producción.
- **Usuario no-root (`appuser`)** dentro de los contenedores de los backends, siguiendo el principio de menor privilegio a nivel de contenedor.
- **RDS compartido con bases de datos lógicas separadas** (`ventas_db`, `despachos_db`) en vez de una instancia RDS por servicio, para simplificar la gestión dado el contexto de laboratorio académico con recursos limitados.
- **Rol `LabRole`** para las Task Definitions en lugar de roles IAM personalizados de mínimo privilegio — esto no fue una decisión de diseño ideal, sino una restricción del entorno de AWS Academy, que no permite crear roles IAM propios. Lo dejamos explícitamente documentado como limitación, no como buena práctica de producción.
- **Autoscaling por Target Tracking** sobre CPU, con un rango de 1 a 3 tareas por servicio, para no sobredimensionar recursos en un entorno de pruebas.
- **Tags versionados en ECR** (SHA corto del commit, además de `latest`) para tener trazabilidad real de qué commit generó cada imagen desplegada.

## 3. Organización del equipo y forma de trabajo

Nos organizamos dividiendo el proyecto por repositorio/servicio (`Front_despacho`, `Back_ventas`, `Back_despacho`, bajo la cuenta `CarlosMUL` en GitHub), cada uno con su propio workflow de CI/CD independiente, lo que nos permitió trabajar en paralelo. Las decisiones de arquitectura las definimos en conjunto, y los problemas que bloqueaban a todo el equipo (como el vencimiento de credenciales de AWS Academy) los resolvíamos coordinándonos en reuniones presenciales o, cuando no era posible, por Discord.

## 4. Cómo funciona la solución implementada

El sistema tiene tres componentes que se comunican por HTTP a través de variables de entorno:

- El **frontend** (React + Vite, servido por Nginx) corre en la subred pública y actúa como punto de entrada; consume los endpoints REST de ambos backends.
- El **backend de Ventas** (Spring Boot, puerto 8080) expone `/api/v1/ventas` y gestiona las órdenes de compra.
- El **backend de Despachos** (Spring Boot, puerto 8081) expone `/api/v1/despachos` y gestiona el estado de las entregas.
- Ambos backends se conectan a la misma instancia de **Amazon RDS MySQL**, cada uno a su propia base de datos lógica, usando el parámetro `createDatabaseIfNotExist=true` en la URL JDBC para que la base se cree automáticamente al primer intento de conexión.

Todo el ciclo de despliegue está automatizado: cuando se hace push a la rama `deploy` en cualquiera de los tres repositorios, GitHub Actions ejecuta el workflow correspondiente, que hace login a ECR, corre las pruebas, construye la imagen Docker, la etiqueta (con `latest` y con el SHA del commit), la sube a Amazon ECR, y finalmente fuerza un nuevo despliegue del servicio en ECS (`aws ecs update-service --force-new-deployment`), lo que hace que ECS reemplace las tareas corriendo por la nueva imagen sin downtime manual.

Cada servicio tiene su propia política de autoscaling basada en el uso de CPU, y sus logs quedan centralizados en CloudWatch, lo que nos permitió verificar que las alarmas de escalado se activan correctamente cuando se genera carga.

## 5. En qué partes participé directamente

Trabajé principalmente en la parte de infraestructura en la nube: configuré los Dockerfiles multietapa de los tres servicios y el `docker-compose.yml` para el entorno local, creé los repositorios en Amazon ECR y dejé andando el flujo de build y push de las imágenes, y configuré las Task Definitions y los servicios en ECS dentro del clúster `Cluster-innovatech`. También resolví la conexión entre los backends y RDS (ajustando la URL JDBC para que las bases de datos se crearan automáticamente) y documenté los Security Groups que faltaban del informe de EP3.

Para la EFT en particular, me encargué de cerrar los puntos pendientes de la pauta: agregué la etapa de test/lint en los tres workflows, implementé los tags versionados en ECR, y corrí el escaneo de vulnerabilidades con Docker Scout. Cuando se nos vencieron los créditos de AWS Academy, verifiqué localmente esos dos últimos mecanismos y documenté esa limitación de forma transparente en vez de simular una evidencia que no correspondía.

## 6. Cuál fue mi principal aporte dentro del equipo

Mi aporte estuvo en tomar el proyecto que ya funcionaba desde EP3 y llevarlo al nivel que pedía la EFT: cerrar las brechas de la pauta y ser honesto en la documentación sobre qué se pudo verificar en la nube y qué solo se pudo probar localmente por la restricción de créditos.

## 7. Qué aspectos técnicos domino y puedo defender

Puedo explicar con confianza el flujo completo de CI/CD con GitHub Actions (qué hace cada step y cómo se conecta con ECR y ECS), la diferencia entre ECS con Fargate y un despliegue manual en EC2, y cómo funciona el build multietapa de Docker. También domino la arquitectura de red (por qué el frontend es público y los backends privados, y cómo funcionan los Security Groups `gs-front`/`gs-back`), la conexión de un servicio ECS con RDS, el autoscaling por Target Tracking, y los resultados del escaneo de Docker Scout con sus acciones correctivas.

Puedo justificar además por qué usamos el rol `LabRole` en vez de roles IAM de mínimo privilegio — una limitación del entorno académico, no una decisión de producción — y cómo se resolvería en un entorno real.
