# Despacho Frontend — Rama deploy-eks

Frontend en JavaScript / React para la gestión de despachos, equipado con un pipeline de integración y despliegue continuo (CI/CD) automatizado hacia AWS EKS (Elastic Kubernetes Service) a través de GitHub Actions.

## 1. Arquitectura del Frontend

El proyecto sigue una arquitectura modular basada en componentes, garantizando la separación de responsabilidades y reutilización de código:

**Framework y Dependencias:**
- React 18.x con JavaScript (ES6+)
- Bundler: Webpack / Vite (según configuración)
- Gestor de Dependencias: npm / yarn

**Estructura de Carpetas:**
- `src/components`: Componentes reutilizables de la interfaz de usuario
- `src/pages`: Páginas principales de la aplicación
- `src/services`: Servicios para consumir APIs REST
- `src/hooks`: Custom hooks para lógica reutilizable
- `src/utils`: Funciones utilitarias y helpers
- `src/styles`: Estilos globales y módulos CSS
- `public`: Archivos estáticos (imágenes, favicon, etc.)

**Características Principales:**
- Interfaz responsiva y moderna para la gestión de despachos
- Integración con backend mediante APIs REST
- Gestión de estado (Context API, Redux o similar según implementación)
- Autenticación y autorización cliente-side
- Validación de formularios
- Manejo de errores y notificaciones

## 2. Configuración de Variables de Entorno

La aplicación requiere las siguientes variables de entorno para conectarse correctamente con el backend y servicios externos:

- `REACT_APP_API_BASE_URL`: URL base del backend API (ej: `http://localhost:8080/api/v1`)
- `REACT_APP_ENVIRONMENT`: Ambiente de ejecución (development, staging, production)
- `REACT_APP_API_TIMEOUT`: Timeout en milisegundos para las peticiones HTTP
- Otras variables específicas según la configuración de servicios externos

**Nota:** Las variables de entorno deben estar prefijadas con `REACT_APP_` para ser accesibles en tiempo de ejecución.

## 3. Pipeline CI/CD (deploy-eks.yml)

El flujo de despliegue automatizado está definido en `.github/workflows/deploy-eks.yml` y se dispara automáticamente al hacer un push a la rama `deploy-eks`, o bien de forma manual (`workflow_dispatch`).

El workflow se compone de dos trabajos (jobs) principales:

**Job 1: Build and Push Image**
- Descarga del código fuente mediante Checkout
- Configuración de credenciales de AWS
- Autenticación en Amazon ECR
- Instalación de dependencias (npm install)
- Build de la aplicación React (npm run build)
- Construcción (Build) de la imagen Docker
- Publicación (Push) de la imagen utilizando dos tags de forma simultánea: el hash del commit (`${github.sha}`) y `latest`

**Job 2: Deploy to EKS**
- Descarga del código fuente (Checkout de los manifiestos de Kubernetes)
- Configuración de credenciales de AWS
- Instalación y configuración de kubectl
- Actualización del contexto de Kubernetes con `aws eks update-kubeconfig` apuntando a `EKS_CLUSTER_NAME`
- Creación o actualización del secreto de Kubernetes (frontend-env-secret) inyectando las siguientes variables:
  - `REACT_APP_API_BASE_URL` ← API_BASE_URL
  - `REACT_APP_ENVIRONMENT` ← ENVIRONMENT
  - Otras variables de configuración según sea necesario
- Aplicación de manifiestos: `kubectl apply -f k8s/ -n <namespace>`
- Actualización de la imagen del Deployment (`kubectl set image`) utilizando la nueva imagen publicada en ECR
- Verificación del estado del despliegue (rollout status) y listado de Pods/Services

## 4. Recursos de Kubernetes

Existentes en el directorio `k8s/`:
- `deployment.yaml`: Configuración del pod del frontend de la aplicación
- `service.yaml`: Expone el frontend internamente dentro del clúster (ClusterIP)
- `hpa.yaml`: Escalado horizontal automático (Horizontal Pod Autoscaler) para el frontend
- `ingress.yaml` (recomendado): Configura el acceso externo a la aplicación

**Faltantes o recomendables para entornos de producción**

⚠️ Notas de optimización arquitectónica:

- **Manifiesto del Namespace**: Actualmente se asume preexistente en el clúster. Se recomienda incluir su declaración explícita
- **Acceso Externo**: Implementar un recurso Ingress con configuración de dominio personalizado y certificados TLS/SSL
- **Optimización de Imágenes**: Utilizar multi-stage builds en Docker para reducir el tamaño final de la imagen (separar node build del contenedor nginx de producción)
- **Caché y CDN**: Configurar directivas de caché HTTP en el servidor web (nginx/apache) y considerar integración con CloudFront
- **Políticas y Probes**: Se sugiere robustecer los parámetros de Liveness/Readiness Probes, añadir NetworkPolicy para aislar el tráfico y configurar un PodDisruptionBudget
- **Seguridad**: Implementar Content Security Policy (CSP), habilitar CORS de forma restrictiva y validar todas las entradas del usuario

