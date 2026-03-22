# Progetto: API Gateway con Go su GCP

## Panoramica del Progetto

Sistema di API Gateway written in Go, deployato su Google Cloud Platform utilizzando Cloud Run. L'architettura prevede un servizio dedicato per funzionalità (es. auth-service, user-service, order-service, etc.), comunicazione tramite HTTP/REST, autenticazione centralizzata con Firebase, database NoSQL Firestore e storage per file su Google Cloud Storage.

## Stack Tecnologico

### Backend

- **Linguaggio**: Go 1.21+
- **Framework**: Fiber o Gin (HTTP per comunicazione interna ed esterna)
- **Database**: Firestore (Google Cloud Firestore)
- **Storage**: Google Cloud Storage
- **Autenticazione**: Firebase Auth
- **Deploy**: Cloud Run (Google Cloud Platform)

### Frontend

- **Framework**: React o Angular
- **Hosting**: Firebase Hosting
- **Autenticazione**: Firebase Auth (client SDK)

## Architettura

```
┌─────────────────┐
│   Frontend     │  (React/Angular su Firebase Hosting)
│  (SPA Static)  │
└────────┬────────┘
          │ HTTPS
          ▼
┌─────────────────────────────────────┐
│      Cloudflare WAF                 │  (Protezione DDoS, Bot, SQLi, XSS)
└────────┬────────────────────────────┘
          │
          ▼
┌─────────────────┐
│  API Gateway   │  (Go su Cloud Run)
│   (Go + Gin)   │
└────────┬────────┘
          │ HTTP/REST
          ▼
┌─────────────────────────────────────┐
│         Microservices               │
├─────────────────────────────────────┤
│  • Auth Service                     │
│  • User Service                     │
│  • Order Service                    │
│  • Product Service                  │
│  • Payment Service                  │
│  • Notification Service             │
└────────┬──────────────────┬────────┘
          │                  │
          ▼                  ▼
┌─────────────────┐  ┌─────────────────┐
│   Firestore     │  │   Cloud Storage │
│   (Database)    │  │   (File Store)  │
└─────────────────┘  └─────────────────┘
```

## Servizi Backend

### API Gateway

- Punto di ingresso unico per tutte le richieste
- Rate limiting per proteggere i servizi downstream
- Request routing basato su path
- Validazione JWT token Firebase
- Logging e monitoring centralizzato
- Circuit breaker per resilienza
- Comunicazione con i microservizi tramite HTTP/REST

### Auth Service

- Verifica token Firebase
- Gestione sessioni
- Ruoli e permessi utente
- Middleware di autenticazione per altri servizi

### User Service

- CRUD utenti
- Profilo utente
- Preferenze e impostazioni
- Indirizzi e informazioni di shipping

### Order Service

- Gestione ordini
- Stati ordine (pending, paid, shipped, delivered, cancelled)
- Cronologia ordini
- Integrazione pagamenti

### Product Service

- Catalogo prodotti
- Categorie e tag
- Ricerca e filtri
- Inventario

### Payment Service

- Integrazione payment gateway
- Gestione pagamenti
- Rimborsi
- Ricevute/fatture

### Notification Service

- Email notifications
- Push notifications
- SMS (opzionale)
- Template notifications

## Service Discovery and Communication Contract

### Service Registration

Each microservice must expose a discovery endpoint (`GET /_discover`) accessible only from the API Gateway (via Cloud Run ingress rules). This endpoint returns a JSON document describing the service:

```json
{
  "serviceName": "string",
  "version": "string (semver)",
  "metadata": {
    // arbitrary key-value pairs for service-specific configuration
  },
  "endpoints": [
    {
      "path": "/api/v1/resource",
      "method": "GET|POST|PUT|DELETE|PATCH",
      "summary": "Brief description of endpoint purpose",
      "inputSchema": {
        // JSON Schema describing valid request body/query parameters
        // Optional if endpoint takes no input
      },
      "outputSchema": {
        // JSON Schema describing successful response body
        // Optional for endpoints with no meaningful response
      },
      "authRequired": boolean,
      "rateLimit": {
        "requestsPerMinute": integer,
        "burst": integer
      }
    }
  ]
}
```

### Gateway Responsibilities

1. **Service Discovery**: On startup and periodically (configurable interval), the gateway queries each registered service's `/_discover` endpoint.
2. **Dynamic Route Registration**: For each endpoint discovered, the gateway creates an external-facing route that proxies to the service.
3. **Request/Response Validation**: If schemas are provided, the gateway validates incoming requests against `inputSchema` and outgoing responses against `outputSchema`.
4. **Error Handling**: Transforms service errors into standardized HTTP error responses.
5. **Security**: Enforces authentication requirements and applies rate limits as specified.

### Best Practices

- **Versioning**: Services should include semantic version in discovery to allow graceful evolution.
- **Idempotency**: All endpoints should be designed to be idempotent where appropriate (PUT, DELETE, safe POSTs).
- **Standard Error Format**: Services should return errors in the project's standard format:
  ```json
  {
    "success": false,
    "error": {
      "code": "ERROR_CODE",
      "message": "Human readable message",
      "details": { /* optional additional info */ }
    }
  }
  ```
