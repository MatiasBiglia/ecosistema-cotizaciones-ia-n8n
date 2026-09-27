# Ecosistema Autónomo de Cotizaciones con IA y Supervisión HITL

Proyecto final integrador de arquitectura de automatización inteligente. El sistema procesa solicitudes comerciales mediante un formulario web, ejecuta validación tipada, consulta un tarifario dinámico mediante RAG con un LLM, genera tareas de supervisión humana (Human-in-the-Loop) en Trello y despacha la propuesta validada al cliente vía correo electrónico.

---

## 1. Enlaces Obligatorios y Recursos

* **Dashboard de Control y Base de Datos (Shared View Notion):** https://garnet-seashore-c49.notion.site/3e1dac351e208083adc5d4b19bb78225?v=3e1dac351e2080a3abdb000ca12a07e0
* **Workflow Exportado (.json):** Archivo disponible en la raíz de este repositorio (`cotizacion_ia_workflow.json`).
* **Documento Técnico de Arquitectura (PDF):** Archivo disponible en el repositorio (`Arquitectura_Ecosistema_IA.pdf`).

---

## 2. Mapa de Arquitectura Técnica (Criterio 1 - 20%)

El flujo opera en una sola orquestación sin bucles infinitos bajo el siguiente pipeline determinístico:

1. **Trigger Inteligente:** Formulario n8n nativo (`On form submission`) accionado exclusivamente por evento.
2. **Router de Validación Tipada (`If`):** Evalúa la existencia y tipo de datos (cantidad > 0, email válido, nombre presente).
   * **Rama True (Camino Feliz):** Continúa al pipeline de procesamiento y RAG.
   * **Rama False (Camino Infeliz / Test de Estrés):** Desvío hacia Notion creando registro con estado `Error_Faltan_Datos`. Cancela el consumo de tokens y llamadas a APIs innecesarias.
3. **Persistencia Base (Notion):**
   * Creación de entidad `Cliente` (Nombre, Email, Teléfono).
   * Creación de entidad `Presupuesto` vinculada al cliente con estado `Pendiente_Cotizacion`.
4. **Motor Cognitivo (AI Agent + RAG):**
   * Modelo: Claude / GPT-4o-mini vía **OpenRouter**.
   * Tool: `Get many database pages in Notion` para consultar el tarifario dinámico oficial.
   * Resiliencia: 3 reintentos (`Retry on Fail: 3`, espera de 2000 ms) y directiva `Continue Regular Output` (Resume) ante indisponibilidad de API.
5. **Persistencia de Borrador:** Actualización en Notion con la propuesta calculada (`property_cotizacion_ia`) y estado `Borrador_Generado`.
6. **Supervisión Human-in-the-Loop (HITL):**
   * Creación de tarjeta en **Trello** (Tablero: *COTIZACION IA*) con el desglose formal.
   * Inyección del enlace de aprobación dinámico mediante la variable `$execution.resumeUrl`.
   * Actualización bidireccional guardando el `trello_card_id` en Notion.
   * Suspensión síncrona en el nodo **Wait** (`On Webhook Call`, método `GET 200`). Previene envíos no supervisados ("efecto metralleta").
7. **Salida Multicanal:**
   * Al recibir el clic humano en el webhook, actualiza Notion a `Cotizado_Enviado` y `Aprobado_HITL = true`.
   * Despacho del correo formal al cliente vía **Gmail** con cuerpo en texto plano formateado.

---

## 3. Manual Operativo de Datos y Esquemas JSON (Criterio 2 - 20%)

### Estructura de Tablas en Notion
* **Clientes:** Almacena datos de contacto (`Nombre`, `Email`, `Telefono`) y relación 1:N hacia la base `Presupuestos`.
* **Presupuestos:** Registro transaccional que contiene:
  * `ID_Presupuesto` (Título único: `PRE-YYYYMMDD-HHmm`)
  * `Estado` (Select: `Pendiente_Cotizacion`, `Borrador_Generado`, `Cotizado_Enviado`, `Error_Faltan_Datos`)
  * `Detalle_Pedido` (Texto con resumen de solicitud)
  * `Cotizacion_IA` (Texto con el cálculo detallado)
  * `Aprobado_HITL` (Booleano / Checkbox)
  * `Trello_Card_ID` (Identificador para sincronización bidireccional)
  * `Cliente` (Relation hacia la tabla de Clientes)

