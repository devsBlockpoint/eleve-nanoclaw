# Spec — Fuente única de prompt con proyecciones por canal

**Fecha:** 2026-07-30 · **Estado:** diseño aprobado en conversación, pendiente de implementar
**Problema que resuelve:** hoy hay **dos prompts que se desincronizan**. El maestro saltó a v4.0
(terminología "apartado", el cobro **antes** de las fechas, usted por defecto, política cerrada) y
la voz quedó en v3.5 hasta que se detectó a mano. Además, reglas de texto se filtraron a voz
(Mónica dictó una CLABE y dijo *"por aquí en el chat"* en una llamada).

---

## 1. Principio

**Un documento maestro. Dos proyecciones. Cero edición manual duplicada.**

El negocio edita **un** Google Doc. Un build genera la versión de texto y la de voz, cada una con
sus reglas de canal. Nadie mantiene dos prompts.

---

## 2. Las tres categorías de contenido

La clave del diseño: no todo es "igual" o "distinto". Hay una tercera categoría.

### ① Canon puro — idéntico en ambos canales
Identidad · doctrina comercial · guardrails duros (seguridad clínica, sin promesas, precios siempre
por tool, política comercial cerrada) · trato (usted, español de Sinaloa) · flujo comercial (Lectura
→ **apartado antes de fechas** → agendar) · banderas rojas y **qué** escala · léxico.

### ② Misma política, distinta ejecución ⭐
El canon fija el **QUÉ**; el archivo de canal fija el **CÓMO**. Es la categoría que previene las
regresiones ya vividas.

| Política (canon) | Ejecución TEXTO | Ejecución VOZ |
|---|---|---|
| Los datos del apartado se entregan íntegros en el turno del cierre, nunca se difieren | Se escriben completos en el mensaje | **No se dictan**: monto + mecanismo, y se envían por WhatsApp |
| Se confirma con la paciente antes de agendar | Resumen escrito + "¿lo dejo agendado?" | **Confirmación hablada** explícita |
| Dar espacio para decidir | **Silencio estratégico** (no escribir) | ⚠️ **Inverso**: el silencio es *dead air* → pausa breve y seguir |
| Escalar a humano | "Una asesora entra **a este chat**" | **Transferencia en caliente o callback** — nunca "a este chat" |
| Una idea por turno | Puede mandar 2-3 mensajes cortos seguidos | **Un turno = una idea + una pregunta** |
| Entregar la ubicación | Dirección + link de mapa en el mensaje | Dirección **hablada**; el mapa **se envía** |

### ③ Solo-canal
- **Texto:** sintaxis WhatsApp (`*negrita*`, bullets `•`, URLs planas), máx. 3-4 líneas, emojis,
  **fotos entrantes** (señal de alta intención), notas de voz entrantes, sustitución de placeholders,
  sanitización de tags XML/JSON.
- **Voz:** prosodia (cifras en palabras), **barge-in**, pronunciación *"e-le-vé"*, `caller_id` como
  contexto, nunca dictar URLs.

---

## 3. Fuentes y por qué están donde están

| Contenido | Vive en | Por qué |
|---|---|---|
| **Canon + Conocimiento** | **Google Doc maestro** | Lo edita el negocio, cambia seguido (ya vamos en v4.0) |
| **Reglas de canal** (①②③ del lado canal) | **Repo** (`docs/voz/canal-*.md`) | Son técnicas, casi no cambian, necesitan versionado y revisión de código |

### Marcas en el Doc maestro
El negocio agrega marcas de sección para que el script corte solo:

```
[[CANON]]
§0 Directivas · §1 Identidad · §2 Doctrina · §4 Trato · §6.1 Apartado · §9 Banderas rojas · §17 Léxico…

[[CONOCIMIENTO]]
§5 Catálogo · §5.6 Fichas · §6.3-6.6 Precios · §7 Personas · §10.bis SPIN · §12 Objeciones · §15 Operación
```

**Degradación segura:** si el doc **no** trae marcas, el build trata **todo como CANON** — que es
exactamente el comportamiento de hoy para texto. Nada se rompe mientras el negocio agrega las marcas.

---

## 4. Arquitectura del build

