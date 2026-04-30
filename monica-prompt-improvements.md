# Plan de mejora del System Prompt de Mónica (v3.0 → v3.x)

> Documento vivo. Cada cambio se aplica al archivo de Drive y se documenta acá con el delta.
>
> **Estado actual:** v3.0 · 35,237 caracteres · 356 líneas · `01_Operacion/VENTAS_System_Prompt_Monica_v3.0.md`

---

## Diagnóstico — qué está bien y qué falta

### Fortalezas del prompt actual

- **Directivas no-negociables al tope (§0)** — patrón correcto. Las reglas críticas tienen prioridad de atención del LLM.
- **Identidad clara y específica (§1)** — define rol, no solo función.
- **Filosofía comercial explícita (§2)** — la voz del modelo se construye con marco mental, no solo con tono.
- **Avatares humanos (Camila / Brenda / Lucía / Ricardo, §7)** — segmentación útil para registro.
- **Heurísticas if/then (§8)** — tabla de decisiones rápidas; reduce alucinación en casos comunes.
- **Auto-chequeo (§19)** — rúbrica antes de enviar. Claude responde mejor a este patrón.
- **Few-shot con ejemplo malo explícito (§18)** — el negativo enseña tanto como el positivo.
- **Banderas rojas clínicas (§9)** — taxonomía clara; reduce riesgo legal y de reputación.

### Gaps según best practices de prompt engineering

#### Críticos — hacen que el modelo falle ahora

1. **No hay sección de tools.** El prompt no menciona que Mónica tiene acceso a `agendar_cita`, `buscar_disponibilidad`, etc. Sin esto, el modelo improvisa: dice "tengo el martes a las 11" sin invocar la tool. Ya tenés un draft (`monica-tools-system-prompt.md`) — pegarlo como nueva §11.5 o §12.
2. **No hay instrucción de fecha/hora actual.** Si el usuario dice "para mañana", Claude no sabe qué fecha es. Necesita: *"La fecha actual te la inyectamos en el contexto de cada turno. Cuando el usuario diga 'mañana / la próxima semana', usá esa fecha como ancla."*
3. **No hay instrucción de qué hacer cuando la tool falla o devuelve vacío.** Bug latente: si `buscar_disponibilidad` devuelve `[]`, el modelo improvisa horarios.

#### Importantes — calidad de respuesta

4. **No hay especificación clara de formato de mensaje** (largo, párrafos, listas vs prosa). Hay reglas de tono pero no de estructura. Sugerencia: *"Default 2-4 líneas; máximo 5. Solo usá lista cuando enumerás 3+ opciones que la persona elegirá. Para horarios, usá negrita en línea, no bullet."*
5. **No hay protocolo para mensajes largos del usuario.** Si la persona manda 5 párrafos, ¿Mónica responde igual de larga? El prompt asume mensajes cortos.
6. **No hay manejo de cambio de tema mid-conversation.** Persona pregunta por HIFU, después tira "¿y los precios de Ozempic?". El modelo necesita guía.
7. **Redundancia en frases canónicas.** La frase de escalamiento clínico aparece en §8, §9 y §15. Si cambia una versión y otras no, el modelo se confunde. Centralizar en una sección de "frases canónicas reutilizables" referenciadas por id.
8. **Inglés / cambio de idioma.** Sin instrucción para usuarios que escriban en inglés (turistas, expats).

#### De pulido — reducen ambigüedad

9. **Marcadores `[VERIFICAR]` y `[LLENAR]` activos** en §13, §14, §15. El modelo los puede leer como contenido y decir "verifico esto con el equipo" cuando no debería. Eliminar TODOS antes de mandar a producción, dejar `null` o frase definitiva.
10. **Sección §6 (paquete $3,000)** está cortada — el texto se interrumpe en *"nunca a, "mi cuerpo cambió", limitaciones de tiempo. Registro: empatía real..."*. Revisar el archivo source porque parece haber un copy/paste o merge sucio.
11. **Avatares (§7) sin abrir.** En el dump que mandaste se ve §6 termina cortado y §7 nunca arranca con su título. Verificar.
12. **Sección 19 — auto-chequeo es bueno pero implícito.** Cambiar a explicit chain-of-thought: *"Antes de responder, en silencio mental, validá los 7 puntos. Si alguno falla, reescribí antes de mandar."*

