# B2C Commerce Cloud — Guia de App Nativa Mobile

**Autor:** Luciano Diaz — Principal Solution Engineer en Salesforce

**Fecha:** Junio 2026

---

## Introduccion

Esta guia cubre los fundamentos para construir apps de comercio nativas (Android/iOS) que se conectan a Salesforce B2C Commerce Cloud como motor de comercio. El approach usa **SCAPI (Shopper Commerce API)** y **SLAS (Shopper Login and API Access Service)** como capa de integracion — las mismas APIs que usa Composable Storefront, pero consumidas directamente desde codigo nativo mobile.

La app mobile actua como un cliente headless: B2C Commerce Cloud provee catalogo, pricing, promociones, carrito y ordenes via REST APIs, mientras que la app nativa es duenia de toda la capa de presentacion y experiencia de usuario.

### Arquitectura

```
┌─────────────────────────────────────────────┐
│           App Mobile Nativa                  │
│  ┌───────────┐  ┌───────────────────────┐   │
│  │  iOS      │  │  Android              │   │
│  │  (Swift)  │  │  (Kotlin)             │   │
│  └─────┬─────┘  └──────────┬────────────┘   │
└────────┼────────────────────┼────────────────┘
         │                    │
         ▼                    ▼
┌─────────────────────────────────────────────┐
│         SLAS (Autenticacion)                 │
│   OAuth 2.0 — Public Client (sin secret)    │
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
│  (Catalogo, Pricing, Inventario, Checkout)   │
└─────────────────────────────────────────────┘
```

### Decisiones Clave

| Decision | Recomendacion |
|----------|---------------|
| Tipo de cliente auth | **Public client** (SLAS) — sin client secret en el dispositivo |
| HTTP client | Retrofit + OkHttp (Android) / URLSession (iOS) |
| Storage de tokens | Android Keystore / iOS Keychain |
| Checkout | **WebView** — evita que la app entre en scope PCI DSS |
| Carga de imagenes | Coil/Glide (Android) / SDWebImage/Kingfisher (iOS) |

---

## 1. Autenticacion (SLAS)

SLAS es la capa de autenticacion para todas las Shopper APIs. Para apps mobile se usa un **public client** (sin client secret) porque los secrets no se pueden guardar de forma segura en dispositivos.

### URL Base de SLAS

```
https://{shortCode}.api.commercecloud.salesforce.com/shopper/auth/v1/organizations/{organizationId}
```

### Token de Invitado (Sesion Anonima)

Toda sesion de la app arranca con un guest token. Esto permite navegar, buscar y agregar productos al carrito sin necesidad de registrarse.

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

// Uso
val token = slasApi.getGuestToken(
    clientId = "tu-slas-public-client-id",
    siteId = "tu-site-id"
)
// Guardar token.access_token de forma segura en Android Keystore
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

### Login de Usuario Registrado

Para clientes registrados se usa el grant type `credentials`. El flujo involucra un endpoint de login que devuelve un authorization code, que despues se intercambia por un access token.

#### Android (Kotlin)

