# Mapeo — Portar el "expediente vivo por persona" (A01) de Valentina a Mónica

**Fecha:** 2026-07-30 · **Estado:** mapeo contra código, previo al diseño
**Objetivo:** que Mónica reconozca a la misma persona a través de canales y sesiones, y retome
dónde quedó — lo que en Valentina es el A01 (orquestador + expediente vivo).

> **Aclaración de alcance:** Valentina es el **modelo de referencia**, no el objetivo. Su
> "reestructura en 11 cerebros" **no es un spec de construcción** — el propio análisis de Ernesto
> (`blockpoint/docs/valentina/analisis-2026-07-20`, §4) prohíbe implementar 11 agentes. Los 11
> cerebros son un **modelo funcional de enrutamiento**. Lo que se porta es **A01**: la continuidad.

---

## 1. El criterio de éxito (copiado del A01 de Valentina, traducido a clínica)

> Una paciente escribe por WhatsApp preguntando por HIFU, no cierra. Tres días después **llama**.
> Mónica la reconoce, sabe que ya le explicó HIFU, sabe que quedó pendiente el precio del paquete,
> y **retoma ahí** — no la hace empezar de cero.

**Hoy eso no ocurre.** Voz y texto no comparten memoria de persona.

---

## 2. Mapeo pieza por pieza (Valentina → Mónica)

| Pieza en Valentina | Implementación en Valentina | Equivalente en Mónica | Estado |
|---|---|---|---|
| **Espina de identidad** | `person` + `person_identifier` (UNIQUE `kind,value`, hash con pepper) | **`pacientes` + `channel_identities`** (UNIQUE `channel,external_id`) + trigger `sync_paciente_link` | ✅ **Ya existe** — es una base sólida |
| **Estado del recorrido** | `person_state`: 15 estados, `paso_funnel`, `proxima_accion`, `due_at`, `hechos_confirmados`, `preguntas_pendientes` | — | ❌ **No existe nada per-persona** |
| **Máquina de estados** | `services/persona/fsm.ts`, escritor único, precedencia | — | ❌ No existe |
| **Resolver A01 (entrada)** | `eleven-init-webhook.ts` + `init-context.ts` | **Voz:** `/webhooks/eleven-init` (ya construido) · **Texto:** — | ⚠️ **Mitad**: voz sí, texto no |
| **Variables dinámicas** | 26 vars inyectadas por llamada | **Voz:** nativo de ElevenLabs ✅ · **Texto:** no existe templating | ⚠️ Mitad |
| **Modos** | 3 workflow nodes | 1 solo comportamiento | ❌ (y probablemente **no hace falta** — ver §5) |
| **A10 Guardián** | Guardrails + validación server-side | Guardrails en el prompt (§0 del prompt de voz) | ⚠️ Parcial, sin capa dura |
| **A11 Auditor** | — (brecha declarada también allá) | — | ❌ Ninguno de los dos lo tiene |

**Conclusión del mapeo:** Mónica ya tiene **la espina de identidad** (lo más caro de construir) y
**el resolver del lado de voz**. Lo que falta es **(a)** el estado del recorrido por persona, y
**(b)** que el lado de texto participe de esa identidad.

---

## 3. Las tres capas de Mónica y dónde vive el A01

```
        ┌──────────── ÉLEVÉ Supabase (verdad del negocio) ────────────┐
        │  pacientes · channel_identities · citas · leads             │
        │  ← AQUÍ debe vivir el expediente (ambos cerebros ya la usan)│
        └───────────▲───────────────────────────▲────────────────────┘
                    │                           │
        ┌───────────┴──────────┐     ┌──────────┴────────────┐
        │  nanoclaw  (TEXTO)   │     │ ElevenLabs   (VOZ)    │
        │  WhatsApp/IG/Msgr    │     │ llamadas              │
        │  ❌ sin resolver     │     │ ✅ init webhook = A01 │
        │  ❌ sin identidad    │     │ ✅ dynamic variables  │
        └──────────────────────┘     └───────────────────────┘
```

**Decisión de arquitectura (la más importante):** el expediente **NO** va en nanoclaw ni en
ElevenLabs — va en **Supabase**, porque:
1. Es la única fuente que **ambos cerebros ya consultan** (vía `mcp-monica` / `/tools`).
2. La espina (`pacientes` + `channel_identities`) ya está ahí.
3. El CRM puede mostrarlo sin integraciones nuevas.

nanoclaw y ElevenLabs quedan como **clientes** del expediente, no dueños.

---

## 4. Gaps reales, ordenados por lo que bloquean

### 🔴 Bloqueantes de la continuidad
1. **No hay estado de recorrido por persona.** `pacientes` no tiene ni una columna de estado
   (`primary_intent`, `sentiment`, `tags` existen pero son **por conversación**, no por persona).
   Sin esto no hay "retomar dónde quedó".
2. **nanoclaw no tiene concepto de persona.** Cada `conversation_id` es una identidad nueva y
   aislada (`nanoclaw/src/db/schema.ts:58-60` lo dice explícitamente: *"no linking yet"*).
   Una paciente que vuelve con otro `conversation_id` recibe **amnesia total**.
3. **nanoclaw no tiene variables dinámicas.** `system-prompt-loader.ts` devuelve texto crudo, sin
   interpolación. No hay forma hoy de inyectar contexto por persona al prompt de texto.

### 🟠 Corrupción de identidad (silenciosa, y ya está ocurriendo)
4. **`pacientes` no tiene clave natural.** Cero constraints únicos en `whatsapp`/`telefono`/`correo`.
   `book-appointment` crea pacientes sin verificar duplicados. *(Evidencia de que ya dolió: existen
   `pacientes_limpios`, `_v2`, `_v3` en la base — tres limpiezas manuales.)*
