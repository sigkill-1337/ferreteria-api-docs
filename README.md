# Ferretería — Documentación del proyecto

Documentación de integración del sistema de punto de venta e inventario para la
ferretería: base de datos MariaDB, API REST en PHP y aplicación Android.

Todo lo que aparece aquí está **verificado contra el servidor en producción**,
no contra el código fuente. Las respuestas de ejemplo son respuestas reales.

> ⚠️ **Repo privado.** Este README incluye la API key real. No copies la key a un
> repo público ni la dejes escrita en el código de la app: va en
> `local.properties`, que se inyecta en `BuildConfig` (ver
> [Integración Kotlin](#integración-kotlin)).

---

## Índice

1. [Los tres repositorios](#los-tres-repositorios)
2. [Arquitectura](#arquitectura)
3. [Acceso](#acceso)
4. [Modelo de datos](#modelo-de-datos)
5. [Endpoints CRUD](#endpoints-crud)
6. [Reportes](#reportes)
7. [Procedimientos almacenados y triggers](#procedimientos-almacenados-y-triggers)
8. [Códigos de error](#códigos-de-error)
9. [Integración Kotlin](#integración-kotlin)
10. [Despliegue](#despliegue)

---

## Los tres repositorios

| Repo | Contiene | Visibilidad |
|---|---|---|
| `sigkill-1337/ferreteria-api` | Backend PHP y scripts SQL | Privado |
| `sigkill-1337/ferreteria-app` | App Android (Kotlin + Compose) | Privado |
| `sigkill-1337/ferreteria-api-docs` | Este documento | Privado |

## Arquitectura

```
┌──────────────────────┐
│  App Android         │  Kotlin + Jetpack Compose
│  Retrofit + Moshi    │  MVVM, sin base de datos local
└──────────┬───────────┘
           │ HTTPS · header X-API-Key
           ▼
┌──────────────────────┐
│  Cloudflare Tunnel   │  certificado válido, sin puertos abiertos
└──────────┬───────────┘
           ▼
┌──────────────────────┐
│  Apache + PHP 8.5    │  PDO, prepared statements
│  /var/www/ferreteria-api/
│    public/  ← DocumentRoot
│    inc/     ← fuera del webroot, aquí viven las credenciales
└──────────┬───────────┘
           ▼
┌──────────────────────┐
│  MariaDB 11.8        │  escucha solo en 127.0.0.1
│  8 tablas            │  3 procedimientos · 3 triggers
└──────────────────────┘
```

**La lógica de negocio vive en la base de datos y en PHP, no en la app.** La app
manda `id_producto` y `cantidad`; el servidor busca los precios, calcula
subtotales y total con `bcmath`, descuenta existencias y hace commit, todo
dentro de una transacción. La app nunca calcula un importe que se vaya a
guardar.

## Acceso

| | |
|---|---|
| **Base URL** | `https://ferreteria-api.proxmox-lab.cc/api/` |
| **Autenticación** | Header `X-API-Key` |
| **Formato** | JSON (`Content-Type: application/json`) |
| **TLS** | Certificado válido de Cloudflare |

```
X-API-Key: 46472a83defc364341f42101ee161c6754a8ed78629ab54878844fa0a49bca33
```

```bash
curl -H "X-API-Key: 46472a83defc364341f42101ee161c6754a8ed78629ab54878844fa0a49bca33" \
  https://ferreteria-api.proxmox-lab.cc/api/productos.php
```

Sin el header, o con un valor incorrecto, la API responde `401` **sin tocar la
base de datos**. La validación usa `hash_equals()` para no filtrar información
por el tiempo de respuesta.

## Modelo de datos

Ocho tablas. Las seis primeras son el esquema original; `VENTA.canal` y
`SEGUIMIENTO_CLIENTE` se agregaron para el reto final.

```
CATEGORIA ──┐
            ├──< PRODUCTO >──┐
PROVEEDOR ──┘                │
                             ├──< DETALLE_VENTA >── VENTA >── EMPLEADO
CLIENTE ─────────────────────┼───────────────────────┘  │
   │                         │                          │
   └──< SEGUIMIENTO_CLIENTE >┘                          │
            (el trigger la llena) ◄─────────────────────┘
```

`VENTA` ↔ `PRODUCTO` es una relación **muchos a muchos** resuelta con la tabla
puente `DETALLE_VENTA`.

### Tipos de dato

| Tabla | Campo | Tipo | Notas |
|---|---|---|---|
| `CATEGORIA` | `id_categoria` | `INT AUTO_INCREMENT` | PK |
| | `nombre_categoria` | `VARCHAR(50)` | obligatorio |
| | `descripcion` | `VARCHAR(150)` | opcional |
| `PROVEEDOR` | `id_proveedor` | `INT AUTO_INCREMENT` | PK |
| | `rfc` | `VARCHAR(13)` | **único** |
| | `nombre_empresa` | `VARCHAR(100)` | obligatorio |
| | `telefono_proveedor` | `VARCHAR(15)` | obligatorio |
| | `email_proveedor` | `VARCHAR(100)` | **único** |
| `PRODUCTO` | `id_producto` | `INT AUTO_INCREMENT` | PK |
| | `nombre_producto` | `VARCHAR(100)` | obligatorio |
| | `descripcion_producto` | `VARCHAR(200)` | opcional |
| | `precio_compra` | `DECIMAL(10,2)` | `CHECK >= 0` |
| | `precio_venta` | `DECIMAL(10,2)` | `CHECK >= 0` |
| | `stock` | `INT` | `CHECK >= 0` |
| | `id_categoria` | `INT` | FK → `CATEGORIA` |
| | `id_proveedor` | `INT` | FK → `PROVEEDOR` |
| `CLIENTE` | `id_cliente` | `INT AUTO_INCREMENT` | PK |
| | `nombre_cliente` | `VARCHAR(50)` | obligatorio |
| | `ap_paterno_cliente` | `VARCHAR(50)` | obligatorio |
| | `ap_materno_cliente` | `VARCHAR(50)` | opcional |
| | `telefono_cliente` | `VARCHAR(15)` | obligatorio |
| | `email_cliente` | `VARCHAR(100)` | **único** |
| `EMPLEADO` | `id_empleado` | `INT AUTO_INCREMENT` | PK |
| | `nombre_empleado` | `VARCHAR(50)` | obligatorio |
| | `ap_paterno_empleado` | `VARCHAR(50)` | obligatorio |
| | `ap_materno_empleado` | `VARCHAR(50)` | opcional |
| | `puesto` | `VARCHAR(50)` | obligatorio |
| | `sueldo` | `DECIMAL(10,2)` | `CHECK >= 0` |
| | `telefono_empleado` | `VARCHAR(15)` | obligatorio |
| | `email_empleado` | `VARCHAR(100)` | **único** |
| `VENTA` | `id_venta` | `INT AUTO_INCREMENT` | PK |
| | `fecha` | `DATETIME` | obligatorio |
| | `total` | `DECIMAL(10,2)` | `CHECK >= 0` |
| | `canal` | `ENUM('APP','WEB','MOSTRADOR')` | por omisión `APP` |
| | `id_cliente` | `INT` | FK → `CLIENTE` |
| | `id_empleado` | `INT` | FK → `EMPLEADO` |
| `DETALLE_VENTA` | `id_detalle` | `INT AUTO_INCREMENT` | PK |
| | `cantidad` | `INT` | `CHECK > 0` |
| | `precio_unitario` | `DECIMAL(10,2)` | foto del precio al vender |
| | `subtotal` | `DECIMAL(10,2)` | `CHECK >= 0` |
| | `id_venta` | `INT` | FK → `VENTA` |
| | `id_producto` | `INT` | FK → `PRODUCTO` |
| `SEGUIMIENTO_CLIENTE` | `id_seguimiento` | `INT AUTO_INCREMENT` | PK |
| | `id_cliente` | `INT` | FK → `CLIENTE`, `ON DELETE CASCADE` |
| | `nombre_cliente` | `VARCHAR(160)` | foto del nombre al comprar |
| | `canal` | `ENUM('APP','WEB','MOSTRADOR')` | obligatorio |
| | `total_compra` | `DECIMAL(10,2)` | obligatorio |
| | `fecha_hora` | `DATETIME` | obligatorio |
| | `atendido` | `TINYINT(1)` | por omisión `0` |
| | `id_venta` | `INT NULL` | FK → `VENTA`, `ON DELETE SET NULL` |

### Dos decisiones que conviene poder defender

**`DETALLE_VENTA.precio_unitario` y `SEGUIMIENTO_CLIENTE.nombre_cliente` parecen
redundantes y no lo son.** Ambos son datos históricos: guardan el valor *tal como
estaba en el momento del hecho*. Si mañana sube el precio de un martillo, los
tickets ya emitidos no deben cambiar; si un cliente corrige su apellido, la
bitácora debe seguir mostrando con qué nombre se le atendió. No violan la tercera
forma normal porque no dependen del valor actual de la otra tabla.

**Las reglas de borrado de `SEGUIMIENTO_CLIENTE` se eligieron por un problema
concreto.** Con la regla por omisión, cancelar una venta fallaría por llave
foránea, porque su registro de bitácora la referencia. De ahí `ON DELETE SET
NULL` en `id_venta`: la venta se cancela y el registro histórico sobrevive con la
referencia vacía. Y `ON DELETE CASCADE` en `id_cliente` para que dar de baja a un
cliente siga siendo posible.

## Endpoints CRUD

Todos requieren `X-API-Key`. Todos responden JSON.

| Recurso | Listar | Uno | Crear | Editar | Borrar |
|---|---|---|---|---|---|
| `/api/productos.php` | `GET` | `GET ?id=` | `POST` | `PUT ?id=` | `DELETE ?id=` |
| `/api/clientes.php` | `GET` | `GET ?id=` | `POST` | `PUT ?id=` | `DELETE ?id=` |
| `/api/empleados.php` | `GET` | `GET ?id=` | `POST` | `PUT ?id=` | `DELETE ?id=` |
| `/api/categorias.php` | `GET` | `GET ?id=` | `POST` | `PUT ?id=` | `DELETE ?id=` |
| `/api/proveedores.php` | `GET` | `GET ?id=` | `POST` | `PUT ?id=` | `DELETE ?id=` |
| `/api/ventas.php` | `GET` | `GET ?id=` | `POST` | `PUT ?id=` | `DELETE ?id=` |

`PUT` es **reemplazo completo**: hay que mandar todos los campos editables, no
solo los que cambiaron.

### `GET /api/productos.php`

Categoría y proveedor vienen resueltos por `JOIN`.

```json
[
  {
    "id_producto": 1,
    "nombre_producto": "Martillo",
    "descripcion_producto": "Martillo de una libra",
    "precio_compra": "90.00",
    "precio_venta": "150.00",
    "stock": 34,
    "id_categoria": 1,
    "nombre_categoria": "Herramientas",
    "id_proveedor": 1,
    "proveedor": "Truper"
  }
]
```

### `POST /api/clientes.php`

El alta **no es un `INSERT` directo**: se delega al procedimiento almacenado
`sp_alta_cliente`, que captura la violación de la restricción única de
`email_cliente`. Ver [Procedimientos](#procedimientos-almacenados-y-triggers).

```json
{
  "nombre_cliente": "Mariana",
  "ap_paterno_cliente": "Salinas",
  "ap_materno_cliente": "Vega",
  "telefono_cliente": "8999602298",
  "email_cliente": "mariana.salinas@mail.com"
}
```

Responde `201` con el cliente creado, o `409` si el correo ya existe.

### `GET /api/ventas.php`

Sin `id`, lista el historial con `num_productos` y sin el arreglo `productos`,
de la más reciente a la más antigua.

### `GET /api/ventas.php?id=7`

```json
{
  "id_venta": 7,
  "fecha": "2026-01-12 10:05:00",
  "total": "480.00",
  "canal": "APP",
  "id_cliente": 1,
  "nombre_cliente": "Ana",
  "ap_paterno_cliente": "Lopez",
  "ap_materno_cliente": "Sanchez",
  "id_empleado": 2,
  "nombre_empleado": "Jose Luis",
  "ap_paterno_empleado": "Huerta",
  "productos": [
    {
      "id_detalle": 13,
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

La app manda solo qué y cuánto:

```json
{
  "id_cliente": 2,
  "id_empleado": 3,
  "canal": "APP",
  "productos": [
    { "id_producto": 1, "cantidad": 3 },
    { "id_producto": 5, "cantidad": 2 }
  ]
}
```

Dentro de una transacción el servidor bloquea los productos con
`SELECT ... FOR UPDATE`, verifica existencias, calcula subtotales y total con
`bcmath`, inserta en `VENTA` y `DETALLE_VENTA`, y descuenta el stock. Si algo
falla hace `ROLLBACK` y no cambia nada. Responde `201` con la venta completa,
igual que `GET ?id=`.

`canal` acepta `APP`, `WEB` o `MOSTRADOR`; si no se manda, se asume `APP`. Las
de `APP` y `WEB` disparan el trigger de seguimiento.

**Las cantidades se agrupan por producto antes de validar.** Si el carrito manda
el mismo `id_producto` en dos renglones, se suman. Sin eso, cada renglón pasaba
la validación de existencias por separado y después reventaba el
`CHECK (stock >= 0)` con un `500`.

### `PUT /api/ventas.php?id=`

Repone al inventario el stock del detalle anterior, valida el nuevo y reescribe
la venta. La `fecha` original no se toca.

### `DELETE /api/ventas.php?id=`

Cancela la venta: **devuelve el stock al inventario**, borra el detalle y borra
la venta.

## Reportes

`GET /api/reportes.php?tipo=<reporte>`. Solo lectura. Sin `tipo`, devuelve la
lista de reportes disponibles.

| `tipo` | Parámetros | Se resuelve con |
|---|---|---|
| `ventas-dia` | `fecha=AAAA-MM-DD` | `sp_ventas_del_dia` |
| `clientes-trimestre` | `anio=2026` | `sp_clientes_vigentes_trimestre` |
| `top-productos` | — | `GROUP BY` sobre `DETALLE_VENTA` |
| `ventas-por-canal` | — | `GROUP BY` sobre `VENTA.canal` |
| `directorio` | — | `UNION` de `CLIENTE` y `EMPLEADO` |
| `seguimiento` | `limite=50` | Tabla que llena `trg_venta_seguimiento` |

### `?tipo=ventas-dia&fecha=2026-08-10`

```json
{
  "fecha": "2026-08-10",
  "num_ventas": 1,
  "monto_total": "1150.00",
  "ticket_promedio": "1150.00",
  "venta_mayor": "1150.00",
  "ventas": [
    {
      "id_venta": 1,
      "fecha": "2026-08-10 10:15:00",
      "hora": "10:15",
      "canal": "MOSTRADOR",
      "total": "1150.00",
      "id_cliente": 1,
      "cliente": "Ana Lopez",
      "empleado": "Guillermo Daniel Ramirez",
      "renglones": 2,
      "piezas": 3
    }
  ]
}
```

El procedimiento devuelve **dos conjuntos de resultados** —el desglose y el
resumen agrupado—; PHP los recorre con `nextRowset()` y los junta en un solo
JSON.

### `?tipo=ventas-por-canal`

```json
[
  { "canal": "MOSTRADOR", "num_ventas": 3, "monto_total": "3550.00", "porcentaje": 38.3 },
  { "canal": "APP",       "num_ventas": 6, "monto_total": "3390.00", "porcentaje": 36.6 },
  { "canal": "WEB",       "num_ventas": 3, "monto_total": "2330.00", "porcentaje": 25.1 }
]
```

### `?tipo=clientes-trimestre&anio=2026`

Devuelve `periodo: "01/01/2026 al 31/03/2026"` y la lista de clientes con sus
pedidos y montos, ordenada por monto descendente.

## Procedimientos almacenados y triggers

Viven en `ferreteria-api/sql/02-programabilidad.sql`.

> `TRY/CATCH` es sintaxis de SQL Server. En MariaDB el equivalente es
> `DECLARE ... HANDLER` para capturar y `SIGNAL` para lanzar. Es lo que se usa
> aquí, y conviene decirlo en el informe para que no parezca una omisión.

### `sp_ventas_del_dia(IN p_fecha DATE)`

Devuelve dos resultados: el desglose de las ventas de esa fecha y el resumen con
la suma de los montos, el ticket promedio y la venta mayor. Compara con
`DATE(v.fecha) = p_fecha` para ignorar la hora.

### `sp_clientes_vigentes_trimestre(IN p_anio INT)`

Clientes que compraron entre el 1 de enero y el 31 de marzo, con su número de
pedidos y monto.

Usa un intervalo semiabierto —`>= 1-ene AND < 1-abr`— y no `BETWEEN`. Con
`BETWEEN '2026-01-01' AND '2026-03-31'` sobre un `DATETIME`, la segunda fecha se
interpreta como `2026-03-31 00:00:00` y se perderían todas las ventas de ese día.
En los datos de muestra hay una venta el 31 de marzo a las 19:45 puesta a
propósito para demostrarlo.

### `sp_alta_cliente(...)`

Da de alta un cliente y captura la violación de la restricción única de
`email_cliente`:

```sql
DECLARE EXIT HANDLER FOR 1062
BEGIN
    SET p_codigo  = 1062;
    SET p_mensaje = CONCAT('Ya existe un cliente registrado con el correo ', p_email, '.');
END;
```

Parámetros de salida: `p_id_cliente`, `p_codigo` (0 si todo fue bien), `p_mensaje`.
`clientes.php` traduce `p_codigo = 1062` a un `409` con ese mensaje.

### `trg_producto_no_duplicado_ins` / `_upd`

`BEFORE INSERT` y `BEFORE UPDATE` sobre `PRODUCTO`. Impiden que un mismo
proveedor tenga dos productos con el mismo nombre, lanzando
`SIGNAL SQLSTATE '45000'`.

Lo normal sería un índice `UNIQUE (nombre_producto, id_proveedor)`, pero
entonces el trigger nunca alcanzaría a dispararse: el índice rechazaría la fila
primero.

El trigger de `UPDATE` excluye el propio renglón con `id_producto <>
OLD.id_producto`. Sin esa condición, cada descuento de existencias de una venta
chocaría consigo mismo y **ninguna venta podría registrarse**.

### `trg_venta_seguimiento`

`AFTER INSERT` sobre `VENTA`. Con cada venta de canal `APP` o `WEB` escribe
nombre del cliente, canal, monto, fecha y hora en `SEGUIMIENTO_CLIENTE`. Las de
`MOSTRADOR` no se registran: a ese cliente ya se le atendió en persona.

## Códigos de error

Los errores siempre tienen la forma `{"error": "mensaje"}`. Todos los ejemplos
de abajo son respuestas reales del servidor.

| Código | Cuándo | Ejemplo |
|---|---|---|
| `200` | `GET`, `PUT` o `DELETE` exitoso | |
| `201` | Recurso creado | |
| `400` | Campo inválido, JSON malformado, stock insuficiente, referencia inexistente | `Stock insuficiente para el producto id 4. Disponible: 7, solicitado: 9999.` |
| `401` | Falta `X-API-Key` o no coincide | `API key invalida o ausente. Envie el header X-API-Key.` |
| `404` | El recurso no existe | `Producto no encontrado.` |
| `405` | Método no permitido; la respuesta incluye `Allow: GET, POST, PUT, DELETE` | |
| `409` | Valor único duplicado, o borrado bloqueado por registros que dependen de él | `Ya existe un cliente registrado con el correo ana.lopez@mail.com.` |
| `500` | Error interno | |

Dos de los `409` no los produce PHP sino la base de datos, y el mensaje que ves
es el que escribieron el procedimiento y el trigger:

```
POST /api/clientes.php  con un correo repetido
  → 409  Ya existe un cliente registrado con el correo ana.lopez@mail.com.

POST /api/productos.php con nombre y proveedor repetidos
  → 409  Ese proveedor ya surte un producto con ese nombre.
         No se permiten duplicados en el inventario.
```

## Integración Kotlin

### Dependencias

```kotlin
implementation("com.squareup.retrofit2:retrofit:2.11.0")
implementation("com.squareup.retrofit2:converter-moshi:2.11.0")
implementation("com.squareup.moshi:moshi-kotlin:1.15.1")
implementation("com.squareup.okhttp3:logging-interceptor:4.12.0")
```

### La API key, sin escribirla en el código

`local.properties` (no se sube a git):

```properties
FERRETERIA_API_KEY=46472a83defc364341f42101ee161c6754a8ed78629ab54878844fa0a49bca33
```

`app/build.gradle.kts`:

```kotlin
import java.util.Properties   // necesario: dentro del script, "java" resuelve
                              // a la extensión de Gradle y java.util.Properties
                              // no compila

val ferreteriaApiKey: String = run {
    val archivo = rootProject.file("local.properties")
    if (!archivo.exists()) return@run ""
    val props = Properties()
    archivo.inputStream().use { flujo -> props.load(flujo) }
    props.getProperty("FERRETERIA_API_KEY", "").trim()
}

android {
    buildFeatures { buildConfig = true }
    defaultConfig {
        buildConfigField("String", "FERRETERIA_API_KEY", "\"$ferreteriaApiKey\"")
    }
}
```

### Cliente Retrofit

```kotlin
private const val BASE_URL = "https://ferreteria-api.proxmox-lab.cc/api/"

// Moshi necesita KotlinJsonAdapterFactory explícitamente. Con
// MoshiConverterFactory.create() a secas, un data class de Kotlin sin codegen
// lanza excepción en tiempo de ejecución al parsear: no falla al compilar.
val moshi: Moshi = Moshi.Builder()
    .add(KotlinJsonAdapterFactory())
    .build()

private val interceptorApiKey = Interceptor { chain ->
    chain.proceed(
        chain.request().newBuilder()
            .addHeader("X-API-Key", BuildConfig.FERRETERIA_API_KEY)
            .build()
    )
}

private val log = HttpLoggingInterceptor().apply {
    level = HttpLoggingInterceptor.Level.BODY
    redactHeader("X-API-Key")   // sin esto la key acaba impresa en Logcat
}

private val cliente = OkHttpClient.Builder()
    .addInterceptor(interceptorApiKey)
    .addInterceptor(log)
    .build()

val api: FerreteriaApi = Retrofit.Builder()
    .baseUrl(BASE_URL)
    .client(cliente)
    .addConverterFactory(MoshiConverterFactory.create(moshi))
    .build()
    .create(FerreteriaApi::class.java)
```

### Modelos

Los nombres van en snake_case a propósito: son exactamente las llaves que manda
el backend, así no hace falta anotar nada con `@Json`.

**Los importes son `String`, no `Double`.** En MariaDB son `DECIMAL(10,2)` y PDO
los devuelve como texto. Para operar hay que convertirlos a `BigDecimal`: con
`Double`, `0.1 + 0.2` no da `0.3` y los totales acaban con centavos fantasma.

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
    val proveedor: String,
)

data class ProductoRequest(
    val nombre_producto: String,
    val descripcion_producto: String?,
    val precio_compra: String,
    val precio_venta: String,
    val stock: Int,
    val id_categoria: Int,
    val id_proveedor: Int,
)

data class Cliente(
    val id_cliente: Int,
    val nombre_cliente: String,
    val ap_paterno_cliente: String,
    val ap_materno_cliente: String?,
    val telefono_cliente: String,
    val email_cliente: String,
)

data class Empleado(
    val id_empleado: Int,
    val nombre_empleado: String,
    val ap_paterno_empleado: String,
    val ap_materno_empleado: String?,
    val puesto: String,
    val sueldo: String,
    val telefono_empleado: String,
    val email_empleado: String,
)

data class Categoria(
    val id_categoria: Int,
    val nombre_categoria: String,
    val descripcion: String?,
)

data class Proveedor(
    val id_proveedor: Int,
    val rfc: String,
    val nombre_empresa: String,
    val telefono_proveedor: String,
    val email_proveedor: String,
)

data class DetalleVenta(
    val id_detalle: Int,
    val cantidad: Int,
    val precio_unitario: String,
    val subtotal: String,
    val id_producto: Int,
    val nombre_producto: String,
)

/** La devuelve GET ?id=, POST y PUT. */
data class Venta(
    val id_venta: Int,
    val fecha: String,
    val total: String,
    val canal: String,
    val id_cliente: Int,
    val nombre_cliente: String,
    val ap_paterno_cliente: String,
    val ap_materno_cliente: String?,
    val id_empleado: Int,
    val nombre_empleado: String,
    val ap_paterno_empleado: String,
    val productos: List<DetalleVenta>,
)

/** Renglón del listado: igual que Venta pero sin el detalle. */
data class VentaResumen(
    val id_venta: Int,
    val fecha: String,
    val total: String,
    val canal: String,
    val id_cliente: Int,
    val nombre_cliente: String,
    val ap_paterno_cliente: String,
    val ap_materno_cliente: String?,
    val id_empleado: Int,
    val nombre_empleado: String,
    val ap_paterno_empleado: String,
    val num_productos: Int,
)

data class VentaItemRequest(val id_producto: Int, val cantidad: Int)

data class VentaRequest(
    val id_cliente: Int,
    val id_empleado: Int,
    val canal: String,
    val productos: List<VentaItemRequest>,
)

data class Mensaje(val mensaje: String?)
data class ApiError(val error: String?)
```

Modelos de los reportes:

```kotlin
data class ReporteVentasDia(
    val fecha: String,
    val num_ventas: Int,
    val monto_total: String,
    val ticket_promedio: String,
    val venta_mayor: String,
    val ventas: List<VentaDelDia>,
)

data class VentaDelDia(
    val id_venta: Int,
    val fecha: String,
    val hora: String,
    val canal: String,
    val total: String,
    val id_cliente: Int,
    val cliente: String,
    val empleado: String,
    val renglones: Int,
    val piezas: Int,
)

data class ReporteClientesTrimestre(
    val anio: Int,
    val periodo: String,
    val clientes: List<ClienteVigente>,
)

data class ClienteVigente(
    val id_cliente: Int,
    val cliente: String,
    val email_cliente: String,
    val telefono_cliente: String,
    val pedidos: Int,
    val monto_total: String,
    val primera_compra: String,
    val ultima_compra: String,
    val dias_desde_ultima: Int,
)

data class ProductoTop(
    val id_producto: Int,
    val nombre_producto: String,
    val nombre_categoria: String,
    val piezas_vendidas: Int,
    val importe_vendido: String,
    val aparece_en_ventas: Int,
)

data class PersonaDirectorio(
    val tipo: String,
    val id: Int,
    val nombre: String,
    val telefono: String,
    val email: String,
    val detalle: String,
)

data class SeguimientoCliente(
    val id_seguimiento: Int,
    val id_cliente: Int,
    val nombre_cliente: String,
    val canal: String,
    val total_compra: String,
    val fecha_hora: String,
    val fecha_legible: String,
    val atendido: Int,
    val id_venta: Int?,
)

data class VentasPorCanal(
    val canal: String,
    val num_ventas: Int,
    val monto_total: String,
    val porcentaje: Double,
)
```

### Interfaz

```kotlin
interface FerreteriaApi {

    @GET("productos.php")
    suspend fun getProductos(): Response<List<Producto>>

    @POST("productos.php")
    suspend fun crearProducto(@Body producto: ProductoRequest): Response<Producto>

    @PUT("productos.php")
    suspend fun actualizarProducto(
        @Query("id") id: Int,
        @Body producto: ProductoRequest,
    ): Response<Producto>

    @DELETE("productos.php")
    suspend fun borrarProducto(@Query("id") id: Int): Response<Mensaje>

    // … igual para clientes, empleados, categorias y proveedores

    @GET("ventas.php")
    suspend fun getVentas(): Response<List<VentaResumen>>

    @GET("ventas.php")
    suspend fun getVenta(@Query("id") id: Int): Response<Venta>

    @POST("ventas.php")
    suspend fun crearVenta(@Body venta: VentaRequest): Response<Venta>

    // Los reportes cuelgan todos de reportes.php y se distinguen por "tipo".
    // El tipo se manda desde el repositorio, NO como valor por omisión del
    // parámetro: Retrofit lee la firma por reflexión y los argumentos por
    // omisión de Kotlin generan un método sintético que no reconoce.
    @GET("reportes.php")
    suspend fun getVentasDelDia(
        @Query("tipo") tipo: String,
        @Query("fecha") fecha: String,
    ): Response<ReporteVentasDia>
}
```

### Manejo de errores

```kotlin
val respuesta = api.crearVenta(venta)
if (respuesta.isSuccessful) {
    val nueva = respuesta.body()
} else {
    // respuesta.code() = 400 / 401 / 404 / 405 / 409 / 500
    // respuesta.errorBody()?.string() trae {"error": "..."}
}
```

Conviene extraer el texto de `{"error": "..."}` y mostrarlo: así el mensaje del
procedimiento almacenado o del trigger llega tal cual a la pantalla, en vez de un
texto genérico.

## Despliegue

```
/var/www/ferreteria-api/
├── inc/
│   ├── db.php        ← credenciales reales: NUNCA se sobrescribe al actualizar
│   └── helpers.php
└── public/           ← DocumentRoot de Apache
    └── api/
        ├── productos.php   clientes.php    empleados.php
        ├── categorias.php  proveedores.php ventas.php
        └── reportes.php
```

Instalación de la base, en orden:

```bash
mysql -u root -p < sql/01-esquema.sql
mysql -u root -p ferreteria < sql/02-programabilidad.sql
mysql -u root -p ferreteria < sql/03-datos-muestra.sql
```

El orden importa: los triggers se crean **antes** de cargar los datos, así el de
seguimiento se dispara con las ventas de muestra y la bitácora queda poblada
sola. Eso mismo sirve de prueba de que funciona.

Permisos del usuario de la API:

```sql
GRANT SELECT, INSERT, UPDATE, DELETE ON ferreteria.* TO 'ferreteria_api'@'localhost';
-- Imprescindible: sin EXECUTE fallan todos los CALL, y con ellos los reportes
-- y el alta de clientes.
GRANT EXECUTE ON ferreteria.* TO 'ferreteria_api'@'localhost';
FLUSH PRIVILEGES;
```

Requisitos de PHP: `pdo_mysql`, `bcmath` y `mbstring`.

### Seguridad

- Todo el tráfico va por HTTPS con certificado válido de Cloudflare. No hace
  falta `network_security_config.xml` ni confiar en certificados propios.
- MariaDB escucha solo en `127.0.0.1`: la app nunca se conecta directo a la base,
  siempre pasa por la API.
- Todas las consultas usan *prepared statements*. La app no construye SQL.
- La API key se valida con `hash_equals()` para evitar ataques por tiempo.
- `inc/` queda fuera del DocumentRoot, así que `db.php` no es accesible por web.

---

Backend: Apache + PHP 8.5 (PDO) + MariaDB 11.8 · Túnel: Cloudflare
(`ferreteria-api.proxmox-lab.cc`) · App: Kotlin + Jetpack Compose
