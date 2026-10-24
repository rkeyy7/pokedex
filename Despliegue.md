# PokeDex en Azure App Service

Tutorial para desplegar de forma segura la aplicación **PokeDex** (Angular) en **Microsoft Azure**, usando una cuenta **Azure for Students**, integración y despliegue continuos con **GitHub Actions** y encabezados de seguridad HTTP.

| | |
|---|---|
| **Aplicación en producción** | https://app-pokedex-prod-rkeyy7-e7edh5bah5aje7g2.mexicocentral-01.azurewebsites.net |
| **securityheaders.com** | **A+** |
| **SSL Labs** | **A** |

> **Créditos:** el código fuente de la aplicación fue desarrollado originalmente por [Keiler Mora](https://github.com/keilermora/pokedex-angular) y facilitado por PumasLab para este caso de uso.

Este documento es un **tutorial paso a paso para crear la cuenta en la nube y preparar los recursos**. Para publicar la aplicación, continúa con **[Despliegue.md](Despliegue.md)**.

---

## Tabla de contenido

1. [Requisitos previos](#1-requisitos-previos)
2. [Crear la cuenta Azure for Students](#2-crear-la-cuenta-azure-for-students)
3. [Conceptos básicos de Azure](#3-conceptos-básicos-de-azure)
4. [Revisar las restricciones de la suscripción](#4-revisar-las-restricciones-de-la-suscripción)
5. [Convención de nombres](#5-convención-de-nombres)
6. [Crear el grupo de recursos](#6-crear-el-grupo-de-recursos)
7. [Crear el App Service](#7-crear-el-app-service)
8. [Gestión de costos y limpieza](#8-gestión-de-costos-y-limpieza)

---

## 1. Requisitos previos

- Correo electrónico **institucional** de una universidad.
- Cuenta de **GitHub**.
- **Git** instalado (`git --version`).
- **Node.js 22 LTS** (`node -v`), para compilar y probar localmente.
- Un navegador web moderno.

---

## 2. Crear la cuenta Azure for Students

**Azure for Students** es una oferta de Microsoft para estudiantes universitarios que incluye:

- **$100 USD de crédito** válidos por **12 meses**.
- Acceso a servicios gratuitos de Azure dentro de sus límites.
- **No requiere tarjeta de crédito.**

### Pasos

1. Ingresa a **https://azure.microsoft.com/free/students**.
2. Haz clic en **Start free** (Comenzar gratis).
3. Inicia sesión con tu **correo institucional**. Si tu universidad usa Microsoft 365, usa esa misma cuenta.
4. Completa la **verificación académica**: Microsoft valida el dominio del correo institucional y, en algunos casos, envía un código al correo o solicita un documento adicional.
5. Verifica tu identidad con un **número de teléfono** (recibirás un código por SMS).
6. Acepta los términos del acuerdo de suscripción.
7. Al finalizar, serás redirigido al **Portal de Azure** (https://portal.azure.com) con la suscripción **Azure for Students** activa.

### Comprobación

- En el portal, busca **Subscriptions** (Suscripciones). Debe aparecer **Azure for Students** con estado **Active**.
- Consulta tu crédito en **https://www.microsoftazuresponsorships.com/Balance** o desde el **Education Hub** del portal.

> **Importante:** cuando se agota el crédito o se cumplen los 12 meses, la suscripción se **deshabilita** automáticamente. No se generan cobros, pero los recursos dejan de funcionar. Si sigues siendo estudiante, puedes renovar la oferta.

---

## 3. Conceptos básicos de Azure

| Concepto | Qué es | En este tutorial |
|---|---|---|
| **Suscripción** | Contrato de facturación que agrupa los recursos y define sus límites y políticas. | Azure for Students |
| **Grupo de recursos** | Contenedor lógico que agrupa los recursos de un proyecto. Al eliminarlo, se eliminan todos sus recursos. | `rg-pokedex-prod` |
| **Región** | Ubicación física del centro de datos donde se ejecutan los recursos. | Mexico Central |
| **App Service Plan** | El **"servidor"**: define la capacidad de cómputo, el sistema operativo y el **precio**. Es el recurso que se factura. | `asp-pokedex-prod` (Free F1, Windows) |
| **App Service (Web App)** | La **aplicación** que se ejecuta sobre el plan. Un plan puede alojar varias aplicaciones. | `app-pokedex-prod-rkeyy7` |

### ¿Por qué Azure App Service?

| Necesidad del proyecto | Cómo la cubre App Service |
|---|---|
| Publicar una aplicación Angular (archivos estáticos) | IIS sirve los archivos compilados. |
| HTTPS | Certificado incluido para `*.azurewebsites.net`. |
| Encabezados de seguridad personalizados | Configurables con un archivo `web.config` (Windows/IIS). |
| Despliegue automático | Integración nativa con GitHub Actions. |
| Costo | Plan gratuito **F1**. |

---

## 4. Revisar las restricciones de la suscripción

Las suscripciones **Azure for Students** tienen restricciones asignadas por Microsoft. Revisarlas **antes** de crear recursos evita errores durante la creación.

### 4.1 Regiones permitidas

1. En el portal, busca **Policy**.
2. Entra a **Assignments** (Asignaciones).
3. Abre la asignación **"Allowed resource deployment regions"**.
4. En **Parameters** verás la lista de regiones donde puedes crear recursos. En esta suscripción fueron:

```
brazilsouth, canadacentral, southcentralus, mexicocentral, westus3
```

Esta política la asigna Microsoft y no se puede modificar. Elige siempre una de estas regiones y verifica que el servicio que quieres usar esté disponible en ella.

> **Nota:** por esta razón se eligió **App Service**. Otros servicios, como Azure Static Web Apps, solo están disponibles en regiones que no forman parte de esta lista.

### 4.2 Cuotas por región

Azure limita la cantidad de recursos de cada tipo **por región y por suscripción**. La cuota del plan gratuito **F1** puede ser cero en algunas regiones.

1. En el portal, busca **Quotas**.
2. Selecciona **App Service** y filtra por región.
3. Elige una región permitida que tenga cuota disponible para **F1**.

En este tutorial se usó **Mexico Central**, que además es la región más cercana a Colombia de la lista (menor latencia).

> Si al crear el App Service aparece el mensaje *"Operation cannot be completed without additional quota"*, repite la creación en otra región permitida.

---

## 5. Convención de nombres

Se usa la convención recomendada por Microsoft (*Cloud Adoption Framework*): **prefijo del tipo de recurso + nombre del proyecto + entorno**.

| Recurso | Prefijo | Nombre |
|---|---|---|
| Grupo de recursos | `rg-` | `rg-pokedex-prod` |
| App Service Plan | `asp-` | `asp-pokedex-prod` |
| App Service | `app-` | `app-pokedex-prod-rkeyy7` |

El sufijo `-prod` identifica el **entorno de producción**. En proyectos con varios entornos coexistirían, por ejemplo, `rg-pokedex-dev` y `rg-pokedex-prod`.

---

## 6. Crear el grupo de recursos

1. En el portal, busca **Resource groups** → **+ Create**.
2. Configura:
   - **Subscription:** Azure for Students
   - **Resource group:** `rg-pokedex-prod`
   - **Region:** una de las regiones permitidas (por ejemplo, South Central US)
3. Haz clic en **Review + create** → **Create**.

> La región del grupo de recursos solo indica dónde se guardan sus **metadatos**. Los recursos que contiene pueden estar en otras regiones.

---

## 7. Crear el App Service

En el portal: **App Services → + Create → Web App**.

### Pestaña *Basics*

| Opción | Valor | Justificación |
|---|---|---|
| Subscription | Azure for Students | |
| Resource Group | `rg-pokedex-prod` | |
| Name | `app-pokedex-prod-rkeyy7` | Debe ser único en todo Azure. |
| Secure unique default hostname | **Enabled** | Ver explicación abajo. |
| Publish | Code | Se publican los archivos compilados, no un contenedor. |
| Runtime stack | Node 22 LTS | En Windows, IIS sirve la aplicación como archivos estáticos y Node no se ejecuta en producción. Se elige Node para que Azure genere un workflow de GitHub Actions basado en Node, necesario para compilar Angular. |
| Operating System | **Windows** | IIS permite configurar las rutas y los encabezados de seguridad con un archivo `web.config`. |
| Region | Mexico Central | Región permitida con cuota F1 disponible. |
| Pricing plan | Nuevo: `asp-pokedex-prod`, **Free F1** | Sin costo. |

### Pestaña *Deployment*

| Opción | Valor | Justificación |
|---|---|---|
| Continuous deployment | Desactivado | Se configura después desde el *Deployment Center* (ver [Despliegue.md](Despliegue.md)). |
| Basic authentication | **Disabled** | Deshabilita los métodos de despliegue con usuario y contraseña (FTP, *publish profile*). El despliegue usará una identidad administrada. |

### Pestaña *Monitoring*

| Opción | Valor | Justificación |
|---|---|---|
| Application Insights | Desactivado | No es necesario para este caso y evita consumir crédito. |

Finalmente, haz clic en **Review + create** → **Create** y espera a que termine el despliegue.

### *Secure unique default hostname*

Con esta opción, el dominio incluye un código aleatorio y la región:

```
app-pokedex-prod-rkeyy7-e7edh5bah5aje7g2.mexicocentral-01.azurewebsites.net
```

Esto previene el ataque de **subdomain takeover** (secuestro de subdominio): si algún día eliminas la aplicación, nadie más puede crear una con el mismo nombre y apropiarse de tu URL.

### Comprobación

1. Haz clic en **Go to resource**.
2. En **Overview** verifica:
   - **Status:** Running
   - **Operating System:** Windows
   - **Default domain:** tu dominio.
3. Abre el dominio en el navegador: verás la página por defecto de Azure, ya con **HTTPS**.

Los recursos están listos. El siguiente paso es configurar la seguridad de la plataforma y publicar la aplicación: **[Despliegue.md](Despliegue.md)**.

---

## 8. Gestión de costos y limpieza

- **Costo:** $0. El plan F1 es gratuito y GitHub Actions no tiene costo en repositorios públicos.
- **Limitaciones del plan F1:** no incluye *Always On* (la aplicación "se duerme" tras un periodo sin visitas y la primera carga tarda unos segundos) ni la plataforma de 64 bits. No afectan la seguridad ni el funcionamiento.
- **Consultar el crédito:** https://www.microsoftazuresponsorships.com/Balance

### Eliminar los recursos

Cuando ya no necesites la aplicación, elimina el grupo de recursos:

1. Portal → **Resource groups** → `rg-pokedex-prod`.
2. **Delete resource group**, escribe el nombre para confirmar → **Delete**.

Esto elimina en una sola operación el App Service Plan y el App Service.

<!--stackedit_data:
eyJoaXN0b3J5IjpbMTEzNDA1ODY0N119
-->