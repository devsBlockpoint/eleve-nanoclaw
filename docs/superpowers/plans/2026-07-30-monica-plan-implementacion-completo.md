# Plan de implementación completo — Mónica

**Fecha:** 2026-07-30 · **Estado:** propuesta
**Alcance:** todo lo detectado en la sesión de arquitectura (canal de voz, CRM, expediente por
persona, fuente única de prompt) ordenado por dependencia y por lo que hoy le falla a la paciente.

> **Encuadre.** El objetivo NO es "implementar 11 cerebros". Los cerebros A03–A07 (educir, educar,
> responder, evidencia, cierre) **ya viven en el prompt y las KBs de Mónica** — son comportamiento,
> no software. Lo que falta construir es **A01 (expediente), A08 (seguimiento), A10 (guardián duro)
> y A11 (auditor)**, más los bloqueantes funcionales. Los **3 modos de Valentina no se portan**: son
> de un negocio con tres audiencias; Mónica tiene una, y "nueva vs recurrente" lo resuelve el
> expediente.

---

## FASE 0 · Bloqueantes — promesas rotas al paciente
*Esto no es arquitectura: es que hoy Mónica dice cosas que no cumple.*

**0.1 · Escalamiento a humano desde voz** 🔴
`escalate_to_human` exige un `conversation_id` de `whatsapp_conversations` que **no existe en una
llamada**. Ante una queja, una bandera roja clínica o datos sensibles, **Mónica-voz no puede
derivar** — y su propio prompt le ordena hacerlo.
→ Edge function que acepte origen "voz" (con `caller_id` + `conversation_id` de ElevenLabs) y
registre en `escalation_queue`. Exponerla como tool.

**0.2 · Tool de envío de WhatsApp + plantillas** 🔴
En cada cierre Mónica dice *"le mando los datos por WhatsApp"* — **y no se manda nada**. Es la vía
por la que el paciente recibe la CLABE (que en voz no se dicta).
→ `enviar_plantilla_whatsapp` (edge function + tool) + someter a Meta las plantillas **P0**
(`datos_pago_apartado`, `confirmacion_cita`). Ya especificadas en `docs/voz/whatsapp-plantillas-plan.md`.

**0.3 · Los dos bugs de identidad en voz** 🟠 *(archivos locales, sin aplicar)*
- `channel_identities.channel` **no acepta `'llamada'`** → la voz no puede crear identidad durable.
  Falta ampliar ese CHECK en la migración.
- `ingest-call-transcript` escribe `paciente_id` directo, **saltándose `channel_identities`** → cada
  llamada re-resuelve desde cero y `search_patient` nunca encuentra a la persona por voz.

**Dependencias:** ninguna. **Desbloquea:** exponer Mónica-voz a pacientes reales.

---

## FASE 1 · Prompt — estructura ElevenLabs + fuente única

**1.1 · Reestructurar el prompt de voz al framework oficial** ⭐ *(nuevo — mejora resultados)*
La guía de ElevenLabs recomienda seis bloques con nombres específicos, **y el modelo está afinado
para prestar atención extra a un encabezado `# Guardrails`**:

```
# Personality   → quién es (rol + rasgos)
# Environment   → dónde/cómo opera (llamada telefónica, clínica, Los Mochis)
# Tone          → estilo de habla
# Goal          → objetivo y pasos del flujo (Lectura → apartado → agendar)
# Guardrails    → reglas no negociables  ← el modelo les da peso extra
# Tools         → cada tool con "cuándo usarla", formato de parámetros y manejo de errores
```
El prompt actual (~13.8k chars ≈ 3.5k tokens) **excede el presupuesto recomendado (<2k tokens)**,
lo que según la doc sube latencia y costo. Al reestructurar, más conocimiento baja a la KB.

**1.2 · Normalización de números por plataforma** ⭐ *(arregla un problema real)*
ElevenLabs tiene `text_normalisation_type`. Hoy estamos resolviendo el problema **por prompt**
("di las cifras en palabras"). La opción `"elevenlabs"` usa su normalizador de TTS, **más confiable**
y sin gastar instrucciones. → Probar y, si mejora, adoptarla y quitar esas reglas del prompt.

**1.3 · Proyector de fuente única**
Ya especificado en `docs/superpowers/specs/2026-07-30-prompt-fuente-unica-proyecciones.md`:
`mcp-monica` sirve `GET /prompt/texto` (nanoclaw ya recarga c/1h) y hace push programado a
ElevenLabs. Marcas `[[CANON]]` / `[[CONOCIMIENTO]]` / `[[SOLO-TEXTO]]` — **ya solicitadas al negocio**.

**Dependencias:** 1.3 se beneficia de las marcas, pero **degrada seguro** sin ellas.
**Desbloquea:** que nada vuelva a desincronizarse (la v4.0 quedó solo en texto por esto).

---

## FASE 2 · Identidad sólida
*Sin esto, el expediente de la Fase 3 se construye sobre arena.*

**2.1 · Un solo resolver canónico**
Hoy hay **tres normalizaciones de teléfono en conflicto** (`search-patient` solo dígitos,
`book-appointment` igualdad exacta, `ingest-call-transcript` dígitos + quita `52`).
→ `resolve-persona(channel, external_id, phone)` como única puerta: resuelve **y siempre hace upsert
en `channel_identities`**. Todos los caminos pasan por ahí (voz, texto, CRM, book-appointment).

