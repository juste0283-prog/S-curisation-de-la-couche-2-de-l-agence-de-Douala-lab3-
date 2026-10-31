TP — Sécurité des couches 1 et 2

Réalisé en binôme par : KEUMEKA SOKING WILFRIED GULLIT et NOUMEME WEMBOU ARNAULD

Ce dépôt présente notre travail sur la sécurisation du réseau de l’agence de Douala dans EVE-NG. L’objectif est de passer d’un réseau à plat à un réseau segmenté, puis de vérifier les protections contre les serveurs DHCP non autorisés, les usurpations d’adresses et le branchement d’équipements non autorisés.

 Environnement utilisé

| Équipement | Rôle |
|---|---|
| VMware Workstation | Hébergement de la machine virtuelle EVE-NG |
| EVE-NG | Simulation de la topologie |
| MikroTik CHR — R-DLA | Routage entre VLAN et serveur DHCP |
| Cisco IOSvL2 — SW-DLA | Commutation et protections de niveau 2 |
| Ubuntu Server | Serveur FreeRADIUS et client 802.1X |
| VPCS | Postes utilisateurs et équipements de test |

 Préparation de l’environnement

Nous avons rencontré un problème de virtualisation après l’installation de Docker Desktop. Le diagnostic a montré que l’hyperviseur Windows et VBS étaient actifs.

Nous avons créé une entrée de démarrage Windows dédiée aux TP VMware, avec le lancement de l’hyperviseur désactivé. Cette méthode permet de conserver l’entrée de démarrage habituelle pour Docker Desktop.

Après redémarrage sur l’entrée dédiée, nous avons exécuté :

```powershell
Get-CimInstance Win32_ComputerSystem |
    Select-Object HypervisorPresent
```

Le résultat obtenu était :

```text
HypervisorPresent
-----------------
False
```

 Segmentation du réseau

Le réseau du laboratoire est organisé selon le plan suivant :

| VLAN | Usage | Réseau | Passerelle |
|---|---|---|---|
| 10 | Guichets | 10.20.10.0/24 | 10.20.10.1 |
| 20 | Back-office | 10.20.20.0/24 | 10.20.20.1 |
| 40 | Caméras et IoT | 10.20.40.0/24 | 10.20.40.1 |
| 50 | Invités | 10.20.50.0/24 | 10.20.50.1 |
| 99 | Administration | 10.20.99.0/24 | 10.20.99.1 |
| 998 | VLAN natif inutilisé | — | — |
| 999 | Ports inutilisés | — | — |

La configuration consiste à créer les interfaces VLAN sur le routeur, à attribuer les adresses des passerelles et à préparer les services DHCP pour les VLAN 10, 20 et 50.

Le lien entre le routeur et le commutateur est configuré en trunk. Les ports des équipements sont placés en mode access dans leur VLAN respectif. La négociation DTP est désactivée et les ports inutilisés sont arrêtés dans le VLAN 999.

Les contrôles reposent notamment sur les commandes suivantes :

```text
show vlan brief
show interfaces trunk
show interfaces GigabitEthernet0/1 switchport
```

Les tests DHCP et les pings permettent de vérifier l’adressage et la connectivité. À cette étape, le routeur autorise encore les communications entre VLAN : leur filtrage doit être configuré séparément.

 Protections de niveau 2

L’activation des protections suit un ordre précis pour éviter de couper l’accès aux équipements légitimes :

1. Activer DHCP snooping.
2. Renouveler les baux et vérifier la table de liaisons DHCP.
3. Préparer les exceptions pour les équipements à adresse fixe.
4. Activer Dynamic ARP Inspection et IP Source Guard.
5. Configurer port-security.
6. Activer les protections Spanning Tree.

Chaque mécanisme est associé à un scénario de test.

| Incident simulé | Protection | Comportement attendu |
|---|---|---|
| Box distribuant des adresses DHCP non autorisées | DHCP snooping | Rejet des réponses DHCP sur un port non fiable |
| Poste se faisant passer pour la passerelle | Dynamic ARP Inspection | Rejet des annonces ARP non conformes |
| Poste utilisant une adresse IP non autorisée | IP Source Guard | Blocage du trafic ne correspondant pas à la liaison autorisée |
| Trois postes raccordés derrière un même port | Port-security | Limitation à deux adresses MAC |
| Commutateur émettant des BPDU sur un port d’accès | BPDU Guard | Passage du port en état `err-disabled` |

Les commandes de vérification sont :

```text
show ip dhcp snooping binding
show ip dhcp snooping statistics
show ip arp inspection statistics vlan 10
show ip verify source
show port-security interface GigabitEthernet1/3
show interfaces status err-disabled
show spanning-tree root
show logging
```

Pour valider chaque test, nous devons relever le comportement avant et après activation de la protection, ainsi que les compteurs et les messages des journaux.

 Authentification avec FreeRADIUS

La dernière partie concerne le contrôle d’accès au réseau avec 802.1X et MAB. FreeRADIUS vérifie les identifiants et transmet au commutateur les paramètres d’autorisation, notamment le VLAN à attribuer.

Les vérifications prévues sont les suivantes :

- **CLIENT-DLA** : authentification 802.1X avec attribution dynamique du VLAN 20.
- **CAMERA** : reconnaissance par adresse MAC avec MAB et affectation au VLAN 40.
- **SALLE-ATTENTE** : affectation au VLAN invités 50 en l’absence de réponse 802.1X.
- **Identifiants incorrects** : refus d’authentification et port non autorisé.

Sur le commutateur, les sessions sont vérifiées avec :

```text
show authentication sessions interface Gi0/2 details
show authentication sessions interface Gi0/3 details
show authentication sessions interface Gi2/0 details
```

Sur le serveur, le mode débogage permet d’observer les réponses `Access-Accept` et `Access-Reject` :

```bash
sudo freeradius -X
```

MAB reste moins robuste que 802.1X, car une adresse MAC peut être copiée.

 Résultats et preuves

Nous complétons ce tableau avec les résultats réellement observés pendant les manipulations.

| Vérification | Résultat observé | Preuve |
|---|---|---|
| Hyperviseur Windows arrêté sur le démarrage TP | `HypervisorPresent : False` | Capture PowerShell |
| VLAN et trunk | À compléter | À ajouter |
| Attribution DHCP et connectivité | À compléter | À ajouter |
| DHCP snooping | À compléter | À ajouter |
| Dynamic ARP Inspection | À compléter | À ajouter |
| IP Source Guard | À compléter | À ajouter |
| Port-security | À compléter | À ajouter |
| BPDU Guard | À compléter | À ajouter |
| Authentification 802.1X et MAB | À compléter | À ajouter |

 Sauvegarde des configurations

Les configurations sont à exporter avec les commandes suivantes :

```text
SW-DLA : show running-config
R-DLA  : /export
```

Les mots de passe et les secrets partagés doivent être retirés des fichiers avant leur publication dans le dépôt.

 Bilan

Ce travail nous permet de comprendre comment les VLAN, les protections de niveau 2 et l’authentification se complètent pour sécuriser un réseau d’agence. La démarche consiste à configurer progressivement, à simuler les incidents dans le laboratoire et à conserver les preuves des résultats obtenus.
