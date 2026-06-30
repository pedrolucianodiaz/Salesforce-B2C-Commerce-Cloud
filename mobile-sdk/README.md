# B2C Commerce Cloud — Native Mobile App Guide

**Author:** Luciano Diaz — Principal Solution Engineer at Salesforce

**Date:** June 2026

---

## Overview

This guide covers the fundamentals for building native mobile commerce apps (Android/iOS) that connect to Salesforce B2C Commerce Cloud as their commerce engine. The approach uses **SCAPI (Shopper Commerce API)** and **SLAS (Shopper Login and API Access Service)** as the integration layer — the same APIs used by Composable Storefront, but consumed directly from native mobile code.

The mobile app acts as a headless client: B2C Commerce Cloud provides catalog, pricing, promotions, cart, and order services via REST APIs, while the native app owns the entire presentation layer and user experience.

### Architecture

```
┌─────────────────────────────────────────────┐
│           Native Mobile App                  │
│  ┌───────────┐  ┌───────────────────────┐   │
│  │  iOS      │  │  Android              │   │
│  │  (Swift)  │  │  (Kotlin)             │   │
│  └─────┬─────┘  └──────────┬────────────┘   │
└────────┼────────────────────┼────────────────┘
         │                    │
         ▼                    ▼
┌─────────────────────────────────────────────┐
│         SLAS (Authentication)                │
│   OAuth 2.0 — Public Client (no secret)     │
└──────────────────┬──────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────┐
│      SCAPI (Shopper Commerce API)            │
│  Products · Search · Baskets · Orders        │
│  Customers · Promotions · Categories         │
└──────────────────┬──────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────┐
│        B2C Commerce Cloud                    │
│  (Catalog, Pricing, Inventory, Checkout)     │
└─────────────────────────────────────────────┘
```

### Key Decisions

| Decision | Recommendation |
|----------|---------------|
| Auth client type | **Public client** (SLAS) — no client secret stored on device |
| HTTP client | Retrofit + OkHttp (Android) / URLSession (iOS) |
| Token storage | Android Keystore / iOS Keychain |
| Checkout | **WebView** — avoids PCI DSS scope for the native app |
| Image loading | Coil/Glide (Android) / SDWebImage/Kingfisher (iOS) |

---

## 1. Authentication (SLAS)

SLAS is the authentication layer for all Shopper APIs. For mobile apps, use a **public client** (no client secret) since secrets cannot be securely stored on devices.

### SLAS Base URL

```
https://{shortCode}.api.commercecloud.salesforce.com/shopper/auth/v1/organizations/{organizationId}
```

### Guest Token (Anonymous Session)

Every app session starts with a guest token. This allows browsing, searching, and adding items to a basket without registration.

#### Android (Kotlin)

```kotlin
data class TokenResponse(
    val access_token: String,
    val refresh_token: String,
    val expires_in: Int,
    val token_type: String,
    val usid: String,
    val customer_id: String
)

interface SlasApi {
    @FormUrlEncoded
    @POST("oauth2/token")
    suspend fun getGuestToken(
        @Field("grant_type") grantType: String = "client_credentials",
        @Field("client_id") clientId: String,
        @Field("channel_id") siteId: String
    ): TokenResponse
}

// Usage
val token = slasApi.getGuestToken(
    clientId = "your-slas-public-client-id",
    siteId = "your-site-id"
)
// Store token.access_token securely in Android Keystore
```

#### iOS (Swift)

```swift
struct TokenResponse: Codable {
    let accessToken: String
    let refreshToken: String
    let expiresIn: Int
    let tokenType: String
    let usid: String
    let customerId: String

    enum CodingKeys: String, CodingKey {
        case accessToken = "access_token"
        case refreshToken = "refresh_token"
        case expiresIn = "expires_in"
        case tokenType = "token_type"
        case usid
        case customerId = "customer_id"
    }
}

func getGuestToken() async throws -> TokenResponse {
    var request = URLRequest(url: slasTokenURL)
    request.httpMethod = "POST"
    request.setValue("application/x-www-form-urlencoded", forHTTPHeaderField: "Content-Type")

    let body = "grant_type=client_credentials&client_id=\(clientId)&channel_id=\(siteId)"
    request.httpBody = body.data(using: .utf8)

    let (data, _) = try await URLSession.shared.data(for: request)
    return try JSONDecoder().decode(TokenResponse.self, from: data)
}
```

### Registered Shopper Login

For registered customers, use the `credentials` grant type. The flow involves a login endpoint that returns an authorization code, which is then exchanged for an access token.

