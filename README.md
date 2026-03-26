# Microsoft 365 Service Health Dashboard

> Dashboard web per il monitoraggio in tempo reale dello stato dei servizi Microsoft 365, incident attivi e degradazioni operative. Progettata per sysadmin M365 che vogliono una vista centralizzata senza consultare manualmente il portale Microsoft 365 Admin Center.

<img width="1873" height="849" alt="image" src="https://github.com/user-attachments/assets/b4f58f61-8b4f-4695-b2d3-b8a05fa89ee1" />

---

## Indice

- [Architettura](#architettura)
- [Prerequisiti](#prerequisiti)
- [Struttura repository](#struttura-repository)
- [Setup step-by-step](#setup-step-by-step)
  - [1. App Registration su Azure AD](#1-app-registration-su-azure-ad)
  - [2. Azure Function App](#2-azure-function-app)
  - [3. Azure Static Web App](#3-azure-static-web-app)
  - [4. CORS](#4-cors)
- [Script e file](#script-e-file)
  - [run.ps1 — Azure Function](#runps1--azure-function)
  - [index.html — Frontend dashboard](#indexhtml--frontend-dashboard)
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
        │  fetch /api/GetServiceHealth?code=<function-key>
        ▼
Azure Function App (PowerShell HTTP Trigger)
        │
        │  POST /oauth2/v2.0/token  →  access_token
        ▼
Microsoft Entra ID (App Registration)
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

**Componenti:**

| Componente | Tecnologia | Ruolo |
|---|---|---|
| Frontend | HTML + CSS + JavaScript (vanilla) | Dashboard UI, filtri, timeline |
| Backend | Azure Function — PowerShell 7.4 | Proxy autenticato verso Graph API |
| Autenticazione | OAuth 2.0 Client Credentials | App-only, senza utente interattivo |
| Hosting frontend | Azure Static Web Apps (Free tier) | Deploy da GitHub, CDN globale |
| Hosting backend | Azure Function App (Consumption plan) | Serverless, pay-per-use |

---

## Prerequisiti

- Tenant Microsoft 365 attivo
- Accesso come **Global Admin** o **Privileged Role Admin** (necessario per il consenso ai permessi Graph)
- Subscription Azure attiva (anche free tier)
- Account GitHub (per il deploy automatico della Static Web App)

**Permessi Microsoft Graph richiesti (Application permissions):**

| Permesso | Tipo | Scopo |
|---|---|---|
| `ServiceHealth.Read.All` | Application | Lettura stato servizi e incident |
| `ServiceMessage.Read.All` | Application | Lettura post e aggiornamenti incident |

Entrambi richiedono il **Admin Consent** esplicito da parte di un Global Admin o Privileged Role Admin.

---

## Struttura repository

```
/
├── index.html              # Frontend dashboard (single file)
├── GetServiceHealth/
│   └── run.ps1             # Azure Function — PowerShell HTTP trigger
└── README.md               # Questa documentazione
```

---

## Setup step-by-step

### 1. App Registration su Azure AD

1. Vai su [portal.azure.com](https://portal.azure.com) → **Microsoft Entra ID** → **App registrations** → **New registration**

2. Compila:
   - **Name:** `ServiceHealthDashboard` (o nome a scelta)
   - **Supported account types:** `Accounts in this organizational directory only`
   - **Redirect URI:** lascia vuoto

3. Dopo la registrazione, dalla scheda **Overview** annota:
   - `Application (client) ID` → sarà il tuo `CLIENT_ID`
   - `Directory (tenant) ID` → sarà il tuo `TENANT_ID`

4. Vai su **Certificates & secrets** → **New client secret**
   - Description: `dashboard-secret`
   - Expiry: `24 months`
   - **Copia subito il valore** — non sarà più visibile dopo la chiusura del pannello

5. Vai su **API permissions** → **Add a permission** → **Microsoft Graph** → **Application permissions**
   - Aggiungi: `ServiceHealth.Read.All`
   - Aggiungi: `ServiceMessage.Read.All`
   - Clicca **Grant admin consent for [tenant]** e conferma

> **Nota:** se ricevi un errore 403 al primo test, verifica che il consent sia stato concesso correttamente. Il pulsante "Grant admin consent" deve mostrare il segno di spunta verde accanto a entrambi i permessi.

---

### 2. Azure Function App

#### Creazione

1. Vai su [portal.azure.com](https://portal.azure.com) → **Create a resource** → **Function App**

2. Configura:

   | Campo | Valore |
   |---|---|
   | Subscription | La tua subscription |
   | Resource Group | Crea nuovo o usa esistente |
   | Function App name | Nome univoco (es. `servicehealthdashboard`) |
   | Runtime stack | `PowerShell Core` |
   | Version | `7.4` |
   | Region | `West Europe` (o la più vicina) |
   | Plan type | `Consumption (Serverless)` |

3. Lascia tutti gli altri tab ai valori di default e clicca **Review + Create** → **Create**

#### Variabili d'ambiente

Dopo la creazione, vai su **Settings** → **Environment variables** → tab **App settings** e aggiungi:

| Name | Value |
|---|---|
| `TENANT_ID` | Il tuo Directory (tenant) ID |
| `CLIENT_ID` | Il tuo Application (client) ID |
| `CLIENT_SECRET` | Il valore del client secret creato al passo 1 |

Clicca **Apply** → **Confirm** per salvare.

#### Creazione della Function

1. Vai su **Functions** → **Create**
2. Seleziona **HTTP trigger**
3. Configura:
   - **Name:** `GetServiceHealth`
   - **Authorization level:** `Function`
4. Apri **Code + Test** e incolla il contenuto di [`GetServiceHealth/run.ps1`](#runps1--azure-function)
5. Clicca **Save**

#### Test

Vai su **Code + Test** → **Test/Run** → seleziona metodo `GET` → clicca **Run**.

Nel tab **Output** dovresti ricevere:
```json
{
  "health": [ ... ],
  "issues": [ ... ]
}
```

Se ricevi `{"health": null, "issues": null}` o un errore 500, controlla il log nella stessa pagina e verifica le variabili d'ambiente e i permessi Graph.

#### Recupero Function URL

Vai su **Function Keys** (tab in alto nella pagina della function) → copia il valore della chiave `default`.

L'URL completo da usare nel frontend è:
```
https://<nome-app>.azurewebsites.net/api/GetServiceHealth?code=<function-key>
```

---

### 3. Azure Static Web App

1. Vai su [portal.azure.com](https://portal.azure.com) → **Create a resource** → **Static Web App**

2. Configura:
   - **Name:** nome a scelta
   - **Plan type:** `Free`
   - **Source:** `GitHub`
   - **Repository:** seleziona la tua repo
   - **Branch:** `main`
   - **App location:** `/`
   - **API location:** *(lascia vuoto)*
   - **Output location:** `/`

3. Azure crea automaticamente un file di workflow GitHub Actions nella repo (`.github/workflows/`). Ogni push su `main` triggera il deploy automatico.

---

### 4. CORS

Per permettere al browser di chiamare la Function dalla Static Web App, configura CORS nella Function App:

1. Vai su **Settings** → **CORS**
2. Aggiungi i seguenti origin:

```
https://<nome-app>.azurestaticapps.net
```

Opzionale, per sviluppo locale:
```
http://localhost:5500
http://127.0.0.1:5500
```

3. Salva le modifiche

> Se noti che la Function risponde correttamente ma il browser riceve un errore CORS, verifica che l'URL della Static Web App sia esatto (senza slash finale).

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

---

### `index.html` — Frontend dashboard

Single-file HTML/CSS/JS. Non ha dipendenze esterne (nessun framework, nessuna libreria npm). Carica i font da Google Fonts e non richiede build step.

**Funzionalità implementate:**

| Feature | Implementazione |
|---|---|
| Dark/Light mode | Toggle in header, preferenza salvata in `localStorage` |
| Summary bar | Conteggio live di servizi totali, operativi, degradati, incident |
| Griglia servizi | Auto-grid CSS, ordinata per criticità (degradati in cima) |
| Ricerca servizi | Filtro live su `input` event |
| Filtro stato servizi | Select: Tutti / Solo degradati / Solo operativi |
| Incident espandibili | Click su card — toggle CSS class `open` |
| Barra età incident | Colore progressivo verde→rosso in base ai giorni dall'apertura |
| Metadata incident | Servizio, feature, featureGroup, classificazione, durata, workload impattati |
| Timeline aggiornamenti | Post in ordine cronologico inverso (più recente in cima) |
| Post collassabili | Testo lungo troncato a 3 righe con "Mostra tutto" |
| Ricerca incident | Filtro su titolo, ID, nome servizio |
| Filtro per servizio | Select popolato dinamicamente dai dati ricevuti |
| Filtro classificazione | Incident / Advisory |
| Skeleton loading | Placeholder animati durante il caricamento |
| Animazioni | `fadeUp` su card al render, `pulse` su status dot degradati |
| Responsive | Layout adattivo per mobile (griglia summary 2 colonne) |

**Mappa dei campi JSON → UI:**

| Campo JSON | Dove appare |
|---|---|
| `health[].service` | Nome nella griglia servizi |
| `health[].status` | Badge stato + dot animato |
| `issues[].id` | ID nel header incident |
| `issues[].title` | Titolo card incident |
| `issues[].service` | Sottotitolo + filtro servizi |
| `issues[].classification` | Badge Incident / Advisory |
| `issues[].feature` | Sezione metadata espansa |
| `issues[].featureGroup` | Sottotitolo card + metadata |
| `issues[].startDateTime` | Data inizio + calcolo durata |
| `issues[].lastModifiedDateTime` | Ordinamento lista + metadata |
| `issues[].posts[].createdDateTime` | Data nella timeline |
| `issues[].posts[].description.content` | Testo post (HTML strippato) |
| `issues[].details[AffectedChildWorkloads]` | Riga workload nei metadata |

---

## Endpoint Graph API utilizzati

### `GET /admin/serviceAnnouncement/healthOverviews`

Restituisce lo stato attuale di tutti i servizi Microsoft 365.

**Response (estratto):**
```json
{
  "value": [
    {
      "service": "Exchange Online",
      "status": "serviceDegradation",
      "id": "Exchange"
    }
  ]
}
```

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

---

### `GET /admin/serviceAnnouncement/issues`

Restituisce gli incident attivi (o risolti se non filtrati).

**Query parameters usati:**
```
$filter=isResolved eq false
$orderby=startDateTime desc
$top=20
```

**Struttura `issues[]` (campi rilevanti):**

```json
{
  "id": "EX1255397",
  "title": "Some users may be unable to access...",
  "service": "Exchange Online",
  "feature": "Tenant Administration (Provisioning, Remote PowerShell)",
  "featureGroup": "Management and Provisioning",
  "classification": "advisory",
  "status": "serviceDegradation",
  "startDateTime": "2026-03-18T21:53:53Z",
  "lastModifiedDateTime": "2026-03-24T20:11:52.487Z",
  "endDateTime": null,
  "isResolved": false,
  "impactDescription": "Users may be unable to...",
  "details": [
    { "name": "NotifyInApp", "value": "True" },
    { "name": "AffectedChildWorkloads", "value": "Exchange Online,Teams" }
  ],
  "posts": [
    {
      "createdDateTime": "2026-03-18T21:55:12.567Z",
      "postType": "regular",
      "description": {
        "contentType": "html",
        "content": "<html content>"
      }
    }
  ]
}
```

**Valori `classification`:**
- `incident` — interruzione di servizio con impatto diretto sugli utenti
- `advisory` — degradazione o funzionalità limitata, impatto parziale

---

## Funzionalità dashboard

### Barra età incident

La barra colorata sotto ogni header di incident indica visivamente da quanto tempo è aperto:

| Soglia | Colore | Significato |
|---|---|---|
| ≤ 3 giorni | Verde `#107c10` | Incident recente |
| 4–7 giorni | Giallo `#835b00` | Incident in corso |
| 8–14 giorni | Arancione `#c44` | Attenzione |
| > 14 giorni | Rosso `#a00` | Incident critico per durata |

La larghezza della barra è proporzionale ai giorni (100% = 30 giorni).

### Ordinamento incident

Gli incident vengono ordinati per `lastModifiedDateTime` decrescente: quello aggiornato più di recente appare in cima, facilitando l'identificazione delle situazioni in evoluzione.

### Parsing dei post

Il contenuto dei post Graph API è in formato HTML. La dashboard effettua uno strip dei tag HTML e una normalizzazione degli spazi bianchi eccessivi prima della visualizzazione.

---

## Troubleshooting

### `{"health": null, "issues": null}` dalla Function

**Cause possibili:**
1. Le variabili d'ambiente (`TENANT_ID`, `CLIENT_ID`, `CLIENT_SECRET`) non sono state salvate correttamente → vai su **Environment variables** e verifica
2. Il client secret è scaduto → rigenera il secret su Entra ID e aggiorna la variabile
3. L'admin consent non è stato concesso → vai su **API permissions** e verifica il segno di spunta verde

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

### La Function risponde lentamente al primo avvio

Il Consumption plan di Azure Functions ha un cold start di alcuni secondi dopo periodi di inattività. È normale. Per mitigarlo è possibile abilitare il **Always On** (richiede piano App Service) o impostare un timer trigger che invoca la Function periodicamente.

### Deploy Static Web App non si aggiorna

Verifica che il workflow GitHub Actions in `.github/workflows/` sia in esecuzione correttamente. Vai su **Actions** nella repo GitHub e controlla l'ultimo run.

---

## Limitazioni e sicurezza

| Limitazione | Dettaglio | Impatto |
|---|---|---|
| Function Key esposta nel frontend | La chiave API è visibile nel sorgente HTML | Chiunque abbia accesso alla pagina può chiamare la Function |
| Nessuna autenticazione utente | La dashboard è pubblica (se la Static Web App è pubblica) | Accesso non controllato |
| Nessun caching | Ogni apertura della pagina chiama Graph API | Leggero overhead, possibile throttling su uso intensivo |
| `$top=20` sugli incident | Limitato agli ultimi 20 incident aperti | Incident storici non visibili |

**Raccomandazioni per ambienti enterprise:**
- Abilitare **Entra ID Authentication** sulla Static Web App (Easy Auth) per richiedere il login Microsoft
- Sostituire la Function Key con autenticazione Managed Identity o token Entra ID
- Implementare un layer di caching (Azure Cache for Redis o Azure Table Storage) per ridurre le chiamate a Graph
- Spostare l'URL della Function in una variabile d'ambiente gestita dalla Static Web App invece di hardcodarla nell'HTML

---

## Roadmap

- [ ] Autenticazione utente via Entra ID (Easy Auth su Static Web App)
- [ ] Rimozione Function Key dal frontend (sostituzione con Managed Identity)
- [ ] Caching dei dati con TTL configurabile (Azure Table Storage)
- [ ] Storico incident risolti (ultimi N giorni)
- [ ] Notifiche automatiche via Teams Webhook su nuovi incident
- [ ] Esportazione PDF / CSV dello stato corrente
- [ ] Supporto multi-tenant

---

## Autore

Progetto sviluppato per ottimizzare il monitoraggio operativo quotidiano dei servizi Microsoft 365 in contesti amministrativi e sysadmin M365.

Costruito con: Azure Functions · Microsoft Graph API · Azure Static Web Apps · PowerShell · HTML/CSS/JS vanilla
