---
type: roadmap
updated_at: 2026-06-12
---

# Nero — Roadmap

> What we intend to build and why.

Una app móvil que automatiza la **ingesta de facturas de compra**: hoy el operario las digita a
mano en el POS, lo que consume **~1 h/día (~14% de un turno de 7 h) · ~27 h/mes** por la operación.
El objetivo es liberar ese tiempo repetitivo para tareas de mayor valor.

## Contexto — volumen de ingesta (2026, ene–11 jun)

Línea base de `fact_compras`: 990 facturas de compra, 5 cajer@s activos.

| Cajera | Facturas | Prom. día | Prom. mes | Productos/factura |
|---|---:|---:|---:|---:|
| maritza pascumal | 523 | 3,96 | 87,2 | 5,0 |
| geraldin | 206 | 3,68 | 68,7 | 5,5 |
| alejandra ** | 193 | 4,02 | 96,5 | 5,2 |
| alejandra bermudez | 62 | 3,65 | 62,0 | 5,4 |
| luis | 6 | 3,00 | 6,0 | 2,7 |

- **Global:** ~5,2 productos por factura · ~165 facturas/mes · ~6 facturas/día (equipo).
- Maritza concentra el 53% de la ingesta.
- Promedios día/mes sobre días/meses activos de cada cajera (junio parcial).

## Now

- Definir la **topología de escritura** al SQLite del POS (riesgo abierto — ver knowledge.md).
- Validar la **extracción foto→JSON** con Claude vision sobre facturas reales.
- Obtener el **esquema de Stock/triggers del POS** para confirmar si insertar
  `Document`/`DocumentItem` mueve el stock.

## Next

- **Captura multi-foto**: una factura de varias páginas → un solo documento.
- **Servicio de mapeo**: proveedor↔`Customer` y producto↔catálogo (código exacto → fuzzy → embeddings).
- **Pantalla de revisión/edición** de la factura en la app (corregir proveedor o líneas no mapeadas).
- **Creación de proveedor/producto nuevo** cuando no exista en el POS.
- **Escritura del documento** en el POS para que aparezca en caja.
- **Interfaz para editar el precio** del producto (`Product.Price`/`Cost`/`Markup`) cuando el
  proveedor lo sube o baja.

## Later

- Reportes de costo en tiempo real.
- Optimización automática de la curva de precios (márgenes).

## Implementation Plan

Se agregan a continuación los puntos de implementación escalable, organizados como elementos de trabajo:

- Definir alcance y requisitos: recopilar casos de uso, KPIs, restricciones POS y datos de entrada (fotos).
- Diseñar arquitectura general: microservicios, colas, almacenamiento objetos, BFF/mobile, DBs y límites de escala.
- Especificar modelos de datos y APIs: esquemas JSON para `Document`, `DocumentItem`, `Supplier`, `Product` y contratos REST/gRPC.
- Prototipo OCR/Visión: validar extracción foto→JSON (Claude Vision, Tesseract o AI APIs) con dataset real.
- Servicio de mapeo y matching: proveedor↔Customer y producto↔catálogo (código exacto → fuzzy → embeddings).
- Pipeline de procesamiento: orquestar ingestión multi-foto→merge, normalización, NLU y encolamiento asíncrono.
- Integración POS y persistencia: adaptador para escribir `Document`/`DocumentItem` en el POS, manejo transaccional y efectos en stock.
- App móvil (captura + UI): cliente (React Native/Flutter), captura multi-página offline, cola local y UX de revisión.
- Infra, CI/CD y despliegue: contenedores, IaC (Terraform), pipelines, despliegue escalable (K8s/autoscaling).
- Observabilidad y telemetría: logs correlacionados, tracing distribuido, métricas y alertas para SLA/KPI.
- Seguridad y cumplimiento: autenticación, autorización, cifrado en tránsito/descanso, gestión de PII y retención.
- Pruebas y validación: unit, integración y end-to-end (captura→POS), performance y validación con datos reales.
- Plan de despliegue gradual: canary, feature flags, rollout por tiendas y plan de rollback.
- Runbooks y mantenimiento: playbooks para incidentes, backups, rotación de keys y SLAs.
- Estimación y roadmap por fases: priorizar MVP → mejoras de matching → optimizaciones de ML y escala.
