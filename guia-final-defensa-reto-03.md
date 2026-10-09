# Guía final de defensa técnica
## Reto 03 · Agente de órdenes de compra con SAP simulado

**Repositorio:** https://github.com/StraikerG/reto-03-ordenes-compra  
**Aplicación:** https://reto-03-ordenes-compra-opl6.onrender.com  
**Health check:** https://reto-03-ordenes-compra-opl6.onrender.com/api/health  
**Modelo desplegado:** `gemini-3.5-flash-lite`

---

# 1. Explicación inicial: 60 segundos

> Construí un agente conversacional en TypeScript con Bun para automatizar la validación y creación de órdenes de compra en un SAP simulado. Gemini se utiliza para interpretar la solicitud del usuario y decidir qué herramientas pedir, pero no toma decisiones de negocio ni crea órdenes directamente. Los controles RC1–RC10, cálculos, autorizaciones, confirmaciones, idempotencia y auditoría están implementados en TypeScript determinista. La solución incluye frontend de chat, herramientas tipadas con Zod, demo sin LLM para seis casos, SAP simulado, Docker, pruebas automatizadas y despliegue público en Render.

Frase clave:

> El modelo propone acciones; el backend valida, autoriza y ejecuta.

---

# 2. Problema que resuelve

El proceso manual de compras puede permitir registros inconsistentes: proveedor inexistente, aprobador sin autoridad, valor superior al tope, IVA incorrecto, condiciones de pago no permitidas o creación retroactiva de una OC después de la factura.

La solución busca:

- aplicar controles de forma uniforme;
- bloquear errores críticos;
- solicitar confirmación humana ante excepciones;
- conservar evidencia y trazabilidad;
- evitar órdenes duplicadas;
- explicar al usuario por qué una solicitud fue creada, bloqueada o quedó pendiente.

---

# 3. Arquitectura

```text
Usuario / navegador
       │
       ▼
web/index.html
       │ HTTP
       ▼
src/server.ts
       │
       ▼
src/agent.ts
       │
       ├── agent/prompt.md
       ├── src/knowledge/ordenes-compra.md
       │
       ├── src/llm/factory.ts
       │       ├── src/llm/gemini.ts
       │       └── src/llm/anthropic.ts (fallback)
       │
       ├── src/tools/oc.ts
       ├── src/domain/
       └── src/sap/mock.ts

fixtures/  → maestros y solicitudes de prueba
out/       → trazabilidad, control.csv, logs y SAP simulado
```

## Responsabilidades

| Componente | Responsabilidad |
|---|---|
| `agent/prompt.md` | Orienta la conversación y el orden esperado de herramientas. |
| `src/agent.ts` | Ciclo del agente, sesiones, límites, confirmación y respuestas terminales. |
| `src/llm/gemini.ts` | Adaptador REST de Gemini. |
| `src/llm/factory.ts` | Selecciona el proveedor según variables de entorno. |
| `src/tools/oc.ts` | Frontera de herramientas, schemas Zod y ejecución. |
| `src/domain/` | Reglas RC1–RC10 y construcción del payload. |
| `src/sap/mock.ts` | Persistencia simulada, idempotencia y control.csv. |
| `src/server.ts` | API, frontend, rate limiting y health check. |

---

# 4. Flujo de una conversación

1. El usuario escribe: `Procesa el caso sol-006`.
2. El frontend llama a `POST /api/chat`.
3. El servidor recupera o crea la sesión.
4. El agente envía prompt, historial y herramientas a Gemini.
5. Gemini solicita herramientas.
6. El backend valida argumentos con Zod y ejecuta herramientas.
7. Las herramientas leen fixtures, aplican reglas y devuelven resultados estructurados.
8. Si hay una excepción, el servidor conserva el caso pendiente y solicita confirmación humana.
9. Con confirmación válida, se crea la OC.
10. La respuesta final se construye de forma determinista después de la creación para evitar que el LLM repita una confirmación ya consumida.

---

# 5. Por qué las herramientas releen fixtures

## Pregunta probable

