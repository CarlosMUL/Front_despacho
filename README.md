# Frontend - Despacho Dashboard

Aplicación web desarrollada con React y Vite para la gestión de órdenes de compra y despacho del sistema ITPCargo.

## Tecnologías

- React 18
- Vite 5
- Tailwind CSS
- Axios
- React Router DOM
- Docker
- Nginx

## Funcionalidades

- Consultar órdenes de compra (ventas)
- Generar órdenes de despacho desde una venta
- Revisar y gestionar órdenes de despacho

## Correr localmente con Docker

1. Clonar el repositorio
2. Ejecutar:

```bash
docker-compose up -d
```

El servicio estará disponible en `http://localhost:80`

## Configuración del Backend

El frontend se comunica con los backends a través del nginx. La IP privada del backend se configura en el archivo `nginx.conf`:

```nginx
location /api/v1/ventas {
    proxy_pass http://IP_PRIVADA_BACKEND:8080;
}

location /api/v1/despachos {
    proxy_pass http://IP_PRIVADA_BACKEND:8081;
}
```

Se usa la IP privada del EC2 backend para la comunicación dentro del mismo VPC de AWS.

## Pipeline CI/CD

El pipeline se activa automáticamente al hacer push sobre la rama `deploy`.

Pasos del pipeline:
1. Construcción de la imagen Docker (multi-stage build)
2. Publicación de la imagen en Docker Hub
3. Despliegue automático en la instancia EC2

### Secrets requeridos en GitHub

| Secret | Descripción |
|--------|-------------|
| DOCKERHUB_USERNAME | Usuario de Docker Hub |
| DOCKERHUB_TOKEN | Token de acceso Docker Hub |
| EC2_FRONTEND_HOST | IP pública del servidor EC2 frontend |
| EC2_USER | Usuario SSH del EC2 |
| EC2_SSH_KEY | Clave privada SSH |

## Dockerfile

El Dockerfile utiliza multi-stage build:
- **Stage 1 (builder):** Instala dependencias y compila la aplicación React con Vite
- **Stage 2 (runtime):** Sirve los archivos estáticos con Nginx, con usuario sin privilegios root

## Arquitectura

```
Internet
    ↓
EC2 Frontend (puerto 80) - nginx
    ↓
EC2 Backend (puertos 8080/8081) - Spring Boot
    ↓
MySQL (contenedor Docker)
```

Solo el frontend es accesible desde internet. El backend solo acepta tráfico proveniente del security group del frontend.