#### Android (Kotlin)

```kotlin
// Step 1: Authenticate credentials and get authorization code
@FormUrlEncoded
@POST("oauth2/login")
suspend fun login(
    @Header("Authorization") basicAuth: String, // Base64(username:password)
    @Field("client_id") clientId: String,
    @Field("channel_id") siteId: String,
    @Field("response_type") responseType: String = "code",
    @Field("redirect_uri") redirectUri: String
): LoginResponse

// Step 2: Exchange code for token
@FormUrlEncoded
@POST("oauth2/token")
suspend fun exchangeCode(
    @Field("grant_type") grantType: String = "authorization_code_pkce",
    @Field("client_id") clientId: String,
    @Field("code") code: String,
    @Field("redirect_uri") redirectUri: String,
    @Field("code_verifier") codeVerifier: String,
    @Field("channel_id") siteId: String
): TokenResponse
```

#### iOS (Swift)

```swift
func loginRegisteredShopper(username: String, password: String) async throws -> TokenResponse {
    // Step 1: Generate PKCE code verifier and challenge
    let codeVerifier = generateCodeVerifier()
    let codeChallenge = generateCodeChallenge(from: codeVerifier)

    // Step 2: Get authorization code
    var loginRequest = URLRequest(url: slasLoginURL)
    loginRequest.httpMethod = "POST"
    let credentials = "\(username):\(password)".data(using: .utf8)!.base64EncodedString()
    loginRequest.setValue("Basic \(credentials)", forHTTPHeaderField: "Authorization")

    let loginBody = "client_id=\(clientId)&channel_id=\(siteId)&response_type=code&redirect_uri=\(redirectUri)&code_challenge=\(codeChallenge)"
    loginRequest.httpBody = loginBody.data(using: .utf8)

    let (loginData, _) = try await URLSession.shared.data(for: loginRequest)
    let loginResponse = try JSONDecoder().decode(LoginResponse.self, from: loginData)

    // Step 3: Exchange code for token
    var tokenRequest = URLRequest(url: slasTokenURL)
    tokenRequest.httpMethod = "POST"
    let tokenBody = "grant_type=authorization_code_pkce&client_id=\(clientId)&code=\(loginResponse.code)&redirect_uri=\(redirectUri)&code_verifier=\(codeVerifier)&channel_id=\(siteId)"
    tokenRequest.httpBody = tokenBody.data(using: .utf8)

    let (tokenData, _) = try await URLSession.shared.data(for: tokenRequest)
    return try JSONDecoder().decode(TokenResponse.self, from: tokenData)
}
```

### Token Refresh

Access tokens expire (typically 30 minutes). Use the refresh token to obtain a new access token without re-authentication.

```kotlin
// Android
@FormUrlEncoded
@POST("oauth2/token")
suspend fun refreshToken(
    @Field("grant_type") grantType: String = "refresh_token",
    @Field("client_id") clientId: String,
    @Field("refresh_token") refreshToken: String,
    @Field("channel_id") siteId: String
): TokenResponse
```

```swift
// iOS
func refreshAccessToken(refreshToken: String) async throws -> TokenResponse {
    var request = URLRequest(url: slasTokenURL)
    request.httpMethod = "POST"
    let body = "grant_type=refresh_token&client_id=\(clientId)&refresh_token=\(refreshToken)&channel_id=\(siteId)"
    request.httpBody = body.data(using: .utf8)
    let (data, _) = try await URLSession.shared.data(for: request)
    return try JSONDecoder().decode(TokenResponse.self, from: data)
}
```

---

## 2. Product Carousels (Featured Products / Recommendations)

Product carousels typically display curated product sets or search-driven recommendations. Use the **Shopper Products** or **Shopper Search** API depending on the source.

### SCAPI Base URL

```
https://{shortCode}.api.commercecloud.salesforce.com/product/shopper-products/v1/organizations/{organizationId}
```

### Fetching Products by IDs (Curated Carousel)

When Business Manager defines a product set or slot with specific product IDs:

#### Android (Kotlin)

