# Alcance y requisitos — Proyecto Nero

## Propósito
Automatizar la ingesta de facturas de compra mediante captura móvil y procesamiento IA para registrar documentos en el POS, reduciendo tiempo manual de digitación.

## Stakeholders
- Operarias/cajeras (Maritza, Geraldin, etc.)
- Equipo de operaciones / inventario
- Equipo de producto y diseño
- Equipo de backend / POS
- Equipo de ML/Infra

## Objetivos medibles
- Reducir tiempo por factura en al menos 70% por operario.
- Precisión de mapeo proveedor/producto >= 92% en dataset real inicial.
- 99% disponibilidad del pipeline en horario operativo.

## Alcance (In-Scope)
- App móvil para captura multi-página y revisión (offline básico, sincronización).
- Servicio de extracción imagen→JSON (OCR + NLU) probado con facturas reales.
- Servicio de mapeo proveedor↔Customer y producto↔catálogo (fuzzy + embeddings).
- Pipeline asíncrono para preprocesamiento, fusión de páginas, normalización y enriquecimiento.
- Adaptador para escribir `Document`/`DocumentItem` en el POS (sandbox y producción) — soportando múltiples mecanismos de persistencia (SQLite, PostgreSQL, MySQL o APIs).
- Interfaz de revisión en app para corregir proveedor y líneas no mapeadas.
- Métricas y logging básico, autenticación y cifrado en tránsito.

## Fuera de alcance (por ahora)
- Optimización automática de precios y ajustes de margen.
- Reportes analíticos avanzados en tiempo real.
- Reescritura del POS ni cambios estructurales profundos (solo adaptadores).

## Requisitos funcionales clave
1. Captura: permitir 1+ fotos por documento y vista previa de páginas.
2. Procesamiento: unificación páginas → JSON con proveedor, fecha, líneas, cantidades, precios.
3. Mapeo: sugerir correspondencias con puntuación; permitir selección manual.
4. Edición: editar cantidad, precio y asignar producto/proveedor nuevo.
5. Persistencia: crear/actualizar `Document` en el POS de forma transaccional.
6. UX: flujo claro para revisar y confirmar antes de escribir en POS.

## Requisitos no funcionales
- Escalabilidad: admitir aumento de volumen por factor 10 sin rediseño.
- Latencia: procesar documento promedio (2 páginas) en <30s en pipeline estándar.
- Seguridad: TLS, autenticación con tokens y rotación de claves.
- Privacidad: políticas de retención y manejo de PII conforme a normativas locales.

## Datos y accesos necesarios
- Muestras reales de facturas (varios proveedores) para evaluación OCR/ML.
- Esquema y accesos al/los sistema(s) de almacenamiento del POS (SQLite, PostgreSQL, MySQL o API) en sandbox y lista de triggers que afectan stock.
- Catálogo de productos del POS (códigos, precios, flags de edición de precio).

## Criterios de aceptación (MVP)
- Captura y procesamiento end-to-end con dataset de prueba: >=90% extracción de campos clave.
- Flujo de revisión usable por operarias y escritura correcta en POS en sandbox.
- Documentación mínima para despliegue (readme, runbook básico).

## Suposiciones
- El POS permite escritura mediante adaptador (acceso a SQLite o API).
- El POS permite escritura mediante adaptador (acceso a bases de datos como SQLite/Postgres/MySQL o APIs).
- Disponibilidad de ejemplos de facturas para entrenamiento/validación.

## Próximos pasos inmediatos
1. Inventariar y obtener 50–200 facturas reales representativas.
2. Obtener el esquema SQLite del POS y preparar sandbox.
3. Probar extracción con 2–3 proveedores usando Claude Vision / alternativa.
4. Definir primer backlog de tareas (sprint 0).

---

Archivo creado para iniciar la definición de alcance; actualizar iterativamente con hallazgos y decisiones técnicas.