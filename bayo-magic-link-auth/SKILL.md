---
name: bayo-magic-link-auth
description: >
  Añade autenticación passwordless con magic link a un proyecto Phoenix (Elixir).
  Genera el sistema completo: una sola vista unificada de acceso/registro (patrón Notion/Linear),
  login con link temporal (15 min), sesiones con remember me (14 días), sudo mode para acciones
  sensibles y emails via Swoosh. El usuario no necesita contraseña para acceder.
  Usar cuando el usuario diga "magic link", "login por email", "autenticación sin contraseña",
  "passwordless", "añadir auth", "sistema de login", "phx.gen.auth".
  Compatible con Phoenix 1.7+. Opcionalmente puede añadirse soporte de contraseña tradicional.
---

# bayo-magic-link-auth

Configura un sistema de autenticación completo basado en magic links para Phoenix:
una sola vista unificada de acceso — si el email no existe se crea la cuenta automáticamente,
si existe se envía el link directamente. El usuario introduce su email, recibe un link
de un solo uso válido 15 minutos, y al hacer click queda autenticado. Sin contraseñas
obligatorias. Patrón idéntico al que usan Notion y Linear.

---

## Flujo paso a paso

### 1. Descubrir el contexto del proyecto

Lee los siguientes archivos para entender la estructura antes de tocar nada:

- `mix.exs` → nombre de la app (`:app`), dependencias existentes
- `config/dev.exs` → configuración actual de email (mailer)
- `lib/<app>_web/router.ex` → rutas existentes y pipelines
- `config/config.exs` → configuración de Ecto y Repo

Extrae:
- **Nombre de la app**: ej. `my_app` → módulo `MyApp`, web `MyAppWeb`
- **¿Ya tiene auth?**: busca módulos `UserAuth`, `UserToken`, `Accounts` — si existen, para y avisa al usuario
- **¿Usa Swoosh?**: si no está en `mix.exs`, habrá que añadirlo
- **Idioma del proyecto**: ¿español o inglés? Adapta textos de emails y flash messages

### 2. Generar la auth base con phx.gen.auth

Ejecuta el generador de Phoenix con magic link:

```bash
mix phx.gen.auth Accounts User users --live false
```

> `--live false` genera controladores clásicos (no LiveView). Si el proyecto ya usa LiveView
> extensivamente, omite el flag y ajusta los pasos siguientes.

Esto crea:
- Migración `users` y `users_tokens`
- `lib/<app>/accounts.ex` (contexto)
- `lib/<app>/accounts/user.ex` (schema)
- `lib/<app>/accounts/user_token.ex` (tokens)
- `lib/<app>/accounts/user_notifier.ex` (emails)
- `lib/<app>_web/user_auth.ex` (plugs)
- `lib/<app>_web/controllers/user_session_controller.ex`
- `lib/<app>_web/controllers/user_registration_controller.ex` ← **se eliminará en paso 5**
- `lib/<app>_web/controllers/user_settings_controller.ex`
- Templates en `lib/<app>_web/controllers/*_html/`
- `test/support/fixtures/accounts_fixtures.ex`

Después ejecuta:

```bash
mix ecto.migrate
```

### 3. Configurar magic link (ajustes clave en UserToken)

Abre `lib/<app>/accounts/user_token.ex` y verifica/ajusta las constantes de expiración:

```elixir
# Tiempo de validez del magic link (corto — es por email)
@magic_link_validity_in_minutes 15

# Cambio de email
@change_email_validity_in_days 7

# Sesión con remember me
@session_validity_in_days 14
```

Verifica que existan estas tres funciones clave:
- `build_email_token/2` — genera token para magic link (context `"login"`)
- `verify_magic_link_token_query/1` — valida el token y lo expira
- `build_session_token/1` — genera token de sesión

Si el generador no los incluyó (versiones antiguas), añádelos manualmente basándote en el patrón:

```elixir
# Genera token de email: retorna {encoded_base64, %UserToken{}}
# El token original va al usuario, el hash SHA256 va a la BD
def build_email_token(user, context) do
  token = :crypto.strong_rand_bytes(@rand_size)
  hashed_token = :crypto.hash(:sha256, token)
  encoded_token = Base.url_encode64(token, padding: false)
  {encoded_token,
    %UserToken{
      token: hashed_token,
      context: context,
      sent_to: user.email,
      user_id: user.id
    }}
end

# Valida magic link: decodifica, hashea, busca en BD y verifica expiración
def verify_magic_link_token_query(token) do
  case Base.url_decode64(token, padding: false) do
    {:ok, decoded_token} ->
      hashed_token = :crypto.hash(:sha256, decoded_token)
      expiry = @magic_link_validity_in_minutes * 60

      query =
        from token in token_and_context_query(hashed_token, "login"),
          join: user in assoc(token, :user),
          where: token.inserted_at > ago(^expiry, "second"),
          where: token.sent_to == user.email,
          select: {user, token}

      {:ok, query}

    :error ->
      :error
  end
end
```