## 5. Rutas y Funcionalidades Expuestas

**Base Path:** `/`

| Ruta | Descripción |
|------|-------------|
| `/` | Página de inicio de la aplicación |
| `/despachos` | Listado de despachos |
| `/despachos/nuevo` | Formulario para crear un nuevo despacho |
| `/despachos/:id` | Detalle y edición de un despacho específico |
| `/login` | Página de autenticación |
| `/dashboard` | Panel de control y analytics |
| `/configuracion` | Configuración de usuario y aplicación |

**Documentación Adicional**: Consulta la documentación de Swagger del backend en `/swagger-ui.html` para conocer los endpoints disponibles de la API REST.

## 6. Clonado y Ejecución Local

### Requisitos Previos
- Node.js 16.x o superior (recomendado 18.x)
- npm 7.x o superior (o yarn 3.x)
- Backend API ejecutándose en local o accesible remotamente

### Pasos para iniciar la aplicación

**Clonar el repositorio:**
```bash
git clone <repo-url>
cd front-despacho-devops-ev2
git checkout deploy-eks
```

**Instalar las dependencias:**
```bash
npm install
```

**Configurar las variables de entorno (Crear archivo `.env`):**
```bash
REACT_APP_API_BASE_URL=http://localhost:8080/api/v1
REACT_APP_ENVIRONMENT=development
REACT_APP_API_TIMEOUT=5000
```

**Ejecutar la aplicación en modo desarrollo:**
```bash
npm start
```

La aplicación estará disponible en `http://localhost:3000`

**Ejecutar las pruebas automatizadas:**
```bash
npm test
```

**Generar la build de producción:**
```bash
npm run build
```

### Construcción y ejecución local con Docker

Para validar el empaquetado de la imagen de forma local, ejecuta:

```bash
# Construir la imagen local
docker build -t despacho-frontend:local .

# Ejecutar el contenedor mapeando el puerto 3000
docker run --rm -p 3000:80 \
  -e REACT_APP_API_BASE_URL=http://host.docker.internal:8080/api/v1 \
  -e REACT_APP_ENVIRONMENT=development \
  despacho-frontend:local
```

La aplicación estará disponible en `http://localhost:3000`

## 7. Despliegue en EKS (Resumen Operacional)

Para que el flujo de GitHub Actions se ejecute de manera correcta, es mandatorio configurar los siguientes secretos en el repositorio (Settings > Secrets and variables > Actions):

**Credenciales AWS:**
- `AWS_ACCOUNT_ID`
- `AWS_REGION`
- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`
- `AWS_SESSION_TOKEN` (si aplica)
- `AWS_ECR_REPOSITORY`

**Variables de EKS:**
- `EKS_CLUSTER_NAME`
- `EKS_NAMESPACE`
- `K8S_DEPLOYMENT_NAME`
- `K8S_CONTAINER_NAME`

**Variables de Configuración de la Aplicación:**
- `API_BASE_URL`: URL del backend en el ambiente de EKS
- `ENVIRONMENT`: Ambiente (staging, production, etc.)

Una vez configurados los secretos, basta con realizar un push a la rama `deploy-eks` para iniciar el despliegue automático. Puedes monitorear el progreso en la pestaña **Actions** de GitHub.

## 8. Notas Importantes

- **Imagen Placeholder**: El archivo `deployment.yaml` original define una imagen ligera temporal (nginx:alpine). El workflow se encarga de sobreescribir esta propiedad dinámicamente con la imagen correcta construida en ECR mediante el comando `kubectl set image`
- **Variables de Entorno**: Los manifiestos incluidos en `k8s/` consumen de forma mandatoria un secret de Kubernetes llamado `frontend-env-secret` que contiene las variables de configuración de la aplicación
- **Dockerfile**: Asegúrate de utilizar un multi-stage build para separar la fase de construcción (node) de la fase de distribución (nginx), optimizando así el tamaño final de la imagen
- **Flujos Alternativos**: El repositorio cuenta con otro archivo de workflow (`.github/workflows/main.yml`) diseñado exclusivamente para despliegues orientados a instancias AWS EC2 tradicionales a través de AWS Systems Manager (SSM). No debe confundirse con la arquitectura EKS de esta rama

## 9. Estructura de Archivos Clave

```
front-despacho-devops-ev2/
├── src/
│   ├── components/        # Componentes React reutilizables
│   ├── pages/            # Páginas principales
│   ├── services/         # Servicios de API
│   ├── hooks/            # Custom hooks
│   ├── utils/            # Funciones auxiliares
│   ├── styles/           # Estilos CSS
│   └── App.js            # Componente raíz
├── public/               # Archivos estáticos
├── k8s/                  # Manifiestos de Kubernetes
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   └── hpa.yaml
├── .github/workflows/    # Pipelines de CI/CD
│   ├── deploy-eks.yml
│   └── main.yml
├── Dockerfile            # Imagen Docker multi-stage
├── package.json          # Dependencias y scripts
├── .env.example          # Template de variables de entorno
└── README.md             # Este archivo
```

---

**Última actualización:** Junio 2026
