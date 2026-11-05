# Chatbot / agente IA con CRM propio (Perú): documento único

Fecha: 2026-11-05. Investigación hecha en una sesión de nube; **nada de lo descrito está instalado ni
construido todavía**. Las afirmaciones vienen de blogs, agregadores y de leer el código de los repos
clonados. Lo marcado **[verificar]** no se confirmó en fuente oficial. Los archivos de la sección 10
son borradores **sin probar**.

---

## 1. Objetivo y reglas de negocio

- Agente IA por WhatsApp que atiende clientes, consulta inventario (hoy en Google Sheets) y gestiona
  pedidos de punta a punta.
- **Única intervención humana: confirmar el pago.** Todo lo demás lo hace el agente.
- CRM propio. IA local en tu PC (Ollama). Sin baneo del número. País: Perú.

## 2. Baneo del número: decisión

- Baileys y whatsapp-web.js (y por tanto Evolution API y WAHA) son no oficiales: riesgo alto de baneo.
  Úsalos solo para pruebas con un número desechable. No existe librería no oficial "a prueba de baneo".
- **Ruta recomendada: WhatsApp Cloud API oficial**, directa o vía Botmaker. En la sesión de nube había
  un token de Botmaker conectado; no se probó.
- **Coexistence** (misma línea en la app Business y en la API): proveedores dicen que está activo en
  AR, MX, CO, CL, PE, BR y UY **[verificar con Meta o tu proveedor]**. Límites: menor throughput, los
  grupos no se sincronizan, el número debe estar ya en uso en la app Business.
- Política de Meta (enero 2026): prohíbe los asistentes de IA de propósito general en la API; los bots
  de atención, pedidos y reservas siguen permitidos **[verificar términos oficiales]**.
- Un reporte (una sola fuente) dice que desde el 1 de octubre de 2026 se cobran los mensajes de
  servicio sin cupo gratuito **[verificar en la página de precios de Meta]**.
- Buenas prácticas: plantillas aprobadas, ventana de 24 h, opt-in, y mantener baja la tasa de bloqueos.

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

- **Inventario:** Google Sheets como única fuente es riesgoso: dos pedidos simultáneos pueden vender el
  mismo stock. Postgres decide con descuento atómico; Sheets queda como vista y edición, sincronizado
  en ambos sentidos. (Criterio de ingeniería propio; las búsquedas no lo trataron.)
- **CRM propio:** FastAPI + Postgres (clientes, conversaciones, productos, stock, pedidos, pagos).
  Chatwoot queda opcional, solo como bandeja para el agente humano.
- **Pedido:** consulta → cotización → reserva de stock → `esperando_pago` → (humano confirma) →
  confirmado → despachado. Estados terminales: `cancelado`, `expirado`.
- **Yape / Plin:** no se encontró integración. El cliente envía el comprobante y el humano lo confirma
  en el CRM, que coincide con tu regla de negocio.

## 4. Modelos locales (Ollama)

- **Qwen3 30B-A3B** (Apache 2.0): mejor candidato para 24 GB de VRAM, con tool calling nativo.
  Tag sugerido: `ollama pull qwen3:30b-a3b` **[verificar]**.
- Llama 3.3 70B: soporta español, pero con offload a CPU va lento (unos 8 tok/s según una fuente).
- DeepSeek: el R1 completo no entra en una GPU; solo las versiones destiladas.
- Ninguna fuente evaluó español ni tool calling a fondo. Probar con conversaciones reales tuyas.
- Dato pendiente: tu GPU y RAM, para fijar el modelo.

## 5. Skills y plugins revisados

Clonados y revisados en la sesión: sin `curl | sh`, `eval`, `base64 -d` ni `rm -rf` peligrosos. Los
hooks de sesión solo inyectan contexto o validan flujos. Todos MIT.

| Recurso | Plugin / ubicación | Para qué |
|---|---|---|
| obra/superpowers | `superpowers@superpowers-dev` | brainstorming, planes, TDD, subagentes, git worktrees |
| czlonkowski/n8n-skills | `n8n-mcp-skills@n8n-mcp-skills` | construir flujos n8n con validación (15 skills) |
| aj-geddes/claude-code-bmad-skills | `bmad-planning-orchestrator@bmad-method-harness` | fases de proyecto con agente orquestador |
| wshobson/agents | marketplace `claude-code-workflows` | `conductor`, `agent-teams`, `llm-application-dev`, `customer-sales-automation`, `database-design`, `backend-development`, `python-development` |
| automazeio/ccpm | skill: copiar `skill/ccpm` a `.claude/skills/ccpm` | PRD → épicas → issues de GitHub → agentes en paralelo; requiere `gh` |
| xing5/mcp-google-sheets | MCP (`uvx mcp-google-sheets@latest`) | leer y escribir Sheets; requiere credenciales de Google Cloud |

