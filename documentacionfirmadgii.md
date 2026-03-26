# Documentación Técnica: Microservicio de Firma DGII

> **Proyecto:** `firmadgii` — Microservicio Node.js/TypeScript para firma digital y transmisión de e-CF a la DGII (República Dominicana)
> **Fecha de análisis:** 2026-03-26
> **Stack:** Node.js 20 + TypeScript + Express.js + librería `dgii-ecf`

---

## PILAR 1: Manual de Uso y Contrato (¿Cómo funciona?)

### 1.1 Arquitectura General

```
SistemaVentaADV  ──►  [POST /api/invoice/sign]  ──►  Microservicio firmadgii
                         ↓ (x-api-key requerida)
                       CertificateService (carga .p12)
                         ↓
                       Signature.signXml()  (librería dgii-ecf)
                         ↓
                       XML firmado + código de seguridad  ──►  Respuesta JSON
```

### 1.2 Autenticación del Cliente

**Todos los endpoints `/api/*` requieren el header:**

```
x-api-key: <tu_api_key_secreta>
```

Los endpoints públicos (`/fe/*`, `/health`) **no requieren autenticación** (son para el protocolo Emisor-Receptor de DGII).

---

### 1.3 Endpoints Expuestos

#### GRUPO A — Endpoints Protegidos (requieren `x-api-key`)

| Método | URL | Descripción |
|--------|-----|-------------|
| `POST` | `/api/auth/dgii` | Autenticar con DGII y obtener token |
| `GET`  | `/api/certificate/info` | Info del certificado cargado |
| `POST` | `/api/invoice/sign` | **Firmar XML sin enviar a DGII** ← _el más usado_ |
| `POST` | `/api/invoice/sign-file` | Firmar XML enviado como archivo raw |
| `POST` | `/api/invoice/send` | Firmar + enviar ECF completo a DGII |
| `POST` | `/api/invoice/send-summary` | Firmar + enviar RFCE (factura de consumo) |
| `POST` | `/api/invoice/send-summary-with-ecf` | Firmar ECF y RFCE juntos, enviar RFCE |
| `GET`  | `/api/invoice/status/:trackId` | Consultar estado de envío |
| `GET`  | `/api/invoice/tracks/:rnc/:encf` | Historial de trackIds por e-NCF |
| `POST` | `/api/invoice/inquire` | Consulta de validez de e-CF |
| `POST` | `/api/invoice/receipt` | Enviar ARECF (acuse de recibo) |
| `POST` | `/api/invoice/approval` | Enviar ACECF (aprobación comercial) |
| `POST` | `/api/invoice/void` | Enviar ANECF (anulación de secuencia) |
| `POST` | `/api/invoice/acecf` | Generar y enviar ACECF desde datos |
| `POST` | `/api/invoice/acecf/from-ecf` | Generar ACECF extrayendo datos del XML del ECF |
| `GET`  | `/api/invoice/customer-directory/:rnc` | Consultar directorio de clientes DGII |
| `GET`  | `/api/invoice/qr/generate` | Generar URL de código QR DGII |
| `POST` | `/api/invoice/receive-ecf-json` | Recibir ECF y generar ARECF (modo JSON) |

#### GRUPO B — Endpoints Públicos (protocolo Emisor-Receptor DGII)

| Método | URL | Descripción |
|--------|-----|-------------|
| `GET`  | `/fe/autenticacion/api/semilla` | Genera semilla XML para autenticación |
| `POST` | `/fe/autenticacion/api/validacioncertificado` | Valida semilla firmada, devuelve token |
| `POST` | `/fe/recepcion/api/ecf` | Recibe ECF de DGII/emisor, responde con ARECF firmado |
| `POST` | `/fe/aprobacioncomercial/api/ecf` | Recibe notificación ACECF |
| `GET`  | `/health` | Health check del servicio |

---

### 1.4 Endpoint Principal: `POST /api/invoice/sign`

> **Este es el endpoint más relevante para SistemaVentaADV.** Recibe un XML sin firmar y devuelve el XML firmado con el código de seguridad.

#### Request

```http
POST /api/invoice/sign
Content-Type: application/json
x-api-key: tu_api_key_aqui

{
  "xmlData": "<ECF>...</ECF>",
  "documentType": "ECF",
  "rnc": "131880978"
}
```

**Campos del body:**

| Campo | Tipo | Requerido | Descripción |
|-------|------|-----------|-------------|
| `xmlData` | `string` | ✅ Sí | XML completo sin firmar |
| `documentType` | `string` | ✅ Sí | Tipo: `ECF`, `RFCE`, `ARECF`, `ACECF`, `ANECF` |
| `rnc` | `string` | ❌ No | RNC del certificado a usar (multi-tenant). Si se omite, usa el certificado por defecto |

