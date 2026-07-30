# Runbook — Desvío de llamadas: Teléfono Telcel → Mónica voz (ElevenLabs)

> **Objetivo:** que una llamada telefónica normal al número **físico Telcel** de la clínica sea
> atendida por **Mónica en voz** (ElevenLabs Conversational AI), **sin Twilio** y **sin tocar el
> WhatsApp** de Mónica.
>
> **Estrategia:** se conserva la SIM Telcel tal cual. Telcel **desvía** todas las llamadas
> entrantes a un **DID +52 de Telnyx**, que entra por **SIP** al agente de voz de Mónica.
> El desvío es **reversible en segundos** (`##21#`).

**Fecha:** 2026-07-28 · **Estado:** listo para ejecutar · **Fase:** 1 (voz por teléfono)
**Fuentes:** verificadas contra docs oficiales de Telnyx, ElevenLabs y Telcel (ver §9).

---

## 0. Panorama

```
  Paciente marca el número Telcel de la clínica
                    │  (PSTN / celular)
                    ▼
        Telcel  ──  desvío incondicional  **21*<DID>#
                    │
                    ▼
     Telnyx DID +52  ── SIP Connection (FQDN) ──►  sip.rtc.elevenlabs.io
                    │
                    ▼
     ElevenLabs Conv.AI  ── agente "Mónica-voz" (modelo Claude)
                    │
                    ▼
        🔒 mcp-monica (/mcp con token)  →  Supabase  (citas · leads · escalación)
```

**Lo que este runbook NO hace:** no porta el número (la SIM sigue viva con datos/SMS), no toca el
canal de WhatsApp/texto (lo sigue manejando nanoclaw), no cubre la llamada *dentro* de WhatsApp
(eso es Fase 2).

---

## 1. Prerequisitos (tener a mano antes de empezar)

- [ ] **Número Telcel** de la clínica, con acceso físico a la SIM/equipo para marcar códigos GSM.
- [ ] **Plan Telcel con minutos nacionales ilimitados** (ej. "Sin Límite"). ⚠️ La pata desviada
      **consume minutos** del plan; sin ilimitado, cada llamada desviada gasta tiempo aire.