**¿Por qué las herramientas releen los fixtures e ignoran valores que podría pasar el modelo?**

## Respuesta recomendada

> Porque el modelo no es una fuente de verdad. Puede equivocarse, resumir mal, interpretar información incompleta o recibir instrucciones maliciosas. Las herramientas toman como fuente de verdad los maestros y documentos originales: proveedor, aprobación, solicitud, cotización, factura y centros de costo. De esa forma, aunque el modelo solicite una operación, el backend valida con datos confiables y reproducibles.

## Beneficios

- evita alucinaciones;
- evita que el modelo altere importes, NIT, aprobadores o IVA;
- permite auditoría reproducible;
- protege frente a prompt injection;
- permite ejecutar `demo.ts` sin modelo de lenguaje.

---

# 6. Herramientas del agente

| Herramienta | Función |
|---|---|
| `oc_leer_paquete` | Lee solicitud, correo, cotización, aprobación y factura. |
| `oc_validar` | Ejecuta RC1–RC10 y calcula bloqueos, excepciones y derivados. |
| `oc_construir_payload` | Construye el payload de orden si el caso lo permite. |
| `oc_generar_evidencia` | Genera trazabilidad y evidencia de aprobación. |
| `oc_crear` | Crea la OC, devuelve idempotencia, bloqueo o confirmación requerida. |

Frase clave:

> Gemini no puede crear una OC directamente. Solo puede solicitar `oc_crear`; la herramienta decide si la operación procede.

---

# 7. Confirmación humana aplicada por servidor

## Pregunta probable

**¿Cómo aseguras que la confirmación se cumpla en el servidor y no solo en el prompt?**

## Respuesta recomendada

> Cuando una herramienta detecta una excepción, por ejemplo diferencia de cotización o compra retroactiva, el servidor guarda el caso en `sesion.pendientes`. En el turno siguiente se toma una copia de los casos confirmables y `oc_crear` consulta `puedeConfirmar(caso)`. Por eso el modelo no puede inventar `confirmado=true` para un caso que no estaba pendiente. La confirmación se consume después de crear o procesar el caso.

## Flujo

```text
oc_validar / oc_crear
       │
       ▼
REQUIERE_CONFIRMACION
       │
       ▼
sesion.pendientes.add(caso)
       │
       ▼
usuario confirma en turno posterior
       │
       ▼
puedeConfirmar(caso) === true
       │
       ▼
oc_crear permitido
```

---

# 8. Controles RC1–RC10

| Control | Regla | Resultado |
|---|---|---|
| RC1 | Proveedor existe, coincide y está activo. | Bloqueo |
| RC2 | Aprobador autorizado para el centro de costo. | Bloqueo |
| RC3 | Valor dentro del tope del aprobador. | Bloqueo |
| RC4 | Subárea permitida para el centro. | Bloqueo |
| RC5 | Diferencia de cotización superior a tolerancia o cotización ausente. | Confirmación |
| RC6 | IVA válido; si falta, deriva default del proveedor. | Bloqueo / confirmación |
| RC7 | Condición de pago válida; si falta, deriva default. | Bloqueo / derivación |
| RC8 | Factura anterior a solicitud: compra retroactiva. | Confirmación |
| RC9 | Aprobación anterior a solicitud. | Confirmación |
| RC10 | Cantidad × precio coincide con total, con tolerancia. | Bloqueo |

---

# 9. RC2 y RC3: caso sol-003

## Pregunta probable

**¿Por qué RC2 y RC3 se revisan juntas en `sol-003`?**

## Respuesta recomendada

> RC2 verifica si la persona que aprobó tiene autoridad para el centro de costo. RC3 verifica si el monto de la compra está dentro del tope autorizado. Una aprobación solo es válida si se cumplen ambas condiciones: la persona correcta y el valor permitido. En `sol-003`, el aprobador no está autorizado para `CC-2020`; además, el valor de COP 74.000.000 supera el tope aplicable de COP 30.000.000. Por eso la OC queda bloqueada y no se pide confirmación.

---

# 10. Casos de prueba que debes conocer

