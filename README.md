# monitoring-graylog
Gestion et analyse centralisées des journaux avec Graylog et OpenSearch. Collecte, surveillance, dépannage et optimisation du volume des journaux Syslog.

# Centralisation et supervision des logs avec Graylog

![Graylog](https://img.shields.io/badge/Graylog-SIEM%20%2F%20Log%20Management-FF3633)
![OpenSearch](https://img.shields.io/badge/OpenSearch-Search%20Engine-005EB8)
![Linux](https://img.shields.io/badge/Linux-Server-FCC624)
![Fortinet](https://img.shields.io/badge/Fortinet-Firewall-EE3124)
![Syslog](https://img.shields.io/badge/Syslog-UDP%20%2F%20TCP-blue)

## Présentation

Ce projet présente la mise en place et l'exploitation d'une infrastructure **Graylog** permettant de centraliser, analyser et superviser les journaux provenant de plusieurs équipements d'une infrastructure IT.

L'objectif principal était de disposer d'un point central permettant de :

* collecter les logs des équipements réseau ;
* centraliser les événements des firewalls ;
* intégrer progressivement les logs systèmes ;
* rechercher rapidement des événements ;
* faciliter les investigations lors d'incidents ;
* surveiller le volume de logs généré ;
* réduire les événements inutiles ;
* optimiser la consommation des ressources ;
* préparer une utilisation orientée supervision et sécurité.

> Ce dépôt correspond à une version anonymisée d'un projet réalisé dans un environnement professionnel. Toutes les informations permettant d'identifier l'organisation ou son infrastructure ont été supprimées ou remplacées.

---

# Architecture

L'architecture repose principalement sur les composants suivants :

```text
                    +----------------------+
                    |      Firewall        |
                    |       Fortinet       |
                    +----------+-----------+
                               |
                               | Syslog
                               | UDP / TCP
                               v
                    +----------------------+
                    |       Graylog        |
                    |                      |
                    | Inputs / Pipelines   |
                    | Streams / Searches   |
                    +----------+-----------+
                               |
                               v
                    +----------------------+
                    |      OpenSearch      |
                    |                      |
                    | Indexation / Search  |
                    +----------------------+
```

D'autres sources peuvent ensuite être intégrées :

```text
                +-------------------+
                | Windows Servers   |
                +---------+---------+
                          |
                +---------v---------+
                |   Linux Servers   |
                +---------+---------+
                          |
                +---------v---------+
                | Network Devices   |
                +---------+---------+
                          |
                          v
                    +-----------+
                    |  Graylog  |
                    +-----------+
```

---

# Stack technique

| Technologie | Utilisation                               |
| ----------- | ----------------------------------------- |
| Graylog     | Centralisation et analyse des logs        |
| OpenSearch  | Stockage et indexation                    |
| Linux       | Hébergement de la plateforme              |
| Syslog      | Transmission des événements               |
| Fortinet    | Source principale de logs réseau/sécurité |
| UDP         | Collecte Syslog rapide                    |
| TCP         | Collecte Syslog fiable                    |
| journald    | Analyse des logs système Linux            |
| curl        | Tests et interrogation OpenSearch         |
| Bash        | Diagnostic et mesures                     |

---

# Configuration Graylog

## Inputs

Deux principaux inputs Syslog ont été configurés.

### Syslog UDP

```text
Protocol : UDP
Port     : 1514
Status   : RUNNING
```

Utilisé principalement pour les équipements réseau générant un volume important de logs.

### Syslog TCP

```text
Protocol : TCP
Port     : 1515
Status   : RUNNING
```

TCP peut être utilisé lorsque l'on souhaite privilégier la fiabilité de livraison des événements.

---

# Exemple de configuration Syslog côté firewall

Exemple volontairement générique :

```text
config log syslogd setting

    set status enable
    set server <GRAYLOG_IP>
    set mode udp
    set port 1514
    set facility local7

end
```

Dans un environnement réel :

```text
Firewall
   |
   | Syslog UDP :1514
   |
   v
Graylog
```

Le serveur Graylog reçoit ensuite les messages et les transmet vers les mécanismes d'indexation.

---

# OpenSearch

Graylog utilise OpenSearch pour stocker et rechercher les événements.

Vérification de l'état du cluster :

```bash
curl -s http://127.0.0.1:9200/_cluster/health?pretty
```

Exemple de résultat attendu :

```json
{
  "cluster_name": "graylog",
  "status": "green",
  "number_of_nodes": 1
}
```

Un état :

```text
green
```

indique que les shards nécessaires sont correctement disponibles.

---

# Vérification des index

Lister les index :

```bash
curl -s "http://127.0.0.1:9200/_cat/indices?v"
```

Exemple :

```text
health status index               docs.count
green  open   graylog_0           ...
green  open   gl-events_0         ...
green  open   gl-system-events_0  ...
```

Les principaux index observés sont notamment :

```text
graylog_x
gl-events_x
gl-system-events_x
```

---

# Analyse du volume de logs

L'un des points importants du projet a été l'analyse du volume généré par les équipements.

Un firewall peut produire plusieurs centaines d'événements par minute, voire beaucoup plus selon :

* la politique de journalisation ;
* le nombre d'utilisateurs ;
* le trafic réseau ;
* les règles de sécurité ;
* les connexions autorisées ;
* les connexions refusées ;
* les DNS ;
* les sessions ;
* les événements applicatifs.

Il était donc nécessaire de mesurer précisément le débit avant de mettre en place des filtres.

---

## Mesurer le nombre de logs par minute

Exemple de script Bash utilisé pour calculer la croissance d'un index sur 60 secondes :

```bash
A=$(curl -s \
"http://127.0.0.1:9200/_cat/indices/graylog_1?h=docs.count" \
| tr -d ' ')

echo "Départ : $A"

sleep 60

B=$(curl -s \
"http://127.0.0.1:9200/_cat/indices/graylog_1?h=docs.count" \
| tr -d ' ')

echo "Après 60 secondes : $B"

echo "Logs par minute : $((B-A))"
```

Lors d'une mesure effectuée sur l'environnement :

```text
Logs par minute : ~580
```

Cette valeur dépend bien entendu fortement du trafic observé.

---

# Pourquoi mesurer le volume ?

Avant de filtrer des événements, il est important de connaître leur impact.

Par exemple :

```text
580 logs/minute
```

représente environ :

```text
34 800 logs/heure
835 200 logs/jour
25 000 000 logs/mois
```

Cela montre rapidement pourquoi une mauvaise stratégie de journalisation peut générer :

* une croissance importante des index ;
* une consommation disque élevée ;
* davantage de CPU ;
* davantage de RAM ;
* des recherches plus longues ;
* une rétention plus faible.

---

# Diagnostic des ressources

Lorsqu'un volume important de logs était observé, plusieurs vérifications Linux ont été réalisées.

## CPU

```bash
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%cpu | head -10
```

Cette commande permet d'identifier rapidement les processus consommant le plus de CPU.

---

## Mémoire

```bash
free -h
```

Exemple :

```text
               total        used        free
Mem:             ...         ...         ...
Swap:            ...         ...         ...
```

---

## Charge système

```bash
uptime
```

ou :

```bash
cat /proc/loadavg
```

---

## Disque

```bash
df -h
```

Permet notamment de surveiller les partitions contenant les données OpenSearch.

---

# OpenSearch et vm.max_map_count

OpenSearch nécessite un nombre suffisamment élevé de zones mémoire mappées.

Vérification :

```bash
sysctl vm.max_map_count
```

Configuration utilisée :

```text
vm.max_map_count = 262144
```

Modification temporaire :

```bash
sudo sysctl -w vm.max_map_count=262144
```

Configuration persistante :

```bash
echo "vm.max_map_count=262144" \
| sudo tee -a /etc/sysctl.conf
```

Puis :

```bash
sudo sysctl -p
```

---

# Journaux Linux

Pour analyser les journaux du serveur :

```bash
journalctl
```

Afficher les derniers événements :

```bash
journalctl -n 100
```

Suivre les événements en temps réel :

```bash
journalctl -f
```

Pour un service spécifique :

```bash
journalctl -u graylog-server
```

---

# Recherche Graylog

Une fois les événements centralisés, Graylog permet d'effectuer des recherches très rapidement.

## Recherche globale

```text
*
```

---

## Recherche par adresse IP

```text
src_ip:192.0.2.10
```

---

## Recherche par destination

```text
dst_ip:198.51.100.20
```

---

## Recherche sur une action firewall

```text
action:deny
```

ou :

```text
action:accept
```

---

## Combiner plusieurs critères

```text
src_ip:192.0.2.10 AND action:deny
```

---

## Recherche sur un port

```text
dst_port:443
```

---

## Recherche HTTP/HTTPS

```text
dst_port:443 OR dst_port:80
```

---

# Utilisation dans le cadre d'un incident

Graylog permet par exemple de partir d'une adresse IP et de reconstruire une partie de son activité.

```text
Utilisateur
    |
    v
Firewall
    |
    +---- connexion HTTPS
    |
    +---- connexion refusée
    |
    +---- requête DNS
    |
    +---- session VPN
    |
    v
Graylog
```

Une recherche ciblée permet ensuite de retrouver les événements associés.

Exemple :

```text
src_ip:<IP_UTILISATEUR>
```

Puis filtrage supplémentaire :

```text
src_ip:<IP_UTILISATEUR> AND dst_port:443
```

---

# Problématique rencontrée : trop de logs

L'un des principaux problèmes rencontrés était la quantité d'événements envoyés par le firewall.

Dans certains cas, Graylog recevait plusieurs centaines de milliers d'événements sur une période très courte.

Le problème n'était pas nécessairement Graylog lui-même.

Il était surtout lié à la quantité de données envoyées par les sources.

---

# Stratégie adoptée

Avant de supprimer des événements, la démarche suivante a été appliquée :

```text
1. Mesurer le volume
        |
        v
2. Identifier les plus gros producteurs
        |
        v
3. Identifier les événements répétitifs
        |
        v
4. Vérifier leur utilité
        |
        v
5. Définir les filtres
        |
        v
6. Mesurer à nouveau
```

Cette approche évite de supprimer accidentellement des événements utiles pour une investigation de sécurité.

---

# Types de logs à analyser

Dans un environnement firewall, certains événements peuvent représenter une part importante du volume :

```text
traffic
dns
system
event
utm
webfilter
vpn
application
```

Le filtrage ne doit donc pas simplement être effectué sur la quantité.

Il faut déterminer :

```text
Valeur sécurité
      VS
Coût stockage
```

---

# Exemple de logique de filtrage

Certains événements peuvent être considérés comme peu utiles selon le contexte.

Exemple conceptuel :

```text
Tous les logs
    |
    +---- Événements critiques ----------> conserver
    |
    +---- Connexions bloquées -----------> conserver
    |
    +---- Événements VPN -----------------> conserver
    |
    +---- Alertes sécurité ---------------> conserver
    |
    +---- Trafic répétitif interne -------> analyser
    |
    +---- Événements sans valeur ---------> filtrer
```

Le filtrage doit toujours être adapté aux besoins du SI.

---

# Pipelines Graylog

Graylog permet d'utiliser des pipelines afin de traiter les événements avant leur indexation.

Architecture :

```text
Input
  |
  v
Pipeline
  |
  +---- parsing
  |
  +---- normalisation
  |
  +---- enrichissement
  |
  +---- filtrage
  |
  v
Stream
  |
  v
Index
```

---

# Streams

Les Streams permettent de séparer les événements.

Exemple d'organisation :

```text
All Messages
     |
     +---- Firewall Logs
     |
     +---- Windows Logs
     |
     +---- Linux Logs
     |
     +---- Network Logs
     |
     +---- Security Alerts
```

Cela permet d'obtenir des recherches plus lisibles et une meilleure organisation.

---

# Normalisation

Une autre amélioration possible consiste à normaliser les champs.

Par exemple :

```text
source_ip
destination_ip
source_port
destination_port
username
action
protocol
device
severity
```

Cela simplifie ensuite les recherches.

Exemple :

```text
source_ip:192.0.2.10 AND action:deny
```

---

# Rotation des index

La rotation des index est essentielle pour éviter une croissance infinie des données.

Cycle logique :

```text
graylog_0
   |
   v
graylog_1
   |
   v
graylog_2
   |
   v
graylog_3
```

Lorsqu'un index atteint la limite définie, Graylog en crée un nouveau.

---

# Rétention

La stratégie de rétention dépend de plusieurs critères :

* espace disque ;
* volume journalier ;
* exigences de sécurité ;
* besoins d'investigation ;
* contraintes réglementaires ;
* capacité de l'infrastructure.

Il est important de déterminer :

```text
Logs/jour
      ×
Taille moyenne/log
      ×
Durée de conservation
```

afin d'estimer le stockage nécessaire.

---

# Supervision de Graylog

Les éléments importants à surveiller sont :

```text
CPU
RAM
Load Average
Disk Usage
OpenSearch Cluster Health
Index Size
Message Rate
Journal Size
Input Status
```

---

# Vérifier les ports

Exemple :

```bash
ss -lntup
```

On doit notamment retrouver les services associés à :

```text
9000/tcp   Graylog Web
1514/udp   Syslog UDP
1515/tcp   Syslog TCP
```

Les ports peuvent évidemment être différents selon l'environnement.

---

# Vérifier l'interface Graylog

L'interface Web est généralement exposée via :

```text
http://<GRAYLOG_SERVER>:9000
```

Dans un environnement de production, il est préférable de placer Graylog derrière :

```text
Reverse Proxy
      +
HTTPS
```

---

# Sécurité

Plusieurs bonnes pratiques doivent être appliquées.

## Ne pas exposer directement Graylog sur Internet

L'accès à l'interface doit être limité à un réseau d'administration ou à un VPN.

---

## Utiliser HTTPS

L'interface Web devrait être protégée avec TLS.

---

## Restreindre les ports Syslog

Seules les sources autorisées devraient pouvoir envoyer des événements.

---

## Séparer les comptes

Éviter l'utilisation permanente du compte administrateur.

---

## Ne jamais versionner les secrets

Ne jamais placer dans GitHub :

```text
passwords
tokens
API keys
private keys
internal IP addresses
production domains
usernames
customer data
```

Exemple :

```bash
GRAYLOG_PASSWORD=<SECRET>
```

et non :

```bash
GRAYLOG_PASSWORD=MyRealPassword123
```

---

# Anonymisation du projet

Comme ce projet provient d'un environnement professionnel, plusieurs éléments ont volontairement été modifiés.

Les adresses utilisées dans les exemples appartiennent notamment aux plages réservées à la documentation :

```text
192.0.2.0/24
198.51.100.0/24
203.0.113.0/24
```

Aucune adresse IP de production n'est présente dans ce dépôt.

Les éléments suivants sont également anonymisés :

```text
noms des serveurs
noms des utilisateurs
domaines
VLAN
architecture réseau détaillée
identifiants
informations clients
règles firewall
secrets
```

---

# Méthodologie du projet

Le projet a été réalisé progressivement.

## Phase 1 — Audit

```text
Analyse infrastructure
       |
       v
Identification des sources
       |
       v
Analyse Graylog
       |
       v
Analyse OpenSearch
```

---

## Phase 2 — Collecte

```text
Firewall
   |
   v
Syslog
   |
   v
Graylog Input
```

---

## Phase 3 — Vérification

Vérification :

* réception des logs ;
* parsing ;
* indexation ;
* état OpenSearch ;
* consommation CPU ;
* consommation RAM ;
* stockage.

---

## Phase 4 — Analyse du volume

Calcul :

```text
messages/minute
messages/heure
messages/jour
messages/mois
```

---

## Phase 5 — Optimisation

Identification :

```text
logs utiles
logs redondants
logs volumineux
logs sécurité
logs système
```

Puis mise en place progressive de règles de filtrage.

---

# Difficultés rencontrées

## Volume important

Le firewall générait une quantité très importante d'événements.

### Solution

Mesurer précisément la croissance des index avant toute modification.

---

## Croissance rapide du stockage

OpenSearch pouvait accumuler plusieurs centaines de milliers de documents rapidement.

### Solution

Analyse :

```text
Index size
Document count
Message rate
Retention
```

---

## Risque de filtrage excessif

Supprimer trop de logs aurait pu empêcher certaines investigations.

### Solution

Mettre en place les filtres progressivement et comparer les volumes avant/après.

---

# Compétences mises en œuvre

Ce projet m'a permis de travailler sur plusieurs domaines.

### Linux

```text
systemd
journald
network
processes
memory
storage
shell
```

### Log Management

```text
Graylog
Syslog
Streams
Inputs
Pipelines
Search
Indexes
```

### OpenSearch

```text
Cluster health
Indexes
Documents
Storage
REST API
```

### Réseau

```text
TCP/IP
UDP
Syslog
Firewall
Ports
VLAN
```

### Sécurité

```text
Log analysis
Incident investigation
Firewall events
Access control
Centralized logging
```

### Troubleshooting

```text
CPU
RAM
Disk
Processes
Network
Application logs
```

---

# Améliorations futures

Plusieurs évolutions peuvent être ajoutées au projet.

* [ ] Intégration complète des événements Windows
* [ ] Intégration des serveurs Linux
* [ ] Création de pipelines avancés
* [ ] Normalisation des champs
* [ ] Création de Streams par équipement
* [ ] Dashboards sécurité
* [ ] Dashboards réseau
* [ ] Alertes sur événements critiques
* [ ] Détection des erreurs d'authentification
* [ ] Détection des scans réseau
* [ ] Détection des connexions VPN suspectes
* [ ] Gestion automatisée de la rétention
* [ ] Reverse Proxy HTTPS
* [ ] Supervision de Graylog
* [ ] Export de métriques
* [ ] Documentation automatisée

---

# Exemple de dashboard cible

```text
+----------------------------------------------------+
|                SECURITY DASHBOARD                  |
+----------------------------------------------------+
| Messages / sec | Firewall events | Failed logins  |
+----------------------------------------------------+
|                                                    |
|             Messages over time                     |
|                                                    |
+-------------------------+--------------------------+
| Top Source IP           | Top Destination IP       |
+-------------------------+--------------------------+
| Denied Connections      | Security Events          |
+-------------------------+--------------------------+
```

---

# Arborescence possible du dépôt

```text
graylog-project/
│
├── README.md
│
├── docs/
│   ├── architecture.md
│   ├── installation.md
│   ├── troubleshooting.md
│   └── security.md
│
├── scripts/
│   ├── check-log-rate.sh
│   ├── check-opensearch.sh
│   └── health-check.sh
│
├── examples/
│   ├── syslog.conf.example
│   ├── pipeline-rule.example
│   └── search-queries.md
│
└── diagrams/
    └── architecture.png
```

---

# Scripts utiles

## `check-log-rate.sh`

```bash
#!/bin/bash

INDEX="graylog_1"

A=$(curl -s \
"http://127.0.0.1:9200/_cat/indices/${INDEX}?h=docs.count" \
| tr -d ' ')

echo "Documents au départ : $A"

sleep 60

B=$(curl -s \
"http://127.0.0.1:9200/_cat/indices/${INDEX}?h=docs.count" \
| tr -d ' ')

echo "Documents après 60 secondes : $B"

RATE=$((B-A))

echo "Messages/minute : $RATE"
echo "Messages/heure   : $((RATE*60))"
echo "Messages/jour    : $((RATE*60*24))"
```

---

## `check-opensearch.sh`

```bash
#!/bin/bash

echo "=== OpenSearch Cluster Health ==="

curl -s \
"http://127.0.0.1:9200/_cluster/health?pretty"

echo
echo "=== OpenSearch Indexes ==="

curl -s \
"http://127.0.0.1:9200/_cat/indices?v"
```

---

## `health-check.sh`

```bash
#!/bin/bash

echo "================================"
echo " Graylog Server Health Check"
echo "================================"

echo
echo "[CPU]"
uptime

echo
echo "[MEMORY]"
free -h

echo
echo "[DISK]"
df -h

echo
echo "[TOP PROCESSES]"
ps -eo pid,ppid,cmd,%mem,%cpu \
--sort=-%cpu | head -10

echo
echo "[OPEN PORTS]"
ss -lntup

echo
echo "[OPENSearch]"
curl -s \
"http://127.0.0.1:9200/_cluster/health?pretty"
```

---

# Ce que ce projet m'a appris

La mise en place d'une solution de centralisation des logs ne consiste pas uniquement à installer Graylog.

Il faut également comprendre toute la chaîne :

```text
Source
  ↓
Transport
  ↓
Collecte
  ↓
Parsing
  ↓
Filtrage
  ↓
Indexation
  ↓
Stockage
  ↓
Recherche
  ↓
Analyse
  ↓
Alerte
```

L'un des enseignements principaux du projet a été l'importance de maîtriser le volume de données.

Collecter tous les événements sans stratégie peut rapidement augmenter :

```text
CPU
RAM
Stockage
Coût
Temps de recherche
```

Une bonne architecture de logs doit donc trouver un équilibre entre :

```text
VISIBILITÉ
    +
SÉCURITÉ
    +
PERFORMANCE
    +
RÉTENTION
```

---

# Objectif personnel

Ce projet fait partie de mon portfolio autour de :

```text
Infrastructure
Linux
Network
Cybersecurity
Automation
Monitoring
Observability
Platform Engineering
```

L'objectif de ce dépôt est de documenter une mise en situation réelle tout en respectant strictement la confidentialité de l'environnement professionnel dans lequel le projet a été réalisé.

---

## Disclaimer

Ce dépôt est fourni uniquement à des fins de démonstration technique et de portfolio.

Toutes les données présentées sont :

* fictives ;
* anonymisées ;
* génériques ;
* ou issues de plages réservées à la documentation.

Aucune information confidentielle appartenant à une entreprise, un client ou un utilisateur n'est publiée.

---

## Auteur

**Marouf Medhy**

Infrastructure • Systems • Network • Cybersecurity • Platform Engineering
