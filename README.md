# Projet Cloud Privé avec OpenNebula 7.2

## 1. Présentation du projet

L'objectif de ce projet est de mettre en place une infrastructure de cloud privé avec **OpenNebula 7.2**, permettant de gérer et de déployer des machines virtuelles à partir d'une interface Web.

L'infrastructure est composée de deux machines virtuelles Ubuntu Server :

- **one-front** : serveur Front-end OpenNebula.
- **one-node** : nœud de virtualisation (hyperviseur KVM).

Une troisième machine virtuelle sera créée dans OpenNebula pour tester le fonctionnement de l'infrastructure.

---

## 2. Architecture du laboratoire

| Machine | Rôle | Système | Adresse IP |
|---|---|---|---|
| one-front | Front-end OpenNebula | Ubuntu Server 24.04 | 192.168.137.135 |
| one-node | Hyperviseur KVM | Ubuntu Server 24.04 | 192.168.137.136 |
| vm-alpine-01 | Machine virtuelle cliente | Alpine Linux 3.24.2 | DHCP |

Réseau VMware utilisé : `192.168.137.0/24`

Passerelle VMware : `192.168.137.2`

---

## 3. Installation des composants

### 3.1. Sur le serveur Front-end (one-front)

Packages et composants à installer :

- `openssh-server` : permet les connexions SSH.
- `OpenNebula 7.2` : plateforme de gestion du cloud privé.
- `FireEdge` : interface Web permettant d'administrer OpenNebula.

OpenNebula est installé depuis son dépôt officiel.

Interface Web :

`http://192.168.137.135:2616`

Le service OpenNebula fonctionne sur le port `2633`.

### 3.2. Sur le nœud de virtualisation (one-node)

Packages à installer :

- `qemu-kvm` : moteur de virtualisation des machines virtuelles.
- `libvirt-daemon-system` : service de gestion des machines virtuelles.
- `libvirt-clients` : commandes pour administrer KVM.
- `bridge-utils` : outils pour gérer les ponts réseau.
- `virtinst` : outils pour créer des machines virtuelles.
- `qemu-system-x86` : émulateur système x86.
- `qemu-utils` : outils de gestion des disques virtuels.
- `opennebula-node-kvm` : intégration du nœud KVM avec OpenNebula.

---

## 4. Configuration de l'infrastructure OpenNebula

### 4.1. Configuration SSH

Une communication SSH sans mot de passe a été configurée entre le Front-end et le nœud de virtualisation avec l'utilisateur `oneadmin`.

Résultat :
- one-front → one-node : connexion SSH fonctionnelle.
- one-node → one-front : connexion SSH fonctionnelle.

Le fichier `/etc/hosts` du nœud a également été complété pour permettre la résolution du nom `one-front.lab`.

### 4.2. Ajout du nœud KVM

Depuis le Front-end, le nœud a été enregistré avec la commande :

```bash
sudo -u oneadmin onehost create 192.168.137.136 -i kvm -v kvm
```

Vérification :

```bash
sudo -u oneadmin onehost list
```

Résultat attendu : le nœud apparaît avec l'état `ON`.

### 4.3. Cluster

Le cluster par défaut a été utilisé.

Vérification :

```bash
sudo -u oneadmin onecluster show 0
```

Le cluster contient le nœud de virtualisation.

### 4.4. Datastores

Trois datastores sont disponibles :

| ID | Nom | Type |
|---|---|---|
| 0 | system | SYSTEM |
| 1 | default | IMAGE |
| 2 | files | FILES |

Ils sont associés au cluster `default`.

### 4.5. Configuration du réseau virtuel

Un réseau virtuel nommé `Lab-NAT` a été créé.

Configuration :

| Paramètre | Valeur |
|---|---|
| Nom | Lab-NAT |
| Type | IPv4 |
| VN_MAD | bridge |
| Bridge | virbr0 |
| Adresse réseau | 192.168.122.0 |
| Masque | 255.255.255.0 |
| Passerelle | 192.168.122.1 |
| DNS | 8.8.8.8 |
| Première adresse du pool | 192.168.122.10 |
| Taille du pool | 40 adresses |

Le réseau a été associé au cluster `default`.

Commande de vérification :

```bash
sudo -u oneadmin onevnet show 0
```

Sur le nœud, le réseau NAT par défaut de libvirt utilise également le bridge `virbr0`, avec une plage DHCP de `192.168.122.100` à `192.168.122.254`.

---

## 5. Création des images virtuelles

Une image ISO d'Alpine Linux 3.24.2 a été téléchargée et importée dans OpenNebula.

Image utilisée :

`alpine-virt-3.24.2-x86_64.iso`

Deux images sont disponibles dans le datastore `default` :

