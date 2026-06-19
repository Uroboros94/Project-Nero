# Arquitectura completa integrada: Flujo de sincronización

Este documento muestra cómo los componentes de `Arch.md` trabajan juntos en el flujo de sincronización offline/online.

---

## 1️⃣ Captura local (Síncrono)

```
┌────────────────────────────────────────────┐
│  presentation/capture/                    │
│  - CameraScreen.kt (multi-página)         │
│  - CaptureViewModel.kt                    │
└────────────┬─────────────────────────────┘
             │ User toma fotos
             ▼
┌────────────────────────────────────────────┐
│  data/local/image_storage/                │
│  - ImageMemoryCache.kt (ByteArray, RAM)   │
│  - Imágenes volátiles, no persisten       │
└────────────┬─────────────────────────────┘
             │ "Procesar"
             ▼
┌────────────────────────────────────────────┐
│  domain/usecase/capture/                  │
│  - CaptureInvoiceUseCase.kt               │
└────────────┬─────────────────────────────┘
             │ Crear documento
             ▼
┌────────────────────────────────────────────┐
│  data/local/database/                     │
│  - InvoiceDocumentEntity (Room)           │
│  - Estado: CAPTURED                       │
│  - Serializar fotos en separate table     │
└────────────┬─────────────────────────────┘
             │ Disparar SyncWorker
             ▼
      [STEP 2: Sincronización]
```

---

## 2️⃣ Sincronización y failover (WorkManager + ConnectivityListener)

```
┌─────────────────────────────────────────────────────────────┐
│  data/local/worker/                                         │
│  - SyncWorker.kt (CoroutineWorker)                         │
│  - ConnectivityListener.kt (observa cambios de red)        │
└────────────┬──────────────────────────────────────────────┘
             │ Timer 30s O connectivity change
             ▼
    ┌────────────────────────┐
    │ ¿Internet disponible?  │
    │ (ConnectivityManager)  │
    └────────┬───────────────┘
     NO ┌────┴────┐ YES
        │         │
        ▼         ▼
    ┌───────┐  ┌──────────────────────────┐
    │ SKIP  │  │ Proceder (STEP 3+)       │
    │ Retry│  │ Descargar de cola local   │
    │ later│  └──────────────────────────┘
    └───────┘
```

---

## 3️⃣ Extracción OCR (Remoto, asincrónico)

```
┌────────────────────────────────────────────┐
│  domain/usecase/extraction/                │
│  - ExtractInvoiceDataUseCase.kt            │
└────────────┬─────────────────────────────┘
             │ Coordina extracción
             ▼
┌────────────────────────────────────────────┐
│  data/repository/                          │
│  - DocumentOCRRepository.kt                │
│  - Lee imágenes de Room, llama servicio    │
└────────────┬─────────────────────────────┘
             │ POST /extract
             ▼
┌────────────────────────────────────────────┐
│  data/remote/services/                     │
│  - DocumentOCRService.kt                   │
│  - Llama backend Nero (REST API)           │
└────────────┬─────────────────────────────┘
             │ HTTP POST
             ▼
┌────────────────────────────────────────────┐
│  BACKEND NERO (Remoto)                     │
│  - Fusiona multi-página                    │
│  - Llama Claude Vision API                 │
│  - Extrae: proveedor, fecha, líneas       │
│  - Retorna JSON estructurado               │
└────────────┬─────────────────────────────┘
             │ Response: ExtractedData
             ▼
┌────────────────────────────────────────────┐
│  data/local/database/                      │
│  - ExtractedDataEntity (Room)              │
│  - Estado: EXTRACTED                       │
│  - Guardar JSON extraído                   │
└────────────┬─────────────────────────────┘
             │ ¿Precisión >= 90%?
        NO ┌─┴──┐ YES
           │    │
        FALLAR  ▼
     (Retry)  [STEP 4: Matching]
```

---

## 4️⃣ Matching (Local + Remoto)

