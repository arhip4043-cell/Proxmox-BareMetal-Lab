# Enterprise Home Datacenter: Bare-Metal Proxmox VE Deployment

![Proxmox VE](https://img.shields.io/badge/Proxmox_VE-8.4-E57000?style=for-the-badge&logo=proxmox)
![Debian](https://img.shields.io/badge/Debian-12_Bookworm-A81D33?style=for-the-badge&logo=debian)
![LXC](https://img.shields.io/badge/LXC-Linux_Containers-linux?style=for-the-badge&logo=linux)
![Networking](https://img.shields.io/badge/Networking-Bridge_&_VLAN-005571?style=for-the-badge)

## Executive Summary
Questo repository documenta la transizione architetturale di un hardware legacy (ASUS Intel Core i7 2.4GHz 5th Gen Quad-Core, 8GB RAM) in un vero e proprio **Datacenter basato su Hypervisor di Tipo 1 (Proxmox VE)**. 

L'obiettivo del progetto non è la semplice virtualizzazione, ma documentare la creazione di un ambiente *Enterprise-grade* altamente ottimizzato per ospitare infrastrutture di test in ambito **Cyber Security, SIEM Deployment, e OT Network Simulation**. Il documento traccia il processo di *Problem-Solving* adottato per aggirare i limiti del firmware originale e implementare il sistema "The Debian Way".

---

## Fasi di Deployment e Troubleshooting Ingegneristico

### Fase 1: Disaster Recovery & Data Preservation
Prima di formattare il disco (160 GB) della macchina Asus, è stata applicata una rigorosa procedura di salvaguardia dei dati esistenti. 
Tramite un ambiente Live (MiniOS) e interfaccia a riga di comando, è stata eseguita una **clonazione fisica bit-a-bit** verso un HDD esterno da 1TB utilizzando il comando atomico `dd`. Questo ha garantito il backup a freddo dei dati del precedente OS (Zorin OS) e di tutti i pacchetti Python pre-configurati.

### Fase 2: Bare-Metal Hardening & BIOS Tuning
L'installazione diretta dell'ISO di Proxmox ha generato conflitti hardware con il firmware Aptio dell'host:
1. **Boot Loop (Optical Drive Bug):** Il sistema forzava la ricerca dell'ISO sul device ottico vuoto (`/dev/sr0`), bypassando l'USB.
2. **Kernel Panic:** Il kernel di Proxmox 8.4 andava in crash durante l'allocazione della memoria con errore `invalid buffer alignment`.

**La Soluzione:** Accesso avanzato al BIOS per la disattivazione del **Secure Boot** e l'abilitazione del **Launch CSM (Legacy Boot)**, forzando la gestione della memoria in modalità compatibile e bypassando i colli di bottiglia dell'interfaccia UEFI.

### Fase 3: "The Debian Way" Headless Deployment
Vista l'instabilità dell'installer nativo di Proxmox sull'hardware specifico, è stata adottata la best-practice dei datacenter: il deployment in due step.
1. **Installazione Headless:** Preparazione di una USB con `Debian 12 Netinst` tramite BalenaEtcher su macOS (Apple Silicon). Installazione minimale, rimuovendo ogni ambiente grafico (GUI) e ottimizzando la CPU esclusivamente per il demone `OpenSSH-Server`.
2. **SSH Key Management:** Per poter controllare la macchina Asus Debian da remoto è stato necessario risolvere il blocco crittografico macOS (`REMOTE HOST IDENTIFICATION HAS CHANGED`) tramite pulizia del file `known_hosts`, garantendo accesso sicuro via terminale come utente `root`.
3. **Elevazione a Proxmox VE:** Tramite CLI, sono stati inseriti i repository ufficiali Proxmox. L'esecuzione di `apt install proxmox-ve` ha trasformato l'installazione Debian in un Hypervisor puro, pulito e nativo.

### Fase 4: Datacenter Optimization & Networking
Con l'accesso alla dashboard web (GUI) sulla porta `8006`, l'infrastruttura è stata finalizzata per l'operatività h24:
* **Networking (Linux Bridge):** Creazione manuale del bridge virtuale `vmbr0`, connessa all'interfaccia Ethernet fisica per garantire connettività trasparente al modem/router.
* **Ottimizzazione Energetica:** Modifica del parametro `HandleLidSwitch=ignore` nel file `/etc/systemd/logind.conf` di Linux. Il server opera ora a "coperchio chiuso", minimizzando consumi elettrici e ingombro fisico.
* **Repository Tuning:** Disattivazione dei repository *Enterprise* e attivazione della suite *No-Subscription* per garantire aggiornamenti profondi del kernel via `apt full-upgrade`. Disabilitazione del pop-up di subscription tramite script Bash.

---

## Fase 5: LXC Provisioning & Primi Servizi Online
Per testare la capacità di virtualizzazione e abbattere il *Compute Budget*, l'infrastruttura è stata basata su **Linux Containers (LXC)**, che condividendo il kernel dell'host garantiscono latenze quasi nulle rispetto alle VM KVM tradizionali.

| CT ID | Servizio | Descrizione Operativa | Status |
| :---: | :--- | :--- | :---: |
| **100** | **Pi-hole (DNS Sinkhole)** | Configurato con IP Statico (`192.168.1.50`). Agisce come server DNS primario per abbattere il traffico pubblicitario della LAN e fungere da primo layer di detection contro beaconing malware. | 🟢 Attivo |
| **101** | **Rocky Linux 9 (Web Server)**| Ambiente RHEL-based puro. Provisioning completato con stack `httpd` (Apache) in esecuzione per test di Application Security e Web Hosting. | 🟢 Attivo |

---

## Progetti Futuri
L'infrastruttura `pve-lab` funge da fondamenta per i successivi progetti. Le future implementazioni (che verranno documentate) includono:

- Segmentazione di rete avanzata tramite VLAN (DMZ / Produzione).
- Deploy di **Advanced Python SIEM** in architettura distribuita.
- Configurazione di sonde di rete e IDS (es. Suricata/Snort) sul traffico in ingresso.
- Test in ambiente isolato su vettori OT (Operational Technology) e Automotive.

