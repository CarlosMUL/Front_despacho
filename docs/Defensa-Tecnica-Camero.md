# Defensa Técnica Individual — Evaluación Final Transversal

**Estudiante:** Carlos Camero
**Asignatura:** Introducción a Herramientas DevOps (ISY1101)
**Proyecto:** Innovatech Chile — CI/CD end-to-end sobre AWS ECS Fargate
**Equipo:** Carlos Muñoz Leyton, Bastián Brisso, Carlos Camero

---

## 1. Cómo se desarrolló el proyecto

El proyecto lo trabajamos en tres etapas a lo largo del semestre, y cada una fue construyendo sobre la anterior:

En la **EP2** partimos con lo más básico: contenedorizar los tres servicios (frontend en React, y los backends de Ventas y Despachos en Spring Boot) con Dockerfiles multietapa, y desplegarlos manualmente sobre instancias EC2 usando Docker. En esa etapa configuramos Docker Compose para poder levantar todo en local antes de subirlo a la nube, con una red interna y volúmenes para las bases de datos MySQL.

En la **EP3** dimos el salto a una arquitectura de orquestación real: migramos de EC2 manual a **Amazon ECS con Fargate**, incorporamos **Amazon ECR** para tener un registro privado de imágenes, y separamos la base de datos a una instancia de **Amazon RDS** compartida (con bases de datos lógicas separadas para cada servicio). También armamos el pipeline de CI/CD con GitHub Actions para automatizar el build, push y deploy.

En la **EFT** cerramos los puntos que quedaban pendientes de la pauta: agregamos una etapa de test explícita en el pipeline (`mvn test` para los backends, `npm run lint` para el frontend), implementamos tags versionados en las imágenes de ECR (además de `latest`), documentamos los Security Groups que faltaban, y corrimos un escaneo de vulnerabilidades con Docker Scout sobre las tres imágenes.

Un problema real que tuvimos hacia el final fue que se nos vencieron los créditos de AWS Academy antes de poder verificar en la nube los últimos cambios (el tag versionado en ECR y la ejecución del step de test dentro de GitHub Actions). Por eso, para esas dos partes, la evidencia que presentamos es de ejecución local, dejando documentado que el mecanismo está implementado y funciona, aunque no se pudo confirmar en el entorno cloud por esa restricción externa.

## 2. Qué decisiones técnicas se tomaron

- **AWS como proveedor cloud**, provisto por Duoc a través de AWS Academy Learner Lab, y ECS + Fargate porque nos permitía desplegar contenedores sin administrar servidores directamente.
- **Fargate por sobre EC2 como Capacity Provider**, para no gestionar instancias subyacentes — Fargate se encarga de la infraestructura y nosotros solo definimos la Task Definition.
- **Imágenes base Alpine** (`eclipse-temurin:17-jre-alpine`, `nginx:alpine`, `node:20-alpine`) para mantener las imágenes livianas y reducir la superficie de ataque.
- **Dockerfiles multietapa**, separando build (Maven/npm) de runtime, para que la imagen final no cargue herramientas de compilación innecesarias.
- **Usuario no-root (`appuser`)** dentro de los contenedores de los backends.
- **RDS compartido con bases de datos lógicas separadas** (`ventas_db`, `despachos_db`) en vez de una instancia por servicio, dado el contexto académico con recursos limitados.
- **Rol `LabRole`** en las Task Definitions en lugar de roles IAM de mínimo privilegio — restricción del entorno AWS Academy, no una decisión de diseño de producción.
- **Autoscaling por Target Tracking** sobre CPU, con rango de 1 a 3 tareas por servicio.
- **Tags versionados en ECR** (SHA corto del commit, además de `latest`) para trazabilidad.

## 3. Organización del equipo y forma de trabajo