| Caso | Resultado | Qué demuestra |
|---|---|---|
| `sol-001` | OC creada e idempotencia | Flujo exitoso sin excepción. |
| `sol-002` | Bloqueada RC1 | Proveedor inexistente. |
| `sol-003` | Bloqueada RC2/RC3 | Autoridad y tope de aprobación. |
| `sol-004` | Confirmación RC5 | Diferencia entre cotización y solicitud. |
| `sol-005` | Confirmación RC8 | Compra retroactiva. |
| `sol-006` | Deriva IVA/pago y confirma RC6 | Defaults de maestro más confirmación. |

## Dos demos públicas recomendadas

### Caso exitoso con excepción

```text
Procesa el caso sol-006
```

Después:

```text
Sí, confirmo. Crea la OC con esas excepciones.
```

Demuestra derivación de IVA, condiciones de pago, confirmación y creación.

### Caso bloqueado

```text
Procesa el caso sol-003
```

Demuestra que el sistema bloquea y no crea una OC.

---

# 11. Idempotencia y auditoría

## Pregunta probable

**¿Cómo evitas crear dos OCs para la misma solicitud?**

## Respuesta recomendada

> El SAP simulado busca la solicitud antes de crear una nueva orden. La clave funcional es `solicitud_id`. Si ya existe una OC, devuelve un resultado idempotente en lugar de crear otra. Esto es importante porque los usuarios pueden reintentar por errores de red o porque un LLM puede repetir una solicitud.

Archivos generados:

```text
out/control.csv
out/log.jsonl
out/sap/ordenes.jsonl
out/<caso>/trazabilidad.json
out/<caso>/aprobacion.txt
```

---

# 12. Corrección de múltiples llamadas de herramientas

## Qué problema se detectó

Al terminar el turno inmediatamente después de la primera OC creada, una segunda llamada de herramienta de la misma respuesta de Gemini podía quedar sin resultado en el historial.

Esto era riesgoso porque una llamada de herramienta sin `functionResponse` puede dejar la sesión inconsistente para el proveedor LLM.

## Cómo se corrigió

- Se acumulan las OCs creadas en `creadas[]`.
- Se ejecutan todas las llamadas del lote actual.
- Se agrega un mensaje `tool` para cada `toolCallId`.
- Solo después se retorna la respuesta terminal con `resumenCreadas(creadas)`.
- Se agregó una prueba automatizada que simula dos llamadas `oc_crear` en una misma respuesta y verifica que ambas tengan respuesta en el historial.

Frase clave:

> El agente ahora completa todas las respuestas de herramientas antes de cerrar un turno exitoso.

---

# 13. Gemini y function calling

La integración se implementó mediante REST nativo y `fetch`.

## Detalles técnicos

- `GEMINI_API_KEY` solo se usa en backend y Render.
- `LLM_PROVIDER=gemini` selecciona Gemini.
- `GEMINI_MODEL=gemini-3.5-flash-lite` selecciona el modelo desplegado.
- Las herramientas se envían como `functionDeclarations`.
- Los resultados se devuelven como `functionResponse`.
- El adaptador elimina campos de JSON Schema no aceptados por Gemini, como `additionalProperties`.
- El adaptador conserva `thoughtSignature` cuando Gemini la entrega en una llamada de herramienta.
- Si Gemini no devuelve texto ni tool calls, se genera un error legible en lugar de una burbuja vacía.

## Pregunta probable

**¿Qué es `thoughtSignature`?**

> Es información que Gemini puede asociar a una llamada de función. Debe conservarse al reenviar el historial después de ejecutar la herramienta. Si se pierde, Gemini puede rechazar el siguiente ciclo de herramientas o degradar la conversación.

---

# 14. Costos y tokens

## Medición estimada

- System prompt: 5.277 caracteres, aproximadamente 1.319 tokens.
- Declaraciones de herramientas: 2.857 caracteres, aproximadamente 714 tokens.
- Modelo: `gemini-3.5-flash-lite`.
- La tabla es una estimación reproducible basada en prompt, herramientas, fixtures, historial y cantidad de llamadas.