**2.2 · Clave natural en `pacientes`**
Cero constraints únicos en teléfono/whatsapp/correo → `book-appointment` crea duplicados en silencio.
*Evidencia de que ya dolió: existen `pacientes_limpios`, `_v2` y `_v3` en la base — tres limpiezas
manuales.* → Índice único sobre teléfono normalizado + dedup previa.

**2.3 · El botón "Vincular" del CRM debe escribir `channel_identities`**
Hoy solo escribe `whatsapp_conversations.paciente_id` → **las correcciones humanas nunca se vuelven
conocimiento cross-canal**. (Requiere edge function: el RLS no deja escribir identidades desde el browser.)

**Dependencias:** 0.3. **Desbloquea:** Fase 3.

---

## FASE 3 · A01 — El expediente vivo *(la pieza arquitectónica de fondo)*

**Criterio de éxito:** *una paciente pregunta por HIFU por WhatsApp, no cierra; tres días después
llama. Mónica la reconoce, sabe qué se le explicó y qué quedó pendiente, y retoma ahí.*

**3.1 · Tabla `paciente_estado`** (1 fila por paciente, escritor único)
`estado` · `paso` · `proxima_accion` · `due_at` · `hechos_confirmados` jsonb ·
`preguntas_pendientes` jsonb · `ultimo_contacto_at` · `canal_ultimo`.
Estados propuestos (a validar con el negocio): `nuevo → explorando → interesado → agendado →
anticipo_pendiente → confirmado → atendido → seguimiento`, transversales `pausado`, `no_contactar`,
`escalado`.

**3.2 · Cablear a voz** — el init webhook ya resuelve por `caller_id`; sumarle la carga del estado
como dynamic variables **y referenciarlo en el prompt**.
> ⚠️ **Lección de Valentina:** allá el expediente existe hace días y los ~57k caracteres de prompt
> **no lo mencionan ni una vez** — la fontanería sin cablear no sirve. No repetir ese error.

**3.3 · Cablear a texto (nanoclaw)** — lo más grande, porque nanoclaw **no tiene concepto de persona**
(cada `conversation_id` es identidad nueva y aislada; una paciente que vuelve recibe amnesia total).
→ Resolver en `eleve-http.ts:204` (único punto con `sender.phone` + `channel` intactos) → escribir el
contexto en `inbound.db` antes de cada wake (patrón ya probado: `writeSessionRouting`) → exponerlo en
el system prompt.

**Dependencias:** Fase 2. **Desbloquea:** Fase 4.

---

## FASE 4 · A08 — Seguimiento
Con `proxima_accion` + `due_at` poblados, un job dispara el reenganche **con permiso y valor nuevo**
(nunca "¿ya lo pensó?"). Es la diferencia entre un lead que se enfría y uno que vuelve.
**Dependencias:** Fase 3.

---

## FASE 5 · Gobierno

**5.1 · A10 — Guardián como capa dura**
Hoy los guardrails viven **dentro** del prompt comercial. Lo mínimo con impacto real: validación
**server-side** en las tools que mutan (que el backend rechace lo que el prompt prohíbe), en vez de
confiar solo en que el modelo obedezca.

**5.2 · A11 — Auditor**
**Ni Mónica ni Valentina lo tienen.** Es lo que responde *"¿Mónica está cumpliendo el canon?"* sin
que alguien escuche llamadas a mano. Con el post-call webhook ya construido, los transcripts **ya
llegan**: falta la rúbrica + un set dorado de casos + evaluación por lote.
> Hoy los problemas se detectan solo porque el dueño prueba y escucha. Eso no escala.

---

## TRANSVERSAL · Telefonía y CRM (en curso)

| Pendiente | Estado |
|---|---|
| KYC Telnyx | ✅ hecho |
| SIP Connection + DID → ElevenLabs | ✅ probado (llamada directa al DID funciona) |
| **Desvío Telcel** | ❌ **falla** — pendiente el test decisivo: desviar a otro celular para aislar si es la línea o el formato |
| CRM: llamadas + transcripts | ⏸️ implementado local, esperando revisión de Lovable |
| Redeploy `nanoclaw-host` (fix `--rm`) | ⏸️ pendiente, en rato de poco tráfico |

---

## Orden recomendado y por qué

1. **Fase 0** — son promesas rotas al paciente; nada de lo demás importa si Mónica no puede escalar
   una bandera roja clínica.
2. **Fase 1** — barata, mejora resultados de inmediato (estructura + normalización) y **evita que
   todo lo que siga se desincronice**.
3. **Fase 2** — la corrupción de identidad **ya está ocurriendo**; cada día que pasa hay más duplicados.
4. **Fase 3** — la pieza de fondo, y la que da el salto de experiencia.
5. **Fases 4-5** — sobre esa base.

## Decisiones abiertas
- ¿El set de estados clínicos de 3.1 es correcto, o lo define el negocio?
- ¿`hechos_confirmados` libre (jsonb) o campos tipados (objetivo estético, zona, contraindicaciones)?
- ¿La dedup de `pacientes` (2.2) entra en este alcance o es proyecto aparte?
- ¿Mónica-voz sale a pacientes reales antes o después de la Fase 3?
</content>
