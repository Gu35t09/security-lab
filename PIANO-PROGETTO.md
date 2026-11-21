# Security Lab: documento di progetto

Stato: bozza di progettazione, nessuna implementazione.
Ultimo aggiornamento: 2026-11-19.

## 1. Obiettivo

Costruire un laboratorio di test per imparare e sperimentare con la piattaforma
di sicurezza open source più completa che l'hardware disponibile può reggere.
Il laboratorio non è produzione. Deve restare completamente separato
dall'ambiente gestito da `ansible-prod-srv`, senza richiedere modifiche ad esso.

## 2. Decisioni prese

| Tema | Decisione |
|---|---|
| Switch | Gestito, con VLAN e porta di mirroring disponibili |
| Windows e Active Directory | Sì, ottimizzato per la RAM minima. Accettabile la lentezza e l'alternanza tra scenario Windows e scenario Linux |
| Internet | I sistemi di sicurezza escono tramite allowlist. Le VM bersaglio e di attacco non escono mai |
| Firewall | OPNsense dedicato al lab, collegato direttamente al router dell'operatore, in parallelo all'OPNsense di produzione |
| Instradamento | NAT in uscita sull'OPNsense del lab. Rotte statiche sul router solo per raggiungere il lab dal notebook di produzione senza VPN |
| Mirroring | Mirror di Open vSwitch su `nw2` e sul N100, ognuno con un cavo diretto verso `nw1` |
| Scheda di rete di `nw1` | 4x 1 GbE al posto della 1x 2.5 GbE |
| SIEM aggiuntivo | Graylog Open, in parallelo a Security Onion |
| Virtualizzazione | Proxmox VE in cluster su tre nodi |
| Storage | ext4 con LVM-thin, l'installazione predefinita di Proxmox. Niente ZFS, niente Ceph, niente repliche |
| Switch | Tutto a 1 GbE |
| Automazione | Nuovo repository con le stesse convenzioni dei repository di produzione |

## 3. Hardware e ruoli dei nodi

| Nodo | Hardware | Ruolo |
|---|---|---|
| `nw1` | NodeWeaver, Xeon 4c/8t, 32 GB, SSD 2 TB, 2x 1 GbE integrate, scheda 4x 1 GbE | Security Onion standalone |
| `nw2` | NodeWeaver, Xeon 4c/8t, 24 GB, SSD 2 TB, 2x 1 GbE, 1x 2.5 GbE | Bersagli, Kali, Graylog |
| `n100` | Mini PC N100 4c/4t, 32 GB, SSD 1 TB, 4x 2.5 GbE | OPNsense del lab, Wazuh, Caldera, servizi SOC leggeri, quorum |

La NodeWeaver da 32 GB deve essere `nw1`. Security Onion standalone chiede
almeno 24 GB e non entra nel nodo da 24 GB.

Riserve per ogni nodo:

| Voce | RAM |
|---|---|
| Sistema Proxmox | 2 GB |
| RAM utile su `nw1` e `n100` | circa 30 GB |
| RAM utile su `nw2` | circa 22 GB |

### 3.0 Storage

Installazione standard di Proxmox su ext4: il sistema sta su un volume ext4,
le VM su un pool LVM-thin nello stesso disco.

LVM-thin non usa RAM per la cache e conserva le due funzioni che servono al lab:
- **snapshot** delle VM, per tornare allo stato "pulito" dopo ogni attacco;
- **cloni collegati** dai template, che risparmiano disco.

Ogni nodo ha il suo storage locale. Senza repliche, una VM vive solo sul suo
nodo, e se il disco si guasta la VM si recupera solo dal backup.
Le ISO e i backup stanno su uno storage a directory, sempre ext4.
Un backup sullo stesso disco della VM non protegge dal guasto del disco. I
backup delle VM SOC vanno quindi su un altro nodo, condiviso via NFS, oppure
su un disco esterno.