```
┌────────────────────────────────────────────┐
│  domain/usecase/matching/                  │
│  - MatchSupplierUseCase.kt                 │
│  - MatchProductUseCase.kt                  │
└────────────┬─────────────────────────────┘
             │ Orquesta matching
             ▼
┌────────────────────────────────────────────┐
│  data/repository/                          │
│  - MatchingRepository.kt                   │
│  - Cache local de catálogo                 │
└────────────┬─────────────────────────────┘
             │
    ┌────────┴──────────┐
    │                   │
    ▼                   ▼
┌──────────────┐  ┌──────────────────┐
│ Fuzzy Local  │  │ Semantic Remoto  │
│ (Si match<85%
)│  │ (Embeddings)   │
└──┬───────────┘  └────┬─────────────┘
   │ Levenshtein   │ POST /match
   └────┬──────────┘
        ▼
┌────────────────────────────────────────────┐
│  domain/service/                           │
│  - MatchingEngine.kt (interfaz)            │
└────────────┬─────────────────────────────┘
             │ MatchingResult[supplier, products]
             ▼
┌────────────────────────────────────────────┐
│  data/local/database/                      │
│  - MatchingCacheEntity (Room)              │
│  - Estado: MATCHED                         │
│  - Guardar sugerencias con scores          │
└────────────┬─────────────────────────────┘
             │ [STEP 5: Revisión User]
```

---

## 5️⃣ Revisión por usuario (UI Síncrono)

```
┌────────────────────────────────────────────┐
│  presentation/review/                      │
│  - ReviewInvoiceScreen.kt                  │
│  - MatchSupplierDialog.kt                  │
│  - MatchProductDialog.kt                   │
│  - EditPriceDialog.kt                      │
│  - ReviewViewModel.kt                      │
└────────────┬─────────────────────────────┘
             │ User confirma/edita
             ▼
┌────────────────────────────────────────────┐
│  domain/usecase/review/                    │
│  - ReviewAndEditInvoiceUseCase.kt          │
└────────────┬─────────────────────────────┘
             │ Guardar cambios
             ▼
┌────────────────────────────────────────────┐
│  data/local/database/                      │
│  - InvoiceDocumentEntity (actualizar)      │
│  - Estado: REVIEWED                        │
│  - Persistir cambios del usuario           │
└────────────┬─────────────────────────────┘
             │ [STEP 6: Escritura POS]
```

---

## 6️⃣ Escritura a POS (Transaccional con failover)

```
┌────────────────────────────────────────────┐
│  domain/usecase/sync/                      │
│  - ProcessInvoicePipelineUseCase.kt        │
│  - WriteToPosUseCase.kt                    │
└────────────┬─────────────────────────────┘
             │ Llamar backend
             ▼
┌────────────────────────────────────────────┐
│  data/repository/                          │
│  - SyncRepository.kt                       │
│  - Estado: WRITING                         │
└────────────┬─────────────────────────────┘
             │ POST /write-to-pos
             ▼
┌────────────────────────────────────────────┐
│  data/remote/services/                     │
│  - PosAdapterService.kt                    │
└────────────┬─────────────────────────────┘
             │ HTTP POST
             ▼
┌────────────────────────────────────────────┐
│  BACKEND NERO (Remoto)                     │
│  - Selecciona PosAdapterFactory            │
│  - Según: appConfig.posType                │
└────────────┬─────────────────────────────┘
             │
    ┌────────┴───────────────────┐
    │                            │
    ▼                            ▼
┌──────────────┐        ┌─────────────────┐
│ SQLite       │        │ Postgres/MySQL  │
│Adapter.kt    │        │Adapter.kt       │
└──┬───────────┘        └────┬────────────┘
   │ INSERT Document      │ INSERT Document
   │ INSERT Items         │ INSERT Items
   │ COMMIT/ROLLBACK      │ COMMIT/ROLLBACK
   └────────┬─────────────┘
            ▼
        ┌──────────────────────────┐
        │ ¿Éxito?                  │
        └────────┬────────────┬────┘
             YES │            │ NO
                 │            │
            [OK]  │        ┌───▼──────────────┐
                 ▼        │ Clasificar error: │
          ┌──────────┐    └───┬───────────────┘
          │ State:   │        │
          │ WRITTEN  │   ┌────┴──────────────┐
          └──────────┘   │                   │
                    Transient  │              │ Validation
                    (Red)      │              │ (Datos)
                         ┌─────▼──┐     ┌────▼────┐
                         │Enqueue  │     │FAILED_  │
                         │Retry    │     │FATAL    │
                         │Local    │     │Notify   │
                         └─────────┘     └─────────┘
```

