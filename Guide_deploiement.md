# Guide de Déploiement : Stormshield SNS & Windows LAN

Ce guide détaille toutes les étapes techniques, de la configuration de l'infrastructure Cloud jusqu'au premier ping réussi.

## Prérequis
* Un projet **OVH Public Cloud** actif.
* Accès à l'interface Horizon (OpenStack) ou au Manager OVH.
* Une paire de clés SSH (facultatif pour SNS, requis pour Linux).

---

## Phase 1 : Préparation de l'Infrastructure (OVH)

Avant de lancer les machines, nous devons créer un réseau privé.

### 1.1 Création du Réseau Privé (LAN)
1.  Dans l'interface OVH, aller dans **Network > Private Networks**.
2.  Créer un nouveau réseau nommé `SNS-LAN`.
3.  **Important :** Ne pas activer le DHCP sur ce réseau (nous utiliserons des IP statiques pour la sécurité).
4.  Plage d'adressage (subnet) : `10.0.0.0/8` (Optionnel si pas de DHCP, mais recommandé pour la logique).

### 1.2 Déploiement du Stormshield (SNS)
1.  Aller dans **Compute > Instances > Create Instance**.
2.  **Modèle :** Choisir l'image `Stormshield Network Security` (Licence BYOL ou PAYG).
3.  **Configuration Réseau (Crucial)** :
    * Interface 1 : `Ext-Net` (Public Internet) -> Deviendra l'interface WAN.
    * Interface 2 : `SNS-LAN` (Privé) -> Deviendra l'interface LAN.
4.  Lancer l'instance et noter son nom (ex: `SNS-Gateway`).

### 1.3 Déploiement du Windows Server
1.  Créer une nouvelle instance.
2.  **Image :** Windows Server 2025 (ou 2019/2022).
3.  **Configuration Réseau** :
    * Sélectionner **uniquement** le réseau privé `SNS-LAN`.
    * *Note : Cette machine n'aura pas d'accès Internet direct pour l'instant.*

---

## Phase 2 : Configuration du Stormshield (Shell)

Le Stormshield démarre par défaut en DHCP sur le WAN, mais l'interface LAN est souvent mal configurée ou sur une plage différente (`192.168...`). Nous allons forcer la configuration via la console.

### 2.1 Accès Console
1.  Ouvrir la console VNC (NoVNC) de l'instance `SNS-Gateway` depuis l'interface OVH.
2.  S'identifier :
    * Login : `admin`
    * Password : `admin` (ou le mot de passe défini à l'initialisation).

### 2.2 Modification du Fichier Réseau
Les interfaces physiques virtuelles sont nommées `vtnet0` (WAN) et `vtnet1` (LAN).

1.  Vérifier l'état actuel :

    ```bash
    ifconfig
    ```
3.  Éditer (ou écraser) le fichier de configuration réseau pour définir notre plan d'adressage `10.0.0.x`.
    *Tapez la commande suivante pour écrire la configuration propre :*

    ```bash
    echo "[Ethernet0]" > /usr/Firewall/ConfigFiles/network
    echo "State=1" >> /usr/Firewall/ConfigFiles/network
    echo "Name=Wan" >> /usr/Firewall/ConfigFiles/network
    echo "Method=DHCP" >> /usr/Firewall/ConfigFiles/network
    echo "" >> /usr/Firewall/ConfigFiles/network
    echo "[Ethernet1]" >> /usr/Firewall/ConfigFiles/network
    echo "State=1" >> /usr/Firewall/ConfigFiles/network
    echo "Protected=1" >> /usr/Firewall/ConfigFiles/network
    echo "Name=Lan" >> /usr/Firewall/ConfigFiles/network
    echo "Address=10.0.0.254" >> /usr/Firewall/ConfigFiles/network
    echo "Mask=255.0.0.0" >> /usr/Firewall/ConfigFiles/network
    ```

4.  Appliquer les changements par un redémarrage :

    ```bash
    reboot
    ```

---

## Phase 3 : Configuration du Client Windows

Le serveur Windows est isolé. Nous devons lui attribuer une IP fixe pour qu'il puisse discuter avec le Stormshield.

### 3.1 Configuration IP Statique
1.  Accéder à la console VNC de l'instance Windows.
2.  Ouvrir le **Panneau de Configuration > Centre Réseau et Partage > Modifier les paramètres de la carte**.
3.  Clic droit sur la carte Ethernet > **Propriétés** > **IPv4**.
4.  Remplir les champs suivants :
    * **Adresse IP** : `10.0.0.8`
    * **Masque de sous-réseau** : `255.0.0.0`
    * **Passerelle par défaut** : `10.0.0.254` (L'IP du Stormshield configurée précédemment).
    * **DNS** : `8.8.8.8` (Google) ou `213.186.33.99` (OVH).

### 3.2 Autorisation des flux (Pare-feu Windows)
Par défaut, Windows bloque les réponses au Ping (ICMP) sur les réseaux qu'il ne connaît pas. Pour tester, nous allons autoriser le trafic.

1.  Ouvrir une invite de commande (cmd) en Administrateur.
2.  Désactiver le pare-feu pour le profil courant (ou créer une règle ICMP) :

    ```DOS
    netsh advfirewall set allprofiles state off
    ```
    *(Note : En production, privilégiez la création d'une règle spécifique d'autorisation).*

---

## Phase 4 : Validation et Tests

### 4.1 Test de Connectivité (Ping)
Depuis le Windows Server, testons la communication avec la passerelle Stormshield.

Ouvrir `cmd.exe` et taper :
```cmd
ping 10.0.0.254
```

> **Résultat attendu :**
> `Reply from 10.0.0.254: bytes=32 time<1ms TTL=64`

### 4.2 Accès à l'Administration (WebAdmin)

Depuis le navigateur du Windows Server (Edge/Chrome) :

1. Aller sur : `https://10.0.0.254/admin`
2. Accepter l'alerte de certificat de sécurité (Self-signed).
3. La page de login du Stormshield doit s'afficher.

---

## 5 Dépannage (Troubleshooting)

| Symptôme | Cause probable | Solution |
| :--- | :--- | :--- |
| **Ping échoue (Timeout)** | Windows Firewall actif | Désactiver le pare-feu Windows ou autoriser ICMPv4. |
| **Ping échoue (Unreachable)** | Mauvaise IP / Masque | Vérifier que Windows est bien en `/8` (255.0.0.0). |
| **Erreur 501 sur le Web** | Service SNS planté | Le fichier `/usr/Firewall/ConfigFiles/network` contient une erreur de syntaxe. |
| **Pas d'Internet sur Windows** | Règles de filtrage | Le Stormshield bloque le trafic sortant par défaut. Il faut créer une règle `Pass All` dans la politique de sécurité. |
