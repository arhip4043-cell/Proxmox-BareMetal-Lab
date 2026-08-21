## DevOps & Dynamic App Deployment (Node.js + SQLite)

![Node.js](https://img.shields.io/badge/Node.js-Backend-339933?style=for-the-badge&logo=nodedotjs)
![SQLite](https://img.shields.io/badge/SQLite-Database-003B57?style=for-the-badge&logo=sqlite)
![DevOps](https://img.shields.io/badge/DevOps-Containerization-blue?style=for-the-badge)

Dopo aver stabilizzato l'infrastruttura Zero Trust, il test successivo ha riguardato l'erogazione di un servizio Web Dinamico (MEGA Dashboard), passando da una logica di puro web-serving statico a una vera architettura applicativa *Stateful*.

### L'Architettura Applicativa
L'applicativo che il server doveva ospitare non era statico (HTML/CSS), ma richiedeva un *Runtime Environment* **Node.js** e si appoggiava a un database relazionale **SQLite** per la persistenza dei dati. Per ottimizzare il *Compute Budget* del server legacy, il vecchio container Nginx è stato decommissionato in favore di uno Stack applicativo dedicato.

### Troubleshooting: Cross-Architecture Dependency Mismatch
Durante la migrazione dei file dal computer host (macOS) al server di produzione (Debian Linux), il container andava in crash tentando di interrogare il database. 

**Root Cause Analysis:** La libreria SQLite utilizza *C-bindings* pre-compilati. Trasferendo l'intera directory `node_modules` creata su architettura Darwin/ARM, i binari risultavano incompatibili con il Kernel Linux/x86_64 dell'host.
**Remediation:** Cancellazione forzata della cache delle dipendenze (`node_modules`) e configurazione dello startup command del container in `sh -c "npm install && node server.js"`. Questo ha forzato Docker a ricompilare i driver nativi per Linux al momento del boot, stabilizzando l'applicativo.

### Container Orchestration (Portainer Stack)
Il deployment è stato eseguito creando un nuovo Stack su Portainer:
* **Image Optimization:** Utilizzo dell'immagine ufficiale `node:alpine`. La scelta della distribuzione *Alpine Linux* (basata su `musl libc` e `BusyBox`) riduce drasticamente l'impronta in RAM e la superficie d'attacco rispetto alle immagini Node.js standard.
* **Network Mapping:** Isolamento dell'applicativo esponendo la porta interna `4000` del container sulla porta custom `8085` dell'host.

![Specifiche hardware del router](https://github.com/arhip4043-cell/Proxmox-BareMetal-Lab/blob/main/ServerWeb.png?raw=true)

### Storage & Permission Management
Essendo SQLite un database basato su file singolo, richiedeva l'accesso in scrittura sul filesystem dell'host (Bind Mount). Le rigide policy di sicurezza di Debian bloccavano il container.
* **Fix Operativo:** Modifica dei permessi Unix (`chmod -R 777`) sulla directory del progetto per consentire all'utente del container Docker di eseguire le operazioni di I/O sul file `database.sqlite`.
* Propabilmente in un ambiente di produzione Enterprise esposto al pubblico, l'assegnazione dei permessi globali verrebbe sostituita da un mapping preciso dello User Namespace (UID/GID) tra host e container per rispettare il principio del Least Privilege).*
