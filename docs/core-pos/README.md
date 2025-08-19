# CorePOS SDK Documentation

Welcome to the CorePOS Android SDK documentation. This comprehensive guide will help you integrate POS functionality into your Android applications.

## 📚 Documentation Overview

### Getting Started
- **[Installation Guide](getting-started/installation.md)** - Set up the SDK in your project
- **[Quick Start Guide](getting-started/quick-start.md)** - Get up and running in minutes

### API Reference
- **[Inventory API](api-reference/inventory.md)** - Manage products and categories
- **[Order API](api-reference/orders.md)** - Handle orders and transactions *(Coming Soon)*
- **[Merchant API](api-reference/merchant.md)** - Access store information *(Coming Soon)*
- **[Tender API](api-reference/tender.md)** - Manage payment methods *(Coming Soon)*
- **[Printer API](api-reference/printer.md)** - Print receipts *(Coming Soon)*

### Guides
- **[Error Handling Guide](guides/error-handling.md)** - Comprehensive error handling
- **[Best Practices](guides/best-practices.md)** - Development guidelines and patterns

### Examples
- **[Basic Usage](examples/basic-usage/)** - Simple integration examples *(Coming Soon)*
- **[Advanced Features](examples/advanced-features/)** - Complex use cases *(Coming Soon)*

### Troubleshooting
- **[Common Issues](troubleshooting/common-issues.md)** - Solutions to frequent problems

## 🚀 Quick Navigation

### For New Users
1. Start with the [Installation Guide](getting-started/installation.md)
2. Follow the [Quick Start Guide](getting-started/quick-start.md)
3. Explore the [API Reference](api-reference/) for your specific needs

### For Experienced Developers
1. Review the [Best Practices](guides/best-practices.md)
2. Check the [Error Handling Guide](guides/error-handling.md)
3. Browse [Examples](examples/) for implementation patterns

### When You Need Help
1. Check [Common Issues](troubleshooting/common-issues.md)
2. Review the [Error Handling Guide](guides/error-handling.md)
3. Contact support with detailed error logs

## 📋 SDK Features

The CorePOS SDK provides comprehensive POS functionality:

### Core Features
- **Inventory Management** - Products, categories, pricing
- **Order Processing** - Orders, line items, transactions
- **Merchant Information** - Store details and configuration
- **Payment Handling** - Tender methods and cash drawer
- **Receipt Printing** - Print receipts and custom content

### Technical Features
- **AIDL Communication** - Reliable inter-process communication
- **Thread Safety** - Built-in background execution
- **Error Handling** - Comprehensive error management
- **Multiple Environments** - Production, sandbox, and development support

## 🔧 Requirements

- **Minimum SDK**: API 21 (Android 5.0)
- **Target SDK**: API 35
- **Java Version**: 11
- **Kotlin**: 1.9.24+
- **CorePOS App**: Must be installed on the device

## 📦 Installation

Add the SDK to your project:

```gradle
dependencies {
    implementation 'com.coreposnow:sdk:0.1.4-RC1'
}
```

## 🏗️ Architecture

The SDK uses a connector-based architecture:

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Your App      │    │  CorePOS SDK    │    │  CorePOS App    │
│                 │◄──►│                 │◄──►│                 │
│ - Activities    │    │ - Connectors    │    │ - Services      │
│ - ViewModels    │    │ - Data Models   │    │ - Business      │
│ - Repositories  │    │ - Error Handling│    │   Logic         │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

### Connectors
- `InventoryConnector` - Item and category management
- `OrderConnector` - Order and line item operations
- `MerchantConnector` - Merchant information access
- `TenderConnector` - Payment method management
- `PrinterConnector` - Receipt printing functionality

## 🔄 Basic Usage

```kotlin
// Initialize connectors
val inventoryConnector = InventoryConnector(context)
val orderConnector = OrderConnector(context)
val merchantConnector = MerchantConnector(context)

// Get merchant information
lifecycleScope.launch(Dispatchers.IO) {
    val merchant = merchantConnector.getMerchant()
    // Handle merchant data
}

// Retrieve inventory items
lifecycleScope.launch(Dispatchers.IO) {
    val items = inventoryConnector.getItems()
    // Handle items data
}

// Get active order
lifecycleScope.launch(Dispatchers.IO) {
    val order = orderConnector.getActiveOrder()
    // Handle order data
}
```

## ⚠️ Important Notes

### Threading
- **All SDK operations must be called from background threads**
- The SDK will throw `IllegalStateException` if called from the main thread
- Use `lifecycleScope.launch(Dispatchers.IO)` for SDK operations

### Error Handling
- Always wrap SDK calls in try-catch blocks
- Handle specific exceptions: `PermissionDeniedException`, `BindingException`, `IllegalArgumentException`
- Implement retry logic for transient errors

### Lifecycle Management
- Store connectors as class properties
- Call `disconnect()` when done to clean up resources
- Handle configuration changes properly

## 🆘 Support

### Documentation Issues
If you find issues with this documentation:
- Create an issue in the repository
- Suggest improvements via pull requests

### SDK Issues
For SDK-related problems:
- Check the [Troubleshooting Guide](troubleshooting/common-issues.md)
- Review the [Error Handling Guide](guides/error-handling.md)
- Contact support with:
  - SDK version: `0.1.4-RC1`
  - Android version and device info
  - Complete error logs
  - Steps to reproduce

### Community
- GitHub Issues: [Repository Issues](https://github.com/your-repo/issues)
- Developer Portal: [CorePOS Developer Portal](https://developer.corepos.com)
- Documentation: [CorePOS Docs](https://docs.corepos.com)

## 📄 License

[Add your license information here]

## 🔄 Version History

- **0.1.4-RC1** - Initial release candidate with core POS functionality

---

**Need help?** Start with the [Quick Start Guide](getting-started/quick-start.md) or check [Common Issues](troubleshooting/common-issues.md). 