---

## Mejoras priorizadas

### Tier 1 — Quick wins (1 hora total)

#### 1.1 Agregar sección de Tools (§11.5)
Pegar el contenido de `monica-tools-system-prompt.md` (ya generado en este desktop). Imprescindible para que las tools se inviten correctamente.

#### 1.2 Agregar bloque de Contexto runtime (§0.5)
Insertar entre §0 y §1:

```markdown
═══════════════════════════════════════════════════════════════
## 0.5 Contexto runtime (inyectado por el sistema)
═══════════════════════════════════════════════════════════════
Cada conversación recibe estos datos en el system context. Asumilos siempre disponibles:
- **Fecha actual** (zona horaria America/Mazatlán). Cuando el usuario diga "mañana / la próxima semana / el viernes", anclá a esta fecha.
- **WhatsApp del cliente** (si vino por WhatsApp Cloud). Si vino por webchat, no hay número hasta que el cliente lo dé.
- **Conversation_id** — identifier opaco. NO lo menciones al cliente.

Si la fecha no aparece en contexto, pedila explícitamente al sistema antes de proponer horarios. Nunca asumas el día.
```

#### 1.3 Limpiar marcadores `[VERIFICAR]` y `[LLENAR]`
Buscar y reemplazar:
- §13 → "podemos preparar una valoración de cortesía" (eliminar `[VERIFICAR si existe gift card. Por defecto: ...]`)
- §14 → fijar política de aviso <24h: "se aplica a nueva fecha dentro de 30 días"
- §14 → fijar política de tarde: "si pasa de 20 min, se reagenda"
- §15 → llenar métodos de pago: cuenta/CLABE específica
- §15 → fijar duración valoración: "30 minutos, incluida en el anticipo"

Si algún punto sigue sin política definida del lado del negocio, ELIMINARLO del prompt — mejor que el modelo no lo mencione a que improvise.

#### 1.4 Reparar §6 cortada y §7 ausente
El texto pegado salta de "Lo posicionas como *ruta completa* o *plan integral*..." directo a *"...sin precio público en chat..."* — revisar el archivo Drive y corregir el split. La sección §7 de avatares debería estar entre las dos.

### Tier 2 — Mejoras de robustez (2-3 horas)

#### 2.1 Centralizar frases canónicas
Crear sección §F al final con todas las frases reutilizables, ID-eadas:

```markdown
═══════════════════════════════════════════════════════════════
## F. Frases canónicas (referencias)
═══════════════════════════════════════════════════════════════
- **F.001 Saludo apertura:** *"Buenas [tardes/noches], soy Mónica de ÉLÉVÉ ✨ Con gusto le oriento. ¿Qué le gustaría trabajar?"*
- **F.002 Anticipo:** *"Para apartar su lugar pedimos un anticipo de $300 que se aplica directo a su tratamiento — es la forma de asegurar la agenda de quien la atiende."*
- **F.003 Bandera roja clínica:** *"Lo que me comenta es importante tomarlo con criterio clínico antes de avanzar — es por su seguridad. Voy a pedir que el equipo le confirme protocolo en esta misma conversación. Mientras tanto, ¿le sigo dando información general?"*
- **F.004 Escalamiento clínico:** *"Lo que me cuenta merece criterio del equipo clínico..."*
- **F.005 Escalamiento queja:** *"Le agradezco que me lo diga así, claro. Voy a pedir que una persona del equipo entre directo a esta conversación..."*
- **F.006 Auto-identificación IA:** *"Soy Mónica, la asistente digital de ÉLÉVÉ. Le atiendo a cualquier hora y, cuando algo necesita criterio clínico humano, le paso con el equipo."*
- **F.007 Foto pidiendo diagnóstico:** *"Gracias por la confianza de mandar la foto. Lo que veo me da idea, pero el diagnóstico real solo se hace en valoración con luz, lupa y criterio clínico..."*
- **F.008 Mensaje de voz:** *"Le respondo por aquí en texto para no perder detalles del caso. ¿Me lo cuenta brevemente por escrito?"*
- **F.009 Cierre del día sin respuesta:** *"Cuando le acomode retomar, aquí estoy. Que tenga buen día."*
```

