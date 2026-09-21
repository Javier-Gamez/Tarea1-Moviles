# 5. El servidor de sistema de archivos

## 5.1 "FS" no es parte del protocolo

Es importante dejarlo explícito porque es uno de los errores conceptuales más comunes:
**el sistema de archivos no es una primitiva de MCP ni forma parte de la
especificación.** MCP no sabe nada de archivos, directorios ni sistemas operativos; solo
define cómo un servidor publica y expone herramientas, recursos y prompts en general.

`@modelcontextprotocol/server-filesystem` es únicamente **uno de los servidores de
referencia** publicados por el equipo de MCP (junto con otros como el servidor de
memoria, el de Git, el de SQLite, el de Fetch/HTTP, etc.), pensado como ejemplo — y
como herramienta genuinamente útil — de qué se puede construir con el protocolo. Nada
impide construir un servidor MCP que no tenga absolutamente nada que ver con archivos
(por ejemplo, un servidor que consulte el clima o un servidor que controle luces
inteligentes). El protocolo es agnóstico al dominio; "FS" es solo una de sus
implementaciones posibles.

## 5.2 Herramientas que expone

El servidor de sistema de archivos de referencia expone, entre otras, las siguientes
herramientas (nombres tal como aparecen en `tools/list`):

| Herramienta | Qué hace |
|---|---|
| `list_directory` | Lista el contenido de un directorio (nombre y tipo: archivo o carpeta). |
| `list_directory_with_sizes` | Igual que la anterior, pero incluye el tamaño de cada archivo. |
| `directory_tree` | Devuelve la estructura recursiva de un directorio como árbol (JSON). |
| `read_text_file` | Lee el contenido de un archivo de texto. |
| `read_media_file` | Lee un archivo binario (imagen, etc.) codificado en base64 con su tipo MIME. |
| `read_multiple_files` | Lee varios archivos en una sola llamada. |
| `write_file` | Crea un archivo nuevo o sobreescribe uno existente con el contenido dado. |
| `edit_file` | Aplica una edición puntual (reemplazo de un fragmento) dentro de un archivo existente. |
| `create_directory` | Crea un directorio nuevo (incluyendo carpetas intermedias si no existen). |
| `move_file` | Mueve o renombra un archivo o directorio. |
| `search_files` | Busca archivos por patrón de nombre (glob) dentro del alcance permitido. |
| `get_file_info` | Devuelve metadatos de un archivo (tamaño, fechas, permisos). |
| `list_allowed_directories` | Devuelve la lista de directorios a los que el servidor tiene acceso — es la forma en que el propio modelo puede consultar cuál es su alcance. |

Estas herramientas cubren exactamente las operaciones que pide la actividad: listar,
leer, escribir/crear, modificar y buscar archivos.

## 5.3 Cómo se delimita el alcance

El servidor **exige** al menos un directorio permitido para poder iniciar: si no se le
indica ninguno, falla al arrancar. El mecanismo principal para delimitar ese alcance es
un **argumento de línea de comandos** al lanzar el proceso del servidor: se le pasa la
ruta (o rutas) de los directorios a los que tendrá acceso, por ejemplo:

```bash
npx -y @modelcontextprotocol/server-filesystem "C:\ruta\al\directorio\permitido"
```