```kotlin
data class ProductResult(
    val data: List<Product>
)

data class Product(
    val id: String,
    val name: String,
    val price: Double?,
    val currency: String?,
    val imageGroups: List<ImageGroup>?
)

data class ImageGroup(
    val viewType: String,
    val images: List<ProductImage>
)

data class ProductImage(
    val link: String,
    val alt: String?,
    val title: String?
)

interface ShopperProductsApi {
    @GET("products")
    suspend fun getProducts(
        @Header("Authorization") token: String,
        @Query("siteId") siteId: String,
        @Query("ids") ids: String // comma-separated product IDs
    ): ProductResult
}

// Usage — fetch products for a carousel
val products = shopperProductsApi.getProducts(
    token = "Bearer $accessToken",
    siteId = "your-site-id",
    ids = "product-1,product-2,product-3,product-4,product-5"
)
```

#### iOS (Swift)

```swift
struct ProductResult: Codable {
    let data: [Product]
}

struct Product: Codable {
    let id: String
    let name: String
    let price: Double?
    let currency: String?
    let imageGroups: [ImageGroup]?
}

struct ImageGroup: Codable {
    let viewType: String
    let images: [ProductImage]
}

struct ProductImage: Codable {
    let link: String
    let alt: String?
    let title: String?
}

func getProducts(ids: [String]) async throws -> [Product] {
    let idsParam = ids.joined(separator: ",")
    let url = URL(string: "\(scapiBase)/products?siteId=\(siteId)&ids=\(idsParam)")!

    var request = URLRequest(url: url)
    request.setValue("Bearer \(accessToken)", forHTTPHeaderField: "Authorization")

    let (data, _) = try await URLSession.shared.data(for: request)
    let result = try JSONDecoder().decode(ProductResult.self, from: data)
    return result.data
}
```

### Search-Driven Carousel (Trending, New Arrivals)

Use Shopper Search to power dynamic carousels based on search queries, sorting, or refinements:

```kotlin
// Android — Top sellers in a category
interface ShopperSearchApi {
    @GET("product-search")
    suspend fun searchProducts(
        @Header("Authorization") token: String,
        @Query("siteId") siteId: String,
        @Query("q") query: String = "",
        @Query("refine") refine: String? = null,
        @Query("sort") sort: String? = null,
        @Query("limit") limit: Int = 10,
        @Query("offset") offset: Int = 0
    ): SearchResult
}

// Fetch trending products
val trending = searchApi.searchProducts(
    token = "Bearer $accessToken",
    siteId = "your-site-id",
    sort = "best-matches",
    limit = 10
)

// Fetch new arrivals in a specific category
val newArrivals = searchApi.searchProducts(
    token = "Bearer $accessToken",
    siteId = "your-site-id",
    refine = "cgid=new-arrivals",
    sort = "most-recent",
    limit = 10
)
```

---

## 3. Navigation Menu and Categories

The navigation structure comes from the **Shopper Products** category API. B2C Commerce Cloud organizes products into a hierarchical category tree managed in Business Manager.

### Fetching the Category Tree

```
GET /organizations/{orgId}/categories/{categoryId}?levels=2&siteId={siteId}
```

The `levels` parameter controls how many levels of subcategories are returned (max 2 per request).

#### Android (Kotlin)

```kotlin
data class Category(
    val id: String,
    val name: String,
    val description: String?,
    val image: String?,
    val categories: List<Category>?, // subcategories
    val parentCategoryId: String?
)

interface ShopperCategoriesApi {
    @GET("categories/{categoryId}")
    suspend fun getCategory(
        @Header("Authorization") token: String,
        @Path("categoryId") categoryId: String,
        @Query("siteId") siteId: String,
        @Query("levels") levels: Int = 2
    ): Category
}

// Fetch the root category tree for navigation menu
val rootCategory = categoriesApi.getCategory(
    token = "Bearer $accessToken",
    categoryId = "root", // "root" returns the top-level category tree
    siteId = "your-site-id",
    levels = 2
)

// rootCategory.categories contains L1 categories (Men, Women, Electronics...)
// Each L1 category has its own .categories for L2 (Shirts, Pants, Shoes...)
```

#### iOS (Swift)

```swift
struct Category: Codable {
    let id: String
    let name: String
    let description: String?
    let image: String?
    let categories: [Category]?
    let parentCategoryId: String?
}

func getCategoryTree() async throws -> Category {
    let url = URL(string: "\(scapiBase)/categories/root?siteId=\(siteId)&levels=2")!
    var request = URLRequest(url: url)
    request.setValue("Bearer \(accessToken)", forHTTPHeaderField: "Authorization")

    let (data, _) = try await URLSession.shared.data(for: request)
    return try JSONDecoder().decode(Category.self, from: data)
}
```

### Building the Navigation Menu

The typical navigation pattern for a mobile commerce app:

```
┌──────────────────────────────────┐
│  ☰  Menu                         │
├──────────────────────────────────┤
│  > Women                         │
│  > Men                           │
│  > Electronics                   │
│  > Home & Garden                 │
│  > Sale                          │
├──────────────────────────────────┤
│  My Account                      │
│  Orders                          │
│  Wishlist                        │
└──────────────────────────────────┘

// Tap "Women" → expand subcategories
┌──────────────────────────────────┐
│  ← Women                         │
├──────────────────────────────────┤
│  View All Women                  │
│  Dresses                         │
│  Tops                            │
│  Pants                           │
│  Shoes                           │
│  Accessories                     │
└──────────────────────────────────┘
```

Map `rootCategory.categories` to the L1 menu items, and each subcategory's `.categories` to the L2 drill-down.

---

## 4. Category Listing Page (PLP)

When a user taps a category, use Shopper Search to retrieve products with pagination, sorting, and filtering.

#### Android (Kotlin)

```kotlin
data class SearchResult(
    val total: Int,
    val hits: List<ProductHit>,
    val refinements: List<Refinement>,
    val sortingOptions: List<SortOption>
)

data class ProductHit(
    val productId: String,
    val productName: String,
    val price: Double?,
    val image: ProductImage?,
    val variationAttributes: List<VariationAttribute>?
)

data class Refinement(
    val attributeId: String,
    val label: String,
    val values: List<RefinementValue>
)

data class RefinementValue(
    val label: String,
    val value: String,
    val hitCount: Int
)

// Fetch products in a category with refinements
val categoryProducts = searchApi.searchProducts(
    token = "Bearer $accessToken",
    siteId = "your-site-id",
    refine = "cgid=womens-dresses",
    sort = "price-low-to-high",
    limit = 24,
    offset = 0
)

// Apply additional refinements (color, size, price range)
val filtered = searchApi.searchProducts(
    token = "Bearer $accessToken",
    siteId = "your-site-id",
    refine = "cgid=womens-dresses&c_refinementColor=Black&price=(50..100)",
    sort = "price-low-to-high",
    limit = 24,
    offset = 0
)
```

#### iOS (Swift)

```swift
struct SearchResult: Codable {
    let total: Int
    let hits: [ProductHit]
    let refinements: [Refinement]?
    let sortingOptions: [SortOption]?
}

func searchByCategory(categoryId: String, offset: Int = 0, limit: Int = 24) async throws -> SearchResult {
    let url = URL(string: "\(searchBase)/product-search?siteId=\(siteId)&refine=cgid=\(categoryId)&limit=\(limit)&offset=\(offset)")!
    var request = URLRequest(url: url)
    request.setValue("Bearer \(accessToken)", forHTTPHeaderField: "Authorization")

    let (data, _) = try await URLSession.shared.data(for: request)
    return try JSONDecoder().decode(SearchResult.self, from: data)
}
```

---

## 5. Product Detail Page (PDP)

The product detail page requires a full product fetch including images, variations, pricing, and availability.

### Fetching a Complete Product

```
GET /organizations/{orgId}/products/{productId}?siteId={siteId}&allImages=true&expand=availability,promotions,variations,prices
```

#### Android (Kotlin)

```kotlin
data class ProductDetail(
    val id: String,
    val name: String,
    val shortDescription: String?,
    val longDescription: String?,
    val price: Double?,
    val priceMax: Double?,
    val currency: String?,
    val imageGroups: List<ImageGroup>,
    val variationAttributes: List<VariationAttribute>?,
    val variants: List<Variant>?,
    val inventory: Inventory?
)

data class VariationAttribute(
    val id: String,
    val name: String,
    val values: List<VariationValue>
)

data class VariationValue(
    val value: String,
    val name: String,
    val image: ProductImage?,
    val orderable: Boolean?
)

data class Inventory(
    val ats: Double?,
    val orderable: Boolean,
    val stockLevel: Double?
)

interface ShopperProductsApi {
    @GET("products/{productId}")
    suspend fun getProduct(
        @Header("Authorization") token: String,
        @Path("productId") productId: String,
        @Query("siteId") siteId: String,
        @Query("allImages") allImages: Boolean = true,
        @Query("expand") expand: String = "availability,promotions,variations,prices"
    ): ProductDetail
}

// Fetch product with full details
val product = productsApi.getProduct(
    token = "Bearer $accessToken",
    productId = "25502228M",
    siteId = "your-site-id"
)

// Display variation options (Color, Size)
product.variationAttributes?.forEach { attr ->
    println("${attr.name}: ${attr.values.map { it.name }}")
    // Color: [Black, Blue, Red]
    // Size: [S, M, L, XL]
}
```

