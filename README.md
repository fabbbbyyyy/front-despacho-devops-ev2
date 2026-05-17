# Front Despacho DevOps EV2

Aplicación frontend en **React + Vite** para gestionar el flujo de despachos:

1. Consultar órdenes de compra disponibles.
2. Generar una orden de despacho desde una compra.
3. Revisar despachos creados.
4. Modificar intentos de entrega y cerrar despacho.

## Tabla de contenido

- [Tecnologías](#tecnologías)
- [Arquitectura y estructura](#arquitectura-y-estructura)
- [Flujo funcional](#flujo-funcional)
- [Integración con APIs](#integración-con-apis)
- [Requisitos](#requisitos)
- [Instalación y ejecución local](#instalación-y-ejecución-local)
- [Build de producción](#build-de-producción)
- [Ejecución con Docker](#ejecución-con-docker)
- [Scripts disponibles](#scripts-disponibles)
- [Datos y comportamiento esperado](#datos-y-comportamiento-esperado)
- [Validación y troubleshooting](#validación-y-troubleshooting)

## Tecnologías

- React 18
- Vite 5
- React Router DOM 6
- Axios
- React Hook Form
- SweetAlert2
- Tailwind CSS
- ESLint
- Nginx (en despliegue Docker)

## Arquitectura y estructura

```text
src/
  main.jsx                     # Punto de entrada
  Routes/AppRoutes.jsx         # Router principal (ruta /)
  componentes/
    CrudAdmin.jsx              # Layout principal de dashboard
    CrudAdmin/
      PruebaCards.jsx          # Selector de vistas (compras/despachos)
      TableCompras.jsx         # Tabla de compras + generar despacho
      TableDespachos.jsx       # Tabla de despachos + cierre de despacho
      FormDespacho.jsx         # Formulario de creación de despacho
      FormCierreDespacho.jsx   # Formulario de edición/cierre
      Modal.jsx                # Modal reutilizable
      CardComponent.jsx        # Tarjeta reutilizable
    Layouts/
      Navbar.jsx
      Reviews.jsx
      Footer.jsx
```

La aplicación expone una sola ruta (`/`) que renderiza el dashboard de administración logística.

## Flujo funcional

### 1) Consulta de órdenes de compra

- Al abrir **“Consultar Órdenes de compra”**, se muestra `TableCompras`.
- Se hace `GET /api/v1/ventas`.
- Solo se visualizan compras con `despachoGenerado = false`.

### 2) Generación de despacho

Desde la tabla de compras, botón **“Generar Despacho”**:

- Abre modal con `FormDespacho`.
- Envía:
  - `PUT /api/v1/ventas/{idVenta}` para marcar la compra como procesada (`despachoGenerado: true`).
  - `POST /api/v1/despachos` para crear el despacho con fecha, patente, compra asociada, dirección y valor.
- Muestra confirmación visual con SweetAlert.

### 3) Revisión de despachos

- Al abrir **“Revisar Órdenes de despacho”**, se muestra `TableDespachos`.
- Se hace `GET /api/v1/despachos`.
- Se listan estado de entrega e intentos.

### 4) Cierre/modificación de despacho

Desde la tabla de despachos, botón **“Cerrar despacho”**:

- Abre modal con `FormCierreDespacho`.
- Permite cambiar intentos y estado de cierre.
- Envía `PUT /api/v1/despachos/{idDespacho}`.
- Muestra confirmación visual.

## Integración con APIs

### Desarrollo local (Vite)

`vite.config.js` define proxy para `/api` hacia:

`https://qic534o8o0.execute-api.us-east-1.amazonaws.com`

Con reescritura:

- `/api/v1/ventas` → `/v1/ventas`
- `/api/v1/despachos` → `/v1/despachos`

### Producción (Nginx en Docker)

`nginx.conf` enruta:

- `/api/v1/ventas` hacia `backend_ventas` (`10.0.130.47:8080`)
- `/api/v1/despachos` hacia `backend_despachos` (`10.0.141.199:8081`)

> Nota: esas IP son internas del entorno de despliegue y deben ajustarse según infraestructura.

## Requisitos

- Node.js 20+
- npm 9+

Opcional para contenedor:

- Docker

## Instalación y ejecución local

1. Instalar dependencias:

```bash
npm ci
```

2. Levantar en desarrollo:

```bash
npm run dev
```

3. Abrir en navegador:

`http://localhost:5173`

## Build de producción

```bash
npm run build
```

El resultado queda en `dist/`.

Para previsualizar build local:

```bash
npm run preview
```

## Ejecución con Docker

Construir imagen:

```bash
docker build -t front-despacho .
```

Ejecutar contenedor:

```bash
docker run --rm -p 8080:80 front-despacho
```

Abrir:

`http://localhost:8080`

## Scripts disponibles

- `npm run dev`: servidor de desarrollo con HMR.
- `npm run build`: build optimizado para producción.
- `npm run preview`: sirve build localmente.
- `npm run lint`: validación ESLint.

## Datos y comportamiento esperado

- El frontend espera que los servicios de compras y despachos estén disponibles y respondan en JSON.
- Si la API no responde o hay error de red, se registra error en consola (`console.error`).
- Existe un `db.json` de referencia con ventas de ejemplo, útil como dataset de apoyo.

## Validación y troubleshooting

- Si falla `npm run lint`, revisa reglas de ESLint y props validation.
- Si falla la carga de tablas, valida:
  - conectividad con los backends,
  - rutas (`/api/v1/ventas`, `/api/v1/despachos`),
  - configuración de proxy en desarrollo o Nginx en producción.
- Si la UI carga pero no hay datos, revisa respuesta real de API en DevTools (Network).
