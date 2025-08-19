# Common Issues and Troubleshooting

This guide helps you resolve common issues when integrating the CorePOS SDK into your application.

## Connection Issues

### Issue: "Cannot call service methods from the main thread"

**Symptoms:**
- App crashes with `IllegalStateException`
- Error message: "Cannot call service methods from the main thread"

**Cause:**
SDK methods are being called from the main/UI thread.

**Solution:**
Always call SDK methods from a background thread:

```kotlin
// ❌ Wrong - Called from main thread
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    val items = inventoryConnector.getItems() // This will crash!
}

// ✅ Correct - Called from background thread
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    lifecycleScope.launch(Dispatchers.IO) {
        val items = inventoryConnector.getItems()
    }
}
```

### Issue: "Could not bind to Android service"

**Symptoms:**
- `BindingException` thrown
- Error message: "Could not bind to Android service"

**Causes:**
1. CorePOS app not installed
2. CorePOS service not running
3. Incorrect package name configuration
4. Network connectivity issues

**Solutions:**

1. **Check if CorePOS app is installed:**
```kotlin
private fun isCorePOSInstalled(): Boolean {
    return try {
        packageManager.getPackageInfo("com.csiworks.corepos", 0)
        true
    } catch (e: PackageManager.NameNotFoundException) {
        false
    }
}
```

2. **Verify package name configuration:**
```kotlin
// Check your build.gradle configuration
android {
    productFlavors {
        prod {
            buildConfigField "String", "COREPOS_PACKAGE", "\"com.csiworks.corepos\""
        }
        sandbox {
            buildConfigField "String", "COREPOS_PACKAGE", "\"com.csiworks.corepos.sandbox\""
        }
        dev {
            buildConfigField "String", "COREPOS_PACKAGE", "\"com.csiworks.corepos.dev\""
        }
    }
}
```

3. **Implement retry logic:**
```kotlin
private suspend fun connectWithRetry(maxAttempts: Int = 3): Boolean {
    repeat(maxAttempts) { attempt ->
        try {
            val merchant = merchantConnector.getMerchant()
            return true
        } catch (e: BindingException) {
            if (attempt == maxAttempts - 1) throw e
            delay(1000L * (attempt + 1)) // Exponential backoff
        }
    }
    return false
}
```

### Issue: "Permission denied" errors

**Symptoms:**
- `PermissionDeniedException` thrown
- Error message: "Permission denied"

**Causes:**
1. Missing `BIND_SERVICE` permission
2. User denied permissions
3. App not authorized to access CorePOS

**Solutions:**

1. **Add required permissions to AndroidManifest.xml:**
```xml
<uses-permission android:name="android.permission.BIND_SERVICE" />
<uses-permission android:name="android.permission.INTERNET" />
```

2. **Request permissions at runtime:**
```kotlin
private fun requestPermissions() {
    if (ContextCompat.checkSelfPermission(this, 
        Manifest.permission.BIND_SERVICE) != PackageManager.PERMISSION_GRANTED) {
        ActivityCompat.requestPermissions(this, 
            arrayOf(Manifest.permission.BIND_SERVICE), PERMISSION_REQUEST_CODE)
    }
}

override fun onRequestPermissionsResult(
    requestCode: Int,
    permissions: Array<out String>,
    grantResults: IntArray
) {
    super.onRequestPermissionsResult(requestCode, permissions, grantResults)
    if (requestCode == PERMISSION_REQUEST_CODE) {
        if (grantResults.isNotEmpty() && grantResults[0] == PackageManager.PERMISSION_GRANTED) {
            // Permission granted, proceed with SDK operations
            initializeCorePOS()
        } else {
            // Permission denied, show explanation
            showPermissionExplanationDialog()
        }
    }
}
```

## Data Issues

### Issue: Invalid price type errors

**Symptoms:**
- `IllegalArgumentException` thrown
- Error message: "Invalid price type"

**Cause:**
Invalid price type value passed to SDK methods.

**Solution:**
Use the `PriceType` enum to ensure valid values:

```kotlin
// ❌ Wrong - Using raw integer
val item = Item(
    name = "Product",
    priceType = 5, // Invalid value
    // ... other properties
)

// ✅ Correct - Using enum
val item = Item(
    name = "Product",
    priceType = PriceType.FIXED.code, // Valid value (0)
    // ... other properties
)

// Or validate before creating
val priceType = when (userInput) {
    "fixed" -> PriceType.FIXED.code
    "variable" -> PriceType.VARIABLE.code
    "per_unit" -> PriceType.PER_UNIT.code
    else -> throw IllegalArgumentException("Invalid price type: $userInput")
}
```

### Issue: Null or empty data returned

**Symptoms:**
- Methods return `null` or empty lists
- No error thrown but no data received

**Causes:**
1. No data available
2. Filter criteria too restrictive
3. Service connection issues

**Solutions:**

1. **Check for empty results:**
```kotlin
val items = inventoryConnector.getItems()
if (items.isNullOrEmpty()) {
    // Handle empty state
    showEmptyState()
} else {
    // Process items
    displayItems(items)
}
```

2. **Verify filter criteria:**
```kotlin
// Check if category exists before filtering
val categories = inventoryConnector.getCategories()
val targetCategory = categories?.find { it.name == "Electronics" }

if (targetCategory != null) {
    val filter = ItemFilter(categoryId = targetCategory.categoryId)
    val items = inventoryConnector.getItems(filter)
} else {
    // Category not found, show error
    showError("Category not found")
}
```

3. **Implement fallback logic:**
```kotlin
private suspend fun getItemsWithFallback(filter: ItemFilter?): List<Item> {
    return try {
        inventoryConnector.getItems(filter) ?: emptyList()
    } catch (e: Exception) {
        Log.e("Inventory", "Failed to get items with filter: ${e.message}")
        // Fallback to all items
        inventoryConnector.getItems() ?: emptyList()
    }
}
```

## Performance Issues

### Issue: Slow response times

**Symptoms:**
- SDK operations take a long time
- UI becomes unresponsive
- Timeout errors

**Causes:**
1. Network connectivity issues
2. Large data sets
3. Service overload

**Solutions:**

1. **Implement caching:**
```kotlin
class CachedInventoryManager(private val context: Context) {
    private val cache = mutableMapOf<String, List<Item>>()
    private var lastUpdate = 0L
    private val cacheTimeout = 5 * 60 * 1000L // 5 minutes
    
    suspend fun getItems(filter: ItemFilter?): List<Item> {
        val cacheKey = filter?.categoryId ?: "all"
        val now = System.currentTimeMillis()
        
        // Return cached data if still valid
        if (now - lastUpdate < cacheTimeout && cache.containsKey(cacheKey)) {
            return cache[cacheKey]!!
        }
        
        // Fetch fresh data
        val items = inventoryConnector.getItems(filter) ?: emptyList()
        cache[cacheKey] = items
        lastUpdate = now
        return items
    }
}
```

2. **Add loading indicators:**
```kotlin
class InventoryViewModel : ViewModel() {
    private val _loading = MutableLiveData<Boolean>()
    val loading: LiveData<Boolean> = _loading
    
    fun loadItems() {
        viewModelScope.launch {
            _loading.value = true
            try {
                val items = inventoryConnector.getItems()
                _items.value = items
            } finally {
                _loading.value = false
            }
        }
    }
}
```

3. **Implement pagination for large datasets:**
```kotlin
// If your SDK supports pagination
data class PaginationParams(
    val offset: Int = 0,
    val limit: Int = 50
)

suspend fun getItemsPaginated(params: PaginationParams): List<Item> {
    // Implement pagination logic
    return inventoryConnector.getItems()
}
```

## Memory Issues

### Issue: Memory leaks from connectors

**Symptoms:**
- App memory usage increases over time
- OutOfMemoryError exceptions
- App becomes slow

**Causes:**
1. Connectors not properly disconnected
2. Large data objects not released
3. Background operations not cancelled

**Solutions:**

1. **Proper lifecycle management:**
```kotlin
class MainActivity : AppCompatActivity() {
    private lateinit var inventoryConnector: InventoryConnector
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        inventoryConnector = InventoryConnector(this)
    }
    
    override fun onDestroy() {
        super.onDestroy()
        inventoryConnector.disconnect()
    }
}
```