#### Respuesta exitosa (HTTP 200)

```json
{
  "success": true,
  "data": {
    "signedXml": "<?xml version=\"1.0\" encoding=\"UTF-8\"?>\n<ECF>\n  ...\n  <Signature xmlns=\"http://www.w3.org/2000/09/xmldsig#\">\n    <SignedInfo>\n      <CanonicalizationMethod Algorithm=\"http://www.w3.org/TR/2001/REC-xml-c14n-20010315\"/>\n      <SignatureMethod Algorithm=\"http://www.w3.org/2001/04/xmldsig-more#rsa-sha256\"/>\n      <Reference URI=\"\">\n        <Transforms>\n          <Transform Algorithm=\"http://www.w3.org/2000/09/xmldsig#enveloped-signature\"/>\n        </Transforms>\n        <DigestMethod Algorithm=\"http://www.w3.org/2001/04/xmlenc#sha256\"/>\n        <DigestValue>BASE64_HASH_AQUI==</DigestValue>\n      </Reference>\n    </SignedInfo>\n    <SignatureValue>BASE64_FIRMA_RSA_AQUI==</SignatureValue>\n    <KeyInfo>\n      <X509Data>\n        <X509Certificate>BASE64_CERT_X509_AQUI=</X509Certificate>\n      </X509Data>\n    </KeyInfo>\n  </Signature>\n</ECF>",
    "securityCode": "123456"
  }
}
```

**Campos de respuesta:**

| Campo | Descripción |
|-------|-------------|
| `success` | `true` en éxito |
| `data.signedXml` | XML completo con `<Signature>` incrustada (enveloped) |
| `data.securityCode` | Código de seguridad de 6 dígitos extraído de la firma (para QR y reportes) |

#### Respuesta de error (HTTP 400/401/500)

```json
{
  "success": false,
  "error": "Error signing XML: descripcion_del_error"
}
```

---

### 1.5 Endpoint: `POST /api/invoice/sign-file`

Alternativa para enviar el XML como archivo raw (binario o texto plano).

```http
POST /api/invoice/sign-file?documentType=ECF&rnc=131880978&download=true
Content-Type: application/xml
x-api-key: tu_api_key_aqui

<?xml version="1.0" encoding="UTF-8"?>
<ECF>
  <Encabezado>...</Encabezado>
</ECF>
```

- Si `download=true` (default): responde con `Content-Disposition: attachment` y el XML firmado como binario.
- Si `download=false`: responde con `Content-Type: application/xml` y el XML firmado como texto.
- Si no se especifica `documentType`, el servicio **detecta automáticamente** el tipo buscando el nodo raíz del XML.

---

### 1.6 Endpoint: `POST /api/invoice/send`

Firma el ECF **y lo envía a DGII** en una sola llamada. Útil si quieres delegar todo el proceso.

#### Request

```http
POST /api/invoice/send
Content-Type: application/json
x-api-key: tu_api_key_aqui

{
  "invoiceData": {
    "ECF": {
      "Encabezado": {
        "Version": "1.0",
        "IdDoc": {
          "TipoeCF": 31,
          "eNCF": "E310000000001",
          "TipoIngresos": "01",
          "TipoPago": 1,
          "FechaLimitePago": "2026-04-26",
          "TotalPaginas": 1
        },
        "Emisor": {
          "RNCEmisor": "131880978",
          "RazonSocialEmisor": "Mi Empresa SRL",
          "DireccionEmisor": "Calle Falsa 123",
          "FechaEmision": "2026-03-26"
        },
        "Comprador": {
          "RNCComprador": "101000000",
          "RazonSocialComprador": "Cliente SRL"
        },
        "Totales": {
          "MontoGravadoTotal": 1000.00,
          "MontoGravadoI1": 1000.00,
          "MontoExento": 0,
          "TotalITBIS": 180.00,
          "TotalITBIS1": 180.00,
          "MontoTotal": 1180.00
        }
      },
      "DetallesItems": {
        "Item": [
          {
            "NumeroLinea": 1,
            "IndicadorBienoServicio": 2,
            "NombreItem": "Servicio de consultoría",
            "CantidadItem": 1,
            "UnidadMedida": 43,
            "PrecioUnitarioItem": 1000.00,
            "TablaSubDescuento": null,
            "DescuentoMonto": 0,
            "TablaSubRecargo": null,
            "RecargoMonto": 0,
            "MontoItem": 1000.00
          }
        ]
      }
    }
  },
  "rnc": "131880978",
  "encf": "E310000000001",
  "environment": "cert"
}
```

#### Respuesta exitosa