#### iOS (Swift)

```swift
struct ProductDetail: Codable {
    let id: String
    let name: String
    let shortDescription: String?
    let longDescription: String?
    let price: Double?
    let priceMax: Double?
    let currency: String?
    let imageGroups: [ImageGroup]
    let variationAttributes: [VariationAttribute]?
    let variants: [Variant]?
    let inventory: Inventory?
}

func getProductDetail(productId: String) async throws -> ProductDetail {
    let url = URL(string: "\(scapiBase)/products/\(productId)?siteId=\(siteId)&allImages=true&expand=availability,promotions,variations,prices")!
    var request = URLRequest(url: url)
    request.setValue("Bearer \(accessToken)", forHTTPHeaderField: "Authorization")

    let (data, _) = try await URLSession.shared.data(for: request)
    return try JSONDecoder().decode(ProductDetail.self, from: data)
}
```

---

## 6. Cart (Basket Management)

The basket (cart) in B2C Commerce Cloud is a server-side resource tied to the shopper's session. The **Shopper Baskets API** handles all cart operations.

### SCAPI Baskets Base URL

```
https://{shortCode}.api.commercecloud.salesforce.com/checkout/shopper-baskets/v1/organizations/{organizationId}
```

### Create a Basket

A new basket is created when the shopper first adds an item. One active basket per shopper session.

#### Android (Kotlin)

```kotlin
data class Basket(
    val basketId: String,
    val productItems: List<ProductItem>?,
    val productSubTotal: Double?,
    val productTotal: Double?,
    val orderTotal: Double?,
    val shippingTotal: Double?,
    val taxTotal: Double?,
    val currency: String?
)

data class ProductItem(
    val itemId: String,
    val productId: String,
    val productName: String,
    val quantity: Int,
    val price: Double,
    val basePrice: Double
)

interface ShopperBasketsApi {

    // Create a new empty basket
    @POST("baskets")
    suspend fun createBasket(
        @Header("Authorization") token: String,
        @Query("siteId") siteId: String,
        @Body body: Map<String, Any> = emptyMap()
    ): Basket

    // Add item to basket
    @POST("baskets/{basketId}/items")
    suspend fun addItemToBasket(
        @Header("Authorization") token: String,
        @Path("basketId") basketId: String,
        @Query("siteId") siteId: String,
        @Body items: List<AddItemRequest>
    ): Basket

    // Update item quantity
    @PATCH("baskets/{basketId}/items/{itemId}")
    suspend fun updateItem(
        @Header("Authorization") token: String,
        @Path("basketId") basketId: String,
        @Path("itemId") itemId: String,
        @Query("siteId") siteId: String,
        @Body body: UpdateItemRequest
    ): Basket

    // Remove item from basket
    @DELETE("baskets/{basketId}/items/{itemId}")
    suspend fun removeItem(
        @Header("Authorization") token: String,
        @Path("basketId") basketId: String,
        @Path("itemId") itemId: String,
        @Query("siteId") siteId: String
    ): Basket

    // Get current basket
    @GET("baskets/{basketId}")
    suspend fun getBasket(
        @Header("Authorization") token: String,
        @Path("basketId") basketId: String,
        @Query("siteId") siteId: String
    ): Basket
}

data class AddItemRequest(
    val productId: String,
    val quantity: Int
)

data class UpdateItemRequest(
    val quantity: Int
)
```

#### iOS (Swift)