2. **Cancel background operations:**
```kotlin
class InventoryViewModel : ViewModel() {
    fun loadItems() {
        viewModelScope.launch {
            try {
                val items = inventoryConnector.getItems()
                _items.value = items
            } catch (e: CancellationException) {
                // Operation was cancelled, clean up
                Log.d("Inventory", "Operation cancelled")
            }
        }
    }
}
```

3. **Use weak references for callbacks:**
```kotlin
class InventoryManager(context: Context) {
    private val weakContext = WeakReference(context)
    
    fun loadItems(callback: (List<Item>) -> Unit) {
        lifecycleScope.launch(Dispatchers.IO) {
            val items = inventoryConnector.getItems() ?: emptyList()
            weakContext.get()?.let { context ->
                withContext(Dispatchers.Main) {
                    callback(items)
                }
            }
        }
    }
}
```

## Debugging Tips

### 1. Enable Debug Logging

Add debug logging to track SDK operations:

```kotlin
object CorePOSDebug {
    private const val TAG = "CorePOS"
    private var isDebugEnabled = BuildConfig.DEBUG
    
    fun log(message: String) {
        if (isDebugEnabled) {
            Log.d(TAG, message)
        }
    }
    
    fun logError(message: String, throwable: Throwable? = null) {
        if (isDebugEnabled) {
            Log.e(TAG, message, throwable)
        }
    }
    
    fun enableDebug(enabled: Boolean) {
        isDebugEnabled = enabled
    }
}
```

### 2. Check Service Status

Create a utility to check CorePOS service status:

```kotlin
object CorePOSStatusChecker {
    fun checkServiceStatus(context: Context): ServiceStatus {
        return try {
            // Try to get merchant info as a health check
            val merchantConnector = MerchantConnector(context)
            val merchant = merchantConnector.getMerchant()
            if (merchant != null) {
                ServiceStatus.Connected
            } else {
                ServiceStatus.NoData
            }
        } catch (e: BindingException) {
            ServiceStatus.NotConnected
        } catch (e: PermissionDeniedException) {
            ServiceStatus.NoPermission
        } catch (e: Exception) {
            ServiceStatus.Unknown(e.message)
        }
    }
}

sealed class ServiceStatus {
    object Connected : ServiceStatus()
    object NotConnected : ServiceStatus()
    object NoPermission : ServiceStatus()
    object NoData : ServiceStatus()
    data class Unknown(val message: String?) : ServiceStatus()
}
```

### 3. Network Diagnostics

Add network diagnostics for connection issues:

```kotlin
object NetworkDiagnostics {
    fun checkConnectivity(context: Context): Boolean {
        val connectivityManager = context.getSystemService(Context.CONNECTIVITY_SERVICE) as ConnectivityManager
        val network = connectivityManager.activeNetwork
        val capabilities = connectivityManager.getNetworkCapabilities(network)
        return capabilities != null && (
            capabilities.hasTransport(NetworkCapabilities.TRANSPORT_WIFI) ||
            capabilities.hasTransport(NetworkCapabilities.TRANSPORT_CELLULAR)
        )
    }
    
    fun logNetworkInfo(context: Context) {
        val connectivityManager = context.getSystemService(Context.CONNECTIVITY_SERVICE) as ConnectivityManager
        val network = connectivityManager.activeNetwork
        val capabilities = connectivityManager.getNetworkCapabilities(network)
        
        Log.d("NetworkDiagnostics", "Network: $network")
        Log.d("NetworkDiagnostics", "Capabilities: $capabilities")
    }
}
```

## Getting Help

If you're still experiencing issues:

1. **Check the logs:** Look for error messages in Android Studio's Logcat
2. **Verify setup:** Ensure all installation steps were followed correctly
3. **Test with minimal example:** Create a simple test app to isolate the issue
4. **Check CorePOS app:** Ensure the CorePOS application is running and up to date
5. **Contact support:** Provide detailed error logs and reproduction steps

### Support Information

When contacting support, include:
- SDK version: `0.1.4-RC1`
- Android version and device information
- Complete error logs
- Steps to reproduce the issue
- Code snippets showing the problem

## Related Documentation

- [Error Handling Guide](../guides/error-handling.md)
- [Installation Guide](../getting-started/installation.md)
- [API Reference](../api-reference/)
- [Best Practices](../guides/best-practices.md) 