Su `nw2` va attivato KSM, la deduplicazione della memoria di Proxmox.
Più VM Windows identiche condividono molte pagine e il risparmio è concreto.

### 3.1 OPNsense del lab: virtuale sul N100

Raccomandazione: OPNsense come VM sul N100, con la porta WAN assegnata in
passthrough PCI alla VM.

Motivi:
- il N100 resta il terzo nodo del cluster, necessario per il quorum;
- le quattro porte bastano per WAN, LAN, cluster e mirror verso Security Onion;
- con il passthrough l'host Proxmox non ha mai un indirizzo sulla rete del router.

Ripiego se il passthrough non funziona: un bridge dedicato sulla porta WAN,
senza indirizzo IP sull'host.

Conseguenza accettata: se il N100 si spegne, tutto il lab resta senza rete.

Alternativa scartata: OPNsense installato direttamente sul N100. Richiederebbe
un altro dispositivo per il quorum e toglierebbe 32 GB di RAM al cluster.

## 4. Rete

### 4.1 Collegamento al router

```
Router operatore
 ├── OPNsense produzione  (invariato)
 └── OPNsense lab, WAN    (nuovo, sul N100)
        └── VLAN del lab
```

Rotte statiche sul router:

| Destinazione | Gateway |
|---|---|
| 10.66.0.0/16 | indirizzo WAN dell'OPNsense del lab |

Funzionamento deciso:
- **Uscita verso Internet:** l'OPNsense del lab applica il NAT in uscita sulla WAN. Il router vede solo l'indirizzo WAN del lab.
- **Ingresso dal notebook di produzione:** il router instrada 10.66.0.0/16 verso la WAN del lab. Il notebook raggiunge LAB-MGMT senza VPN.
- **Indirizzo WAN fisso** per l'OPNsense del lab, con prenotazione DHCP o indirizzo statico.

Percorso del traffico dal notebook:

```
notebook  ->  OPNsense prod  ->  router  ->  OPNsense lab  ->  LAB-MGMT
notebook  <-  OPNsense prod  <----------------  OPNsense lab
```

Le risposte non ripassano dal router: l'OPNsense del lab risponde direttamente
alla WAN dell'OPNsense di produzione, che sta sulla stessa rete. È un percorso
asimmetrico. Le due OPNsense lo gestiscono senza problemi, ma alcuni router
degli operatori scartano le connessioni TCP di cui vedono una sola direzione.
Va provato in fase 1. Se il router le scarta, l'unica correzione è una rotta
statica verso 10.66.0.0/16 sull'OPNsense di produzione, cioè una piccola
modifica alla produzione.

### 4.2 Segmenti

ID VLAN e sottoreti sono segnaposto. Vanno verificati contro il piano di
indirizzi di produzione per evitare sovrapposizioni.

| VLAN | Nome | Sottorete | Contenuto |
|---|---|---|---|
| 600 | LAB-MGMT | 10.66.0.0/24 | Interfacce Proxmox, web UI degli strumenti |
| 602 | LAB-CLUSTER | 10.66.2.0/24 | Traffico corosync del cluster Proxmox, senza gateway |
| 610 | LAB-SOC | 10.66.10.0/24 | Security Onion, Graylog, Wazuh, Velociraptor, MISP, IRIS, Shuffle |
| 620 | LAB-TGT-WIN | 10.66.20.0/24 | Dominio AD e client Windows |
| 621 | LAB-TGT-LNX | 10.66.21.0/24 | Server Linux vulnerabili |
| 630 | LAB-ATTACK | 10.66.30.0/24 | Kali, Caldera |
| 640 | LAB-BUILD | 10.66.40.0/24 | Preparazione dei template con Internet in allowlist |

Il traffico copiato non usa una VLAN: viaggia su un cavo diretto, vedi 4.5.

Indirizzi fissi principali:

| Host | Indirizzo |
|---|---|
| OPNsense lab, gateway di ogni VLAN | .1 |
| `nw1`, `nw2`, `n100` su LAB-MGMT | 10.66.0.11, .12, .13 |
| `nw1`, `nw2`, `n100` su LAB-CLUSTER | 10.66.2.11, .12, .13 |

### 4.3 Regole di traffico

L'OPNsense del lab è l'unico filtro tra il lab e il resto del mondo.
L'OPNsense di produzione protegge comunque la produzione, perché vede il lab
come qualsiasi altro host sul lato WAN.

| Da | Verso | Consentito |
|---|---|---|
| Accesso amministrativo, vedi 4.4 | LAB-MGMT | HTTPS e SSH |
| LAB-MGMT | Qualsiasi VLAN lab | Gestione |
| LAB-SOC | Internet | Solo allowlist: repository dei pacchetti, regole ET, feed MISP, GitHub |
| LAB-BUILD | Internet | Solo allowlist: Windows Update, mirror Debian, Ubuntu e Kali |
| LAB-TGT-WIN e LAB-TGT-LNX | LAB-SOC | Solo le porte degli agenti: Wazuh, Velociraptor, Elastic Agent, Graylog Sidecar |
| LAB-ATTACK | LAB-TGT-WIN e LAB-TGT-LNX | Tutto |
| LAB-ATTACK | LAB-SOC | Niente, salvo esercizi specifici |
| Qualsiasi VLAN lab | Rete del router e indirizzi privati esterni al lab | Mai, tranne il router come gateway |
| Bersagli e attacco | Internet | Mai |

La regola sulla rete del router è importante. L'OPNsense del lab condivide
quella rete con la WAN dell'OPNsense di produzione, e nessun host del lab
deve poterla esplorare.

Il DNS dei bersagli è il controller di dominio o l'OPNsense del lab, senza
inoltro verso l'esterno.

### 4.4 Accesso amministrativo

Accesso normale: dal notebook `v-admin`, attraverso le rotte statiche, senza VPN.

L'OPNsense di produzione applica il NAT verso il router. Il lab vede quindi
ogni host della produzione con lo stesso indirizzo, la WAN dell'OPNsense di
produzione, e non distingue il notebook dal server. Contromisure:
- **Regola sull'OPNsense del lab:** LAB-MGMT accetta HTTPS e SSH solo dall'indirizzo WAN dell'OPNsense di produzione. Tutto il resto della rete del router è bloccato.
- **SSH con YubiKey**, come già avviene verso il server di produzione.
- **Secondo fattore** sulle interfacce web di Proxmox e OPNsense, con WebAuthn sulla YubiKey o TOTP.

Se l'uscita del notebook dall'OPNsense di produzione è in allowlist, serve una
regola che consenta 10.66.0.0/16. Se l'uscita è libera, non serve nulla.

Accesso di emergenza: una porta dello switch configurata in accesso su
LAB-MGMT. Si collega il notebook col cavo quando il percorso via router non
funziona. Notebook e host Proxmox stanno sulla stessa sottorete, quindi
l'accesso funziona anche con l'OPNsense del lab spento.

### 4.5 Porte fisiche

Su `nw1` la scheda PCIe 1x 2.5 GbE viene sostituita con una scheda 4x 1 GbE.
`nw1` passa così a sei porte 1 GbE, due integrate e quattro sulla scheda.

