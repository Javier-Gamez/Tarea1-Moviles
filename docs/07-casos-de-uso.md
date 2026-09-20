# 7. Casos de uso

## 7.1 Herramientas reales que implementan MCP

### 1. Claude Code / Claude Desktop (Anthropic)

Es el cliente/host que usamos en la Parte 2 de este trabajo. Claude Code (CLI y app de
escritorio) actúa como **host** MCP: se conecta a servidores locales (por stdio) o
remotos (por Streamable HTTP) declarados en un archivo de configuración por
proyecto o por usuario, descubre sus herramientas y se las ofrece al modelo dentro de la
conversación. Anthropic publicó originalmente la especificación de MCP en noviembre de
2024, precisamente para resolver el problema de que cada integración con una fuente de
datos distinta requería código a la medida.

### 2. Visual Studio Code + GitHub Copilot (Microsoft / GitHub)

VS Code incorporó soporte nativo para MCP en Copilot Chat/Copilot Agent Mode: desde un
archivo `mcp.json` (a nivel de usuario o de workspace) se declaran servidores MCP —
locales o remotos— y el editor los expone al modelo dentro del chat agéntico. Se usa,
por ejemplo, para darle al agente acceso a un servidor de GitHub (gestión de *issues* y
*pull requests*), a bases de datos, o a servidores de documentación de un framework
específico, sin que Microsoft tenga que programar esa integración dentro del propio VS
Code: cualquiera puede publicar un servidor MCP nuevo y conectarlo.

### 3. Google Antigravity (Google)

Antigravity es el entorno de desarrollo agéntico de Google (distinto de Gemini o de la
familia de modelos que usa por debajo): permite planear, escribir, probar y ejecutar
cambios sobre un repositorio completo a partir de una sola instrucción. Agregó soporte
para servidores MCP —locales y remotos— a inicios de 2026, incluyendo integraciones de
un clic con servidores MCP para servicios de Google Cloud, además de permitir conectar
servidores de terceros (por ejemplo, GitHub para gestión de *pull requests*, o
servidores con conocimiento del esquema de una base de datos).

### 4. Cursor (Anysphere)

Editor basado en VS Code, orientado específicamente a desarrollo asistido por IA. Soporta
servidores MCP configurables por proyecto, lo que permite, por ejemplo, darle al agente
acceso a un servidor de base de datos para que escriba y valide migraciones consultando
el esquema real, en vez de adivinarlo a partir del código.

> **Nota sobre precisión terminológica**: *Qwen* es una **familia de modelos** de
> Alibaba, no una plataforma de desarrollo ni un cliente MCP — un modelo, por sí solo,
> no "implementa" MCP (ver [`02-aislamiento.md`](02-aislamiento.md) y
> [`03-mcp-vs-api.md`](03-mcp-vs-api.md): un modelo no ejecuta protocolos, los usa un
> host/cliente). Si se quisiera citar el ecosistema de Qwen como caso de uso, habría que
> referirse a una herramienta concreta que lo use como uno de sus modelos disponibles —
> por ejemplo un CLI o editor agéntico específico —, nunca a "Qwen" a secas como si
> fuera el cliente. Por eso no lo incluimos como uno de los tres ejemplos principales de
> este trabajo: preferimos casos donde la herramienta cliente/host está identificada sin
> ambigüedad.

## 7.2 Cómo editan repositorios completos sin subir archivos manualmente

Las cuatro herramientas anteriores comparten el mismo mecanismo de fondo, que es
justamente el que investigamos en este trabajo: en lugar de que la persona copie y
pegue el contenido de sus archivos dentro de una conversación (como se hacía con un
chatbot de navegador tradicional), el **cliente/host corre en la misma máquina que el
código** y mantiene conexiones MCP (u otro mecanismo de *tool calling* equivalente) hacia
servidores con acceso real al sistema de archivos y, frecuentemente, a una terminal.

Eso le permite al modelo, dentro de una sola conversación:

1. **Listar y leer** los archivos relevantes del repositorio bajo demanda (con
   herramientas como `list_directory` / `read_text_file`), en vez de depender de que el
   usuario adivine qué archivos pegar.
2. **Buscar** dónde vive determinado símbolo, función o texto en todo el repositorio
   (`search_files` o equivalentes), igual que haría una persona con su editor.
3. **Escribir y modificar** varios archivos en la misma tarea (`write_file`,
   `edit_file`), aplicando un cambio coherente a través de todo el proyecto —por
   ejemplo, renombrar una función y actualizar todos sus usos— sin que el usuario tenga
   que copiar cada archivo modificado de vuelta a su editor.
4. Frecuentemente, además, **ejecutar comandos** (tests, linters, compilación) a través
   de un servidor de terminal/shell, para verificar que el cambio funciona antes de
   dárselo por terminado al usuario.

Todo esto ocurre porque el *descubrimiento* de qué herramientas están disponibles y la
*decisión* de cuáles usar suceden en tiempo real dentro de la conversación (ver
[`03-mcp-vs-api.md`](03-mcp-vs-api.md)), no porque el modelo tenga acceso mágico al disco:
sigue siendo el proceso servidor, con permisos delimitados por el usuario, quien ejecuta
cada operación.

## Referencias de esta sección

Ver lista completa de referencias en formato APA en el [`README.md`](../README.md).
