# Performance Audit & QoS: Stress-Testing Legacy Hardware (2005)

![Benchmark](https://img.shields.io/badge/Benchmark-ApacheBench-D22128?style=for-the-badge)
![CPU](https://img.shields.io/badge/CPU-AMD_Turion_64-black?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Zero_Packet_Drop-success?style=for-the-badge)

## Executive Summary
L'obiettivo di questo test è misurare la resilienza e il *Compute Budget* reale dell'infrastruttura. È facile far girare container su server moderni da 32-core. La vera sfida ingegneristica è erogare un'applicazione dinamica (Node.js + SQLite containerizzata via Docker) su un processore **Single-Core con architettura di oltre due decenni fa**, garantendo la Business Continuity sotto stress (DDoS / High-Volume Traffic simulation).

I risultati dimostrano che l'eliminazione dell'overhead grafico (Debian Headless) e l'uso di container minimi (Alpine Linux) permettono a un hardware "obsoleto" di gestire carichi di produzione con **zero errori di risposta**.

---

## 1. Hardware Baseline (L'Analisi del Silicio)
Le caratteristiche dei parametri hardware evidenzia i colli di bottiglia fisici della macchina ospitante:

*   **CPU Model:** `AMD Turion(tm) 64 Mobile ML-40`
*   **Architecture:** `1 Core / 1 Thread` (Assenza totale di parallelismo hardware/Hyper-Threading).
*   **Clock Speed:** `2.20 GHz`
*   **L2 Cache:** `1 MiB` (Estremamente limitata per gli standard attuali).
*   **Security Mitigations:** Le patch per `Spectre v1/v2` sono attive a livello Kernel, il che impone un ulteriore degrado fisiologico delle performance per la sanificazione dei puntatori in memoria.

> **Analisi:** Il server non ha capacità di calcolo parallelo. Ogni singola richiesta web, query al database o pacchetto di rete deve mettersi in fila ed essere processata in modo sequenziale dall'unico core disponibile.

---

## 2. Quality of Service (QoS) & Stress Test
Per validare l'architettura, l'endpoint dell'applicazione Node.js (`http://192.168.1.29:8085/`) è stato sottoposto a tre round di stress-test incrementali utilizzando **ApacheBench (`ab`)**, simulando picchi di traffico concorrente.

### 🟢 Round 1: The Warm-Up (500 Requests / 10 Concurrency)
Un carico leggero per testare il tempo di risposta base del web server e del database SQLite.
*   **Requests per second (RPS):** `272.89 #/sec`
*   **Median Latency:** `33 ms`
*   **Failed Requests:** `0`
*   **Esito:** Il single-core gestisce le 10 connessioni simultanee con assoluta fluidità. L'I/O sul disco e il database SQLite rispondono in tempi ottimali.

![Specifiche hardware del router](https://github.com/arhip4043-cell/Proxmox-BareMetal-Lab/blob/main/stressTest1.png?raw=true)
---

### 🟡 Round 2: The Sweet Spot (1000 Requests / 20 Concurrency)
Il carico viene raddoppiato per trovare il picco di massima efficienza (Throughput massimo) del processore.
*   **Requests per second (RPS):** `360.65 #/sec`  *(Picco massimo registrato)*
*   **Median Latency:** `47 ms`
*   **Transfer Rate:** `10.47 MB/sec`
*   **Failed Requests:** `0`
*   **Esito:** A 20 connessioni simultanee, il server raggiunge il suo stato di grazia. Riesce a servire 360 richieste dinamiche al secondo mantenendo il 50% delle risposte sotto i 47 millisecondi. Un risultato sbalorditivo per una CPU del 2003.

![Specifiche hardware del router](https://github.com/arhip4043-cell/Proxmox-BareMetal-Lab/blob/main/stressTest2.png?raw=true)
---

### 🔴 Round 3: The Saturation Point (2000 Requests / 40 Concurrency)
Il test definitivo per mandare in Denial of Service (DoS) la CPU. A 40 connessioni simultanee, il *Context Switching* dell'unico core disponibile diventa il collo di bottiglia fisico.
*   **Requests per second (RPS):** `324.30 #/sec`
*   **Median Latency:** `118 ms` (Max `173 ms`)
*   **Failed Requests:** `0` 🛡️
*   **Esito:** La latenza aumenta fisiologicamente (il processore deve mettere in attesa le richieste), ma il dato fondamentale è **Zero Failed Requests**. Il Web Server e Docker hanno "messo in coda" perfettamente tutte le 2.000 richieste senza scartare o corrompere un singolo pacchetto TCP. 

![Specifiche hardware del router](https://github.com/arhip4043-cell/Proxmox-BareMetal-Lab/blob/main/stressTest3.png?raw=true)
---

## 3. Conclusioni Architetturali (Engineering Takeaway)

I dati raccolti certificano che **l'hardware non è il limite, se l'architettura software è progettata per l'efficienza.**

Un processore mobile single-core del 2003 è stato in grado di:
1. Sostenere un trasferimento dati di quasi **60 Megabyte in 6 secondi** (`59494000 bytes`).
2. Mantenere l'uptime del database SQLite integrato senza incorrere in *Database Locks* dovuti alla concorrenza.
3. Dimostrare una stabilità del demone Docker impeccabile, rifiutando di collassare sotto saturazione.

Questo Audit dimostra che l'eliminazione del "Debito Tecnico" (rimozione della GUI dell'OS, utilizzo di container minimi Alpine, networking isolato) permette di trasformare hardware destinato al macero in un nodo infrastrutturale *Stateful* perfettamente capace di gestire carichi di produzione reali.
