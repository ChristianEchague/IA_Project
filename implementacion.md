# Implementación de OpenClaw con Ollama en Local — Guía Completa

## Ecosistema

| Componente | Descripción |
|---|---|
| PC | I9 14900 — 64 GB RAM — 1 TB M2 — RTX 5070 |
| OS Host | Windows 11 |
| Docker Desktop | Orquestador de contenedores |
| Visual Studio Code | Editor y terminal |
| Ollama | Servidor LLM (contenedor Docker) |
| Open WebUI | Gestor visual de Ollama (contenedor Docker) |
| openclaw_ubuntu | Ubuntu con OpenClaw instalado (contenedor Docker) |

---

## PASO 1 — Crear los archivos de Docker

Abrir Visual Studio Code en modo administrador, crear una carpeta de proyecto y generar los siguientes archivos.

### `Dockerfile.ubuntu-ssh`

```dockerfile
FROM node:22

RUN apt-get update && apt-get install -y \
    openssh-server \
    vim \
    curl \
    git \
    ca-certificates \
    build-essential \
    && apt-get clean

RUN npm install -g pnpm

ENV SSH_PASSWORD=admin1234

RUN mkdir /var/run/sshd && \
    echo "root:${SSH_PASSWORD}" | chpasswd && \
    sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config

EXPOSE 22

CMD ["/usr/sbin/sshd", "-D"]
```

### `docker-compose.yml`

```yaml
services:
  ollama:
    image: ollama/ollama
    ports:
      - "11434:11434"
    volumes:
      - ollama_data:/root/.ollama
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]

  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    ports:
      - "3000:8080"
    environment:
      - OLLAMA_BASE_URL=http://ollama:11434
    depends_on:
      - ollama

  openclaw_ubuntu:
    build:
      context: .
      dockerfile: Dockerfile.ubuntu-ssh
    ports:
      - "2222:22"
      - "18790:18789"
      - "18791:18791"
    environment:
      - SSH_PASSWORD=admin1234
    volumes:
      - openclaw_ubuntu_home:/root

volumes:
  ollama_data:
  openclaw_ubuntu_home:
```

---

## PASO 2 — Levantar los contenedores

Desde una terminal en la carpeta del proyecto (puede tardar ~10 minutos la primera vez):

```bash
docker compose up -d
```

Si ya existían contenedores de una instalación anterior:

```bash
docker compose down
docker compose up -d --build
```

Verificar que los 3 contenedores estén corriendo en Docker Desktop.

---

## PASO 3 — Configurar modelos en Ollama (Open WebUI)

1. Abrir `http://localhost:3000` en el navegador
2. Registrar usuario administrador (primera vez)
3. Ir a **Tu nombre → Administración → Ajustes → Modelos → Gestionar**
4. En el campo "Extraer un modelo desde ollama.com" ingresar: `qwen3.5:9b`
5. Esperar la descarga (~6.6 GB)
6. Presionar `F5` para refrescar y confirmar que aparece en la lista
7. Crear un nuevo chat y verificar que el modelo responde