```json
{
  "success": true,
  "data": {
    "trackId": "TRK-ABC123XYZ",
    "codigo": "1",
    "estado": "ACEPTADO",
    "signedXml": "<?xml version...",
    "securityCode": "123456",
    "qrCodeUrl": "https://ecf.dgii.gov.do/TesteCF/ConsultaTimbre?RncEmisor=131880978&eNCF=E310000000001&FechaEmision=2026-03-26&MontoTotal=1180.00&FechaFirma=2026-03-26T...&CodigoSeguridad=123456"
  }
}
```

---

### 1.7 Endpoint: `GET /api/invoice/status/:trackId`

```http
GET /api/invoice/status/TRK-ABC123XYZ
x-api-key: tu_api_key_aqui
```

Respuesta:
```json
{
  "success": true,
  "data": {
    "trackId": "TRK-ABC123XYZ",
    "codigo": "1",
    "estado": "ACEPTADO",
    "rnc": "131880978",
    "encf": "E310000000001",
    "fechaRecepcion": "2026-03-26T10:00:00Z",
    "mensajes": []
  }
}
```

---

### 1.8 Endpoint: `POST /api/invoice/inquire`

```http
POST /api/invoice/inquire
Content-Type: application/json
x-api-key: tu_api_key_aqui

{
  "rncEmisor": "131880978",
  "encf": "E310000000001",
  "rncComprador": "101000000",
  "securityCode": "123456"
}
```

---

### 1.9 Ambientes DGII Disponibles

| Valor en API | Ambiente DGII | URL Base |
|-------------|---------------|----------|
| `"test"` | TesteCF (desarrollo) | `https://ecf.dgii.gov.do/TesteCF/...` |
| `"cert"` | CerteCF (certificación) | `https://ecf.dgii.gov.do/CerteCF/...` |
| `"prod"` | eCF (producción) | `https://ecf.dgii.gov.do/eCF/...` |

---

## PILAR 2: Auditoría de Cumplimiento DGII y Seguridad

### 2.1 Estándar Criptográfico — XMLDSig Enveloped con X.509

**Veredicto: ✅ CUMPLE con el estándar XMLDSig Enveloped requerido por la DGII.**

#### Análisis del código de firma

El proceso de firma en `src/services/dgiiService.ts` línea 46-62:

```typescript
// dgiiService.ts:51-54
const certs = certificateService.getCertificate(rnc);
const signature = new Signature(certs.key, certs.cert);
const signedXml = signature.signXml(xmlData, documentType) as string;
const securityCode = getCodeSixDigitfromSignature(signedXml) as string;
```

La clase `Signature` de la librería `dgii-ecf` produce un XML con la siguiente estructura de firma (XMLDSig Enveloped):

```xml
<Signature xmlns="http://www.w3.org/2000/09/xmldsig#">
  <SignedInfo>
    <CanonicalizationMethod Algorithm="http://www.w3.org/TR/2001/REC-xml-c14n-20010315"/>
    <SignatureMethod Algorithm="http://www.w3.org/2001/04/xmldsig-more#rsa-sha256"/>
    <Reference URI="">
      <Transforms>
        <!-- Transform ENVELOPED: firma incrustada dentro del mismo documento -->
        <Transform Algorithm="http://www.w3.org/2000/09/xmldsig#enveloped-signature"/>
      </Transforms>
      <DigestMethod Algorithm="http://www.w3.org/2001/04/xmlenc#sha256"/>
      <DigestValue>HASH_SHA256_BASE64==</DigestValue>
    </Reference>
  </SignedInfo>
  <SignatureValue>FIRMA_RSA_SHA256_BASE64==</SignatureValue>
  <KeyInfo>
    <X509Data>
      <!-- Certificado X.509 del emisor incrustado en la firma -->
      <X509Certificate>CERT_X509_BASE64=</X509Certificate>
    </X509Data>
  </KeyInfo>
</Signature>
```

**Confirmación de cumplimiento:**

| Requisito DGII | Estado |
|---------------|--------|
| `<Signature>` presente | ✅ Sí |
| `<SignedInfo>` con `CanonicalizationMethod` | ✅ Sí |
| `<Reference URI="">` (referencia al documento completo) | ✅ Sí |
| Transform `enveloped-signature` (firma dentro del documento) | ✅ Sí |
| `DigestMethod` SHA-256 | ✅ Sí |
| `<SignatureValue>` RSA-SHA256 | ✅ Sí |
| `<KeyInfo>` con `<X509Certificate>` | ✅ Sí |
| Certificado X.509 incrustado | ✅ Sí |

---

### 2.2 Gestión de la Llave Privada y Contraseña del Certificado

#### ¿Cómo se gestiona? — Análisis de `certificateService.ts`

```typescript
// certificateService.ts:21-39
const reader = new P12Reader(config.certificatePassword);  // ← contraseña de env var

// Opción 1: desde archivo .p12
certs = reader.getKeyFromFile(certificatePath);

// Opción 2: desde Base64 (para cloud sin filesystem)
certs = reader.getKeyFromStringBase64(config.certificateBase64);
```

