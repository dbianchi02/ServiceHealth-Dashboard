# Microsoft 365 Service Health Dashboard

## Overview

Questa dashboard fornisce una vista centralizzata e aggiornata dello stato dei servizi Microsoft 365, inclusi incident attivi e stato operativo dei principali workload (Exchange, Teams, SharePoint, ecc.).

L’obiettivo è semplificare il monitoraggio quotidiano evitando la consultazione manuale del Microsoft 365 Service Health e del Message Center.

<img width="1859" height="934" alt="image" src="https://github.com/user-attachments/assets/5e7901a3-09d2-480c-bb45-f2894719a0c3" />


---

## Architettura

La soluzione è composta da due componenti principali:

* **Frontend (Static Web App)**

  * HTML + JavaScript
  * Deploy su Azure Static Web Apps
  * Visualizzazione dashboard in tempo reale

* **Backend (Azure Function)**

  * Runtime: PowerShell
  * Autenticazione: OAuth2 (client credentials)
  * Integrazione con Microsoft Graph API:

    * `/admin/serviceAnnouncement/healthOverviews`
    * `/admin/serviceAnnouncement/issues`

---

## Flusso dei dati

```text
Frontend (Static Web App)
        ↓
Azure Function (HTTP Trigger)
        ↓
Microsoft Graph API
        ↓
Response JSON (health + issues)
        ↓
Rendering dashboard
```

---

## Prerequisiti

* Tenant Microsoft 365
* Permessi Microsoft Graph (Application):

  * `ServiceHealth.Read.All`
  * `ServiceMessage.Read.All`
* App Registration in Entra ID:

  * `client_id`
  * `client_secret`
  * `tenant_id`
* Azure Function App (PowerShell)
* Azure Static Web App

---

## Configurazione

### 1. Azure Function

Configurare le variabili d’ambiente:

* `TENANT_ID`
* `CLIENT_ID`
* `CLIENT_SECRET`

La Function:

* ottiene un access token OAuth2
* interroga Microsoft Graph
* restituisce i dati in formato JSON

---

### 2. CORS

Configurare i seguenti origin nella Function App:

* URL della Static Web App:

  ```text
  https://<nome-app>.azurestaticapps.net
  ```

* (Opzionale) ambiente locale:

  ```text
  http://localhost:5500
  http://127.0.0.1:5500
  ```

---

### 3. Static Web App

Configurazione di deploy:

* App location: `/`
* API location: *(vuoto)*
* Output location: `/`

Il file principale deve essere:

```text
index.html
```

---

## Utilizzo

Una volta deployata, la dashboard è accessibile via browser:

```text
https://<nome-app>.azurestaticapps.net
```

La pagina:

* mostra lo stato dei servizi Microsoft 365
* evidenzia eventuali degradazioni o incident
* elenca gli incident attivi con dettagli

---

## Integrazione con Microsoft 365

### SharePoint

La dashboard può essere:

* caricata come file HTML in una document library
* incorporata tramite Web Part Embed (se consentito)

### Microsoft Teams

Può essere aggiunta come:

* tab di tipo "Website"
* oppure tramite pagina SharePoint integrata

---

## Limitazioni attuali

* La Function Key è esposta nel frontend (uso prototipale)
* Nessuna autenticazione lato utente
* Nessun caching dei dati (chiamate live a Graph)

---

## Miglioramenti futuri

* Integrazione con Entra ID (Easy Auth)
* Rimozione della Function Key dal frontend
* Introduzione di caching (Azure Table / Redis)
* Storico incident e trend
* Notifiche automatiche (Teams / Email)

---

## Sicurezza

Questa implementazione è pensata per uso personale o ambienti controllati.

Per ambienti enterprise si raccomanda:

* autenticazione tramite Entra ID
* protezione delle API backend
* rimozione di segreti dal frontend

---

## Autore

Progetto sviluppato per ottimizzare il monitoraggio operativo quotidiano dei servizi Microsoft 365 in contesti amministrativi e sysadmin.

---
