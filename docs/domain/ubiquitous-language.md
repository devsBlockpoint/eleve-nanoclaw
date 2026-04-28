# Ubiquitous Language — ÉLEVÉ

Glosario de términos del dominio. Cuando el código, las tools del MCP, los docs y las conversaciones del equipo usan estas palabras, deben referirse exactamente a lo definido aquí.

Convenciones:
- **Term** (inglés/técnico) → **Término** (español/dominio): definición.
- Cuando el LLM las usa (en español preferido para ÉLEVÉ), aparece **(LLM)**.

## Personas

- **Paciente / Patient** **(LLM: paciente)**: persona que recibe servicios estéticos en ÉLEVÉ. Identificada principalmente por su número de WhatsApp (`+10 dígitos`) y `id` en tabla `pacientes`. Si un mensaje entra de un WhatsApp no registrado, una tool puede crear el paciente automáticamente.
- **Esteticista / Aesthetician** **(LLM: esteticista)**: empleada que ejecuta los servicios. Tabla `esteticistas`. Tiene estado `Activa | Inactiva`. La asignación a una cita se hace auto: la esteticista activa con menos citas ese día.
- **Operador humano / Human operator**: persona del equipo de ÉLEVÉ que toma el control de una conversación cuando el agente escala. Vive del lado del CRM web.

## Conversación con el agente

- **Mónica**: nombre del agente conversacional. Antes vivía en la edge function `chat-assistant`; ahora vive en nanoclaw + mcp-monica.
- **Conversación / WhatsApp Conversation**: una sesión persistente entre un paciente y Mónica. Tabla `whatsapp_conversations`. Su `id` (UUID) es la **clave de sesión** para nanoclaw.
- **Mensaje / Message**: cada turno de la conversación. Tiene `sender_type` ∈ `{user, agent, human}` y queda persistido del lado de ÉLEVÉ.
- **`agent_mode`**: estado de control de la conversación.
  - `bot`: Mónica responde automáticamente.
  - `human`: un operador humano tomó el control; Mónica no responde aunque entre un mensaje.
  - `escalated`: solicitud activa de escalamiento, pendiente de que un humano la tome.
- **Auto-return-to-bot**: cron job (cada 5 min, `pg_cron`) que devuelve conversaciones de `human`/`escalated` a `bot` después de un período de inactividad. Edge function `auto-return-to-bot`.

## Servicios y agendamiento

- **Servicio / Service** **(LLM: servicio)**: tratamiento ofrecido (ej. limpieza facial). Tabla `servicios`.
- **Tratamiento / Treatment** **(LLM: tratamiento)**: visión más detallada de un servicio (descripción, contraindicaciones, duración, precio). Sirve para `get_treatment_detail`.
- **Cita / Appointment** **(LLM: cita)**: reserva concreta de un servicio para un paciente con una esteticista en una fecha y hora. Tabla `citas`. Estados:
  - `Pendiente de Anticipo`: recién creada, falta el pago del anticipo. Estado por defecto al llamar `agendar_cita`.
  - `Confirmada`: anticipo pagado, cita firme.
  - `Realizada`: el servicio se ejecutó.
  - `Cancelada`: el paciente o el sistema canceló.
  - `Reprogramada`: se movió de fecha/hora.
- **Slot / Slot de horario**: combinación válida de fecha + hora para una cita. La tool `buscar_disponibilidad` devuelve slots con cantidad de esteticistas disponibles.

## Pagos

- **Anticipo / Deposit**: pago previo a la cita que la mueve de `Pendiente de Anticipo` a `Confirmada`.
- **Payment link / Link de pago** **(LLM: link de pago)**: URL única que se manda al paciente por WhatsApp para que pague el anticipo. Generado por `send_payment_link`.

## Promociones y leads

- **Promoción / Promotion** **(LLM: promoción)**: campaña vigente con descuento o paquete. Tabla `promociones`. La tool `get_current_promotions` filtra activas por fecha.
- **Lead**: contacto interesado capturado en una conversación que aún no es paciente confirmado. La tool `capture_lead_from_chat` lo registra para seguimiento del equipo comercial.

## Escalación y control

- **Escalación / Escalation**: cambio de `agent_mode` a `human` o `escalated`. Tabla `escalation_queue`. Razones típicas: queja, tema legal, complejidad médica, sentiment muy negativo, pedido explícito.
- **Prioridad de escalación**: `low | medium | high | urgent`. Default: `medium`.

## Infraestructura conversacional

- **Channel adapter**: pieza dentro de nanoclaw que adapta un canal externo (WhatsApp en este caso, vía ÉLEVÉ) al modelo interno de inbound/outbound. Para esta v1 hay un solo channel adapter custom: `eleve-http`.
- **Agent group / Grupo de agente**: agrupación lógica de configuración de un agente en nanoclaw (CLAUDE.md, MCP, container config). En v1 hay uno solo: `monica`.
- **Sesión nanoclaw**: par `inbound.db` + `outbound.db` ligado a un `whatsapp_conversation_id`. Persistente; sobrevive a restarts del container.

## Edge functions del dominio (lo que el MCP llama)

Documentadas en `mcp-monica/mcp/tools/*.md`. Resumen:

| Edge function | Tool MCP (LLM) | Cuándo |
|---|---|---|
| `book-appointment` | `agendar_cita` | Confirmar cita después de elegir slot |
| `check-availability` | `buscar_disponibilidad` | Ver slots libres en una fecha |
| `cancel-appointment` | `cancel_appointment` | Cancelar cita existente |
| `reschedule-appointment` | `reschedule_appointment` | Mover cita a otra fecha/hora |
| `get-appointments` | `get_appointments` | Listar citas de un paciente |
| `get-services` | `obtener_servicios` | Catálogo de servicios |
| `get-treatment-detail` | `get_treatment_detail` | Detalle de un tratamiento |
| `get-current-promotions` | `get_current_promotions` | Promociones activas |
| `search-patient` | `search_patient` | Buscar paciente por WhatsApp |
| `capture-lead-from-chat` | `capture_lead_from_chat` | Registrar lead nuevo |
| `send-payment-link` | `send_payment_link` | Enviar link de pago al paciente |
| `escalate-to-human` | `escalate_to_human` | Pasar conversación a humano |

## Lo que NO es ubiquitous language

- Detalles de implementación de Supabase, Postgres, Vercel, Bun, Docker.
- Nombres de variables de entorno (esos están en `.env.example`).
- Términos del Claude Agent SDK (`Task tool`, `MCP`, etc.) — esos son lenguaje técnico, no de dominio.
