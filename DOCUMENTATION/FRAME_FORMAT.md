# SimpleGCS — Format de trame série

Ce document décrit le format binaire que le **Router** (`pythonBackEnd/router/router.py`) attend sur le port série (ou tout autre source de données). Il sert de référence pour tout firmware ou simulateur qui génère des trames à destination de SimpleGCS.

---

## Structure générale

```
┌─────────────────────┬──────────────────────────┬─────────────────┐
│   HEADER  (4 bytes) │   PAYLOAD  (N bytes)     │  CRC16 (2 bytes)│
└─────────────────────┴──────────────────────────┴─────────────────┘
```

| Champ   | Taille   | Description                                                  |
|---------|----------|--------------------------------------------------------------|
| Header  | 4 bytes  | Magic byte + identifiant de mission (SSID + APID) + taille payload |
| Payload | N bytes  | Données utiles (N déclaré dans le header)                    |
| CRC16   | 2 bytes  | Checksum CRC16-CCITT sur header + payload                    |

**Taille totale d'une trame :** `4 + N + 2` bytes

---

## Header — détail (4 bytes)

Le header est encodé en **little-endian** via `struct.pack("<BHB", MAGIC_BYTE, header_word, payload_size)`.

```
Byte 0          Byte 1          Byte 2          Byte 3
┌──────────┬───────────────────────────────┬───────────────┐
│ MAGIC    │  header_word (uint16 LE)      │ payload_size  │
│ 0xEB     │  [SSID:10 bits][APID:6 bits]  │   (uint8)     │
└──────────┴───────────────────────────────┴───────────────┘
```

### Magic byte (Byte 0) — `0xEB`

Octet de synchronisation. Sa présence permet au router de **resynchroniser rapidement** après un trou de transmission : il scanne le buffer pour la prochaine occurrence de `0xEB` au lieu de tenter un parsing byte-à-byte.

Choix de la valeur `0xEB` : convention aérospatiale (préfixe du sync IRIG-106 `0xEB90`), motif binaire `1110 1011` distinctif et peu probable en bruit aléatoire.

### Décomposition de `header_word` (16 bits, little-endian)

```
Bit:  15  14  13  12  11  10   9   8   7   6 │ 5   4   3   2   1   0
      ├──────────────────────────────────────┼───────────────────────┤
      │           SSID (10 bits)             │     APID (6 bits)     │
      └──────────────────────────────────────┴───────────────────────┘
```

```
header_word = (SSID << 6) | APID
```

### `payload_size` (Byte 3)

Nombre de bytes du payload qui suit. Le router utilise cette valeur pour savoir combien d'octets lire après le header avant d'atteindre le CRC.

---

## SSID — 10 bits

