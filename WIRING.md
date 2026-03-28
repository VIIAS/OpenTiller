# OpenTiller — Guide de Câblage

> Document de référence pour le câblage des cartes contrôleur dans le contexte OpenTiller.
> Toutes les connexions sont documentées avec les niveaux de tension et les précautions à respecter.

---

## 1. CC3D — Pinout Complet

### 1.1 Main Port (USART1) — JST-SH 4 broches
**C'est le port principal de communication avec le Raspberry Pi Zero.**

| Pin | Couleur fil std | Tension | Fonction série | Fonction I2C | Notes |
|-----|----------------|---------|---------------|-------------|-------|
| 1 | Noir | GND | GND | GND | Masse commune |
| 2 | Rouge | 4.8–15V | VCC non régulée | VCC non régulée | ⚠️ tension batterie brute — ne pas alimenter le Pi depuis ici |
| 3 | Bleu | **3.3V** | TX (sortie CC3D) | SCL | Direct compatible Pi Zero GPIO |
| 4 | Orange | **3.3V** (5V tolérant) | RX (entrée CC3D) | SDA | Accepte du 5V en entrée |

> **Pour OpenTiller** : Main Port = USART1 dans Cleanflight/Betaflight. C'est sur ce port qu'arrive la consigne de cap depuis le Pi Zero.

```
Pi Zero GPIO14 (TX) ──────► CC3D Main Port Pin 4 (RX)  [3.3V → 3.3V, OK direct]
Pi Zero GPIO15 (RX) ◄────── CC3D Main Port Pin 3 (TX)  [3.3V → 3.3V, OK direct]
Pi Zero GND         ──────► CC3D Main Port Pin 1 (GND)
```

⚠️ **Ne pas connecter la Pin 2 (VCC) au Pi Zero** — tension non régulée dangereuse pour le Pi.

---

### 1.2 Flexi Port (USART3 / I2C) — JST-SH 4 broches
**Même connecteur physique que le Main Port. Double usage exclusif : UART3 OU I2C.**

| Pin | Couleur fil std | Tension | Fonction UART3 | Fonction I2C | Notes |
|-----|----------------|---------|---------------|-------------|-------|
| 1 | Noir | GND | GND | GND | |
| 2 | Rouge | 4.8–15V | VCC non régulée | VCC non régulée | ⚠️ même avertissement |
| 3 | Bleu | **3.3V** | TX (UART3) | SCL | |
| 4 | Orange | **3.3V** (5V tolérant) | RX (UART3) | SDA | |

> **Règle importante** : USART3 et I2C sont **mutuellement exclusifs** sur ce port. Par défaut le Flexi Port est en mode I2C.

> **Pour OpenTiller avec CC3D** : utiliser le Flexi Port en **mode I2C** pour le compas HMC5883L externe.

```
HMC5883L SDA ──► CC3D Flexi Port Pin 4 (SDA)
HMC5883L SCL ──► CC3D Flexi Port Pin 3 (SCL)
HMC5883L VCC ──► 3.3V externe (du Pi Zero ou régulateur)
HMC5883L GND ──► CC3D Flexi Port Pin 1 (GND)
```

---

### 1.3 Sorties PWM — 6 broches standard (triple rangée)
**Format physique : rangée triple — GND (extérieur) / +5V (milieu) / Signal (intérieur)**

| Broche | Fonction drone | Fonction OpenTiller | Notes |
|--------|---------------|-------------------|-------|
| PWM 1 | Moteur 1 | **Servo moteur de barre** | Configurer en Servo Mode (50Hz) |
| PWM 2 | Moteur 2 | Servo secondaire (optionnel) | |
| PWM 3–6 | Moteurs 3–6 | Non utilisé | |

```
CC3D PWM1 Signal ──► Moteur de barre (fil signal)
CC3D PWM1 GND    ──► Moteur de barre (fil GND)
CC3D PWM1 +5V    ──► Moteur de barre (fil alim, si besoin)
```