La contraseña se lee de `config.certificatePassword` que viene de la variable de entorno `CERTIFICATE_PASSWORD`.

**Evaluación de seguridad:**

| Aspecto | Estado | Detalle |
|---------|--------|---------|
| Contraseña en variable de entorno | ✅ Correcto | No está hardcodeada en el código |
| Contraseña en logs | ✅ No se loguea | El logger solo registra rutas y nombres de RNC |
| Contraseña en respuestas HTTP | ✅ No se expone | Nunca retornada al cliente |
| Certificado en respuestas HTTP | ✅ No se expone | Solo se expone en `KeyInfo` del XML firmado (requerido por estándar) |
| Llave privada en memoria | ⚠️ Atención | La llave privada queda en RAM en el objeto `certs.key` mientras dure el proceso |
| Archivo .p12 en filesystem | ⚠️ Atención | El archivo físico debe tener permisos restrictivos (600 en Linux) |
| Certificado Base64 en env var | ⚠️ Atención | Si se usa `CERTIFICATE_BASE64`, asegurar que el `.env` no esté en git |

**Recomendaciones de hardening:**

1. En producción, montar el certificado como volumen Docker read-only (`ro`).
2. Usar un gestor de secretos (AWS Secrets Manager, HashiCorp Vault, Azure Key Vault) para `CERTIFICATE_PASSWORD` en vez de archivo `.env`.
3. Verificar que el archivo `.p12` tiene permisos `chmod 600` y el directorio `certificates/` no es accesible públicamente.
4. Agregar el directorio `certificates/` al `.gitignore` para no commitear accidentalmente los certificados.

---

### 2.3 Concurrencia y Thread-Safety en Producción

#### Análisis del patrón de caché de certificados

```typescript
// certificateService.ts:10 — SINGLETON exportado
export default new CertificateService();

// certificateService.ts:10 — caché compartida entre todas las peticiones
private certificates: Map<string, any> = new Map();
```

El servicio se exporta como **singleton** (`new CertificateService()`), por lo que la instancia única y su `Map` de caché son compartidos por todas las peticiones concurrentes.

**¿Hay problemas de thread-safety?**

> **Node.js es single-threaded** (event loop). No existe paralelismo real en el sentido de múltiples hilos accediendo simultáneamente a la misma memoria. Por lo tanto, **no hay riesgo de race condition** en la caché del `Map`.

Sin embargo, hay un escenario de **cuello de botella** potencial:

```typescript
// dgiiService.ts:50-53 — signXml() crea nueva instancia de Signature por petición
const certs = certificateService.getCertificate(rnc);   // ← caché en Map, O(1)
const signature = new Signature(certs.key, certs.cert); // ← nueva instancia por petición
const signedXml = signature.signXml(xmlData, documentType); // ← operación síncrona RSA
```

**Evaluación de concurrencia:**

| Escenario | Estado | Detalle |
|-----------|--------|---------|
| Caché del certificado (Map) | ✅ Seguro | Node.js event loop garantiza acceso secuencial |
| Instancia `Signature` por petición | ✅ Seguro | Se crea nueva instancia por petición, sin estado compartido |
| Operación `signXml()` es síncrona | ⚠️ Bloqueante | Si hay muchas peticiones simultáneas, la firma RSA bloquea el event loop |
| Llamadas HTTP a DGII (`ecf.authenticate()`) | ✅ Async | Son `await`, no bloquean el event loop |
| Singleton `senderReceiver` (línea 11) | ✅ Seguro | `SenderReceiver` es stateless para firmas individuales |

**Riesgo principal identificado:**

La operación `signature.signXml()` es **síncrona** y computacionalmente intensiva (RSA-SHA256). En un escenario de alta concurrencia (>50 peticiones/segundo), esto puede bloquear el event loop y causar latencia en **todas** las rutas de la aplicación (incluyendo `/health`).

**Solución recomendada para alta carga:**

Ejecutar múltiples instancias del microservicio detrás de un load balancer (ya incluido en el `docker-compose.yml` con Nginx Proxy Manager o Traefik que está en el repositorio). Cada instancia maneja su propio event loop.

---

## PILAR 3: Plan de Acción para SistemaVentaADV

### Clase Cliente C# para Consumir el Microservicio

La siguiente plantilla está lista para copiar y usar en tu proyecto `.NET`. Cubre el caso de uso más común: **enviar un XML sin firmar y recibir el XML firmado**.