Riesgos y exclusiones:
- El `mcp.json` de n8n-skills apunta al servicio alojado `api.n8n-mcp.com`. Con datos reales, usa
  n8n-mcp autoalojado.
- El README de mcp-google-sheets instala `uv` con `curl | sh` (instalador oficial de uv).
- **Excluido a propósito:** `lharries/whatsapp-mcp` (conexión por QR, riesgo de baneo).
- **Sin verificar** (solo vistos en listados): VoltAgent/awesome-claude-code-subagents,
  claude-task-master, OpenSpec, spec-kit, catálogos ComposioHQ y travisvn/awesome-claude-skills.
- El marketplace de wshobson trae 185 skills y 204 agentes: por eso solo se activan siete plugins.

### Alcance: ¿dónde se instalan?

- **Solo un proyecto:** `.claude/settings.json` del repo. Lo cargan las sesiones abiertas en ese repo.
- **Todos tus repos en una máquina:** `--scope user`, que guarda en `~/.claude/`.
- **Todas las sesiones de claude.ai:** skills subidos en Ajustes de la cuenta.
- La sesión de nube tiene `~/` efímero: lo instalado ahí no viaja a tu PC. Instala en tu PC.
- Recomendado: solo en el repo, para un entorno integrado.

### Comandos en tu PC (alcance usuario)

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

Para solo un repo, instala desde su carpeta con `--scope project` **[verificar con
`claude plugin install --help`]**. Una guía de terceros reporta que los plugins de alcance proyecto
a veces aparecen instalados pero no disponibles; comprueba con `/plugin` tras instalar.

### Skills propios por escribir (no existe equivalente)

1. Agente de ventas por WhatsApp en español de Perú: tono, reglas, cuándo escalar.
2. Máquina de estados del pedido.
3. Sincronización Sheets ⇄ Postgres con descuento atómico.
4. Checklist anti-baneo: plantillas, ventana de 24 h, opt-in, tasa de bloqueos.
5. Suite de pruebas del agente con casos reales (promptfoo y Langfuse **[verificar]**).

## 6. Perú

- **Facturación SUNAT:** Greenter (PHP), OpenInvoicePeru (C#) y cpe-engine (Python, MIT, compatible con
  Greenter; su uso en producción no está confirmado más allá de su README). Generan y envían
  comprobantes, pero no gestionan notificaciones de pago.
- Botmaker: su página de precios lista una cuota única de 99 USD para abrir la cuenta de WhatsApp
  Business Platform y un plan Standard de 149 USD; no se confirmó si aplican a Perú. Meta cobra aparte
  por mensaje **[confirmar con Botmaker y con la página de precios de Meta]**.

## 7. Bases de partida existentes

Ningún repo trae todo. Lo más cercano:
- `n8n-io/self-hosted-ai-starter-kit`: n8n + Ollama + Qdrant + Postgres con Docker Compose y perfiles
  (`--profile gpu-nvidia`, `cpu`). n8n advierte que no está listo para producción sin endurecerlo.
- `coleam00/local-ai-packaged`: n8n, Open WebUI, Ollama, Supabase, Qdrant, Flowise, Langfuse, Caddy.
- `161sam/n8n-installer`: n8n + Flowise + Langfuse + Postgres + Redis con un script.

Falta en todos: WhatsApp oficial, el CRM propio y Chatwoot. Por eso se propone un monorepo propio.
(Una sola búsqueda; puede haber repos comunitarios pequeños que no aparecieron.)

## 8. Estructura del monorepo propuesto

```
Privado/
├── docker-compose.yml     # Postgres, Ollama, n8n, CRM (Langfuse y Chatwoot en fase 2)
├── .env.example           # secretos y tokens (Botmaker / Cloud API)
├── Makefile               # make up / pull-models / down / logs
├── db/init/               # SQL de arranque (bases y esquema)
├── crm/                   # CRM propio: FastAPI + Postgres + migraciones
├── agent/                 # prompts, herramientas, máquina de estados
├── sync/                  # Sheets ⇄ Postgres
├── .claude/settings.json  # plugins activados + skills propios
├── .mcp.json              # MCP de Sheets y n8n local
├── CLAUDE.md              # contexto del proyecto
└── docs/                  # este documento
```