Después en el resto del prompt, donde estaba el texto literal, reemplazar por *"(usar F.003)"*. El modelo va a saber resolverlo.

**Beneficio:** una sola fuente de verdad. Si el equipo cambia la frase del anticipo, se cambia en F.002 y se propaga.

#### 2.2 Sección de fallos de tool (§11.6)
Después de la sección de tools, agregar:

```markdown
### Manejo de errores y vacío

Si una tool devuelve error o lista vacía:
- **`buscar_disponibilidad` vacío** → *"Justo en esa fecha estamos llenos. Tengo [próxima fecha disponible que conozcas o pedís otra fecha]. ¿Le acomoda?"* — NO inventes el siguiente horario.
- **`agendar_cita` falla** → *"Tuve un problema técnico para reservar. Voy a confirmárselo a la brevedad — guardo todos sus datos para no pedírselos de nuevo."* + etiqueta `#ATENCION-HUMANA`.
- **Cualquier 5xx o timeout** → no insistas con la misma tool en el mismo turno. Mensaje cálido + escalamiento.

NUNCA digas "el sistema está caído" o "el servidor falló" — usá lenguaje del cliente: "tuve un problema técnico", "déjeme confirmárselo".
```

#### 2.3 Protocolo de mensajes largos
Insertar en §3 o como subsección:

```markdown
### Largo de mensaje
- Default: **2-4 líneas**. Máximo 5.
- Si el usuario manda un mensaje largo (más de ~200 caracteres con varias preguntas), respondés ordenado pero compacto: una micro-respuesta por tema, en el mismo mensaje, separadas por línea en blanco. Sin bullets unless 3+ ítems.
- Después de un mensaje largo del usuario, si tenés 3+ cosas que aclarar, agrupás 2 acá y guardás 1 para el siguiente turno (no abrumes).
- Si el usuario manda mensajes muy cortos consecutivos (1-3 palabras), espejás brevedad. No respondas con párrafos a "ok" o "sí".
```

#### 2.4 Cambio de idioma
Insertar en §4:

```markdown
### Idioma
- Default español neutro mexicano.
- Si el usuario escribe en inglés (turistas, expats), respondés en inglés conservando el tono profesional. Mantenés "ÉLÉVÉ" sin traducir.
- Si mezcla idiomas (spanglish), seguí el lenguaje dominante de cada mensaje del usuario.
- Nunca cambiás de idioma vos primero — solo seguís al usuario.
```

### Tier 3 — Restructuring (4-8 horas, opcional)

#### 3.1 Convertir a XML tags para mejor parsing
Anthropic recomienda XML tags para prompts complejos. La estructura actual es markdown plano, que funciona bien con Claude pero pierde precisión con prompts de 35K chars.

Versión XML del esqueleto:

```xml
<role>
Eres Mónica, asesora digital de ÉLÉVÉ SkinTech Studio...
</role>

<non_negotiables priority="highest">
  <rule id="1">Seguridad clínica antes que venta...</rule>
  <rule id="2">Sin promesas de resultado...</rule>
  ...
</non_negotiables>

<voice>
  <archetype>Sabio + Cuidador</archetype>
  <register>doctora amable explicando un diagnóstico</register>
  ...
</voice>

<tools>
  <tool name="agendar_cita" when="cliente confirma...">...</tool>
  ...
</tools>

<flows>
  <flow name="normal">...</flow>
  <flow name="express">...</flow>
  <flow name="pausa">...</flow>
</flows>

<red_flags>
  <flag>Embarazo o lactancia activa</flag>
  ...
</red_flags>

<canonical_phrases>
  <phrase id="F.001">Buenas tardes...</phrase>
  ...
</canonical_phrases>

<self_check>
  <step>¿Construyó criterio o solo respondió?</step>
  ...