```csharp
// FirmaDgiiClient.cs
// Dependencia NuGet requerida: ninguna (usa HttpClient nativo de .NET)
// Compatible con: .NET 6, .NET 7, .NET 8, .NET 9

using System;
using System.Net.Http;
using System.Net.Http.Headers;
using System.Text;
using System.Text.Json;
using System.Text.Json.Serialization;
using System.Threading.Tasks;

namespace SistemaVentaADV.Infrastructure.DGII
{
    /// <summary>
    /// Cliente HTTP para el microservicio de firma DGII (firmadgii).
    /// Registrar como Singleton en el contenedor de inyección de dependencias.
    /// </summary>
    public class FirmaDgiiClient
    {
        private readonly HttpClient _httpClient;
        private readonly FirmaDgiiOptions _options;

        public FirmaDgiiClient(HttpClient httpClient, FirmaDgiiOptions options)
        {
            _httpClient = httpClient ?? throw new ArgumentNullException(nameof(httpClient));
            _options    = options    ?? throw new ArgumentNullException(nameof(options));

            _httpClient.BaseAddress = new Uri(_options.BaseUrl.TrimEnd('/') + "/");
            _httpClient.DefaultRequestHeaders.Add("x-api-key", _options.ApiKey);
            _httpClient.Timeout = TimeSpan.FromSeconds(_options.TimeoutSeconds);
        }

        // ─────────────────────────────────────────────────────────────────
        // OPERACIÓN 1: Firmar XML sin enviar a DGII
        // Endpoint: POST /api/invoice/sign
        // ─────────────────────────────────────────────────────────────────

        /// <summary>
        /// Envía un XML sin firmar y recibe el XML con firma digital XMLDSig Enveloped.
        /// </summary>
        /// <param name="xmlSinFirmar">Contenido XML completo (ECF, RFCE, ARECF, etc.)</param>
        /// <param name="tipoDocumento">Tipo: "ECF", "RFCE", "ARECF", "ACECF", "ANECF"</param>
        /// <param name="rnc">RNC del certificado a usar. Null para el certificado por defecto.</param>
        /// <returns>XML firmado y código de seguridad de 6 dígitos</returns>
        public async Task<FirmaResult> FirmarXmlAsync(
            string xmlSinFirmar,
            string tipoDocumento,
            string? rnc = null)
        {
            var requestBody = new SignXmlRequest
            {
                XmlData      = xmlSinFirmar,
                DocumentType = tipoDocumento,
                Rnc          = rnc
            };

            var json    = JsonSerializer.Serialize(requestBody, JsonOptions.Default);
            var content = new StringContent(json, Encoding.UTF8, "application/json");

            var response = await _httpClient.PostAsync("api/invoice/sign", content);
            var body     = await response.Content.ReadAsStringAsync();

            if (!response.IsSuccessStatusCode)
            {
                var errorObj = JsonSerializer.Deserialize<ErrorResponse>(body, JsonOptions.Default);
                throw new FirmaDgiiException(
                    $"Error al firmar XML [HTTP {(int)response.StatusCode}]: {errorObj?.Error ?? body}",
                    (int)response.StatusCode);
            }

            var result = JsonSerializer.Deserialize<ApiResponse<FirmaData>>(body, JsonOptions.Default)
                ?? throw new FirmaDgiiException("Respuesta vacía del microservicio de firma.");

            return new FirmaResult
            {
                XmlFirmado     = result.Data!.SignedXml,
                CodigoSeguridad = result.Data.SecurityCode
            };
        }

        // ─────────────────────────────────────────────────────────────────
        // OPERACIÓN 2: Firmar XML como archivo raw (útil para XMLs grandes)
        // Endpoint: POST /api/invoice/sign-file
        // ─────────────────────────────────────────────────────────────────

        /// <summary>
        /// Envía el XML como cuerpo raw y recibe el XML firmado directamente (sin JSON wrapper).
        /// Útil cuando el XML ya está generado y se quiere el archivo firmado para descarga.
        /// </summary>
        public async Task<string> FirmarXmlFileAsync(
            string xmlSinFirmar,
            string tipoDocumento,
            string? rnc = null)
        {
            var url = $"api/invoice/sign-file?documentType={Uri.EscapeDataString(tipoDocumento)}&download=false";
            if (!string.IsNullOrWhiteSpace(rnc))
                url += $"&rnc={Uri.EscapeDataString(rnc)}";

            var content = new StringContent(xmlSinFirmar, Encoding.UTF8, "application/xml");

            var response = await _httpClient.PostAsync(url, content);
            var body     = await response.Content.ReadAsStringAsync();

            if (!response.IsSuccessStatusCode)
                throw new FirmaDgiiException(
                    $"Error al firmar archivo XML [HTTP {(int)response.StatusCode}]: {body}",
                    (int)response.StatusCode);

            return body; // XML firmado como string
        }

        // ─────────────────────────────────────────────────────────────────
        // OPERACIÓN 3: Enviar ECF completo a DGII (firma + envío en un paso)
        // Endpoint: POST /api/invoice/send
        // ─────────────────────────────────────────────────────────────────

        /// <summary>
        /// Firma el ECF y lo envía a DGII. Devuelve el trackId para rastrear el estado.
        /// </summary>
        public async Task<EnvioResult> EnviarEcfAsync(
            object invoiceData,
            string rnc,
            string encf,
            string environment = "cert")
        {
            var requestBody = new
            {
                invoiceData,
                rnc,
                encf,
                environment
            };

            var json    = JsonSerializer.Serialize(requestBody, JsonOptions.Default);
            var content = new StringContent(json, Encoding.UTF8, "application/json");

            var response = await _httpClient.PostAsync("api/invoice/send", content);
            var body     = await response.Content.ReadAsStringAsync();

            if (!response.IsSuccessStatusCode)
            {
                var errorObj = JsonSerializer.Deserialize<ErrorResponse>(body, JsonOptions.Default);
                throw new FirmaDgiiException(
                    $"Error al enviar ECF [HTTP {(int)response.StatusCode}]: {errorObj?.Error ?? body}",
                    (int)response.StatusCode);
            }

            var result = JsonSerializer.Deserialize<ApiResponse<EnvioData>>(body, JsonOptions.Default)
                ?? throw new FirmaDgiiException("Respuesta vacía al enviar ECF.");

            return new EnvioResult
            {
                TrackId        = result.Data!.TrackId,
                Codigo         = result.Data.Codigo,
                Estado         = result.Data.Estado,
                XmlFirmado     = result.Data.SignedXml,
                CodigoSeguridad = result.Data.SecurityCode,
                QrCodeUrl      = result.Data.QrCodeUrl
            };
        }

        // ─────────────────────────────────────────────────────────────────
        // OPERACIÓN 4: Consultar estado de un envío
        // Endpoint: GET /api/invoice/status/:trackId
        // ─────────────────────────────────────────────────────────────────

        /// <summary>
        /// Consulta el estado de procesamiento de un e-CF en DGII por su trackId.
        /// </summary>
        public async Task<EstadoResult> ConsultarEstadoAsync(string trackId)
        {
            var response = await _httpClient.GetAsync($"api/invoice/status/{Uri.EscapeDataString(trackId)}");
            var body     = await response.Content.ReadAsStringAsync();

            if (!response.IsSuccessStatusCode)
                throw new FirmaDgiiException(
                    $"Error al consultar estado [HTTP {(int)response.StatusCode}]: {body}",
                    (int)response.StatusCode);

            var result = JsonSerializer.Deserialize<ApiResponse<EstadoData>>(body, JsonOptions.Default)
                ?? throw new FirmaDgiiException("Respuesta vacía al consultar estado.");

            return new EstadoResult
            {
                TrackId        = result.Data!.TrackId,
                Codigo         = result.Data.Codigo,
                Estado         = result.Data.Estado,
                Rnc            = result.Data.Rnc,
                Encf           = result.Data.Encf,
                FechaRecepcion = result.Data.FechaRecepcion
            };
        }

        // ─────────────────────────────────────────────────────────────────
        // OPERACIÓN 5: Información del certificado
        // Endpoint: GET /api/certificate/info
        // ─────────────────────────────────────────────────────────────────

        /// <summary>
        /// Obtiene información del certificado digital cargado en el microservicio.
        /// Útil para monitorear la fecha de vencimiento.
        /// </summary>
        public async Task<CertificadoInfo> ObtenerInfoCertificadoAsync(string? rnc = null)
        {
            var url = "api/certificate/info";
            if (!string.IsNullOrWhiteSpace(rnc))
                url += $"?rnc={Uri.EscapeDataString(rnc)}";

            var response = await _httpClient.GetAsync(url);
            var body     = await response.Content.ReadAsStringAsync();

            if (!response.IsSuccessStatusCode)
                throw new FirmaDgiiException(
                    $"Error al obtener info del certificado [HTTP {(int)response.StatusCode}]: {body}",
                    (int)response.StatusCode);

            var result = JsonSerializer.Deserialize<ApiResponse<CertificadoInfo>>(body, JsonOptions.Default)
                ?? throw new FirmaDgiiException("Respuesta vacía al obtener info del certificado.");

            return result.Data!;
        }
    }

    // =========================================================================
    // MODELOS DE DATOS
    // =========================================================================

    public class FirmaDgiiOptions
    {
        /// <summary>URL base del microservicio. Ej: "http://localhost:3000" o "http://firma-dgii:3000"</summary>
        public string BaseUrl { get; set; } = "http://localhost:3000";

        /// <summary>API Key configurada en el microservicio (variable DGII_API_KEY)</summary>
        public string ApiKey { get; set; } = string.Empty;

        /// <summary>Timeout en segundos para las peticiones HTTP</summary>
        public int TimeoutSeconds { get; set; } = 30;
    }

    public class FirmaResult
    {
        public string XmlFirmado      { get; set; } = string.Empty;
        public string CodigoSeguridad { get; set; } = string.Empty;
    }

    public class EnvioResult
    {
        public string TrackId         { get; set; } = string.Empty;
        public string? Codigo         { get; set; }
        public string? Estado         { get; set; }
        public string? XmlFirmado     { get; set; }
        public string? CodigoSeguridad { get; set; }
        public string? QrCodeUrl      { get; set; }
    }

    public class EstadoResult
    {
        public string TrackId            { get; set; } = string.Empty;
        public string? Codigo            { get; set; }
        public string? Estado            { get; set; }
        public string? Rnc               { get; set; }
        public string? Encf              { get; set; }
        public DateTimeOffset? FechaRecepcion { get; set; }
    }

    public class CertificadoInfo
    {
        [JsonPropertyName("subject")]
        public string? Subject { get; set; }

        [JsonPropertyName("issuer")]
        public string? Issuer { get; set; }

        [JsonPropertyName("validFrom")]
        public string? ValidFrom { get; set; }

        [JsonPropertyName("validTo")]
        public string? ValidTo { get; set; }

        [JsonPropertyName("serialNumber")]
        public string? SerialNumber { get; set; }

        [JsonPropertyName("fingerprint")]
        public string? Fingerprint { get; set; }
    }

    // ─── Modelos internos para deserialización JSON ────────────────────────

    internal class SignXmlRequest
    {
        [JsonPropertyName("xmlData")]
        public string XmlData { get; set; } = string.Empty;

        [JsonPropertyName("documentType")]
        public string DocumentType { get; set; } = string.Empty;

        [JsonPropertyName("rnc")]
        public string? Rnc { get; set; }
    }

    internal class ApiResponse<T>
    {
        [JsonPropertyName("success")]
        public bool Success { get; set; }

        [JsonPropertyName("data")]
        public T? Data { get; set; }

        [JsonPropertyName("error")]
        public string? Error { get; set; }
    }

    internal class FirmaData
    {
        [JsonPropertyName("signedXml")]
        public string SignedXml { get; set; } = string.Empty;

        [JsonPropertyName("securityCode")]
        public string SecurityCode { get; set; } = string.Empty;
    }

    internal class EnvioData
    {
        [JsonPropertyName("trackId")]
        public string TrackId { get; set; } = string.Empty;

        [JsonPropertyName("codigo")]
        public string? Codigo { get; set; }

        [JsonPropertyName("estado")]
        public string? Estado { get; set; }

        [JsonPropertyName("signedXml")]
        public string? SignedXml { get; set; }

        [JsonPropertyName("securityCode")]
        public string? SecurityCode { get; set; }

        [JsonPropertyName("qrCodeUrl")]
        public string? QrCodeUrl { get; set; }
    }

    internal class EstadoData
    {
        [JsonPropertyName("trackId")]
        public string TrackId { get; set; } = string.Empty;

        [JsonPropertyName("codigo")]
        public string? Codigo { get; set; }

        [JsonPropertyName("estado")]
        public string? Estado { get; set; }

        [JsonPropertyName("rnc")]
        public string? Rnc { get; set; }

        [JsonPropertyName("encf")]
        public string? Encf { get; set; }

        [JsonPropertyName("fechaRecepcion")]
        public DateTimeOffset? FechaRecepcion { get; set; }
    }

    internal class ErrorResponse
    {
        [JsonPropertyName("error")]
        public string? Error { get; set; }

        [JsonPropertyName("message")]
        public string? Message { get; set; }
    }

    internal static class JsonOptions
    {
        public static readonly JsonSerializerOptions Default = new()
        {
            PropertyNamingPolicy        = JsonNamingPolicy.CamelCase,
            DefaultIgnoreCondition      = JsonIgnoreCondition.WhenWritingNull,
            PropertyNameCaseInsensitive = true
        };
    }

    // ─── Excepción personalizada ──────────────────────────────────────────

    public class FirmaDgiiException : Exception
    {
        public int StatusCode { get; }

        public FirmaDgiiException(string message, int statusCode = 500)
            : base(message) => StatusCode = statusCode;
    }
}
```