---

### 1.4 Receiver Port — JST-SH 8 broches
En mode **RX_PPM / RX_SERIAL** (celui utilisé pour OpenTiller) :

| Pin | Fonction | Notes OpenTiller |
|-----|----------|-----------------|
| 1 | GND | |
| 2 | +5V | |
| 4 | SoftSerial1 TX / Sonar trigger | Disponible |
| 5 | SoftSerial1 RX / RSSI ADC | Disponible |
| 6 | Courant (0–3.3V ADC) | Monitoring courant (optionnel) |
| 7 | Tension batterie (0–3.3V ADC) | Monitoring Vbat |
| 8 | Entrée PPM | Non utilisé |

> Le Receiver Port peut aussi être reconfiguré en **4 sorties PWM supplémentaires** (canaux 7–10), portant le total à 10 sorties.

---

### 1.5 Alimentation CC3D

| Source | Tension | Notes |
|--------|---------|-------|
| USB | 5V | Configuration uniquement (si BL Cleanflight) |
| Servo headers (Pin milieu) | 4.8–15V | Source principale en utilisation |
| Receiver Port Pin 2 | 4.8–15V | Alternative |
| Consommation propre | ~70mA | Hors capteurs/servos |

> ⚠️ **Une seule source d'alimentation 5V à la fois.** Ne pas connecter plusieurs BEC simultanément.

---

### 1.6 Résumé câblage CC3D → OpenTiller

```
┌──────────────────────────────────────────────────────────────────┐
│                        CC3D (OpenTiller)                         │
│                                                                  │
│  Main Port                    Flexi Port                         │
│  ┌─────────────┐              ┌─────────────┐                    │
│  │ GND  ───────┼──────────────┼─► Pi Zero GND                   │
│  │ VCC  (NC)   │              │ VCC  (NC)   │                    │
│  │ TX   ───────┼──────────────┼─► Pi Zero RX (GPIO15)           │
│  │ RX   ◄──────┼──────────────┼── Pi Zero TX (GPIO14)           │
│  └─────────────┘              │ SCL  ───────┼─► HMC5883L SCL    │
│                               │ SDA  ───────┼─► HMC5883L SDA    │
│                               └─────────────┘                    │
│  PWM outputs                                                     │
│  PWM1 Signal ─────────────────────────────────► Servo barre     │
│  PWM1 GND ─────────────────────────────────────► Servo GND      │
└──────────────────────────────────────────────────────────────────┘
```

---

## 2. Skyline32 (EMAX) — Pinout Complet

### 2.1 Port ①RT (UART1) — connecteur 4 broches
**Port de communication série avec le Raspberry Pi Zero.**

| Broche | Signal | Niveau | Notes |
|--------|--------|--------|-------|
| GND | Masse | — | Commun avec Pi Zero |
| +5V | Alim | 5V | ⚠️ Ne pas connecter au Pi Zero (3.3V GPIO) |
| RXD | Réception Skyline32 | 3.3V | ← sortie TX du Pi Zero |
| TXD | Émission Skyline32 | 3.3V | → entrée RX du Pi Zero |

```
Pi Zero GPIO14 (TX) ──────► Skyline32 RT RXD  [3.3V, OK direct]
Pi Zero GPIO15 (RX) ◄────── Skyline32 RT TXD  [3.3V, OK direct]
Pi Zero GND         ──────► Skyline32 RT GND
```

> ⚠️ Vérifier le niveau logique du port RT avant connexion. La majorité des Skyline32 sortent du 3.3V sur le UART même si la logique interne est 5V. Un level shifter 5V↔3.3V reste conseillé par précaution.

---

### 2.2 Port ⑦ I2C — connecteur 4 broches
**Compas HMC5883L intégré sur ce bus. Extensible pour périphériques externes.**