```kotlin
// Paso 1: Autenticar credenciales y obtener authorization code
@FormUrlEncoded
@POST("oauth2/login")
suspend fun login(
    @Header("Authorization") basicAuth: String, // Base64(username:password)
    @Field("client_id") clientId: String,
    @Field("channel_id") siteId: String,
    @Field("response_type") responseType: String = "code",
    @Field("redirect_uri") redirectUri: String
): LoginResponse

// Paso 2: Intercambiar code por token
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
    // Paso 1: Generar PKCE code verifier y challenge
    let codeVerifier = generateCodeVerifier()
    let codeChallenge = generateCodeChallenge(from: codeVerifier)

    // Paso 2: Obtener authorization code
    var loginRequest = URLRequest(url: slasLoginURL)
    loginRequest.httpMethod = "POST"
    let credentials = "\(username):\(password)".data(using: .utf8)!.base64EncodedString()
    loginRequest.setValue("Basic \(credentials)", forHTTPHeaderField: "Authorization")

    let loginBody = "client_id=\(clientId)&channel_id=\(siteId)&response_type=code&redirect_uri=\(redirectUri)&code_challenge=\(codeChallenge)"
    loginRequest.httpBody = loginBody.data(using: .utf8)

    let (loginData, _) = try await URLSession.shared.data(for: loginRequest)
    let loginResponse = try JSONDecoder().decode(LoginResponse.self, from: loginData)

    // Paso 3: Intercambiar code por token
    var tokenRequest = URLRequest(url: slasTokenURL)
    tokenRequest.httpMethod = "POST"
    let tokenBody = "grant_type=authorization_code_pkce&client_id=\(clientId)&code=\(loginResponse.code)&redirect_uri=\(redirectUri)&code_verifier=\(codeVerifier)&channel_id=\(siteId)"
    tokenRequest.httpBody = tokenBody.data(using: .utf8)

    let (tokenData, _) = try await URLSession.shared.data(for: tokenRequest)
    return try JSONDecoder().decode(TokenResponse.self, from: tokenData)
}
```

### Refresh de Token

Los access tokens expiran (tipicamente 30 minutos). Se usa el refresh token para obtener un nuevo access token sin volver a autenticar.

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

## 2. Carruseles de Productos (Destacados / Recomendaciones)

Los carruseles de productos tipicamente muestran conjuntos curados de productos o recomendaciones basadas en busqueda. Se usa la **Shopper Products** o **Shopper Search** API dependiendo del origen de los datos.

### URL Base SCAPI

```
https://{shortCode}.api.commercecloud.salesforce.com/product/shopper-products/v1/organizations/{organizationId}
```

### Traer Productos por IDs (Carrusel Curado)

Cuando Business Manager define un product set o slot con IDs especificos de productos:

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
        @Query("ids") ids: String // IDs separados por coma
    ): ProductResult
}

