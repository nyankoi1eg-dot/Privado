# Contenido automático para tienda de ropa

Formulario → ComfyUI (video, local) + Ollama (copy, local) → aprobación por Telegram → Postiz → Instagram y Facebook.

## 1. Requisitos en Windows (fuera de Docker)
- **ComfyUI** en `http://127.0.0.1:8188`, iniciado con `--listen` (ver handoff; PyTorch **cu128** para la RTX 5070).
- **Ollama** (`ollama pull qwen2.5:7b`). Escucha en el puerto 11434.
- **Docker Desktop**.

## 2. Levantar n8n y Postiz
```
copy .env.example .env      # edita los valores
docker compose up -d
```
- n8n: http://localhost:5678 · Postiz: http://localhost:5000
- Meta exige HTTPS en el login OAuth: usa un túnel (`cloudflared tunnel --url http://localhost:5000`) y pon esa URL en `POSTIZ_PUBLIC_URL`.

## 3. Postiz
1. Crea tu cuenta, luego pon `DISABLE_REGISTRATION=true`.
2. Crea una app en developers.facebook.com (tipo Business) con permisos de páginas e Instagram; pega `FACEBOOK_APP_ID/SECRET` en `.env`.
3. Conecta Facebook e Instagram (Add Channel). Copia el **ID** de cada canal.
4. Settings → Developers → copia la **API key**. Completa `.env` y `docker compose up -d`.

## 4. n8n
1. Telegram: crea un bot con @BotFather, escríbele, y obtén tu chat id (@userinfobot). Crea la credencial "Telegram bot" en n8n.
2. Importa `n8n/workflow.json` (menú ⋯ → Import from file) y asigna la credencial a los dos nodos Telegram.
3. En ComfyUI arma tu workflow imagen→video, actívalo con *Save (API Format)* y pégalo en el nodo **Preparar prompt ComfyUI** (`template`). Cambia el nombre de imagen del nodo LoadImage por `__IMAGE__` y el texto positivo por `__MOTION__`.
4. Activa el workflow y abre la URL del formulario.

## Puntos a verificar al probar
- Nombre de la propiedad binaria de la foto: asumí `field-0` (primer campo). Si falla, míralo en la salida del nodo Formulario.
- Esquema de la API pública de Postiz (`/public/v1/upload` y `/public/v1/posts`, campo `settings.__type`): confirma contra la versión que instales.
- Para subir el video a Instagram, Postiz lo publica como Reel; revisa que cumpla duración y formato.