| Broche | Signal | Niveau | Notes |
|--------|--------|--------|-------|
| GND | Masse | — | |
| +5V | Alim | 5V | Pour périphériques I2C |
| SCL | Horloge I2C | 3.3V | |
| SDA | Données I2C | 3.3V | |

> Sur la version Full, le HMC5883L est déjà câblé en interne sur ce bus. Un OLED ou autre module I2C peut être ajouté sur ce connecteur (adresses différentes).

---

### 2.3 Port ⑤ PWM_OUT — barrette 8 broches (PWM1 à PWM6 + +5V + GND)

| Signal | Mode Standard | Mode Servo | Fréquence |
|--------|--------------|-----------|----------|
| PWM1 | Moteur 1 | **Servo S1** | 50Hz (servo) ou 400Hz+ (moteur) |
| PWM2 | Moteur 2 | Servo S2 | idem |
| PWM3–6 | Moteurs 3–6 | Moteurs 3–6 | idem |

> **Servo Mode** à activer dans Cleanflight : `feature SERVO_TILT` ou via l'onglet Configuration. En Servo Mode, PWM1 et PWM2 passent automatiquement en 50Hz standard (1000–2000µs).

```
Skyline32 PWM1 Signal ──────────────────────────────► Servo/moteur de barre
Skyline32 PWM1 GND ──────────────────────────────────► GND servo
```

---

### 2.4 Port ② RC_IN — barrette 10 broches

| Broche | Signal | Mode CPPM | Notes |
|--------|--------|----------|-------|
| GND | Masse | GND | |
| +5V | Alim 5V | +5V | Alimentation récepteur |
| CH1 | Canal 1 | **Signal CPPM** | En CPPM, CH1 seul suffit |
| CH2 | Canal 2 | — | |
| CH3 | Canal 3 | GPS TX | En mode CPPM |
| CH4 | Canal 4 | GPS RX | En mode CPPM |
| CH5 | Canal 5 | LED OUT | WS2812 |
| CH6–CH8 | Canaux 6–8 | — | |

> **Pour OpenTiller** : RC_IN non utilisé (pas de radiocommande). Potentiellement réutilisable pour d'autres entrées/sorties.

---

### 2.5 Port ④ Vbat — 2 broches

| Broche | Signal | Notes |
|--------|--------|-------|
| GND | Masse | |
| VCC | Tension batterie | Jusqu'à 6S LiPo (~25V max). ⚠️ Pas de protection contre l'inversion de polarité |

---

### 2.6 Port ⑧ SWD — 5 broches

| Broche | Signal |
|--------|--------|
| SWDIO | Données debug |
| SWDCLK | Horloge debug |
| NRST | Reset |
| +3.3V | Alim debug |
| GND | Masse |

---

### 2.7 Alimentation Skyline32

| Source | Tension | Notes |
|--------|---------|-------|
| Micro USB | 5V | Configuration Cleanflight |
| +5V sur RC_IN ou PWM | 5V | Source principale via BEC |
| Vbat | 4S–6S | Monitoring tension uniquement |

> ⚠️ Pas de protection contre l'inversion de polarité sur Vbat. Vérifier la polarité avant connexion.

---

### 2.8 Résumé câblage Skyline32 → OpenTiller

```
┌──────────────────────────────────────────────────────────────────┐
│                     Skyline32 (OpenTiller)                       │
│                                                                  │
│  Port RT (UART)               I2C                                │
│  ┌─────────────┐              ┌─────────────┐                    │
│  │ GND  ───────┼──────────────┼─► Pi Zero GND                   │
│  │ +5V  (NC)   │              │ HMC5883L intégré (interne)       │
│  │ RXD  ◄──────┼──────────────┼── Pi Zero TX (GPIO14)           │
│  │ TXD  ───────┼──────────────┼─► Pi Zero RX (GPIO15)           │
│  └─────────────┘              └─────────────┘                    │
│                                                                  │
│  PWM_OUT (Servo Mode)                                            │
│  PWM1 Signal ─────────────────────────────────► Servo barre     │
│  PWM1 GND ─────────────────────────────────────► Servo GND      │
│                                                                  │
│  Vbat                                                            │
│  VCC ──────────────────────────────────────────► Batterie bord  │
└──────────────────────────────────────────────────────────────────┘
```