> Benchmark de modelos útil: [pinchbench.com](https://pinchbench.com/)

---

## PASO 4 — Conectarse al contenedor por SSH

Abrir PowerShell como administrador:

```bash
ssh root@localhost -p 2222
```

**Si aparece error de host key cambiada** (instalaciones previas):
```bash
ssh-keygen -R "[localhost]:2222"
```
Luego intentar nuevamente. Escribir `yes` cuando pida confirmación.

**Password:** `admin1234`

Prompt correcto tras conectarse:
```
root@<container_id>:~#
```

---

## PASO 5 — Actualizar el sistema

```bash
apt-get update && apt-get upgrade -y
```

---

## PASO 6 — Instalar OpenClaw desde el código fuente

```bash
git clone https://github.com/openclaw/openclaw.git
cd openclaw
pnpm install
pnpm build
```

> La primera compilación puede tardar varios minutos.

---

## PASO 7 — Parchear el servidor del browser (fix crítico)

Por defecto el servidor de browser escucha en `127.0.0.1` lo cual impide el acceso desde el host. Se debe cambiar a `0.0.0.0`:

```bash
sed -i 's/app.listen(port, "127.0.0.1"/app.listen(port, "0.0.0.0"/' extensions/browser/src/server.ts
```

Verificar el cambio:
```bash
grep -n "app.listen" extensions/browser/src/server.ts
```

Resultado esperado:
```
91:    const s = app.listen(port, "0.0.0.0", () => resolve(s));
```

Recompilar con el parche aplicado:
```bash
pnpm build
```

---

## PASO 8 — Crear el bot de Telegram *(requisito previo a la configuración)*

1. Abrir Telegram y buscar `@BotFather`
2. Enviar `/newbot`
3. Seguir las instrucciones (nombre + username terminado en `bot`)
4. Guardar el token recibido: `Use this token to access the HTTP API: XXXXXXXXX:YYYYYYY`

### Obtener tu ID numérico de Telegram (con @RawDataBot)

1. En Telegram buscar `@RawDataBot`
2. Enviarle cualquier mensaje (por ejemplo `hola`)
3. El bot responde con un JSON. Tu ID está en el campo `"id"` dentro de `"from"`:
   ```json
   "from": {
     "id": 123456789,
     "first_name": "Christian",
     ...
   }
   ```
4. Anotar el valor numérico de `"id"` — ese es tu `TU_TELEGRAM_ID`

> A diferencia de `@userinfobot`, `@RawDataBot` muestra el JSON completo del mensaje, lo que permite ver el ID numérico exacto sin ambigüedad.

---

## PASO 9 — Crear la configuración de OpenClaw

```bash
mkdir -p ~/.openclaw
cat > ~/.openclaw/openclaw.json << 'EOF'
{
  "gateway": {
    "mode": "local",
    "bind": "lan",
    "controlUi": {
      "allowedOrigins": [
        "http://localhost:18789",
        "http://127.0.0.1:18789",
        "http://localhost:18790",
        "http://127.0.0.1:18790"
      ]
    },
    "auth": {
      "mode": "token",
      "token": "deddb9d0ab4c42882be20de2490b16680eda4d09ab1636f1"
    }
  },
  "agents": { "defaults": { "model": "ollama/qwen3.5:9b" } },
  "models": {
    "providers": {
      "ollama": {
        "baseUrl": "http://ollama:11434",
        "models": [
          { "id": "qwen3.5:9b", "name": "Qwen 3.5 9B" }
        ]
      }
    }
  },
  "channels": {
    "telegram": {
      "botToken": "TU_BOT_TOKEN",
      "dmPolicy": "allowlist",
      "allowFrom": ["TU_TELEGRAM_ID"],
      "enabled": true
    }
  },
  "plugins": { "entries": { "google": { "enabled": true } } }
}
EOF
```

> Reemplazar `TU_BOT_TOKEN` con el token obtenido en el PASO 8 y `TU_TELEGRAM_ID` con tu ID numérico de Telegram (también del PASO 8).

Validar la configuración:
```bash
pnpm openclaw config validate
```

Resultado esperado:
```
Config valid: ~/.openclaw/openclaw.json
```

---

## PASO 10 — Iniciar el gateway

```bash
nohup pnpm openclaw gateway --port 18789 >> ~/.openclaw/gateway.log 2>&1 &
```

Verificar que arrancó correctamente:
```bash
tail -15 ~/.openclaw/gateway.log
```

Señales de que todo está OK:
- `[gateway] starting channels and sidecars...`
- `[browser/server] Browser control listening on http://0.0.0.0:18791/`
- `[telegram] [default] starting provider`
- `[ws] webchat connected`

---

## PASO 11 — Crear los archivos del workspace del agente

OpenClaw requiere una serie de archivos `.md` en `~/.openclaw/workspace/` para el funcionamiento del agente. Si no existen, el bot responderá con errores del tipo `Edit: in ~/.openclaw/workspace/IDENTITY.md failed`.

```bash
mkdir -p ~/.openclaw/workspace

cat > ~/.openclaw/workspace/IDENTITY.md << 'EOF'
# IDENTITY.md - Who Am I?
Name: Assistant
Creature: 🦞
Vibe: helpful, concise, local
Emoji: 🤖
Avatar:
EOF

cat > ~/.openclaw/workspace/AGENTS.md << 'EOF'
# AGENTS.md - Your Workspace
Sos un asistente local útil y conciso. Respondé siempre en el idioma del usuario.
EOF

cat > ~/.openclaw/workspace/SOUL.md << 'EOF'
# SOUL.md - Who You Are
Asistente personal local. Tono amigable y directo.
EOF

cat > ~/.openclaw/workspace/TOOLS.md << 'EOF'
# TOOLS.md - Local Notes
EOF

cat > ~/.openclaw/workspace/USER.md << 'EOF'
# USER.md - User Profile
Name: Christian
Preferred address: Christian
Pronouns:
Timezone: America/Argentina/Buenos_Aires
Notes:
EOF

cat > ~/.openclaw/workspace/MEMORY.md << 'EOF'
# MEMORY.md
EOF

cat > ~/.openclaw/workspace/HEARTBEAT.md << 'EOF'
# HEARTBEAT.md
EOF
```

> Los archivos críticos son `IDENTITY.md`, `AGENTS.md` y `SOUL.md`. Los demás evitan advertencias futuras.

---

## PASO 12 — Verificación final

| Verificación | URL / Método |
|---|---|
| Control UI | `http://localhost:18790` → token: `deddb9d0ab4c42882be20de2490b16680eda4d09ab1636f1` |
| Telegram | Enviar mensaje al bot desde tu cuenta |
| Open WebUI | `http://localhost:3000` |
| Ollama API | `http://localhost:11434` → debe responder `Ollama is running` |

---

## Notas importantes

### Puntos críticos que NO deben omitirse
- El parche de `server.ts` (PASO 7) es **obligatorio** para que el Control UI sea accesible desde el host.
- `pnpm build` debe ejecutarse **dos veces**: una inicial y otra después del parche.
- El campo `models` en la configuración debe ser un array de objetos `{id, name}`, no un array de strings.
- El `baseUrl` de Ollama usa el nombre del servicio Docker (`http://ollama:11434`), no `localhost`.

### Reiniciar el gateway tras reboot del contenedor

Los contenedores Docker no persisten procesos en segundo plano entre reinicios — solo persisten los archivos en los volúmenes. Cada vez que el contenedor `openclaw_ubuntu` se reinicia (por un `docker compose down/up`, un reinicio de Docker Desktop, o un reboot del host), el proceso del gateway deja de existir y hay que levantarlo manualmente.

**Procedimiento:**

1. Conectarse por SSH al contenedor (igual que el PASO 4):
   ```bash
   ssh root@localhost -p 2222
   ```

2. Pararse en la carpeta de OpenClaw:
   ```bash
   cd ~/openclaw
   ```

3. Lanzar el gateway en segundo plano:
   ```bash
   nohup pnpm openclaw gateway --port 18789 >> ~/.openclaw/gateway.log 2>&1 &
   ```

4. Verificar que arrancó correctamente:
   ```bash
   tail -15 ~/.openclaw/gateway.log
   ```

**Qué hace cada parte del comando:**
- `nohup` — impide que el proceso muera al cerrar la sesión SSH
- `>> ~/.openclaw/gateway.log` — acumula los logs en el mismo archivo (no lo sobreescribe)
- `2>&1` — redirige también los errores al log
- `&` — lo deja corriendo en segundo plano

> Si querés evitar hacer esto manualmente cada vez, podés agregar el comando al archivo `~/.bashrc` o crear un script de inicio. Sin embargo, para un entorno de uso ocasional, iniciarlo a mano cuando se necesita es suficiente.

---

### Cambiar el token de autenticación del Control UI

El token es la contraseña que protege el acceso al Control UI (`http://localhost:18790`). Está definido en `~/.openclaw/openclaw.json` bajo `gateway.auth.token`.

**Procedimiento:**

1. Conectarse por SSH al contenedor.

2. Editar el archivo de configuración:
   ```bash
   vim ~/.openclaw/openclaw.json
   ```
   Dentro de vim: presionar `i` para entrar en modo edición, modificar el token, luego `Esc` → `:wq` → `Enter` para guardar y salir.

3. El gateway detecta el cambio automáticamente sin necesidad de reiniciarlo ("recarga en caliente"). En pocos segundos el nuevo token ya está activo.

4. Verificar en el log que no hubo errores tras el cambio:
   ```bash
   tail -5 ~/.openclaw/gateway.log
   ```

**Requisitos del token:** debe ser una cadena hexadecimal. Para generar uno nuevo podés usar:
```bash
openssl rand -hex 24
```

> Tras cambiar el token, actualizar también el valor guardado en el navegador si usás autocompletado, ya que el anterior quedará inválido.
