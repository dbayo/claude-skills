---
name: bayo-cookiebot-cmp
description: >
  Integra Cookiebot CMP en una app Phoenix que ya tiene Google Tag Manager (GTM)
  y Consent Mode v2. Añade el script de Cookiebot como primer script del <head>,
  antes del bloque de Consent Mode v2 y antes de GTM, siguiendo el orden correcto
  para que el banner de cookies y el modelado de datos de Google funcionen bien.
  Usar cuando el usuario diga "integrar Cookiebot", "añadir banner de cookies",
  "configurar CMP", "GDPR cookies", "Consent Mode v2 con Cookiebot", o cuando
  tenga el script de Cookiebot en mano y quiera pegarlo en la app.
---

# bayo-cookiebot-cmp

Integra el script de Cookiebot CMP en el layout raíz de una app Phoenix,
respetando el orden de carga requerido por Google Consent Mode v2.

---

## Flujo paso a paso

### 1. Obtener el script de Cookiebot

Pide al usuario el script que Cookiebot le ha dado en su panel de implementación.
Tiene esta forma:

```html
<script id="Cookiebot" src="https://consent.cookiebot.com/uc.js"
  data-cbid="XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX"
  data-blockingmode="auto" type="text/javascript"></script>
```

El `data-cbid` es único por cuenta — no hardcodear ningún valor de ejemplo.

### 2. Localizar el layout raíz

Busca el archivo del layout raíz de Phoenix:

```bash
find lib -name "root.html.heex" | head -5
```

Normalmente está en `lib/<app>_web/components/layouts/root.html.heex`.

### 3. Verificar el orden actual de scripts en el <head>

Lee el archivo y comprueba qué scripts hay ya en el `<head>`. El orden correcto es:

1. **Cookiebot** (primero de todo)
2. **Consent Mode v2** — bloque `gtag('consent', 'default', {...})` — debe ir antes de GTM
3. **GTM** — el snippet `(function(w,d,s,l,i){...})(...)`

Si ya hay un bloque de Consent Mode v2 con denegación por defecto, mantenlo — Cookiebot
se encargará de disparar el `consent update` cuando el usuario acepte.

Si no hay Consent Mode v2 ni GTM, añade solo el script de Cookiebot como primer script
del `<head>` y avisa al usuario de que debería configurar también GTM y Consent Mode v2.

### 4. Insertar el script de Cookiebot

Añade el script **justo antes** del comentario o bloque de Consent Mode v2.
Si no existe ese bloque, ponlo como primer `<script>` del `<head>`, después de los
`<meta>`, `<link>` de CSS y fuentes.

Ejemplo de resultado final en el `<head>`:

```heex
<%!-- Cookiebot CMP — debe ser el PRIMER script del <head>.
     Gestiona el banner de consentimiento, cumplimiento RGPD,
     y dispara gtag('consent','update',...) a través de la integración con GTM. --%>
<script id="Cookiebot" src="https://consent.cookiebot.com/uc.js"
  data-cbid="XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX"
  data-blockingmode="auto" type="text/javascript">
</script>

<%!-- Consent Mode v2 — debe ir ANTES de GTM. --%>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('consent', 'default', {
    ad_storage: 'denied',
    ...
  });
</script>

<%!-- Google Tag Manager --%>
<script>
  (function(w,d,s,l,i){...})(window,document,'script','dataLayer','GTM-XXXXXXX');
</script>
```

### 5. Verificar que no hay scripts sueltos de GA4 o Google Ads

Busca si hay algún script `gtag.js` o tag `G-XXXXXXXXX` / `GT-XXXXXXXXX` cargado
directamente en el HTML (fuera de GTM):

```bash
grep -rn "gtag/js\|G-[A-Z0-9]\{8,\}\|GT-[A-Z0-9]\{6,\}" lib --include="*.heex" --include="*.html"
```

Si los encuentra, elimínalos — deben dispararse a través de GTM, no directamente.

### 6. Crear PR

Commitear el cambio y abrir PR con el skill `/ship`.

---

## Consideraciones importantes

- **Orden estricto**: Cookiebot → Consent Mode v2 → GTM. Si el orden es incorrecto,
  las cookies pueden cargarse antes de obtener el consentimiento.
- **`data-blockingmode="auto"`**: bloquea automáticamente todos los scripts de terceros
  hasta que el usuario acepte. Es la opción recomendada.
- **Plan gratuito de Cookiebot**: soporta hasta 50 subpáginas y 1 dominio. Suficiente
  para la mayoría de proyectos en fase inicial.
- **Consent Mode v2**: si el proyecto ya tiene el bloque de denegación por defecto,
  no hace falta duplicarlo — Cookiebot se integra con GTM para el `consent update`.
- **La agencia de marketing**: si hay una agencia configurando GTM, comunicarles
  que el CMP es Cookiebot y el `data-cbid` para que puedan configurar la integración
  en el contenedor de GTM.