```swift
struct Basket: Codable {
    let basketId: String
    let productItems: [ProductItem]?
    let productSubTotal: Double?
    let productTotal: Double?
    let orderTotal: Double?
    let shippingTotal: Double?
    let taxTotal: Double?
    let currency: String?
}

class BasketService {
    private var currentBasketId: String?

    // Create basket if none exists
    func getOrCreateBasket() async throws -> Basket {
        if let basketId = currentBasketId {
            return try await getBasket(basketId: basketId)
        }

        let url = URL(string: "\(basketsBase)/baskets?siteId=\(siteId)")!
        var request = URLRequest(url: url)
        request.httpMethod = "POST"
        request.setValue("Bearer \(accessToken)", forHTTPHeaderField: "Authorization")
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")
        request.httpBody = "{}".data(using: .utf8)

        let (data, _) = try await URLSession.shared.data(for: request)
        let basket = try JSONDecoder().decode(Basket.self, from: data)
        currentBasketId = basket.basketId
        return basket
    }

    // Add product to basket
    func addToBasket(productId: String, quantity: Int) async throws -> Basket {
        let basket = try await getOrCreateBasket()
        let url = URL(string: "\(basketsBase)/baskets/\(basket.basketId)/items?siteId=\(siteId)")!

        var request = URLRequest(url: url)
        request.httpMethod = "POST"
        request.setValue("Bearer \(accessToken)", forHTTPHeaderField: "Authorization")
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")

        let body = [["productId": productId, "quantity": quantity]] as [[String: Any]]
        request.httpBody = try JSONSerialization.data(withJSONObject: body)

        let (data, _) = try await URLSession.shared.data(for: request)
        return try JSONDecoder().decode(Basket.self, from: data)
    }

    // Update item quantity
    func updateQuantity(itemId: String, quantity: Int) async throws -> Basket {
        guard let basketId = currentBasketId else { throw BasketError.noBasket }
        let url = URL(string: "\(basketsBase)/baskets/\(basketId)/items/\(itemId)?siteId=\(siteId)")!

        var request = URLRequest(url: url)
        request.httpMethod = "PATCH"
        request.setValue("Bearer \(accessToken)", forHTTPHeaderField: "Authorization")
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")
        request.httpBody = try JSONEncoder().encode(["quantity": quantity])

        let (data, _) = try await URLSession.shared.data(for: request)
        return try JSONDecoder().decode(Basket.self, from: data)
    }

    // Remove item
    func removeItem(itemId: String) async throws -> Basket {
        guard let basketId = currentBasketId else { throw BasketError.noBasket }
        let url = URL(string: "\(basketsBase)/baskets/\(basketId)/items/\(itemId)?siteId=\(siteId)")!

        var request = URLRequest(url: url)
        request.httpMethod = "DELETE"
        request.setValue("Bearer \(accessToken)", forHTTPHeaderField: "Authorization")

        let (data, _) = try await URLSession.shared.data(for: request)
        return try JSONDecoder().decode(Basket.self, from: data)
    }
}
```

### Basket Lifecycle

| Event | API Call | Notes |
|-------|----------|-------|
| App launch (guest) | Get guest token via SLAS | No basket yet |
| First "Add to Cart" | `POST /baskets` + `POST /baskets/{id}/items` | Creates basket, adds item |
| Subsequent adds | `POST /baskets/{id}/items` | Same basket |
| Change quantity | `PATCH /baskets/{id}/items/{itemId}` | Update quantity |
| Remove item | `DELETE /baskets/{id}/items/{itemId}` | Removes from basket |
| Apply coupon | `POST /baskets/{id}/coupons` | `{ "code": "SAVE10" }` |
| User logs in | Basket merges automatically via SLAS session bridge | Guest basket → registered basket |

---

## 7. Checkout — WebView Strategy (PCI Compliance)

### Why WebView for Checkout

Payment processing involves collecting credit card numbers, CVVs, and cardholder data. If the native app handles this data directly, the app falls within **PCI DSS scope**, requiring:

- Annual security audits (SAQ-D or ROC)
- Quarterly network scans
- Strict code review and penetration testing requirements
- Significant compliance overhead and cost

**The recommended approach:** delegate checkout (specifically payment collection) to a **WebView** that loads B2C Commerce Cloud's checkout page or a PCI-compliant payment page. The native app never touches card data, keeping it **out of PCI scope**.

### Implementation Approaches

#### Option A: Full Checkout in WebView (Recommended for Simplicity)

Load the entire checkout flow (shipping, billing, payment, review, confirmation) in a WebView. The SFRA or Composable Storefront checkout page handles everything.

```
┌──────────────────────────────────┐
│  Native App                       │
│  ┌────────────────────────────┐  │
│  │  Cart (Native)             │  │
│  │  [Proceed to Checkout] ────┼──┼──► Opens WebView
│  └────────────────────────────┘  │
│                                   │
│  ┌────────────────────────────┐  │
│  │  WebView                   │  │
│  │  ┌──────────────────────┐  │  │
│  │  │ Checkout Page (SFRA  │  │  │
│  │  │ or Composable)       │  │  │
│  │  │                      │  │  │
│  │  │ • Shipping address   │  │  │
│  │  │ • Shipping method    │  │  │
│  │  │ • Payment info ← PCI│  │  │
│  │  │ • Order review       │  │  │
│  │  │ • Place order        │  │  │
│  │  └──────────────────────┘  │  │
│  └────────────────────────────┘  │
│                                   │
│  On order confirmation URL ───────┼──► Close WebView, show native confirmation
└──────────────────────────────────┘
```

