# Zero Trust Architecture on Legacy Hardware (2003)

![Docker](https://img.shields.io/badge/Docker-Microservices-2496ED?style=for-the-badge&logo=docker)
![Tailscale](https://img.shields.io/badge/Tailscale-Zero_Trust-FFFFFF?style=for-the-badge&logo=tailscale)
![Caddy](https://img.shields.io/badge/Caddy-Reverse_Proxy-00A2FF?style=for-the-badge)
![Vaultwarden](https://img.shields.io/badge/Vaultwarden-Credential_Vault-175DDC?style=for-the-badge&logo=bitwarden)

## Executive Summary
Questo progetto documenta il recupero di un hardware obsoleto (Laptop HP Compaq del 2003) e la sua trasformazione in un server sicuro per la gestione delle credenziali (Password Vault). L'obiettivo ingegneristico è stato l'azzeramento della superficie di attacco (Attack Surface) sfruttando containerizzazione e reti overlay (SDP - Software Defined Perimeter), senza aprire alcuna porta sul firewall perimetrale.

---

## Architettura e Implementazione

### 1. Bare-Metal Hardening
*   **Disk Wiping:** Bonifica a basso livello del disco di destinazione (`/dev/sda`) tramite utility `wipefs` e `parted` per la rimozione di firme corrotte e partizioni preesistenti.
*   **OS Deployment:** Installazione di `Debian 12` in modalità rigorosamente *headless* (CLI-only). L'assenza di GUI ha permesso di massimizzare il *Compute Budget* per l'orchestrazione dei servizi, ovviando ai limiti di RAM e CPU dell'hardware legacy.

### 2. Containerization (Docker Stack)
Abbandonato il deployment nativo per evitare conflitti di dipendenze, l'intero stack è stato orchestrato tramite **Docker**.
*   **Portainer:** Deployato in ascolto sulla porta TCP `9443` per garantire la gestione centralizzata dei container via WebGUI dalla rete locale.

### 3. Zero Trust Network Access (ZTNA)
La sfida principale per l'erogazione di un Password Manager è l'esposizione sicura. Si è evitato il tradizionale (e insicuro) Port Forwarding implementando un'architettura Zero Trust:
*   **Tailscale:** Creazione di un mesh network basato su *WireGuard*. Il server comunica esclusivamente all'interno del tunnel cifrato.
*   **SSL Offloading & PKI:** Implementazione di **Caddy Server** come Reverse Proxy. Caddy è configurato per interfacciarsi con il *MagicDNS* di Tailscale, ottenendo e rinnovando automaticamente un certificato SSL/TLS valido (`*.ts.net`) per la crittografia *Data-in-Transit*.

### 4. Il Payload: Vaultwarden
Al centro dello stack è stato deployato **Vaultwarden**, implementazione alternativa dell'API di Bitwarden scritta in *Rust* per garantire estrema leggerezza computazionale e sicurezza della memoria (Memory Safety). 

## Risultato Finale
L'istanza è ora raggiungibile in HTTPS puro. Lo stack operativo risulta completamente invisibile a scanner perimetrali esterni (es. Shodan / Nmap mass-scans) poiché non espone porte fisiche su Internet, azzerando il rischio di vulnerabilità legate a servizi esposti.