---

### 3.1 Registro en el Contenedor de DI (Program.cs)

```csharp
// Program.cs — Registro como Singleton (HttpClient se maneja con IHttpClientFactory)

builder.Services.Configure<FirmaDgiiOptions>(
    builder.Configuration.GetSection("FirmaDgii"));

builder.Services.AddHttpClient<FirmaDgiiClient>((serviceProvider, httpClient) =>
{
    var options = serviceProvider
        .GetRequiredService<IOptions<FirmaDgiiOptions>>().Value;

    httpClient.BaseAddress = new Uri(options.BaseUrl.TrimEnd('/') + "/");
    httpClient.DefaultRequestHeaders.Add("x-api-key", options.ApiKey);
    httpClient.Timeout = TimeSpan.FromSeconds(options.TimeoutSeconds);
});
```

### 3.2 Configuración en `appsettings.json`

```json
{
  "FirmaDgii": {
    "BaseUrl": "http://localhost:3000",
    "ApiKey": "tu_api_key_secreta_aqui",
    "TimeoutSeconds": 30
  }
}
```

> **Seguridad:** En producción, almacena `ApiKey` en **User Secrets** (`dotnet user-secrets`) o en Azure Key Vault / AWS Secrets Manager. Nunca la incluyas en el código fuente ni en `appsettings.json` commiteado.

