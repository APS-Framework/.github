## Reglas para el agente

El servidor MCP de `aps-framework` es la fuente autoritativa de convenciones, contratos y patrones del proyecto. Las tools están disponibles como deferred tools bajo el prefijo `mcp_aps-framework_aps_framework__` (asumiendo que el servidor está configurado con la clave `aps-framework` en `mcp.json`).

**Antes de implementar, revisar o modificar cualquier cosa**, buscar las tools relevantes con `tool_search_tool_regex` usando el tema principal de la tarea como keyword: el nombre del paquete, el dominio técnico (`auth`, `cosmos`, `blob`, `messaging`…) o el tipo de artefacto (`manifest`, `docs`, `setup`).

Esto aplica sin excepción a: implementar features, revisar documentación, actualizar el manifest, escribir ejemplos, extender una librería o diagnosticar un error de configuración.

---

Instalar `@aps-framework/sdk-mcp-server` una sola vez por máquina para que todos los repositorios
puedan usar el MCP sin scripts auxiliares ni `npx`.

**Prerequisitos:** Node.js ≥ 18, GitHub CLI (`gh`) con sesión activa y scope `read:packages` habilitado (solo necesario para la instalación; el runtime solo necesita `repo` o `Contents: read`).

```powershell
# Verificar antes de instalar
gh auth status
gh auth refresh --scopes "read:packages"
```

```powershell
$token = gh auth token
$tempNpmrc = Join-Path $env:TEMP ("npmrc-" + [guid]::NewGuid().ToString() + ".txt")

Set-Content -Path $tempNpmrc -Value @(
    "@aps-framework:registry=https://npm.pkg.github.com",
    "//npm.pkg.github.com/:_authToken=$token",
    "always-auth=true"
)

$env:NPM_CONFIG_USERCONFIG = $tempNpmrc
npm install -g @aps-framework/sdk-mcp-server

Remove-Item $tempNpmrc -Force
```

Verificar que quedó disponible:

```powershell
Get-Command sdk-mcp-server
```

> Una vez instalado, los repositorios solo necesitan un `.vscode/mcp.json` mínimo con la URL de conexión:
>
> ```json
> {
>   "servers": {
>     "aps-framework": {
>       "type": "http",
>       "url": "http://127.0.0.1:7512/mcp?discovery={org}:{topic1,topic2}&exclude={org}/{nombre-repo-actual}"
>     }
>   }
> }
> ```
>
> El parámetro `discovery` acepta uno o varios topics de la misma org separados por comas (`org:topic1,topic2`). Para incluir repos de varias orgs, repetir el parámetro: `&discovery=otra-org:topic3`. Para excluir repos: `&exclude=org/repo`.
>
> **El repo consumidor debe excluirse siempre.** Si el repo desde el que se trabaja también tiene topic de discovery, aparecería en su propia lista de tools. Usar `&exclude=org/nombre-repo-actual` en la URL del `.vscode/mcp.json` de ese repo para evitarlo.
>
> Para actualizar a una nueva versión minor/patch: `npm update -g @aps-framework/sdk-mcp-server`
> Para un salto de major version: `npm install -g @aps-framework/sdk-mcp-server@latest`
> Tras cualquier actualización, reiniciar el servidor: `pm2 restart sdk-mcp-server` (si usa PM2) o matar el proceso y relanzarlo.

---

## Arranque del servidor

Hay dos opciones. **Cuando el desarrollador pida arrancar el servidor, preguntar antes cuál prefiere:**

> ¿Quieres arrancarlo solo para esta sesión (una terminal abierta) o dejarlo configurado para que arranque automáticamente con tu sesión de usuario (PM2)?

### Opción A — Solo esta sesión

```powershell
sdk-mcp-server
# sdk-mcp-server listening on http://127.0.0.1:7512/mcp
```

Dejar la terminal abierta. Hay que repetirlo en cada sesión de trabajo.

### Opción B — Arranque automático con PM2

```powershell
npm install -g pm2
npm install -g pm2-windows-startup
$bin = Join-Path (npm root -g) "@APS-Framework/sdk-mcp-server/bin/sdk-mcp-server.js"
pm2 start node --name sdk-mcp-server -- $bin
pm2 save
pm2-startup install
```

