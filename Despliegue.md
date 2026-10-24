
# Despliegue de la PokeDex en Azure App Service

Tutorial para publicar la aplicación **PokeDex** en **Azure App Service** de forma segura, con despliegue automático mediante **GitHub Actions** y encabezados de seguridad HTTP.

> **Requisito:** tener creada la cuenta Azure for Students y el App Service, como se explica en el [README.md](https://claude.ai/chat/README.md).

----------

## Tabla de contenido

1.  [Visión general](https://claude.ai/chat/5aeb919c-0e76-4992-bd94-3a61be87d0c4#1-visi%C3%B3n-general)
2.  [Preparar el repositorio](https://claude.ai/chat/5aeb919c-0e76-4992-bd94-3a61be87d0c4#2-preparar-el-repositorio)
3.  [Compilar y probar localmente](https://claude.ai/chat/5aeb919c-0e76-4992-bd94-3a61be87d0c4#3-compilar-y-probar-localmente)
4.  [Configurar la seguridad de la plataforma](https://claude.ai/chat/5aeb919c-0e76-4992-bd94-3a61be87d0c4#4-configurar-la-seguridad-de-la-plataforma)
5.  [Conectar GitHub con Azure](https://claude.ai/chat/5aeb919c-0e76-4992-bd94-3a61be87d0c4#5-conectar-github-con-azure)
6.  [Preparar los cambios de despliegue](https://claude.ai/chat/5aeb919c-0e76-4992-bd94-3a61be87d0c4#6-preparar-los-cambios-de-despliegue)
7.  [Publicar los cambios en un solo commit](https://claude.ai/chat/5aeb919c-0e76-4992-bd94-3a61be87d0c4#7-publicar-los-cambios-en-un-solo-commit)
8.  [Verificar el despliegue](https://claude.ai/chat/5aeb919c-0e76-4992-bd94-3a61be87d0c4#8-verificar-el-despliegue)
9.  [Operación y mantenimiento](https://claude.ai/chat/5aeb919c-0e76-4992-bd94-3a61be87d0c4#9-operaci%C3%B3n-y-mantenimiento)

----------

## 1. Visión general

**Aplicación**

Angular 14 (SPA) que consume la PokéAPI (REST y GraphQL)

**Hosting**

Azure App Service, Windows (IIS), plan Free F1

**CI/CD**

GitHub Actions con autenticación OIDC (sin contraseñas)

**URL**

https://app-pokedex-prod-rkeyy7-e7edh5bah5aje7g2.mexicocentral-01.azurewebsites.net

### Flujo de despliegue

```mermaid
flowchart LR
    A[Desarrollador<br/>git push] --> B[GitHub<br/>rama main]
    B --> C[GitHub Actions<br/>job build]
    C -->|npm ci<br/>npm run build<br/>npm run test:cov| D[Artefacto<br/>dist/pokedex-angular]
    D --> E[GitHub Actions<br/>job deploy]
    E -->|OIDC| F[Azure App Service<br/>IIS + web.config]
    F --> G[Usuario<br/>HTTPS + encabezados]

```

Una vez configurado, cada `git push` a la rama `main` compila, prueba y publica la aplicación automáticamente en unos **5 minutos**.

### Estrategia de commits

Este tutorial agrupa **todas las modificaciones del código en un único commit** ([sección 7](https://claude.ai/chat/5aeb919c-0e76-4992-bd94-3a61be87d0c4#7-publicar-los-cambios-en-un-solo-commit)):

-   Los cambios se verifican **localmente** antes de subirlos.
-   Se ejecuta **un solo despliegue**, en lugar de uno por cada cambio.
-   El historial del repositorio queda limpio y fácil de revisar.

Commit

Contenido

1

Código fuente original

2

Workflow generado automáticamente por Azure

3

Todas las adaptaciones para Azure: rutas, workflow, `web.config` y seguridad

----------

## 2. Preparar el repositorio

### 2.1 Crear el repositorio

1.  En GitHub, haz clic en **New repository**.
2.  Configura:
    -   **Repository name:** `pokedex`
    -   **Visibility:** Public
    -   **Add README:** activado
    -   **Add .gitignore:** Node
3.  Haz clic en **Create repository** y clónalo:

```bash
git clone https://github.com/<tu-usuario>/pokedex.git
cd pokedex

```

### 2.2 Agregar el código fuente

1.  Descarga el código de la PokeDex (carpeta `source/pokedex-angular/` del repositorio de PumasLab).
2.  Copia el **contenido** de esa carpeta a la **raíz** de tu repositorio, de modo que `package.json` y `angular.json` queden al mismo nivel que el `README.md`.
3.  **No copies** estas carpetas si existen:

Carpeta

Motivo

`node_modules/`

Dependencias (cientos de MB). Se regeneran con `npm ci`.

`.angular/`

Caché de compilación local.

`.github/`

Workflows del autor original para GitHub Pages; se reemplazarán por el de Azure.

4.  Reemplaza tu `.gitignore` por el del proyecto Angular. Verifica que incluya:

```gitignore
/dist
/node_modules
/.angular/cache

```

5.  Sube el código original **sin modificaciones** (commit 1):

```bash
git add .
git commit -m "Código fuente original de PumasLab"
git push

```

> Subir primero el código original permite que el historial muestre claramente qué se modificó para desplegarlo.

### 2.3 Convención de commits

El proyecto incluye **husky** y **Commitizen**, que aplican la convención **Conventional Commits**. Al ejecutar `git commit` (sin `-m`), un asistente solicita:

Pregunta

Ejemplo

Tipo de cambio

`feat`, `fix`, `ci`, `docs`

Alcance

`deploy`, `security`

Descripción corta

`adapta la aplicacion para azure app service`

> **Consejo:** ejecuta los comandos de uno en uno. Si pegas varios a la vez, el asistente interactivo toma los siguientes comandos como respuestas.

----------

## 3. Compilar y probar localmente

### 3.1 Versión de Node.js

Usa **Node.js 22 LTS**. Aunque Angular 14 fue diseñado para Node 14/16, esas versiones ya no reciben actualizaciones de seguridad, y el build de producción funciona correctamente con Node 22.

```bash
node -v   # v22.x

```

### 3.2 Instalar dependencias

```bash
npm ci

```

`npm ci` (**clean install**) instala **exactamente** las versiones de `package-lock.json`, sin modificarlo, y crea `node_modules` desde cero. Es el comando recomendado para despliegues porque garantiza la **reproducibilidad**: tu equipo y el servidor de CI instalan lo mismo.

### 3.3 Compilar para producción

```bash
npm run build

```

El resultado queda en **`dist/pokedex-angular`** (propiedad `outputPath` de `angular.json`). Es la carpeta que se publica en Azure.

> Las advertencias de _budgets_ (tamaño del bundle) son recomendaciones de rendimiento y no impiden el build.

### 3.4 Revisar las vulnerabilidades de dependencias

```bash
npm audit              # todas las dependencias
npm audit --omit=dev   # solo las que llegan al navegador

```

-   **`devDependencies`** (herramientas de compilación y pruebas) no llegan al navegador del usuario.
-   **`dependencies`** (Angular, Apollo, etc.) forman parte de la aplicación publicada.

Resultado:

Análisis

Vulnerabilidades

`npm audit` (todas)

**124** (11 bajas, 53 moderadas, 52 altas, 8 críticas)

`npm audit --omit=dev` (producción)

**11** (7 moderadas, 4 altas)

**113 de las 124 vulnerabilidades (91 %) están en herramientas de desarrollo** que nunca llegan al navegador del usuario. Las 11 restantes corresponden a:

-   **Paquetes de Angular** (`@angular/core`, `@angular/common`, `@angular/compiler` y dependientes), afectados en versiones `<= 19.2.25`. La mayoría son variantes de **XSS** por omisión de sanitización y fugas de información en el cliente HTTP. Su corrección exige migrar a Angular 20 o superior, un cambio mayor que no se incluye en este despliegue.
-   **`graphql`** (consumo descontrolado de recursos), que se corrige con `npm audit fix` sin cambios incompatibles.

**Mitigación:** la directiva `script-src 'self'` de la Content-Security-Policy ([sección 6.6](https://claude.ai/chat/5aeb919c-0e76-4992-bd94-3a61be87d0c4#66-entender-la-content-security-policy)) impide que se ejecute JavaScript inyectado, lo que reduce el impacto de las vulnerabilidades de XSS mientras se planifica la migración.

> No ejecutes `npm audit fix --force`: instalaría Angular 21 sobre un proyecto de Angular 14 (un cambio incompatible) y rompería la aplicación. La solución definitiva es una migración planificada a una versión vigente de Angular.

----------

## 4. Configurar la seguridad de la plataforma

Configura el App Service para **reducir la superficie de ataque**. Estos ajustes se hacen en el portal y no requieren commits.

En el portal: tu App Service → **Settings → Configuration → General settings → Platform settings**.

Opción

Valor

Justificación

**HTTPS only**

On

Redirige todo el tráfico `http://` a `https://`.

**Minimum Inbound TLS Version**

1.3

Rechaza versiones antiguas del protocolo con vulnerabilidades conocidas.

**SCM Minimum Inbound TLS Version**

1.3

Aplica la misma exigencia al canal de administración y despliegue (SCM/Kudu).

**SCM Basic Auth Publishing Credentials**

Off

El canal de despliegue no acepta usuario y contraseña.

**FTP Basic Auth Publishing Credentials**

Off

Igual para FTP.

**FTP state**

Disabled

Cierra un canal de acceso que no se usa.

**HTTP version**

2.0

Mejor rendimiento.

**Session affinity**

Off

Evita una cookie innecesaria: la aplicación no guarda estado en el servidor.

**Remote debugging**

Off

La depuración remota no debe estar habilitada en producción.

Haz clic en **Save** y confirma con **Continue** (Azure reinicia la aplicación automáticamente).

**Comprobación:** abre tu dominio con `http://`. El navegador debe redirigirte a `https://`.

Este enfoque aplica **defensa en profundidad**: varias capas de protección (plataforma, transporte y navegador), de modo que si una falla, las demás siguen protegiendo.

----------

## 5. Conectar GitHub con Azure

### 5.1 Configurar el Deployment Center

1.  En tu App Service: **Deployment → Deployment Center → Settings**.
2.  Configura:

Opción

Valor

Source

GitHub (autoriza el acceso a tu cuenta)

Organization

tu usuario

Repository

`pokedex`

Branch

`main`

Workflow option

Add a workflow

Authentication type

**User-assigned identity**

3.  Haz clic en **Save**.

Azure crea el archivo `.github/workflows/main_<nombre-app>.yml` mediante un commit en tu repositorio (commit 2). Ese commit lanza una primera ejecución del workflow.

> **Cancela esa primera ejecución.** El workflow generado por Azure es genérico y todavía no está adaptado a esta aplicación (ver [sección 6.2](https://claude.ai/chat/5aeb919c-0e76-4992-bd94-3a61be87d0c4#62-ajustar-el-workflow)): su paso de pruebas no termina nunca. En GitHub, pestaña **Actions**, entra a la ejecución y haz clic en **Cancel workflow**.

Trae el workflow a tu equipo:

```bash
git pull

```

### 5.2 Autenticación sin contraseñas (OIDC)

Como la autenticación básica está deshabilitada, el despliegue no usa contraseñas. Azure crea una **identidad administrada** y guarda tres identificadores como _secrets_ del repositorio (cliente, tenant y suscripción).

El permiso `id-token: write` del workflow permite que GitHub emita un **token temporal firmado** (JWT) que demuestra que la ejecución proviene de tu repositorio y rama. Azure confía en ese token mediante una **credencial federada (OIDC)**. El token dura minutos, así que **no existe ninguna credencial permanente que pueda filtrarse**.

----------

## 6. Preparar los cambios de despliegue

En esta sección se hacen **todas las modificaciones** necesarias. **No hagas commit todavía**: se publicarán juntas en la [sección 7](https://claude.ai/chat/5aeb919c-0e76-4992-bd94-3a61be87d0c4#7-publicar-los-cambios-en-un-solo-commit).

#

Archivo

Cambio

6.1

`src/environments/environment.prod.ts`

Ruta de imágenes para la raíz del dominio

6.2

`.github/workflows/main_<nombre-app>.yml`

Adaptar el workflow a Angular

6.3

`src/web.config` (nuevo)

Rutas de la SPA y encabezados de seguridad

6.4

`src/index.html` y `src/404.html`

Eliminar código en línea de GitHub Pages

6.5

`angular.json`

Incluir `web.config` y desactivar el CSS crítico en línea

### 6.1 Adaptar la ruta de imágenes

El código original se publicaba en GitHub Pages, bajo una subcarpeta. En Azure, la aplicación se publica en la **raíz** del dominio.

Angular usa un archivo de configuración por entorno; en producción, `environment.ts` se reemplaza por `environment.prod.ts` (sección `fileReplacements` de `angular.json`). Corrige la ruta en **`src/environments/environment.prod.ts`**:

```diff
- imagesPath: '/pokedex-angular/assets/images',
+ imagesPath: '/assets/images',

```

### 6.2 Ajustar el workflow

El workflow que genera Azure es genérico para Node. Aplica estos cambios:

Cambio

Motivo

`actions/setup-node@v3` → `@v4`

Versión vigente de la acción.

`npm install` → `npm ci`

Instalación reproducible.

`npm run test` → `npm run test:cov`

El script `test` se ejecuta en **modo watch** (queda esperando cambios y nunca termina). `test:cov` usa `--no-watch` y un Chrome **headless** (sin interfaz gráfica), adecuado para servidores.

`path: .` → `path: dist/pokedex-angular`

Publica solo el resultado del build, no todo el repositorio.

> **Cuidado con la sangría:** en YAML los espacios definen la estructura. Mantén exactamente la sangría del archivo original.

Workflow resultante:

```yaml
name: Build and deploy Node.js app to Azure Web App - app-pokedex-prod-rkeyy7

on:
  push:
    branches:
      - 'main'
  workflow_dispatch:          # permite lanzar el despliegue manualmente

jobs:
  build:
    runs-on: windows-latest
    permissions:
      contents: read

    steps:
      - uses: actions/checkout@v4

      - name: Set up Node.js version
        uses: actions/setup-node@v4
        with:
          node-version: '22.x'

      - name: npm ci, build, and test
        run: |
          npm ci
          npm run build
          npm run test:cov

      - name: Upload artifact for deployment job
        uses: actions/upload-artifact@v4
        with:
          name: node-app
          path: dist/pokedex-angular   # solo el resultado del build

  deploy:
    runs-on: ubuntu-latest
    needs: build                       # solo se ejecuta si build termina bien
    permissions:
      id-token: write                  # necesario para OIDC
      contents: read

    steps:
      - name: Download artifact from build job
        uses: actions/download-artifact@v4
        with:
          name: node-app

      - name: Login to Azure
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZUREAPPSERVICE_CLIENTID_... }}
          tenant-id: ${{ secrets.AZUREAPPSERVICE_TENANTID_... }}
          subscription-id: ${{ secrets.AZUREAPPSERVICE_SUBSCRIPTIONID_... }}

      - name: 'Deploy to Azure Web App'
        uses: azure/webapps-deploy@v3
        with:
          app-name: 'app-pokedex-prod-rkeyy7'
          slot-name: 'Production'
          package: .

```

> Los nombres de los _secrets_ los genera Azure automáticamente; conserva los de tu archivo.

**Cómo funciona:**

-   **`on`:** se ejecuta con cada push a `main` o manualmente (`workflow_dispatch`).
-   **Job `build`:** descarga el código, instala Node 22, instala dependencias, compila y ejecuta las **51 pruebas unitarias**. Si alguna prueba falla, el despliegue se detiene (_quality gate_). Al final empaqueta `dist/pokedex-angular` como **artefacto**.
-   **Job `deploy`:** depende de `build` (`needs`). Como los jobs corren en máquinas distintas, el artefacto es la forma de pasar los archivos. Inicia sesión en Azure con OIDC y publica.

### 6.3 Crear `src/web.config`

App Service Windows usa **IIS**, que lee un archivo `web.config` en la raíz del sitio. Con él se resuelven dos necesidades:

1.  **Rutas de la SPA:** al abrir o recargar directamente una ruta como `/pokemon/3`, el navegador pide al servidor un archivo con ese nombre, que no existe. Sin configuración, IIS responde **404**.
2.  **Encabezados de seguridad.**

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <system.webServer>

    <!-- 1. Enrutamiento de la SPA -->
    <rewrite>
      <rules>
        <rule name="Angular SPA" stopProcessing="true">
          <match url=".*" />
          <conditions logicalGrouping="MatchAll">
            <add input="{REQUEST_FILENAME}" matchType="IsFile" negate="true" />
            <add input="{REQUEST_FILENAME}" matchType="IsDirectory" negate="true" />
          </conditions>
          <action type="Rewrite" url="/index.html" />
        </rule>
      </rules>
    </rewrite>

    <!-- 2. Encabezados de seguridad -->
    <httpProtocol>
      <customHeaders>
        <remove name="X-Powered-By" />
        <add name="Strict-Transport-Security" value="max-age=31536000; includeSubDomains" />
        <add name="X-Content-Type-Options" value="nosniff" />
        <add name="X-Frame-Options" value="DENY" />
        <add name="Referrer-Policy" value="strict-origin-when-cross-origin" />
        <add name="Permissions-Policy" value="camera=(), microphone=(), geolocation=(), payment=(), usb=()" />
        <add name="Content-Security-Policy" value="default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline' https://fonts.googleapis.com; font-src 'self' https://fonts.gstatic.com; img-src 'self' data: https://assets.pokemon.com; connect-src 'self' https://pokeapi.co https://beta.pokeapi.co; object-src 'none'; base-uri 'self'; form-action 'self'; frame-ancestors 'none'; upgrade-insecure-requests" />
      </customHeaders>
    </httpProtocol>

    <!-- 3. Ocultar la tecnología del servidor -->
    <security>
      <requestFiltering removeServerHeader="true" />
    </security>

    <!-- 4. Tipo MIME para las fuentes -->
    <staticContent>
      <remove fileExtension=".woff2" />
      <mimeMap fileExtension=".woff2" mimeType="font/woff2" />
    </staticContent>

  </system.webServer>
</configuration>

```

**Bloque 1, reescritura:** si la petición **no** corresponde a un archivo ni a una carpeta reales, IIS entrega `index.html`. Angular arranca, lee la URL y muestra la vista correcta. Es una **reescritura** (_Rewrite_), no una redirección: la URL del navegador no cambia.

**Bloque 2, encabezados de seguridad:**

Encabezado

Qué hace

Ataque que previene

**Strict-Transport-Security**

Indica al navegador que durante un año (31 536 000 s) solo se conecte por HTTPS. A diferencia de _HTTPS Only_ (que redirige en el servidor), protege incluso la primera petición.

Interceptación (_man-in-the-middle_), _SSL stripping_

**X-Content-Type-Options**

Impide que el navegador "adivine" el tipo de un archivo.

Ejecución de archivos disfrazados (_MIME sniffing_)

**X-Frame-Options**

Prohíbe mostrar la aplicación dentro de un `<iframe>` de otro sitio.

_Clickjacking_

**Referrer-Policy**

Al navegar hacia otro sitio, solo envía el dominio de origen, no la URL completa.

Filtración de información

**Permissions-Policy**

Desactiva funciones del navegador que la aplicación no usa (cámara, micrófono, ubicación, pagos, USB).

Abuso de APIs del navegador

**Content-Security-Policy**

Lista blanca de orígenes permitidos (ver [sección 6.6](https://claude.ai/chat/5aeb919c-0e76-4992-bd94-3a61be87d0c4#66-entender-la-content-security-policy)).

**Cross-Site Scripting (XSS)**

Además, `<remove name="X-Powered-By" />` elimina el encabezado que revela ASP.NET.

**Bloque 3:** elimina el encabezado `Server`, que revela IIS y su versión. Cuanta menos información tenga un atacante sobre el servidor, mejor.

**Bloque 4:** asegura que IIS sirva las fuentes `.woff2` con el tipo MIME correcto.

### 6.4 Eliminar el código en línea de GitHub Pages

Una CSP estricta prohíbe el JavaScript dentro del HTML. El código original incluye un mecanismo de rutas para GitHub Pages basado en scripts en línea, que en Azure reemplaza la regla de reescritura del `web.config`:

1.  **`src/index.html`:** elimina el bloque entre `<!-- Start Single Page Apps for GitHub Pages -->` y `<!-- End Single Page Apps for GitHub Pages -->`, incluidos ambos comentarios.
2.  **Elimina el archivo `src/404.html`.**

### 6.5 Ajustar `angular.json`

**a) Incluir `web.config` en el build.** Para que llegue a Azure, debe quedar dentro de `dist/pokedex-angular`. En `architect > build > options > assets`, quita `src/404.html` y agrega `src/web.config`:

```json
"assets": [
  "src/favicon.ico",
  "src/og-image.jpg",
  "src/assets",
  "src/web.config"
]

```

> Modifica solo la lista de la sección `build`, no la de `test`.

**b) Desactivar el CSS crítico en línea.** Por defecto, Angular inyecta en `index.html` CSS en línea y un atributo `onload="..."` (JavaScript en línea). En `configurations > production`, agrega antes de `"budgets"`:

```json
"optimization": {
  "scripts": true,
  "styles": { "minify": true, "inlineCritical": false },
  "fonts": true
},

```

### 6.6 Entender la Content-Security-Policy

La **CSP** es el encabezado más importante contra **XSS**: aunque un atacante lograra inyectar un `<script>` en la página, el navegador se negaría a ejecutarlo. Por eso fue necesario eliminar el código en línea (6.4 y 6.5).

Orígenes que usa la aplicación:

Recurso

Origen

JavaScript

Solo archivos propios

CSS

Archivos propios, Google Fonts y estilos que Angular inyecta en tiempo de ejecución

Fuentes

Fuente Pokémon local y Roboto (`fonts.gstatic.com`)

Imágenes

Sprites locales y _artwork_ oficial (`assets.pokemon.com`)

Datos

PokéAPI REST (`pokeapi.co`) y GraphQL (`beta.pokeapi.co`)

> Algunas URL se construyen dinámicamente en el código (por ejemplo, el _artwork_ del detalle en `pokemon-specie.mapper.ts`), así que conviene confirmar los orígenes navegando por la aplicación con las herramientas de desarrollador abiertas (F12 → Network).

Política aplicada:

```
default-src 'self';
script-src 'self';
style-src 'self' 'unsafe-inline' https://fonts.googleapis.com;
font-src 'self' https://fonts.gstatic.com;
img-src 'self' data: https://assets.pokemon.com;
connect-src 'self' https://pokeapi.co https://beta.pokeapi.co;
object-src 'none';
base-uri 'self';
form-action 'self';
frame-ancestors 'none';
upgrade-insecure-requests

```

Directiva

Significado

`default-src 'self'`

Por defecto, solo recursos del propio dominio.

`script-src 'self'`

JavaScript solo desde archivos propios. **Sin `'unsafe-inline'`**: es la protección principal contra XSS.

`style-src 'self' 'unsafe-inline' https://fonts.googleapis.com`

Estilos propios, de Google Fonts y en línea (ver nota).

`font-src 'self' https://fonts.gstatic.com`

Fuentes propias y de Google Fonts.

`img-src 'self' data: https://assets.pokemon.com`

Imágenes propias, incrustadas y del sitio oficial de Pokémon.

`connect-src 'self' https://pokeapi.co https://beta.pokeapi.co`

Las peticiones de datos solo pueden ir a la PokéAPI.

`object-src 'none'`

Bloquea plugins (`<object>`, `<embed>`).

`base-uri 'self'`

Impide alterar la URL base del documento.

`form-action 'self'`

Los formularios solo envían datos al propio dominio.

`frame-ancestors 'none'`

Versión moderna de `X-Frame-Options: DENY`.

`upgrade-insecure-requests`

Todo recurso pedido por `http://` se solicita por `https://`.

> **Nota sobre `'unsafe-inline'` en `style-src`:** Angular 14 inyecta los estilos de cada componente como etiquetas `<style>` al ejecutarse, y Font Awesome hace lo mismo con los íconos. Sin esta directiva, la aplicación perdería sus estilos. Es un riesgo aceptado: inyectar CSS es mucho menos peligroso que inyectar JavaScript, que sí está bloqueado. Desde Angular 16 se puede eliminar mediante _nonces_.

**Principio de mínimo privilegio:** cada directiva permite exactamente los dominios necesarios, nunca comodines como `https:` o `*`.

----------

## 7. Publicar los cambios en un solo commit

### 7.1 Verificar localmente

Antes de subir, comprueba que todo compila:

```bash
npm run build

```

Revisa la carpeta `dist/pokedex-angular`:

-   [ ] Contiene `web.config`.
-   [ ] **No** contiene `404.html`.
-   [ ] En `index.html` no hay ningún `<script>` con código dentro ni atributos `onload`; solo `<script src="...">`.

### 7.2 Revisar los archivos modificados

```bash
git status

```

Deben aparecer exactamente estos archivos:

```
modified:   .github/workflows/main_<nombre-app>.yml
modified:   angular.json
deleted:    src/404.html
modified:   src/environments/environment.prod.ts
modified:   src/index.html
Untracked files:
            src/web.config

```

### 7.3 Commit y push (commit 3)

Ejecuta los comandos de uno en uno:

```bash
git add .

```

```bash
git commit

```

En el asistente de Commitizen:

Pregunta

Respuesta

Type

`feat`

Scope

`deploy`

Description

`adapta la aplicacion para azure app service con encabezados de seguridad`

Longer description

`Ajusta ruta de imagenes, workflow de CI/CD, web.config con rewrite SPA y encabezados de seguridad, y elimina codigo en linea para la CSP.`

Breaking changes / issues

`N`

```bash
git push

```

El push lanza el workflow. En la pestaña **Actions** de GitHub, la ejecución debe terminar con **build** y **deploy** en verde en unos 5 minutos.

----------

## 8. Verificar el despliegue

### 8.1 Pruebas funcionales

Abre la aplicación en una ventana de incógnito con **F12 → Console** abierta:

Prueba

Resultado esperado

Lista principal

Carga con los sprites de los Pokémon.

Tipografía

Se ve la fuente pixelada de Pokémon.

Búsqueda

Filtra los Pokémon.

Detalle de un Pokémon

Muestra el _artwork_ y los datos de la API.

Cambio de versión de la Pokédex

Cambia los sprites.

Acceso directo a `/pokemon/3`

Carga el detalle (regla de reescritura).

`http://` en lugar de `https://`

Redirige a HTTPS.

Vista móvil (F12 → modo dispositivo)

Se adapta a la pantalla.

Consola

Sin errores **"Refused to..."** (bloqueos de CSP).

> Si aparece un error _"Refused to load... violates the following Content Security Policy directive"_, el mensaje indica qué recurso se bloqueó y qué directiva lo hizo. Agrega solo ese dominio a la directiva correspondiente del `web.config`.

### 8.2 securityheaders.com

1.  Entra a **https://securityheaders.com**.
2.  Pega tu dominio, marca **Hide results** y haz clic en **Scan**.

**Resultado: A+**, con los seis encabezados en verde: Strict-Transport-Security, Content-Security-Policy, X-Frame-Options, X-Content-Type-Options, Referrer-Policy y Permissions-Policy.

Configuración

Calificación

App Service sin configurar

F

Con el `web.config` y la CSP

**A+**

### 8.3 SSL Labs

1.  Entra a **https://www.ssllabs.com/ssltest/**.
2.  Pega tu dominio (sin `https://`), marca **Do not show the results on the boards** y haz clic en **Submit**.

**Resultado: A**

Categoría

Puntaje

Certificate

100

Protocol Support

100

Key Exchange

90

Cipher Strength

90

Hallazgos:

-   ✅ **TLS 1.3** soportado. El puntaje de 100 en _Protocol Support_ se debe a haber configurado TLS 1.3 como versión mínima.
-   ✅ **HSTS de larga duración** detectado.
-   ✅ **Política CAA** en el DNS.
-   ⚠️ Sin intercambio de claves **post-cuántico (PQC)**.

El certificado, los algoritmos de cifrado y el soporte de PQC del dominio `*.azurewebsites.net` los administra Microsoft, y no se pueden modificar desde la aplicación. Es el **modelo de responsabilidad compartida** de la nube:

Responsabilidad del cliente (configurado en este tutorial)

Responsabilidad del proveedor (Microsoft)

TLS mínimo 1.3 y HTTPS Only

Certificado de `*.azurewebsites.net`

HSTS y demás encabezados

Algoritmos de cifrado e intercambio de claves

Deshabilitar FTP y autenticación básica

Soporte de criptografía post-cuántica

----------

## 9. Operación y mantenimiento

### Publicar un cambio

```bash
git pull         # traer cambios hechos desde GitHub o StackEdit
# ... modificar el código ...
npm run build    # verificar localmente
git add .
git commit       # asistente de Conventional Commits
git push         # dispara el despliegue automáticamente

```

> Agrupa en un mismo commit los cambios relacionados y verifícalos localmente antes de subirlos: cada push genera un despliegue.

El progreso se sigue en la pestaña **Actions** del repositorio. Para volver a desplegar sin cambios, usa **Run workflow** (`workflow_dispatch`).

### Recomendaciones de mejora

-   Migrar a una versión vigente de Angular para resolver vulnerabilidades de dependencias y eliminar `'unsafe-inline'` de `style-src` mediante _nonces_.
-   Usar un dominio propio con certificado administrado.
-   Agregar un entorno de pruebas con _deployment slots_ (requiere un plan de pago).

### Eliminar los recursos

Ver [README.md, sección 8](https://claude.ai/chat/README.md#8-gesti%C3%B3n-de-costos-y-limpieza).
<!--stackedit_data:
eyJoaXN0b3J5IjpbNzgzMzUxMTQzLDY5MTE2MjM1MCwxMTM0MD
U4NjQ3XX0=
-->