// Uso — traer productos para un carrusel
val products = shopperProductsApi.getProducts(
    token = "Bearer $accessToken",
    siteId = "tu-site-id",
    ids = "producto-1,producto-2,producto-3,producto-4,producto-5"
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

### Carrusel Dinamico por Busqueda (Trending, Novedades)

Se usa Shopper Search para armar carruseles dinamicos basados en queries de busqueda, ordenamiento o refinamientos:

```kotlin
// Android — Mas vendidos en una categoria
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

// Traer productos trending
val trending = searchApi.searchProducts(
    token = "Bearer $accessToken",
    siteId = "tu-site-id",
    sort = "best-matches",
    limit = 10
)

// Traer novedades de una categoria especifica
val novedades = searchApi.searchProducts(
    token = "Bearer $accessToken",
    siteId = "tu-site-id",
    refine = "cgid=novedades",
    sort = "most-recent",
    limit = 10
)
```

---

## 3. Menu de Navegacion y Categorias

La estructura de navegacion viene de la **Shopper Products** category API. B2C Commerce Cloud organiza los productos en un arbol jerarquico de categorias que se administra desde Business Manager.

### Traer el Arbol de Categorias

```
GET /organizations/{orgId}/categories/{categoryId}?levels=2&siteId={siteId}
```

El parametro `levels` controla cuantos niveles de subcategorias se devuelven (maximo 2 por request).

#### Android (Kotlin)

```kotlin
data class Category(
    val id: String,
    val name: String,
    val description: String?,
    val image: String?,
    val categories: List<Category>?, // subcategorias
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

// Traer el arbol raiz de categorias para el menu de navegacion
val rootCategory = categoriesApi.getCategory(
    token = "Bearer $accessToken",
    categoryId = "root", // "root" devuelve el arbol de categorias de nivel superior
    siteId = "tu-site-id",
    levels = 2
)

// rootCategory.categories contiene categorias L1 (Mujer, Hombre, Electronica...)
// Cada categoria L1 tiene su propio .categories para L2 (Remeras, Pantalones, Zapatos...)
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

### Armando el Menu de Navegacion

El patron tipico de navegacion para una app de comercio mobile:

```
┌──────────────────────────────────┐
│  ☰  Menu                         │
├──────────────────────────────────┤
│  > Mujer                         │
│  > Hombre                        │
│  > Electronica                   │
│  > Hogar y Jardin                │
│  > Ofertas                       │
├──────────────────────────────────┤
│  Mi Cuenta                       │
│  Pedidos                         │
│  Favoritos                       │
└──────────────────────────────────┘

// Tap en "Mujer" → expande subcategorias
┌──────────────────────────────────┐
│  ← Mujer                         │
├──────────────────────────────────┤
│  Ver Todo Mujer                  │
│  Vestidos                        │
│  Remeras                         │
│  Pantalones                      │
│  Zapatos                         │
│  Accesorios                      │
└──────────────────────────────────┘
```

Se mapea `rootCategory.categories` a los items del menu L1, y el `.categories` de cada subcategoria al drill-down L2.

---

## 4. Listado de Productos por Categoria (PLP)

Cuando un usuario toca una categoria, se usa Shopper Search para traer productos con paginacion, ordenamiento y filtros.

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

// Traer productos de una categoria con refinamientos
val categoryProducts = searchApi.searchProducts(
    token = "Bearer $accessToken",
    siteId = "tu-site-id",
    refine = "cgid=vestidos-mujer",
    sort = "price-low-to-high",
    limit = 24,
    offset = 0
)

// Aplicar filtros adicionales (color, talle, rango de precio)
val filtrado = searchApi.searchProducts(
    token = "Bearer $accessToken",
    siteId = "tu-site-id",
    refine = "cgid=vestidos-mujer&c_refinementColor=Negro&price=(5000..10000)",
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

## 5. Pagina de Producto (PDP)

La pagina de detalle de producto requiere un fetch completo del producto incluyendo imagenes, variaciones, pricing y disponibilidad.

### Traer un Producto Completo

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

// Traer producto con detalle completo
val product = productsApi.getProduct(
    token = "Bearer $accessToken",
    productId = "25502228M",
    siteId = "tu-site-id"
)

// Mostrar opciones de variacion (Color, Talle)
product.variationAttributes?.forEach { attr ->
    println("${attr.name}: ${attr.values.map { it.name }}")
    // Color: [Negro, Azul, Rojo]
    // Talle: [S, M, L, XL]
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

## 6. Carrito (Gestion del Basket)

El basket (carrito) en B2C Commerce Cloud es un recurso del lado del servidor vinculado a la sesion del comprador. La **Shopper Baskets API** maneja todas las operaciones del carrito.

### URL Base SCAPI Baskets

```
https://{shortCode}.api.commercecloud.salesforce.com/checkout/shopper-baskets/v1/organizations/{organizationId}
```

### Crear un Basket

Se crea un basket nuevo cuando el comprador agrega el primer item. Un solo basket activo por sesion de comprador.

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

    // Crear un basket vacio nuevo
    @POST("baskets")
    suspend fun createBasket(
        @Header("Authorization") token: String,
        @Query("siteId") siteId: String,
        @Body body: Map<String, Any> = emptyMap()
    ): Basket

    // Agregar item al basket
    @POST("baskets/{basketId}/items")
    suspend fun addItemToBasket(
        @Header("Authorization") token: String,
        @Path("basketId") basketId: String,
        @Query("siteId") siteId: String,
        @Body items: List<AddItemRequest>
    ): Basket

    // Actualizar cantidad del item
    @PATCH("baskets/{basketId}/items/{itemId}")
    suspend fun updateItem(
        @Header("Authorization") token: String,
        @Path("basketId") basketId: String,
        @Path("itemId") itemId: String,
        @Query("siteId") siteId: String,
        @Body body: UpdateItemRequest
    ): Basket

    // Eliminar item del basket
    @DELETE("baskets/{basketId}/items/{itemId}")
    suspend fun removeItem(
        @Header("Authorization") token: String,
        @Path("basketId") basketId: String,
        @Path("itemId") itemId: String,
        @Query("siteId") siteId: String
    ): Basket

    // Obtener basket actual
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

    // Crear basket si no existe
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

    // Agregar producto al basket
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

    // Actualizar cantidad
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

    // Eliminar item
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

### Ciclo de Vida del Basket

| Evento | Llamada API | Notas |
|--------|-------------|-------|
| Apertura de app (invitado) | Obtener guest token via SLAS | Todavia no hay basket |
| Primer "Agregar al Carrito" | `POST /baskets` + `POST /baskets/{id}/items` | Crea basket, agrega item |
| Agregados posteriores | `POST /baskets/{id}/items` | Mismo basket |
| Cambiar cantidad | `PATCH /baskets/{id}/items/{itemId}` | Actualiza cantidad |
| Eliminar item | `DELETE /baskets/{id}/items/{itemId}` | Saca del basket |
| Aplicar cupon | `POST /baskets/{id}/coupons` | `{ "code": "DESCUENTO10" }` |
| Usuario se loguea | Basket se mergea automaticamente via SLAS session bridge | Basket invitado → basket registrado |

---

## 7. Checkout — Estrategia WebView (PCI Compliance)

### Por que WebView para el Checkout

El procesamiento de pagos involucra recolectar numeros de tarjeta de credito, CVVs y datos del titular. Si la app nativa maneja estos datos directamente, la app cae dentro del **scope de PCI DSS**, lo que requiere:

- Auditorias de seguridad anuales (SAQ-D o ROC)
- Escaneos de red trimestrales
- Requisitos estrictos de code review y penetration testing
- Overhead de compliance significativo y costoso

**El approach recomendado:** delegar el checkout (especificamente la recoleccion de pago) a un **WebView** que cargue la pagina de checkout de B2C Commerce Cloud o una pagina de pago PCI-compliant. La app nativa nunca toca datos de tarjeta, manteniendola **fuera del scope PCI**.

### Enfoques de Implementacion

#### Opcion A: Checkout Completo en WebView (Recomendado por Simplicidad)

Se carga todo el flujo de checkout (envio, facturacion, pago, revision, confirmacion) en un WebView. La pagina de checkout de SFRA o Composable Storefront se encarga de todo.

```
┌──────────────────────────────────┐
│  App Nativa                       │
│  ┌────────────────────────────┐  │
│  │  Carrito (Nativo)          │  │
│  │  [Ir al Checkout] ────────┼──┼──► Abre WebView
│  └────────────────────────────┘  │
│                                   │
│  ┌────────────────────────────┐  │
│  │  WebView                   │  │
│  │  ┌──────────────────────┐  │  │
│  │  │ Pagina de Checkout   │  │  │
│  │  │ (SFRA o Composable)  │  │  │
│  │  │                      │  │  │
│  │  │ • Direccion envio    │  │  │
│  │  │ • Metodo de envio    │  │  │
│  │  │ • Datos de pago ← PCI│ │  │
│  │  │ • Revision de orden  │  │  │
│  │  │ • Confirmar compra   │  │  │
│  │  └──────────────────────┘  │  │
│  └────────────────────────────┘  │
│                                   │
│  Al detectar URL de confirmacion ─┼──► Cierra WebView, muestra confirmacion nativa
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

                // Detectar confirmacion de orden y volver a nativo
                if (url.contains("/order-confirmation") || url.contains("/orderconfirmation")) {
                    val orderId = extractOrderId(url)
                    showNativeConfirmation(orderId)
                    return true
                }
                return false
            }
        }

        // Cargar checkout con session bridging
        val checkoutUrl = buildCheckoutUrl()
        webView.loadUrl(checkoutUrl)
    }

    private fun buildCheckoutUrl(): String {
        // Opcion 1: URL directa de checkout SFRA con session bridge
        return "https://tu-storefront.com/checkout?token=$accessToken"

        // Opcion 2: URL de SLAS session bridge que redirige al checkout
        // Esto asegura que el WebView comparta la misma sesion de comprador
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
            // Navegar a pantalla nativa de confirmacion de orden
        }
    }
}
```

#### Opcion B: Checkout Hibrido (Nativo + WebView solo para Pago)

Se recolecta la direccion de envio/facturacion de forma nativa (mejor UX), pero se usa un WebView o SDK de pago **solamente para el paso de pago**. Esto mantiene los datos de tarjeta fuera de la app nativa mientras ofrece una experiencia mas nativa para los pasos no sensibles del checkout.

```
Nativo: Direccion Envio → Nativo: Metodo Envio → WebView: Solo Pago → Nativo: Confirmacion
```

Para este approach, se usa la Shopper Baskets API para setear direcciones de envio/facturacion de forma nativa:

```kotlin
// Setear direccion de envio nativamente
@PUT("baskets/{basketId}/shipments/{shipmentId}/shipping-address")
suspend fun setShippingAddress(
    @Header("Authorization") token: String,
    @Path("basketId") basketId: String,
    @Path("shipmentId") shipmentId: String,
    @Query("siteId") siteId: String,
    @Body address: ShippingAddress
): Basket

