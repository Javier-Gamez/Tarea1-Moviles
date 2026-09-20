# 3. MCP frente a una API (punto obligatorio)

## 3.1 ¿Qué es una API?

Una **API** (*Application Programming Interface*) es un contrato entre programas: una
especificación fija de qué *endpoints* existen, qué parámetros acepta cada uno, qué
forma tiene la respuesta y qué errores puede devolver. Ese contrato normalmente se
publica como documentación (OpenAPI/Swagger, documentación en prosa, etc.).

El punto clave es **quién decide**: la persona que desarrolla la integración lee esa
documentación *antes* de escribir código, decide qué endpoint necesita para su caso de
uso, arma la petición (URL, método HTTP, cabeceras, cuerpo) y escribe el código que
interpreta la respuesta y decide qué hacer con ella. Esa decisión — qué se llama, con
qué datos, y en qué momento del flujo del programa — está **codificada de antemano** en
el software: si la persona desarrolladora no escribió una rama de código que llame a
`GET /files/{id}`, el programa nunca va a llamar a `GET /files/{id}`, sin importar qué le
pida el usuario final en tiempo de ejecución.

Ejemplo: una app de clima integra la API de un proveedor meteorológico. El desarrollador
decidió, al escribir el código, que cuando el usuario abra la pantalla principal se
llamará a `GET /forecast?lat=...&lon=...`. Esa decisión no cambia según lo que el
usuario escriba en un campo de texto; el campo de texto ni siquiera existe, hay botones
y pantallas fijas.

## 3.2 ¿Qué es MCP?

**Model Context Protocol (MCP)** es un protocolo *abierto* (no propietario de un solo
fabricante), basado en mensajes **JSON-RPC 2.0**, mediante el cual un **servidor**
publica un catálogo de capacidades — típicamente *herramientas* (tools), pero también
*recursos* y *plantillas de prompt* — cada una con su nombre, su descripción en lenguaje
natural y el esquema (JSON Schema) de los parámetros que acepta.

La diferencia fundamental frente a una API tradicional es **cuándo y quién decide qué se
invoca**:

- Con una API, la decisión de qué endpoint llamar está fija en el código, escrita por el
  desarrollador antes de que el programa se ejecute.
- Con MCP, el **cliente** (la aplicación host, p. ej. Claude Desktop, Claude Code, un
  editor) se conecta al servidor y le pregunta en tiempo de ejecución "¿qué herramientas
  tienes disponibles?" (`tools/list`). El servidor responde con la lista completa,
  incluyendo las descripciones. Esa lista se entrega al **modelo** como parte de su
  contexto. Es el **modelo**, durante la conversación, quien decide —en tiempo real,
  según lo que el usuario pidió en lenguaje natural— cuál de esas herramientas invocar,
  con qué argumentos, y en qué orden. El desarrollador de la aplicación cliente nunca
  escribió "si el usuario pide leer un archivo, llama a `read_file`"; el modelo llega a
  esa decisión por sí mismo a partir de la descripción de la herramienta y del pedido
  del usuario.

En términos de protocolo: MCP define un *handshake* de inicialización, una forma
estándar de descubrir capacidades (`tools/list`, `resources/list`, `prompts/list`) y una
forma estándar de invocarlas (`tools/call`) — todo como mensajes JSON-RPC 2.0
intercambiados sobre un transporte (stdio o HTTP, ver
[`04-arquitectura-mcp.md`](04-arquitectura-mcp.md)).

## 3.3 Tabla comparativa

