# Frontend - Innovatech Chile

Aplicación web desarrollada con React y Vite para la gestión de órdenes de compra y despacho de Innovatech Chile. Desplegada en AWS ECS con Fargate.

## Tecnologías

- React 18
- Vite
- Tailwind CSS
- Axios
- React Router DOM
- Docker
- Nginx

## Arquitectura

El frontend corre como contenedor Docker en AWS ECS (Fargate), con imagen almacenada en Amazon ECR. Se comunica con los backends de Ventas y Despachos a través de variables de entorno configuradas en la Task Definition.

## Variables de entorno

| Variable | Descripción |
|----------|-------------|
| `VENTAS_BACKEND_HOST` | IP o DNS del backend de ventas |
| `DESPACHOS_BACKEND_HOST` | IP o DNS del backend de despachos |

## Pipeline CI/CD

Cada push a la rama `deploy` activa el workflow de GitHub Actions que:
1. Construye la imagen Docker
2. Hace push a Amazon ECR
3. Fuerza un nuevo despliegue en ECS

## Cómo correr localmente

```bash
npm install
npm run dev
```

Acceder en `http://localhost:5173`
