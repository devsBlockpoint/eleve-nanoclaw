# Plan de plantillas de WhatsApp (HSM) — Mónica voz

> **Para qué:** en una **llamada de voz**, Mónica no puede dictar la CLABE ni una liga de pago →
> los manda por WhatsApp. Como una llamada **no abre la ventana de 24 h** de mensajería libre de
> WhatsApp, esos envíos iniciados por el negocio requieren **plantillas HSM aprobadas por Meta**.
> Este doc las especifica listas para **someter a aprobación**; no las crea.
>
> **Fecha:** 2026-07-28 · **Idioma:** `es_MX` · **Estado:** plan para generar/aprobar después.

---

## 0 · Antes de someter — reglas de Meta a respetar

- **Categorías:** `UTILITY` (transaccional, ligado a una interacción/cita — aprobación rápida, sin límite de marketing) · `MARKETING` (promocional — requiere opt-in y cuenta contra límites) · `AUTHENTICATION` (OTP; no aplica aquí).
- **Variables** posicionales `{{1}}`, `{{2}}`… con **ejemplo obligatorio** al someter.
- **UTILITY no puede sonar promocional** (sin "aprovecha", "promo", "últimos lugares" — coincide con los guardrails de Mónica §17). Si suena a venta, Meta la reclasifica a MARKETING o la rechaza.
- **Botones:** URL (estática o dinámica), quick-reply, o llamada.
- **Grafía oficial ÉLÉVÉ** siempre correcta en el cuerpo.

## ⚠️ 1 · Dependencia crítica — ¿QUIÉN envía la plantilla?

Hoy el canal WhatsApp lo maneja **ÉLÉVÉ (Supabase/n8n)**, y **`mcp-monica` NO tiene una tool para
enviar WhatsApp** (Valentina sí tenía `whatsapp_send_message`; Mónica no). Para que la voz dispare
estos envíos hay que construir **una** de estas vías (decisión de la fase de setup, no de este doc):

- **Opción A — Tool nueva `enviar_plantilla_whatsapp`** en `mcp-monica` → edge function que llama a la
  WhatsApp Cloud API con la plantilla + variables. El agente de ElevenLabs la invoca en el cierre.
- **Opción B — Post-call webhook** de ElevenLabs → ÉLÉVÉ/n8n lee el resultado de la llamada (cita
  agendada) y dispara la plantilla. Menos inmediato (post-llamada), pero no toca `mcp-monica`.

Recomendación: **A** para `datos_pago_anticipo` y `confirmacion_cita` (deben salir en el momento del
cierre), **B** para recordatorios/reactivación (son diferidos y ya pueden vivir en cron de ÉLÉVÉ).

---

## 2 · Plantillas — prioridad P0 (habilitan el cierre por voz)

### 2.1 `datos_pago_anticipo` · UTILITY · P0
**Cuándo:** justo después de `agendar_cita` exitoso en una llamada. Es la que "salva el cobro".
**Variables:** `{{1}}` nombre · `{{2}}` servicio · `{{3}}` día · `{{4}}` hora.
**Botón URL (estático):** "Pagar anticipo" → `https://mpago.la/32zQRS3` (liga fija de $300).

```
Hola {{1}}, aquí ÉLÉVÉ ✨ Le confirmo su cita:
{{2}} · {{3}} a las {{4}}.

Para apartar su lugar pedimos un anticipo de $300 que se acredita íntegro al servicio que decida.
Puede pagarlo como le acomode:

• Transferencia (SPEI):
  Banco: Banamex
  Titular: Ernestina Villaseñor Atwood
  CLABE: 002743089576356962
  Concepto: {{1}} — {{2}}
  Monto: $300

• O con tarjeta en un toque con el botón de abajo.

Estos son los datos de la cuenta receptora y la liga oficial de pago; nunca pedimos su número de
tarjeta ni datos bancarios por chat. Cuando lo haga, me comparte el comprobante por aquí 🙌
```
> Nota: el bloque bancario es el **fallback canónico** (§6.2 del prompt maestro). Si `agendar_cita`
> devuelve `datos_pago` distintos, la vía A puede pasarlos como variables adicionales en vez de fijos.

### 2.2 `confirmacion_cita` · UTILITY · P0
**Cuándo:** tras confirmarse la cita (o tras el anticipo), con dirección + mapa.
**Variables:** `{{1}}` nombre · `{{2}}` día · `{{3}}` hora.
**Botón URL (estático):** "Ver mapa" → `https://maps.app.goo.gl/wqd2yhN17uT4Uv1f8`.

```
{{1}}, le esperamos el {{2}} a las {{3}} en ÉLÉVÉ 📍
Guillermo Prieto 221 sur, entre B. Juárez y Cjón. Guadalupe Victoria, Col. Centro, Los Mochis.
Cualquier ajuste, por aquí estoy 😊
```

