# Configuración utilizada

## `mcp.json`

Copia exacta del archivo `.mcp.json` generado en la raíz del repositorio al registrar el
servidor con el CLI de Claude Code (sin credenciales — el servidor de sistema de
archivos no requiere ninguna):

```bash
claude mcp add filesystem --scope project -- npx -y @modelcontextprotocol/server-filesystem "<ruta-absoluta-al-directorio-workspace>"
```

`--scope project` hace que la configuración se guarde en `.mcp.json` en la raíz del
repositorio (versionado con el proyecto) en vez de en la configuración global del
usuario, de modo que cualquiera que clone este repositorio y tenga Claude Code instalado
puede reproducir exactamente el mismo servidor sin volver a escribir el comando — solo
tiene que aprobar el servidor la primera vez que abra una sesión en esta carpeta (ver
`README.md` en la raíz, sección de instalación).

El único dato que cambia entre máquinas es la ruta absoluta al directorio `workspace/`,
que no es secreta: es simplemente la ubicación local del propio repositorio.
