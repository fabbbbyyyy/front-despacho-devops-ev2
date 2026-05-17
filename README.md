# Flujo CI/CD

Este repositorio usa **GitHub Actions** con el workflow:

- `.github/workflows/main.yml`

## Disparador (Trigger)

El pipeline se ejecuta en:

- `push` a la rama `main`

## Resumen del pipeline

El flujo tiene 2 jobs secuenciales:

1. `build-and-push`
2. `deploy-to-ec2` (depende de `build-and-push`)

---

## Job 1: build-and-push

Objetivo: construir imagen Docker del frontend y publicarla en Amazon ECR.

Pasos:

1. Checkout del código (`actions/checkout@v4`).
2. Configuración de credenciales AWS (`aws-actions/configure-aws-credentials@v4`).
3. Login en ECR (`aws-actions/amazon-ecr-login@v2`).
4. Build de imagen con 2 tags:
   - `${{ github.sha }}`
   - `latest`
5. Push de ambos tags al repositorio ECR.

Variables usadas:

- `REGISTRY_URL = ${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.${{ secrets.AWS_REGION }}.amazonaws.com`
- `ECR_REPOSITORY = ${{ secrets.AWS_ECR_REPOSITORY }}`
- `IMAGE_TAG = ${{ github.sha }}`

---

## Job 2: deploy-to-ec2

Objetivo: desplegar la imagen `latest` en una instancia EC2 por AWS SSM.

Pasos:

1. Configuración de credenciales AWS.
2. Ejecución de `aws ssm send-command` sobre `${{ secrets.EC2_INSTANCE_ID }}`.
3. Comandos remotos ejecutados en EC2:
   - crear directorio de trabajo,
   - login de Docker contra ECR,
   - pull de imagen `latest`,
   - stop/remove de contenedor anterior (`react-frontend`),
   - run de nuevo contenedor en `-p 80:80`,
   - limpieza de imágenes (`docker image prune -a -f`).

---

## Secrets requeridos

El workflow depende de estos secrets de GitHub:

- `AWS_ACCOUNT_ID`
- `AWS_REGION`
- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`
- `AWS_SESSION_TOKEN`
- `AWS_ECR_REPOSITORY`
- `EC2_INSTANCE_ID`

## Flujo visual

`push main` → `build-and-push` (build + push a ECR) → `deploy-to-ec2` (SSM en EC2, pull y run de contenedor)
