# Sprint 0 — Checklist de accesos y muestras

Resumen: checklist listo para compartir con Infra y Operaciones para habilitar el sandbox y recopilar muestras.

## 1) Muestras de facturas (50–200)
- Formatos aceptados: `pdf`, `jpg`, `png`.
- Variación requerida: al menos 10 proveedores distintos, varias calidades (buena/media/baja), facturas de 1–4 páginas.
- Metadatos por fichero (CSV o JSON): `filename, proveedor, fecha, numero_documento, paginas, observaciones`.
- Entregar en carpeta segura (S3/bucket compartido o carpeta en drive con acceso controlado).

## 2) Dataset de prueba rápido (10 facturas)
- Seleccionar 10 facturas representativas para pruebas iniciales de OCR.
- Objetivo de prueba rápido: extracción de campos clave >= 90%.

## 3) Acceso POS sandbox
- Tipo de acceso requerido: elegir uno o varios según disponibilidad:
  - SQLite: cadena ejemplo `sqlite:///path/to/pos.db`
  - PostgreSQL: `postgresql://user:password@host:5432/dbname`
  - MySQL: `mysql://user:password@host:3306/dbname`
  - API HTTP: `https://pos.example.com/api/v1` (Bearer token)
- Proveer: host, puerto, usuario, alcance (DB/schema), método de conexión y contacto de infra.

## 4) Esquema y triggers
- Exportar esquema del POS (DDL) y listar triggers/procedimientos que impacten stock o movimientos.
- Indicar tablas relevantes: `Document`, `DocumentItem`, `Product`, `Supplier`, `Stock`.

## 5) Permisos mínimos para usuario de prueba
- SELECT en catálogo y tablas de referencia.
- INSERT/UPDATE en `Document` y `DocumentItem` (sandbox).
- Permiso para consultar metadatos/triggers.

## 6) Seguridad y entrega de credenciales
- Entregar credenciales via secret manager (HashiCorp Vault, AWS Secrets, Azure Key Vault) o mediante archivo cifrado.
- TLS obligatorio; certificados provisionales si es sandbox.

## 7) Sandbox y red
- Sandbox con esquema y triggers igual al prod (anonymized copy o subset).
- Endpoint accesible desde la red de desarrollo (IP whitelisting si aplica) o mediante bastion/SSH tunnel.

## 8) Requisitos del adaptador
- Supportar conexiones a SQLite/Postgres/MySQL y llamadas a APIs REST.
- Modo `dry-run` para validar sin aplicar efectos (evitar cambios de stock).
- Endpoints de salud y métricas.

## 9) Backup y migraciones
- Procedimiento de backup antes de pruebas de integración.
- Scripts de migración versionados si se requieren cambios de esquema (Flyway/liquibase).

## 10) Observabilidad y pruebas
- Tests de integración automatizados contra sandbox: flujo captura→procesamiento→escritura.
- Métricas esperadas: tiempo por documento, tasa de errores, precisión OCR por campo.

## 11) Documentación y entregables
- Lista de contactos (Infra, Operaciones, Producto).
- Instrucciones para acceso (pasos para conectar, ejemplos de connection string).
- Checklist firmado/confirmado por Infra y Operaciones con fecha límite.


---

Guardar este archivo en `docs/alcance/sprint-0-checklist.md` y compartir el link con Infra/Operaciones.
