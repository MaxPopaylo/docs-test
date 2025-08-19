# Best Practices Guide

This guide provides comprehensive best practices for developing robust applications with the CorePOS SDK.

## Architecture Guidelines

### 1. Use Repository Pattern

Implement a repository pattern to abstract SDK operations:

```kotlin
interface InventoryRepository {
    suspend fun getItems(filter: ItemFilter? = null): List<Item>
    suspend fun getItem(itemId: String): Item?
    suspend fun saveItem(item: Item, imageUri: String?): Item?
    suspend fun deleteItem(itemId: String): Boolean
}

class InventoryRepositoryImpl(
    private val context: Context
) : InventoryRepository {
    private val inventoryConnector = InventoryConnector(context)
    
    override suspend fun getItems(filter: ItemFilter?): List<Item> {
        return try {
            inventoryConnector.getItems(filter) ?: emptyList()
        } catch (e: Exception) {
            Log.e("InventoryRepo", "Failed to get items: ${e.message}")
            emptyList()
        }
    }
    
    override suspend fun getItem(itemId: String): Item? {
        return try {
            inventoryConnector.getItem(itemId)
        } catch (e: Exception) {
            Log.e("InventoryRepo", "Failed to get item: ${e.message}")
            null
        }
    }
    
    override suspend fun saveItem(item: Item, imageUri: String?): Item? {
        return try {
            inventoryConnector.saveItem(item, imageUri)
        } catch (e: Exception) {
            Log.e("InventoryRepo", "Failed to save item: ${e.message}")
            null
        }
    }
    
    override suspend fun deleteItem(itemId: String): Boolean {
        return try {
            inventoryConnector.deleteItem(itemId)
            true
        } catch (e: Exception) {
            Log.e("InventoryRepo", "Failed to delete item: ${e.message}")
            false
        }
    }
}
```

### 2. Implement Dependency Injection

