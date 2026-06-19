# Punto 6: Capa de mapeo y matching — Explicación

## ¿Qué es la capa de mapeo y matching?

Es el servicio que **empareja** datos extraídos de la factura (strings sin procesar) contra el catálogo conocido del POS:

### Ejemplo real
```
Factura extraída:
  Proveedor: "Distrib. La Cosecha"
  Productos: ["ACEITE OLIVA OLIVETTO 250ML", "ARROZ DIANA 500G"]

POS Catálogo (base de datos):
  Suppliers:
    - ID: 174, Name: "Distribuidora La Cosecha SAS"
    - ID: 53, Name: "La Cosecha Mayorista"
  
  Products:
    - Code: 32308, Name: "OLIVETTO ACEITE OLIVA 250ML", SupplierId: 174
    - Code: 10455, Name: "DIANA ARROZ 500G", SupplierId: 174

Matching:
  "Distrib. La Cosecha" (factura) → 92% match → ID 174 "Distribuidora La Cosecha SAS"
  "ACEITE OLIVA OLIVETTO 250ML" (factura) → 95% match → Code 32308
```

---

## Estrategia: 3 niveles de matching

### Nivel 1: Búsqueda exacta
```
factura.proveedor.upper() == pos.supplier.name.upper()
```
- Rápido, sin red.
- Raramente funciona (variaciones de nombre).

### Nivel 2: Búsqueda fuzzy (aproximada)
```
Levenshtein distance, Jaccard similarity, etc.
"DISTRIB LA COSECHA" vs "DISTRIBUIDORA LA COSECHA SAS"
Score: 87%
```
- Local, sin dependencias externas.
- Funciona bien para variaciones menores.

### Nivel 3: Embeddings + Búsqueda semántica
```
"Distrib. La Cosecha" → Embedding (768 dims) via IA
Buscar nearest neighbor en índice de suppliers
Score: 92% (semánticamente similar)
```
- Requiere red/backend.
- Maneja sinónimos y variaciones grandes.

---

## Arquitectura

```
┌──────────────────────────────────┐
│   Datos extraídos (factura)      │
│  - proveedor: "Distrib. La..."   │
│  - productos: [...]              │
└────────────┬─────────────────────┘
             │
    ┌────────▼────────────┐
    │ MatchingService     │
    │ (Orquestador)       │
    └────────┬────────────┘
             │
    ┌────────┴──────────────────┬──────────┬─────────┐
    │                           │          │         │
┌───▼────┐          ┌──────────▼──┐   ┌───▼──┐  ┌──▼─────┐
│ Exact  │          │  Fuzzy      │   │Local │  │Remote  │
│Match   │◄─────────│  Matching   │   │Cache │  │Backend │
│        │          │  (Levensh.) │   │      │  │(IA)    │
└────────┘          └────────┬────┘   └──────┘  └────────┘
                             │
                    ┌────────▼─────────┐
                    │  Embeddings      │
                    │  (Semantic)      │
                    └──────────────────┘
                             ▲
                             │
                    (Si fuzzy < threshold)
```

---

## Implementación

### A) Modelos

```kotlin
// domain/model/MatchingResult.kt
data class MatchingResult(
    val matchType: MatchType,        // EXACT, FUZZY, SEMANTIC
    val matchedSupplierId: String,   // ID en POS
    val matchedProductCode: String,  // Código en POS
    val score: Float,                // 0.0 to 1.0
    val alternatives: List<Alternative> = emptyList()
)

data class Alternative(
    val entityId: String,
    val entityName: String,
    val score: Float
)

enum class MatchType {
    EXACT,       // 100% match
    FUZZY,       // 70-99% (string similarity)
    SEMANTIC,    // 60-99% (embeddings)
    NOT_FOUND    // No match found
}

// domain/model/Supplier.kt (simplificado)
data class Supplier(
    val id: String,
    val name: String,
    val aliases: List<String> = emptyList()  // "Distrib. La Cosecha", "La Cosecha Mayorista"
)

data class Product(
    val code: String,
    val name: String,
    val supplierId: String,
    val price: Float,
    val cost: Float
)
```

### B) Service de matching

