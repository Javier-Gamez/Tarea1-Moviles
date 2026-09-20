# 4. Arquitectura de MCP

> **Versión de la especificación consultada para este documento:** `2026-07-28`
> (marcada como *current* — la versión estable vigente — en el sitio oficial
> `modelcontextprotocol.io` en la fecha en que se escribió este documento, **20 de
> septiembre de 2026**). La especificación usa versionado por fecha (`YYYY-MM-DD`) y se
> actualiza con relativa frecuencia manteniendo compatibilidad hacia atrás cuando es
> posible; por eso este documento indica explícitamente a qué revisión se refiere. En
> algunos puntos se señala además cuándo el comportamiento descrito difiere de
> revisiones anteriores muy usadas en la práctica (p. ej. `2025-06-18`), porque muchos
> servidores y clientes en producción —incluido el servidor de sistema de archivos que
> instalamos en este trabajo— todavía se comportan según esas versiones previas.

## 4.1 Modelo host / cliente / servidor

MCP define tres roles, todos corriendo como procesos separados:

- **Host**: la aplicación que el usuario tiene abierta y con la que interactúa
  directamente. Es quien crea y administra las conexiones cliente-servidor, aplica las
  políticas de seguridad y consentimiento, y coordina la integración con el modelo de
  lenguaje. En nuestra implementación, el host es **Claude Code** (la app de escritorio
  / CLI de Anthropic).
- **Cliente**: un componente *dentro* del host que mantiene una conexión 1:1 con **un**
  servidor específico. Un mismo host puede crear varios clientes, cada uno hablando con
  un servidor distinto (por ejemplo, un cliente hacia el servidor de archivos y otro
  hacia un servidor de Git, simultáneamente). El cliente es quien envía las peticiones
  JSON-RPC y le adjunta la versión de protocolo y las capacidades soportadas.
- **Servidor**: un proceso independiente (local o remoto) que expone un conjunto
  específico de capacidades — en nuestro caso, operaciones sobre el sistema de
  archivos — a través del protocolo. En este trabajo, el servidor es
  `@modelcontextprotocol/server-filesystem`, ejecutado como subproceso local.

En el ejemplo que instalamos: **Claude Code = host**, el proceso interno que gestiona la
conexión al servidor de archivos = **cliente**, y el proceso Node.js que corre
`server-filesystem` = **servidor**. El modelo (Claude) no es ninguno de los tres roles
del protocolo: es el consumidor del contexto que el host le arma a partir de lo que el
cliente obtiene del servidor.

Es un error común confundir el cliente con el servidor, o decir que "el modelo accede
directamente al disco". Ninguna de las dos cosas es correcta: el modelo solo ve texto
(la lista de herramientas y los resultados que el cliente le entrega); quien de verdad
toca el sistema de archivos es el proceso servidor, con los permisos que el usuario le
haya configurado.

## 4.2 Primitivas del lado del servidor

Un servidor MCP puede exponer tres tipos de primitivas:

1. **Tools (herramientas)**: funciones que el **modelo** puede decidir ejecutar. Cada
   una tiene un nombre, una descripción en lenguaje natural y un esquema JSON de sus
   parámetros. Es la primitiva más usada en la práctica y la que usa el servidor de
   sistema de archivos (`read_text_file`, `write_file`, `list_directory`, etc.).
2. **Resources (recursos)**: datos o contenido que el servidor pone a disposición del
   **usuario o del modelo** como contexto — por ejemplo, el contenido de un archivo
   identificado por una URI (`file:///...`), un registro de una base de datos, o el
   resultado de una consulta. A diferencia de una herramienta, un recurso no "hace" algo
   con efectos secundarios: solo expone información direccionable por URI, que el host
   puede decidir adjuntar al contexto de la conversación (con o sin intervención del
   modelo).
3. **Prompts (plantillas de prompt)**: flujos o mensajes predefinidos y parametrizables
   que el servidor ofrece, pensados para que el **usuario** los invoque explícitamente
   (por ejemplo, desde un menú), no para que el modelo decida usarlos por su cuenta. Sirven
   para estandarizar tareas comunes contra ese servidor (p. ej. una plantilla
   "resume los cambios de este directorio").

Un documento de investigación incompleto suele mencionar solo las *tools* y omitir
*resources* y *prompts* — los tres son primitivas de la especificación, no solo las
herramientas.

## 4.3 Primitivas del lado del cliente

El cliente (dentro del host) también puede exponer capacidades *hacia* el servidor, para
que el servidor pueda pedirle cosas al humano o al modelo a través de él:

- **Elicitation**: le permite a un servidor pedirle al cliente que solicite información
  adicional al usuario en medio de una operación (por ejemplo, pedir confirmación o un
  dato faltante antes de continuar una tarea), sin que el servidor tenga que construir
  su propia interfaz de usuario.
- **Sampling**: le permite a un servidor pedirle al cliente que genere una respuesta del
  modelo de lenguaje del host, de modo que un servidor pueda apoyarse en capacidades de
  IA sin tener que integrar su propio modelo.
- **Roots**: una forma estándar de que el cliente le indique a un servidor qué
  directorios o archivos son relevantes para la sesión actual (por ejemplo, la carpeta
  del proyecto abierto). **Nota sobre la versión consultada**: en la revisión
  `2026-07-28` de la especificación, *roots* aparece marcada como **deprecated**
  (obsoleta) a favor de pasar rutas directamente como parámetros de herramienta, URIs de
  recurso o configuración del propio servidor — aunque se mantiene en la especificación
  por política de retrocompatibilidad durante al menos doce meses. En revisiones
  anteriores muy usadas (p. ej. `2025-06-18`) *roots* era una primitiva estándar no
  obsoleta, y es el mecanismo que documentación y tutoriales anteriores describen. En la
  práctica, en este trabajo el alcance del servidor de archivos no se configura por
  *roots* sino por **argumentos de línea de comandos al iniciar el servidor** (ver
  [`05-servidor-filesystem.md`](05-servidor-filesystem.md)), que es el mecanismo primario
  y el que usa nuestra instalación.

## 4.4 Transportes

La especificación define dos transportes estándar (además de permitir transportes
personalizados):

- **stdio**: el cliente **lanza el servidor como proceso hijo local** y se comunica con
  él escribiendo y leyendo mensajes JSON-RPC delimitados por saltos de línea sobre la
  entrada/salida estándar (`stdin`/`stdout`) de ese proceso. Es el transporte que usamos
  en este trabajo: Claude Code ejecuta `npx @modelcontextprotocol/server-filesystem
  <directorio>` como subproceso y habla con él por stdio. No hay red de por medio; el
  servidor vive y muere con el proceso del cliente.
- **Streamable HTTP**: pensado para servidores **remotos**. Cada mensaje del cliente se
  envía como una petición HTTP `POST` a un único endpoint MCP; la respuesta puede llegar
  como un objeto JSON simple o como un flujo `Server-Sent Events` (SSE) cuando el
  servidor necesita enviar varios mensajes o notificaciones para una misma petición. Este
  transporte es el que se usa cuando el servidor no corre en la misma máquina que el
  cliente (por ejemplo, un servidor MCP expuesto como servicio en la nube).

## Referencias de esta sección

Ver lista completa de referencias en formato APA en el [`README.md`](../README.md),
incluyendo la cita de la especificación en la versión `2026-07-28`.