```
                              Internet
                                 |
                    +------------+------------+
                    |    Router operatore     |
                    |  rotta 10.66.0.0/16     |
                    |  via WAN OPNsense lab   |
                    +----+---------------+----+
                         |               |
              +----------+-------+       |
              | OPNsense prod    |       |
              | invariato        |       |
              +----------+-------+       |
                         |               |
                 rete produzione         |
                 notebook v-admin        |
                                         | p1, WAN in passthrough
                         +---------------+---------------+
                         |             n100              |
                         |  VM OPNsense lab, Wazuh, SOC  |
                         |      p2         p3        p4  |
                         +-------+----------+---------+--+
                                 |          |         |
                            trunk tutte   VLAN 602    | mirror N100:
                            le VLAN lab   corosync    | gestione, SOC, build
                                 |          |         |
        +------------------------+----------+------+  |
        |        Switch gestito, tutto 1 GbE       |  |  porta VLAN 600:
        |                                          +--+--- emergenza,
        +--+--------+--------+-------+--------+----+  |   cavo al notebook
           |        |        |       |        |       |
        trunk  VLAN 602    SPAN   trunk  VLAN 602     |
           |        |        |       |        |       |
    +------+--------+--------+--+ +--+--------+-----+ |
    |  nw1   32 GB              | |  nw2   24 GB    | |
    |  int. n.1   trunk         | |  1G n.1  trunk  | |
    |  int. n.2   corosync      | |  1G n.2  coros. | |
    |  scheda n.3 SPAN switch   | |                 | |
    |  scheda n.1 ascolto  +----+-+--+ 2.5G uscita  | |
    |  scheda n.2 ascolto  +----+-+  | mirror OVS   | |
    |  scheda n.4 riserva  |    | |  +--------------+ |
    |  Security Onion      +----+-+-------------------+
    +-------------------------+
         cavi diretti: solo traffico copiato,
         nessun IP, non passano dallo switch
```

| Nodo | Porta | Collegamento | Velocità effettiva | Uso |
|---|---|---|---|---|
| `n100` | 2.5 GbE n.1 | Router | Dipende dal router | WAN dell'OPNsense del lab, in passthrough |
| `n100` | 2.5 GbE n.2 | Switch, trunk | 1 GbE | Tutte le VLAN del lab |
| `n100` | 2.5 GbE n.3 | Switch, VLAN 602 | 1 GbE | Corosync |
| `n100` | 2.5 GbE n.4 | Cavo diretto verso `nw1`, scheda n.2 | 1 GbE | Uscita del mirror del N100 |
| `nw1` | Integrata 1 GbE n.1 | Switch, trunk | 1 GbE | Gestione e VLAN SOC |
| `nw1` | Integrata 1 GbE n.2 | Switch, VLAN 602 | 1 GbE | Corosync |
| `nw1` | Scheda 1 GbE n.1 | Cavo diretto da `nw2` | 1 GbE | Ascolto: bersagli e attacco |
| `nw1` | Scheda 1 GbE n.2 | Cavo diretto dal N100 | 1 GbE | Ascolto: gestione, SOC, build |
| `nw1` | Scheda 1 GbE n.3 | Switch, destinazione SPAN | 1 GbE | Ascolto: esperimenti sul traffico fisico |
| `nw1` | Scheda 1 GbE n.4 | Libera | | Riserva |
| `nw2` | 1 GbE n.1 | Switch, trunk | 1 GbE | Traffico delle VM |
| `nw2` | 1 GbE n.2 | Switch, VLAN 602 | 1 GbE | Corosync |
| `nw2` | 2.5 GbE | Cavo diretto verso `nw1` | 1 GbE | Uscita del mirror di `nw2` |
| Switch | Una porta | Accesso VLAN 600 | 1 GbE | Accesso di emergenza |

Le porte 2.5 GbE di `nw2` e del N100 lavorano a 1 GbE, perché la scheda di
`nw1` è a 1 GbE. Per il lab basta: scansioni e attacchi generano poco traffico.
Il limite si sente solo copiando file pesanti tra VM sullo stesso nodo.

Corosync usa due anelli: il primo sulla VLAN 602 dedicata, il secondo sulla
gestione come riserva.

Il trunk a 1 GbE basta per il lab. Diventa il collo di bottiglia solo per
trasferimenti pesanti tra nodi, per esempio il backup o lo spostamento di una VM.

Compatibilità della scheda con lo slot PCIe e con lo chassis: verificata.

### 4.6 Mirroring verso Security Onion