Le SSID identifie la **mission** (la fusée ou l'engin émetteur). Il est composé de deux sous-champs :

```
 Bits 9–8 │ Bits 7–0
 ├────────┼────────────────────────┐
 │  TYPE  │      NUM (0–255)       │
 └────────┴────────────────────────┘
```

### Types disponibles (bits 9–8)

| Valeur binaire | Préfixe  | Exemple d'identifiant |
|---------------|----------|-----------------------|
| `00`          | `FX`     | `FX99`, `FX7`         |
| `01`          | `MF`     | `MF12`, `MF1`         |
| `10`          | `BALLOON`| `BALLOON3`            |
| `11`          | `OTHER`  | `OTHER200`            |

### Exemples de SSID

| Identifiant  | Type bits | Num (dec) | SSID (hex) | SSID (bin)           |
|-------------|-----------|-----------|------------|----------------------|
| `FX99`      | `00`      | 99        | `0x063`    | `00 01100011`        |
| `FX7`       | `00`      | 7         | `0x007`    | `00 00000111`        |
| `MF12`      | `01`      | 12        | `0x10C`    | `01 00001100`        |
| `BALLOON3`  | `10`      | 3         | `0x203`    | `10 00000011`        |
| `OTHER200`  | `11`      | 200       | `0x3C8`    | `11 11001000`        |

---

## APID — 6 bits

L'APID (*Application Process Identifier*) identifie le **type de paquet** dans une même mission. Valeur sur 6 bits : `0–63`.

Le serveur route les trames vers le bon décom en utilisant la paire `(SSID, APID)` comme clé.

---

## CRC16-CCITT

Le CRC est calculé sur la **totalité du header (4 bytes) + du payload (N bytes)**, puis ajouté à la fin de la trame en **little-endian**.

### Paramètres

| Paramètre    | Valeur   |
|-------------|----------|
| Polynôme    | `0x1021` |
| Valeur init | `0xFFFF` |
| Réflexion   | Non      |
| Endianness  | Little-endian dans la trame (`struct.pack("<H", crc)`) |

### Algorithme de référence (Python)

```python
def crc16_ccitt(data: bytes, poly=0x1021, init=0xFFFF) -> int:
    crc = init
    for byte in data:
        crc ^= (byte << 8)
        for _ in range(8):
            if crc & 0x8000:
                crc = (crc << 1) ^ poly
            else:
                crc <<= 1
            crc &= 0xFFFF
    return crc

# Usage
crc = crc16_ccitt(header + payload)
crc_bytes = struct.pack("<H", crc)   # 2 bytes little-endian
```

L'implémentation partagée se trouve dans `pythonBackEnd/utils/frame_protocol.py` — utilisée à la fois par le router (validation) et le simulateur (génération).

---

## Construction complète d'une trame (Python)

```python
import struct
from pythonBackEnd.utils.frame_protocol import MAGIC_BYTE, crc16_ccitt

def build_frame(ssid_identifier: str, apid_value: int, payload: bytes) -> bytes:
    """
    ssid_identifier : ex. "FX99", "MF12", "BALLOON3"
    apid_value      : entier 0–63
    payload         : bytes de longueur arbitraire
    """
    ssid  = build_ssid(ssid_identifier)          # 10 bits
    apid  = apid_value & 0x3F                    # 6 bits
    header_word = (ssid << 6) | apid             # fusion en 16 bits
    payload_size = len(payload)                  # uint8

    header = struct.pack("<BHB", MAGIC_BYTE, header_word, payload_size)
    crc    = crc16_ccitt(header + payload)
    frame  = header + payload + struct.pack("<H", crc)
    return frame
```

---

## Exemple concret — FX99, APID 7, payload 20 bytes

```
MAGIC = 0xEB
SSID  = FX99  → type=00, num=99  → 0x063
APID  = 7                        → 0x07
header_word = (0x063 << 6) | 0x07 = 0x18C7
payload_size = 20 (0x14)

Header bytes (little-endian) : EB C7 18 14
                                ↑   ↑     ↑
                            magic uint16  uint8
                                   LE

Trame complète :
  [EB C7 18 14] [<20 bytes payload>] [<CRC lo> <CRC hi>]
     header             payload             CRC LE
```

### Parsing (côté router)

```python
magic, header_word, payload_size = struct.unpack("<BHB", header_bytes)
assert magic == 0xEB
ssid = (header_word >> 6) & 0x03FF   # → 0x063 = FX99
apid = header_word & 0x3F            # → 7
```

---

## Comportement du router en réception

1. **Accumulation** — les octets sont accumulés dans un buffer.
2. **Resync** — si le 1er octet du buffer ≠ `MAGIC_BYTE`, le router cherche la prochaine occurrence du magic byte et jette tout ce qui précède (`frames_lost += offset`).
3. **Tentative d'extraction** — dès que le buffer contient ≥ 4 bytes alignés sur un magic byte, le router parse le header.
4. **Header invalide** — si le parsing échoue malgré le magic byte (cas rare : faux magic byte), le router avance d'1 octet et retente la resync (`frames_lost++`).
5. **Attente payload** — si le buffer ne contient pas encore `4 + payload_size + 2` bytes, le router attend (`frames_incomplete`).
6. **Validation CRC** — le CRC reçu est comparé au CRC recalculé sur header+payload. En cas de mismatch, la trame est consommée mais **non transmise** au serveur (`frames_crc_error++`).
7. **Envoi au serveur** — si le CRC est valide, la trame est sérialisée en JSON et envoyée via WebSocket au serveur central.

### Objet JSON émis vers le serveur

```json
{
  "ssid":         99,
  "ssid_str":     "FX99",
  "apid":         7,
  "payload_size": 20,
  "header_hex":   "ebc71814",
  "payload_hex":  "...",
  "crc_hex":      "abcd",
  "frame_hex":    "ebc71814...abcd"
}
```