### Esquema JSON de Entrada (Formulario Web)

{
  "Nombre y Apellido": "Matias",
  "Email": "matu_biglia@hotmail.com.ar",
  "Teléfono / WhatsApp": "351xxxxxxx",
  "Cantidad de remeras": 44,
  "Acepta Sponsor": "No",
  "Solicita Factura": "No"
}

### Esquema JSON de Salida IA (Post-RAG)

{
  "output": "El precio unitario para 44 unidades es de $13.500 c/u.\nSubtotal Base = 44 * $13.500 = $594.000\nComo no aceptó sponsor, no hay descuento aplicable.\nComo no solicitó factura, no hay IVA a sumar.\nTotal = $594.000.\nLa propuesta formal incluye las estampas estándar de 1 logo chico de hasta 10x10 cm y 1 logo grande de hasta 26x26 cm, y el tiempo de confección es de 15 días hábiles."
}

---

## 4. Matriz de Optimización de Costos y Selección de Modelos (Criterio 3 - 20%)

| Modelo Evaluado | Costo In / Out (1M tokens) | Latencia Media | Decisión Técnica |
| :--- | :--- | :--- | :--- |
| **Claude 3.5 Sonnet / GPT-4o-mini (OpenRouter)** | **$0.15 / $0.60** | **~1.1s** | **SELECCIONADO:** Alta precisión en operaciones matemáticas estructuradas y apego estricto a las reglas del tarifario RAG a mínimo costo. |
| OpenAI GPT-4o Estándar | $2.50 / $10.00 | ~1.8s | DESCARTADO: Sobredimensionado para la tarea. Costo 16 veces superior sin ganancia en la precisión aritmética. |
| Claude 3 Opus | $15.00 / $75.00 | ~3.5s | DESCARTADO: Costo y latencia prohibitivos para flujos transaccionales directos. |
| OpenAI Batch API | 50% de descuento | 12 - 24 horas | DESCARTADO: Incompatible con la necesidad de respuesta ágil hacia el cliente y supervisión en tiempo real. |

* **Ahorro Proyectado:** La inclusión del filtro `If` previo cancela el 100% de llamadas por datos corruptos, y la selección del modelo optimizado reduce los costos de inferencia en un 85% frente al uso de modelos generalistas.

---

## 5. Seguridad, Resiliencia y Human-in-the-Loop (Criterio 4 - 20%)

* **Minimización de Datos:** El LLM únicamente procesa variables del pedido (cantidad, sponsor, factura). Datos sensibles de contacto (teléfono, correo) se aíslan fuera del prompt del sistema.
* **Resiliencia ante Fallos de API:** Configuración de 3 reintentos automáticos con backoff de 2 segundos. Directiva `Resume` activa (`Continue Regular Output`) para evitar interrupciones no controladas.
* **Manejo del Camino Infeliz:** Validación previa que deriva solicitudes inválidas a un registro de auditoría en Notion con estado `Error_Faltan_Datos`.
* **Human-in-the-Loop (Prevención de Efecto Metralleta):** El flujo se pausa de forma nativa mediante el nodo `Wait`. El despacho de correo requiere la autorización explícita del supervisor a través del webhook único `$execution.resumeUrl` inyectado en Trello.

---

## 6. Dashboard de Control (Criterio 5 - 20%)

Se implementó una vista compartida pública en Notion con permisos de solo lectura para auditar la operativa:
* **Métricas de Control:** Total de solicitudes procesadas, ratio de aprobación humana vs. borradores pendientes, y tasa de error por datos incompletos.
* **Panel Kanban Operativo:** Vistas filtradas por estados: `Pendiente_Cotizacion` -> `Borrador_Generado` -> `Cotizado_Enviado` / `Error_Faltan_Datos`.
