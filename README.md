# Marketing y clientes · analítica GA4

Explora adquisición, conversión y retorno de usuarios a partir de eventos de
comercio electrónico: cohortes, embudo y afinidad por categoría sobre una muestra
pública y ofuscada de **Google Analytics 4**.

[![Build](https://github.com/Dev-HugoAguilar/ga4-marketing-analytics/actions/workflows/tests.yml/badge.svg)](https://github.com/Dev-HugoAguilar/ga4-marketing-analytics/actions/workflows/tests.yml)
[![React](https://img.shields.io/badge/React-TypeScript-3178C6?logo=react&logoColor=white)](app)
[![Datos](https://img.shields.io/badge/muestra-5%25%20de%20usuarios-b7791f)](#datos-y-atribución)

> **Alcance.** Los resultados describen **exclusivamente la muestra publicada** y
> no representan el desempeño comercial actual de Google ni se extrapolan a la
> población original.

---

## Ejecutar

Aplicación React + Vite sin backend propio: consume la API compartida
[`portfolio-api`](https://github.com/Dev-HugoAguilar/portfolio-api) (FastAPI + MongoDB)
en el puerto **8010**. La API no se copia en este repositorio — se clona aparte —
para no duplicar código ni datos.

### 1. Requisitos

- **Python 3.12** para la API.
- **Node.js 20 o superior** con `npm` (verificado con Node 24.11.0 y npm 11.6.1).
- **MongoDB**, Atlas o local: solo hace falta la URI de conexión.

### 2. API en el puerto 8010

```bash
git clone https://github.com/Dev-HugoAguilar/portfolio-api.git
cd portfolio-api
python -m venv .venv
```

Activa el entorno virtual:

```bash
# Windows (PowerShell o CMD)
.venv\Scripts\activate
# Linux / macOS
source .venv/bin/activate
```

Instala y arranca:

```bash
pip install -r requirements-dev.txt
cp .env.example .env        # Windows PowerShell: Copy-Item .env.example .env
python -m uvicorn app.main:app --port 8010
```

Variables que hay que rellenar en `.env` (no se versiona):

| Variable | Qué poner |
|---|---|
| `MONGODB_URI` | **Obligatoria.** URI de conexión de MongoDB: cadena de Atlas (`mongodb+srv://…`, con el usuario y la clave del clúster) o local (`mongodb://127.0.0.1:27017/hugo_portfolio`). |
| `MONGODB_DB` | Base donde están las colecciones publicadas (`hugo_portfolio`). |
| `ALLOWED_ORIGINS` | Orígenes permitidos por CORS; ver el paso 3. |
| `MONGO_ROOT_*`, `APP_DB_*`, `BACKEND_HOST_PORT` | Solo si levantas MongoDB/API con Docker Compose. |
| `MARKETING_GCP_PROJECT` | Solo para **volver a extraer** los datos de GA4 desde BigQuery; la API y el tablero leen de MongoDB. |

`/health/ready` comprueba proceso **y** MongoDB:

```bash
curl http://127.0.0.1:8010/api/v1/health/ready
# {"status":"ready","mongo":"ok","version":"1.0.0"}
```

### 3. Aplicación en el puerto 5173

```bash
cd app
npm ci
npm run dev
```

Vite sirve en <http://127.0.0.1:5173>. El cliente consulta la API en
`http://127.0.0.1:8010/api/v1/marketing` (ver `app/src/api/cliente.ts`), así que la
petición es **de otro origen** (5173 → 8010).

**CORS.** Si la API no autoriza el origen del dev server, el navegador bloquea la
respuesta: error de CORS en la consola y el tablero se queda sin datos. Añade los
dos orígenes de desarrollo a `ALLOWED_ORIGINS` en el `.env` de la API y reinicia
uvicorn:

```env
ALLOWED_ORIGINS=http://localhost:8000,http://127.0.0.1:8000,http://127.0.0.1:5173,http://localhost:5173
```

La lista va separada por comas, sin espacios.

### 4. Compilación de producción

```bash
npm run build
```

Ejecuta `tsc --noEmit`, `vite build` y el minificador de `app/tools/`. La salida
queda en **`app/dist/`** (`app.js` y `app.css`), el módulo que el portafolio carga
con `import()`.

### 5. Comprobación funcional

Endpoint real de este tablero (definido en `app/api/v1/routes/marketing.py` de la
API):

```bash
curl http://127.0.0.1:8010/api/v1/marketing/estado
```

Debe responder `HTTP 200` con `datos_disponibles`, `version`, `periodo`, `cifras`,
`capacidades`, `muestra` y `cobertura_muestra`. Obtenido al verificar este
repositorio: `"datos_disponibles": true`, `"version": "ga4-e9764d626c"` y periodo
`2020-11-01` a `2021-01-31`. Si `datos_disponibles` es `false`, la base no tiene
versión publicada: volver a extraerla exige credenciales de BigQuery y se hace con
`python -m scripts.importar_marketing` en el repo de la API (la API no inventa
cifras).

Con la API en marcha, abre <http://127.0.0.1:5173>: adquisición, cohortes, embudo
y afinidad deben mostrar la misma versión que devolvió `/estado`.

> **Versión compatible:** probado con `portfolio-api` en el commit `8a27227`.

![Portada del tablero de marketing](docs/cover.jpg)

## Qué demuestra

- **Cohortes honestas**: los usuarios se cuentan de forma distinta dentro de cada
  selección; no se suman los conteos diarios.
- **Compras deduplicadas** por usuario y transacción antes de agregar.
- **Embudo con ventana declarada**: sesiones que completan los pasos en orden
  dentro de una ventana de 24 horas, y la ventana se declara en la interfaz.
- **Trazabilidad del origen**: cada cifra indica la muestra y el periodo del que
  procede.

## Datos y atribución

**Google Analytics 4**, muestra pública y ofuscada de comercio electrónico
(`bigquery-public-data.ga4_obfuscated_sample_ecommerce`, tablas `events_*`).
Periodo observado: **1 de noviembre de 2020 a 31 de enero de 2021**. El demo usa
una selección reproducible del **5 % de los usuarios: 13 509 personas** de una
población original de 270 154.

## Estructura

```text
app/                 Aplicación React + TypeScript (Vite)
  src/               Vistas de adquisición, cohortes, embudo y afinidad
  tools/             Minificador y utilidades de build
docs/LEEME.md        Documentación original del proyecto
docs/cover.jpg       Captura real de la aplicación
```

## Límites

- La muestra es histórica y del 5 % de usuarios: no se extrapola a la población.
- La fuente está ofuscada y puede contener valores de marcador o información
  desconocida.
- El canal describe la **adquisición** del usuario según su primera actividad
  observada: no es atribución por sesión ni evidencia del efecto causal de la
  publicidad.
- El embudo aplica una ventana analítica de 24 horas.
- Las compras por categoría pueden solaparse; la afinidad corresponde al periodo
  completo publicado.

## Seguridad y contribución

Sin credenciales ni datos personales en el repositorio. Ver [`SECURITY.md`](SECURITY.md)
y [`CONTRIBUTING.md`](CONTRIBUTING.md).