---

## 7️⃣ Cola offline (SyncQueueEntity)

```
┌─────────────────────────────────────────────────┐
│  data/local/database/entity/                    │
│  - SyncQueueEntity.kt                           │
│    * documentId (PK)                            │
│    * documentJson (serializado)                 │
│    * state (OFFLINE_QUEUED, FAILED_WRITE)      │
│    * retryCount, nextRetryTimeMs                │
│    * lastErrorCode, lastErrorMessage            │
└─────────────────────────────────────────────────┘
             ▲
             │ Si error transient (red)
             │ O sin internet
             │
┌─────────────────────────────────────────────────┐
│  data/local/worker/                             │
│  - SyncWorker.kt                                │
│    - calculateBackoff(retryCount)               │
│    - 1s, 2s, 4s, 8s, 16s (exponencial)        │
└─────────────────────────────────────────────────┘
             ▲
             │ Cuando internet vuelve
             │ ConnectivityListener dispara
             │
┌─────────────────────────────────────────────────┐
│  infrastructure/connectivity/                   │
│  - ConnectivityManager.kt                       │
│    - observeConnectivity() → Flow<Boolean>     │
│    - Enqueue SyncWorker si conectado            │
└─────────────────────────────────────────────────┘
```

---

## 8️⃣ Observabilidad (Infrastructure Layer)

```
Todos los pasos registran eventos correlacionados:

┌─────────────────────────────────────────────────┐
│  infrastructure/logging/                        │
│  - TraceContext.kt (ThreadLocal de documentId) │
│  - LoggerFactory.kt                             │
│  - TelemetryInterceptor.kt (en Retrofit)       │
└─────────────────────────────────────────────────┘

Ejemplo de logs correlacionados:
  [doc-abc123] Iniciando captura
  [doc-abc123] Imagen 1 guardada (150 KB)
  [doc-abc123] Imagen 2 guardada (145 KB)
  [doc-abc123] POST /extract → 200 OK
  [doc-abc123] Precisión OCR: 94%
  [doc-abc123] Matching proveedor: 92% score
  [doc-abc123] User confirmó
  [doc-abc123] POST /write-to-pos → 201 Created
  [doc-abc123] Documento escrito: DOC-2026-000991

┌─────────────────────────────────────────────────┐
│  infrastructure/metrics/                        │
│  - MetricsCollector.kt                          │
│  - DocumentMetrics.kt (latencia, estado)       │
│  - ErrorReporter.kt                             │
└─────────────────────────────────────────────────┘

Métricas:
  - document_processing_time_ms: 23500
  - extract_duration_ms: 8200
  - match_duration_ms: 3100
  - write_duration_ms: 1200
  - final_state: WRITTEN
  - error_count: 0
  - retry_count: 0

┌─────────────────────────────────────────────────┐
│  infrastructure/config/                         │
│  - AppConfig.kt (posType, connectionString)    │
│  - FeatureFlags.kt (canary deploy)             │
│  - AppConfigProvider.kt (carga desde backend)  │
└─────────────────────────────────────────────────┘
```

---

## 9️⃣ Estados y transiciones completas