---

### 3.3 Ejemplo de Uso en un Servicio de Negocio

```csharp
// EcfService.cs — Ejemplo de uso del cliente en SistemaVentaADV

public class EcfService
{
    private readonly FirmaDgiiClient _firmaDgii;
    private readonly ILogger<EcfService> _logger;

    public EcfService(FirmaDgiiClient firmaDgii, ILogger<EcfService> logger)
    {
        _firmaDgii = firmaDgii;
        _logger    = logger;
    }

    /// <summary>
    /// Flujo completo: genera XML → firma → guarda XML firmado → devuelve código de seguridad
    /// </summary>
    public async Task<string> FirmarYGuardarFacturaAsync(string xmlSinFirmar, string rnc)
    {
        try
        {
            _logger.LogInformation("Enviando XML para firma al microservicio DGII...");

            var resultado = await _firmaDgii.FirmarXmlAsync(
                xmlSinFirmar: xmlSinFirmar,
                tipoDocumento: "ECF",
                rnc: rnc
            );

            _logger.LogInformation(
                "XML firmado correctamente. Código de seguridad: {Codigo}",
                resultado.CodigoSeguridad);

            // Aquí guardas resultado.XmlFirmado en tu base de datos o sistema de archivos
            await GuardarXmlFirmadoAsync(resultado.XmlFirmado);

            return resultado.CodigoSeguridad;
        }
        catch (FirmaDgiiException ex) when (ex.StatusCode == 401)
        {
            _logger.LogError("API Key inválida o expirada para el microservicio de firma.");
            throw;
        }
        catch (FirmaDgiiException ex)
        {
            _logger.LogError(ex, "Error al firmar el XML en el microservicio DGII.");
            throw;
        }
        catch (HttpRequestException ex)
        {
            _logger.LogError(ex, "No se pudo conectar al microservicio de firma DGII. Verificar que esté corriendo.");
            throw;
        }
    }

    private Task GuardarXmlFirmadoAsync(string xmlFirmado)
    {
        // Tu lógica para persistir el XML firmado
        throw new NotImplementedException();
    }
}
```