### 4. Configurar el contexto Accounts para magic link

En `lib/<app>/accounts.ex`, verifica que existan estas funciones (el generador las crea si usas Phoenix 1.8+):

```elixir
# Obtiene usuario a partir del token de magic link (sin consumirlo)
def get_user_by_magic_link_token(token) do
  with {:ok, query} <- UserToken.verify_magic_link_token_query(token),
       {user, _token} <- Repo.one(query) do
    user
  else
    _ -> nil
  end
end

# Login real: valida token, confirma usuario si es necesario, expira el token
def login_user_by_magic_link(token) do
  with {:ok, query} <- UserToken.verify_magic_link_token_query(token),
       {user, token_record} <- Repo.one(query) do
    Repo.transaction(fn ->
      # Confirma al usuario si aún no lo está
      user =
        if is_nil(user.confirmed_at) do
          {:ok, confirmed} = confirm_user_with_token(user)
          confirmed
        else
          user
        end

      # Elimina el magic link usado (no se puede reutilizar)
      Repo.delete!(token_record)

      # Devuelve el usuario con authenticated_at (virtual field)
      %{user | authenticated_at: DateTime.utc_now()}
    end)
  else
    _ -> :error
  end
end

# Genera token y envía email de login/confirmación
def deliver_login_instructions(%User{} = user, magic_link_url_fun) do
  {encoded_token, user_token} = UserToken.build_email_token(user, "login")
  Repo.insert!(user_token)
  url = magic_link_url_fun.(encoded_token)

  if is_nil(user.confirmed_at) do
    UserNotifier.deliver_confirmation_instructions(user, url)
  else
    UserNotifier.deliver_magic_link_instructions(user, url)
  end
end
```

### 5. Unificar acceso y registro — eliminar UserRegistrationController

**Elimina** el controlador de registro generado y sus templates, ya que no se usarán:

```bash
rm lib/<app>_web/controllers/user_registration_controller.ex
rm lib/<app>_web/controllers/user_registration_html.ex
rm -r lib/<app>_web/controllers/user_registration_html/
```

En `lib/<app>_web/controllers/user_session_controller.ex`, actualiza `create/2` para que maneje los tres casos. El caso clave es el del magic link: si el email no existe, se crea el usuario automáticamente antes de enviar el link:

```elixir
# Caso 1: Magic link recibido por email (token en params)
def create(conn, %{"user" => %{"token" => token}} = params) do
  case Accounts.login_user_by_magic_link(token) do
    {:ok, user} ->
      conn
      |> put_flash(:info, "Welcome back!")
      |> UserAuth.log_in_user(user, params["user"])
    _ ->
      conn
      |> put_flash(:error, "The log in link is invalid or it has expired.")
      |> redirect(to: ~p"/users/log-in")
  end
end

# Caso 2: Login con email + password (tradicional, opcional)
def create(conn, %{"user" => %{"email" => email, "password" => password} = user_params})
    when is_binary(password) and byte_size(password) > 0 do
  case Accounts.get_user_by_email_and_password(email, password) do
    user when not is_nil(user) ->
      UserAuth.log_in_user(conn, user, user_params)
    _ ->
      conn
      |> put_flash(:error, "Invalid email or password.")
      |> render(:new, error_message: "Invalid email or password.")
  end
end

# Caso 3: Solo email → acceso o registro unificado
# Si el usuario existe → envía magic link; si no existe → crea la cuenta y envía el link
def create(conn, %{"user" => %{"email" => email}}) do
  user =
    Accounts.get_user_by_email(email) ||
      case Accounts.register_user(%{"email" => email}) do
        {:ok, new_user} -> new_user
        {:error, _} -> nil
      end

  if user do
    Accounts.deliver_login_instructions(user, &url(~p"/users/log-in/#{&1}"))
  end

  # Siempre responde igual — no revelar si el email existía o no
  conn
  |> put_flash(:info, "Te hemos enviado un enlace de acceso. Revisa tu email.")
  |> redirect(to: ~p"/users/log-in")
end

# GET /users/log-in/:token → página de confirmación
def confirm(conn, %{"token" => token}) do
  case Accounts.get_user_by_magic_link_token(token) do
    nil ->
      conn
      |> put_flash(:error, "Log in link is invalid or it has expired.")
      |> redirect(to: ~p"/users/log-in")
    user ->
      render(conn, :confirm, user: user, token: token)
  end
end
```

