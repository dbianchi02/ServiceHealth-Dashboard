# Microsoft 365 Service Health Dashboard

> Dashboard web per il monitoraggio in tempo reale dello stato dei servizi Microsoft 365, incident attivi e degradazioni operative. Progettata per sysadmin M365 che vogliono una vista centralizzata senza consultare manualmente il portale Microsoft 365 Admin Center.

<img width="1873" height="849" alt="image" src="https://github.com/user-attachments/assets/b4f58f61-8b4f-4695-b2d3-b8a05fa89ee1" />

---

## Indice

- [Architettura](#architettura)
- [Prerequisiti](#prerequisiti)
- [Struttura repository](#struttura-repository)
- [Setup step-by-step](#setup-step-by-step)
  - [1. App Registration — Graph API](#1-app-registration--graph-api)
  - [2. Azure Function App](#2-azure-function-app)
  - [3. Azure Static Web App](#3-azure-static-web-app)
  - [4. Autenticazione Entra ID (Easy Auth)](#4-autenticazione-entra-id-easy-auth)
  - [5. CORS](#5-cors)
- [Script e file](#script-e-file)
- [Endpoint Graph API utilizzati](#endpoint-graph-api-utilizzati)
- [Funzionalità dashboard](#funzionalità-dashboard)
- [Troubleshooting](#troubleshooting)
- [Limitazioni e sicurezza](#limitazioni-e-sicurezza)
- [Roadmap](#roadmap)

---

## Architettura

```
Browser (index.html)
        │
        │  — Richiede login Microsoft (Easy Auth) se non autenticato
        ▼
Azure Static Web Apps (Easy Auth — Entra ID)
        │
        │  fetch /api/GetServiceHealth?code=<function-key>
        ▼
Azure Function App (PowerShell HTTP Trigger)
        │
        │  POST /oauth2/v2.0/token  →  access_token
        ▼
Microsoft Entra ID (App Registration — Graph API)
        │
        │  GET /admin/serviceAnnouncement/healthOverviews
        │  GET /admin/serviceAnnouncement/issues
        ▼
Microsoft Graph API
        │
        │  JSON { health: [...], issues: [...] }
        ▼
Azure Function (serializza e restituisce)
        │
        ▼
Browser (rendering dashboard)
```

| Componente | Tecnologia | Ruolo |
|---|---|---|
| Frontend | HTML + CSS + JavaScript (vanilla) | Dashboard UI, filtri, timeline |
| Backend | Azure Function — PowerShell 7.4 | Proxy autenticato verso Graph API |
| Autenticazione utente | Easy Auth — Microsoft Entra ID | Login obbligatorio, solo utenti del tenant |
| Autenticazione Graph | OAuth 2.0 Client Credentials | App-only, senza utente interattivo |
| Hosting frontend | Azure Static Web Apps (Free tier) | Deploy da GitHub, CDN globale |
| Hosting backend | Azure Function App (Consumption plan) | Serverless, pay-per-use |

---

## Prerequisiti

- Tenant Microsoft 365 attivo
- Accesso come Global Admin o Privileged Role Admin (necessario per il consenso ai permessi Graph e per la configurazione di Easy Auth)
- Subscription Azure attiva (anche free tier)
- Account GitHub (per il deploy automatico della Static Web App)

**Permessi Microsoft Graph richiesti (Application permissions):**

| Permesso | Tipo | Scopo |
|---|---|---|
| ServiceHealth.Read.All | Application | Lettura stato servizi e incident |
| ServiceMessage.Read.All | Application | Lettura post e aggiornamenti incident |

Entrambi richiedono il Admin Consent esplicito da parte di un Global Admin o Privileged Role Admin.

---

## Struttura repository

```
/
├── index.html                    # Frontend dashboard (single file)
├── staticwebapp.config.json      # Configurazione routing e autenticazione Static Web App
├── GetServiceHealth/
│   └── run.ps1                   # Azure Function — PowerShell HTTP trigger
└── README.md                     # Questa documentazione
```

---

## Setup step-by-step

### 1. App Registration — Graph API

> Questa App Registration gestisce l'accesso app-only a Microsoft Graph (Client Credentials flow). È separata da quella usata per l'autenticazione utente.

1. Vai su **portal.azure.com → Microsoft Entra ID → App registrations → New registration**

   | Campo | Valore |
   |---|---|
   | Name | `ServiceHealthDashboard` (o nome a scelta) |
   | Supported account types | Accounts in this organizational directory only |
   | Redirect URI | lascia vuoto |

2. Dalla scheda **Overview** annota:
   - **Application (client) ID** → `CLIENT_ID`
   - **Directory (tenant) ID** → `TENANT_ID`

3. Vai su **Certificates & secrets → New client secret**

   | Campo | Valore |
   |---|---|
   | Description | `dashboard-secret` |
   | Expiry | 24 months |

   Copia subito il valore — non sarà più visibile dopo la chiusura del pannello.

4. Vai su **API permissions → Add a permission → Microsoft Graph → Application permissions**
   - Aggiungi: `ServiceHealth.Read.All`
   - Aggiungi: `ServiceMessage.Read.All`
   - Clicca **Grant admin consent for [tenant]** e conferma

> **Nota:** se ricevi un errore 403 al primo test, verifica che il consent sia stato concesso correttamente. Il pulsante "Grant admin consent" deve mostrare il segno di spunta verde accanto a entrambi i permessi.

---

### 2. Azure Function App

**Creazione**

Vai su **portal.azure.com → Create a resource → Function App**

| Campo | Valore |
|---|---|
| Subscription | La tua subscription |
| Resource Group | Crea nuovo o usa esistente |
| Function App name | Nome univoco (es. `servicehealthdashboard`) |
| Runtime stack | PowerShell Core |
| Version | 7.4 |
| Region | West Europe (o la più vicina) |
| Plan type | Consumption (Serverless) |

**Variabili d'ambiente**

Dopo la creazione, vai su **Settings → Environment variables → tab App settings** e aggiungi:

| Name | Value |
|---|---|
| `TENANT_ID` | Il tuo Directory (tenant) ID |
| `CLIENT_ID` | Il tuo Application (client) ID |
| `CLIENT_SECRET` | Il valore del client secret creato al passo 1 |

Clicca **Apply → Confirm** per salvare.

**Creazione della Function**

1. Vai su **Functions → Create**
2. Seleziona **HTTP trigger**
3. Configura:
   - Name: `GetServiceHealth`
   - Authorization level: `Function`
4. Apri **Code + Test** e incolla il contenuto di `GetServiceHealth/run.ps1`
5. Clicca **Save**

**Test**

Vai su **Code + Test → Test/Run → seleziona metodo GET → clicca Run**.

Nel tab Output dovresti ricevere:
```json
{
  "health": [ ... ],
  "issues": [ ... ]
}
```

**Recupero Function URL**

Vai su **Function Keys** → copia il valore della chiave `default`.

L'URL completo da usare nel frontend è:
```
https://<nome-app>.azurewebsites.net/api/GetServiceHealth?code=<function-key>
```

---

### 3. Azure Static Web App

1. Vai su **portal.azure.com → Create a resource → Static Web App**

   | Campo | Valore |
   |---|---|
   | Name | nome a scelta |
   | Plan type | Free |
   | Source | GitHub |
   | Repository | seleziona la tua repo |
   | Branch | main |
   | App location | `/` |
   | API location | (lascia vuoto) |
   | Output location | `/` |

2. Azure crea automaticamente un file di workflow GitHub Actions nella repo (`.github/workflows/`). Ogni push su `main` triggera il deploy automatico.

---

### 4. Autenticazione Entra ID (Easy Auth)

L'accesso alla dashboard è protetto tramite **Azure Static Web Apps Easy Auth** con Microsoft Entra ID. Gli utenti non autenticati vengono reindirizzati automaticamente alla pagina di login Microsoft e devono appartenere al tenant configurato.

#### 4.1 App Registration — Autenticazione utente

> Questa App Registration è separata da quella Graph API del passo 1. Gestisce esclusivamente il login degli utenti tramite Easy Auth.

1. Vai su **Microsoft Entra ID → App registrations → New registration**

   | Campo | Valore |
   |---|---|
   | Name | `ServiceHealthDashboard-Auth` |
   | Supported account types | Accounts in this organizational directory only |
   | Redirect URI | lascia vuoto per ora |

2. Dalla scheda **Overview** annota:
   - **Application (client) ID**
   - **Directory (tenant) ID**

3. Vai su **Certificates & secrets → New client secret**

   | Campo | Valore |
   |---|---|
   | Description | `static-web-app-auth` |
   | Expiry | 24 months |

4. Vai su **Authentication → Add a platform → Web** e aggiungi il Redirect URI:
   ```
   https://<nome-app>.azurestaticapps.net/.auth/login/aad/callback
   ```
   Spunta **ID tokens** sotto "Implicit grant and hybrid flows", poi **Save**.

#### 4.2 Configurazione Easy Auth sulla Static Web App

1. Vai su **Static Web Apps → [la tua app] → Settings → Authentication**
2. Clicca **Add identity provider**
3. Configura:

   | Campo | Valore |
   |---|---|
   | Provider | Entra ID |
   | Client ID | Application (client) ID dell'App Registration Auth |
   | Client Secret | client secret creato al punto 4.1 |

4. Clicca **Add**

   > Al primo accesso di un utente (o di un amministratore la prima volta), Entra ID mostrerà una schermata di consenso per i permessi base del profilo. Spunta **"Acconsenti per conto dell'organizzazione"** prima di cliccare Accetto — così il consenso vale per tutti gli utenti del tenant e non viene richiesto nuovamente.

#### 4.3 File `staticwebapp.config.json`

Crea il file `staticwebapp.config.json` nella root della repo (stessa cartella di `index.html`):

```json
{
  "auth": {
    "identityProviders": {
      "azureActiveDirectory": {
        "userDetailsClaim": "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/name",
        "registration": {
          "openIdIssuer": "https://login.microsoftonline.com/<TENANT_ID>/v2.0",
          "clientIdSettingName": "AZURE_CLIENT_ID",
          "clientSecretSettingName": "AZURE_CLIENT_SECRET"
        }
      }
    }
  },
  "routes": [
    {
      "route": "/*",
      "allowedRoles": ["authenticated"]
    }
  ],
  "responseOverrides": {
    "401": {
      "redirect": "/.auth/login/aad",
      "statusCode": 302
    }
  }
}
```

Sostituisci `<TENANT_ID>` con il tuo Directory (tenant) ID.

| Sezione | Funzione |
|---|---|
| `auth.identityProviders` | Limita il login agli utenti del tenant specificato |
| `routes` | Protegge tutte le route — solo utenti autenticati possono accedere |
| `responseOverrides` | Reindirizza automaticamente al login invece di restituire 401 |

Committa e pusha il file. GitHub Actions farà il deploy automaticamente.

---

### 5. CORS

Per permettere al browser di chiamare la Function dalla Static Web App, configura CORS nella Function App:

1. Vai su **Settings → CORS**
2. Aggiungi:
   ```
   https://<nome-app>.azurestaticapps.net
   ```
3. Opzionale, per sviluppo locale:
   ```
   http://localhost:5500
   http://127.0.0.1:5500
   ```
4. Salva le modifiche

> Se la Function risponde correttamente ma il browser riceve un errore CORS, verifica che l'URL della Static Web App sia esatto (senza slash finale).

---

## Script e file

### `GetServiceHealth/run.ps1` — Azure Function

Script PowerShell deployato come Azure Function HTTP Trigger. Ottiene un token OAuth2 con client credentials flow e chiama due endpoint Graph API in sequenza, restituendo i dati combinati in JSON.

```powershell
using namespace System.Net

param($Request, $TriggerMetadata)

$tenantId     = $env:TENANT_ID
$clientId     = $env:CLIENT_ID
$clientSecret = $env:CLIENT_SECRET

# Ottieni access token via OAuth2 client credentials
$tokenBody = @{
    grant_type    = "client_credentials"
    client_id     = $clientId
    client_secret = $clientSecret
    scope         = "https://graph.microsoft.com/.default"
}

$tokenResponse = Invoke-RestMethod `
    -Uri "https://login.microsoftonline.com/$tenantId/oauth2/v2.0/token" `
    -Method POST `
    -Body $tokenBody

$headers = @{ Authorization = "Bearer $($tokenResponse.access_token)" }

# Chiama Graph API — stato servizi
$health = (Invoke-RestMethod `
    -Uri "https://graph.microsoft.com/v1.0/admin/serviceAnnouncement/healthOverviews" `
    -Headers $headers).value

# Chiama Graph API — incident attivi, ordinati per data decrescente
$issues = (Invoke-RestMethod `
    -Uri "https://graph.microsoft.com/v1.0/admin/serviceAnnouncement/issues?`$filter=isResolved eq false&`$orderby=startDateTime desc&`$top=20" `
    -Headers $headers).value

$body = @{ health = $health; issues = $issues } | ConvertTo-Json -Depth 10

Push-OutputBinding -Name Response -Value ([HttpResponseContext]@{
    StatusCode = [HttpStatusCode]::OK
    Headers    = @{
        "Content-Type"                = "application/json"
        "Access-Control-Allow-Origin" = "*"
    }
    Body = $body
})
```

**Note tecniche:**
- `Start-Job` non è supportato nell'ambiente Azure Functions (PowerShell hosted in-process) — le chiamate Graph vengono eseguite in sequenza
- Il token OAuth2 ha durata di 3600 secondi; la Function non implementa caching (ogni invocazione ottiene un token nuovo)
- `$top=20` limita gli incident restituiti; aumenta il valore se necessario
- L'header `Access-Control-Allow-Origin: *` è necessario per le chiamate cross-origin dal browser

### `index.html` — Frontend dashboard

Single-file HTML/CSS/JS. Non ha dipendenze esterne (nessun framework, nessuna libreria npm). Carica i font da Google Fonts e non richiede build step.

**Funzionalità implementate:**

| Feature | Implementazione |
|---|---|
| Dark/Light mode | Toggle in header, preferenza salvata in localStorage |
| Summary bar | Conteggio live di servizi totali, operativi, degradati, incident |
| Griglia servizi | Auto-grid CSS, ordinata per criticità (degradati in cima) |
| Ricerca servizi | Filtro live su input event |
| Filtro stato servizi | Select: Tutti / Solo degradati / Solo operativi |
| Incident espandibili | Click su card — toggle CSS class open |
| Barra età incident | Colore progressivo verde→rosso in base ai giorni dall'apertura |
| Metadata incident | Servizio, feature, featureGroup, classificazione, durata, workload impattati |
| Timeline aggiornamenti | Post in ordine cronologico inverso (più recente in cima) |
| Post collassabili | Testo lungo troncato a 3 righe con "Mostra tutto" |
| Ricerca incident | Filtro su titolo, ID, nome servizio |
| Filtro per servizio | Select popolato dinamicamente dai dati ricevuti |
| Filtro classificazione | Incident / Advisory |
| Skeleton loading | Placeholder animati durante il caricamento |
| Animazioni | fadeUp su card al render, pulse su status dot degradati |
| Responsive | Layout adattivo per mobile (griglia summary 2 colonne) |

---

## Endpoint Graph API utilizzati

### `GET /admin/serviceAnnouncement/healthOverviews`

Restituisce lo stato attuale di tutti i servizi Microsoft 365.

**Valori `status` possibili:**

| Valore | Significato | UI |
|---|---|---|
| `serviceOperational` | Servizio operativo | Badge verde |
| `serviceDegradation` | Degradazione parziale | Badge giallo + dot animato |
| `serviceInterruption` | Interruzione parziale | Badge giallo + dot animato |
| `serviceOutage` | Outage completo | Badge rosso + dot animato |
| `restoringService` | In ripristino | Badge giallo |
| `falsePositive` | Falso positivo | Badge verde |
| `serviceRestored` | Ripristinato | Badge verde |

### `GET /admin/serviceAnnouncement/issues`

Restituisce gli incident attivi (o risolti se non filtrati).

**Query parameters usati:**
- `$filter=isResolved eq false`
- `$orderby=startDateTime desc`
- `$top=20`

**Valori `classification`:**
- `incident` — interruzione di servizio con impatto diretto sugli utenti
- `advisory` — degradazione o funzionalità limitata, impatto parziale

---

## Funzionalità dashboard

### Barra età incident

| Soglia | Colore | Significato |
|---|---|---|
| ≤ 3 giorni | Verde `#107c10` | Incident recente |
| 4–7 giorni | Giallo `#835b00` | Incident in corso |
| 8–14 giorni | Arancione `#c44` | Attenzione |
| > 14 giorni | Rosso `#a00` | Incident critico per durata |

La larghezza della barra è proporzionale ai giorni (100% = 30 giorni).

### Ordinamento incident

Gli incident vengono ordinati per `lastModifiedDateTime` decrescente: quello aggiornato più di recente appare in cima.

### Parsing dei post

Il contenuto dei post Graph API è in formato HTML. La dashboard effettua uno strip dei tag HTML e una normalizzazione degli spazi bianchi eccessivi prima della visualizzazione.

---

## Troubleshooting

### `{"health": null, "issues": null}` dalla Function

Cause possibili:
- Le variabili d'ambiente (`TENANT_ID`, `CLIENT_ID`, `CLIENT_SECRET`) non sono state salvate correttamente
- Il client secret è scaduto → rigenera il secret su Entra ID e aggiorna la variabile
- L'admin consent non è stato concesso → vai su **API permissions** e verifica il segno di spunta verde

### Errore CORS nel browser

```
Access to fetch at '...' from origin '...' has been blocked by CORS policy
```

Aggiungi l'URL esatto della Static Web App nella sezione CORS della Function App (senza slash finale).

### Errore 401 dalla Function

Il token OAuth2 non è valido. Verifica che `CLIENT_ID`, `CLIENT_SECRET` e `TENANT_ID` corrispondano all'App Registration su Entra ID.

### Errore 403 dalla Function

Il token è valido ma l'app non ha i permessi necessari. Verifica che:
- I permessi `ServiceHealth.Read.All` e `ServiceMessage.Read.All` siano aggiunti come **Application permissions** (non Delegated)
- L'admin consent sia stato concesso (segno di spunta verde nella colonna "Status" dei permessi)

### Il login mostra la schermata di consenso ad ogni accesso

Al primo accesso è normale. Per evitarlo per tutti gli utenti successivi, durante la schermata di consenso spunta **"Acconsenti per conto dell'organizzazione"** prima di cliccare Accetto. Richiede un account Global Admin o Privileged Role Admin.

### La dashboard si apre senza richiedere il login

Verifica che il file `staticwebapp.config.json` sia nella root della repo e che il deploy GitHub Actions sia stato completato correttamente. Controlla il tab **Actions** nella repo GitHub.

### La Function risponde lentamente al primo avvio

Il Consumption plan di Azure Functions ha un cold start di alcuni secondi dopo periodi di inattività. È normale. Per mitigarlo è possibile abilitare **Always On** (richiede piano App Service) o impostare un timer trigger che invoca la Function periodicamente.

### Deploy Static Web App non si aggiorna

Vai su **Actions** nella repo GitHub e controlla l'ultimo run del workflow `.github/workflows/`.

---

## Limitazioni e sicurezza

| Limitazione | Dettaglio | Impatto |
|---|---|---|
| Function Key esposta nel frontend | La chiave API è visibile nel sorgente HTML | Chiunque abbia accesso alla pagina può chiamare la Function |
| `$top=20` sugli incident | Limitato agli ultimi 20 incident aperti | Incident storici non visibili |
| Nessun caching | Ogni apertura della pagina chiama Graph API | Leggero overhead, possibile throttling su uso intensivo |
| Client secret con scadenza | I due client secret (Graph e Auth) scadono dopo 24 mesi | Richiede rotazione manuale periodica |

**Raccomandazioni per ambienti enterprise:**
- Sostituire la Function Key con autenticazione Managed Identity o token Entra ID
- Implementare un layer di caching (Azure Cache for Redis o Azure Table Storage) per ridurre le chiamate a Graph
- Spostare l'URL della Function in una variabile d'ambiente gestita dalla Static Web App invece di hardcodarla nell'HTML
- Configurare un alert su Entra ID per la scadenza dei client secret

---

## Roadmap

- [x] Autenticazione utente via Entra ID (Easy Auth su Static Web App)
- [ ] Rimozione Function Key dal frontend (sostituzione con Managed Identity)
- [ ] Caching dei dati con TTL configurabile (Azure Table Storage)
- [ ] Storico incident risolti (ultimi N giorni)
- [ ] Notifiche automatiche via Teams Webhook su nuovi incident
- [ ] Esportazione PDF / CSV dello stato corrente
- [ ] Supporto multi-tenant

---

## Autore

Progetto sviluppato per ottimizzare il monitoraggio operativo quotidiano dei servizi Microsoft 365 in contesti amministrativi e sysadmin M365.

Costruito con: Azure Functions · Microsoft Graph API · Azure Static Web Apps · Microsoft Entra ID · PowerShell · HTML/CSS/JS vanilla
