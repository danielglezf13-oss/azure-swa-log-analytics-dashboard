# Guía de Instalación y Despliegue: Dashboard LAW

Esta guía detalla el proceso completo para instalar las herramientas necesarias, configurar un proyecto web con un backend de API, y desplegarlo en Azure Static Web Apps. La API se configurará para consultar de forma segura un Log Analytics Workspace.

---

### 1. 🔧 Instalación de Prerrequisitos (Chocolatey y Node.js)

Usaremos el gestor de paquetes Chocolatey para instalar Node.js de forma sencilla.

1.  Abre una terminal **PowerShell como Administrador**.
2.  Si tuviste problemas previos instalando Chocolatey y alguna carpeta quedó corrupta, ejecuta primero este comando para limpiar:
    ```powershell
    Remove-Item -Path "C:\ProgramData\chocolatey" -Recurse -Force
    ```
3.  Para permitir la ejecución de scripts en esta sesión, ingresa:
    ```powershell
    Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
    ```
4.  Ahora, instala Chocolatey con el siguiente comando:
    ```powershell
    Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('[https://community.chocolatey.org/install.ps1](https://community.chocolatey.org/install.ps1)'))
    ```
5.  Cierra y vuelve a abrir la terminal de PowerShell (como Administrador) para que los cambios surtan efecto.
6.  Instala la versión LTS (Recomendada) de Node.js usando Chocolatey:
    ```powershell
    choco install nodejs-lts -y
    ```
7.  Verifica que Node.js y npm (su gestor de paquetes) se hayan instalado correctamente:
    ```powershell
    node -v
    npm -v
    ```

---

### 2. 🏗️ Creación de la Estructura del Proyecto