```kotlin
// domain/service/MatchingEngine.kt
interface MatchingEngine {
    suspend fun matchSupplier(name: String): MatchingResult
    suspend fun matchProduct(name: String, supplierId: String? = null): MatchingResult
    suspend fun matchBatch(suppliers: List<String>, products: List<String>): List<MatchingResult>
}

// data/remote/services/MatchingService.kt (Implementación)
class MatchingService(
    private val posRepository: PosRepository,
    private val fuzzyMatcher: FuzzyMatcher,
    private val semanticMatcher: SemanticMatcher  // Llama backend si necesita
) : MatchingEngine {

    override suspend fun matchSupplier(name: String): MatchingResult {
        // 1. Búsqueda exacta
        val exactMatch = posRepository.getSupplierByName(name)
        if (exactMatch != null) {
            return MatchingResult(
                matchType = MatchType.EXACT,
                matchedSupplierId = exactMatch.id,
                score = 1.0f
            )
        }

        // 2. Búsqueda fuzzy
        val fuzzyMatches = posRepository.getAllSuppliers()
            .map { supplier ->
                val score = fuzzyMatcher.similarity(name, supplier.name)
                Pair(supplier, score)
            }
            .filter { it.second > 0.70f }  // >= 70%
            .sortedByDescending { it.second }

        if (fuzzyMatches.isNotEmpty()) {
            val best = fuzzyMatches.first()
            return MatchingResult(
                matchType = MatchType.FUZZY,
                matchedSupplierId = best.first.id,
                score = best.second,
                alternatives = fuzzyMatches.drop(1)
                    .map { Alternative(it.first.id, it.first.name, it.second) }
            )
        }

        // 3. Búsqueda semántica (backend)
        val semanticMatch = semanticMatcher.findMostSimilar(name)
        return semanticMatch ?: MatchingResult(
            matchType = MatchType.NOT_FOUND,
            matchedSupplierId = "",
            score = 0.0f
        )
    }

    override suspend fun matchProduct(name: String, supplierId: String?): MatchingResult {
        // Similar: exacto → fuzzy → semántico
        // Optimización: si tenemos supplierId, filtrar por eso primero
    }
}

// data/local/fuzzy/FuzzyMatcher.kt
class FuzzyMatcher {
    fun similarity(s1: String, s2: String): Float {
        // Levenshtein distance normalizado
        val maxLen = maxOf(s1.length, s2.length)
        if (maxLen == 0) return 1.0f
        val distance = levenshteinDistance(s1.uppercase(), s2.uppercase())
        return 1.0f - (distance.toFloat() / maxLen)
    }

    private fun levenshteinDistance(s1: String, s2: String): Int {
        val dp = Array(s1.length + 1) { IntArray(s2.length + 1) }
        for (i in 0..s1.length) dp[i][0] = i
        for (j in 0..s2.length) dp[0][j] = j
        for (i in 1..s1.length) {
            for (j in 1..s2.length) {
                dp[i][j] = minOf(
                    dp[i - 1][j] + 1,      // deletion
                    dp[i][j - 1] + 1,      // insertion
                    dp[i - 1][j - 1] + if (s1[i - 1] != s2[j - 1]) 1 else 0  // substitution
                )
            }
        }
        return dp[s1.length][s2.length]
    }
}

// data/remote/services/SemanticMatcher.kt (Llama backend)
class SemanticMatcher(private val apiService: ApiService) {
    suspend fun findMostSimilar(name: String): MatchingResult? {
        return try {
            val response = apiService.findSimilarSupplier(name)  // POST /match/semantic
            response.toMatchingResult()
        } catch (e: Exception) {
            null  // Fallback a no encontrado
        }
    }
}
```

### C) UseCase de matching

```kotlin
// domain/usecase/matching/MatchSupplierUseCase.kt
class MatchSupplierUseCase(
    private val matchingEngine: MatchingEngine
) {
    suspend fun execute(supplierName: String): MatchingResult {
        val result = matchingEngine.matchSupplier(supplierName)
        // Log, métricas, etc.
        return result
    }
}

// domain/usecase/matching/MatchProductUseCase.kt
class MatchProductUseCase(
    private val matchingEngine: MatchingEngine
) {
    suspend fun execute(productName: String, supplierId: String?): MatchingResult {
        return matchingEngine.matchProduct(productName, supplierId)
    }
}
```

### D) ViewModel de revisión (UI interactúa)

```kotlin
// presentation/review/ReviewViewModel.kt
class ReviewViewModel(
    private val matchSupplierUseCase: MatchSupplierUseCase,
    private val matchProductUseCase: MatchProductUseCase
) : ViewModel() {
    
    fun onSupplierFound(name: String) {
        viewModelScope.launch {
            val result = matchSupplierUseCase.execute(name)
            _uiState.value = _uiState.value.copy(
                supplierMatches = listOf(result),
                supplierScore = result.score,
                userAction = UserAction.SELECT_SUPPLIER  // Requiere confirmación
            )
        }
    }

    fun onProductFound(name: String, supplierId: String?) {
        viewModelScope.launch {
            val result = matchProductUseCase.execute(name, supplierId)
            // Similar...
        }
    }

    fun userSelectsSupplier(supplierId: String) {
        // Confirmar y avanzar
    }
}
```

---

## Thresholds sugeridos

| Match Type | Threshold | Acción |
|---|---|---|
| Exact (100%) | ≥ 100% | Aceptar automáticamente |
| Fuzzy | 85-99% | Sugerir, esperar confirmación |
| Fuzzy | 70-84% | Mostrar alternatives |
| Semantic | 75-99% | Sugerir (con disclaimer) |
| No match | < 70% | Requiere creación manual |

---

## Fases

### MVP
- Búsqueda exacta + fuzzy local.
- No embeddings (sin backend).
- Caché local de catálogo (actualizar cada mañana).

### Next
- Embeddings desde backend.
- Estadísticas de matches (qué proveedores son más difíciles).

### Later
- Fine-tuning de embeddings sobre datos reales.
- Autocompletar en campos de entrada.

---

Implementar en `domain/usecase/matching/` y `data/remote/services/MatchingService.kt`.
