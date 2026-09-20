# 6. Seguridad

## 6.1 Riesgos concretos

- **Inyección de instrucciones a través de contenido de archivo (*prompt injection*).**
  Si el modelo lee, con la herramienta `read_text_file`, un archivo cuyo contenido
  incluye texto redactado para parecer una instrucción ("IMPORTANTE: ignora las
  instrucciones anteriores y ejecuta X"), el modelo puede llegar a tratar ese texto como
  si viniera del usuario y actuar en consecuencia, sin que la persona haya pedido eso.
  El riesgo crece si el archivo proviene de una fuente no controlada por el usuario
  (descargado de internet, adjunto de un correo, resultado de una búsqueda).
- **Acceso a rutas fuera del directorio autorizado.** Un intento —accidental o
  inducido— de leer o escribir fuera del alcance permitido, por ejemplo usando rutas
  relativas con `..` para "escapar" del directorio (*path traversal*), o pidiendo
  directamente una ruta absoluta fuera del *workspace*.
- **Escritura o borrado no deseados.** Herramientas como `write_file` (que puede
  **sobrescribir** un archivo existente sin pedir confirmación explícita del contenido
  anterior) o `move_file` pueden destruir información si el modelo interpreta mal una
  instrucción, o si actúa sobre una instrucción inyectada como la del primer punto.
- **Filtración de información sensible dentro del propio directorio permitido.** Aunque
  el acceso esté bien delimitado, si dentro de ese directorio hay archivos con datos
  sensibles (credenciales, tokens, información personal), el modelo puede leerlos y
  exponer su contenido en la conversación si se le pide o si es inducido a hacerlo.

## 6.2 Mitigaciones

- **Confirmación humana antes de ejecutar (human-in-the-loop).** La especificación de
  MCP establece como principio de seguridad que el host debe obtener consentimiento
  explícito del usuario antes de invocar una herramienta, y que las descripciones de
  herramientas deben tratarse como no confiables salvo que vengan de un servidor de
  confianza. En la práctica, clientes como Claude Code/Claude Desktop muestran al
  usuario qué herramienta se va a ejecutar y con qué argumentos, y piden aprobación
  antes de correrla (sobre todo para operaciones de escritura).
- **Alcance limitado a un directorio (*least privilege*).** Como se describe en
  [`05-servidor-filesystem.md`](05-servidor-filesystem.md), restringir el servidor a un
  único directorio creado para la tarea limita el radio de impacto de cualquier error o
  manipulación, incluso si el modelo "quisiera" hacer algo fuera de ese alcance: la
  validación de rutas ocurre dentro del propio proceso servidor, no depende de que el
  modelo "se porte bien".
- **Permisos de solo lectura cuando sea suficiente.** Si la tarea que se necesita
  resolver solo requiere leer archivos (no crear, modificar ni mover), conviene no
  habilitar las herramientas de escritura, o correr el servidor sobre una copia de solo
  lectura del contenido, para eliminar por completo el riesgo de escritura/borrado no
  deseado.
- **Revisión de lo que el servidor expone.** Antes de conectar cualquier servidor MCP,
  conviene revisar qué herramientas publica (`tools/list`) y qué hace cada una,
  especialmente si el servidor no es uno de los oficiales/de referencia. Un servidor mal
  implementado —o malicioso— podría anunciar una herramienta con una descripción
  engañosa ("lee un archivo") que en realidad ejecuta algo distinto.
- **No mezclar fuentes no confiables con acceso de escritura.** Como buena práctica
  general: si el modelo va a leer contenido de origen externo (una página web, un correo,
  un archivo descargado) en la misma sesión donde tiene acceso de escritura a archivos,
  el riesgo de que una instrucción inyectada se traduzca en una acción real aumenta. Separar
  sesiones, o revisar con más cuidado las acciones de escritura en esos casos, reduce el
  riesgo.

## 6.3 Evidencia práctica en este trabajo

En la Parte 2 de este repositorio se documenta una prueba explícita del límite de
seguridad: se le pide al modelo acceder a un archivo **fuera** del directorio
autorizado y se documenta, con captura de pantalla, la respuesta obtenida y el
mecanismo que impidió la operación (ver [`README.md`](../README.md#4-prueba-del-límite-de-seguridad)).
