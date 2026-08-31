# 🛡️ Automated Disaster Recovery & 3-2-1 Backup Architecture

![Disaster Recovery](https://img.shields.io/badge/Disaster_Recovery-3--2--1_Rule-darkred?style=for-the-badge)
![Cron/Bash](https://img.shields.io/badge/Automation-Cron%2FBash-4EAA25?style=for-the-badge&logo=gnu-bash)
![Telegram API](https://img.shields.io/badge/SecOps_Alerting-Telegram_API-2CA5E0?style=for-the-badge&logo=telegram)
![Docker Volumes](https://img.shields.io/badge/Stateful_Data-Docker_Volumes-2496ED?style=for-the-badge&logo=docker)

## 🎯 Executive Summary
Questo documento tecnico descrive la progettazione e l'implementazione di una strategia di **Business Continuity** e **Disaster Recovery** per un'infrastruttura containerizzata basata su hardware legacy. 

Il progetto applica rigorosamente la regola aurea del backup "3-2-1" (3 copie dei dati, su 2 supporti diversi, di cui 1 off-site) per proteggere applicazioni *Stateful* (Node.js, SQLite, Nginx, Stirling-PDF). Inoltre, implementa una logica di **No-Noise Alerting** tramite Webhook per abbattere l'Alert Fatigue operativa.

---

## 🏗️ Architettura del Flusso di Backup (Mermaid Diagram)

grafico che mostra la sequenza del funzionamento del backup sui server

---

## ⚙️ Fasi di Implementazione e Ingegnerizzazione

### Fase 1: Local Backup & Data Compression
L'estrazione dei dati dai container in esecuzione senza causare *Database Locks* (su SQLite) è gestita tramite il container `offen/docker-volume-backup`. 
Il task è schedulato via cronjob alle ore **02:00 AM**. I dati vengono estratti dai volumi fisici, compressi in formato `.tar.gz` per minimizzare l'impatto sul *Compute Budget* e lo storage I/O del disco. È stata impostata una **Retention Policy di 7 giorni** per la rotazione automatica e la prevenzione della saturazione dello spazio.

### Fase 2: On-Premise Redundancy (Cross-Node Failover)
Per mitigare il rischio di guasto hardware (Hardware Failure) del Nodo 1 (HP Compaq, IP `.29`), è stato implementato un failover a freddo sul Nodo 2 (Lenovo ThinkCentre, IP `.40` - Network Director).
*   **Tecnologia:** Trasferimento via `SCP` (Secure Copy Protocol) autenticato tramite chiavi SSH asimmetriche, schedulato alle ore **02:20 AM**.
*   **Risultato:** In caso di perdita totale del Nodo 1, il Nodo 2 possiede una replica speculare del cluster pronta per il ripristino immediato dei servizi.

### Fase 3: Off-Site Cloud Exfiltration & Troubleshooting
Il terzo livello della regola 3-2-1 prevede lo storage in un sito geograficamente separato (Off-Site). 
*   **Troubleshooting Architetturale:** I tentativi iniziali di deploy tramite container dedicati su Portainer hanno restituito errori di registry (`pull access denied`) dovuti a immagini obsolete. La soluzione è stata un *pivoting* a livello di sistema operativo (Bare-Metal): installazione del tool nativo Linux `megatools`.
*   **Esecuzione:** Schedulazione del push sul Cloud alle ore **03:00 AM**. Durante la fase di testing è stata gestita un'eccezione **HTTP 402 (Payment Required / Quota Exceeded)**, risolta tramite data-cleaning e ottimizzazione della retention lato Cloud (Un errore della logica di calcolo dello spazio sugli account mega impedica ai file di essere caricati).

---

## 🚨 SecOps: "No-Noise" Alerting & Automation

Il problema principale dei sistemi di monitoraggio è la desensibilizzazione dell'operatore dovuta alle continue notifiche di "Successo" (Alert Fatigue).
Per risolvere questo problema, l'automazione di backup è stata "wrappata" in uno script Bash che funge da supervisore logico:

1. Lo script cattura l'`exit code` del processo `megatools`.
2. Se il processo va a buon fine (`exit code 0`), il sistema esce in modo silenzioso (Silent Exit).
3. **Se il processo fallisce**, lo script innesca un Webhook verso le **API di Telegram**, inviando una notifica Push in tempo reale sullo smartphone dell'amministratore contenente l'errore specifico (es. saturazione banda/disco).

Questo approccio garantisce l'attenzione dell'operatore *esclusivamente* quando è richiesto un intervento manuale di Incident Response.

---

## 🔄 Lifecycle Management: Watchtower
A chiusura della finestra di manutenzione notturna (Maintenance Window), alle ore **04:00 AM**, entra in esecuzione il container **Watchtower**.
Il suo scopo è la *Continuous Delivery* e il patch management: analizza le immagini dei container in esecuzione, le aggiorna alle ultime versioni patchate disponibili e, tramite il flag `--cleanup`, distrugge le immagini orfane per prevenire l'esaurimento dello spazio su disco dell'hardware legacy.

---