Il traffico tra VM sullo stesso nodo non passa mai dallo switch, quindi servono
mirror interni. Ogni mirror ha un cavo diretto verso `nw1`, senza VLAN di
trasporto e senza passare dallo switch.

| Origine | VLAN copiate | Porta di arrivo su `nw1` |
|---|---|---|
| Mirror OVS su `nw2` | 620, 621, 630: bersagli e attacco | Scheda n.1 |
| Mirror OVS sul N100 | 600, 610, 640: gestione, SOC, build | Scheda n.2 |
| SPAN dello switch | A scelta, per esperimenti | Scheda n.3 |

Le tre porte di arrivo sono collegate alla VM di Security Onion come interfacce
di ascolto, senza IP. Security Onion le unisce in un'unica interfaccia logica.

Il mirror del N100 copia solo le VLAN che `nw2` non copia già. Così il traffico
tra Kali e bersagli, che passa anche dall'OPNsense, non viene visto due volte.
Alcuni flussi compariranno comunque doppi, per esempio gli agenti dei bersagli
verso Wazuh. Per un lab è accettabile.

La WAN dell'OPNsense è in passthrough e non passa da Open vSwitch, quindi
nessun mirror la vede. La copre Suricata integrato in OPNsense, attivo sulla
WAN, con gli allarmi inviati a Graylog e a Security Onion.

Se una copia supera 1 GbE, Security Onion perde pacchetti ma il lab continua
a funzionare.

## 5. Dimensionamento delle VM

### 5.1 `nw1`

| VM | vCPU | RAM | Disco | Note |
|---|---|---|---|---|
| Security Onion standalone | 8 | 26 GB | 1 TB | Minimo ufficiale 24 GB, consigliati 32. Gestione su LAB-SOC, tre interfacce di ascolto sui cavi diretti e sulla SPAN |

Senza la cache ZFS, Security Onion può avere 26 GB invece di 24.

Ripiego se la RAM non basta: modalità Eval, che chiede 8 GB ma rinuncia a
Logstash, Redis e alla cattura completa dei pacchetti.

### 5.2 `n100`

| VM o container | vCPU | RAM | Disco | VLAN |
|---|---|---|---|---|
| OPNsense lab | 2 | 2 GB | 20 GB | WAN e tutte le VLAN, come gateway |
| Wazuh all-in-one | 4 | 6 GB | 100 GB | LAB-SOC |
| Velociraptor server | 1 | 1 GB | 50 GB | LAB-SOC |
| MISP | 2 | 4 GB | 50 GB | LAB-SOC |
| DFIR-IRIS | 2 | 2 GB | 20 GB | LAB-SOC |
| Shuffle | 2 | 4 GB | 30 GB | LAB-SOC |
| Caldera | 1 | 2 GB | 20 GB | LAB-ATTACK |
| Totale | | 21 GB | 290 GB | |

Caldera è spostato qui da `nw2` per liberare RAM sul nodo da 24 GB.

Wazuh scende a 6 GB riducendo l'heap dell'indexer. I servizi di questa tabella
non devono girare tutti insieme fin dall'inizio: si accendono per fase.

### 5.3 `nw2`, servizi fissi

| VM | vCPU | RAM | Disco | Note |
|---|---|---|---|---|
| Graylog | 4 | 8 GB | 200 GB | Graylog 7.2, MongoDB e Data Node nella stessa VM. VLAN LAB-SOC |
| Kali | 2 | 3 GB | 60 GB | Desktop leggero XFCE |
| Totale | | 11 GB | 260 GB | |

Ripartizione indicativa della RAM di Graylog: 2 GB al server Graylog, 4 GB al
Data Node, 1 GB a MongoDB, 1 GB al sistema. Senza la cache ZFS Graylog torna
a 8 GB. Se serve RAM, scende a 6 GB con 1 GB al server e 3 GB al Data Node.

Graylog sta su `nw2` perché è l'unico nodo con RAM libera. È nella VLAN SOC
anche se condivide l'hypervisor con i bersagli. Per un lab il rischio di fuga
dalla VM è trascurabile.