```
LOCAL CAPTURE:
  CAPTURED → (SyncWorker dispara)

ONLINE PATH:
  CAPTURED
    ↓
  EXTRACTING → (POST /extract)
    ↓
  EXTRACTED
    ↓
  MATCHING → (POST /match)
    ↓
  MATCHED
    ↓
  REVIEWING → (User)
    ↓
  REVIEWED
    ↓
  WRITING → (POST /write-to-pos)
    ↓
  WRITTEN ✓

OFFLINE PATH (sin internet):
  CAPTURED
    ↓
  OFFLINE_QUEUED → (Guardar en SyncQueueEntity)
    ↓
  (Esperar... esperar...)
    ↓
  (ConnectivityListener: ¡Internet vuelve!)
    ↓
  SyncWorker dispara
    ↓
  Reintentar desde EXTRACTING...

ERROR PATH (Transient):
  WRITING
    ↓
  FAILED_WRITE (POST timeout)
    ↓
  enqueuForRetry(backoff=2s)
    ↓
  Esperar 2s → Retry
    ↓
  WRITING (intento 2)
    ↓
  WRITTEN ✓

ERROR PATH (Fatal):
  WRITING
    ↓
  FAILED_WRITE (Validación POS fallida)
    ↓
  FAILED_FATAL (Marcar irrecuperable)
    ↓
  Notificar usuario + crear ticket soporte
```

---

## 🔟 Inyección de dependencias (Hilt)

```
┌─────────────────────────────────────────────────┐
│  infrastructure/di/                             │
│  - RepositoryModule.kt                          │
│    @Provides fun provideSyncRepository(...)    │
│  - ServiceModule.kt                             │
│    @Provides fun provideDocumentOCRService...  │
│  - PosAdapterModule.kt                          │
│    @Provides fun providePosAdapterFactory(...)│
│  - AppModule.kt                                 │
│    Singleton instances                          │
└─────────────────────────────────────────────────┘

Inyección en WorkManager:
  @HiltWorker
  class SyncWorker @AssistedInject constructor(
    @Assisted context: Context,
    @Assisted params: WorkerParameters,
    private val syncRepository: SyncRepository,
    private val posAdapterFactory: PosAdapterFactory,
    private val metricsCollector: MetricsCollector
  ) : CoroutineWorker(context, params)
```

---

## Flujo completo resumido

```
┌─────────────────┐
│  User Capture   │
│  Multi-página   │
└────────┬────────┘
         │ ImageMemoryCache (RAM)
         ▼
┌─────────────────┐
│  SyncWorker     │
│  ¿Internet?     │
└────────┬────────┘
         │
    ┌────┴─────────────────┐
    │                      │
  NO │ YES                 │
    │  │                   │
    ▼  ▼                   ▼
┌───────────┐    ┌──────────────────────┐
│OFFLINE_   │    │ POST /extract        │
│QUEUED     │    │ POST /match          │
│(Retry)    │    │ User review          │
└───┬───────┘    │ POST /write-to-pos   │
    │            └──────────────────────┘
    │                    │ OK?
    │              ┌─────┴─────┐
    │              │           │
    │              ▼           ▼
    │        ┌─────────┐  ┌──────────┐
    │        │WRITTEN  │  │FAILED_   │
    │        │✓        │  │WRITE     │
    │        └─────────┘  └────┬─────┘
    │                          │
    │          ┌───────────────┘
    │          │ Retry (exponential)
    │          │ O esperar internet
    │          │
    └──────────┴─── [Loop]

FINAL STATE: WRITTEN ✓
  + Limpiar ImageMemoryCache
  + Remover SyncQueueEntity
  + Registrar métricas
  + Confirmation screen
```

---

## Checklist de integración

- [x] Captura multi-página (ImageMemoryCache volátil).
- [x] Persistencia local (Room: InvoiceDocumentEntity, SyncQueueEntity).
- [x] SyncWorker con manejo de internet y failover.
- [x] PosAdapterFactory para múltiples DBs.
- [x] Pipeline orquestado (5 pasos asincronos).
- [x] Logging correlacionado (TraceContext).
- [x] Métricas y observabilidad.
- [x] Estados y transiciones explícitas.
- [x] Reintentos exponenciales.
- [x] Hilt para DI.

---

**Referencia cruzada:**
- Arch.md: Estructura completa de paquetes.
- integration-failover-proposal.md: Detalles de SyncWorker y adaptadores.
- pipeline-orchestration.md: ProcessInvoicePipelineUseCase.
- observability-proposal.md: TraceContext y SyncState.
- matching-explanation.md: MatchingEngine.
