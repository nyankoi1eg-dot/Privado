# Chatbot / agente IA con CRM propio (Perú): resumen para llevar a la PC principal

Fecha: 2026-11-05. Resultado de una investigación en sesión de nube. Las cifras y afirmaciones
vienen de blogs y agregadores; lo marcado **[verificar]** no se confirmó en fuente oficial.

## 1. Objetivo

Agente IA atendiendo clientes por WhatsApp que consulta inventario (hoy en Google Sheets), gestiona
pedidos de punta a punta y solo pide intervención humana para **confirmar el pago**. CRM propio.
IA local en tu PC (Ollama). Sin baneo del número.

## 2. Baneo del número: decisión

- Baileys / whatsapp-web.js (y por tanto Evolution API y WAHA) son no oficiales: riesgo alto de baneo.
  Solo para pruebas con número desechable.
- **Ruta recomendada: WhatsApp Cloud API oficial**, directa o vía Botmaker (en la sesión de nube había
  un token de Botmaker conectado; no se probó).
- **Coexistence** (misma línea en la app Business y la API): proveedores dicen que está activo en Perú
  **[verificar con Meta o tu proveedor]**. Límites: menor throughput, los grupos no se sincronizan.
- Política de Meta (ene 2026): prohíbe asistentes de IA de propósito general en la API; los bots de
  atención, pedidos y reservas siguen permitidos **[verificar términos oficiales]**.
- Respetar plantillas, ventana de 24 h, opt-in y mantener baja la tasa de bloqueos.

## 3. Arquitectura propuesta

```
WhatsApp (Cloud API / Botmaker)
      ↓ webhook
CRM propio (FastAPI + Postgres)  ←→  humano: solo confirma pagos
      ↓
Orquestador del agente (n8n, o servicio Python con Pydantic AI / LangGraph)
      ↓                         ↓
LLM local (Ollama)        Postgres = fuente de verdad del inventario ⇄ Google Sheets
```

- **Inventario:** Sheets no debe ser la única fuente (dos pedidos simultáneos pueden vender el mismo
  stock). Postgres con descuento atómico (`UPDATE … WHERE stock >= qty`); Sheets como vista y edición,
  sincronizado en ambos sentidos.
- **Pedido:** consulta → cotización → reserva de stock → `esperando_confirmación_de_pago` →
  (humano confirma) → confirmado → despachado. El humano solo actúa en el paso de pago.
- **CRM propio:** tablas de clientes, conversaciones, productos, stock, pedidos y pagos.
  Chatwoot queda opcional, solo como bandeja para el agente humano.

## 4. Modelos locales (Ollama)

- **Qwen3 30B-A3B** (Apache 2.0): mejor candidato para 24 GB de VRAM, con tool calling nativo.
  Tag sugerido: `ollama pull qwen3:30b-a3b` **[verificar]**.
- Llama 3.3 70B: soporta español pero va lento con offload a CPU.
- DeepSeek: el R1 completo no entra en una GPU; solo las versiones destiladas.
- Ninguna fuente evaluó español ni tool calling a fondo: probar con conversaciones reales tuyas.
- Dato pendiente: tu GPU y RAM, para fijar el modelo.

## 5. Skills y plugins revisados

Clonados y revisados (sin comandos peligrosos). Todos MIT salvo indicación. Ninguno está instalado
todavía: la escritura en `.claude/settings.json` fue bloqueada por el clasificador de permisos de la
sesión ("Untrusted Code Integration").

| Recurso | Para qué |
|---|---|
| `obra/superpowers` (plugin `superpowers@superpowers-dev`) | brainstorming, planes, TDD, subagentes, git worktrees |
| `czlonkowski/n8n-skills` (plugin `n8n-mcp-skills@n8n-mcp-skills`) | construir flujos n8n con validación |
| `aj-geddes/claude-code-bmad-skills` (plugin `bmad-planning-orchestrator@bmad-method-harness`) | fases de proyecto con agente orquestador |
| `wshobson/agents` (marketplace `claude-code-workflows`): `conductor`, `agent-teams`, `llm-application-dev`, `customer-sales-automation`, `database-design`, `backend-development`, `python-development` | orquestación, gestión de proyecto, backend y bases de datos |
| `automazeio/ccpm` (skill, se copia a `.claude/skills/ccpm`) | PRD → épicas → issues de GitHub → agentes en paralelo; requiere `gh` |
| `xing5/mcp-google-sheets` | MCP para leer y escribir Sheets; requiere credenciales de Google Cloud |

**Riesgo conocido:** el `mcp.json` de n8n-skills apunta al servicio alojado `api.n8n-mcp.com`. Con datos
reales conviene usar n8n-mcp autoalojado.

**Excluido a propósito:** `lharries/whatsapp-mcp` (conexión por QR, riesgo de baneo).

**Sin verificar:** VoltAgent/awesome-claude-code-subagents, claude-task-master, OpenSpec, spec-kit,
catálogos ComposioHQ y travisvn. Solo vistos en listados.

**Skills propios por escribir** (no existe equivalente): agente de ventas por WhatsApp en español,
máquina de estados del pedido, sync Sheets ⇄ Postgres, checklist anti-baneo, suite de pruebas del
agente (promptfoo / Langfuse, **[verificar]**).

### Comandos para tu PC (alcance usuario: valen para todos tus repos)

```bash
claude plugin marketplace add obra/superpowers
claude plugin install superpowers@superpowers-dev

claude plugin marketplace add czlonkowski/n8n-skills
claude plugin install n8n-mcp-skills@n8n-mcp-skills

claude plugin marketplace add aj-geddes/claude-code-bmad-skills
claude plugin install bmad-planning-orchestrator@bmad-method-harness

claude plugin marketplace add wshobson/agents
claude plugin install conductor@claude-code-workflows
claude plugin install agent-teams@claude-code-workflows
claude plugin install llm-application-dev@claude-code-workflows
claude plugin install customer-sales-automation@claude-code-workflows
claude plugin install database-design@claude-code-workflows
claude plugin install backend-development@claude-code-workflows
claude plugin install python-development@claude-code-workflows
```

Para que valgan solo en un repo, instalar con `--scope project` desde la carpeta del repo
**[verificar flag con `claude plugin install --help`]**.

## 6. Perú

- **Facturación SUNAT:** Greenter (PHP), OpenInvoicePeru (C#), cpe-engine (Python, MIT, compatible
  con Greenter; producción sin confirmar).
- **Yape / Plin:** no se encontró integración. Mientras tanto el humano confirma el comprobante,
  que coincide con tu regla de negocio.

## 7. Repos base ya existentes (punto de partida)

Ninguno trae todo. Lo más cercano:
- `n8n-io/self-hosted-ai-starter-kit`: n8n + Ollama + Qdrant + Postgres con Docker Compose y perfiles
  (`--profile gpu-nvidia`, `cpu`). El propio n8n dice que no está listo para producción sin endurecerlo.
- `coleam00/local-ai-packaged`: n8n, Open WebUI, Ollama, Supabase, Qdrant, Flowise, Langfuse, Caddy.
- `161sam/n8n-installer`: n8n + Flowise + Langfuse + Postgres + Redis con un script.

Falta en todos: WhatsApp oficial, el CRM propio y Chatwoot.

## 8. Decisiones pendientes

1. ¿Activo los plugins solo en este repo o para todo tu usuario?
2. ¿n8n-skills con el servicio alojado o con n8n-mcp local?
3. ¿Tu número hoy es WhatsApp Business app o normal? (define si usamos coexistence)
4. ¿GPU y RAM de tu PC?
5. ¿Pruebo el token de Botmaker?