Nos organizamos dividiendo el proyecto por repositorio/servicio (`Front_despacho`, `Back_ventas`, `Back_despacho`, bajo la cuenta `CarlosMUL` en GitHub), cada uno con su propio workflow de CI/CD independiente, lo que nos permitió trabajar en paralelo. Las decisiones de arquitectura (elección de AWS, estructura de red, estrategia de despliegue) las definimos en conjunto, y los problemas que bloqueaban a todo el equipo (como el vencimiento de credenciales de AWS Academy) los resolvimos coordinándonos.

Estuve más a cargo del repositorio `Back_despacho`, principalmente revisando la configuración de secretos y variables de entorno de ese servicio. Nos coordinábamos como equipo a través de reuniones presenciales, y cuando no era posible juntarnos, seguíamos coordinándonos por Discord.

## 4. Cómo funciona la solución implementada

El sistema tiene tres componentes que se comunican por HTTP a través de variables de entorno:

- El **frontend** (React + Vite, servido por Nginx) corre en la subred pública y actúa como punto de entrada; consume los endpoints REST de ambos backends.
- El **backend de Ventas** (Spring Boot, puerto 8080) expone `/api/v1/ventas` y gestiona las órdenes de compra.
- El **backend de Despachos** (Spring Boot, puerto 8081) expone `/api/v1/despachos` y gestiona el estado de las entregas.
- Ambos backends se conectan a la misma instancia de **Amazon RDS MySQL**, cada uno a su propia base de datos lógica.

Todo el ciclo de despliegue está automatizado: al hacer push a la rama `deploy`, GitHub Actions corre las pruebas, construye la imagen, la etiqueta y sube a ECR, y fuerza el redeploy en ECS. Cada servicio tiene autoscaling basado en CPU, y sus logs quedan centralizados en CloudWatch.

## 5. En qué partes participé directamente

Mi participación estuvo enfocada en apoyar la revisión del autoscaling y la configuración de los servicios en ECS, principalmente dentro del repositorio `Back_despacho`, donde también revisé la configuración de secretos y variables de entorno. Ayudé a revisar la configuración de las políticas de Target Tracking (umbrales de CPU, rango de tareas mínimo/máximo) para los tres servicios, y leí en profundidad la sección del informe correspondiente a orquestación y escalabilidad para entender cómo se justificaba la elección de ECS con Fargate frente a un despliegue manual.

También trabajé directamente en la consola de AWS configurando tasks y ajustando services durante los despliegues.

No participé directamente en la configuración de los Dockerfiles, el pipeline de CI/CD, ni en la infraestructura de red (VPC, Security Groups) — esas partes estuvieron principalmente a cargo de Carlos Muñoz.

## 6. Cuál fue mi principal aporte dentro del equipo

Mi aporte estuvo en revisar y entender en detalle el mecanismo de autoscaling, lo que me permitió aportar una segunda mirada sobre si los umbrales configurados (5% para Ventas, 50% para Despachos y Frontend) tenían sentido para el comportamiento esperado del sistema, además de apoyar directamente en la configuración de tasks y servicios en ECS durante los despliegues.

## 7. Qué aspectos técnicos domino y puedo defender

Puedo explicar con confianza cómo funciona el autoscaling con ECS Fargate: la política de Target Tracking basada en la métrica `ECSServiceAverageCPUUtilization`, cómo se define un umbral objetivo por servicio, el rango de tareas configurado (mínimo 1, máximo 3), y por qué se usa este mecanismo en vez de un ajuste manual de capacidad. También puedo explicar por qué se optó por ECS con Fargate en lugar de un despliegue manual sobre EC2: recuperación automática ante fallos de tareas, rolling deployment sin intervención manual, y no tener que gestionar servidores subyacentes.

Al haber configurado tasks y ajustado services directamente en la consola de AWS, puedo explicar también el flujo práctico de cómo se actualiza un servicio en ECS y qué relación tiene con la Task Definition asociada.

Para las partes de contenedorización, pipeline CI/CD y seguridad de red, mi comprensión es más general (a nivel de qué hace cada pieza), no de haberlas configurado yo mismo.