### 5.4 `nw2`, scenario Windows

| VM | vCPU | RAM | Disco | Note |
|---|---|---|---|---|
| DC01, Windows Server Core | 2 | 2 GB | 40 GB | Controller di dominio e DNS, senza interfaccia grafica |
| SRV01, Windows Server Core | 2 | 2 GB | 40 GB | File server e servizi vulnerabili, opzionale |
| WS01, Windows 11 Enterprise | 2 | 4 GB | 60 GB | Workstation dell'utente vittima |
| Totale | | 8 GB | 140 GB | |

### 5.5 `nw2`, scenario Linux

| VM | vCPU | RAM | Disco | Note |
|---|---|---|---|---|
| LNX01, Ubuntu Server | 2 | 2 GB | 30 GB | Juice Shop e DVWA in container |
| LNX02, Metasploitable | 1 | 1 GB | 20 GB | Servizi volutamente vulnerabili |
| LNX03, Debian | 1 | 1 GB | 20 GB | Server "pulito" con agenti, per i falsi positivi |
| Totale | | 4 GB | 70 GB | |

### 5.6 Bilancio di `nw2`

| Configurazione | RAM |
|---|---|
| RAM utile | 22 GB |
| Servizi fissi e scenario Windows | 19 GB |
| Servizi fissi e scenario Linux | 15 GB |
| Servizi fissi e due scenari insieme | 23 GB, non entra |
| Due scenari insieme con Graylog a 6 GB | 21 GB, entra con 1 GB di margine |

Con 24 GB l'alternanza tra scenario Windows e scenario Linux resta il modo
normale di lavorare. Tenerli accesi insieme è possibile solo riducendo Graylog,
e con un margine troppo piccolo per un uso continuo.
I due scenari vanno raggruppati in pool di Proxmox, così si accendono e
spengono con un solo comando. KSM e ballooning danno qualche GB di margine,
ma non vanno considerati nel bilancio.

### 5.7 Strategia per contenere la RAM di Windows

- **Server Core** al posto dell'interfaccia grafica, gestito da RSAT sulla workstation.
- **Driver VirtIO e ballooning** su tutte le VM Windows.
- **KSM** attivo su `nw2`.
- **Agenti a rotazione.** Sysmon è sempre attivo. Wazuh, Elastic Agent, Graylog Sidecar e Velociraptor si attivano a seconda della fase, non tutti insieme sulla stessa VM.
- **Servizi superflui disattivati** nei template, per esempio ricerca e telemetria.

## 6. Template e ciclo di vita delle VM

1. Ogni template nasce nella VLAN LAB-BUILD, l'unica con aggiornamenti dall'esterno.
2. Il template viene aggiornato, ripulito, generalizzato con Sysprep per Windows e convertito in template di Proxmox.
3. Le VM bersaglio sono cloni collegati del template, nelle VLAN isolate. Il clone collegato risparmia disco.
4. Prima di ogni esercizio si fa uno snapshot "pulito". Dopo l'esercizio si torna allo snapshot.
5. Una VM bersaglio non torna mai nella VLAN LAB-BUILD. Per aggiornarla si ricostruisce il template.

Licenze: le ISO di valutazione di Windows Server durano 180 giorni, Windows 11
Enterprise 90 giorni. La ricostruzione periodica deve essere automatica.

Dominio vulnerabile: costruito da zero per conoscerlo bene, poi popolato con
strumenti come BadBlood o vulnerable-AD per avere utenti, gruppi e configurazioni
deboli realistiche. GOAD-Mini resta un'alternativa pronta.

## 7. Piattaforma di sicurezza

