# Propuesta: Observabilidad y Manejo de Errores

## 4. Observabilidad y manejo de errores — Estrategia

### Contexto
El pipeline de procesamiento de facturas es crítico: un error no manejado puede dejar documentos en estado inconsistente (ej. capturado pero no procesado, procesado pero no sincronizado).

### Propuesta

#### A) Logging correlacionado
- Cada invocación de pipeline genera un `DocumentId` (UUID).
- Todos los logs incluyen `documentId`, `userId`, `timestamp`, `step` (capture/extract/match/write).
- Implementar mediante `TraceContext` (ThreadLocal en Android con WorkManager).

```kotlin
// infrastructure/logging/TraceContext.kt
object TraceContext {
    private val threadLocal = ThreadLocal<String>()
    
    fun setDocumentId(id: String) = threadLocal.set(id)
    fun getDocumentId(): String? = threadLocal.get()
    fun clear() = threadLocal.remove()
}

// Usage en cualquier layer
Log.d("NERO", "[${TraceContext.getDocumentId()}] Iniciando extracción...")
```

#### B) Estados y transiciones con historial
Modelar máquina de estados con auditoría completa:

```kotlin
// domain/model/SyncState.kt
enum class SyncState {
    CAPTURED,           // Foto(s) capturada(s), en caché local
    EXTRACTING,         // Enviando a servicio OCR
    EXTRACTED,          // Datos extraídos, pendiente mapeo
    MATCHING,           // Buscando proveedor/productos
    MATCHED,            // Mapeos sugeridos, esperando revisión
    REVIEWING,          // Operaria editando
    REVIEWED,           // Listo para sincronizar
    WRITING,            // Escribiendo a POS
    WRITTEN,            // Éxito en POS
    FAILED_EXTRACT,     // Error en extracción (reintentable)
    FAILED_MATCH,       // Error en mapeo (reintentable)
    FAILED_WRITE,       // Error en escritura (reintentable)
    FAILED_FATAL,       // Error irrecuperable (requiere intervención)
    OFFLINE_QUEUED      // Cola de espera (sin internet)
}

// domain/model/SyncHistory.kt
data class SyncHistory(
    val documentId: String,
    val transitions: List<StateTransition>,  // Quién, cuándo, de qué a qué
    val errors: List<ErrorEvent>              // Qué pasó, cuándo, contexto
)

data class StateTransition(
    val from: SyncState,
    val to: SyncState,
    val timestamp: Long,
    val userId: String? = null,  // null si automático
    val reason: String? = null
)

data class ErrorEvent(
    val state: SyncState,
    val errorCode: String,
    val message: String,
    val stackTrace: String? = null,
    val retryCount: Int = 0,
    val nextRetryTime: Long? = null,
    val timestamp: Long
)
```

#### C) Reintentos inteligentes
- Errores transitorios (red): reintento exponencial (1s, 2s, 4s, 8s, 16s).
- Errores de validación (OCR baja precisión): enviar a cola de revisión manual.
- Errores fatales (POS inaccesible): fallback offline, sincronizar cuando internet vuelva.

```kotlin
// domain/service/RetryPolicy.kt
sealed class RetryPolicy {
    data class Exponential(val baseDelayMs: Long = 1000, val maxRetries: Int = 5) : RetryPolicy()
    object Manual : RetryPolicy()  // Requiere intervención
    object None : RetryPolicy()    // No reintentar
}

// domain/model/ErrorClassification.kt
enum class ErrorSeverity {
    TRANSIENT,   // Reintentar automáticamente
    VALIDATION,  // Revisar manualmente
    FATAL        // Requiere soporte
}
```

#### D) Métricas y alertas
- **Por documento**: tiempo total, tiempo por step, tasa de error por step.
- **Agregadas**: throughput (docs/hora), latencia p50/p95/p99, tasa de éxito.
- Enviar a backend para dashboard (Maritza puede ver su progreso).

```kotlin
// infrastructure/metrics/MetricsCollector.kt
data class DocumentMetrics(
    val documentId: String,
    val captureDurationMs: Long,
    val extractDurationMs: Long,
    val matchDurationMs: Long,
    val writeDurationMs: Long,
    val totalDurationMs: Long,
    val finalState: SyncState,
    val errorCount: Int,
    val retryCount: Int,
    val timestamp: Long
)
```

#### E) Reporting y alertas
- Dashboard en backend: estado de documentos, alertas por cajera, SLA (documento escrito <5 min).
- Notificaciones push si documento falla por >2 reintentos.
- Endpoint `/health` en app: reporte local de caché, connectivity, estado del worker.

---

## Implementación por fase

### MVP (Sprint 0–1)
- TraceContext y logging básico por estado.
- Tabla en Room para historial (SyncHistory).
- Reintentos exponenciales para errores de red.

### Next (Sprint 2–3)
- Dashboard backend con métricas agregadas.
- Alertas push por fallos.
- Modo manual para OCR baja precisión.

### Later
- Reportes de cost/performance por cajera/proveedor.
- Optimización de SLA basada en ML (predecir fallos).

---

## Endpoints y contratos

### `/metrics` (GET)
```json
{
  "documents": {
    "total": 150,
    "succeeded": 145,
    "failed": 3,
    "pending": 2
  },
  "latency": {
    "p50_ms": 15000,
    "p95_ms": 28000,
    "p99_ms": 45000
  },
  "errors": {
    "extract_failures": 2,
    "match_failures": 1,
    "write_failures": 0
  },
  "last_sync": "2026-06-19T14:23:00Z"
}
```

### `/document/{id}/history` (GET)
```json
{
  "documentId": "doc-abc123",
  "states": [
    { "state": "CAPTURED", "timestamp": "2026-06-19T14:00:00Z" },
    { "state": "EXTRACTING", "timestamp": "2026-06-19T14:00:05Z" },
    { "state": "EXTRACTED", "timestamp": "2026-06-19T14:00:15Z" }
  ],
  "errors": [
    {
      "state": "EXTRACTED",
      "errorCode": "SUPPLIER_NOT_FOUND",
      "message": "No se encontró proveedor 'Distribuidora XYZ'",
      "retryCount": 0,
      "timestamp": "2026-06-19T14:00:16Z"
    }
  ]
}
```

---

Usar como blueprint para `infrastructure/` layer en Arch.md.
