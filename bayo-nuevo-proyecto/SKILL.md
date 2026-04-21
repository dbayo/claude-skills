---
name: bayo-nuevo-proyecto
description: >
  Crea un proyecto nuevo con el stack estándar: Elixir + Phoenix + LiveView + TailwindCSS,
  sube el código a GitHub, y despliega en Fly.io. Usar siempre que el usuario diga
  "nuevo proyecto", "crear proyecto", "iniciar proyecto", "quiero hacer una app", o
  mencione arrancar desde cero con Elixir o Phoenix. No esperes a que el usuario pida
  explícitamente cada paso — ejecuta el flujo completo guiado.
---

# nuevo-proyecto

Automatiza la creación completa de un proyecto Elixir/Phoenix desde cero hasta producción.

## Stack siempre fijo

- **Elixir + Phoenix** con LiveView (`--live`)
- **TailwindCSS** (incluido por defecto en Phoenix 1.7+)
- **GitHub** para el repositorio remoto
- **Fly.io** para el despliegue

---

## Flujo paso a paso

### 1. Obtener el nombre del proyecto

Pregunta al usuario el nombre del proyecto en `snake_case` (ej: `mi_app`, `gestor_facturas`).

También pregunta:
- ¿El repositorio de GitHub debe ser **público** o **privado**? (default: privado)
- ¿En qué directorio quieres crear el proyecto? (default: directorio actual)

### 2. Crear el proyecto Phoenix

```bash
mix phx.new <nombre> --live
```

Cuando `mix phx.new` pida confirmación para instalar dependencias, responde `Y`.

Una vez creado, entra al directorio:
```bash
cd <nombre>
```

**Nota:** Phoenix 1.7+ incluye TailwindCSS automáticamente. No hace falta instalarlo por separado. Verifica que `assets/css/app.css` importa Tailwind y que `config/config.exs` o `assets/tailwind.config.js` están presentes.

### 3. Verificar que todo compila

```bash
mix deps.get
mix compile
```

Si hay errores, repórtaselos al usuario antes de continuar.

### 4. Inicializar git y subir a GitHub

```bash
git init
git add .
git commit -m "Initial commit: Phoenix + LiveView + TailwindCSS"
```

Luego crea el repositorio en GitHub y empuja:
```bash
gh repo create <nombre> --<public|private> --source=. --remote=origin --push
```

Usa `--public` o `--private` según lo que pidió el usuario.

Confirma con el usuario el nombre del repo y la visibilidad **antes** de ejecutar este comando.

### 5. Configurar Dockerfile y fly.toml desde plantillas

En lugar de usar los ficheros que genera `flyctl launch`, usa las plantillas probadas de este proyecto. Lee los ficheros de referencia:
- `references/Dockerfile.template`
- `references/fly.toml.template`

Copia ambos al raíz del proyecto nuevo y sustituye **todas** las ocurrencias de `APP_NAME` por el nombre del proyecto en snake_case:

```bash
# Ejemplo para un proyecto llamado "mi_app":
sed 's/APP_NAME/mi_app/g' <ruta-skill>/references/Dockerfile.template > Dockerfile
sed 's/APP_NAME/mi_app/g' <ruta-skill>/references/fly.toml.template > fly.toml
```

El `fly.toml` usa por defecto:
- **Región:** `cdg` (París) — pregunta al usuario si quiere otra
- **Memoria:** 512mb, 1 CPU shared — ajusta si el usuario lo pide
- El nombre de la app en Fly.io será `<nombre>` — pregunta si quiere un nombre diferente al del proyecto

Después registra la app en Fly.io:
```bash
flyctl apps create <nombre> --org personal
```

**Importante:** los nombres de app en Fly.io son globales. Si `<nombre>` ya está tomado, prueba `<nombre>-app` o `<nombre>-<iniciales>`. Confirma el nombre final con el usuario antes de continuar.

### 6. Configurar secrets y base de datos

Antes del primer despliegue, configura los secrets necesarios:

```bash
# SECRET_KEY_BASE — obligatorio para Phoenix en producción
flyctl secrets set SECRET_KEY_BASE="$(mix phx.gen.secret)" --app <nombre>

# ECTO_IPV6 — obligatorio en Fly.io (la red privada es solo IPv6)
flyctl secrets set ECTO_IPV6=true --app <nombre>
```

Luego crea la base de datos Postgres:

```bash
# Usa 512MB mínimo — 256MB es insuficiente para Postgres en Fly.io
flyctl postgres create --name <nombre>-db --org personal --region <región> \
  --initial-cluster-size 1 --vm-size shared-cpu-1x --volume-size 1
```

Y enlázala con la app:

```bash
flyctl postgres attach <nombre>-db --app <nombre>
```

Esto crea automáticamente el secret `DATABASE_URL`. Asegúrate de que usa el hostname `.internal` (no `.flycast`):

```bash
flyctl secrets set DATABASE_URL="postgres://<user>:<pass>@<nombre>-db.internal:5432/<db>?sslmode=disable" --app <nombre>
```

### 7. Primer despliegue

Pregunta al usuario: **"¿Quieres hacer el primer despliegue ahora?"**

Si dice sí:
```bash
flyctl deploy
```

Tras el despliegue, verifica que la app responde:
```bash
curl -s -o /dev/null -w "%{http_code}" https://<nombre>.fly.dev/
```

Debe devolver `200`. Si da error, revisa los logs con `flyctl logs --app <nombre>`.

### 8. Confirmar que todo está listo

Al final, muestra un resumen con:
- URL del repositorio GitHub
- URL de la app en Fly.io (si se desplegó)
- Comando para levantar el servidor en local: `mix phx.server`

---

## Gestión de errores

- Si `mix phx.new` falla: verifica que Elixir y Phoenix estén instalados (`mix archive.install hex phx_new` si falta).
- Si `gh` falla: comprueba que el usuario tenga `gh auth login` hecho.
- Si `flyctl` falla: verifica que el usuario tenga cuenta en Fly.io (`flyctl auth login`).
- Si la app arranca pero no conecta a la DB con `:nxdomain`: falta `ECTO_IPV6=true`.
- Si la app arranca pero no conecta a la DB con `tcp recv: closed`: el `DATABASE_URL` usa `.flycast` en vez de `.internal`, o la DB se queda sin memoria (escala a 512MB con `flyctl machine update <id> --vm-memory 512 --yes --app <nombre>-db`).
- Si cualquier paso falla, detente, explica el error, y pregunta al usuario cómo quiere proceder.

---

## Comportamiento esperado

- Sé proactivo: no esperes que el usuario te pida cada paso.
- Sí espera confirmación explícita antes de: crear el repo en GitHub, hacer el primer `flyctl deploy`.
- Si el usuario quiere saltarse un paso (ej: "no quiero GitHub aún"), adáptate sin problema.
- Mantén al usuario informado del progreso con mensajes cortos entre pasos.