| Area | Strumento | Dove |
|---|---|---|
| Rete e SIEM | Security Onion: Suricata, Zeek, Strelka, Elastic | `nw1` |
| Log management e SIEM | Graylog Open 7.2 | `nw2` |
| Endpoint e XDR | Wazuh | `n100` |
| Telemetria Windows | Sysmon con configurazione di riferimento | Bersagli Windows |
| Forense live | Velociraptor | `n100` |
| Gestione incidenti | DFIR-IRIS | `n100` |
| Automazione | Shuffle | `n100` |
| Threat intelligence | MISP | `n100` |
| Emulazione avversari | Caldera su `n100`, Atomic Red Team sui bersagli | `n100`, `nw2` |
| Regole | Sigma, convertite per Security Onion, Graylog e Wazuh | Repository |

### 7.1 Ruolo di Graylog

Security Onion resta lo strumento principale per la rete. Graylog diventa il
punto centrale dei log, con queste sorgenti:
- **OPNsense del lab** via syslog. È il caso d'uso classico di Graylog e funziona da subito.
- **Eventi Windows e Sysmon** via Graylog Sidecar.
- **Allarmi di Wazuh** inoltrati da Wazuh.
- **Log Linux** dei bersagli via Sidecar o syslog.

Cosa offre Graylog Open, secondo la pagina del prodotto:
- ricerca, dashboard, stream e pipeline di elaborazione;
- allarmi su soglia e su eventi;
- import di regole Sigma da repository Git privati;
- licenza SSPL, gratuita, senza limiti di volume per uso interno.

Restano nelle edizioni a pagamento: rilevamento delle anomalie, investigazioni,
contenuti di rilevamento già pronti, punteggi di rischio. Il supporto Sigma
nell'edizione Open va verificato in pratica nella fase 5.

Lo stesso attacco visto in Security Onion e in Graylog permette un confronto
diretto tra lo stack Elastic e quello Graylog.

## 8. Fasi e criteri di completamento

### Fase 0: inventario
- Stato dei dischi e modello di CPU delle NodeWeaver.
- Piano di indirizzi verificato contro la produzione.
- Identificazione della NodeWeaver da 32 GB, che diventa `nw1`.
- Verifica delle regole di uscita del notebook sull'OPNsense di produzione, vedi 4.4.
- **Fatto quando:** questo documento non ha più segnaposto.

### Fase 1: infrastruttura
- Proxmox su tre nodi, cluster formato con due anelli corosync, storage ext4 con LVM-thin, KSM su `nw2`.
- OPNsense del lab sul N100 con WAN in passthrough e NAT in uscita, rotte statiche sul router.
- VLAN sullo switch, regole della sezione 4.3, secondo fattore sulle interfacce web, porta di emergenza.
- Prova del percorso asimmetrico dal notebook, vedi 4.1.
- Backup delle VM SOC.
- **Fatto quando:** da una VM di prova su LAB-TGT-WIN non si raggiunge né Internet né la rete del router. Da LAB-SOC si raggiunge solo l'allowlist. Dal notebook di produzione si apre una sessione SSH stabile verso `nw1` e l'interfaccia web di Proxmox.

### Fase 2: template e bersagli
- Template Windows Server Core, Windows 11, Ubuntu, Debian, Kali.
- Dominio `lab.local` con DC01 e WS01 uniti al dominio.
- **Fatto quando:** lo scenario Windows si accende e spegne dal pool e torna pulito da snapshot in pochi minuti.

### Fase 3: visibilità sugli endpoint
- Sysmon e agente Wazuh sui bersagli.
- **Fatto quando:** un'esecuzione di Atomic Red Team, per esempio un dump di credenziali, genera un allarme in Wazuh con la tecnica ATT&CK corretta.

### Fase 4: visibilità di rete
- Scheda 4x 1 GbE montata su `nw1`.
- Security Onion installato, mirror OVS su `nw2` e sul N100, cavi diretti attivi.
- Suricata sulla WAN dell'OPNsense del lab.
- **Fatto quando:** una scansione Nmap da Kali appare in Zeek e genera allarmi Suricata. Il traffico tra due VM sullo stesso nodo è visibile. Un aggiornamento di pacchetti da LAB-SOC appare in Zeek, e lo stesso flusso tra Kali e un bersaglio non compare due volte.

