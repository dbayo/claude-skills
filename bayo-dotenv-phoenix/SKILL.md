---
name: bayo-dotenv-phoenix
description: >
  Configura carga automática de ficheros .env en proyectos Elixir/Phoenix usando dotenvy.
  Elimina la necesidad de hacer `set -a && source .env && set +a` antes de cada comando mix.
  Usar cuando el usuario diga "cargar .env automáticamente", "no quiero hacer source del .env",
  "evitar source .env", "dotenv en phoenix", "cargar variables de entorno automáticamente",
  "set -a source .env". Compatible con Phoenix 1.7+ y cualquier proyecto Mix estándar.
---

# bayo-dotenv-phoenix

Añade `dotenvy` al proyecto para que `mix phx.server`, `mix test` y cualquier comando Mix
carguen el `.env` automáticamente al arrancar, sin necesidad de sourcear el fichero a mano.

---

## Flujo paso a paso

### 1. Verificar el contexto del proyecto

Lee `mix.exs` para confirmar:
- Que es un proyecto Mix/Phoenix estándar
- Que `dotenvy` no está ya como dependencia
- El nombre de la app (por si hay algún `.env.{app}` específico)

Lee `config/runtime.exs` para confirmar que existe y que no hay ya una llamada a `Dotenvy.source`.

### 2. Añadir la dependencia en mix.exs

En la lista `deps`, añade `dotenvy` con `only: [:dev, :test]` para que no se incluya en producción:

```elixir
{:dotenvy, "~> 0.8", only: [:dev, :test]}
```

Añadirlo justo al final de la lista, después de la última dependencia existente.

### 3. Cargar el .env en config/runtime.exs

Añade al principio de `config/runtime.exs`, justo después de `import Config` y antes de cualquier `System.get_env/1`:

```elixir
if config_env() in [:dev, :test] do
  Dotenvy.source([".env", ".env.#{config_env()}", ".env.#{config_env()}.local"])
end
```

El orden de carga es (el último sobreescribe al anterior):
1. `.env` — valores base compartidos por el equipo
2. `.env.dev` / `.env.test` — overrides por entorno
3. `.env.dev.local` / `.env.test.local` — overrides personales (no commitear)

### 4. Instalar la dependencia

```bash
mix deps.get
```

### 5. Actualizar .gitignore (si aplica)

Si el proyecto tiene `.gitignore`, verificar que `.env` ya está ignorado. Añadir también:

```
.env.*.local
```

Para que los ficheros de override personal nunca se commiteen accidentalmente.

### 6. Crear PR

Commitear los tres ficheros modificados (`mix.exs`, `mix.lock`, `config/runtime.exs`) con un mensaje descriptivo y abrir PR.

---

## Consideraciones importantes

- **Solo dev/test**: `dotenvy` se declara con `only: [:dev, :test]`, por lo que las builds de producción (Fly, Docker, releases) no lo incluyen ni lo necesitan — en prod las variables llegan por el entorno del sistema.
- **No sobreescribe variables del sistema**: si una variable ya está definida en el entorno del sistema operativo, `dotenvy` la respeta y no la sobreescribe. Útil en CI.
- **Debe ir antes de System.get_env**: el bloque `Dotenvy.source` debe aparecer en `runtime.exs` antes de cualquier llamada a `System.get_env/1`, de lo contrario las vars del `.env` llegan tarde.
- **No funciona en config/config.exs ni config/dev.exs**: esos ficheros se evalúan en tiempo de compilación, no en runtime. `dotenvy` solo funciona en `runtime.exs`.