- [ ] **Cuenta Telnyx** (Mission Control Portal) con método de pago y documentos KYC (§2).
- [ ] **Cuenta ElevenLabs** — la **misma de Valentina** por ahora. Acceso al workspace (Agents).
- [ ] **Agente "Mónica-voz" ya creado en ElevenLabs** con su prompt/persona configurados.
      → *Depende del paso de persona de la Fase 1 (reusar prompt de Mónica + venta "intensamente
      relajado" adaptado a ÉLEVÉ). Si aún no existe, se puede crear un agente mínimo para probar el
      cableado y luego pulir la persona.*
- [ ] **`mcp-monica` protegido con token en `/mcp`** antes de adjuntarlo al agente.
      ⚠️ **Bloqueante de seguridad** — ver §8. Para probar SOLO el enrutamiento de voz se puede
      diferir, pero **no** conectes las tools a ElevenLabs hasta que `/mcp` valide credenciales.

---

## 2. Paso A — Comprar el DID +52 en Telnyx

1. Portal Telnyx → **Numbers → Search & Buy Numbers** → país **Mexico (+52)**, tipo **Local**.
2. **Cualquier número geográfico MX sirve** para recibir el desvío — el código de área no afecta
   el enrutamiento. ✅ **No bloquees el proyecto esperando un número de Los Mochis (668)**; su
   disponibilidad en Telnyx es dinámica y solo importaría para el caller-ID de salida (fuera de
   alcance de Fase 1).
3. **KYC requerido** (verificación manual, no instantánea — pedirlo con anticipación):
   - Persona física: nombre + teléfono de contacto + **copia de pasaporte o identificación**.
   - Empresa: nombre de la compañía + teléfono + **acta/certificado de registro de la empresa**.
   - **Dirección** (calle, número, CP, ciudad, país) — no exige que sea dirección en México.
4. Anotá el número comprado en **formato E.164**: `+52##########` → lo llamaremos `<DID>`.

---

## 3. Paso B — Configurar Telnyx (SIP Connection → ElevenLabs)

En el portal Telnyx, **Voice → SIP Trunking → Create SIP Connection**:

1. **Connection Type: FQDN.**
2. **FQDN:** agregar `sip.rtc.elevenlabs.io`
   - Puerto: `5060` si usás **TCP** (recomendado por la guía Telnyx×ElevenLabs), `5061` si **TLS**.
3. Pestaña **Inbound**:
   - **Destination Number Format:** `+E.164`
   - **SIP Transport Protocol:** `TCP` (debe coincidir con lo que configures en ElevenLabs)
   - **SIP Region:** la más cercana (ej. una región de Norteamérica/México)
4. Pestaña **Outbound / Outbound Voice Profile:** adjuntá un Outbound Voice Profile a la connection.
   *Solo es estrictamente necesario para salientes ElevenLabs→Telnyx; no daña dejarlo puesto.*
5. **Asignar el `<DID>` a esta SIP Connection**
   (Numbers → tu número → **Connection/Voice settings** → seleccioná la SIP Connection creada).
   Así, toda llamada entrante al `<DID>` sale como **SIP INVITE hacia `sip.rtc.elevenlabs.io`**.

---

## 4. Paso C — Importar el número en ElevenLabs y ligarlo al agente

En **ElevenLabs → Agents → Phone Numbers → Import a phone number from SIP trunk**:

1. **Label:** `Monica-voz-telcel` (o similar).
2. **Phone Number:** el `<DID>` en **E.164** (`+52##########`) — debe ser **idéntico** al de Telnyx.
3. **Inbound trunk:**
   - **Transport Type:** `TCP` (que coincida con Telnyx; usá `TLS` si configuraste 5061).
   - **Media Encryption:** `Allowed` (o `Required` si vas por TLS).
   - **Digest auth:** **dejar vacío** — los troncales inbound de Telnyx no usan digest; ElevenLabs
     identifica por match del número / IP de origen.
   - **Codecs:** negocian solos a **G711 (µ-law/A-law)** o **G722**; no hay que tocar nada.
4. **Number Configuration → Agent:** asignar el número al agente **"Mónica-voz"**.

> A este punto, si llamaras directo al `<DID>` de Telnyx, ya debería contestar Mónica. El desvío
> (Paso D) hace que el número **público Telcel** llegue a ese `<DID>`.

---

## 5. Paso D — Activar el desvío en Telcel (el corazón del runbook)

Desde el teléfono con la **SIM Telcel**, marcar los códigos GSM (MMI):

| Acción | Código a marcar |
|---|---|
| **Activar desvío incondicional** (todas las llamadas) | `**21*<DID>#`  y presionar llamar |
| **Verificar estado** del desvío | `*#21#` |
| **Desactivar** el desvío (rollback) | `##21#` |

**Notas críticas:**
- ⚠️ **`<DID>` debe ser un número NACIONAL mexicano.** Desviar a un número internacional hace que
  Telcel cobre **larga distancia internacional (LDI)** por minuto. El DID +52 de Telnyx es MX → OK.
- **Formato del número al marcar el código:** probá primero con el número **nacional a 10 dígitos**
  (sin `+52`); si Telcel lo rechaza, probá con `52` adelante. Confirmá con `*#21#` que quedó
  registrado el número correcto.
- **Desvío incondicional = TODAS las llamadas van a Mónica.** Nadie más contesta esa línea por voz
  mientras esté activo. El **WhatsApp y los datos** de la SIM **no se afectan** (viajan por datos).
- El desvío **no tiene renta**, pero la **pata desviada consume minutos** del plan Telcel
  (≈gratis con plan de minutos ilimitados).

---

## 6. Paso E — Llamada de prueba y verificación

1. Desde **otro teléfono cualquiera**, marcá el **número público Telcel** de la clínica.
2. **Debe contestar Mónica-voz.** Confirmá audio en ambos sentidos (los codecs negocian a G711/G722).
3. **Verificá el caller-ID del paciente** (⚠️ **no lo asumas**):
   - En ElevenLabs, revisá la variable de sistema **`system__caller_id`** y/o el payload del
     **conversation-initiation webhook** (headers SIP `From` / `P-Asserted-Identity` / `Diversion`).
   - Al desviar Telcel→Telnyx, el número **original del paciente** puede **sobrevivir, ser
     reemplazado por el número Telcel, o llegar como restringido/anónimo** — es dependiente del
     operador y **no está garantizado**. Hacé la prueba real ANTES de depender del caller-ID para
     identificar al paciente en Supabase.
4. Si el caller-ID original **no** sobrevive: la identificación por número queda como *best-effort*;
   Mónica debe pedir el dato al paciente por voz (ajuste de prompt).

### Checklist de "listo"
- [ ] Llamada al Telcel → contesta Mónica.
- [ ] Audio limpio en ambos sentidos.
- [ ] `system__caller_id` inspeccionado (sé qué llega realmente).
- [ ] (Si tools conectadas) una tool de prueba responde y `/mcp` exige token.
- [ ] Rollback probado: `##21#` desactiva y la línea vuelve a sonar normal.

---

## 7. Costos aproximados (planificación)

| Componente | Costo |
|---|---|
| DID Telnyx MX (renta) | desde **US$1 / mes** |
| Telnyx inbound (por minuto) | **~US$0.002 / min** |
| ElevenLabs Agents | **US$0.08 / min** (burst US$0.16) + tokens LLM aparte (~US$0.01–0.05/min) |
| Pata desviada Telcel | minutos del plan (**≈US$0** con ilimitado) |
| **Total aprox. por minuto de llamada** | **~US$0.10 / min** + ~US$1/mes fijo |

**Concurrencia ElevenLabs** (según plan): Free 4 · Starter 6 · Creator 10 · Pro 20 · Scale 30 ·
Business 40 llamadas simultáneas. Verificá que el plan alcance el pico esperado de la clínica.

---

## 8. Seguridad — bloqueante antes de conectar tools

Hoy **`mcp-monica` no valida credenciales de entrada en `/mcp`** (es seguro solo porque vive en red
privada). ElevenLabs lo llamaría por **internet público**, así que **antes de adjuntar `mcp-monica`
al agente**:

1. Agregar en `/mcp` la validación de un **bearer token** (rechazar toda request sin el header).
2. En ElevenLabs, registrar el MCP server con `transport: STREAMABLE_HTTP` y el token como
   **`secret_token`** (→ header `Authorization`) o `request_headers` (ej. `X-API-Key`), guardado
   como **workspace secret** (nunca en texto plano).
3. **HTTPS** obligatorio + **IP allowlist** de las IPs de egreso de ElevenLabs.
4. **Aprobación de tools:** usar `require_approval_per_tool` — auto-aprobar lecturas
   (`buscar_disponibilidad`, `obtener_servicios`, `get_current_promotions`) y confirmar/verbalizar
   las que mutan (`agendar_cita`, `capture_lead_from_chat`, `escalate_to_human`).

> Este endurecimiento va en el **spec de la Fase 0/1** (cambio de código en `mcp-monica`), no en este
> runbook. Pero **no conectes las tools reales** hasta tenerlo.

---

## 9. Rollback

Desactivar el desvío devuelve la línea a su comportamiento normal **al instante**:

```
##21#      (desactiva el desvío incondicional)
*#21#      (verifica que quedó desactivado)
```

No hay que deshacer nada en Telnyx/ElevenLabs para el rollback de voz: sin desvío, el `<DID>` deja de
recibir tráfico. El `<DID>` y la config pueden quedar en pie para reactivar con `**21*<DID>#`.

---

## 10. Troubleshooting

| Síntoma | Causa probable / acción |
|---|---|
| La llamada al Telcel da tono/buzón, no contesta Mónica | El desvío no quedó activo → `*#21#`. Revisá formato del `<DID>` (10 dígitos vs `52…`). |
| Llamando directo al `<DID>` tampoco contesta | Telnyx: ¿el DID está asignado a la SIP Connection? ¿FQDN = `sip.rtc.elevenlabs.io`, inbound `+E.164`, TCP? ElevenLabs: ¿número importado en E.164 y ligado al agente? |
| Contesta pero sin audio / audio cortado | Desajuste de transport (TCP vs TLS) o media encryption entre Telnyx y ElevenLabs. Igualá ambos lados. |
| `system__caller_id` vacío / muestra el número de la clínica | Telcel no propaga el CLI original en el desvío (ver §6.3). Identificar al paciente por voz. |
| "Ocupado" en horas pico | Concurrencia del plan ElevenLabs superada (§7). Subir plan o habilitar burst. |
| Cobros de LDI en la factura Telcel | Estás desviando a un número no-nacional. El `<DID>` debe ser MX. |

---

## 11. Referencias (verificadas 2026-07-28)

- ElevenLabs — SIP trunking: https://elevenlabs.io/docs/eleven-agents/phone-numbers/sip-trunking
- ElevenLabs — integración Telnyx: https://elevenlabs.io/docs/eleven-agents/phone-numbers/telephony/telnyx
- ElevenLabs — precios Agents: https://elevenlabs.io/pricing/agents
- ElevenLabs — MCP (custom server + auth): https://elevenlabs.io/docs/eleven-agents/customization/tools/mcp
- ElevenLabs — dynamic variables (`system__caller_id`): https://elevenlabs.io/docs/eleven-agents/customization/personalization/dynamic-variables
- Telnyx — requisitos DID México: https://support.telnyx.com/en/articles/5466793-mexico-did-requirements
- Telnyx — números México: https://telnyx.com/phone-numbers/mexico
- Telnyx — SIP Connection inbound/outbound: https://support.telnyx.com/en/articles/4404448-sip-connection-inbound-outbound-settings
- Telcel — llamadas/desvío: https://www.telcel.com/personas/servicios/llamadas-y-mensajeria/llamadas

---

## 12. Dos pendientes que solo se resuelven en vivo

1. **Sobrevivencia del caller-ID original** del paciente a través del desvío Telcel→Telnyx (§6.3) —
   probar en la línea real.
2. **Disponibilidad de un número Sinaloa/668** y tiempo exacto de aprovisionamiento del DID MX en
   Telnyx — confirmar con soporte Telnyx (no bloquea: cualquier número MX geográfico funciona).
</content>
</invoke>
