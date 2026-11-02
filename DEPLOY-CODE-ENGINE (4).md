# Deploy en IBM Code Engine — Step by Step

> Tutorial basado en el deploy real de la API mock de mock (ABL + Patente).
> Fecha: 17/04/2026

---

## Prerequisitos

- Docker Desktop instalado y corriendo
- Cuenta activa en IBM Cloud
- `jq` instalado (`brew install jq`)

---

## Paso 1 — Instalar IBM Cloud CLI y plugins

```bash
# Instalar IBM Cloud CLI
brew install ibm-cloud-cli

# Instalar plugins necesarios
ibmcloud plugin install code-engine
ibmcloud plugin install container-registry
```

Verificar instalación:
```bash
ibmcloud version
ibmcloud ce version
```

---

## Paso 2 — Login a IBM Cloud

```bash
ibmcloud login --sso
```

- Se abre el browser
- Copiás el one-time passcode
- Lo pegás en la terminal

Seleccionás la **cuenta** y la **región** (`us-south` en nuestro caso).

---

## Paso 3 — Apuntar al Resource Group

```bash
# Ver los resource groups disponibles
ibmcloud resource groups

# Apuntar al grupo correcto
ibmcloud target -g Default
```

> **Importante:** sin este paso, los comandos de `ibmcloud cr` fallan.

---

## Paso 4 — Login al Container Registry

```bash
ibmcloud cr login
```

Esto hace el `docker login` automáticamente a `us.icr.io`.

---

## Paso 5 — Crear un namespace en el Container Registry

```bash
ibmcloud cr namespace-add mock
```

El namespace es el "folder" donde van a vivir tus imágenes Docker dentro del ICR.
La imagen quedará en: `us.icr.io/mock-mock/<nombre-imagen>:tag`

---

## Paso 6 — Preparar la app (checklist del Dockerfile)

Antes de buildear, asegurate de que el **Dockerfile**:

1. Use `--platform linux/amd64` (Code Engine corre en Intel/Linux, no ARM)
2. Copie **todos** los archivos que la app necesita en runtime

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# ⚠️ Incluir TODOS los archivos que usa la app
COPY app.py mock_data.py openapi.yaml ./

EXPOSE 8080

CMD ["gunicorn", "--bind", "0.0.0.0:8080", "--workers", "2", "--timeout", "30", "app:app"]
```

> **Error común:** si olvidás copiar un archivo (ej: `openapi.yaml`), la app
> deploya pero tira 500 al intentar leerlo.

---

## Paso 7 — Build de la imagen Docker

```bash
docker build --platform linux/amd64 \
  -t us.icr.io/mock-mock/endpoints:latest .
```

> **Crítico:** siempre usar `--platform linux/amd64` si buildeas desde Mac con
> Apple Silicon (M1/M2/M3). Sin esto, el contenedor no arranca en Code Engine
> y el error es `Initial scale was never achieved`.

---

## Paso 8 — Push de la imagen al registry

```bash
docker push us.icr.io/mock-mock/endpoints:latest
```

Si da error de autorización, primero corré `ibmcloud cr login` (Paso 4) y reintentá.

---

## Paso 9 — Crear el proyecto en Code Engine

```bash
ibmcloud ce project create --name mock-poc
```

Tarda ~30 segundos en activarse. Luego:

```bash
ibmcloud ce project select --name mock-poc
```

> Un proyecto es el contenedor lógico de todas tus apps en Code Engine.

---

## Paso 10 — Crear el registry secret

Code Engine necesita credenciales para poder pullear la imagen desde el ICR:

```bash
ibmcloud ce registry create \
  --name icr-secret \
  --server us.icr.io \
  --username iamapikey \
  --password $(ibmcloud iam api-key-create ce-key --output json | jq -r .apikey)
```

Esto crea una API key de IBM Cloud y la guarda como secret dentro del proyecto.

---

## Paso 11 — Deploy de la aplicación

```bash
ibmcloud ce application create \
  --name mock-endpoints \
  --image us.icr.io/mock-mock/endpoints:latest \
  --registry-secret icr-secret \
  --port 8080 \
  --min-scale 0 \
  --max-scale 2
```

| Flag | Qué hace |
|---|---|
| `--name` | Nombre de la app dentro del proyecto |
| `--image` | Imagen Docker a usar |
| `--registry-secret` | Secret con credenciales del ICR |
| `--port` | Puerto que expone el contenedor |
| `--min-scale 0` | Se apaga si no hay tráfico (ahorra costos) |
| `--max-scale 2` | Máximo 2 instancias en paralelo |

---

## Paso 12 — Verificar el deploy

```bash
ibmcloud ce application get -n mock-endpoints
```

Buscás estas líneas:

```
Status Summary:  Application deployed successfully
URL:             https://mock-endpoints.28spgrgu2z4e.us-south.codeengine.appdomain.cloud
```

Si dice `Application failed to deploy`, revisás los logs:

```bash
ibmcloud ce application logs -f -n mock-endpoints
```

---

## Actualizar la app (re-deploy)

Cada vez que cambiás el código:

```bash
# 1. Rebuild con plataforma correcta
docker build --platform linux/amd64 -t us.icr.io/mock-mock/endpoints:latest .

# 2. Push
docker push us.icr.io/mock-mock/endpoints:latest

# 3. Update (Code Engine pullea la nueva imagen)
ibmcloud ce application update \
  --name mock-endpoints \
  --image us.icr.io/mock-mock/endpoints:latest
```

---

## URLs del deploy actual

| Recurso | URL |
|---|---|
| API base | `https://mock-endpoints.28spgrgu2z4e.us-south.codeengine.appdomain.cloud` |
| Swagger UI | `https://mock-endpoints.28spgrgu2z4e.us-south.codeengine.appdomain.cloud/docs` |
| OpenAPI YAML | `https://mock-endpoints.28spgrgu2z4e.us-south.codeengine.appdomain.cloud/openapi.yaml` |
| Console IBM Cloud | `https://cloud.ibm.com/codeengine/project/us-south/1b2f9421-6af8-4e72-8355-fe8856e70873/application/mock-endpoints/configuration` |

---

## Errores comunes y soluciones

| Error | Causa | Solución |
|---|---|---|
| `Authorization required` al hacer push | `ibmcloud cr login` no se hizo antes del push | Correr `ibmcloud cr login` y volver a pushear |
| `no resource group is targeted` | Falta apuntar al resource group | `ibmcloud target -g Default` |
| `Initial scale was never achieved` | Imagen buildeada para ARM, Code Engine es amd64 | Rebuild con `--platform linux/amd64` |
| `response status is 500 /openapi.yaml` | El archivo no fue copiado en el Dockerfile | Agregar el archivo al `COPY` del Dockerfile |

---

## Comandos útiles de operación

```bash
# Ver estado de la app
ibmcloud ce application get -n mock-endpoints

# Ver logs en tiempo real
ibmcloud ce application logs -f -n mock-endpoints

# Ver eventos del sistema
ibmcloud ce application events -n mock-endpoints

# Listar todas las apps del proyecto
ibmcloud ce application list

# Listar proyectos
ibmcloud ce project list

# Eliminar la app
ibmcloud ce application delete --name mock-endpoints
```