Use dependency injection to manage SDK dependencies:

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object CorePOSModule {
    
    @Provides
    @Singleton
    fun provideInventoryConnector(@ApplicationContext context: Context): InventoryConnector {
        return InventoryConnector(context)
    }
    
    @Provides
    @Singleton
    fun provideOrderConnector(@ApplicationContext context: Context): OrderConnector {
        return OrderConnector(context)
    }
    
    @Provides
    @Singleton
    fun provideMerchantConnector(@ApplicationContext context: Context): MerchantConnector {
        return MerchantConnector(context)
    }
    
    @Provides
    @Singleton
    fun provideInventoryRepository(
        inventoryConnector: InventoryConnector
    ): InventoryRepository {
        return InventoryRepositoryImpl(inventoryConnector)
    }
}
```

### 3. Use ViewModels for Business Logic

Keep business logic in ViewModels and use LiveData for reactive UI updates:

```kotlin
class InventoryViewModel @Inject constructor(
    private val inventoryRepository: InventoryRepository
) : ViewModel() {
    
    private val _items = MutableLiveData<List<Item>>()
    val items: LiveData<List<Item>> = _items
    
    private val _loading = MutableLiveData<Boolean>()
    val loading: LiveData<Boolean> = _loading
    
    private val _error = MutableLiveData<String?>()
    val error: LiveData<String?> = _error
    
    fun loadItems(filter: ItemFilter? = null) {
        viewModelScope.launch {
            _loading.value = true
            _error.value = null
            
            try {
                val items = inventoryRepository.getItems(filter)
                _items.value = items
            } catch (e: Exception) {
                _error.value = e.message
            } finally {
                _loading.value = false
            }
        }
    }
    
    fun createItem(name: String, price: Long, categoryId: String?) {
        viewModelScope.launch {
            try {
                val item = Item(
                    name = name,
                    priceType = PriceType.FIXED.code,
                    unitCash = price,
                    unitCard = price,
                    unitType = "piece",
                    charges = emptyList(),
                    categories = categoryId?.let { listOf(Category(it, null)) },
                    productCode = null,
                    itemCost = null,
                    quantity = 0,
                    trackInventory = false,
                    dualPricingBasePriceType = PriceType.FIXED.code,
                    isEBT = false,
                    isAvailable = true
                )
                
                val savedItem = inventoryRepository.saveItem(item, null)
                if (savedItem != null) {
                    loadItems() // Refresh the list
                }
            } catch (e: Exception) {
                _error.value = e.message
            }
        }
    }
}
```

## Performance Optimization

### 1. Connection Management

Implement proper connection lifecycle management:

```kotlin
class CorePOSManager @Inject constructor(
    private val context: Context
) {
    private var inventoryConnector: InventoryConnector? = null
    private var orderConnector: OrderConnector? = null
    private var merchantConnector: MerchantConnector? = null
    
    fun initialize() {
        inventoryConnector = InventoryConnector(context)
        orderConnector = OrderConnector(context)
        merchantConnector = MerchantConnector(context)
    }
    
    fun getInventoryConnector(): InventoryConnector {
        return inventoryConnector ?: throw IllegalStateException("CorePOS not initialized")
    }
    
    fun getOrderConnector(): OrderConnector {
        return orderConnector ?: throw IllegalStateException("CorePOS not initialized")
    }
    
    fun getMerchantConnector(): MerchantConnector {
        return merchantConnector ?: throw IllegalStateException("CorePOS not initialized")
    }
    
    fun cleanup() {
        inventoryConnector?.disconnect()
        orderConnector?.disconnect()
        merchantConnector?.disconnect()
        
        inventoryConnector = null
        orderConnector = null
        merchantConnector = null
    }
}
```

### 2. Caching Strategy

Implement caching to reduce API calls:

```kotlin
class CachedInventoryRepository @Inject constructor(
    private val inventoryConnector: InventoryConnector
) : InventoryRepository {
    
    private val cache = mutableMapOf<String, Item>()
    private val itemsCache = mutableListOf<Item>()
    private var lastCacheTime = 0L
    private val cacheValidityDuration = 5 * 60 * 1000L // 5 minutes
    
    override suspend fun getItems(filter: ItemFilter?): List<Item> {
        val now = System.currentTimeMillis()
        
        // Return cached data if still valid
        if (now - lastCacheTime < cacheValidityDuration && itemsCache.isNotEmpty()) {
            return if (filter != null) {
                itemsCache.filter { item ->
                    (filter.categoryId == null || 
                     item.categories?.any { it.categoryId == filter.categoryId } == true) &&
                    (filter.productCode == null || item.productCode == filter.productCode)
                }
            } else {
                itemsCache
            }
        }
        
        // Fetch fresh data
        return try {
            val items = inventoryConnector.getItems(filter) ?: emptyList()
            itemsCache.clear()
            itemsCache.addAll(items)
            lastCacheTime = now
            items
        } catch (e: Exception) {
            Log.e("CachedRepo", "Failed to get items: ${e.message}")
            itemsCache // Return cached data as fallback
        }
    }
    
    override suspend fun getItem(itemId: String): Item? {
        // Check cache first
        cache[itemId]?.let { return it }
        
        // Fetch from API
        return try {
            val item = inventoryConnector.getItem(itemId)
            item?.let { cache[itemId] = it }
            item
        } catch (e: Exception) {
            Log.e("CachedRepo", "Failed to get item: ${e.message}")
            null
        }
    }
    
    fun invalidateCache() {
        cache.clear()
        itemsCache.clear()
        lastCacheTime = 0L
    }
}
```

### 3. Background Processing

Use WorkManager for background operations:

```kotlin
class InventorySyncWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {
    
    @Inject
    lateinit var inventoryRepository: InventoryRepository
    
    override suspend fun doWork(): Result {
        return try {
            // Sync inventory data
            val items = inventoryRepository.getItems()
            
            // Update local database
            updateLocalDatabase(items)
            
            Result.success()
        } catch (e: Exception) {
            Log.e("SyncWorker", "Failed to sync inventory: ${e.message}")
            Result.retry()
        }
    }
    
    private suspend fun updateLocalDatabase(items: List<Item>) {
        // Update local database with inventory data
    }
}

// Schedule periodic sync
fun scheduleInventorySync() {
    val constraints = Constraints.Builder()
        .setRequiredNetworkType(NetworkType.CONNECTED)
        .build()
    
    val syncRequest = PeriodicWorkRequestBuilder<InventorySyncWorker>(
        15, TimeUnit.MINUTES
    ).setConstraints(constraints)
        .build()
    
    WorkManager.getInstance(context).enqueueUniquePeriodicWork(
        "inventory_sync",
        ExistingPeriodicWorkPolicy.KEEP,
        syncRequest
    )
}
```

## Error Handling Best Practices

### 1. Centralized Error Handling

Create a centralized error handling system:

```kotlin
sealed class CorePOSResult<out T> {
    data class Success<T>(val data: T) : CorePOSResult<T>()
    data class Error(val exception: Throwable) : CorePOSResult<Nothing>()
}