Flujo en tu PC: clonar, `cp .env.example .env`, `make up`, abrir Claude Code en la carpeta.

## 9. Fases sugeridas

1. **Base:** Postgres + CRM mínimo (productos, stock, pedidos) + sync desde tu Sheet. Pruebas del
   descuento atómico.
2. **Agente sin WhatsApp:** Ollama + herramientas (`buscar_producto`, `reservar`, `crear_pedido`,
   `estado_pedido`) probadas por consola con conversaciones reales en español.
3. **WhatsApp oficial:** conectar webhook (Cloud API o Botmaker) en un número de prueba.
4. **Pago y humano:** pantalla del CRM para confirmar pagos; el agente retoma al confirmarse.
5. **Producción:** coexistence o número definitivo, endurecimiento, observabilidad (Langfuse),
   facturación SUNAT.

## 10. Borradores de archivos (sin probar)

### 10.1 `.claude/settings.json` (alcance proyecto)

```json
{
  "extraKnownMarketplaces": {
    "caveman": { "source": { "source": "github", "repo": "JuliusBrussee/caveman" } },
    "superpowers-dev": { "source": { "source": "github", "repo": "obra/superpowers" } },
    "n8n-mcp-skills": { "source": { "source": "github", "repo": "czlonkowski/n8n-skills" } },
    "bmad-method-harness": { "source": { "source": "github", "repo": "aj-geddes/claude-code-bmad-skills" } },
    "claude-code-workflows": { "source": { "source": "github", "repo": "wshobson/agents" } }
  },
  "enabledPlugins": {
    "caveman@caveman": true,
    "superpowers@superpowers-dev": true,
    "n8n-mcp-skills@n8n-mcp-skills": true,
    "bmad-planning-orchestrator@bmad-method-harness": true,
    "conductor@claude-code-workflows": true,
    "agent-teams@claude-code-workflows": true,
    "llm-application-dev@claude-code-workflows": true,
    "customer-sales-automation@claude-code-workflows": true,
    "database-design@claude-code-workflows": true,
    "backend-development@claude-code-workflows": true,
    "python-development@claude-code-workflows": true
  }
}
```

El repo ya tiene el marketplace `caveman` activado; los otros cuatro son nuevos.

### 10.2 `docker-compose.yml` (borrador mínimo, GPU NVIDIA)

```yaml
services:
  postgres:
    image: postgres:16
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - pg_data:/var/lib/postgresql/data
      - ./db/init:/docker-entrypoint-initdb.d:ro
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
      interval: 5s
      retries: 10

  ollama:
    image: ollama/ollama
    restart: unless-stopped
    ports: ["127.0.0.1:11434:11434"]
    volumes: [ollama_data:/root/.ollama]
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]

  n8n:
    image: docker.n8n.io/n8nio/n8n
    restart: unless-stopped
    ports: ["127.0.0.1:5678:5678"]
    environment:
      DB_TYPE: postgresdb
      DB_POSTGRESDB_HOST: postgres
      DB_POSTGRESDB_DATABASE: n8n
      DB_POSTGRESDB_USER: ${POSTGRES_USER}
      DB_POSTGRESDB_PASSWORD: ${POSTGRES_PASSWORD}
      N8N_ENCRYPTION_KEY: ${N8N_ENCRYPTION_KEY}
    depends_on:
      postgres: { condition: service_healthy }
    volumes: [n8n_data:/home/node/.n8n]

  crm:
    build: ./crm
    restart: unless-stopped
    ports: ["127.0.0.1:8000:8000"]
    environment:
      DATABASE_URL: postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres:5432/crm
      OLLAMA_URL: http://ollama:11434
    depends_on:
      postgres: { condition: service_healthy }

volumes:
  pg_data:
  ollama_data:
  n8n_data:
```

Sin GPU NVIDIA, elimina el bloque `deploy` de `ollama`. Los puertos quedan en `127.0.0.1` a propósito.
Langfuse v3 necesita servicios extra (ClickHouse, Redis, almacenamiento): va en la fase 5.

### 10.3 `.env.example`

```bash
POSTGRES_USER=privado
POSTGRES_PASSWORD=cambia-esto
N8N_ENCRYPTION_KEY=genera-con-openssl-rand-hex-32
# WhatsApp (Cloud API o Botmaker): rellenar al llegar a la fase 3
WHATSAPP_TOKEN=
WHATSAPP_PHONE_ID=
WEBHOOK_VERIFY_TOKEN=
```

