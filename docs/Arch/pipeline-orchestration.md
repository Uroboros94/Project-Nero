# Pipeline de procesamiento — Orquestación asincrónica

## Flujo completo (MVP)

```
┌─────────────────────────────────────────────────────────────────────┐
│  CAPTURA (Local, síncrono)                                          │
│  - User toma 1+ fotos de la factura                                 │
│  - Guardadas en RAM (ImageMemoryCache)                              │
│  - Estado: CAPTURED                                                 │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
                    UI: Preview fotos
                           │
                ┌──────────▼──────────┐
                │ User presiona       │
                │ "Procesar 2 fotos"  │
                └──────────┬──────────┘
                           │
┌──────────────────────────▼──────────────────────────────────────────┐
│  PERSISTENCIA LOCAL (Síncrono)                                      │
│  - Crear InvoiceDocument en Room (estado: CAPTURED)                │
│  - Serializar fotos en ByteArray → Tabla separate (ImageEntity)    │
│  - Asignar UUID único                                              │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────────────┐
│  SINCRONIZACIÓN (Asincrónico vía WorkManager)                       │
│  - SyncWorker se dispara (trigger manual o timer 30s)              │
│  - Estado: EXTRACTING                                              │
│  - Intenta procesar documentos en cola (CAPTURED, OFFLINE_QUEUED) │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
              ┌────────────▼────────────┐
              │  ¿Internet disponible?  │
              └────────────┬────────────┘
                 NO    ┌───┴────┐    YES
                       │        │
                ┌──────▼──┐ ┌───▼──────────┐
                │ Esperar │ │ Proceder     │
                │ Retry   │ │ (Next step)  │
                └─────────┘ └───┬──────────┘
                                │
┌───────────────────────────────▼──────────────────────────────────────┐
│  EXTRACCIÓN OCR (Remoto, async)                                      │
│  - POST /extract (multi-página)                                      │
│    {                                                                 │
│      "documentId": "doc-abc123",                                     │
│      "pages": [base64_img1, base64_img2]                             │
│    }                                                                 │
│  - DocumentOCRService (backend):                                     │
│    1. Fusionar páginas (si varias)                                  │
│    2. Llamar Claude Vision API                                      │
│    3. Extraer: proveedor, fecha, líneas, cantidades, precios       │
│    4. Normalizar campos                                             │
│  - Respuesta: JSON estructurado                                      │
│  - Estado: EXTRACTED                                                │
│  - Guardar en Room: ExtractedDataEntity                             │
└───────────────────────────┬──────────────────────────────────────────┘
                            │
              ┌─────────────▼─────────────┐
              │  ¿Precisión OCR >= 90%?   │
              └─────────────┬─────────────┘
                 NO   ┌─────┴────┐   YES
                      │          │
          ┌───────────▼───┐ ┌───▼────────┐
          │ FAILED_EXTRACT│ │ Proceder   │
          │ (notify user) │ │ (Next step)│
          └───────────────┘ └───┬────────┘
                                 │
┌────────────────────────────────▼──────────────────────────────────────┐
│  MAPEO / MATCHING (Remoto, async + Local)                            │
│  - Para cada proveedor y producto:                                   │
│    1. Local: Búsqueda exacta en caché                               │
│    2. Local: Fuzzy matching (Levenshtein)                           │
│    3. Remoto: Búsqueda semántica (embeddings, si fuzzy < 85%)      │
│  - Generar lista de alternativas con scores                          │
│  - Estado: MATCHED                                                   │
│  - Guardar sugerencias en Room                                       │
│                                                                      │
│  Respuesta esperada:                                                 │
│  {                                                                  │
│    "supplier": {                                                    │
│      "extracted": "Distrib. La Cosecha",                            │
│      "matched": { "id": 174, "name": "Distribuidora La Cosecha..." }│
│      "score": 0.92,                                                 │
│      "alternatives": [...]                                          │
│    },                                                                │
│    "products": [                                                     │
│      {                                                               │
│        "extracted": "ACEITE OLIVA",                                 │
│        "matched": { "code": 32308, "name": "OLIVETTO..." },         │
│        "score": 0.95                                                │
│      }                                                               │
│    ]                                                                 │
│  }                                                                   │
└────────────────────────────┬──────────────────────────────────────────┘
                             │
┌────────────────────────────▼──────────────────────────────────────────┐
│  REVISIÓN (User interaction, síncrono/UI)                            │
│  - Mostrar ReviewScreen con datos extraídos + matches                │
│  - User puede:                                                        │
│    1. Confirmar matches (score >= 85% auto-confirm en MVP)         │
│    2. Seleccionar alternativas (si score 70-84%)                    │
│    3. Editar cantidad, precio                                       │
│    4. Crear nuevo proveedor/producto                                │
│  - Estado: REVIEWING → REVIEWED (cuando user presiona "Confirmar") │
│  - Guardar cambios en Room                                          │
└────────────────────────────┬──────────────────────────────────────────┘
                             │
┌────────────────────────────▼──────────────────────────────────────────┐
│  ESCRITURA A POS (Remoto, async, transaccional)                      │
│  - Estado: WRITING                                                   │
│  - POST /write-to-pos (backend):                                    │
│    1. Seleccionar adaptador POS (sqlite/postgres/mysql/http)       │
│    2. Crear transacción                                             │
│    3. INSERT Document (con validaciones)                            │
│    4. INSERT DocumentItems (líneas)                                 │
│    5. Verificar triggers (¿cambió stock?)                           │
│    6. COMMIT si todo OK, ROLLBACK si error                          │
│  - Respuesta:                                                         │
│    {                                                                 │
│      "success": true,                                               │
│      "pos_document_id": "DOC-2026-000991",                          │
│      "timestamp": "2026-06-19T14:23:00Z"                            │
│    }                                                                 │
│  - Estado: WRITTEN                                                   │
│  - Guardar confirmación en Room                                      │
│  - Limpiar ImageMemoryCache (liberar RAM)                            │
│  - Limpiar SyncQueueEntity                                           │
└────────────────────────────┬──────────────────────────────────────────┘
                             │
┌────────────────────────────▼──────────────────────────────────────────┐
│  CONFIRMACIÓN (UI)                                                    │
│  - Mostrar SuccessScreen con resumen                                │
│  - "Documento ingresado: 3 productos, $268.488"                      │
│  - Botones: "Escanear otra factura" o "Ir al inicio"               │
│  - Registrar métricas y auditoría                                    │
└────────────────────────────────────────────────────────────────────────┘
```