- **Health Checks**: Services should also expose a `/healthz` endpoint returning 200 when healthy.
- **Security**: Discovery endpoint should be restricted to the gateway's service account or IP range via Cloud Run ingress settings.
- **Schema Evolution**: Use backward-compatible changes to schemas; version endpoints when breaking changes are necessary.

This approach allows the gateway to remain agnostic of specific service implementations while providing a typed, validated interface to external consumers.

## Struttura Progetti Go

```
 /api-gateway
 ├── cmd/
 │   └── gateway/
 │       └── main.go
 ├── internal/
 │   ├── config/
 │   ├── handlers/
 │   ├── middleware/
 │   ├── models/
 │   ├── services/
 │   └── repository/
 ├── pkg/
 ├── go.mod
 └── Dockerfile

 /auth-service
 ├── cmd/
 │   └── auth/
 │       └── main.go
 ├── internal/
 ├── go.mod
 └── Dockerfile
```

## API Design

### Convenzioni REST

- **GET** - Lettura risorse
- **POST** - Creazione risorse
- **PUT** - Aggiornamento completo
- **PATCH** - Aggiornamento parziale
- **DELETE** - Eliminazione risorse

### Formato Response

```json
{
  "success": true,
  "data": { },
  "meta": {
    "page": 1,
    "limit": 20,
    "total": 100
  }
}
```

### Formato Errori

```json
{
  "success": false,
  "error": {
    "code": "ERR_CODE",
    "message": "Descrizione errore",
    "details": { }
  }
}
```

### Codici HTTP

- `200` - OK
- `201` - Created
- `400` - Bad Request
- `401` - Unauthorized
- `403` - Forbidden
- `404` - Not Found
- `429` - Too Many Requests
- `500` - Internal Server Error
- `503` - Service Unavailable

## Autenticazione

### Flusso

1. Frontend effettua login con Firebase (Google, Email/Password, etc.)
2. Firebase restituisce JWT token
3. Frontend invia richieste con header `Authorization: Bearer <token>`
4. API Gateway verifica token con Firebase
5. Token validato → richiesta processata
6. Token invalido → 401 Unauthorized

### Ruoli

- `admin` - Accesso completo
- `user` - Utente autenticato
- `guest` - Accesso pubblico (solo lettura)

## Database Firestore

### Collections

```
users/
  └── {userId}
    ├── email
    ├── displayName
    ├── photoURL
    ├── role
    ├── createdAt
    └── updatedAt

orders/
  └── {orderId}
    ├── userId
    ├── status
    ├── items[]
    ├── total
    ├── shippingAddress
    ├── createdAt
    └── updatedAt

products/
  └── {productId}
    ├── name
    ├── description
    ├── price
    ├── category
    ├── images[]
    ├── stock
    └── createdAt
```

## Cloud Storage

### Buckets

- `project-images/` - Immagini prodotti
- `user-avatars/` - Avatar utenti
- `documents/` - Documenti uploadati
- `exports/` - Export dati

## CI/CD

### GitHub Actions

```yaml
# Trigger su push a main
# 1. Run tests
# 2. Build Docker image
# 3. Push to GCR
# 4. Deploy to Cloud Run
# 5. Run smoke tests
```

## Monitoring e Logging

- **Logging**: Cloud Logging
- **Metrics**: Cloud Monitoring
- **Tracing**: Cloud Trace
- **Alerting**: Cloud Alerts

## Sicurezza

### Cloudflare WAF (gestito esternamente)

Il traffico è protetto da Cloudflare WAF che fornisce:
- Protezione DDoS
- Block list automatica IP malevoli
- SQL injection protection
- XSS protection
- Rate limiting (a livello WAF)
- Bot management
- Geo-blocking

### Applicazione

- HTTPS sempre (terminato da Cloudflare)
- CORS configurato correttamente (gestito dal Gateway)
- Input validation lato applicazione
- Firestore security rules
- Secrets in Secret Manager
- IAM roles minimali
- Token Firebase validation

## Todo

- [ ] Progettare architettura dettagliata
- [ ] Definire specifice API REST
- [ ] Definire contratto di service discovery e comunicazione
- [ ] Creare repository Go base
- [ ] Implementare API Gateway con HTTP/REST
- [ ] Implementare Auth Service
- [ ] Implementare User Service
- [ ] Implementare Order Service
- [ ] Implementare Product Service
- [ ] Configurare CI/CD
- [ ] Setup Firebase project
- [ ] Setup Cloud Run
- [ ] Setup Firestore
- [ ] Setup Cloud Storage
- [ ] Sviluppare frontend

## Note

- Un servizio per container Cloud Run
- Comunicazione interna ed esterna tramite HTTP/REST
- Tutti i servizi devono essere idempotenti
- Implementare health check per Cloud Run
- Usare context Go per timeout e cancellazione
- Cloud Run accessibile solo da Cloudflare (allowlist IP Cloudflare)
- Configurare WAF rules su Cloudflare dashboard
- Bypassare Cloudflare solo per health checks interni