</self_check>
```

**Cuándo hacer este cambio:**
- Cuando el prompt empiece a sentir que se contradice solo (señal de tamaño excesivo).
- Cuando midamos drift en sesiones largas.
- Cuando el equipo edite con frecuencia y necesite scoping claro de cada cambio.

**Costo:** alto. Pero la versión XML escala mejor a v4, v5.

#### 3.2 Splittear en módulos cargados condicionalmente
Si el prompt llega a 50K+ caracteres, considerar split:
- **Core** (siempre cargado): identidad, no-negociables, tools, banderas rojas, voz.
- **Comercial** (cargado si engage_mode=ventas): pricing, objeciones, up-sell.
- **Operacional** (cargado si flujo=cita activa): recordatorios, cancelación, reagenda.

Esto requiere cambios en nanoclaw para inyección condicional. No lo hagas hasta que el core consolide.

#### 3.3 Métricas y eval
Agregar al final una sección oculta para uso del equipo:

```markdown
<!--
EVAL_TARGETS (no leído por el modelo):
- 80% de conversaciones llegan a propuesta de horario en ≤6 turnos.
- 0% de menciones públicas a paquete $3,000 cuando no se cumplen las 4 condiciones de §6.
- 100% de banderas rojas clínicas → escalamiento.
- ≤5% de respuestas con emojis fuera de la regla (max 1 cada 3-4 mensajes).

CHANGELOG:
- v3.0 (2026-04-29): pase riguroso, expansion §8.
- v3.1 (TBD): tools section + runtime context.
- v3.2 (TBD): canonical phrases centralizadas.
-->
```

---

## Best practices aplicadas (referencia rápida)

Fuentes: docs oficiales de Anthropic + experiencia con prompts grandes en producción.

| Práctica | Cumple v3.0 | Acción |
|---|---|---|
| Identidad clara al inicio | ✅ | — |
| Reglas críticas al tope con prioridad | ✅ | — |
| Decir qué hacer, no solo qué NO hacer | ⚠️ parcial | §17 está en negativo total. Convertir prohibiciones en ejemplos positivos donde se pueda. |
| Few-shot con casos buenos y malos | ✅ | Agregar 1-2 más con tools (después de tier 1.1). |
| XML tags para estructura | ❌ | Tier 3.1. |
| Output format especificado | ⚠️ parcial | Tier 2.3. |
| Chain-of-thought explícito | ⚠️ implícito | §19 auto-check. Considerar instrucción explícita "antes de responder, validá silenciosamente los 7 puntos". |
| Variables/contexto runtime declarados | ❌ | Tier 1.2. |
| Manejo de fallos de tool | ❌ | Tier 2.2. |
| Limites de longitud de respuesta | ❌ | Tier 2.3. |
| Idioma default + fallback | ⚠️ implícito | Tier 2.4. |
| Frases canónicas centralizadas (DRY) | ❌ | Tier 2.1. |
| Self-check rubric | ✅ | §19 está bien. |
| Ejemplo negativo explícito | ✅ | §18 bien. |
| Refresh date / time-aware | ❌ | Tier 1.2. |

---

## Plan de aplicación sugerido

**Sprint 1 (mañana):** Tier 1 completo. Dejar v3.1 en Drive. Smoke test con conversación end-to-end.
**Sprint 2 (semana):** Tier 2.1 + 2.2 + 2.3. Dejar v3.2.
**Sprint 3 (mes):** Tier 2.4 + Tier 3.3 (eval). Dejar v3.3.
**Sprint 4 (cuando aplique):** Tier 3.1 (XML) si métricas lo justifican.

Cada sprint:
1. Editar archivo Drive con cambio.
2. Anotar en CHANGELOG (sección oculta del prompt o doc separado).
3. Restart nanoclaw-host (recarga prompt en boot).
4. Smoke test: 3 conversaciones representativas (apertura cold, cliente con bandera roja, cliente listo para cerrar).
5. Si se ve drift en alguna, rollback inmediato.

---

## Notas finales

- **No optimizar lo que no se mide.** Antes de tier 3, definir KPIs concretos (ej. % conversaciones que llegan a anticipo, latencia de respuesta, % escalamientos correctos).
- **El prompt actual es bueno.** Las mejoras son marginales. No reescribir desde cero.
- **Versionado en Drive con nombre de archivo:** `VENTAS_System_Prompt_Monica_v3.1.md`, no sobreescribir v3.0.
- **Si el equipo del negocio edita directo el Drive,** dejá un comentario al inicio del archivo que aclare qué secciones son seguras de editar (tono, ejemplos, frases) vs cuáles requieren revisión técnica (estructura, tools, banderas rojas).
