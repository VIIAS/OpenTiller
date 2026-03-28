# OpenTiller — Protocole de Communication

> **Version** : 0.1 (draft)
> **Interface** : UART série entre Raspberry Pi Zero et carte contrôleur (Skyline32 / CC3D)
> **Statut** : Document fondateur — toute implémentation doit respecter ce contrat

---

## Préambule

Ce protocole est **le cœur documentaire du projet OpenTiller**. Il définit l'interface entre deux systèmes indépendants :

- **Le Navigateur** (Navigator) : Raspberry Pi Zero — calcule où aller
- **Le Contrôleur** (Controller) : Skyline32 ou CC3D — exécute comment y aller

Toute carte microcontrôleur respectant ce protocole devient un contrôleur OpenTiller valide. Toute plateforme respectant ce protocole devient un navigateur valide.

---

## 1. Paramètres physiques de la liaison

| Paramètre | Valeur | Notes |
|-----------|--------|-------|
| Interface | UART série | Matériel (pas SoftSerial) |
| Baudrate | **115200** | Standard, stable sur STM32F103 |
| Bits de données | 8 | |
| Parité | Aucune (N) | |
| Bits de stop | 1 | |
| Contrôle de flux | Aucun | |
| Niveau logique | 3.3V | Compatible direct Pi Zero GPIO |
| Connecteur Pi Zero | GPIO14 (TX) / GPIO15 (RX) | `/dev/serial0` ou `/dev/ttyAMA0` |
| Connecteur Skyline32 | Port ①RT (RXD/TXD) | |
| Connecteur CC3D | Main Port Pin 3 (TX) / Pin 4 (RX) | |

---

## 2. Format des trames

Toutes les trames utilisent la structure suivante :

```
[ STX ] [ CMD ] [ LEN ] [ DATA ... ] [ CRC8 ] [ ETX ]
```

| Champ | Taille | Valeur | Description |
|-------|--------|--------|-------------|
| STX | 1 octet | `0x24` (`$`) | Marqueur de début |
| CMD | 1 octet | voir §3 | Code de commande |
| LEN | 1 octet | 0–32 | Longueur du champ DATA en octets |
| DATA | 0–32 octets | variable | Données de la commande |
| CRC8 | 1 octet | calculé | CRC8 sur CMD + LEN + DATA |
| ETX | 1 octet | `0x0A` (`\n`) | Marqueur de fin (LF) |

**CRC8** : polynôme `0x07` (CRC-8/SMBUS), calculé sur les octets CMD, LEN et tous les octets DATA.

---

## 3. Commandes Navigator → Controller

### `0x01` — SET_HEADING (Consigne de cap)
Envoyé périodiquement par le Pi Zero. Le contrôleur maintient ce cap jusqu'à la prochaine consigne.

| Octet DATA | Type | Plage | Description |
|-----------|------|-------|-------------|
| 0–1 | uint16 big-endian | 0–3599 | Cap cible en dixièmes de degrés (ex. 1800 = 180.0°) |

Exemple : cap 247.5° → `0x09AB`
```
24 01 02 09 AB [CRC] 0A
```

---

### `0x02` — SET_MODE (Mode de fonctionnement)
Change le mode opérationnel du contrôleur.

| Octet DATA | Valeur | Mode |
|-----------|--------|------|
| 0 | `0x00` | **STANDBY** — servo centré, PID inactif |
| 0 | `0x01` | **AUTO** — tenu de cap actif (mode normal) |
| 0 | `0x02` | **MANUAL** — contrôle direct de la barre (voir SET_RUDDER) |

---

### `0x03` — SET_RUDDER (Commande directe barre)
Utilisé uniquement en mode MANUAL. Définit directement la position du servo.

| Octet DATA | Type | Plage | Description |
|-----------|------|-------|-------------|
| 0 | int8 | -100 à +100 | Position barre en % (-100 = bâbord max, +100 = tribord max, 0 = centré) |

---

### `0x04` — SET_PID (Réglage des gains PID)
Permet d'ajuster les gains PID depuis le Pi Zero sans reflasher le contrôleur.

| Octet DATA | Type | Description |
|-----------|------|-------------|
| 0–1 | uint16 | Gain P × 100 (ex. 150 = P=1.50) |
| 2–3 | uint16 | Gain I × 100 |
| 4–5 | uint16 | Gain D × 100 |

---

### `0x05` — PING
Trame vide (LEN=0), permet de vérifier que le contrôleur est vivant.

```
24 05 00 [CRC] 0A
```

---

### `0x06` — REQUEST_STATUS
Demande au contrôleur d'envoyer immédiatement une trame STATUS (voir §4).

---

## 4. Commandes Controller → Navigator

### `0x81` — STATUS (État du contrôleur)
Envoyé automatiquement à intervalle régulier (recommandé : toutes les 200ms) ET sur réception de REQUEST_STATUS.

| Octet DATA | Type | Plage | Description |
|-----------|------|-------|-------------|
| 0–1 | uint16 big-endian | 0–3599 | Cap mesuré actuel (dixièmes de degrés) |
| 2–3 | uint16 big-endian | 0–3599 | Cap cible actuel |
| 4 | uint8 | 0–2 | Mode actuel (0=STANDBY, 1=AUTO, 2=MANUAL) |
| 5 | int8 | -100 à +100 | Position servo actuelle (%) |
| 6 | uint8 | — | Flags d'état (voir §5) |