// Setear metodo de envio nativamente
@PUT("baskets/{basketId}/shipments/{shipmentId}/shipping-method")
suspend fun setShippingMethod(
    @Header("Authorization") token: String,
    @Path("basketId") basketId: String,
    @Path("shipmentId") shipmentId: String,
    @Query("siteId") siteId: String,
    @Body method: ShippingMethodRequest
): Basket

// Despues abrir WebView SOLO para recoleccion de pago
```

#### Opcion C: SDKs de Pago de Terceros

Algunos procesadores de pago proveen SDKs mobile que manejan el compliance PCI:

| Proveedor | SDK | Como funciona |
|-----------|-----|---------------|
| Stripe | Stripe iOS/Android SDK | Tokeniza datos de tarjeta client-side, envia token a tu servidor |
| Adyen | Adyen Drop-in | Componentes de UI de pago pre-armados, PCI-compliant |
| PayPal | PayPal Mobile SDK | Checkout nativo PayPal/Venmo |
| Apple Pay / Google Pay | APIs nativas del OS | Pagos tokenizados, no se tocan datos de tarjeta |
| MercadoPago | MercadoPago SDK | Checkout Pro o tokenizacion, muy usado en LATAM |

Si la instancia de B2C Commerce Cloud esta configurada con alguno de estos procesadores de pago, se puede integrar su SDK mobile directamente. El SDK tokeniza el pago, y se pasa el token a la Shopper Orders API para colocar la orden.

### Session Bridging (WebView ↔ SLAS)

Para que el WebView comparta la misma sesion de comprador que la app nativa, se usa SLAS session bridging:

1. La app nativa tiene el access token de SLAS (de guest o registered login).
2. Se llama al endpoint de session-bridge de SLAS para obtener un session bridge token.
3. Se carga la URL del checkout en el WebView con el session bridge token como parametro.
4. El WebView hereda el mismo basket, perfil de comprador y sesion.

```kotlin
// Obtener session bridge token
@GET("oauth2/session-bridge/token")
suspend fun getSessionBridgeToken(
    @Header("Authorization") token: String,
    @Query("client_id") clientId: String,
    @Query("channel_id") siteId: String
): SessionBridgeResponse
```

---

## 8. Resumen — Mapa Rapido de APIs

| Funcionalidad | API | Patron de Endpoint |
|---------------|-----|-------------------|
| Autenticacion | SLAS | `/shopper/auth/v1/.../oauth2/token` |
| Detalle Producto | Shopper Products | `/product/shopper-products/v1/.../products/{id}` |
| Busqueda Productos | Shopper Search | `/search/shopper-search/v1/.../product-search` |
| Categorias | Shopper Products | `/product/shopper-products/v1/.../categories/{id}` |
| Crear Basket | Shopper Baskets | `POST /checkout/shopper-baskets/v1/.../baskets` |
| Agregar al Basket | Shopper Baskets | `POST .../baskets/{id}/items` |
| Actualizar Item | Shopper Baskets | `PATCH .../baskets/{id}/items/{itemId}` |
| Eliminar Item | Shopper Baskets | `DELETE .../baskets/{id}/items/{itemId}` |
| Aplicar Cupon | Shopper Baskets | `POST .../baskets/{id}/coupons` |
| Setear Envio | Shopper Baskets | `PUT .../baskets/{id}/shipments/{id}/shipping-address` |
| Crear Orden | Shopper Orders | `POST /checkout/shopper-orders/v1/.../orders` |
| Historial Ordenes | Shopper Customers | `GET .../customers/{id}/orders` |

---

## 9. Consideraciones y Buenas Practicas

### Manejo de Tokens
- Guardar tokens en Android Keystore / iOS Keychain — nunca en SharedPreferences ni UserDefaults.
- Implementar refresh automatico del token antes de que expire (chequear `expires_in`).
- Manejar respuestas 401 de forma global con un interceptor que refresque y reintente.

### Caching
- Cachear datos de productos y categorias localmente (Room/CoreData o in-memory) con un TTL.
- Las imagenes deben usar una libreria de caching (Coil/Glide para Android, Kingfisher/SDWebImage para iOS).
- El basket siempre debe traerse fresco — no confiar en estado cacheado del carrito.

### Manejo de Errores
- SCAPI devuelve codigos HTTP estandar: 400 (bad request), 401 (token expirado), 404 (no encontrado), 429 (rate limited).
- Implementar exponential backoff para respuestas 429.
- Mostrar mensajes de error amigables al usuario para fallas de red.

### Performance
- Usar query parameters `expand` y `select` para pedir solo los campos que se necesitan.
- Paginar busquedas de productos con `limit` y `offset`.
- Pre-cargar el arbol de categorias al inicio de la app y cachearlo.

### Consideraciones Offline
- La app depende de las APIs — no hay una experiencia de comercio offline significativa.
- Cachear los ultimos productos y categorias vistos para navegacion offline de contenido ya visto.
- Encolar acciones de agregar-al-carrito si esta offline y sincronizar cuando vuelva la conectividad (opcional, complejo).

### Consideraciones para LATAM
- Los precios se manejan en la moneda configurada en el site de B2C Commerce (ARS, CLP, etc.).
- Los metodos de envio y direcciones deben adaptarse al formato local (provincia/region, codigo postal).
- Para medios de pago locales (MercadoPago, transferencia bancaria), la integracion tipicamente se hace a traves de un procesador de pago configurado en B2C Commerce y se maneja via WebView o SDK del proveedor.

---

## Documentacion Oficial

| Recurso | URL |
|---------|-----|
| Referencia SCAPI | https://developer.salesforce.com/docs/commerce/commerce-api/references/ |
| Guia SLAS | https://developer.salesforce.com/docs/commerce/commerce-api/guide/authorization-for-shopper-apis.html |
| Setup Public Client SLAS | https://developer.salesforce.com/docs/commerce/commerce-api/guide/slas-public-client.html |
| Shopper Baskets API | https://developer.salesforce.com/docs/commerce/commerce-api/references/shopper-baskets |
| Shopper Products API | https://developer.salesforce.com/docs/commerce/commerce-api/references/shopper-products |
| Shopper Search API | https://developer.salesforce.com/docs/commerce/commerce-api/references/shopper-search |
| Shopper Orders API | https://developer.salesforce.com/docs/commerce/commerce-api/references/shopper-orders |
| commerce-sdk-isomorphic (referencia) | https://github.com/SalesforceCommerceCloud/commerce-sdk-isomorphic |