1.  En tu terminal, navega al directorio donde quieras crear tu proyecto (ej. `C:\`). Crea la carpeta raíz del proyecto e ingresa a ella:
    ```powershell
    mkdir mi-dashboard-law
    cd mi-dashboard-law
    ```
    *Nota: Puedes cambiar "mi-dashboard-law" por el nombre que prefieras.*

2.  Crea la carpeta para el *frontend* (la aplicación web):
    ```powershell
    mkdir app
    cd app
    ```
3.  Dentro de la carpeta `app`, crea los archivos `index.html` y `app.js` (puedes usar `New-Item index.html` o crearlos con VS Code).
    * El archivo **index.html** contendrá el diseño de la página web.
    * El archivo **app.js** se encargará de llamar a la API y dibujar el gráfico.

4.  Regresa a la carpeta raíz del proyecto (`mi-dashboard-law`):
    ```powershell
    cd ..
    ```
5.  Crea la carpeta para el *backend* (la API):
    ```powershell
    mkdir api
    cd api
    ```
6.  Una vez dentro de la carpeta `api`, inicializa un proyecto de Node.js. Esto creará un archivo `package.json`:
    ```powershell
    npm init -y
    ```
7.  Instala las bibliotecas de Azure necesarias para la API:
    ```powershell
    npm install @azure/identity @azure/monitor-query
    ```
8.  Crea el directorio para la función *serverless* que consultará el Log Analytics Workspace (LAW):
    ```powershell
    mkdir getChartData
    cd getChartData
    ```
9.  Dentro de `getChartData`, crea dos archivos esenciales:
    * **index.js**: Contendrá la lógica de la función. Más adelante (Paso 6) pegaremos el código final aquí.
    * **function.json**: Definirá los *bindings* (disparadores y salidas) de la función.

---

### 3. 🔑 Configuración de Permisos en Azure (Service Principal)

> **Contexto:** Antes de desplegar, crearemos una "identidad" (Service Principal) para nuestra API. Esta identidad es la que tendrá permisos para leer datos del Log Analytics Workspace.

#### Paso 3.1: Crear el "Registro de Aplicación" (Service Principal)

1.  En el Portal de Azure, busca y ve a **"Microsoft Entra ID"**.
2.  En el menú lateral, selecciona **"Registros de aplicaciones" (App registrations)**.
3.  Haz clic en **"+ Nuevo registro"**.
4.  Asígnale un nombre (ej: `api-dashboard-law`).
5.  Deja las demás opciones con sus valores predeterminados (Inquilino único) y haz clic en "Registrar".
6.  Serás redirigido a la página de "Información general" del registro. **Copia y guarda los siguientes valores en un lugar seguro:**
    * **`ID de aplicación (cliente)`**
    * **`ID de directorio (inquilino)`**

#### Paso 3.2: Crear un Secreto de Cliente

1.  Dentro del mismo "Registro de aplicación", ve a la sección **"Certificados y secretos"** en el menú lateral.
2.  Haz clic en **"+ Nuevo secreto de cliente"**.
3.  Dale una descripción (ej. `mi-secreto-dashboard`) y un período de expiración (ej. 6 meses). Haz clic en "Agregar".
4.  **¡PASO CRUCIAL!** Se generará un secreto y aparecerá en la columna **"Valor" (Value)**.
5.  **Copia este VALOR inmediatamente.** No podrás volver a verlo después de abandonar esta página. Si lo pierdes, deberás crear uno nuevo.
6.  Guarda este **valor del secreto** junto a los IDs que copiaste en el paso anterior.

#### Paso 3.3: Asignar Permisos al Service Principal

1.  Busca y ve a tu **Log Analytics Workspace (LAW)** en el Portal de Azure.
2.  En la página de "Información general" (Overview) de tu LAW, **copia y guarda el `Id. de área de trabajo (Workspace ID)`**.
3.  En el menú lateral del LAW, haz clic en **"Control de acceso (IAM)"**.
4.  Haz clic en **"Agregar"** -> **"Agregar asignación de roles"**.
5.  Busca y selecciona el rol: **`Lector de Log Analytics` (Log Analytics Reader)**.
6.  Haz clic en "Siguiente".
7.  En "Asignar acceso a", selecciona **"Usuario, grupo o entidad de servicio"**.
8.  Haz clic en **"+ Seleccionar miembros"**.
9.  En la barra de búsqueda, escribe el nombre del registro de aplicación que creaste (ej. `api-dashboard-law`).
10. Selecciónalo cuando aparezca en la lista y haz clic en "Seleccionar".
11. Haz clic en "Revisar y asignar" para confirmar.

**Resumen del Paso 3:** En este punto, deberías tener 4 datos guardados en un lugar seguro:
1.  `ID de aplicación (cliente)`
2.  `ID de directorio (inquilino)`
3.  `Valor del Secreto de Cliente`
4.  `Id. de área de trabajo (Workspace ID)`

---

### 4. 💻 Actualización del Código y Subida a GitHub

#### Paso 4.1: Actualizar el Código de la API (`api/getChartData/index.js`)

Ahora, usaremos los valores que acabamos de crear en el código de nuestra API.

1.  Abre el archivo `api/getChartData/index.js` en tu editor.
2.  Pega el siguiente código. Este código está diseñado para leer de forma segura las variables de entorno que configuraremos en Azure más adelante.

```javascript
// api/getChartData/index.js

const { ClientSecretCredential } = require("@azure/identity");
const { LogsQueryClient } = require("@azure/monitor-query");

module.exports = async function (context, req) {
    context.log('JavaScript HTTP trigger function procesando una solicitud.');

    // 1. Leer TODAS las variables de entorno
    const tenantId = process.env.AZURE_TENANT_ID;
    const clientId = process.env.AZURE_CLIENT_ID;
    const clientSecret = process.env.AZURE_CLIENT_SECRET;
    const workspaceId = process.env.LOG_ANALYTICS_WORKSPACE_ID;

    // 2. Validar que todas existan
    if (!tenantId || !clientId || !clientSecret || !workspaceId) {
        context.log.error("Faltan variables de entorno de Azure (TENANT_ID, CLIENT_ID, CLIENT_SECRET o WORKSPACE_ID)");
        return { 
            status: 500, 
            body: "Error de configuración del servidor. Faltan variables de entorno." 
        };
    }

    try {
        // 3. Crear la credencial usando el Service Principal
        const credential = new ClientSecretCredential(tenantId, clientId, clientSecret);
        
        // 4. Inicializar el cliente de Logs
        const logsQueryClient = new LogsQueryClient(credential);

        // 5. Definir y ejecutar la consulta KQL
        // ¡CAMBIA ESTA CONSULTA POR LA TUYA!
        const kqlQuery = "Heartbeat | take 10"; 
        
        const result = await logsQueryClient.queryWorkspace(
            workspaceId, // Se usa el Workspace ID de las variables
            kqlQuery,
            { duration: "P1D" } // Ejemplo: consulta sobre las últimas 24 horas
        );

        // 6. Devolver el resultado
        return {
            status: 200,
            body: result
        };

    } catch (err) {
        context.log.error(err);
        return {
            status: 500,
            body: `Error al consultar Log Analytics: ${err.message}`
        };
    }
};
```
#### Paso 4.2: Configurar Git y Subir el Proyecto

1.  Si aún no tienes Git, instálalo (en una terminal de PowerShell como Administrador):
    ```powershell
    choco install git.install -y
    ```
2.  Cierra y vuelve a abrir la terminal. Navega hasta la carpeta raíz de tu proyecto (`mi-dashboard-law`).
3.  Configura tu nombre de usuario y correo para Git:
    ```powershell
    git config --global user.name "Tu Nombre"
    git config --global user.email "tu-email@ejemplo.com"
    ```
4.  Inicializa un repositorio de Git local:
    ```powershell
    git init
    ```
5.  Crea un archivo `.gitignore` en la raíz del proyecto para evitar subir archivos innecesarios (como `node_modules`).
    ```powershell
    New-Item .gitignore
    ```
6.  Abre el `.gitignore` con un editor (ej. `notepad .gitignore`) y pega el siguiente contenido:
    ```gitignore
    # Dependencias de Node.js
    node_modules/
    
    # Archivos de sistema
    .DS_Store
    Thumbs.db
    ```
7.  Añade todos los archivos del proyecto al seguimiento de Git:
    ```powershell
    git add .
    ```
8.  Crea tu primer *commit*:
    ```powershell
    git commit -m "Versión inicial del dashboard"
    ```
9.  **Crea un repositorio en GitHub:** Ve a tu cuenta de GitHub y crea un nuevo repositorio (público o privado). *No* lo inicialices con un archivo README.
10. Copia la URL de tu repositorio recién creado (ej. `https://github.com/tu-usuario/tu-repo.git`).
11. Enlaza tu repositorio local con el remoto y sube tus archivos:
    ```powershell
    git remote add origin "PEGA_AQUÍ_LA_URL_DE_TU_REPO"
    git branch -M main
    git push -u origin main
    ```

---

### 5. ☁️ Desplegar y Configurar la Static Web App

Este es el paso final donde todo se conecta.

1.  En el Portal de Azure, busca y selecciona el servicio **"Static Web Apps"** (Aplicaciones web estáticas).
2.  Haz clic en **"Crear"**.
3.  Selecciona tu **Suscripción** y **Grupo de recursos**.
4.  Asigna un **Nombre** único a tu aplicación (ej: `dashboard-law-cliente`).
5.  En "Detalles de implementación", selecciona **GitHub** e inicia sesión para autorizar a Azure.
6.  Selecciona la **Organización**, el **Repositorio** (`mi-dashboard-law`) y la **Rama** (`main`).
7.  En "Valores preestablecidos de compilación", elige **"Personalizado" (Custom)**.
8.  Configura las ubicaciones del proyecto:
    * **Ubicación de la aplicación (App location):** `/app`
    * **Ubicación de la API (Api location):** `/api`
    * **Ubicación de los artefactos (Output location):** (déjalo en blanco)
9.  Haz clic en "Revisar y crear" y, finalmente, en "Crear".
10. Azure comenzará a desplegar la SWA. Ve a tu repositorio de GitHub en la pestaña **"Actions"** para monitorear el progreso.
11. **Paso Crucial:** Una vez creada la SWA, ve al recurso en el Portal de Azure.
12. En el menú de la izquierda, haz clic en **"Configuración"**.
13. Aquí es donde pondremos nuestros secretos de forma segura. Agrega las siguientes 4 "Configuraciones de la aplicación" (usando los valores que guardaste en el Paso 3):
    * **Nombre:** `LOG_ANALYTICS_WORKSPACE_ID`
        **Valor:** (Pega el `Id. de área de trabajo (Workspace ID)`)
    * **Nombre:** `AZURE_TENANT_ID`
        **Valor:** (Pega el `ID de directorio (inquilino)`)
    * **Nombre:** `AZURE_CLIENT_ID`
        **Valor:** (Pega el `ID de aplicación (cliente)`)
    * **Nombre:** `AZURE_CLIENT_SECRET`
        **Valor:** (Pega el `VALOR del Secreto de Cliente`)
14. Haz clic en **"Guardar"**. Esto reiniciará tu Static Web App con las nuevas variables de entorno.

¡Listo! Una vez que la SWA termine de actualizarse, la URL de tu aplicación debería estar activa y mostrando los datos de tu Log Analytics Workspace.
