# OpenTiller — Contexte du Projet

> **Alias** : VIAS-PILOT
> **Statut** : En développement — prototype DIY
> **Licence** : À définir (open source)

---

## Qu'est-ce qu'OpenTiller ?

OpenTiller est un **pilote automatique de voilier DIY** à coût minimal, conçu pour être construit à partir de matériel recyclé et de composants courants.

L'objectif est de tenir un cap magnétique de manière autonome en agissant sur le moteur de barre, en recevant des consignes de navigation depuis un ordinateur de bord.

---

## Inspiration et philosophie

OpenTiller est librement inspiré de **PyPilot** de Sean D'Epagnier, dont nous saluons le travail pionnier. Cependant, PyPilot a progressivement dérivé vers une approche de plus en plus commerciale et propriétaire, s'éloignant de l'esprit DIY originel qui l'avait fait connaître.

OpenTiller renoue avec cet esprit :

- **Matériel récupéré** : cartes contrôleur de vol de drone hors service, recyclées pour un nouvel usage
- **Coût quasi nul** en partant du matériel existant
- **Architecture ouverte** : chaque contributeur peut implémenter le protocole sur sa propre carte
- **Documentation exhaustive** en français et en anglais
- **Simplicité avant tout** : chaque couche du système doit rester compréhensible et maintenable

---

## Architecture du système

```
┌─────────────────────────────────────────────────────────────┐
│                    Raspberry Pi Zero                        │
│                                                             │
│  • Navigation haute-niveau                                  │
│  • Calcul de cap cible (waypoints, cap magnétique)          │
│  • Lecture GPS (COG/SOG/position)                           │
│  • Interface utilisateur (web ou console)                   │
│  • Pas de contrainte temps réel                             │
└────────────────────────┬────────────────────────────────────┘
                         │  UART série (PROTOCOL.md)
                         │  Consigne de cap + commandes
                         ▼
┌─────────────────────────────────────────────────────────────┐
│              Carte contrôleur (Skyline32 ou CC3D)           │
│                                                             │
│  • Contrôle temps réel                                      │
│  • Lecture compas (HMC5883L)                                │
│  • Boucle PID : cap mesuré → cap cible → commande servo     │
│  • Pilotage moteur de barre (PWM1 → servo)                  │
│  • Maintien du dernier cap si liaison Pi perdue             │
└─────────────────────────────────────────────────────────────┘
```

**Principe de sécurité fondamental** : si le Pi Zero plante ou perd la liaison, la carte contrôleur continue à tenir le dernier cap reçu. Le bateau ne se met jamais en travers.

---

## Matériel cible

| Composant | Modèle retenu | Rôle |
|-----------|--------------|------|
| Ordinateur de bord | Raspberry Pi Zero | Navigation, GPS, interface |
| Contrôleur primaire | Skyline32 Full (EMAX) | Temps réel, PID, servo |
| Contrôleur spare | CC3D Rev.C (OpenPilot) | Même firmware, flag différent |
| GPS | Module USB/UART | Cap fond, position |
| Compas | HMC5883L (intégré Skyline32) | Cap magnétique |

Voir [HARDWARE_INVENTORY.md](HARDWARE_INVENTORY.md) pour le détail complet.

---

## Philosophie firmware

Les deux cartes (Skyline32 et CC3D) partagent **le même firmware de base**, compilé avec un flag différent selon la cible :

```c
// Exemple de flag de compilation
#ifdef BOARD_SKYLINE32
  // compas et baro intégrés
#elif BOARD_CC3D
  // compas externe HMC5883L sur Flexi Port I2C
#endif
```

Cela garantit :
- Un seul codebase à maintenir
- Les deux cartes sont interchangeables (l'une est le spare de l'autre)
- Tout contributeur disposant d'une carte STM32F103 peut adapter le firmware

---

## Structure du dépôt

```
OpenTiller/
│
├── CONTEXT.md              ← Ce fichier
├── PROTOCOL.md             ← Protocole de communication Pi Zero ↔ contrôleur
├── HARDWARE_INVENTORY.md   ← Inventaire et fiches techniques du matériel
├── WIRING.md               ← Guide de câblage détaillé
│
├── controller/             ← Firmware STM32 (Skyline32 / CC3D)
│   ├── skyline32/          ← Code spécifique Skyline32
│   ├── cc3d/               ← Code spécifique CC3D
│   └── common/             ← Code partagé (PID, protocole, compas)
│
├── navigator/              ← Code Python Raspberry Pi Zero
│   └── ...                 ← Navigation, GPS, interface, daemon UART
│
├── docs/                   ← Documentation technique approfondie
│   └── ...
│
└── tools/                  ← Outils de développement et de test
    └── ...                 ← Simulateur, testeur de protocole, etc.
```

---

## Le protocole comme cœur du projet

Le fichier **PROTOCOL.md** est volontairement le document le plus important du projet. Il définit l'interface entre le Pi Zero et la carte contrôleur de manière suffisamment claire pour que :

1. N'importe qui puisse implémenter un contrôleur compatible sur sa propre carte
2. N'importe qui puisse implémenter un navigateur compatible sur sa propre plateforme
3. Les deux côtés soient testables indépendamment

C'est ce contrat d'interface ouvert qui fait d'OpenTiller un vrai projet communautaire, et non un simple fork de PyPilot.

---

## Contribuer

Toute carte microcontrôleur capable de :
- Lire un compas I2C ou SPI
- Générer un signal PWM 50Hz pour servo
- Communiquer en UART série

...peut devenir un contrôleur OpenTiller en implémentant PROTOCOL.md.

---

*Projet initié le 2026-03-28 par Vianney Sérandour (VIAS-PILOT)*