---

## 3. Écran LCD GMG12864-06D — Câblage et Configuration TinyPilot

### 3.1 Présentation

L'écran **GMG12864-06D Ver.2.2** est un LCD monochrome 128×64 pixels avec rétroéclairage bleu, couramment disponible sur AliExpress. Il est directement compatible avec le driver `jlx12864` de pypilot/TinyPilot.

> ⚠️ Les écrans Nokia 5110/3310 AliExpress (PCD8544, 84×48 pixels) sont fragiles et moins fiables. Le GMG12864-06D est une alternative de meilleure qualité recommandée pour OpenTiller.

---

### 3.2 Câblage GMG12864-06D → Raspberry Pi Zero

| Broche écran | Fonction | GPIO BCM | Pin physique Pi Zero |
|---|---|---|---|
| VDD | Alimentation 3.3V | — | Pin 17 |
| VSS | Masse | — | Pin 20 |
| SI | SPI MOSI (données) | GPIO 10 | Pin 19 |
| SCL | SPI SCLK (horloge) | GPIO 11 | Pin 23 |
| CS | SPI CE0 (chip select) | GPIO 8 | Pin 24 |
| RS | Data/Command | **GPIO 25** | **Pin 22** |
| RSE | Reset | **GPIO 24** | **Pin 18** |
| A (anode BL) | Rétroéclairage + | — | Pin 17 (3.3V) |
| K (cathode BL) | Rétroéclairage − | — | Pin 20 (GND) |

> ⚠️ Les broches **RS (GPIO 25, pin 22)** et **RSE (GPIO 24, pin 18)** sont spécifiques au driver pypilot — différentes des exemples Adafruit génériques (qui utilisent GPIO 23 pour DC). Utiliser impérativement ces pins.

```
Pi Zero                          GMG12864-06D
Pin 17 (3.3V) ──────────────────► VDD + A (backlight)
Pin 20 (GND)  ──────────────────► VSS + K (backlight)
Pin 19 (MOSI, GPIO10) ──────────► SI
Pin 23 (SCLK, GPIO11) ──────────► SCL
Pin 24 (CE0,  GPIO8)  ──────────► CS
Pin 22 (GPIO25) ────────────────► RS  (Data/Command)
Pin 18 (GPIO24) ────────────────► RSE (Reset)
```

---

### 3.3 Configuration TinyPilot — Problèmes connus et solutions

#### Problème 1 : Clignotement de l'écran

**Symptôme** : l'écran clignote en permanence lors du fonctionnement normal de pypilot_hat.

**Cause** : pypilot_hat rafraîchit l'écran en continu. Sur certaines versions du driver jlx12864, ce rafraîchissement rapide produit un clignotement visible.

**Solution** : ce comportement est inhérent au fonctionnement de pypilot_hat. L'image statique (hors pypilot_hat) est parfaitement stable. Aucune modification du câblage n'est nécessaire — le clignotement vient du refresh logiciel, pas du hardware.

#### Problème 2 : Contraste insuffisant (affichage très pâle)

**Symptôme** : l'affichage est visible mais très faible, texte difficilement lisible.

**Cause** : TinyPilot configure par défaut un contraste de ~60 et un backlight de ~13%, trop faibles pour le GMG12864-06D.

**Solution** : modifier `/home/tc/.pypilot/hat.conf` :

```json
{
  "lcd": {
    "contrast": 115,
    "backlight": 100
  }
}
```

