---
name: bayo-tarea-to-skill
description: >
  Convierte la tarea de la sesión actual (o de una PR) en una skill reutilizable
  en /Users/bayo/work/claude_skills/, para poder replicarla en otros proyectos.
  Usar cuando el usuario diga "conviértelo en skill", "haz una skill de esto",
  "quiero poder replicar esto", "bayo-tarea-to-skill", o quiera automatizar
  una tarea que acaba de hacer.
---

# bayo-tarea-to-skill

Analiza lo que se acaba de hacer en la sesión (o en una PR) y genera una skill
`bayo-*` reutilizable que permita replicar esa tarea en cualquier proyecto futuro.

---

## Flujo paso a paso

### 1. Entender la tarea realizada

Repasa la conversación actual o, si el usuario pasa una URL de PR, léela con `gh pr view <url>`.

Extrae:
- **¿Qué se hizo?** — el cambio concreto (añadir página de error, crear endpoint, configurar algo…)
- **¿Por qué?** — el problema que resolvía
- **¿Qué archivos se tocaron?** — rutas, controladores, configs, assets…
- **¿Qué decisiones de diseño o adaptación se tomaron?** — colores, textos, emails, nombres…
- **¿Qué pasos fueron manuales y cuáles pueden automatizarse?**

### 2. Proponer nombre y alcance

Sugiere un nombre `bayo-<nombre-descriptivo>` en kebab-case, corto y accionable.
Ejemplos: `bayo-error-page`, `bayo-auth-setup`, `bayo-sentry-config`.

Confirma con el usuario antes de crear el archivo.

### 3. Crear la skill

Crea `/Users/bayo/work/claude_skills/bayo-<nombre>/SKILL.md` siguiendo esta estructura:

```markdown
---
name: bayo-<nombre>
description: >
  Descripción clara de qué hace y cuándo activarse.
  Incluir palabras clave que el usuario usaría para pedirlo.
---

# bayo-<nombre>

Descripción breve del propósito.

---

## Flujo paso a paso

### 1. Descubrir el contexto del proyecto
[Qué leer/preguntar para adaptar la tarea al proyecto destino]

### 2. [Pasos concretos...]
[Código, comandos, archivos a crear/editar]

### N. Crear PR
[Si aplica, instrucciones para commitear y abrir PR]

---

## Consideraciones importantes
[Lo que NO es obvio, errores comunes, decisiones de diseño]
```

**Principios para escribir la skill:**
- **Genérica pero adaptable**: no hardcodear nombres, colores ni rutas del proyecto origen.
  La skill debe descubrir esos valores leyendo el proyecto destino o preguntando al usuario.
- **Autocontenida**: quien la ejecute no necesita contexto de esta sesión.
- **Accionable**: pasos concretos con comandos reales, no descripciones vagas.
- **Considera el diseño**: si la tarea produce algo visible (UI, página, email…),
  la skill debe incluir un paso para adaptar el diseño al proyecto destino.

### 4. Commitear y pushear

```bash
cd /Users/bayo/work/claude_skills
git add bayo-<nombre>/
git commit -m "Add bayo-<nombre> skill: <descripción de una línea>"
git push
```

No abrir PR — las skills van directamente a `main`.

### 5. Confirmar

Informa al usuario:
- Nombre de la skill creada
- Ruta del archivo
- Cómo invocarla: `/<nombre>`
- Qué hará cuando se ejecute en otro proyecto

---

## Consideraciones importantes

- Si la tarea es muy específica del proyecto (lógica de negocio única), indícaselo al
  usuario — puede no tener sentido convertirla en skill genérica.
- Si la tarea involucra secretos, credenciales o datos sensibles del proyecto origen,
  asegúrate de que la skill los parametriza (pide al usuario que los provea) en vez
  de hardcodearlos.
- Las skills deben vivir en `/Users/bayo/work/claude_skills/bayo-<nombre>/SKILL.md`.
- Mira las skills existentes en ese directorio para mantener consistencia de estilo.