---

## Orquestación: ProcessInvoicePipelineUseCase

```kotlin
// domain/usecase/sync/ProcessInvoicePipelineUseCase.kt
class ProcessInvoicePipelineUseCase(
    private val invoiceRepository: InvoiceRepository,
    private val documentOCRRepository: DocumentOCRRepository,
    private val matchingRepository: MatchingRepository,
    private val syncRepository: SyncRepository,
    private val metricsCollector: MetricsCollector,
    private val logger: Logger
) {
    
    suspend fun execute(
        documentId: String,
        onProgressUpdate: suspend (SyncState) -> Unit
    ): Result<PosDocument> = withContext(Dispatchers.IO) {
        val startTime = System.currentTimeMillis()
        val traceId = UUID.randomUUID().toString()
        TraceContext.setDocumentId(traceId)
        
        try {
            // 1. CAPTURED → EXTRACTING
            logger.d("[$traceId] Iniciando pipeline para $documentId")
            onProgressUpdate(SyncState.EXTRACTING)
            
            // 2. EXTRACCIÓN
            val extracted = extractInvoice(documentId)
                ?: return@withContext Result.failure(Exception("Extracción fallida"))
            logger.d("[$traceId] Extracción completada")
            onProgressUpdate(SyncState.EXTRACTED)
            
            // 3. MATCHING
            val matched = matchInvoiceData(extracted)
                ?: return@withContext Result.failure(Exception("Matching fallido"))
            logger.d("[$traceId] Matching completado")
            onProgressUpdate(SyncState.MATCHED)
            
            // 4. WAITING FOR REVIEW (User action)
            // UI maneja esto; cuando user confirma, llamar a next step
            
            // 5. ESCRIBIR A POS
            onProgressUpdate(SyncState.WRITING)
            val written = writeToPos(matched)
                ?: return@withContext Result.failure(Exception("Escritura POS fallida"))
            logger.d("[$traceId] Documento escrito en POS")
            onProgressUpdate(SyncState.WRITTEN)
            
            // 6. Limpiar recursos
            invoiceRepository.clearImageCache(documentId)
            syncRepository.removeSyncQueue(documentId)
            
            val totalTime = System.currentTimeMillis() - startTime
            metricsCollector.recordSuccess(
                documentId = documentId,
                totalDurationMs = totalTime
            )
            
            return@withContext Result.success(written)
            
        } catch (e: Exception) {
            logger.e("[$traceId] Pipeline fallido: ${e.message}", e)
            handlePipelineError(documentId, e, SyncState.FAILED_FATAL)
            return@withContext Result.failure(e)
        } finally {
            TraceContext.clear()
        }
    }
    
    private suspend fun extractInvoice(documentId: String): ExtractedData? {
        return try {
            val images = invoiceRepository.getImagesByDocumentId(documentId)
            val result = documentOCRRepository.extract(images, documentId)
            
            // Validar precisión
            if (result.confidenceScore < 0.90) {
                syncRepository.updateState(documentId, SyncState.FAILED_EXTRACT)
                return null
            }
            
            syncRepository.saveExtractedData(documentId, result)
            result
        } catch (e: Exception) {
            syncRepository.enqueuForRetry(
                documentId,
                errorCode = "EXTRACTION_ERROR",
                errorMessage = e.message ?: "Unknown"
            )
            null
        }
    }
    
    private suspend fun matchInvoiceData(extracted: ExtractedData): MatchingData? {
        return try {
            // Matching de proveedor
            val supplierMatch = matchingRepository.matchSupplier(extracted.supplierName)
            
            // Matching de productos
            val productMatches = extracted.products.map { product ->
                matchingRepository.matchProduct(
                    product.name,
                    supplierMatch.matchedSupplierId
                )
            }
            
            MatchingData(
                extracted = extracted,
                supplierMatch = supplierMatch,
                productMatches = productMatches
            )
        } catch (e: Exception) {
            logger.e("Matching error: ${e.message}", e)
            null
        }
    }
    
    private suspend fun writeToPos(matched: MatchingData): PosDocument? {
        return try {
            val posDocument = PosDocument.fromMatching(matched)
            syncRepository.writeToPosAdapter(posDocument)
            posDocument
        } catch (e: Exception) {
            logger.e("Write to POS error: ${e.message}", e)
            null
        }
    }
    
    private suspend fun handlePipelineError(
        documentId: String,
        error: Exception,
        state: SyncState
    ) {
        syncRepository.updateState(documentId, state)
        syncRepository.recordError(
            documentId,
            errorCode = error::class.simpleName ?: "UNKNOWN",
            errorMessage = error.message ?: "No message"
        )
        metricsCollector.recordFailure(documentId, error)
    }
}
```

