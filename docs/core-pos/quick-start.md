# Quick Start Guide

Get up and running with the CorePOS SDK in minutes. This guide covers the essential operations you'll need to integrate POS functionality into your app.

## Basic Setup

First, ensure you have the SDK installed and configured as described in the [Installation Guide](installation.md).

## Core Concepts

The CorePOS SDK uses **connectors** to communicate with different POS services:

- `InventoryConnector`: Manage products and categories
- `OrderConnector`: Handle orders and line items
- `MerchantConnector`: Access store information
- `TenderConnector`: Manage payment methods
- `PrinterConnector`: Print receipts

## Quick Examples

### 1. Get Merchant Information

```kotlin
class MainActivity : AppCompatActivity() {
    private lateinit var merchantConnector: MerchantConnector
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        
        merchantConnector = MerchantConnector(this)
        
        // Get merchant info in background thread
        lifecycleScope.launch(Dispatchers.IO) {
            try {
                val merchant = merchantConnector.getMerchant()
                merchant?.let {
                    Log.d("CorePOS", "Store: ${it.name}")
                    Log.d("CorePOS", "Address: ${it.address1}")
                }
            } catch (e: Exception) {
                Log.e("CorePOS", "Failed to get merchant: ${e.message}")
            }
        }
    }
}
```

### 2. Retrieve Inventory Items

```kotlin
class InventoryActivity : AppCompatActivity() {
    private lateinit var inventoryConnector: InventoryConnector
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_inventory)
        
        inventoryConnector = InventoryConnector(this)
        loadItems()
    }
    
    private fun loadItems() {
        lifecycleScope.launch(Dispatchers.IO) {
            try {
                // Get all items
                val items = inventoryConnector.getItems()
                
                // Or filter by category
                val filter = ItemFilter(categoryId = "electronics")
                val filteredItems = inventoryConnector.getItems(filter)
                
                // Update UI on main thread
                withContext(Dispatchers.Main) {
                    updateItemList(items ?: emptyList())
                }
            } catch (e: Exception) {
                Log.e("CorePOS", "Failed to load items: ${e.message}")
            }
        }
    }
}
```

### 3. Manage Orders

```kotlin
class OrderActivity : AppCompatActivity() {
    private lateinit var orderConnector: OrderConnector
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_order)
        
        orderConnector = OrderConnector(this)
        loadActiveOrder()
    }
    
    private fun loadActiveOrder() {
        lifecycleScope.launch(Dispatchers.IO) {
            try {
                val order = orderConnector.getActiveOrder()
                withContext(Dispatchers.Main) {
                    displayOrder(order)
                }
            } catch (e: Exception) {
                Log.e("CorePOS", "Failed to load order: ${e.message}")
            }
        }
    }
    
    fun addItemToOrder(itemId: String, quantity: Double) {
        lifecycleScope.launch(Dispatchers.IO) {
            try {
                val activeOrder = orderConnector.getActiveOrder()
                activeOrder?.let { order ->
                    val lineItem = orderConnector.addPerUnitLineItem(
                        orderId = order.orderId!!,
                        itemId = itemId,
                        quantity = quantity,
                        devNotes = null
                    )
                    
                    withContext(Dispatchers.Main) {
                        onItemAdded(lineItem)
                    }
                }
            } catch (e: Exception) {
                Log.e("CorePOS", "Failed to add item: ${e.message}")
            }
        }
    }
}
```

### 4. Handle Payments

```kotlin
class PaymentActivity : AppCompatActivity() {
    private lateinit var tenderConnector: TenderConnector
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_payment)
        
        tenderConnector = TenderConnector(this)
        loadPaymentMethods()
    }
    
    private fun loadPaymentMethods() {
        lifecycleScope.launch(Dispatchers.IO) {
            try {
                val tenders = tenderConnector.getTenders(packageName)
                withContext(Dispatchers.Main) {
                    displayPaymentMethods(tenders ?: emptyList())
                }
            } catch (e: Exception) {
                Log.e("CorePOS", "Failed to load payment methods: ${e.message}")
            }
        }
    }
    
    fun createPaymentMethod(
        buttonTitle: String,
        tenderName: String,
        enabled: Boolean = true
    ) {
        lifecycleScope.launch(Dispatchers.IO) {
            try {
                val tender = tenderConnector.createTender(
                    buttonTitle = buttonTitle,
                    tenderName = tenderName,
                    packageName = packageName,
                    enabled = enabled,
                    openCashDrawer = false
                )
                
                withContext(Dispatchers.Main) {
                    onPaymentMethodCreated(tender)
                }
            } catch (e: Exception) {
                Log.e("CorePOS", "Failed to create payment method: ${e.message}")
            }
        }
    }
}
```

### 5. Print Receipts

```kotlin
class ReceiptActivity : AppCompatActivity() {
    private lateinit var printerConnector: PrinterConnector
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_receipt)
        
        printerConnector = PrinterConnector(this)
    }
    
    fun printReceipt(bitmap: Bitmap) {
        lifecycleScope.launch(Dispatchers.IO) {
            try {
                printerConnector.printBitmap(bitmap)
                withContext(Dispatchers.Main) {
                    showPrintSuccess()
                }
            } catch (e: Exception) {
                Log.e("CorePOS", "Failed to print: ${e.message}")
                withContext(Dispatchers.Main) {
                    showPrintError(e.message)
                }
            }
        }
    }
}
```

## Important Notes

### Threading

All SDK operations **must** be called from a background thread. The SDK will throw an `IllegalStateException` if called from the main thread.

```kotlin
// ✅ Correct - Background thread
lifecycleScope.launch(Dispatchers.IO) {
    val items = inventoryConnector.getItems()
}

// ❌ Wrong - Main thread
val items = inventoryConnector.getItems() // This will crash!
```

### Error Handling

Always wrap SDK calls in try-catch blocks:

```kotlin
try {
    val result = connector.someMethod()
    // Handle success
} catch (e: PermissionDeniedException) {
    // Handle permission issues
} catch (e: BindingException) {
    // Handle service connection issues
} catch (e: Exception) {
    // Handle other errors
}
```

### Lifecycle Management

Store connectors as class properties and manage their lifecycle:

```kotlin
class MyActivity : AppCompatActivity() {
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

## Next Steps

- [API Reference](../api-reference/) - Detailed API documentation
- [Error Handling Guide](../guides/error-handling.md) - Comprehensive error handling
- [Examples](../examples/) - Complete working examples
- [Best Practices](../guides/best-practices.md) - Development guidelines 