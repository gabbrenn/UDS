# Product Creation Flow - Complete Explanation

## Overview
This document explains how the frontend (`adding_product_widget.dart`) communicates with the backend API to create or update products.

---

## 📋 Table of Contents
1. [User Interaction Flow](#user-interaction-flow)
2. [Data Collection & Validation](#data-collection--validation)
3. [Data Transformation](#data-transformation)
4. [API Request Structure](#api-request-structure)
5. [Request Body Format](#request-body-format)
6. [API Response Handling](#api-response-handling)
7. [Error Handling](#error-handling)
8. [Success Handling](#success-handling)

---

## 1. User Interaction Flow

### Step-by-Step Process:

1. **User fills the form** with product information:
   - Product Name (required)
   - Category (required)
   - Quantity (required)
   - Unit (required)
   - Price (required)
   - Purchase Price (optional)
   - Store (optional)
   - Description (optional)
   - Expiration Date (optional)
   - Product Image (optional)
   - **EBM Fields** (if `useEbmIntegration = true`):
     - Origin Country Code (orgnNatCd)
     - Package Unit Code (pkgUnitCd)
     - Tax Type Code (taxTyCd)
     - Product Type

2. **User clicks "Publish Now" button** (line 421-605)

3. **Validation happens** before API call

4. **API call is made** to create/update product

5. **Response is handled** - success or error message shown

---

## 2. Data Collection & Validation

### Location: Lines 422-452

```dart
// First validation: Required fields
if (_model.textController1.text.isEmpty ||      // Product name
    _model.dropDownValue1 == null ||            // Category
    _model.textController2.text.isEmpty ||      // Quantity
    _model.dropDownValue2 == null ||            // Unit
    _model.textController3.text.isEmpty) {      // Price
  // Show error: "Please fill in all required fields"
  return; // Stop execution
}

// Second validation: EBM fields (if EBM integration enabled)
if (_model.useEbmIntegration) {
  if (_model.dropDownValue4 == null ||          // orgnNatCd
      _model.dropDownValue5 == null ||          // pkgUnitCd
      _model.dropDownValue6 == null ||          // taxTyCd
      _model.dropDownValue7 == null) {           // productType
    // Show error: "Please fill in all EBM integration fields"
    return; // Stop execution
  }
}
```

### Validation Rules:
- **Required fields**: Product name, category, quantity, unit, price
- **EBM fields**: Required only if `useEbmIntegration = true`
- **Optional fields**: Purchase price, store, description, expiration date, image

---

## 3. Data Transformation

### Location: Lines 454-487

Before sending to API, the frontend needs to convert user-friendly values to API-expected IDs:

#### A. Category Name → Category ID
```dart
// Lines 455-469
1. Call: ManajaAPIGroup.getAllCategoriesCall.call()
2. Get list of categories from response
3. Loop through categories
4. Find category where name matches user selection
5. Extract categoryId from that category
```

#### B. Store Name → Store ID
```dart
// Lines 471-487
1. Call: ManajaAPIGroup.getAllStoresCall.call()
2. Get stores list from response: $.data.stores
3. Loop through stores
4. Find store where name matches user selection
5. Extract storeId from that store
```

#### C. Unit Name → Unit Code
```dart
// Lines 50-63: _getQtyUnitCd() helper function
'Kilograms (Kg)' → 'KG'
'Grams (Gr)' → 'GR'
'Liters (L)' → 'L'
'each (ea)' → 'EA'
'Pair (pr)' → 'PR'
'Dozen (doz)' → 'DZ'
' meter' → 'M'
```

#### D. Date Range → Date String
```dart
// Lines 65-69: _formatExpirationDate() helper function
DateTimeRange → 'YYYY-MM-DD' format
Example: DateTimeRange(start: 2025-12-25) → '2025-12-25'
```

---

## 4. API Request Structure

### Location: Lines 489-531

The code checks if it's **edit mode** or **create mode**:

```dart
if (widget.editMode && widget.productId != null) {
  // UPDATE: Call updateItemCall
  response = await ManajaAPIGroup.updateItemCall.call(...);
} else {
  // CREATE: Call createNewItemCall
  response = await ManajaAPIGroup.createNewItemCall.call(...);
}
```

### API Endpoint Details:

**Create Product:**
- **URL**: `POST /api/v1/items`
- **Method**: `POST`
- **Location**: `api_calls.dart` lines 1553-1660

**Update Product:**
- **URL**: `PUT /api/v1/items/{id}`
- **Method**: `PUT`
- **Location**: `api_calls.dart` lines 1688-1772

---

## 5. Request Body Format

### Location: `api_calls.dart` lines 1575-1619

The API call builds a JSON body with the following structure:

#### Standard Request (when `useEbmIntegration = false`):
```json
{
  "name": "Samsung Galaxy S21",
  "categoryId": "category-uuid-here",
  "price": "250000",
  "description": "Latest Samsung smartphone",
  "quantity": 10,
  "stockId": "store-uuid-here",
  "purchasePrice": 200000.0,        // optional
  "expirationDate": "2025-12-31", // optional, empty = never expires
  "qtyUnitCd": "EA",              // mapped from unit dropdown
  "image": "https://..."           // optional, if imageUrl provided
}
```

#### EBM Integration Request (when `useEbmIntegration = true`):
```json
{
  "name": "Samsung Galaxy S21",
  "categoryId": "category-uuid-here",
  "price": "250000",
  "description": "Latest Samsung smartphone",
  "quantity": 10,
  "stockId": "store-uuid-here",
  "orgnNatCd": "RW",              // EBM: Origin Country Code
  "pkgUnitCd": "NT",              // EBM: Package Unit Code
  "qtyUnitCd": "U",               // EBM: Quantity Unit Code
  "taxTyCd": "B",                 // EBM: Tax Type Code
  "productType": "2"              // EBM: Product Type
}
```

### Request Body Building Process:

```dart
// Lines 1575-1619 in api_calls.dart
final bodyMap = <String, dynamic>{};

// Always included
bodyMap['name'] = productName;
bodyMap['price'] = price;              // String
bodyMap['quantity'] = quantityNum;      // Integer (parsed)
bodyMap['qtyUnitCd'] = qtyUnitCd;      // String

// Conditionally included (if not empty)
if (purchasePrice.isNotEmpty) {
  bodyMap['purchasePrice'] = purchasePriceNum;  // Double
}
if (storeId.isNotEmpty) {
  bodyMap['stockId'] = storeId;  // Note: API uses 'stockId', not 'storeId'
}
if (expirationDate.isNotEmpty) {
  bodyMap['expirationDate'] = expirationDate;
}
if (description.isNotEmpty) {
  bodyMap['description'] = description;
}
if (categoryId.isNotEmpty) {
  bodyMap['categoryId'] = categoryId;
}

// EBM fields (only if provided)
if (orgnNatCd.isNotEmpty) {
  bodyMap['orgnNatCd'] = orgnNatCd;
}
if (pkgUnitCd.isNotEmpty) {
  bodyMap['pkgUnitCd'] = pkgUnitCd;
}
if (taxTyCd.isNotEmpty) {
  bodyMap['taxTyCd'] = taxTyCd;
}
if (productType.isNotEmpty) {
  bodyMap['productType'] = productType;
}
```

### Image Handling:

**If image file is uploaded:**
- Uses **multipart/form-data** request
- Image sent as file in `params['image']`
- Other fields sent as form data

**If image URL is provided:**
- Uses **JSON** request
- Image URL sent as `"image": "https://..."`

---

## 6. API Response Handling

### Location: Lines 533-591

### Success Response:
```dart
if (response.succeeded) {
  // Show success message
  ScaffoldMessenger.of(context).showSnackBar(
    SnackBar(
      content: Text('Product created successfully!'),
      backgroundColor: FlutterFlowTheme.of(context).success,
    ),
  );
  // Navigate to store page
  context.pushNamed('store');
}
```

### Error Response:
The code tries multiple ways to extract error messages:

```dart
// Lines 550-572: Error message extraction
1. Try: response.jsonBody['message']
2. Try: response.jsonBody['errorMessage']
3. Try: response.jsonBody['error']
4. Try: response.jsonBody['errors'][0]['message']  // Array of errors
5. Fallback: Generic error message
```

### Error Response Structure Examples:

**Single Error:**
```json
{
  "status": "error",
  "message": "Category not found"
}
```

**Multiple Errors:**
```json
{
  "status": "error",
  "errors": [
    {
      "field": "price",
      "message": "Price must be greater than 0"
    },
    {
      "field": "quantity",
      "message": "Quantity must be a positive number"
    }
  ]
}
```

---

## 7. Error Handling

### Location: Lines 544-591

### Error Display Flow:

1. **Check if response succeeded** (line 533)
   - If `response.succeeded == true` → Success path
   - If `response.succeeded == false` → Error path

2. **Extract error message** (lines 550-572):
   ```dart
   String errorMessage = 'Failed to create product. Please try again.';
   
   // Try different error message fields
   if (response.jsonBody['message'] != null) {
     errorMessage = response.jsonBody['message'];
   } else if (response.jsonBody['errorMessage'] != null) {
     errorMessage = response.jsonBody['errorMessage'];
   }
   // ... etc
   ```

3. **Show error to user** (lines 584-590):
   ```dart
   ScaffoldMessenger.of(context).showSnackBar(
     SnackBar(
       content: Text(errorMessage),
       backgroundColor: FlutterFlowTheme.of(context).error,
       duration: Duration(seconds: 5),
     ),
   );
   ```

4. **Debug logging** (lines 575-581):
   - In debug mode, prints full error response to console
   - Helps developers debug API issues

### Exception Handling:

```dart
// Lines 592-604: Catch any exceptions
try {
  // API call code
} catch (e) {
  // Show exception message to user
  ScaffoldMessenger.of(context).showSnackBar(
    SnackBar(
      content: Text('Error creating product: $e'),
      backgroundColor: FlutterFlowTheme.of(context).error,
    ),
  );
  // Log exception in debug mode
  if (kDebugMode) {
    print('Exception: $e');
  }
}
```

**Common Exceptions:**
- Network errors (no internet)
- Timeout errors
- JSON parsing errors
- Null pointer exceptions

---

## 8. Success Handling

### Location: Lines 533-543

### Success Flow:

1. **Check response status**:
   ```dart
   if (response.succeeded) {
     // Success!
   }
   ```

2. **Show success message**:
   ```dart
   ScaffoldMessenger.of(context).showSnackBar(
     SnackBar(
       content: Text('Product created successfully!'),
       backgroundColor: FlutterFlowTheme.of(context).success,
     ),
   );
   ```

3. **Navigate to store page**:
   ```dart
   context.pushNamed('store');
   ```
   - User is redirected to the store/products list page
   - Can see the newly created product

---

## 🔄 Complete Flow Diagram

```
User fills form
    ↓
Click "Publish Now"
    ↓
Validate required fields
    ↓ (if validation fails)
Show error: "Please fill in all required fields"
    ↓ (if validation passes)
Validate EBM fields (if EBM enabled)
    ↓ (if EBM validation fails)
Show error: "Please fill in all EBM integration fields"
    ↓ (if all validation passes)
Fetch Category ID from category name
    ↓
Fetch Store ID from store name
    ↓
Transform data:
  - Unit name → Unit code
  - Date range → Date string
    ↓
Build request body
    ↓
Make API call (POST /items or PUT /items/{id})
    ↓
Wait for response
    ↓
┌─────────────────┬─────────────────┐
│   SUCCESS       │     ERROR       │
│                 │                 │
│ Show success    │ Extract error   │
│ message         │ message         │
│                 │                 │
│ Navigate to     │ Show error      │
│ store page      │ message         │
└─────────────────┴─────────────────┘
```

---

## 📝 Key Code Locations

| Functionality | File | Lines |
|--------------|------|-------|
| Button click handler | `adding_product_widget.dart` | 422-605 |
| Validation | `adding_product_widget.dart` | 423-452 |
| Data transformation | `adding_product_widget.dart` | 454-487 |
| API call (create) | `api_calls.dart` | 1553-1660 |
| API call (update) | `api_calls.dart` | 1688-1772 |
| Request body building | `api_calls.dart` | 1575-1619 |
| Response handling | `adding_product_widget.dart` | 533-591 |
| Error handling | `adding_product_widget.dart` | 544-604 |
| Helper: Unit mapping | `adding_product_widget.dart` | 50-63 |
| Helper: Date formatting | `adding_product_widget.dart` | 65-69 |

---

## 🎯 Summary

1. **User Input** → Form fields collect product data
2. **Validation** → Required fields and EBM fields checked
3. **Data Transformation** → Names converted to IDs, formats standardized
4. **API Request** → JSON body built and sent to backend
5. **Response Processing** → Success or error message extracted
6. **User Feedback** → SnackBar shows result
7. **Navigation** → On success, redirects to store page

The entire flow is **asynchronous** (uses `async/await`) to prevent UI blocking during API calls.