No subas `.env` a git.

### 10.4 `db/init/01-databases.sql` y `02-schema.sql`

```sql
-- 01-databases.sql
CREATE DATABASE n8n;
CREATE DATABASE crm;
```

```sql
-- 02-schema.sql (ejecutar dentro de la base crm)
\connect crm

CREATE TABLE customers (
  id BIGSERIAL PRIMARY KEY,
  phone TEXT UNIQUE NOT NULL,
  name TEXT,
  created_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE products (
  id BIGSERIAL PRIMARY KEY,
  sku TEXT UNIQUE NOT NULL,
  name TEXT NOT NULL,
  price_pen NUMERIC(10,2) NOT NULL,
  stock INT NOT NULL CHECK (stock >= 0),
  sheet_row INT            -- fila en Google Sheets, para el sync
);

CREATE TYPE order_status AS ENUM
  ('reservado','esperando_pago','confirmado','despachado','cancelado','expirado');

CREATE TABLE orders (
  id BIGSERIAL PRIMARY KEY,
  customer_id BIGINT REFERENCES customers(id),
  status order_status NOT NULL DEFAULT 'reservado',
  total_pen NUMERIC(10,2) NOT NULL DEFAULT 0,
  reserved_until TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE order_items (
  order_id BIGINT REFERENCES orders(id),
  product_id BIGINT REFERENCES products(id),
  qty INT NOT NULL CHECK (qty > 0),
  unit_price_pen NUMERIC(10,2) NOT NULL,
  PRIMARY KEY (order_id, product_id)
);

CREATE TABLE payments (
  id BIGSERIAL PRIMARY KEY,
  order_id BIGINT REFERENCES orders(id),
  proof_url TEXT,
  confirmed_by TEXT,        -- el humano que confirma
  confirmed_at TIMESTAMPTZ
);
```

Reserva atómica de stock (una sola sentencia; si devuelve 0 filas, no hay stock suficiente):

```sql
UPDATE products SET stock = stock - :qty
WHERE id = :product_id AND stock >= :qty
RETURNING id;
```

Las reservas vencidas (`reserved_until` pasado, sin pago) se cancelan y devuelven el stock en un
trabajo periódico.

### 10.5 Prompt de sistema del agente (borrador)

```
Eres el asistente de ventas de <NEGOCIO> por WhatsApp. Hablas español peruano, tono cordial y breve.

Reglas:
- Solo hablas de productos, precios, stock, pedidos y envíos de <NEGOCIO>. Si preguntan otra cosa,
  redirige amablemente.
- Nunca inventes stock, precios ni plazos: consulta siempre las herramientas.
- Nunca confirmes un pago. Cuando el cliente envíe su comprobante, adjúntalo al pedido, di que está
  en revisión y deja el pedido en esperando_pago. Solo una persona confirma pagos.
- Si el cliente pide hablar con una persona, o hay un reclamo, escala y avisa que lo atenderán.
- Antes de crear el pedido, confirma productos, cantidades, total en soles y dirección.
- No pidas datos innecesarios ni compartas datos de otros clientes.
```

### 10.6 `CLAUDE.md` del repo (borrador)

```
# Proyecto: agente de ventas por WhatsApp con CRM propio (Perú)
- Postgres es la fuente de verdad del inventario; Google Sheets es una vista sincronizada.
- El stock se descuenta solo con UPDATE atómico condicional. Nunca leer y luego escribir.
- Único paso humano: confirmar el pago. El agente nunca lo confirma.
- WhatsApp: solo Cloud API oficial o Botmaker. Prohibido Baileys / whatsapp-web.js en producción.
- Secretos solo en .env (fuera de git). Servicios expuestos solo en 127.0.0.1.
- Antes de instalar un skill o plugin de terceros: leer su código.
```

### 10.7 Prompt para pegar en Claude Code en tu PC

```
Lee docs/PROYECTO_CHATBOT_COMPLETO.md. Empieza por la fase 1: crea la estructura del monorepo de la
sección 8, el docker-compose de la 10.2 y el esquema de la 10.4, con pruebas del descuento atómico de
stock (dos pedidos simultáneos por la última unidad). Usa los skills de superpowers para planificar y
TDD. Pregúntame antes de instalar cualquier plugin o conectar WhatsApp.
```

## 11. Decisiones pendientes