### Fase 5: log management con Graylog
- Graylog con Data Node e MongoDB.
- Ingresso dei log dell'OPNsense del lab, poi Sidecar sui bersagli, poi allarmi Wazuh.
- Pipeline per normalizzare i campi, dashboard per firewall ed eventi Windows.
- Test dell'import di regole Sigma.
- **Fatto quando:** l'allarme della fase 3 è visibile in Graylog insieme ai log del firewall relativi allo stesso momento.

### Fase 6: attacchi
- Catena completa da Kali: ricognizione, accesso iniziale, movimento laterale, compromissione del dominio.
- Le stesse tecniche replicate in modo ripetibile con Caldera.
- **Fatto quando:** esiste una tabella con ogni tecnica eseguita e dove è stata rilevata o persa, in Wazuh, Security Onion e Graylog.

### Fase 7: detection engineering
- Ciclo purple team: attacco, verifica, regola Sigma, nuovo attacco.
- Le stesse regole Sigma convertite per Security Onion e Graylog, con confronto dei risultati.
- **Fatto quando:** le tecniche perse in fase 6 hanno una regola testata nel repository, su almeno uno dei due SIEM.

### Fase 8: risposta agli incidenti
- Velociraptor per la raccolta forense, IRIS per i casi.
- Shuffle: allarme, arricchimento con MISP, apertura del caso in IRIS.
- **Fatto quando:** un allarme reale del lab apre da solo un caso arricchito.

### Fase 9: estensioni
- Scelte in base a ciò che è emerso nelle fasi precedenti.
- Candidati: OpenCanary per l'inganno interno, OpenCTI, NetFlow dall'OPNsense verso Graylog.

## 9. Automazione

Nuovo repository, per esempio `ansible-lab-security`, con le convenzioni dei
repository di produzione:
- un solo playbook con tag per area;
- inventario e variabili di gruppo per nodi Proxmox, servizi SOC e bersagli;
- vault cifrato con hook di pre-commit;
- documentazione e file di verifica.

Creazione delle VM: OpenTofu con il provider Proxmox `bpg/proxmox`, oppure i
moduli Proxmox di Ansible. Decisione da prendere in fase 1.
Configurazione dei bersagli Windows: Ansible via WinRM o SSH.
Configurazione dell'OPNsense del lab: da decidere tra la collezione Ansible
per OPNsense e un export versionato della configurazione.

## 10. Rischi

| Rischio | Mitigazione |
|---|---|
| Fuga di traffico malevolo verso Internet | Bersagli e attacco senza uscita, test in fase 1 |
| Host del lab che esplora la rete del router o la WAN della produzione | Regola esplicita sull'OPNsense del lab, vedi 4.3 |
| Router che scarta il traffico asimmetrico dal notebook | Rotta statica sull'OPNsense di produzione, oppure porta di emergenza |
| Lab raggiungibile da qualsiasi host della produzione, per via del NAT | Regola sulla sola WAN di produzione, YubiKey e secondo fattore |
| N100 spento: lab senza rete né quorum | Accettato per un lab |
| Agenti compromessi che attaccano il SOC | Solo le porte degli agenti verso LAB-SOC |
| RAM insufficiente su `nw1` | Ripiego su Security Onion Eval |
| RAM stretta su `nw2`, nodo da 24 GB | Alternanza degli scenari, Caldera su `n100` |
| Guasto di un disco senza repliche | Backup delle VM importanti sul disco di un altro nodo o su un disco esterno |
| N100 troppo lento | Servizi accesi per fase. `nw2` non ha più margine per accoglierli |
| Scadenza delle licenze Windows | Template ricostruiti in automatico |
| Interfaccia di gestione esposta sulla rete del router | LAB-MGMT accetta solo la WAN di produzione e la porta di emergenza |