Verificación: `pm2 status`. A partir de ese momento el servidor arranca solo con Windows.

El servidor resuelve las credenciales de la sesión activa de `gh` al arrancar. No hay que pasar `GITHUB_TOKEN` manualmente. Los repos a los que el usuario no tenga acceso simplemente no aparecen en las tools.

---

## Onboarding por repo

Pasos para configurar un repo consumidor (o un repo de infraestructura que también consume otras libs):

**1. Arrancar el servidor** (ver sección "Arranque del servidor" para elegir entre sesión única o PM2).

**2. Crear `.vscode/mcp.json`** en la raíz del repo con la URL de conexión:

```json
{
  "servers": {
    "aps-framework": {
      "type": "http",
      "url": "http://127.0.0.1:7512/mcp?discovery={org}:{topic1,topic2}&exclude={org}/{nombre-repo-actual}"
    }
  }
}
```

Reemplazar `{org}` por la organización de GitHub, `{topic1,topic2}` por uno o varios topics de discovery de esa org separados por comas, y `{org}/{nombre-repo-actual}` por el `org/repo` real del repositorio en el que se está trabajando. Para incluir repos de varias orgs, repetir el parámetro: `&discovery=otra-org:topic3`. El repo actual debe excluirse siempre con `&exclude=org/repo` para evitar que aparezca en su propia lista de tools.

**3. Conectar desde VS Code:** abrir la paleta de comandos → `MCP: List Servers` → `aps-framework` → `Start`.

**4. Verificar:** abrir el chat en modo `Agent` y preguntar qué tools están disponibles.

---

## Comportamiento del servidor MCP

**El servidor es un proceso HTTP persistente.** A diferencia del transporte `stdio`, el servidor no lo gestiona VS Code: hay que arrancarlo manualmente una vez por sesión de trabajo o usar PM2 para arranque automático (ver sección "Arranque del servidor").

El servidor obtiene las credenciales de la sesión activa de `gh`. Los repos a los que el usuario no tenga acceso no aparecen en las tools.

**El servidor descubre repos al crear la sesión.** Cuando un cliente conecta con una URL nueva, el servidor obtiene los manifests de GitHub y los cachea 5 minutos. Si se añade un repo nuevo después, reconectar el servidor desde VS Code (`MCP: List Servers` → `aps-framework` → `Reconnect`) fuerza un redescubrimiento.

**El repo consumidor debe excluirse explícitamente.** No hay detección automática del repo actual. Si el repo en el que se trabaja también tiene topic de discovery, aparecerá en su propia lista de tools a menos que se excluya con `&exclude=org/repo` en la URL del `.vscode/mcp.json`.

---

## Reglas para `mcp-manifest.json`

**Codificación: UTF-8 sin BOM obligatorio.** El servidor hace `JSON.parse()` del contenido y falla silenciosamente si el archivo tiene BOM. El repo queda excluido de las tools sin ningún error visible.

- En VS Code: guardar con `Change File Encoding` → `UTF-8` (no `UTF-8 with BOM`)
- En PowerShell: usar `Set-Content -Encoding UTF8NoBOM` o `[System.IO.File]::WriteAllText(path, content, [System.Text.UTF8Encoding]::new($false))`
- Verificar con: `Format-Hex mcp-manifest.json | Select-Object -First 1` — los primeros bytes deben ser `7B 0A` (`{` + nueva línea), nunca `EF BB BF`

Para diagnosticar si un repo no aparece en las tools, ejecutar desde el directorio del `sdk-mcp-server` instalado:

```powershell
cd (Join-Path (npm root -g) "@aps-framework/sdk-mcp-server")
node -e "
import('./dist/github-client.js').then(async m => {
  const client = new m.GitHubClient(process.env.GITHUB_TOKEN, 300);
  const content = await client.getFileContent('{tu-org}', '{NOMBRE-REPO}', 'mcp-manifest.json');
  try { JSON.parse(content); console.log('OK'); } catch(e) { console.log('ERROR:', e.message); }
})
"
```