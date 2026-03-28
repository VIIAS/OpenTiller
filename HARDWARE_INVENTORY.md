# OpenTiller — Inventaire Matériel

## Cartes contrôleur de vol (recyclées)

### Skyline32 (version Full) ✅ DISPONIBLE
- **Fabricant** : EMAX (clone Naze32)
- **MCU** : STM32F103CBT6 (72 MHz, 128 KB Flash, 20 KB RAM)
- **IMU** : MPU6050 (gyro + accéléro 6 axes)
- **Compas** : HMC5883L intégré (version Full)
- **Baro** : MS5611 intégré ✅ confirmé visuellement (composant avec 2 ports de pression)
- **Firmware de référence** : Baseflight / Cleanflight (github.com/multiwii/baseflight)
- **Rôle OpenTiller** : Contrôleur primaire (temps réel, PID, pilotage servo de barre)
- **Avantage** : Compas + baro intégrés, aucun composant externe nécessaire

#### Connectique détaillée (Skyline32)
| Port | Broches | Usage OpenTiller |
|------|---------|-----------------|
| ①RT (UART) | GND, +5V, RXD, TXD | **Liaison série Pi Zero ↔ Skyline32** |
| ②RC_IN | GND, +5V, CH1–CH8 | Non utilisé (CH3/CH4 = GPS TX/RX en mode CPPM) |
| ③BOOT | pad | Flashage firmware DFU |
| ④Vbat | GND, VCC | Monitoring tension batterie bord |
| ⑤PWM_OUT | PWM1–PWM6, +5V, GND | **PWM1 → servo moteur de barre** (Servo Mode : S1) |
| ⑦I2C | SDA, SCL, +5V, GND | Compas HMC5883L intégré + extension possible |
| ⑧SWD | SWDIO, SWDCLK, NRST, +3.3V, GND | Debug / programmation JTAG |
| ⑨Micro USB | — | Configuration Cleanflight |
| ⑩FSY | TX, GND | Télémétrie secondaire (optionnel) |

> **Note Servo Mode** : en activant le Servo Mode dans Cleanflight, PWM1→S1 et PWM2→S2 passent en PWM 50Hz standard pour servos. C'est le mode à utiliser pour le moteur de barre.

### CC3D — CopterControl 3D, Rev. C ✅ DISPONIBLE
- **Fabricant** : OpenPilot (open source hardware CC BY-SA)
- **MCU** : STM32F103CBT6 (72 MHz, 128 KB Flash, 20 KB RAM)
- **IMU** : MPU6000 (gyro + accéléro 6 axes)
- **Compas** : ❌ pas intégré — HMC5883L externe sur I2C (Flexi Port ou Main Port)
- **Baro** : ❌ pas intégré
- **Connectique** : PWM x6 sorties, Main Port (UART/I2C), Flexi Port (UART/I2C), SWD, Mini USB
- **Firmware de référence** : LibrePilot / OpenPilot / dRonin
- **Rôle OpenTiller** : Contrôleur secondaire / spare (même firmware, flag de compilation différent)
- **Avantage** : Hardware open source, ports Flexi/Main très pratiques pour liaison Pi Zero

#### Connectique détaillée (CC3D)
| Port | Face | Usage OpenTiller |
|------|------|-----------------|
| Mini USB | Arrière | Configuration Cleanflight/Betaflight (si BL Cleanflight) |
| Main Port | Avant | **Liaison série Pi Zero ↔ CC3D** (UART) + flashage firmware |
| Flexi Port | Avant | UART secondaire / I2C pour HMC5883L externe |
| PWM 1–6 | Avant droite | **PWM1 → servo moteur de barre** |
| SWD | Avant | Debug JTAG |
| Pads SBL + +3.3V | Avant droite | Shunt pour entrer en mode bootloader (si besoin) |

#### Flashage firmware (CC3D) — deux chemins
| Méthode | Format | Bootloader | Outil | Difficulté |
|---------|--------|-----------|-------|-----------|
| Simple | `.BIN` | Garde OpenPilot BL | LibrePilot GCS via USB | ⭐ Facile |
| Complet | `.HEX` | Remplace par Cleanflight BL | FTDI 3.3V + shunt pads SBL↔+3.3V | ⭐⭐ Moyen |

> **Statut actuel** : ❓ Peut-être déjà flashé Betaflight (à vérifier en branchant l'USB — si Cleanflight Configurator la reconnaît directement, c'est bon). Si oui : USB fonctionnel nativement, pas besoin de FTDI.

> **Référence** : https://www.helicomicro.com/2016/01/27/betaflight-cc3d-le-grand-tuto/

### KK Board ⚠️ RÉSERVÉ (valeur historique)
- **MCU** : ATmega168/328 (AVR 8 bits)
- **Rôle OpenTiller** : Aucun — conservé pour mémoire
- **Note** : Trop ancienne, architecture incompatible avec le projet

### APM 2.5 ❓ STATUT INCONNU
- **MCU** : ATmega2560 + ATmega32U2
- **Statut** : Probablement perdue ou hors service
- **Rôle OpenTiller** : Non retenu (architecture différente, overshoot pour ce projet)

---

## Ordinateur de bord

### Raspberry Pi Zero ✅ PRÉVU
- **MCU** : BCM2835 (ARM11 @ 1 GHz)
- **RAM** : 512 MB
- **OS** : Raspberry Pi OS Lite (sans bureau)
- **Rôle OpenTiller** : Navigation, calcul de cap, waypoints GPS, interface utilisateur
- **Connectique utilisée** : UART (vers Skyline32/CC3D), USB (GPS), WiFi (interface web)
- **Avantage** : Travaille sans contrainte temps réel, si crash → la carte contrôleur maintient le dernier cap

---

## Capteurs externes

### GPS ✅ PRÉVU (sur Pi Zero)
- **Interface** : USB ou UART
- **Rôle** : Position, cap fond, vitesse fond (COG/SOG)
- **Note** : Connecté uniquement au Pi Zero

### Compas HMC5883L ✅ PRÉVU (sur CC3D uniquement)
- **Interface** : I2C
- **Rôle** : Cap magnétique temps réel pour le contrôleur
- **Note** : Intégré nativement sur Skyline32 Full — externe requis uniquement sur CC3D

---

## Architecture de communication

```
[GPS] ──USB/UART──► [Raspberry Pi Zero] ──UART──► [Skyline32 ou CC3D]
                           │                              │
                     Navigation                    Contrôle temps réel
                     Waypoints                     PID + Servo barre
                     Interface web                 Tenu de cap
```

---

*Inventaire établi le 2026-03-28 — Photos vérifiées (Skyline32 EMAX + CC3D OpenPilot Rev.C)*