---

## Estados y transiciones

```kotlin
// domain/model/SyncState.kt
enum class SyncState {
    CAPTURED,           // Local: foto(s) capturada
    EXTRACTING,         // Remoto: enviando a OCR
    EXTRACTED,          // Remoto: datos extraídos
    MATCHING,           // Remoto: buscando matches
    MATCHED,            // Pendiente review
    REVIEWING,          // User editando
    REVIEWED,           // Listo para escribir
    WRITING,            // Remoto: escribiendo a POS
    WRITTEN,            // ✓ Éxito
    
    // Estados de error (reintentables)
    FAILED_EXTRACT,     // Error OCR (reintento automático)
    FAILED_MATCH,       // Error matching (reintento automático)
    FAILED_WRITE,       // Error escritura POS (reintento automático)
    
    // Estados especiales
    OFFLINE_QUEUED,     // Sin internet (reintento cuando vuelva)
    FAILED_FATAL,       // Error irrecuperable (requiere intervención)
    REVIEWED_MANUAL     // User lo revisó pero POS aún no escrito
}
```

---

## Failover e internet offline

### Detección de conectividad

```kotlin
// infrastructure/connectivity/ConnectivityManager.kt
class ConnectivityManager(private val context: Context) {
    private val connectivityManager = context
        .getSystemService(Context.CONNECTIVITY_SERVICE) as ConnectivityManager
    
    fun isOnline(): Boolean {
        val network = connectivityManager.activeNetwork ?: return false
        val capabilities = connectivityManager.getNetworkCapabilities(network) ?: return false
        return capabilities.hasCapability(NetworkCapabilities.NET_CAPABILITY_INTERNET)
    }
    
    fun observeConnectivity(): Flow<Boolean> = callbackFlow {
        val callback = object : ConnectivityManager.NetworkCallback() {
            override fun onAvailable(network: Network) {
                trySend(true)
            }
            override fun onLost(network: Network) {
                trySend(false)
            }
        }
        connectivityManager.registerNetworkCallback(NetworkRequest.Builder().build(), callback)
        awaitClose { connectivityManager.unregisterNetworkCallback(callback) }
    }
}

// En WorkManager: observar cambios y disparar sync cuando internet vuelva
WorkManager.getInstance(context)
    .observeConnectivity()
    .collect { isOnline ->
        if (isOnline) {
            WorkManager.getInstance(context).enqueueUniqueWork(
                "sync_worker",
                ExistingWorkPolicy.KEEP,
                SyncWorker.createRequest()
            )
        }
    }
```

---

## Métricas por paso

```kotlin
data class PipelineMetrics(
    val documentId: String,
    val captureDurationMs: Long,
    val extractDurationMs: Long,
    val matchDurationMs: Long,
    val writeDurationMs: Long,
    val totalDurationMs: Long,
    val finalState: SyncState,
    val errorCount: Int,
    val ocrConfidence: Float,
    val matcherScoreAvg: Float
)
```

---

## Checkpoints clave

- [ ] `ProcessInvoicePipelineUseCase` orquesta los 5 pasos.
- [ ] Cada paso es independiente y testeable.
- [ ] Estados persistidos en Room para recuperación ante crash.
- [ ] Reintentos exponenciales en SyncWorker.
- [ ] Métricas correlacionadas por documentId.
- [ ] Tests: simular offline, timeout, y estados fallidos.

---

Ver also: `integration-failover-proposal.md` para detalles de WorkManager y failover.