class CorePOSErrorHandler {
    companion object {
        fun handleError(throwable: Throwable, context: Context) {
            when (throwable) {
                is PermissionDeniedException -> {
                    showPermissionDialog(context)
                }
                is BindingException -> {
                    showConnectionDialog(context)
                }
                is IllegalArgumentException -> {
                    showValidationDialog(context, throwable.message)
                }
                else -> {
                    Log.e("CorePOS", "Unexpected error: ${throwable.message}")
                    showGenericError(context)
                }
            }
        }
        
        private fun showPermissionDialog(context: Context) {
            // Show permission request dialog
        }
        
        private fun showConnectionDialog(context: Context) {
            // Show connection error dialog
        }
        
        private fun showValidationDialog(context: Context, message: String?) {
            // Show validation error dialog
        }
        
        private fun showGenericError(context: Context) {
            // Show generic error dialog
        }
    }
}
```

### 2. Retry Logic

Implement robust retry logic:

```kotlin
class RetryManager {
    suspend fun <T> retry(
        maxAttempts: Int = 3,
        initialDelay: Long = 1000L,
        maxDelay: Long = 10000L,
        factor: Double = 2.0,
        block: suspend () -> T
    ): T {
        var currentDelay = initialDelay
        repeat(maxAttempts) { attempt ->
            try {
                return block()
            } catch (e: Exception) {
                if (attempt == maxAttempts - 1) throw e
                
                if (e is BindingException || e is PermissionDeniedException) {
                    delay(currentDelay)
                    currentDelay = (currentDelay * factor).toLong().coerceAtMost(maxDelay)
                } else {
                    throw e // Don't retry for non-transient errors
                }
            }
        }
        throw IllegalStateException("Retry exhausted")
    }
}
```

## Security Best Practices

### 1. Input Validation

Always validate input before passing to SDK:

```kotlin
object InputValidator {
    fun validateItem(item: Item): ValidationResult {
        return when {
            item.name.isBlank() -> ValidationResult.Error("Item name is required")
            item.unitCash != null && item.unitCash < 0 -> ValidationResult.Error("Price cannot be negative")
            item.priceType !in 0..2 -> ValidationResult.Error("Invalid price type")
            else -> ValidationResult.Success
        }
    }
    
    fun validateOrderParameters(orderId: String?, itemId: String?, quantity: Double): ValidationResult {
        return when {
            orderId.isNullOrBlank() -> ValidationResult.Error("Order ID is required")
            itemId.isNullOrBlank() -> ValidationResult.Error("Item ID is required")
            quantity <= 0 -> ValidationResult.Error("Quantity must be positive")
            else -> ValidationResult.Success
        }
    }
}

sealed class ValidationResult {
    object Success : ValidationResult()
    data class Error(val message: String) : ValidationResult()
}
```

### 2. Secure Data Handling

Implement secure data handling practices:

```kotlin
class SecureDataManager {
    fun sanitizeItemData(item: Item): Item {
        return item.copy(
            name = item.name.trim(),
            productCode = item.productCode?.trim(),
            unitType = item.unitType?.trim()
        )
    }
    
    fun validateImageUri(uri: String?): Boolean {
        if (uri.isNullOrBlank()) return true
        
        return try {
            val parsedUri = Uri.parse(uri)
            parsedUri.scheme in listOf("content", "file")
        } catch (e: Exception) {
            false
        }
    }
}
```

## Testing Best Practices

### 1. Unit Testing

Create comprehensive unit tests:

```kotlin
@RunWith(MockitoJUnitRunner::class)
class InventoryViewModelTest {
    
    @Mock
    private lateinit var inventoryRepository: InventoryRepository
    
    @Mock
    private lateinit var errorHandler: CorePOSErrorHandler
    
    private lateinit var viewModel: InventoryViewModel
    
    @Before
    fun setup() {
        viewModel = InventoryViewModel(inventoryRepository)
    }
    
    @Test
    fun `loadItems should update LiveData with items`() = runTest {
        // Given
        val items = listOf(
            Item(name = "Test Item", priceType = PriceType.FIXED.code, /* ... */)
        )
        whenever(inventoryRepository.getItems(any())).thenReturn(items)
        
        // When
        viewModel.loadItems()
        
        // Then
        assertEquals(items, viewModel.items.value)
        assertFalse(viewModel.loading.value!!)
        assertNull(viewModel.error.value)
    }
    
