# Guía de Despliegue en Azure

Esta guía cubre la creación de todos los recursos de Azure necesarios.
Los comandos de esta guía se ejecutan **una sola vez** desde tu terminal local.
Después, el pipeline de GitHub Actions se encarga de los deploys automáticamente.

---

## Prerequisitos

- [Azure CLI](https://learn.microsoft.com/es-es/cli/azure/install-azure-cli) instalado
- Cuenta en [Neon.tech](https://neon.tech) (PostgreSQL gratuito — recomendado)
- Repositorios `sistema-turnos-back` y `sistema-turnos-front` subidos a GitHub

---

## Paso 1 — Login y suscripción

```bash
az login

# Verificá qué suscripción está activa
az account show --query "{nombre:name, id:id}" -o table

# Si tenés varias, seleccioná la correcta
az account set --subscription "NOMBRE_O_ID_DE_TU_SUSCRIPCION"
```

---

## Paso 2 — Grupo de recursos

```bash
az group create \
  --name rg-sistema-turnos \
  --location eastus
```

---

## Paso 3 — Azure Container Registry (ACR)

> El nombre del ACR debe ser **globalmente único** y solo letras/números.
> Reemplazá `turnosacr` por algo único, ej: `turnosACRtu_nombre`.

```bash
# Crear el ACR (plan Basic ~$5/mes)
az acr create \
  --resource-group rg-sistema-turnos \
  --name turnosacr \
  --sku Basic \
  --admin-enabled true

# Obtener las credenciales (las necesitarás para los Secrets de GitHub)
az acr credential show --name turnosacr --query "{usuario:username, password:passwords[0].value}" -o table
```

Guardá el **username** y **password** que devuelve este comando.

---

## Paso 4 — Base de datos PostgreSQL (Neon — gratis)

1. Entrá a [neon.tech](https://neon.tech) y creá una cuenta gratuita
2. Creá un proyecto llamado `sistema-turnos`
3. Copiá la **Connection String** que tiene este formato:
   ```
   postgresql://usuario:password@ep-xxx-xxx.us-east-2.aws.neon.tech/neondb?sslmode=require
   ```
4. Convertila a formato JDBC para Spring Boot:
   ```
   jdbc:postgresql://ep-xxx-xxx.us-east-2.aws.neon.tech/neondb?sslmode=require
   ```

> **Alternativa paga:** Azure Database for PostgreSQL Flexible Server (B1ms ~$12/mes)
> ```bash
> az postgres flexible-server create \
>   --resource-group rg-sistema-turnos \
>   --name turnos-db-server \
>   --location eastus \
>   --admin-user turnosuser \
>   --admin-password "TuPasswordSegura123!" \
>   --sku-name Standard_B1ms \
>   --tier Burstable \
>   --storage-size 32 \
>   --version 16
> ```

---

## Paso 5 — Azure Container Apps Environment

```bash
# Habilitar la extensión si no la tenés
az extension add --name containerapp --upgrade

# Registrar providers necesarios
az provider register --namespace Microsoft.App
az provider register --namespace Microsoft.OperationalInsights

# Crear el environment (capa gratuita disponible)
az containerapp env create \
  --name turnos-env \
  --resource-group rg-sistema-turnos \
  --location eastus
```

---

## Paso 6 — Primer despliegue manual (solo una vez)

> Antes de correr estos comandos, asegurate de haber subido al menos una imagen al ACR.
> Podés hacerlo con `docker build` + `docker push` manual, o disparar el pipeline una vez
> después de configurar los Secrets.

### 6a — Backend

```bash
ACR_SERVER="turnosacr.azurecr.io"    # reemplazá con tu ACR
ACR_USER="turnosacr"
ACR_PASS="<password-del-paso-3>"

az containerapp create \
  --name turnos-backend \
  --resource-group rg-sistema-turnos \
  --environment turnos-env \
  --image $ACR_SERVER/turnos-backend:latest \
  --registry-server $ACR_SERVER \
  --registry-username $ACR_USER \
  --registry-password $ACR_PASS \
  --target-port 8090 \
  --ingress internal \
  --cpu 0.5 \
  --memory 1.0Gi \
  --min-replicas 0 \
  --max-replicas 1 \
  --env-vars \
    DB_URL="jdbc:postgresql://ep-xxx.neon.tech/neondb?sslmode=require" \
    DB_USERNAME="<neon-user>" \
    DB_PASSWORD="<neon-password>" \
    MAIL_USERNAME="" \
    MAIL_PASSWORD="" \
    TWILIO_ACCOUNT_SID="" \
    TWILIO_AUTH_TOKEN="" \
    TWILIO_PHONE="" \
    TWILIO_WHATSAPP_FROM=""
```

> `--ingress internal` hace que el backend NO sea accesible desde internet,
> solo desde otros servicios del mismo environment (como el frontend).

### 6b — Frontend

```bash
# Primero obtené el hostname interno del backend
BACKEND_FQDN=$(az containerapp show \
  --name turnos-backend \
  --resource-group rg-sistema-turnos \
  --query properties.configuration.ingress.fqdn -o tsv)

echo "Backend interno: https://$BACKEND_FQDN"

az containerapp create \
  --name turnos-frontend \
  --resource-group rg-sistema-turnos \
  --environment turnos-env \
  --image $ACR_SERVER/turnos-frontend:latest \
  --registry-server $ACR_SERVER \
  --registry-username $ACR_USER \
  --registry-password $ACR_PASS \
  --target-port 80 \
  --ingress external \
  --cpu 0.25 \
  --memory 0.5Gi \
  --min-replicas 0 \
  --max-replicas 1 \
  --env-vars \
    BACKEND_URL="https://$BACKEND_FQDN"
```

---

## Paso 7 — Service Principal para GitHub Actions

```bash
# Obtener el ID de tu suscripción
SUB_ID=$(az account show --query id -o tsv)

# Crear el Service Principal con permisos sobre el resource group
az ad sp create-for-rbac \
  --name "sp-github-turnos" \
  --role contributor \
  --scopes /subscriptions/$SUB_ID/resourceGroups/rg-sistema-turnos \
  --json-auth
```

> Si tu Azure CLI es más antiguo y no reconoce `--json-auth`, usá `--sdk-auth`.

Copiá **todo el JSON** que devuelve este comando. Lo necesitás en el paso siguiente.

---

## Paso 8 — Secrets de GitHub

Configurá estos 4 Secrets **en cada repositorio** (`sistema-turnos-back` y `sistema-turnos-front`):

`Settings → Secrets and variables → Actions → New repository secret`

| Secret | Valor |
|---|---|
| `AZURE_CREDENTIALS` | El JSON completo del paso 7 |
| `ACR_LOGIN_SERVER` | `turnosacr.azurecr.io` |
| `ACR_USERNAME` | Username del paso 3 |
| `ACR_PASSWORD` | Password del paso 3 |

---

## Cómo funciona el pipeline

```
push a main
    │
    ├─► sistema-turnos-back  →  build imagen  →  push ACR  →  az containerapp update (backend)
    └─► sistema-turnos-front →  build imagen  →  push ACR  →  az containerapp update (frontend)
```

Cada repo tiene su propio workflow en `.github/workflows/ci-cd.yml`.
El pipeline se dispara independientemente: si solo cambiás el back, solo se redespliega el back.

---

## Costos estimados (Azure)

| Recurso | Plan | Costo/mes |
|---|---|---|
| ACR | Basic | ~$5 |
| Container Apps | Consumo (free tier: 180K vCPU-s/mes) | $0 para tráfico bajo |
| PostgreSQL | Neon free tier | $0 |
| **Total** | | **~$5/mes** |

---

## URLs finales

Después del deploy, obtenés la URL pública del frontend con:

```bash
az containerapp show \
  --name turnos-frontend \
  --resource-group rg-sistema-turnos \
  --query properties.configuration.ingress.fqdn -o tsv
```