(La especificación también contempla que el **cliente** le comunique al servidor una
lista de *roots* en tiempo de ejecución, la cual —cuando se provee— reemplaza el alcance
definido por línea de comandos; ver la nota sobre *roots* en
[`04-arquitectura-mcp.md`](04-arquitectura-mcp.md).

### Hallazgo práctico: los *roots* del cliente sobreescriben el argumento de línea de comandos

En la configuración inicial de este trabajo apuntamos el argumento de línea de comandos
al subdirectorio [`workspace/`](../workspace/), con la intención de que ese fuera el
único alcance del servidor. Al verificar con la herramienta `list_allowed_directories`
**después** de conectar el servidor desde Claude Code, el resultado real fue distinto:

```
Allowed directories:
C:\Users\JAVIER\Documents\Moviles-GH\Tareas\mcp-filesystem-actividad
```

Es decir, el alcance efectivo terminó siendo **toda la carpeta del proyecto**, no solo
`workspace/`. La causa es exactamente el mecanismo descrito arriba: Claude Code, como
cliente, sí soporta el protocolo de *roots* y le comunica al servidor, al conectarse, la
carpeta del proyecto donde se abrió la sesión como *root* — y esa notificación
**reemplaza por completo** el argumento de línea de comandos que configuramos
manualmente, tal como documenta el propio servidor de referencia ("client roots,
notified to the server, completely replace any server-side allowed directories when
provided").

Esto es una lección de seguridad real, no solo teórica: **delimitar el alcance
únicamente por línea de comandos no garantiza cuál va a ser el alcance final** — depende
también de qué le comunique el cliente al servidor, y eso hay que **verificarlo en la
práctica** (con `list_allowed_directories`), no darlo por hecho a partir de la
configuración escrita. En cualquier caso, el "directorio de trabajo delimitado" nunca
fue más amplio que la carpeta completa del proyecto
(`mcp-filesystem-actividad/`) — que sigue siendo una carpeta creada específicamente para
esta actividad, y **no** la carpeta de usuario completa ni la raíz del disco, cumpliendo
igualmente con lo que pide la actividad.

**Actualización:** al repetir esta misma verificación en sesiones posteriores (algunas
por la app de escritorio, otras por una terminal normal ejecutando `claude`), el
resultado de `list_allowed_directories` **no fue siempre el mismo**: unas veces devolvió
la carpeta completa del proyecto (como arriba) y otras veces devolvió únicamente
`workspace/`, tal como se había configurado por línea de comandos. No identificamos con
certeza qué determina cuál de los dos ocurre en cada sesión — parece depender de detalles
de cómo esa sesión específica negocia el protocolo de *roots* con el servidor. La
conclusión práctica se mantiene, y se refuerza: **no asumas el alcance, verifícalo con
`list_allowed_directories` antes de cada demostración.**

Toda ruta que el modelo intente usar en una llamada a herramienta se valida **dentro del
proceso servidor** contra esa lista de directorios permitidos, antes de tocar el disco.
Si la ruta resuelta (después de normalizar `..`, enlaces simbólicos, etc.) cae fuera de
los directorios permitidos, el servidor rechaza la operación y devuelve un error — el
modelo nunca llega a tocar el archivo, ni siquiera para verificar si existe (ver la
prueba del límite de seguridad en el [`README.md`](../README.md)).

## 5.4 Por qué existe ese límite y qué pasaría sin él

El límite existe porque **quien decide qué operación ejecutar es un modelo de lenguaje**,
no una línea de código escrita y revisada por una persona de antemano (ver
[`03-mcp-vs-api.md`](03-mcp-vs-api.md)). Un modelo puede equivocarse, puede ser engañado
por contenido malicioso dentro de un archivo que esté leyendo (inyección de
instrucciones, ver [`06-seguridad.md`](06-seguridad.md)), o simplemente puede
interpretar mal una instrucción ambigua del usuario. Delimitar el alcance a un único
directorio de trabajo —creado específicamente para la tarea, y no la carpeta de usuario
completa ni la raíz del disco— acota el daño posible a ese directorio, sin importar qué
decida hacer el modelo dentro de la conversación.

Sin ese límite, el mismo servidor tendría acceso potencial a **todo el sistema de
archivos** del usuario con los permisos del proceso que lo ejecuta: documentos
personales, credenciales guardadas en archivos de configuración (`.ssh`, `.aws`,
navegadores), código fuente de otros proyectos, etc. Combinado con el riesgo de
inyección de instrucciones, esto convertiría cualquier archivo que el modelo lea (por
ejemplo, el contenido de una página web pegada en la conversación, o un archivo
descargado) en una vía potencial para que un tercero, no el usuario, termine decidiendo
qué se lee, se sobrescribe o se borra en la máquina.