| Caso | Llamadas | Entrada | Salida | Costo estimado en nivel pago |
|---|---:|---:|---:|---:|
| sol-001 | 5 | 13.119 | 68 | USD 0,0041 |
| sol-002 | 4 | 10.005 | 313 | USD 0,0038 |
| sol-003 | 4 | 10.130 | 313 | USD 0,0038 |
| sol-004 | 7 | 20.126 | 359 | USD 0,0069 |
| sol-005 | 7 | 20.210 | 359 | USD 0,0070 |
| sol-006 | 7 | 19.818 | 359 | USD 0,0068 |
| **Total, sesiones separadas** | **34** | **93.408** | **1.771** | **USD 0,0324** |
| **Promedio por caso** | **5,7** | **15.568** | **295** | **USD 0,0054** |

`demo.ts` no llama al modelo, por tanto su costo de IA es USD 0.

## Impacto del historial

Los seis casos dentro de una sola sesión consumieron 204.488 tokens de entrada y 1.771 de salida, con costo estimado de USD 0,0658. El historial acumulado incrementó los tokens de entrada aproximadamente 2,19 veces.

Para producción, propondría compactar historial, resumir casos cerrados y mantener límites por sesión.

---

# 15. SAP real en producción

## Pregunta probable

**¿Qué harías para integrar SAP real? ¿OData o BAPI?**

## Respuesta recomendada

> Mantendría la interfaz del adaptador SAP y reemplazaría `SapMock` por una implementación real. Si el cliente usa SAP S/4HANA y tiene servicios disponibles, priorizaría OData por ser HTTP, estándar y más simple de observar. Si está en ECC o requiere funciones no expuestas por OData, usaría BAPI/RFC mediante un middleware corporativo. La decisión final depende de versión de SAP, servicios expuestos y gobierno de integración.

## Diseño propuesto

| Tema | Producción |
|---|---|
| S/4HANA | API OData de Purchase Orders detrás de SAP API Management o Integration Suite. |
| ECC | BAPI/RFC mediante middleware corporativo. |
| Credenciales | OAuth 2.0, mTLS o cuenta técnica de mínimo privilegio. |
| Secretos | Vault corporativo; nunca `.env` versionado. |
| Idempotencia | Buscar por `solicitud_id` o referencia externa antes de crear. |
| Timeout | Consultar SAP antes de reintentar una creación incierta. |
| Reintentos | Solo errores transitorios; no reintentar errores funcionales. |
| Observabilidad | Logs estructurados, métricas, alertas y correlación de solicitud. |
| Plan B | Cola de reintentos y revisión humana si SAP está caído. |

Frase clave:

> El modelo nunca se conectaría directamente a SAP. Solo el backend y el adaptador autorizado ejecutarían la integración.

---

# 16. Órdenes retroactivas

## Pregunta probable

**¿Qué significa una OC retroactiva y por qué es un riesgo?**

## Respuesta recomendada

> Una OC retroactiva ocurre cuando la factura es anterior a la solicitud o la orden. El gasto ya se realizó, por lo que la OC no previene el compromiso: lo regulariza después. Se pierde control preventivo de presupuesto, aprobación, cotización y negociación. Por eso RC8 no deja pasar el caso silenciosamente: lo marca como retroactivo, exige confirmación y registra la excepción para control interno.

---

# 17. Límites: dilo antes de que te pregunten

> Prefiero declarar los límites de la versión de reto antes de que se interpreten como omisiones.

| Límite | Explicación | Mejora futura |
|---|---|---|
| SAP mock | La persistencia es JSONL local y funciona en una sola réplica. | Base de datos, adaptador SAP real y almacenamiento compartido. |
| Múltiples casos secuenciales | Si el modelo procesa varios casos en iteraciones diferentes y crea una OC en la primera iteración exitosa, el turno se cierra con ese resumen. | Planificador por lote o cola por caso. |
| Evidencia PDF | No se implementó generación de PDF de evidencia. | Generador PDF firmado o integrado con gestión documental. |
| Bonus módulo/ | No se implementó el bonus adicional solicitado. | Módulo separado según prioridad de negocio. |
| Render Free | El filesystem `out/` puede reiniciarse y el servicio puede dormir. | Storage persistente y plan productivo. |
| Free Tier Gemini | La cuota es limitada por RPM/RPD. | Facturación, alertas y fallback por proveedor. |