1. ¿Plugins solo en el repo (recomendado) o para todo tu usuario?
2. ¿n8n-skills con el servicio alojado o con n8n-mcp local (recomendado)?
3. ¿Tu número hoy es WhatsApp Business app o normal? Define si usamos coexistence.
4. ¿GPU y RAM de tu PC? Define el modelo y el bloque `deploy` de Docker.
5. ¿Probamos el token de Botmaker?
6. ¿Qué cuenta de Google Cloud usarás para el acceso a Sheets?

## 12. Fuentes

- [Baileys vs Cloud API, riesgo de baneo (pipes.bot)](https://www.pipes.bot/blog/baileys-vs-whatsapp-cloud-api-ban)
- [Evolution API vs WAHA vs Baileys (pipes.bot)](https://www.pipes.bot/blog/evolution-api-vs-waha-vs-baileys-whatsapp-ban-risk)
- [Automatización de WhatsApp y baneos (adviseai)](https://www.adviseai.in/blog/whatsapp-automation-ban-risk)
- [Política de IA de WhatsApp 2026 (respond.io)](https://respond.io/blog/whatsapp-general-purpose-chatbots-ban)
- [Política de IA de WhatsApp 2026 (Alibaba Cloud)](https://www.alibabacloud.com/help/en/chatapp/use-cases/whatsapp-ai-policy-2026-guide)
- [Coexistence (Kapso)](https://kapso.ai/blog/whatsapp-business-app-coexistence-cloud-api)
- [Coexistence en LATAM (Cliengo)](https://guiawabusiness.cliengo.com/coexistence)
- [Chatwoot, agent bots](https://www.chatwoot.com/hc/user-guide/articles/1677497472-how-to-use-agent-bots)
- [Plantilla n8n 16175 (pedidos + Sheets)](https://n8n.io/workflows/16175)
- [WAHA, precios y licencia](https://waha.devlike.pro/pricing)
- [n8n-skills](https://killer-skills.pages.dev/en/skills/czlonkowski/n8n-skills)
- [travisvn/awesome-claude-skills](https://github.com/travisvn/awesome-claude-skills/wiki)
- [claude-code-bmad-skills](https://github.com/aj-geddes/claude-code-bmad-skills)
- [Frameworks de orquestación para Claude (claudefa.st)](https://claudefa.st/blog/tools/orchestrators/multi-agent-orchestrators)
- [VoltAgent: 100 subagentes](https://dev.to/voltagent/100-claude-code-subagent-collection-1eb0)
- [Alcance de plugins en Claude Code](https://designrevision.com/blog/claude-code-plugins)
- [mcp-google-sheets (PyPI)](https://pypi.org/project/mcp-google-sheets/)
- [whatsapp-mcp (lharries), excluido](https://lobehub.com/mcp/lharries-whatsapp-mcp)
- [Llama vs Qwen3 vs DeepSeek en español](https://www.forgenex.com/en/blog/llama-vs-qwen3-vs-deepseek-la-batalla-de-la-ia-local-que-puedes-ejecutar-en-tu-propio-pc-y-cual-gana-en-espanol)
- [Mejores LLM locales 2026](https://promptzone.com/jordan_lee_72db45ce/best-local-llms-for-consumer-hardware-2026-llama-33-70b-vs-qwen3-30b-a3b-vs-deepseek-r1-distill-336p)
- [Modelos Ollama con tool calling](https://localaimaster.com/blog/best-ollama-models-tool-calling)
- [Botmaker, precios](https://botmaker.com/prices)
- [Greenter (SUNAT)](https://github.com/gmondragon/greenter)
- [OpenInvoicePeru](https://github.com/jhuaraya/openinvoiceperu)
- [cpe-engine](https://pypi.org/project/cpe-engine/)
- [Frappe CRM vs EspoCRM](https://openalternative.co/compare/espocrm/vs/frappe-crm)
- [Twenty vs EspoCRM](https://selfhosting-sh.pages.dev/compare/twenty-vs-espocrm/)
- [NocoDB vs Baserow](https://stackshare.io/stackups/baserow-io-vs-nocodb)
- [n8n Self-hosted AI Starter Kit](https://docs.n8n.io/hosting/starter-kits/ai-starter-kit/)
- [coleam00/local-ai-packaged](https://docsearch.algolia.com/mcp/docs/repo/coleam00/local-ai-packaged)
- [161sam/n8n-installer](https://github.com/161sam/n8n-installer)
