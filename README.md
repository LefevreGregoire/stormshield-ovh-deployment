# Déploiement d'une Infrastructure Sécurisée (SNS) sur Cloud Public

<p align="center">
  <a href="https://opensource.org/licenses/MIT"><img src="https://img.shields.io/badge/License-MIT-yellow.svg" alt="License: MIT"></a>
  <img src="https://img.shields.io/badge/Stormshield-SNS_v4-blue?style=flat-square&logo=shield" alt="Stormshield">
  <img src="https://img.shields.io/badge/OVHcloud-Public_Cloud-012169?style=flat-square&logo=ovh" alt="OVHcloud">
  <img src="https://img.shields.io/badge/Windows_Server-2025-0078D6?style=flat-square&logo=windows" alt="Windows Server">
  <img src="https://img.shields.io/badge/Network-Segmentation-red?style=flat-square" alt="Networking">
</p>

## À propos du projet

Ce repo documente la mise en place d'une architecture réseau sécurisée sur **OVH Public Cloud**. L'objectif est de déployer une appliance **Stormshield Network Security (SNS)** agissant comme passerelle de sécurité pour un réseau privé (LAN) isolé contenant des serveurs critiques.

Le projet détaille l'installation, la configuration des interfaces réseaux en ligne de commande (CLI) et l'intégration d'un client Windows Server.

## Architecture Réseau

Le réseau est segmenté en deux zones distinctes :

* **WAN (Zone Publique)** : Connectée à Internet via le réseau public OVH.
* **LAN (Zone Privée)** : Réseau isolé `10.0.0.0/8`, non accessible depuis l'extérieur sans passer par le Firewall.

| Rôle | OS | Interface WAN | Interface LAN |
| :--- | :--- | :--- | :--- |
| **Firewall / Gateway** | Stormshield SNS (EVA) | DHCP (Public) | `10.0.0.254` |
| **Serveur App** | Windows Server 2025 | *Aucune* | `10.0.0.8` |

## Fonctionnalités implémentées

* [x] Création du vSwitch privé sur OpenStack (OVH).
* [x] Déploiement de l'instance Stormshield via image Cloud.
* [x] Configuration bas niveau (Shell FreeBSD) des interfaces réseaux.
* [x] Configuration statique du client Windows Server.
* [x] Validation des flux ICMP et accès WebAdmin sécurisé.

## Schéma 

```mermaid
flowchart TD
    %% --- Styles ---
    classDef netPublic fill:#e1f5fe,stroke:#0288d1,stroke-width:2px,color:#000
    classDef netPrivate fill:#fff3e0,stroke:#e65100,stroke-width:2px,stroke-dasharray: 5 5,color:#000
    classDef fwNode fill:#ffebee,stroke:#c62828,stroke-width:2px,color:#000
    classDef srvNode fill:#f3e5f5,stroke:#6a1b9a,stroke-width:2px,color:#000

    %% --- Élément Externe ---
    Internet(("Internet")):::netPublic
    
    %% --- Zone OVH ---
    subgraph OVH ["Infrastructure OVH Public Cloud"]
        SNS["Firewall Stormshield SNS<br>(Passerelle de sécurité)"]:::fwNode
        
        subgraph LAN ["Réseau Privé Isolé (vSwitch : 10.0.0.0/8)"]
            WinSrv["Windows Server 2025<br>IP: 10.0.0.8"]:::srvNode
        end
    end

    %% --- Connexions (Placées à la fin pour un calcul optimal de l'espace) ---
    Internet ==>|"WAN (IP Publique DHCP)"| SNS
    SNS -.->|"LAN (Gateway : 10.0.0.254)"| WinSrv
```

## Documentation

[Guide de déploiement complet](./Guide_deploiement.md)
