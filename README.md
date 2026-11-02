# GestorDE · Fiscalizador · Demo interactiva (v2 - Bob)

Sitio estático (HTML + CSS + JS). No tiene build ni llama a ningún backend:
los datos de ejemplo están en memoria dentro de `assets/app.js`.

## Contenido
- `index.html`: la demo.
- `assets/`: `app.js`, `style.css`, `fonts.css` (tipografías DM Sans y Manrope desde Google Fonts).
- `Dockerfile` + `nginx.conf`: imagen nginx que sirve el sitio en el puerto **8080** (el puerto por defecto de Code Engine).

## Probar localmente
    docker build -t gestorde-demo .
    docker run --rm -p 8080:8080 gestorde-demo
    # abrir http://localhost:8080

## Desplegar en IBM Cloud Code Engine
Opción A, desde el código fuente local (Code Engine arma la imagen con el Dockerfile):

    ibmcloud login
    ibmcloud ce project select --name <proyecto>
    ibmcloud ce application create --name gestorde-demo --build-source . --strategy dockerfile --port 8080

Opción B, desde la consola: Application > Source code > subir esta carpeta (o un repo Git que la contenga) > estrategia **Dockerfile** > puerto **8080**.

## Notas
- Las fuentes se descargan de `fonts.gstatic.com`: el navegador de quien use la demo necesita internet. Si no, se ve con una tipografía de reemplazo.
- Si hay una política de seguridad (CSP) en el entorno, hay que permitir ese dominio o alojar las fuentes localmente.