Valeurs validées sur GMG12864-06D Ver.2.2 :
- **Contraste 115** — optimal, texte parfaitement lisible
- **Contraste 100** — lisible, légèrement moins contrasté
- **Backlight 100%** — luminosité maximale recommandée

Commande SSH pour appliquer directement :

```bash
# Se connecter : ssh tc@<ip> (mot de passe : pypilot)
python3.6 -c "
import json
with open('/home/tc/.pypilot/hat.conf') as f:
    c = json.load(f)
c['lcd']['contrast'] = 115.0
c['lcd']['backlight'] = 100.0
with open('/home/tc/.pypilot/hat.conf', 'w') as f:
    json.dump(c, f)
print('OK')
"
sv restart pypilot_hat
```

#### Driver à utiliser dans hat.conf

```json
"hat": {
  "lcd": {
    "driver": "jlx12864",
    "port": "/dev/spidev0.0"
  }
}
```

---

## 4. Raspberry Pi Zero — GPIO utilisés

| GPIO | Fonction | Connecté à |
|------|----------|-----------|
| GPIO14 (TXD) | Émission série | CC3D Main Port RX / Skyline32 RT RXD |
| GPIO15 (RXD) | Réception série | CC3D Main Port TX / Skyline32 RT TXD |
| GPIO8 (CE0) | SPI Chip Select | Écran GMG12864 CS |
| GPIO10 (MOSI) | SPI Data | Écran GMG12864 SI |
| GPIO11 (SCLK) | SPI Clock | Écran GMG12864 SCL |
| GPIO24 | LCD Reset | Écran GMG12864 RSE |
| GPIO25 | LCD Data/Command | Écran GMG12864 RS |
| GND | Masse commune | GND carte contrôleur + écran |
| GPIO2 (SDA) | I2C | ICM20948 SDA |
| GPIO3 (SCL) | I2C | ICM20948 SCL |

> **Note** : Sur Pi Zero v1, le port série matériel est `/dev/ttyAMA0`. TinyPilot l'utilise pour communiquer avec le contrôleur moteur (Naze32/CC3D).

---

## 4. Compatibilité des niveaux de tension

| Interface | CC3D | Skyline32 | Pi Zero | Compatibilité |
|-----------|------|-----------|---------|---------------|
| UART TX/RX | 3.3V | 3.3V* | 3.3V | ✅ Direct |
| I2C SDA/SCL | 3.3V | 3.3V | 3.3V | ✅ Direct |
| PWM sortie | 3.3V–5V | 5V | — | ✅ OK pour servo |

> *La plupart des Skyline32 EMAX utilisent 3.3V sur le UART malgré une logique 5V. À vérifier avec un multimètre en cas de doute.

---

## 5. Précautions générales

1. **Masse commune obligatoire** entre le Pi Zero et la carte contrôleur
2. **Ne jamais alimenter le Pi Zero depuis les VCC non régulés** des ports série des cartes
3. **Un seul BEC à la fois** sur les rails 5V des cartes
4. **Vbat sans protection inversion polarité** sur la Skyline32 — toujours vérifier le sens
5. **Le Main Port CC3D sert à la fois pour la liaison Pi Zero ET pour flasher** — déconnecter le Pi pour reflasher
6. **Flexi Port CC3D** : I2C et UART3 sont exclusifs — choisir l'un ou l'autre dans la config Cleanflight

---

*Sources : [OpenPilot Wiki CC3D](https://opwiki.readthedocs.io/en/latest/user_manual/cc3d/cc3d.html) · [Cleanflight CC3D docs](https://cleanflight.readthedocs.io/en/latest/Board%20-%20CC3D/) · [EMAX Skyline32 pinout](https://blog.seidel-philipp.de/emax-skyline32-pinlayout-anschlussplan/) · [Helicomicro CC3D tuto](https://www.helicomicro.com/2016/01/27/betaflight-cc3d-le-grand-tuto/)*