##### Android (Kotlin)

```kotlin
class CheckoutActivity : AppCompatActivity() {

    private lateinit var webView: WebView

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_checkout)

        webView = findViewById(R.id.checkout_webview)
        webView.settings.apply {
            javaScriptEnabled = true
            domStorageEnabled = true
        }

        webView.webViewClient = object : WebViewClient() {
            override fun shouldOverrideUrlLoading(view: WebView?, request: WebResourceRequest?): Boolean {
                val url = request?.url?.toString() ?: return false

                // Detect order confirmation and return to native
                if (url.contains("/order-confirmation") || url.contains("/orderconfirmation")) {
                    val orderId = extractOrderId(url)
                    showNativeConfirmation(orderId)
                    return true
                }
                return false
            }
        }

        // Load checkout with session bridging
        // The SLAS token is bridged via a session-bridge endpoint
        val checkoutUrl = buildCheckoutUrl()
        webView.loadUrl(checkoutUrl)
    }

    private fun buildCheckoutUrl(): String {
        // Option 1: Direct SFRA checkout URL with session bridge
        return "https://your-storefront.com/checkout?token=$accessToken"

        // Option 2: SLAS session bridge URL that redirects to checkout
        // This ensures the WebView shares the same shopper session
    }

    private fun showNativeConfirmation(orderId: String) {
        finish()
        startActivity(Intent(this, OrderConfirmationActivity::class.java).apply {
            putExtra("orderId", orderId)
        })
    }
}
```

##### iOS (Swift)

```swift
import WebKit

class CheckoutViewController: UIViewController, WKNavigationDelegate {
    private var webView: WKWebView!

    override func viewDidLoad() {
        super.viewDidLoad()

        let config = WKWebViewConfiguration()
        webView = WKWebView(frame: view.bounds, configuration: config)
        webView.navigationDelegate = self
        view.addSubview(webView)

        let checkoutURL = URL(string: buildCheckoutUrl())!
        webView.load(URLRequest(url: checkoutURL))
    }

    func webView(_ webView: WKWebView, decidePolicyFor navigationAction: WKNavigationAction,
                 decisionHandler: @escaping (WKNavigationActionPolicy) -> Void) {
        guard let url = navigationAction.request.url?.absoluteString else {
            decisionHandler(.allow)
            return
        }

        if url.contains("/order-confirmation") || url.contains("/orderconfirmation") {
            let orderId = extractOrderId(from: url)
            decisionHandler(.cancel)
            showNativeConfirmation(orderId: orderId)
            return
        }

        decisionHandler(.allow)
    }

    private func showNativeConfirmation(orderId: String) {
        dismiss(animated: true) {
            // Navigate to native order confirmation screen
        }
    }
}
```

#### Option B: Hybrid Checkout (Native + Payment WebView)

Collect shipping/billing address natively (better UX), but use a WebView or payment SDK **only for the payment step**. This keeps card data out of the native app while offering a more native feel for non-sensitive checkout steps.

```
Native: Shipping Address → Native: Shipping Method → WebView: Payment Only → Native: Confirmation
```

For this approach, you'd use the Shopper Baskets API to set shipping/billing addresses natively:

```kotlin
// Set shipping address natively
@PUT("baskets/{basketId}/shipments/{shipmentId}/shipping-address")
suspend fun setShippingAddress(
    @Header("Authorization") token: String,
    @Path("basketId") basketId: String,
    @Path("shipmentId") shipmentId: String,
    @Query("siteId") siteId: String,
    @Body address: ShippingAddress
): Basket

// Set shipping method natively
@PUT("baskets/{basketId}/shipments/{shipmentId}/shipping-method")
suspend fun setShippingMethod(
    @Header("Authorization") token: String,
    @Path("basketId") basketId: String,
    @Path("shipmentId") shipmentId: String,
    @Query("siteId") siteId: String,
    @Body method: ShippingMethodRequest
): Basket

// Then open WebView ONLY for payment collection
```

#### Option C: Third-Party Payment SDKs

Some payment processors provide mobile SDKs that handle PCI compliance:

| Provider | SDK | How it works |
|----------|-----|-------------|
| Stripe | Stripe iOS/Android SDK | Tokenizes card data client-side, sends token to your server |
| Adyen | Adyen Drop-in | Pre-built payment UI components, PCI-compliant |
| PayPal | PayPal Mobile SDK | Native PayPal/Venmo checkout |
| Apple Pay / Google Pay | Native OS APIs | Tokenized payments, no card data touched |