## Cómo explicar el límite de múltiples casos

> Corregí el caso de múltiples tool calls dentro de una misma respuesta: todas se ejecutan y reciben resultado. El límite restante es distinto: si el modelo planifica varios casos en iteraciones secuenciales y crea una OC en el primer lote exitoso, el turno termina. Para producción lo resolvería con una cola por caso o un modo batch explícito.

---

# 18. Uso responsable de IA

Ajusta esta sección a tu realidad antes de la defensa.

- **ChatGPT:** apoyo para arquitectura, depuración, comandos de Bun/Git/Docker/Render, borradores de documentación y preparación de defensa.
- **Claude:** revisión técnica adicional, identificación de casos de borde y propuesta de pruebas de regresión.
- **Gemini 3.5 Flash-Lite:** proveedor de runtime para conversación y orquestación de herramientas.

## Qué se descartó deliberadamente

- LLM para validar RC1–RC10.
- LLM para calcular valores, IVA, topes o condiciones de pago.
- LLM para autorizar aprobadores.
- LLM para crear directamente una OC/SAP.
- LLM para ejecutar `demo.ts`.
- LLM como único extractor confiable de cotizaciones.

Frase clave:

> La IA mejora interacción y orquestación; TypeScript determinista conserva el control financiero y operativo.

---

# 19. Guion de demo recomendado

1. Abrir la URL pública y `/api/health`.
2. Mostrar GitHub y el historial de commits.
3. En chat, procesar `sol-006`.
4. Mostrar derivación de IVA/pago y confirmación humana.
5. Confirmar y mostrar la OC creada.
6. En otra sesión, procesar `sol-003`.
7. Mostrar bloqueo y explicar RC2/RC3.
8. En terminal, ejecutar `bun run demo`.
9. Mostrar `out/control.csv` y `out/sap/ordenes.jsonl` localmente.
10. Cerrar con SAP productivo, idempotencia y retroactivas.

---

# 20. Preguntas rápidas y respuestas

### ¿Por qué no usar solo un formulario?

El chat permite explicar controles, solicitar confirmaciones y procesar lenguaje natural. Aun así, los controles críticos no dependen del chat: se aplican en herramientas deterministas.

### ¿Qué pasa si Gemini falla o llega al límite?

El backend devuelve un mensaje legible, conserva la sesión y no crea acciones incompletas. La demo determinista sigue disponible sin proveedor LLM.

### ¿Qué pasa si el modelo inventa un proveedor o un valor?

No se usa como fuente de verdad. Las herramientas releen fixtures y validan maestros.

### ¿Cómo proteges la API key?

La key solo vive en variables de entorno locales/Render. No se entrega al frontend, no se sube a Git y no se registra en logs.

### ¿Cómo previenes una orden duplicada?

Con idempotencia por `solicitud_id` antes de crear una nueva OC.

### ¿Cuál fue el mayor reto técnico?

Separar conversación y control. El LLM debe poder ayudar sin adquirir autoridad sobre reglas financieras, confirmaciones ni creación de órdenes.

---

# 21. Checklist antes de defender

- [ ] Abrir Render 5 minutos antes para despertar el servicio.
- [ ] Verificar `/api/health`.
- [ ] Tener una API key/cupo disponible.
- [ ] No mostrar `.env`, claves ni facturación.
- [ ] Verificar `git status` limpio.
- [ ] Tener abierto `SOLUCION.md`.
- [ ] Tener terminal en raíz del proyecto.
- [ ] Ejecutar una vez: `bun run demo`.
- [ ] Tener presentes `sol-006` y `sol-003`.
- [ ] Recordar: Gemini conversa; TypeScript controla.
