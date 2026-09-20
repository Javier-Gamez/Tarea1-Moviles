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
[`04-arquitectura-mcp.md`](04-arquitectura-mcp.md). En nuestra instalación usamos
únicamente el argumento de línea de comandos, que es el mecanismo primario y el más
simple de verificar.)

Toda ruta que el modelo intente usar en una llamada a herramienta se valida **dentro del
proceso servidor** contra esa lista de directorios permitidos, antes de tocar el disco.
Si la ruta resuelta (después de normalizar `..`, enlaces simbólicos, etc.) cae fuera de
los directorios permitidos, el servidor rechaza la operación y devuelve un error — el
modelo nunca llega a tocar el archivo. En este trabajo delimitamos el acceso a la
carpeta [`workspace/`](../workspace/) de este mismo repositorio, creada específicamente
para esta actividad.

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