| Criterio | API tradicional (REST/RPC clásico) | MCP |
|---|---|---|
| **Quién decide qué se invoca** | La persona desarrolladora, en tiempo de diseño/código. La lógica "si pasa X, llama a Y" está escrita de antemano. | El modelo, en tiempo de ejecución, a partir de la petición del usuario en lenguaje natural y de las descripciones publicadas por el servidor. |
| **Cómo se descubren las capacidades** | Leyendo documentación externa (Swagger/OpenAPI, un PDF, un portal de desarrolladores) *antes* de escribir código. El descubrimiento es un paso humano, offline. | El cliente pregunta al servidor en tiempo real (`tools/list`, `resources/list`, `prompts/list`); el catálogo se obtiene en runtime y se le pasa al modelo como parte del contexto de la conversación. |
| **Acoplamiento cliente–servicio** | Alto: el cliente está compilado/escrito contra una versión específica del contrato (nombres de endpoints, forma del JSON). Cambiar el contrato rompe al cliente si no se actualiza el código. | Bajo: el cliente es genérico — sabe hablar JSON-RPC y "listar + invocar herramientas", pero no tiene lógica específica de ningún servidor en particular. El mismo cliente (p. ej. Claude Desktop) puede hablar con un servidor de archivos, uno de bases de datos o uno de Git sin que nadie reprograme el cliente. |
| **Formato de los mensajes** | Variable según la API: JSON, XML, form-encoded, protobuf, gRPC, GraphQL, etc. Cada API define su propio formato de petición/respuesta. | Estandarizado: JSON-RPC 2.0 sobre uno de los transportes definidos por la especificación (stdio o Streamable HTTP). Todas las implementaciones de MCP hablan el mismo formato base. |
| **Autenticación y consentimiento** | Normalmente API keys, OAuth o tokens definidos por cada proveedor; el consentimiento del usuario (si existe) ocurre una vez, al conectar la cuenta, y después el programa actúa sin volver a preguntar en cada llamada. | El protocolo contempla que el **host** exija consentimiento humano explícito antes de invocar una herramienta (no solo al conectar el servidor, sino potencialmente en cada acción sensible), porque quien decide invocar la herramienta es un modelo, no una línea de código auditada de antemano por un humano. La autenticación hacia el recurso de fondo (API key de terceros, etc.) la sigue manejando el servidor MCP, MCP no la reinventa. |
| **Reutilización entre aplicaciones distintas** | Baja: cada aplicación que quiera consumir la API tiene que implementar su propio cliente contra ese contrato específico. | Alta: un mismo servidor MCP (p. ej. el servidor de sistema de archivos) puede conectarse, sin cambios, a cualquier cliente que hable MCP — Claude Desktop, Claude Code, VS Code, Cursor, Zed, etc. — porque el protocolo, no la aplicación, define cómo se descubren e invocan las capacidades. |

## 3.4 Lo que MCP **no** es (errores frecuentes a evitar)

- **MCP no sustituye a las APIs ni las vuelve obsoletas.** Son cosas de naturaleza
  distinta: una API es un contrato de bajo nivel entre programas; MCP es un protocolo de
  *más alto nivel*, pensado específicamente para que un modelo de lenguaje descubra e
  invoque capacidades. De hecho, **casi todo servidor MCP existe envolviendo una API, un
  SDK o un recurso que ya existía antes** (el servidor de sistema de archivos envuelve
  llamadas del sistema operativo; un servidor de GitHub envuelve la API REST de GitHub;
  un servidor de Slack envuelve la API de Slack). MCP es una capa de adaptación encima
  de esos recursos que los hace *descubribles y utilizables por un modelo* — no un
  reemplazo de la lógica que hay debajo.
- **MCP no es tecnología propietaria de un solo proveedor.** Aunque Anthropic publicó la
  especificación original en noviembre de 2024, es un protocolo **abierto**, con
  especificación pública en GitHub, y ha sido adoptado por múltiples proveedores y
  herramientas de distintas compañías (ver [`07-casos-de-uso.md`](07-casos-de-uso.md)):
  OpenAI, Google, Microsoft (VS Code / Copilot), Cursor, entre otros, han añadido soporte
  para MCP en sus propios productos.
- **El modelo nunca "llama directamente" a la API subyacente.** Sigue habiendo una capa
  de por medio (el servidor MCP) que decide cómo traducir "invoca la herramienta
  `read_file` con este argumento" en la llamada real de sistema o de API que corresponda,
  aplicando sus propias reglas de seguridad y alcance.

## Referencias de esta sección

Ver lista completa de referencias en formato APA en el [`README.md`](../README.md).