5. **Tres normalizaciones de teléfono distintas y en conflicto:** `search-patient` (solo dígitos),
   `book-appointment` (igualdad exacta), `ingest-call-transcript` (dígitos + quita `52`). Resuelven
   distinto para la misma persona.
6. **El botón "Vincular" del CRM escribe solo `whatsapp_conversations.paciente_id`**, sin tocar
   `channel_identities` → una corrección humana **nunca** se convierte en conocimiento cross-canal.
7. **No hay merge/dedup** en ningún lado.

### ⚠️ Dos bugs propios (introducidos en la fase de voz de esta sesión)
8. **`channel_identities.channel` NO acepta `'llamada'`.** Mi migración amplió el CHECK de
   `whatsapp_conversations` y `whatsapp_messages`, pero **no el de `channel_identities`** → hoy
   **la voz no puede crear una identidad durable**. Hay que agregarlo a la migración.
9. **`ingest-call-transcript` escribe `whatsapp_conversations.paciente_id` directo**, saltándose
   `channel_identities` → la persona resuelta en una llamada **no queda registrada como identidad**;
   la siguiente llamada la resuelve desde cero y `search_patient` nunca la encuentra por voz.
   Debe hacer upsert en `channel_identities` (como sí hace `update-contact-data`).

### 🔵 Riesgos colaterales encontrados
10. **RLS de `pacientes` abierto**: cualquier usuario autenticado puede **leer, escribir y BORRAR**
    todo el historial clínico. Sin `has_role` (a diferencia de `leads`, que sí está protegida).
11. **`groups/monica/` está montado read-write en TODAS las sesiones** de nanoclaw
    (`container-runner.ts:284`) → si el agente escribe memoria ahí, **se mezcla entre pacientes** y
    hay containers concurrentes compitiendo por el mismo directorio. Es el lugar equivocado para
    memoria por persona.
12. **`citas.paciente_id` no tiene FK** → borrar un paciente deja citas huérfanas.

---

## 5. Qué NO conviene portar

- **Los 3 modos de Valentina** (Ventas / Entrenadora / Prospectadora) responden a un negocio MLM.
  El equivalente en clínica sería a lo sumo *paciente nuevo* vs *paciente recurrente* vs *staff*, y
  eso **el expediente ya lo resuelve** (si `paciente_id` existe y tiene historial, es recurrente).
  **No hace falta multiplicar agentes ni workflow nodes.**
- **Los 15 estados** del FSM de Valentina son de un funnel de reclutamiento. Mónica necesita
  **su propio conjunto**, corto y clínico (propuesta a validar): `nuevo → explorando →
  interesado → agendado → anticipo_pendiente → confirmado → atendido → seguimiento` +
  transversales `pausado`, `no_contactar`, `escalado`.
- **La Constitución / Registro de Verdad** — gobernanza propia de BlockPoint, no aplica.

---

## 6. Forma de implementación propuesta (a validar antes de escribir código)

**Capa 1 · Espina (Supabase)**
- Tabla nueva **`paciente_estado`** (1 fila por paciente): `estado`, `paso`, `proxima_accion`,
  `due_at`, `hechos_confirmados` jsonb, `preguntas_pendientes` jsonb, `ultimo_contacto_at`, `canal_ultimo`.
- **Un solo resolver canónico**: función SQL o edge function `resolve-persona(channel, external_id,
  phone)` → `paciente_id`, que **siempre** hace upsert en `channel_identities`. Todos los caminos
  (voz, texto, CRM, book-appointment) pasan por ahí. Mata los gaps 4-7 de una.
- **Escritor único** del estado (como el FSM de Valentina): nadie más lo escribe a mano.

**Capa 2 · Voz (ya casi lista)**
- El init webhook ya resuelve por `caller_id`; sumarle la carga del `paciente_estado` y devolverlo
  como dynamic variables. **Y cablearlo al prompt** — lección aprendida de Valentina, donde el
  expediente existe pero los ~57k chars de prompt no lo mencionan ni una vez.

**Capa 3 · Texto (lo que falta construir)**
- Resolver en `eleve-http.ts:204` (es el único punto donde el payload trae `sender.phone`, `channel`
  y `metadata` intactos; aguas abajo se descartan).
- Inyectar el expediente por el patrón que **ya existe** en nanoclaw: `writeSessionRouting` escribe
  estado por sesión en `inbound.db` antes de cada wake (`container-runner.ts:108`) → sumar un
  `writePersonContext` simétrico, y exponerlo en el system prompt (`destinations.ts:82`).
  **Es el equivalente real de las dynamic variables**, y sigue un patrón ya probado del repo.

---

## 7. Orden sugerido

1. **Arreglar los bugs 8 y 9** (voz no puede crear identidad) — chico y desbloquea lo demás.
2. **Resolver canónico + `channel_identities` siempre** (gaps 4-7): detiene la corrupción que ya ocurre.
3. **`paciente_estado` + escritor único**: habilita la continuidad.
4. **Cablear al prompt de voz** (rápido, ya hay dynamic variables).
5. **Resolver + contexto en nanoclaw** (lo más grande).
6. Colaterales: RLS de `pacientes`, FK de `citas`, memoria por paciente fuera de `groups/monica/`.

## 8. Preguntas abiertas

- ¿El conjunto de estados clínicos de §5 es el correcto, o lo define el negocio?
- ¿`hechos_confirmados` como jsonb libre, o campos tipados (objetivo estético, zona de interés,
  contraindicaciones ya declaradas)?
- ¿La deduplicación de `pacientes` se resuelve en este alcance o es un proyecto aparte?