---

### `0x82` — PONG
Réponse à un PING. LEN=0.

---

### `0x83` — ACK
Accusé de réception d'une commande. Envoyé après toute commande bien reçue et appliquée.

| Octet DATA | Description |
|-----------|-------------|
| 0 | CMD de la commande acquittée |

---

### `0x84` — NACK (Erreur)
Commande reçue mais rejetée (CRC invalide, valeur hors plage, mode incompatible).

| Octet DATA | Description |
|-----------|-------------|
| 0 | CMD rejetée |
| 1 | Code d'erreur (0x01=CRC, 0x02=hors plage, 0x03=mode incompatible) |

---

## 5. Flags d'état (octet 6 du STATUS)

| Bit | Nom | Description |
|-----|-----|-------------|
| 0 | COMPASS_OK | 1 = compas opérationnel |
| 1 | HEADING_VALID | 1 = cap mesuré valide (pas de saturation magnétique) |
| 2 | LINK_TIMEOUT | 1 = aucune commande reçue depuis >2s (Navigator absent) |
| 3 | PID_SATURATED | 1 = sortie PID en butée (barre à fond) |
| 4–7 | réservés | 0 |

---

## 6. Comportement de sécurité (Failsafe)

**Règle fondamentale** : le contrôleur ne dépend jamais du Pi Zero pour sa sécurité.

| Événement | Comportement contrôleur |
|-----------|------------------------|
| Aucune trame reçue depuis **2 secondes** | Passe LINK_TIMEOUT=1, maintient le dernier cap |
| Aucune trame reçue depuis **30 secondes** | Passe en STANDBY, servo centré |
| CRC invalide reçu | Ignore la trame, envoie NACK, incrémente compteur d'erreurs |
| 10 CRC invalides consécutifs | Log d'erreur, maintien du mode actuel |

> **Principe** : un Pi Zero planté = bateau qui tient son cap, pas un bateau qui loffe ou abat brutalement. Le timeout de 30s pour le STANDBY est conservateur et ajustable via `SET_PID` (futur paramètre de config).

---

## 7. Séquence de démarrage typique

```
Navigator                          Controller
    │                                   │
    │── PING ──────────────────────────►│
    │◄─ PONG ───────────────────────────│
    │                                   │
    │── SET_MODE(STANDBY) ─────────────►│
    │◄─ ACK(SET_MODE) ──────────────────│
    │                                   │
    │── REQUEST_STATUS ────────────────►│
    │◄─ STATUS(cap actuel, STANDBY) ────│
    │                                   │
    │  [calibration compas si besoin]   │
    │                                   │
    │── SET_HEADING(270.0°) ───────────►│
    │◄─ ACK(SET_HEADING) ───────────────│
    │                                   │
    │── SET_MODE(AUTO) ────────────────►│
    │◄─ ACK(SET_MODE) ──────────────────│
    │                                   │
    │◄─ STATUS (toutes les 200ms) ──────│  ← boucle continue
    │── SET_HEADING (si cap change) ───►│
    │                                   │
```

---

## 8. Calcul CRC8

```python
def crc8(data: bytes) -> int:
    """CRC-8/SMBUS — polynôme 0x07"""
    crc = 0x00
    for byte in data:
        crc ^= byte
        for _ in range(8):
            if crc & 0x80:
                crc = (crc << 1) ^ 0x07
            else:
                crc <<= 1
            crc &= 0xFF
    return crc

# Exemple : SET_HEADING 247.5° (0x09AB)
# CMD=0x01, LEN=0x02, DATA=[0x09, 0xAB]
payload = bytes([0x01, 0x02, 0x09, 0xAB])
print(hex(crc8(payload)))  # à vérifier à l'implémentation
```

```c
/* Equivalent C pour le STM32 */
uint8_t crc8(const uint8_t *data, uint8_t len) {
    uint8_t crc = 0x00;
    for (uint8_t i = 0; i < len; i++) {
        crc ^= data[i];
        for (uint8_t j = 0; j < 8; j++) {
            if (crc & 0x80) crc = (crc << 1) ^ 0x07;
            else            crc <<= 1;
        }
    }
    return crc;
}
```

---

## 9. Implémentation de référence

| Côté | Langage | Dossier |
|------|---------|---------|
| Navigator (Pi Zero) | Python 3 | `navigator/` |
| Controller Skyline32 | C (STM32 bare-metal ou Cleanflight fork) | `controller/skyline32/` |
| Controller CC3D | C (même base, flag `BOARD_CC3D`) | `controller/cc3d/` |
| Code partagé controller | C | `controller/common/` |

---

## 10. Évolutions prévues (v0.2+)

- Commande `SET_CONFIG` pour paramètres persistants (timeout failsafe, limites servo, etc.)
- Trame `CALIBRATE_COMPASS` pour lancer la calibration depuis le Pi
- Champ vitesse de barre dans STATUS (pour feedback de charge moteur)
- Support optionnel du baromètre MS5611 dans STATUS (Skyline32 uniquement)

---

*Protocole OpenTiller v0.1 — 2026-03-28*
*Ce document est la référence contractuelle entre Navigator et Controller.*
*Toute modification de ce protocole incrémente le numéro de version.*