| ID | Nom | Type |
|---|---|---|
| 0 | alpine-virt-3.24.2 | CDROM |
| 1 | alpine-disk | DATABLOCK |

L'image `alpine-disk` possède une capacité de 2 Go et utilise le format QCOW2.

---

## 6. Création du modèle de machine virtuelle

Un modèle nommé `vm-alpine` a été créé dans OpenNebula.

Configuration :

| Paramètre | Valeur |
|---|---|
| Nom | vm-alpine |
| CPU | 1 |
| VCPU | 1 |
| RAM | 512 Mo |
| Hyperviseur | KVM |
| Disque système | alpine-disk |
| ISO | Alpine Linux 3.24.2 |
| Réseau | Lab-NAT |
| Graphiques | VNC |

Une interface réseau a été ajoutée au modèle afin de connecter les machines virtuelles au réseau `Lab-NAT`.

---

## 7. Déploiement de la première machine virtuelle

Une machine virtuelle nommée `vm-alpine-01` a été instanciée à partir du modèle `vm-alpine`.

Son identifiant OpenNebula est `0`.

Vérification :

```bash
sudo -u oneadmin onevm list
```

La VM a été déployée sur le nœud `192.168.137.136`.

### 7.1. Configuration de la carte réseau

Une interface réseau a été ajoutée à la VM arrêtée avec :

```bash
sudo -u oneadmin onevm nic-attach 0 --network Lab-NAT
```

Vérification :

```bash
sudo -u oneadmin onevm show 0
```

La VM possède maintenant une interface réseau connectée au bridge `virbr0`.

### 7.2. Installation d'Alpine Linux

La VM a été démarrée et son installation a été effectuée à partir de l'image ISO.

Configuration réalisée :

- Distribution : Alpine Linux 3.24.2.
- Nom de machine : `alpine-01`.
- Clavier : `fr-azerty`.
- Interface réseau : `eth0`.
- Configuration IPv4 : DHCP.
- Disque d'installation : `vda`.
- Mode d'installation : `sys`.
- Capacité du disque : environ 2 Go.

L'installation sur le disque virtuel s'est terminée avec succès.

---

## 8. Tests réseau effectués

Après avoir activé l'interface réseau `eth0`, le client DHCP a attribué une adresse IP à Alpine.

Configuration réseau obtenue :

| Paramètre | Valeur |
|---|---|
| Interface | eth0 |
| Adresse IP DHCP | 192.168.122.118 |
| Passerelle | 192.168.122.1 |
| Serveur DHCP | 192.168.122.1 |
| MAC | 02:00:c0:a8:7a:0a |

Tests effectués :

| Test | Résultat |
|---|---|
| Détection de la carte réseau | Réussi |
| Activation de eth0 | Réussi |
| Attribution IP par DHCP | Réussi |
| Ping vers 192.168.122.1 | 4 paquets reçus |
| Ping vers 8.8.8.8 | 4 paquets reçus |
| Ping vers google.com | 4 paquets reçus |

La connectivité réseau et la résolution DNS ont donc été vérifiées avec succès.

---

## 9. État actuel du projet

### Travaux réalisés

- [x] Installation du Front-end OpenNebula.
- [x] Installation de FireEdge.
- [x] Installation de KVM et libvirt sur le nœud.
- [x] Configuration SSH entre les deux machines.
- [x] Enregistrement du nœud dans OpenNebula.
- [x] Vérification du cluster.
- [x] Configuration des datastores.
- [x] Création du réseau virtuel Lab-NAT.
- [x] Importation de l'image ISO Alpine.
- [x] Création du disque virtuel de 2 Go.
- [x] Création du modèle vm-alpine.
- [x] Déploiement de vm-alpine-01.
- [x] Ajout de l'interface réseau à la VM.
- [x] Installation d'Alpine Linux sur le disque virtuel.
- [x] Vérification de la connectivité réseau et DNS.

### Travaux restant à réaliser

- [ ] Redémarrer Alpine sans utiliser l'image ISO.
- [ ] Vérifier le démarrage depuis le disque virtuel.
- [ ] Vérifier la configuration réseau après redémarrage.
- [ ] Finaliser la configuration de la VM.
- [ ] Réaliser les tests et captures d'écran pour la documentation du projet.

---

## 10. Conclusion provisoire

L'infrastructure OpenNebula 7.2 est opérationnelle. Le Front-end communique avec son nœud KVM, les ressources de stockage et le réseau virtuel sont configurés.

Une première machine virtuelle Alpine Linux a été créée, installée sur un disque virtuel et connectée au réseau NAT.

Les tests de connectivité IP et DNS ont été concluants.

La prochaine étape consiste à vérifier le démarrage autonome de la machine virtuelle depuis son disque virtuel, sans dépendre de l'image ISO.
