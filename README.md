# Ferretería API

Guía de integración REST para la app Android en Kotlin. Backend: Apache + PHP 8.5 (PDO) + MariaDB 11.8, corriendo en una VM Ubuntu detrás de un Cloudflare Tunnel.

| | |
|---|---|
| **Base URL** | `https://ferreteria-api.proxmox-lab.cc/api/` |
| **Auth** | Header `X-API-Key` (ver abajo) |
| **Formato** | JSON (`Content-Type: application/json`) |
| **TLS** | Certificado válido de Cloudflare |

> ⚠️ **Repo privado.** Este README incluye la API key real porque el acceso está controlado por GitHub. **Nunca** copies la key a un repo público ni la hardcodees si vas a publicar el código de la app en abierto — usa `local.properties` / `BuildConfig` (ver sección Retrofit).

## Antes de escribir código

- [ ] Permiso `<uses-permission android:name="android.permission.INTERNET" />` en `AndroidManifest.xml`.
- [ ] La API key de este README, cargada vía `local.properties` (nunca hardcodeada si el repo de la app es público).
- [ ] Dependencias de Retrofit + Moshi + OkHttp (sección [Cliente Retrofit](#cliente-retrofit)).
- [ ] `id_cliente` e `id_empleado` válidos al registrar una venta — vienen de `GET /clientes.php` y de la tabla `EMPLEADO`.

## Autenticación

Cada request debe incluir el header `X-API-Key`. Sin él, o con un valor incorrecto, la API responde `401` sin tocar la base de datos.

```
X-API-Key: 46472a83defc364341f42101ee161c6754a8ed78629ab54878844fa0a49bca33
```

```bash
curl -H "X-API-Key: 46472a83defc364341f42101ee161c6754a8ed78629ab54878844fa0a49bca33" \
  https://ferreteria-api.proxmox-lab.cc/api/productos.php
```

Si esta key se filtra fuera de este repo (por ejemplo, un commit accidental a un repo público), avisa para regenerarla en el servidor — invalida la anterior de inmediato.

## Endpoints

| Método | Ruta | Uso |
|---|---|---|
| `GET` | `/api/productos.php` | Lista todos los productos con categoría y proveedor |
| `GET` | `/api/productos.php?id=` | Un producto específico |
| `GET` | `/api/clientes.php` | Lista de clientes |
| `GET` | `/api/ventas.php?id=` | Detalle de una venta con sus productos |
| `POST` | `/api/ventas.php` | Registra una venta nueva (transaccional) |

### `GET /api/productos.php`

Lista todos los productos, con categoría y proveedor ya resueltos por `JOIN`.

```json
[
  {
    "id_producto": 1,
    "nombre_producto": "Martillo",
    "descripcion_producto": "Martillo de una libra",
    "precio_compra": "90.00",
    "precio_venta": "150.00",
    "stock": 37,
    "id_categoria": 1,
    "nombre_categoria": "Herramientas",
    "id_proveedor": 1,
    "proveedor": "Truper"
  }
]
```

### `GET /api/productos.php?id=3`

Un solo producto. `200` si existe, `404` si no, `400` si `id` no es numérico.

### `GET /api/clientes.php`

```json
[
  {
    "id_cliente": 1,
    "nombre_cliente": "Ana",
    "ap_paterno_cliente": "Lopez",
    "ap_materno_cliente": "Sanchez",
    "telefono_cliente": "8999602293",
    "email_cliente": "ana.lopez@mail.com"
  }
]
```

### `GET /api/ventas.php?id=1`

Detalle de una venta ya registrada, con el arreglo `productos` incluido.

```json
{
  "id_venta": 1,
  "fecha": "2026-08-10 10:15:00",
  "total": "1150.00",
  "id_cliente": 1,
  "nombre_cliente": "Ana",
  "id_empleado": 1,
  "nombre_empleado": "Guillermo Daniel",
  "productos": [
    {
      "id_detalle": 1,
      "cantidad": 2,
      "precio_unitario": "150.00",
      "subtotal": "300.00",
      "id_producto": 1,
      "nombre_producto": "Martillo"
    }
  ]
}
```

### `POST /api/ventas.php`

Registra una venta completa dentro de una transacción: calcula subtotales y total, inserta en `VENTA` y `DETALLE_VENTA`, y descuenta `stock`. Si algo falla (stock insuficiente, cliente inexistente) hace `ROLLBACK` y no cambia nada. La app solo envía `id_producto` y `cantidad` — el servidor calcula el resto.

**Request:**

```json
{
  "id_cliente": 2,
  "id_empleado": 3,
  "productos": [
    { "id_producto": 1, "cantidad": 3 },
    { "id_producto": 5, "cantidad": 2 }
  ]
}
```

**Respuesta `201`:**

```json
{
  "id_venta": 6,
  "id_cliente": 2,
  "id_empleado": 3,
  "total": "540.00",
  "productos": [
    { "id_producto": 1, "cantidad": 3, "precio_unitario": "150.00", "subtotal": "450.00" },
    { "id_producto": 5, "cantidad": 2, "precio_unitario": "45.00", "subtotal": "90.00" }
  ]
}
```

## Códigos de error

Todas las respuestas son JSON. Los errores siempre tienen la forma `{"error": "mensaje"}`.

| Código | Significado en esta API |
|---|---|
| `200` | GET exitoso |
| `201` | Venta creada correctamente |
| `400` | Parámetro inválido/faltante, JSON malformado, stock insuficiente, o `id_cliente`/`id_empleado` inexistente |
| `401` | Falta el header `X-API-Key` o no coincide |
| `404` | El producto, cliente o venta consultado no existe |
| `500` | Error interno del servidor |

## Modelos Kotlin

```kotlin
data class Producto(
    val id_producto: Int,
    val nombre_producto: String,
    val descripcion_producto: String?,
    val precio_compra: String,
    val precio_venta: String,
    val stock: Int,
    val id_categoria: Int,
    val nombre_categoria: String,
    val id_proveedor: Int,
    val proveedor: String
)

data class Cliente(
    val id_cliente: Int,
    val nombre_cliente: String,
    val ap_paterno_cliente: String,
    val ap_materno_cliente: String?,
    val telefono_cliente: String,
    val email_cliente: String
)

data class DetalleVenta(
    val id_detalle: Int,
    val cantidad: Int,
    val precio_unitario: String,
    val subtotal: String,
    val id_producto: Int,
    val nombre_producto: String
)

data class Venta(
    val id_venta: Int,
    val fecha: String,
    val total: String,
    val id_cliente: Int,
    val nombre_cliente: String,
    val id_empleado: Int,
    val nombre_empleado: String,
    val productos: List<DetalleVenta>
)

// -- Para el POST --

data class VentaItemRequest(val id_producto: Int, val cantidad: Int)

data class VentaRequest(
    val id_cliente: Int,
    val id_empleado: Int,
    val productos: List<VentaItemRequest>
)

data class VentaCreada(
    val id_venta: Int,
    val id_cliente: Int,
    val id_empleado: Int,
    val total: String,
    val productos: List<Map<String, Any>>
)

data class ApiError(val error: String)
```

## Cliente Retrofit

### 1. Dependencias (`app/build.gradle.kts`)

```kotlin
implementation("com.squareup.retrofit2:retrofit:2.11.0")
implementation("com.squareup.retrofit2:converter-moshi:2.11.0")
implementation("com.squareup.moshi:moshi-kotlin:1.15.1")
implementation("com.squareup.okhttp3:logging-interceptor:4.12.0")
```

### 2. Interfaz de la API

```kotlin
interface FerreteriaApi {
    @GET("productos.php")
    suspend fun getProductos(): Response<List<Producto>>

    @GET("productos.php")
    suspend fun getProducto(@Query("id") id: Int): Response<Producto>

    @GET("clientes.php")
    suspend fun getClientes(): Response<List<Cliente>>

    @GET("ventas.php")
    suspend fun getVenta(@Query("id") id: Int): Response<Venta>

    @POST("ventas.php")
    suspend fun crearVenta(@Body venta: VentaRequest): Response<VentaCreada>
}
```

### 3. Cargar la key sin hardcodearla (recomendado si este código va a un repo aparte y público)

`local.properties` (nunca se sube a git):

```properties
FERRETERIA_API_KEY=46472a83defc364341f42101ee161c6754a8ed78629ab54878844fa0a49bca33
```

`app/build.gradle.kts`:

```kotlin
android {
    buildFeatures { buildConfig = true }
    defaultConfig {
        val props = java.util.Properties().apply {
            load(rootProject.file("local.properties").inputStream())
        }
        buildConfigField(
            "String", "FERRETERIA_API_KEY",
            "\"${props["FERRETERIA_API_KEY"]}\""
        )
    }
}
```

### 4. Interceptor + instancia de Retrofit

```kotlin
private const val BASE_URL = "https://ferreteria-api.proxmox-lab.cc/api/"

private val apiKeyInterceptor = Interceptor { chain ->
    val request = chain.request().newBuilder()
        .addHeader("X-API-Key", BuildConfig.FERRETERIA_API_KEY)
        .build()
    chain.proceed(request)
}

private val client = OkHttpClient.Builder()
    .addInterceptor(apiKeyInterceptor)
    .build()

val api: FerreteriaApi = Retrofit.Builder()
    .baseUrl(BASE_URL)
    .client(client)
    .addConverterFactory(MoshiConverterFactory.create())
    .build()
    .create(FerreteriaApi::class.java)
```

### 5. Uso — registrar una venta

```kotlin
viewModelScope.launch {
    val venta = VentaRequest(
        id_cliente = 2,
        id_empleado = 3,
        productos = listOf(
            VentaItemRequest(id_producto = 1, cantidad = 3),
            VentaItemRequest(id_producto = 5, cantidad = 2)
        )
    )

    val response = api.crearVenta(venta)
    if (response.isSuccessful) {
        val nuevaVenta = response.body()
        // actualizar UI, refrescar stock, etc.
    } else {
        // response.code() = 400/401/404/500
        // response.errorBody()?.string() trae {"error": "..."}
    }
}
```

## Seguridad

- Todo el tráfico va por HTTPS con certificado válido de Cloudflare — no se necesita `network_security_config.xml` ni confiar en certificados propios.
- MariaDB solo escucha en `127.0.0.1` del servidor: la app nunca se conecta directo a la base de datos, siempre pasa por esta API.
- Todos los queries del servidor usan *prepared statements* (PDO) — la app no construye SQL.
- La API key vive solo en este repo privado y en `local.properties` de cada dev — nunca en el historial de git de un repo público.

---

Backend: Apache + PHP 8.5 (PDO) + MariaDB 11.8 · Túnel: Cloudflare (`ferreteria-api.proxmox-lab.cc`)
