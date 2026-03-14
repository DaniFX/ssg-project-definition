# Progetto: API Gateway con Go su GCP

## Panoramica del Progetto

Sistema di API Gateway written in Go, deployato su Google Cloud Platform utilizzando Cloud Run. L'architettura prevede un servizio dedicato per funzionalità (es. auth-service, user-service, order-service, etc.), comunicazione tramite gRPC, autenticazione centralizzata con Firebase, database NoSQL Firestore e storage per file su Google Cloud Storage.

## Stack Tecnologico

### Backend

- **Linguaggio**: Go 1.21+
- **Framework**: Fiber o Gin (HTTP), gRPC (comunicazione interna)
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
         │ gRPC
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
├── proto/
├── pkg/
├── go.mod
└── Dockerfile

/auth-service
├── cmd/
│   └── auth/
│       └── main.go
├── internal/
├── proto/
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
- [ ] Definire proto files per gRPC
- [ ] Creare repository Go base
- [ ] Implementare API Gateway
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
- Comunicazione interna in gRPC
- Comunicazione esterna via REST attraverso Gateway
- Tutti i servizi devono essere idempotenti
- Implementare health check per Cloud Run
- Usare context Go per timeout e cancellazione
- Cloud Run accessibile solo da Cloudflare (allowlist IP Cloudflare)
- Configurare WAF rules su Cloudflare dashboard
- Bypassare Cloudflare solo per health checks interni
