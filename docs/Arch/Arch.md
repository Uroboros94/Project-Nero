com.datasapiens.nero/
│
├── data/                                 # CAPA DE DATOS
│   ├── local/
│   │   ├── database/                     # Room database (caché, sincronización, historial)
│   │   │   ├── AppDatabase.kt
│   │   │   ├── entity/
│   │   │   │   ├── InvoiceDocumentEntity.kt
│   │   │   │   ├── SyncQueueEntity.kt    # Cola offline
│   │   │   │   ├── SyncHistoryEntity.kt  # Auditoría de estados
│   │   │   │   └── MatchingCacheEntity.kt
│   │   │   └── dao/
│   │   │       ├── InvoiceDocumentDao.kt
│   │   │       ├── SyncQueueDao.kt
│   │   │       └── SyncHistoryDao.kt
│   │   ├── image_storage/                # Imágenes en memoria (ByteArray)
│   │   │   └── ImageMemoryCache.kt       # Cache volátil de fotos
│   │   ├── preferences/                  # EncryptedSharedPreferences
│   │   │   ├── SessionPreferences.kt
│   │   │   └── PosConfigPreferences.kt   # Configuración del adaptador POS
│   │   └── worker/                       # WorkManager para sincronización
│   │       ├── SyncWorker.kt             # Orquestador principal
│   │       └── ConnectivityListener.kt   # Dispara sync cuando internet vuelve
│   │
│   ├── remote/                           # Servicios de red
│   │   ├── interceptor/
│   │   │   ├── AuthInterceptor.kt
│   │   │   └── TelemetryInterceptor.kt   # Tracing distribuido
│   │   │
│   │   ├── services/
│   │   │   ├── AuthService.kt            # Login, tokens
│   │   │   ├── DocumentOCRService.kt     # Extracción de facturas (OCR + NLU)
│   │   │   ├── MatchingService.kt        # Fuzzy + embeddings para mapeo
│   │   │   └── PosAdapterService.kt      # Interfaz unificada para escribir
│   │   │
│   │   ├── pos_adapters/                 # Drivers por tipo de BD/API
│   │   │   ├── IPosAdapter.kt            # Interfaz común
│   │   │   ├── PosAdapterFactory.kt      # Factory para seleccionar adaptador
│   │   │   ├── SqliteAdapter.kt
│   │   │   ├── PostgresAdapter.kt
│   │   │   ├── MySqlAdapter.kt
│   │   │   └── HttpApiAdapter.kt
│   │   │
│   │   └── api/                          # Retrofit interfaces
│   │       ├── NeroApiService.kt
│   │       └── contracts/ (DTOs para requests/responses)
│   │
│   └── repository/                       # Implementaciones que cruzan Local + Remote
│       ├── AuthRepository.kt
│       ├── InvoiceRepository.kt          # Documentos locales
│       ├── DocumentOCRRepository.kt      # Delega a DocumentOCRService
│       ├── MatchingRepository.kt         # Matching con caché local
│       ├── SyncRepository.kt             # Orquestación del pipeline
│       └── PosRepository.kt              # Lectura de catálogo, escritura via adaptadores
│
├── domain/                               # CAPA DE DOMINIO
│   ├── model/
│   │   ├── UserSession.kt
│   │   ├── InvoiceDocument.kt            # Factura capturada
│   │   ├── ExtractedData.kt              # JSON extraído: proveedor, líneas
│   │   ├── MatchingResult.kt             # Resultados de matching con score
│   │   ├── PosDocument.kt                # Documento listo para escribir en POS
│   │   ├── SyncState.kt                  # Estados del pipeline (CAPTURED, EXTRACTING, etc.)
│   │   ├── SyncHistory.kt                # Auditoría: transiciones + errores
│   │   ├── ErrorEvent.kt                 # Información de errores con retry
│   │   ├── Product.kt
│   │   ├── Supplier.kt
│   │   └── MatchType.kt                  # Enum: EXACT, FUZZY, SEMANTIC, NOT_FOUND
│   │
│   ├── service/
│   │   ├── MatchingEngine.kt             # Interfaz abstracta
│   │   ├── PosConnectionManager.kt       # Manejo de conexiones POS
│   │   └── RetryPolicy.kt                # Políticas de reintento
│   │
│   └── usecase/                          # Casos de uso por módulo
│       ├── auth/
│       │   ├── LoginUseCase.kt
│       │   └── LogoutUseCase.kt
│       ├── capture/
│       │   └── CaptureInvoiceUseCase.kt  # Multi-página
│       ├── extraction/
│       │   └── ExtractInvoiceDataUseCase.kt  # OCR + fusión
│       ├── matching/
│       │   ├── MatchSupplierUseCase.kt
│       │   └── MatchProductUseCase.kt
│       ├── review/
│       │   └── ReviewAndEditInvoiceUseCase.kt
│       └── sync/
│           ├── ProcessInvoicePipelineUseCase.kt  # Orquestación: captura→OCR→match→write
│           └── WriteToPosUseCase.kt
│
├── presentation/                         # CAPA DE PRESENTACIÓN
│   ├── auth/
│   │   ├── LoginScreen.kt
│   │   └── LoginViewModel.kt
│   ├── capture/
│   │   ├── CameraScreen.kt               # Multi-página
│   │   ├── CaptureViewModel.kt
│   │   └── PreviewScreen.kt
│   ├── extraction/
│   │   ├── ProcessingScreen.kt           # Spinner + progreso
│   │   └── ProcessingViewModel.kt
│   ├── review/
│   │   ├── ReviewInvoiceScreen.kt        # Mostrar datos extraídos
│   │   ├── MatchSupplierDialog.kt        # Búsqueda y selección
│   │   ├── MatchProductDialog.kt         # Edición de líneas
│   │   ├── EditPriceDialog.kt
│   │   └── ReviewViewModel.kt
│   ├── dashboard/
│   │   ├── DashboardScreen.kt            # Historial, métricas
│   │   └── DashboardViewModel.kt
│   └── common/
│       ├── components/
│       ├── navigation/
│       └── utils/
│
└── infrastructure/                       # INFRAESTRUCTURA
    ├── di/                               # Inyección de dependencias (Hilt)
    │   ├── RepositoryModule.kt
    │   ├── ServiceModule.kt
    │   ├── PosAdapterModule.kt
    │   └── AppModule.kt
    │
    ├── logging/                          # Observabilidad
    │   ├── TraceContext.kt               # ThreadLocal para correlacionar logs
    │   ├── LoggerFactory.kt
    │   └── TraceInterceptor.kt
    │
    ├── metrics/                          # Telemetría y SLA
    │   ├── MetricsCollector.kt
    │   ├── DocumentMetrics.kt            # Por documento: latencia, estado
    │   └── ErrorReporter.kt              # Reportar a backend
    │
    ├── config/
    │   ├── AppConfig.kt                  # POS type, connection strings
    │   ├── FeatureFlags.kt               # Canary deploy, debug modes
    │   └── AppConfigProvider.kt
    │
    └── connectivity/
        └── ConnectivityManager.kt        # Detectar cambios de internet