If the B2C Commerce Cloud instance is configured with one of these payment processors, you can integrate their mobile SDK directly. The SDK tokenizes the payment, and you pass the token to the Shopper Orders API to place the order.

### Session Bridging (WebView ↔ SLAS)

For the WebView to share the same shopper session as the native app, use SLAS session bridging:

1. The native app holds the SLAS access token (from guest or registered login).
2. Call the SLAS session-bridge endpoint to get a session bridge token.
3. Load the WebView checkout URL with the session bridge token as a parameter.
4. The WebView inherits the same basket, shopper profile, and session.

```kotlin
// Get session bridge token
@GET("oauth2/session-bridge/token")
suspend fun getSessionBridgeToken(
    @Header("Authorization") token: String,
    @Query("client_id") clientId: String,
    @Query("channel_id") siteId: String
): SessionBridgeResponse
```

---

## 8. Summary — API Reference Quick Map

| Feature | API | Endpoint Pattern |
|---------|-----|-----------------|
| Authentication | SLAS | `/shopper/auth/v1/.../oauth2/token` |
| Product Detail | Shopper Products | `/product/shopper-products/v1/.../products/{id}` |
| Product Search | Shopper Search | `/search/shopper-search/v1/.../product-search` |
| Categories | Shopper Products | `/product/shopper-products/v1/.../categories/{id}` |
| Create Basket | Shopper Baskets | `POST /checkout/shopper-baskets/v1/.../baskets` |
| Add to Basket | Shopper Baskets | `POST .../baskets/{id}/items` |
| Update Item | Shopper Baskets | `PATCH .../baskets/{id}/items/{itemId}` |
| Remove Item | Shopper Baskets | `DELETE .../baskets/{id}/items/{itemId}` |
| Apply Coupon | Shopper Baskets | `POST .../baskets/{id}/coupons` |
| Set Shipping | Shopper Baskets | `PUT .../baskets/{id}/shipments/{id}/shipping-address` |
| Place Order | Shopper Orders | `POST /checkout/shopper-orders/v1/.../orders` |
| Order History | Shopper Customers | `GET .../customers/{id}/orders` |

---

## 9. Considerations and Best Practices

### Token Management
- Store tokens in Android Keystore / iOS Keychain — never in SharedPreferences or UserDefaults.
- Implement automatic token refresh before expiration (check `expires_in`).
- Handle 401 responses globally with an interceptor that refreshes and retries.

### Caching
- Cache product data and categories locally (Room/CoreData or in-memory) with a TTL.
- Images should use a caching library (Coil/Glide for Android, Kingfisher/SDWebImage for iOS).
- The basket should always be fetched fresh — do not rely on cached basket state.

### Error Handling
- SCAPI returns standard HTTP status codes: 400 (bad request), 401 (expired token), 404 (not found), 429 (rate limited).
- Implement exponential backoff for 429 responses.
- Show user-friendly error messages for network failures.

### Performance
- Use `expand` and `select` query parameters to request only the fields you need.
- Paginate product searches with `limit` and `offset`.
- Prefetch the category tree at app launch and cache it.

### Offline Considerations
- The app is API-dependent — there is no meaningful offline commerce experience.
- Cache the last-viewed products and categories for offline browsing of previously seen content.
- Queue add-to-cart actions if offline and sync when connectivity returns (optional, complex).

---

## Official Documentation

| Resource | URL |
|----------|-----|
| SCAPI Reference | https://developer.salesforce.com/docs/commerce/commerce-api/references/ |
| SLAS Guide | https://developer.salesforce.com/docs/commerce/commerce-api/guide/authorization-for-shopper-apis.html |
| SLAS Public Client Setup | https://developer.salesforce.com/docs/commerce/commerce-api/guide/slas-public-client.html |
| Shopper Baskets API | https://developer.salesforce.com/docs/commerce/commerce-api/references/shopper-baskets |
| Shopper Products API | https://developer.salesforce.com/docs/commerce/commerce-api/references/shopper-products |
| Shopper Search API | https://developer.salesforce.com/docs/commerce/commerce-api/references/shopper-search |
| Shopper Orders API | https://developer.salesforce.com/docs/commerce/commerce-api/references/shopper-orders |
| commerce-sdk-isomorphic (reference) | https://github.com/SalesforceCommerceCloud/commerce-sdk-isomorphic |