**Dónde corre: `mcp-monica`.** Ya es un servicio permanente, público, con acceso a Supabase y a las
credenciales de ElevenLabs. No hace falta infraestructura nueva.

```
        Google Doc maestro  (negocio)
                 │  export?format=txt
                 ▼
        ┌─────────────────────────────┐
        │   mcp-monica — proyector    │
        │   canon + conocimiento      │
        │   + canal-texto.md          │
        │   + canal-voz.md            │
        └───────┬─────────────┬───────┘
        GET /prompt/texto     │ push programado
                │             │
                ▼             ▼
          nanoclaw       ElevenLabs
      (recarga c/1h)   (prompt + 3 KBs)
```

### 4.1 Lado texto — **pull, ya funciona**
`AGENT_SYSTEM_PROMPT_URL` deja de apuntar al Google Doc crudo y apunta a
**`https://<mcp-monica>/prompt/texto`**. nanoclaw ya recarga cada 1h y —clave— el loader
(`system-prompt-loader.ts:36-48`) **cachea en disco y hace fallback al cache** si el fetch falla.
Red de seguridad gratis: si el proyector se cae, nanoclaw sigue con el último prompt bueno.

### 4.2 Lado voz — **push programado**
ElevenLabs no hace polling. `mcp-monica` corre un job (misma cadencia, 1h):
1. Baja el maestro, calcula la proyección de voz y los docs de conocimiento.
2. Compara **hash** contra lo último publicado. Si no cambió → no hace nada.
3. Si cambió → `PATCH` del prompt del agente + actualiza los 3 docs de KB + dispara el **RAG index**.

> Nota: el push a voz reemplaza el trabajo manual que hice esta sesión (cortar sesiones, subir KBs,
> reindexar). Es el mismo procedimiento, automatizado.

---

## 5. Validación antes de publicar (obligatoria)

El maestro lo edita el negocio y se propaga solo → una edición rota llegaría a producción. **El
proyector no publica nada que no pase estos chequeos:**

- El documento descargado **no está vacío** y supera un tamaño mínimo razonable.
- Existen las **secciones de canon obligatorias** (identidad, guardrails, flujo de apartado).
- La proyección de **voz no contiene datos bancarios** (CLABE/tarjeta) — chequeo explícito, porque
  ese fue un bug real.
- La proyección de voz **no contiene frases de otro canal** ("por aquí en el chat", "escríbame").
- El prompt de voz **no excede** el presupuesto de tamaño razonable para ElevenLabs.

**Si algo falla:** no publica, conserva la versión anterior y registra el motivo. Nunca degrada a un
prompt inválido.

---

## 6. Qué se construye

| # | Pieza | Dónde |
|---|---|---|
| 1 | `canal-texto.md` y `canal-voz.md` (reglas ②③ por canal) | `docs/voz/` del monorepo |
| 2 | Proyector: parseo de marcas + composición + validación | `mcp-monica/src/prompt-projection.ts` (TDD) |
| 3 | Endpoint `GET /prompt/texto` | `mcp-monica/src/index.ts` |
| 4 | Job de push a ElevenLabs (hash + PATCH + KB + reindex) | `mcp-monica/src/eleven-sync.ts` |
| 5 | Marcas `[[CANON]]` / `[[CONOCIMIENTO]]` en el Doc | **negocio** |
| 6 | Cambiar `AGENT_SYSTEM_PROMPT_URL` en Easypanel | deploy |

---

## 7. Orden y seguridad del rollout

1. **Piezas 1-3** (proyector + endpoint de texto) sin cambiar nada en producción.
2. **Verificar** el endpoint contra el prompt que nanoclaw usa hoy: la proyección de texto debe ser
   equivalente. Recién ahí se cambia la URL.
3. **Pieza 4** (push a voz) — arranca en **dry-run**: calcula y reporta el diff, **sin publicar**.
   Se habilita el push cuando el diff se vea correcto.
4. Marcas en el Doc: se pueden agregar **después**, gracias a la degradación segura del §3.

## 8. Lo que este spec NO resuelve
- El **expediente por persona** (continuidad cross-canal) — es el otro spec, complementario.
- La consolidación total en un solo cerebro (Opción A) — bloqueada por la latencia del spawn de
  containers de nanoclaw; queda para evaluar con medición.