### 6. Configurar el template de login

Edita `lib/<app>_web/controllers/user_session_html/new.html.heex`. La vista es **única** — sin link a "Regístrate" porque la misma página maneja ambos casos:

1. **Magic link** (solo email) — formulario principal:
```html
<h1>Accede o crea tu cuenta</h1>
<p>Introduce tu email y te enviamos un enlace de acceso instantáneo.</p>

<.simple_form :let={f} for={%{}} as={:user} action={~p"/users/log-in"} method="post">
  <.input field={f[:email]} type="email" label="Email" required autofocus />
  <:actions>
    <.button type="submit">Enviar enlace de acceso</.button>
  </:actions>
</.simple_form>
```

2. **Password** (email + password, solo si el proyecto lo soporta — es opcional):
```html
<.simple_form :let={f} for={%{}} as={:user} action={~p"/users/log-in"} method="post">
  <.input field={f[:email]} type="email" label="Email" required />
  <.input field={f[:password]} type="password" label="Password" required />
  <.input field={f[:remember_me]} type="checkbox" label="Keep me logged in for 14 days" />
  <:actions>
    <.button type="submit">Log in</.button>
  </:actions>
</.simple_form>
```

> No añadas un link "¿No tienes cuenta? Regístrate" — es innecesario y confunde al usuario.

Crea `lib/<app>_web/controllers/user_session_html/confirm.html.heex` para confirmar el magic link:

```html
<.header>Confirm your login</.header>

<p>
  You are logging in as <strong><%= @user.email %></strong>.
</p>

<.simple_form for={%{}} as={:user} action={~p"/users/log-in"} method="post">
  <input type="hidden" name="user[token]" value={@token} />
  <:actions>
    <.button name="user[remember_me]" value="true">Keep me logged in</.button>
    <.button name="user[remember_me]" value="false">Log in only this time</.button>
  </:actions>
</.simple_form>
```

### 7. Configurar rutas en el router

En `lib/<app>_web/router.ex`. Sin rutas de `/users/register` — todo pasa por `/users/log-in`:

```elixir
scope "/", MyAppWeb do
  pipe_through [:browser, :redirect_if_user_is_authenticated]

  get  "/users/log-in",            UserSessionController, :new
  post "/users/log-in",            UserSessionController, :create
  get  "/users/log-in/:token",     UserSessionController, :confirm  # ← magic link
end

scope "/", MyAppWeb do
  pipe_through :browser

  delete "/users/log-out", UserSessionController, :delete
end
```

Si el generador añadió rutas de `/users/register`, **elimínalas** del router.

### 8. Configurar emails con Swoosh

**En `mix.exs`**, añadir si no están:

```elixir
{:swoosh, "~> 1.16"},
{:gen_smtp, "~> 1.2"},   # Para SMTP en producción
```

**En `config/dev.exs`**, usar el adaptador local (no envía emails reales):

```elixir
config :my_app, MyApp.Mailer, adapter: Swoosh.Adapters.Local
```

**En `config/runtime.exs`**, para producción con Gmail:

```elixir
if config_env() == :prod do
  config :my_app, MyApp.Mailer,
    adapter: Swoosh.Adapters.SMTP,
    relay: "smtp.gmail.com",
    port: 587,
    username: System.get_env("MAILER_SMTP_USERNAME"),
    password: System.get_env("MAILER_SMTP_PASSWORD"),
    tls: :always,
    auth: :always

  config :my_app, :mailer_from,
    name: System.get_env("MAILER_FROM_NAME", "MyApp"),
    email: System.get_env("MAILER_FROM_EMAIL")
end
```

**Variables de entorno necesarias en producción:**
```
MAILER_SMTP_USERNAME   → tu_correo@gmail.com
MAILER_SMTP_PASSWORD   → App Password de Google (no la contraseña normal)
MAILER_FROM_NAME       → "NombreApp"
MAILER_FROM_EMAIL      → tu_correo@gmail.com
```

> Para Gmail: activa 2FA → Google Account → Security → App Passwords → genera una para "Mail"

**En `lib/<app>/accounts/user_notifier.ex`**, asegúrate de tener:

```elixir
defp deliver(recipient, subject, body) do
  {from_name, from_email} = get_from()

  email =
    new()
    |> to(recipient)
    |> from({from_name, from_email})
    |> subject(subject)
    |> text_body(body)

  with {:ok, _metadata} <- MyApp.Mailer.deliver(email) do
    {:ok, email}
  end
end

defp get_from do
  case Application.get_env(:my_app, :mailer_from) do
    nil -> {"MyApp", "noreply@myapp.com"}
    cfg -> {cfg[:name], cfg[:email]}
  end
end

def deliver_magic_link_instructions(user, url) do
  deliver(user.email, "Log in instructions", """
  Hi #{user.email},

  You can log into your account by visiting the URL below:

  #{url}

  This link is valid for 15 minutes.

  If you didn't request this, please ignore this email.
  """)
end

def deliver_confirmation_instructions(user, url) do
  deliver(user.email, "Confirmation instructions", """
  Hi #{user.email},

  Welcome! You can confirm your account and log in by visiting the URL below:

  #{url}

  This link is valid for 15 minutes.
  """)
end
```

### 9. Añadir endpoint del mailer local (solo dev)

En `lib/<app>_web/router.ex`:

```elixir
if Mix.env() == :dev do
  forward "/dev/mailbox", Plug.Swoosh.MailboxPreview
end
```

Ahora en desarrollo puedes ver los emails en `http://localhost:4000/dev/mailbox`.

### 10. Verificar que compila y las rutas funcionan

```bash
mix compile
mix phx.routes | grep users
```

Comprueba que aparezcan (sin rutas de `/users/register`):
```
GET    /users/log-in           UserSessionController :new
POST   /users/log-in           UserSessionController :create
GET    /users/log-in/:token    UserSessionController :confirm
DELETE /users/log-out          UserSessionController :delete
```

### 11. Probar el flujo completo en local

1. Arranca el servidor: `mix phx.server`
2. Ve a `http://localhost:4000/users/log-in`
3. Introduce un email nuevo (sin cuenta) y envía — debe crearse el usuario y enviar el link
4. Abre `http://localhost:4000/dev/mailbox` → verás el magic link
5. Haz click en el link → te lleva a la página de confirmación
6. Haz click en "Keep me logged in" → autenticado
7. Verifica logout en `/users/log-out`
8. Vuelve a `/users/log-in` con el mismo email → debe enviarte otro link (usuario ya existe)

### 12. Crear PR

Pregunta al usuario si quiere abrir una PR:

```bash
git add lib/ test/ priv/repo/migrations/ config/
git commit -m "Add magic link authentication with Swoosh email delivery"
git push -u origin <rama>
gh pr create \
  --title "Add magic link authentication" \
  --body "Implements passwordless login via email magic links (15 min expiry), session management with remember me (14 days), and email delivery via Swoosh."
```

---

## Consideraciones importantes

- **Vista unificada acceso/registro**: una sola página `/users/log-in` — si el email no existe se crea la cuenta antes de enviar el link. No hay `/users/register`. Patrón idéntico al de Notion y Linear.
- **No enumerar usuarios**: el endpoint siempre responde con el mismo mensaje independientemente de si el email existía o es nuevo.
- **El token se hashea en BD**: el usuario recibe el token en Base64, en la BD se guarda su SHA256. Si alguien roba la BD, los tokens son inútiles.
- **Token de un solo uso**: `login_user_by_magic_link/1` elimina el token tras usarlo. No se puede reutilizar.
- **Remember me**: cookie de 14 días. Sin remember me, la sesión dura hasta cerrar el navegador (token de sesión válido 14 días igualmente, pero la cookie no persiste).
- **Sudo mode**: para acciones sensibles (cambiar email, cambiar contraseña), exige que el último login haya sido en los últimos 20 minutos. Configura con el plug `require_sudo_mode` en `UserAuth`.
- **Gmail App Password**: en producción con Gmail, NUNCA usar la contraseña normal de Google. Requiere 2FA activado y generar un App Password específico.
- **Si el proyecto ya tiene auth**: no ejecutes `phx.gen.auth` — detecta los módulos existentes y aplica solo los cambios faltantes (magic link functions, confirm route, confirm template).
- **Adaptador SMTP alternativo**: si no usan Gmail, Swoosh soporta Postmark, Mailgun, SendGrid, AWS SES. Ajusta el adaptador en `runtime.exs`.
- **Contraseña opcional**: el generador incluye soporte de password por defecto. Si quieres auth 100% passwordless, elimina el formulario de password del template y las funciones `get_user_by_email_and_password` del controlador (pero mantenlas en el contexto por si acaso).
