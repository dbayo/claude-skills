---
name: bayo-error-page
description: >
  Crea una página de error 500 personalizada y adaptada al diseño del proyecto,
  reemplazando el texto plano por defecto ("Internal Server Error") por una vista
  mobile-first con mensaje amigable y contacto de administración.
  Usar cuando el usuario diga "página de error", "error 500", "error page", "cuando
  la app peta mostrar algo bonito", o quiera mejorar la experiencia ante fallos en producción.
  Compatible principalmente con Phoenix (Elixir), adaptable a otros frameworks.
---

# bayo-error-page

Genera una página de error 500 mobile-first, autocontenida y adaptada visualmente
al proyecto, para sustituir el texto plano de producción.

---

## Flujo paso a paso

### 1. Descubrir el diseño del proyecto

Lee los archivos clave para extraer la identidad visual:

- **Colores**: busca en `assets/css/app.css`, `tailwind.config.js`, o variables CSS.
  Si usa Tailwind, identifica el color primario usado en botones/links.
- **Fuente**: busca la familia tipográfica en el CSS o en el layout root.
- **Nombre de la app**: extrae del layout principal (`root.html.heex`, `<title>`, etc.)
  o del `mix.exs` (campo `:app`).
- **Email de contacto**: busca en el código (`admin`, `soporte`, `contact`, `info`) o
  pregunta al usuario si no lo encuentras.
- **Logo o icono**: si hay un SVG de marca en `priv/static/` o `assets/`, úsalo como
  referencia de color/estilo (no lo incrustes — mantén la página autocontenida).

Si no encuentras información suficiente de diseño, pregunta al usuario:
- Color primario de la app (hex o nombre)
- Email de contacto para errores

### 2. Construir la página 500

Crea `lib/<app>_web/controllers/error_html/500.html.heex` como HTML completo y autocontenido:

**Requisitos obligatorios:**
- HTML completo (`<!DOCTYPE html>` … `</html>`) con estilos **inline** (`<style>` en el `<head>`).
  No dependas del CSS compilado de la app — puede no cargarse si el error es grave.
- Diseño **mobile-first**: funciona en pantallas desde 320px, centrado verticalmente.
- Incluye:
  - Icono visual (SVG inline — usa un triángulo de advertencia o similar, en el color primario)
  - Título en español: "Algo ha fallado" (o en el idioma del proyecto si no es español)
  - Mensaje breve: "Lo sentimos, ha ocurrido un error inesperado. Nuestro equipo ya ha sido notificado."
  - Recuadro de contacto con el email de administración
  - Botón "Volver al inicio" que lleva a `/`
  - Código de error discreto al pie: "Error 500 · <NombreApp>"
- Adapta colores, fuentes y tono al proyecto. Si el proyecto es más corporativo, usa
  un diseño limpio y formal. Si es más consumer/móvil, puede ser más cálido.
- La fuente puede declararse con `font-family` en el CSS inline — no hace falta cargarla
  desde Google Fonts (usa el stack del sistema como fallback).

### 3. Activar el template en Phoenix

Edita `lib/<app>_web/controllers/error_html.ex`:

**Antes:**
```elixir
defmodule AppWeb.ErrorHTML do
  use AppWeb, :html

  # embed_templates "error_html/*"   ← comentado

  def render(template, _assigns) do
    Phoenix.Controller.status_message_from_template(template)
  end
end
```

**Después:**
```elixir
defmodule AppWeb.ErrorHTML do
  use AppWeb, :html

  embed_templates "error_html/*"

  def render(template, _assigns) do
    Phoenix.Controller.status_message_from_template(template)
  end
end
```

La función `render/2` permanece como fallback para otros códigos de error (404, etc.)
que no tengan template propio.

### 4. Verificar que compila

```bash
mix compile
```

Si hay errores de compilación, corrígelos antes de continuar.

### 5. Instrucciones para probar en local

Informa al usuario de cómo ver la página:

1. Cambia temporalmente `debug_errors: true` → `false` en `config/dev.exs`
2. Añade una ruta de test temporal al router:
   ```elixir
   get "/test-500", PageController, :test_error
   ```
   Y en `page_controller.ex`:
   ```elixir
   def test_error(_conn, _params), do: raise("test")
   ```
3. Reinicia el servidor y visita `http://localhost:4000/test-500`
4. **Antes del merge**: revertir `debug_errors` a `true` y eliminar la ruta y acción de test.

### 6. Crear PR

Pregunta al usuario si quiere abrir una PR. Si dice sí:

```bash
git add lib/<app>_web/controllers/error_html.ex \
        lib/<app>_web/controllers/error_html/500.html.heex
git commit -m "Add custom 500 error page for production crashes"
git push -u origin <rama>
gh pr create --title "Add custom 500 error page" \
  --body "Replaces plain-text Internal Server Error with a mobile-friendly error page..."
```

---

## Consideraciones importantes

- **La página debe ser 100% autocontenida.** Sin imports externos, sin `<link>` a CSS de la app,
  sin JS. Solo HTML + `<style>` inline. Esto es crítico: si la app falla gravemente, los assets
  pueden no cargarse.
- **`layout: false`** es la configuración estándar de Phoenix para `render_errors`. La página
  debe incluir su propio `<html>`, `<head>` y `<body>`.
- **No crear página 404 por defecto** a menos que el usuario lo pida explícitamente. El fallback
  de `render/2` la gestiona con texto plano, que es suficiente para 404.
- Si el proyecto no es Phoenix, adapta el paso 3 al mecanismo de error del framework correspondiente.
- Si no hay `error_html.ex` (proyecto no Phoenix), pregunta al usuario cómo gestiona los errores
  antes de continuar.
