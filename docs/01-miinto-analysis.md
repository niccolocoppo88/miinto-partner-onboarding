# Miinto — Company & Partner Analysis

## Company Profile

**Miinto Group** è una piattaforma fashion e-commerce Scandinava, tra le più grandi d'Europa.

### Key Facts
| Dato | Valore |
|------|--------|
| **Fondazione** | 2009, Copenhagen (DK) |
| **Fondatori** | Konrad Kierklo, Mike J. |
| **Headquarters** | Copenhagen, Denmark |
| **Presenza** | DK, NL, BE, UK, IT, US |
| **Partner Shops** | 600+ retailer indipendenti / 1,800+ boutique |
| **Prodotti** | 400,000+ SKUs |
| **Brand** | 5,000+ |
| **Modello** | Marketplace aggregator (non inventory) |
| **Investitore** | Burda Principal Investments |

### Business Model
Miinto funziona come **aggregatore di boutique fashion indipendenti**:
- Partner = negozi fashion indipendenti (boutique)
- Miinto = vetrina unificata per il consumatore finale
- Ordini processati dai partner, logistica gestita esternamente
- Miinto trattiene commissione per transazione

---

## Partner Types

### 1. Boutique Fashion Indipendenti
- Negozi fisici con presenza online limitata
- Tipicamente 1-10 dipendenti
- ERP/POS eterogenei (LightSpeed, Shopify, custom)
- Necessità di integrazione semplificata

### 2. Brand Direct-to-Consumer (DTC)
- Brand affermati che vogliono reach oltre mercato locale
- Maggiori volumi, sistemi più sofisticati
- Richiedono gestione multi-canale

### 3. Grossisti/Distributori
- Inventory più ampia, prezzi wholesale
- Integrazione più complessa (mapping prodotti, prezzi)

---

## API Ecosystem — Partner Order API Service (POAS)

### Architecture Overview
```
┌─────────────────────────────────────────────────────────┐
│                    PARTNER (Boutique)                    │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌──────────┐  │
│  │   POS   │  │   ERP   │  │  Store  │  │  Custom  │  │
│  └────┬────┘  └────┬────┘  └────┬────┘  └─────┬────┘  │
│       └───────────┴───────────┴───────────────┘       │
│                          │                              │
│                          ▼                              │
│               ┌──────────────────┐                      │
│               │   PARTNER APP    │                      │
│               └────────┬─────────┘                      │
└────────────────────────┼────────────────────────────────┘
                         │
                    POAS REST API
                         │
┌────────────────────────┼────────────────────────────────┐
│                        ▼                                │
│               ┌──────────────────┐                      │
│               │   MIINTO CORE    │                      │
│               └──────────────────┘                      │
└──────────────────────────────────────────────────────────┘
```

### POAS Endpoints

#### 1. Authentication
```
POST /auth
Request: { username, password }
Response: { token, partner_id, expiration }
```
- Miinto fornisce credenziali al partner
- Token JWT con scadenza configurabile
- Refresh token per sessioni lunghe

#### 2. Signature Creation
```
POST /signature
Request: { order_id, amount, timestamp }
Response: { signature_hash }
```
- HMAC-SHA256 per integrità messaggi
- Prevents order manipulation

#### 3. Live Stock API (LSA)
```
POST /stock/update
Request: { articles: [{ sku, quantity, price, size }] }
Response: { updated_count, errors }
```
- Real-time inventory sync
- Supporta bulk updates (fino a 1000 items/request)
- 5-15 min latency target

#### 4. Orders Acceptance
```
POST /orders/{id}/accept
POST /orders/{id}/reject
Request: { positions: [{ sku, qty }], reason? }
Response: { status, eta? }
```
- Pending → Accepted/Rejected
- Partial acceptance supportata
- 30-min timeout per default

#### 5. Shipping Booking
```
POST /orders/{id}/ship
Request: { carrier, tracking_number, pickup_date }
Response: { booking_id, label_url }
```
- Integrazione carrier (DHL, UPS, etc.)
- Miinto fornisce etichette pre-pagate

#### 6. Returns Acceptance
```
POST /returns/{id}/accept
POST /returns/{id}/reject
Request: { reason, condition_check }
Response: { rma_id, return_label }
```

### Miinto Communication Channel (MCC)
- Sistema di messaggistica interno
- Notifiche push per ordini/returns
- Status updates automatici
- Ticket support integrato

### New Transactional System (NTS) — Migration 2025
**Deadline: End of 2025**

Tutti i partner DEVONO completare la migrazione a NTS che porta:
- API v2 con breaking changes
- Improved real-time stock
- Better order management
- Enhanced returns flow

---

## Current Pain Points (Onboarding Esistente)

### Problemi Identificati

1. **Manual Onboarding**
   - Partner onboarding gestito via email
   - Tempi medi: 2-4 settimane
   - Nessuno self-service disponibile

2. **Document Handling**
   - Contratti via email/PDF
   - Firma manuale o scanner
   - nessuna traccia audit

3. **Technical Integration**
   - POAS documentation frammentata
   - No sandbox/test environment pubblico
   - Supporto tecnico via ticket

4. **KYC/Compliance**
   - Verifiche manuali
   - tempi di attesa lunghi
   - Feedback non in tempo reale

5. **Training**
   - No self-paced training
   - Video calls con account manager
   - Documentazione sparsi

---

## Strategic Importance

### Why This Project Matters

1. **Scalability**: Onboarding manuale = bottleneck crescita
2. **Time-to-Revenue**: Partner aspettano 2-4 settimane = perdite
3. **Error Rate**: Integrazione manuale = errori frequenti
4. **Partner Satisfaction**: Competitor (Zalando) offrono self-service
5. **NTS Migration**: Necessità di migrate 600+ partner entro 2025

### Target Metrics
| KPI | Current | Target |
|-----|---------|--------|
| Onboarding Time | 2-4 weeks | 24-48 hours |
| Contract Signing | 3-5 days | < 1 hour |
| First Order | 3-4 weeks | 1 week |
| Support Tickets | High volume | Low (self-service) |
| Integration Success Rate | ~70% | >95% |