---

### 3.4 Decisión de Diseño: ¿Cuándo usar `/sign` vs `/send`?

| Situación | Endpoint recomendado | Razón |
|-----------|---------------------|-------|
| Quiero controlar el envío a DGII yo mismo desde SistemaVentaADV | `POST /api/invoice/sign` | Solo firma, no envía. Tú manejas el envío. |
| Quiero delegar firma + envío al microservicio en un solo paso | `POST /api/invoice/send` | El microservicio autentica, firma, envía y devuelve trackId. |
| Tengo el XML ya generado como archivo/string y quiero el XML firmado como descarga | `POST /api/invoice/sign-file` | Más simple para flujos batch. |
| Facturas de consumo (tipo 32, RFCE) | `POST /api/invoice/send-summary` o `send-summary-with-ecf` | Específico para RFCE. |

---

## Apéndice: Variables de Entorno del Microservicio

```bash
# Requeridas
PORT=3000
NODE_ENV=production
CERTIFICATE_PATH=./certificates/certificado.p12  # o nombre_rnc.p12 para multi-tenant
CERTIFICATE_PASSWORD=contraseña_del_p12
DGII_ENVIRONMENT=cert                            # test | cert | prod
API_KEY=api_key_muy_segura_y_larga               # La que usas en x-api-key

# Opcionales
LOG_LEVEL=info
RNC_RECEPTOR=131880978                           # Para el endpoint Emisor-Receptor
ODOO_WEBHOOK_URL=http://odoo/webhook             # Notificaciones Odoo
ODOO_WEBHOOK_API_KEY=odoo_api_key
CERTIFICATE_BASE64=base64_del_p12               # Alternativa a archivo .p12 (cloud)
```

---

## Apéndice: Documentación Interactiva (Swagger)

El microservicio expone documentación interactiva en:

- **UI:** `http://localhost:3000/api-docs`
- **JSON Spec:** `http://localhost:3000/api-docs.json`

Desde ahí puedes probar todos los endpoints directamente con tu API key.
