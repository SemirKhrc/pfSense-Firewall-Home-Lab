# pfSense Firewall Home Lab — Simulazione e Mitigazione Attacco DoS

Lab di cybersecurity virtualizzato che dimostra segmentazione di rete, simulazione di un attacco DoS e mitigazione in tempo reale tramite regole firewall.

---

## Panoramica

Questo progetto riproduce uno scenario realistico attacco/difesa su una rete virtualizzata composta da tre macchine: un firewall pfSense, una macchina attaccante (Kali Linux) e una vittima (Ubuntu). L'obiettivo è osservare il comportamento del traffico malevolo a livello di rete e dimostrare come un firewall perimetrale possa bloccarlo in tempo reale.

---

## Topologia

```
[Router di casa 192.168.0.1]
        |
        | (Adapter Bridged — 192.168.0.0/24)
        |
   +---------+                 +-----------------------+
   |  Kali   |                 |       pfSense         |
   | Linux   |  hping3 flood   |  WAN: 192.168.0.x     |
   | .0.200  | --------------> |  LAN: 192.168.1.1     |
   +---------+  (via route     +-----------+-----------+
                 statica)                  |
                                (LabNet — 192.168.1.0/24)
                                           |
                                  +------------------+
                                  |  Ubuntu Desktop  |
                                  |  192.168.1.100   |
                                  +------------------+
```

| Dispositivo | Rete | IP | Ruolo |
|---|---|---|---|
| Kali Linux | Bridged | 192.168.0.200/24 | Attaccante |
| pfSense WAN | Bridged | 192.168.0.x/24 | Firewall perimetrale |
| pfSense LAN | Internal (LabNet) | 192.168.1.1/24 | Default GW + DHCP |
| Ubuntu Desktop | Internal (LabNet) | 192.168.1.100/24 | Vittima |

---

## Setup Rapido

**Requisiti:** VirtualBox ≥ 7.x, ISO di pfSense CE, Kali Linux e Ubuntu 22.04, host con ≥ 8 GB RAM.

Le tre VM vengono create in VirtualBox con adapter configurati come da topologia. pfSense gestisce il DHCP sulla LAN interna (`192.168.1.100–199`) e funge da default gateway per Ubuntu. Su Kali viene aggiunta una route statica verso la subnet interna tramite l'IP WAN di pfSense, in modo da simulare un attaccante esterno che raggiunge un host interno.

```bash
# Kali — aggiunta route verso la LAN interna
sudo ip route add 192.168.1.0/24 via <IP_WAN_PFSENSE>
```

---

## Cosa Viene Dimostrato

### 1. Segmentazione di rete
pfSense separa fisicamente la rete esterna (WAN/Bridged) dalla rete interna (LAN/LabNet). Ubuntu non è direttamente raggiungibile dalla rete esterna senza una regola esplicita sul firewall — questo è il principio base della segmentazione perimetrale.

### 2. Simulazione attacco DoS
Da Kali viene lanciato un flood ICMP e SYN verso Ubuntu usando `hping3`:

```bash
sudo hping3 -1 --flood 192.168.1.100        # ICMP flood
sudo hping3 --flood -S -p 80 192.168.1.100  # SYN flood
```

Wireshark su Ubuntu cattura in tempo reale il picco di traffico in ingresso, rendendo visibile l'attacco a livello di pacchetto.

### 3. Mitigazione tramite regola firewall
Senza interrompere le VM, viene aggiunta una regola di blocco su pfSense (interfaccia WAN, priorità massima):

```
Azione: Block | Sorgente: IP Kali | Destinazione: 192.168.1.100 | Log: abilitato
```

Immediatamente dopo l'applicazione della regola, il traffico di flood su Wireshark si azzera. I log di pfSense registrano ogni pacchetto bloccato con IP sorgente, timestamp e protocollo.

### 4. Verifica nei log
pfSense → **Status → System Logs → Firewall** mostra le voci di blocco in tempo reale, confermando che la regola è attiva e funzionante.

---

## Competenze Applicate

| Area | Tecnologia / Tecnica |
|---|---|
| Segmentazione di rete | Zone WAN/LAN, VirtualBox Internal Network |
| Firewall management | pfSense — ACL stateful, ordinamento regole, logging |
| Routing | Route statiche, DHCP scope, default gateway |
| Attacco simulato | `hping3` — ICMP flood, SYN flood |
| Analisi del traffico | Wireshark — cattura live, lettura pacchetti |
| Incident verification | Correlazione log firewall e cattura di rete |

---

## Struttura del Repository

```
/
├── README.md
├── screenshots/
│   ├── wireshark-flood.png       # Traffico durante l'attacco
│   ├── wireshark-post-block.png  # Traffico dopo il blocco
│   └── pfsense-logs.png          # Log firewall con pacchetti bloccati
└── topology.png                  # Diagramma di rete
```

---

## Riferimenti

- [Documentazione pfSense](https://docs.netgate.com/pfsense/en/latest/)
- [Man page hping3](https://linux.die.net/man/8/hping3)
- [Guida utente Wireshark](https://www.wireshark.org/docs/wsug_html/)