---

## 3 · Plantillas — prioridad P1 (operación de citas)

### 3.1 `recordatorio_cita_24h` · UTILITY · P1
**Cuándo:** 24 h antes, **solo si el anticipo está confirmado**. Diferido → vía B (cron ÉLÉVÉ).
**Variables:** `{{1}}` nombre · `{{2}}` día · `{{3}}` hora. **Botón URL:** "Ver mapa" (mismo).
```
{{1}}, le recuerdo su cita mañana {{2}} a las {{3}} en ÉLÉVÉ.
📍 Guillermo Prieto 221 sur, Col. Centro, Los Mochis. Cualquier ajuste, por aquí estoy.
```

### 3.2 `anticipo_pendiente` · UTILITY · P1
**Cuándo:** anticipo no confirmado 12 h después de agendar (un solo mensaje cálido).
**Variables:** `{{1}}` nombre. **Botón URL:** "Pagar anticipo" → liga fija.
```
{{1}}, ¿le ayudo si tuvo algún detalle con la transferencia del anticipo? Su lugar sigue apartado;
puede completar el anticipo de $300 por transferencia o con tarjeta en el botón de abajo.
```

### 3.3 `reagenda_confirmacion` · UTILITY · P1
**Cuándo:** tras `reschedule_appointment`. **Variables:** `{{1}}` nombre · `{{2}}` día · `{{3}}` hora.
```
Listo {{1}}, su cita quedó reagendada para el {{2}} a las {{3}}. Aquí estoy para cualquier cosa.
```

---

## 4 · Plantillas — prioridad P2 (seguimiento y reactivación)

### 4.1 `seguimiento_no_show` · UTILITY · P2
**Cuándo:** día siguiente a un no-show, sin reproche. **Variables:** `{{1}}` nombre.
```
{{1}}, le esperábamos ayer en ÉLÉVÉ. Imagino que algo surgió — si quiere retomar, le ayudo a
coordinar una nueva fecha cuando le acomode.
```

### 4.2 `reactivacion_lead` · MARKETING · P2
**Cuándo:** 24–48 h tras una llamada que no cerró (requiere opt-in). Diferido → vía B.
**Variables:** `{{1}}` nombre · `{{2}}` microcápsula educativa (1 frase, específica al caso).
> ⚠️ MARKETING: requiere consentimiento y cuenta contra límites. Sin lenguaje de urgencia.
```
{{1}}, le comparto algo que suele aclarar dudas en casos como el suyo: {{2}}.
Si quiere darle continuidad, con gusto le paso opciones de horario. Aquí estoy 😊
```

### 4.3 `callback_agendado` · UTILITY · P2 (opcional, si se implementa callback)
**Cuándo:** se pactó devolver la llamada. **Variables:** `{{1}}` nombre · `{{2}}` día · `{{3}}` hora.
```
{{1}}, quedamos en marcarle el {{2}} alrededor de las {{3}} para continuar. Si prefiere otro
momento, dígame y lo ajusto.
```

---

## 5 · Resumen para someter a Meta

| Plantilla | Categoría | Prioridad | Variables | Botón | Envío |
|---|---|---|---|---|---|
| `datos_pago_anticipo` | UTILITY | **P0** | 4 | URL pago | Vía A (en llamada) |
| `confirmacion_cita` | UTILITY | **P0** | 3 | URL mapa | Vía A (en llamada) |
| `recordatorio_cita_24h` | UTILITY | P1 | 3 | URL mapa | Vía B (cron) |
| `anticipo_pendiente` | UTILITY | P1 | 1 | URL pago | Vía B (cron) |
| `reagenda_confirmacion` | UTILITY | P1 | 3 | — | Vía A/B |
| `seguimiento_no_show` | UTILITY | P2 | 1 | — | Vía B (cron) |
| `reactivacion_lead` | MARKETING | P2 | 2 | — | Vía B (cron) |
| `callback_agendado` | UTILITY | P2 | 3 | — | Vía A/B |

**Datos fijos que usan las plantillas** (verificar antes de someter):
- Liga de pago (fija $300): `https://mpago.la/32zQRS3`
- Mapa: `https://maps.app.goo.gl/wqd2yhN17uT4Uv1f8`
- Dirección: Guillermo Prieto 221 sur, Col. Centro, Los Mochis, Sinaloa.
- Teléfono: 668 396 5199.
- Cuenta receptora (fallback): Banamex · Ernestina Villaseñor Atwood · CLABE 002743089576356962.

**Orden sugerido:** someter **P0** primero (habilitan el cierre por voz), luego P1, luego P2. Y en
paralelo, decidir/implementar la **vía A** (tool `enviar_plantilla_whatsapp`) para las P0.