    @Test
    fun `loadItems should handle errors`() = runTest {
        // Given
        val error = Exception("Network error")
        whenever(inventoryRepository.getItems(any())).thenThrow(error)
        
        // When
        viewModel.loadItems()
        
        // Then
        assertEquals(error.message, viewModel.error.value)
        assertFalse(viewModel.loading.value!!)
    }
}
```

### 2. Integration Testing

Test SDK integration:

```kotlin
@RunWith(AndroidJUnit4::class)
class CorePOSIntegrationTest {
    
    @get:Rule
    val instantExecutorRule = InstantTaskExecutorRule()
    
    @Test
    fun testInventoryOperations() = runTest {
        // Given
        val context = ApplicationProvider.getApplicationContext<Context>()
        val inventoryConnector = InventoryConnector(context)
        
        // When & Then
        try {
            val items = inventoryConnector.getItems()
            assertNotNull(items)
        } catch (e: Exception) {
            // Handle case where CorePOS app is not installed
            assertTrue(e is BindingException || e is PermissionDeniedException)
        }
    }
}
```

## Code Organization

### 1. Package Structure

Organize code into logical packages:

```
com.yourapp.corepos/
├── data/
│   ├── repository/
│   │   ├── InventoryRepository.kt
│   │   ├── OrderRepository.kt
│   │   └── MerchantRepository.kt
│   ├── model/
│   │   ├── Item.kt
│   │   ├── Order.kt
│   │   └── Merchant.kt
│   └── local/
│       └── CorePOSDatabase.kt
├── domain/
│   ├── usecase/
│   │   ├── GetItemsUseCase.kt
│   │   ├── CreateOrderUseCase.kt
│   │   └── ProcessPaymentUseCase.kt
│   └── model/
│       └── CorePOSResult.kt
├── presentation/
│   ├── viewmodel/
│   │   ├── InventoryViewModel.kt
│   │   └── OrderViewModel.kt
│   ├── ui/
│   │   ├── InventoryActivity.kt
│   │   └── OrderActivity.kt
│   └── adapter/
│       └── ItemAdapter.kt
└── di/
    └── CorePOSModule.kt
```

### 2. Naming Conventions

Follow consistent naming conventions:

```kotlin
// Connectors
val inventoryConnector: InventoryConnector
val orderConnector: OrderConnector
val merchantConnector: MerchantConnector

// Repositories
val inventoryRepository: InventoryRepository
val orderRepository: OrderRepository

// ViewModels
val inventoryViewModel: InventoryViewModel
val orderViewModel: OrderViewModel

// Use cases
val getItemsUseCase: GetItemsUseCase
val createOrderUseCase: CreateOrderUseCase

// Constants
const val CORE_POS_TAG = "CorePOS"
const val INVENTORY_SYNC_WORK_NAME = "inventory_sync"
```

## Performance Monitoring

### 1. Metrics Collection

Implement performance monitoring:

```kotlin
class CorePOSMetrics {
    private val metrics = mutableMapOf<String, Long>()
    
    fun startTimer(operation: String) {
        metrics[operation] = System.currentTimeMillis()
    }
    
    fun endTimer(operation: String): Long {
        val startTime = metrics[operation] ?: return 0L
        val duration = System.currentTimeMillis() - startTime
        metrics.remove(operation)
        
        Log.d("CorePOSMetrics", "$operation took ${duration}ms")
        return duration
    }
    
    fun logError(operation: String, error: Throwable) {
        Log.e("CorePOSMetrics", "Error in $operation: ${error.message}")
    }
}
```

### 2. Usage Analytics

Track SDK usage patterns:

```kotlin
class CorePOSAnalytics {
    fun trackOperation(operation: String, success: Boolean, duration: Long) {
        // Send analytics data
        Log.d("CorePOSAnalytics", "Operation: $operation, Success: $success, Duration: ${duration}ms")
    }
    
    fun trackError(operation: String, error: Throwable) {
        // Track error occurrences
        Log.e("CorePOSAnalytics", "Error in $operation: ${error.message}")
    }
}
```

## Conclusion

Following these best practices will help you build robust, maintainable, and performant applications with the CorePOS SDK. Remember to:

1. **Always handle errors gracefully**
2. **Use background threads for SDK operations**
3. **Implement proper lifecycle management**
4. **Cache data when appropriate**
5. **Validate input data**
6. **Write comprehensive tests**
7. **Monitor performance and errors**
8. **Follow consistent coding patterns**

## Related Documentation

- [Error Handling Guide](error-handling.md)
- [Quick Start Guide](../getting-started/quick-start.md)
- [API Reference](../api-reference/)
- [Troubleshooting](../troubleshooting/) 