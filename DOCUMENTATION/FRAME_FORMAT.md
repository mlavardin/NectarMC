# **NectarMC** — Format de trame série
> Référence des trames binaires attendues par **NectarMC** sur le port série.  
> Destinée à tout logiciel de vol ou simulateur générant des trames à destination de **NectarMC**.
 
---

##
Ce document décrit le format des trames à la fois côté bord et station sol afin que celles-ci puissent être ingérées par **NectarMC**

## Côté bord
 
Une trame Nectar côté bord est composée de trois blocs contigus. **Taille totale :** `4 + N` bytes.
```
┌───────────────────────────────────────────┬─────────────────┬───────────────────┐
│                  HEADER                   |    PAYLOAD      |  PACKET CONTROL   │
├───────────────────────────────────────────┼─────────────────┼───────────────────┤
│   MAGIC     │  Id_mission  │ payload_size │                 |      CRC16        │
│   1 Byte    │   2 Bytes    │   1 Byte     |     N Byte      |     2 Bytes       │
│    0xEB     │              │              |                 |                   │
└─────────────┴──────────────┴──────────────┴─────────────────┴───────────────────┘
```
> Les clubs doivent implémenter uniquement le Header, la Payload obligatoirement.
> L'implémentation du CRC côté bord est optionnel. Cependant il est obligatoire d'avoir un CRC16 en sortie du récepteur pour que la trame puisse être ingérée par **NectarMC**
 
## Header (4 bytes)
 
### Magic byte (Byte 0)
 
Ce champ-là doit être codé par un uint8
 
Il représente un octet de synchronisation. Sa présence permet à **NectarMC** de se **resynchroniser rapidement** après un problème de transmission : il scanne le buffer jusqu'à la prochaine occurrence de `0xEB` plutôt que de tenter un parsing octet par octet.
 
Le choix de `0xEB` repose sur deux critères :
- **Convention aérospatiale** — c'est le préfixe du mot de synchronisation IRIG-106 (`0xEB90`) ;
- **Propriétés binaires** — le motif `1110 1011` présente une densité de transitions élevée, le rendant statistiquement peu probable dans un flux de données aléatoires.
 
 
### Id_mission (Bytes 1–2)
 
Ce champ là doit être codé par un uint16
 
Les 16 bits encodent le SSID (10 bits, poids fort) et l'APID (6 bits, poids faible).
 
Le SSID identifie la **mission** (la fusée ou l'engin émetteur). Il est subdivisé en TYPE (bits 9–8) et NUM (bits 7–0, valeur 0–255).
 
Les 2 Bytes du champ Id_mission se décomposent ainsi :
``` 
Bits:  |15  14  13  12  11  10   9   8   7   6 | 5   4   3   2   1   0|
       ├───────────── SSID (10 bits) ──────────┼───── APID (6 bits) ──┤
       │        TYPE      │      NUM (0-255)   │     Application ID   │
       │      (2 bits)    │      (8 bits)      │      (0-63)          │
``` 
Les quatre types disponibles :
 
| Bits 9–8 | Préfixe    |
|:--------:|------------|
| `00`     | `FX`       |
| `01`     | `MF`       |
| `10`     | `BALLOON`  |
| `11`     | `OTHER`    |
 
Le champ `NUM` lui peut valoir une valeur entre 0 et 255
 
Exemples de SSID encodés :
 
| Identifiant  | Type (bits) | NUM (déc.) | SSID (hex) | SSID (bin)   |
|:------------:|:---------:|:----------:|:----------:|----------------|
| `FX99`       | `00`      | 99         | `0x063`    | `00 01100011`  |
| `FX7`        | `00`      | 7          | `0x007`    | `00 00000111`  |
| `MF12`       | `01`      | 12         | `0x10C`    | `01 00001100`  |
| `BALLOON3`   | `10`      | 3          | `0x203`    | `10 00000011`  |
| `OTHER200`   | `11`      | 200        | `0x3C8`    | `11 11001000`  |
 
 
L' **APID** ou '**Application Process Identifier** identifie le type de trame au sein d'une même mission. Valeur sur 6 bits : `0–63`.
 
Le router utilise la paire `(SSID, APID)` comme clé pour acheminer la trame vers la bonne décom.
 
 
### payload_size (Byte 3)
 
Ce champ est encodé par un uint8
 
Nombre d'octets du payload qui suit. Le router lit exactement ce nombre d'octets après le header avant d'atteindre le footer.

## Payload
 
La Payload est la partie personnalisable des trames. C'est ici où l'équipe du projet peut définir ces capteurs, ou tout autre paramètre qui doit redescendre au sol dans la télémétrie.
 
Si l'utilisateur souhaite décommuter la payload et non simplement la stocker en brut alors, il faut fournir un fichier qui la décrit.
 
Ce fichier est un JSON décrivant chaque champ de la payload un exemple se trouve dans : [FX99_7_decom.JSON](./Decom%20Database%20Example/FX99_7_decom.JSON)
 
Pour faciliter la création de la base de données qui permettra de décommuter la partie payload de la trame, un module est mis à disposition dans **NectarMC** (Decomb Module) qui offre une interface de création.
 
Côté bord, l'implémentation de la trame devra suivre le format généré dans l'outil.

## CRC16
Par défaut le CRC16 n'a pas besoin d'être implémenté dans le logiciel de vol de la fusée. En effet souvent ce sont les composants électroniques eux-mêmes qui font un CRC au niveau matériel à la fois côté récepteur et émetteur.

Toutefois, si vous souhaitez le désactiver, vous pouvez vous-même implémenter un CRC et le placer en fin de payload. Cependant ce sera à vous de le valider à la réception.

Dans tous les cas, une fois la trame au niveau du récepteur, indépendamment de l'implémentation côté fusée, un CRC16 doit être placé en bout de trame pour que Nectar puisse vérifier l'intégrité des données.

Pour réaliser un CRC, celui-ci est calculé sur la **totalité du header (4 bytes) + du payload (N bytes)**, puis annexé à la trame en **little-endian**.
 
### Paramètres du CRC
 
| Paramètre     | Valeur                                      |
|---------------|---------------------------------------------|
| Polynôme      | `0x1021`                                    |
| Valeur init   | `0xFFFF`                                    |
| Réflexion     | Non                                         |
| Endianness    | Little-endian                               |
 

### Exemple d'implémentation en C
```c
#define CRC16_CCITT_POLY 0x1021
#define CRC16_CCITT_INIT 0xFFFF

uint16_t crc16_ccitt(const uint8_t* data, size_t length)
{
    uint16_t crc = CRC16_CCITT_INIT;
    for(size_t i = 0; i < length < i++)
    {
        crc ^= (uint16_t) data[i] << 8;

        for(int bit = 0; bit < 8 bit++)
        {
            if(crc & 0x8000)
            {
                crc = (crc<<1) ^ CRC16_CCITT_POLY;
            }
            else
            {
                crc <<= 1;
            }
        }
    }
    return crc;
}
```

### Exemple d'implémentation en Python
 
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
 
### Exemple d'implémentation en Ada
Fichier spec :
```ada
with Interfaces; use Interfaces;

package CRC16 is

    type Unsigned_8_Array is array (Natural range <>) of Unsigned_8;
    function Crc16_Ccitt (Data : Unsigned_8_Array) return Unsigned_16;

end CRC16;
```
Fichier body :
```ada
package body CRC16 is

    Poly : constant Unsigned_16 := 16#1021#;
    Init : constant Unsigned_16 := 16#ffff#;

    function Crc16_Ccitt (Data : Unsigned_8_Array) return Unsigned_16 is
        Crc : Unsigned_16 := Init;   

        begin
            for I in Data'Range loop
                Crc := Crc xor Shift_Left(Unsigned_16 (Data(I)), 8);
                for Bit in 0..7 loop
                    if (Crc and 16#8000#) /= 0 then
                        Crc := Shift_Left(Crc, 1) xor Poly;
                    else
                        Crc := Shift_Left(Crc, 1);
                    end if;
                end loop;
            end loop;
        return Crc;
    end Crc16_Ccitt;

end CRC16;
```
---

## Construction d'une trame (Python)
 
```python
import struct
from pythonBackEnd.utils.frame_protocol import MAGIC_BYTE, crc16_ccitt
 
def build_frame(ssid_identifier: str, apid_value: int, payload: bytes) -> bytes:
    """
    ssid_identifier : ex. "FX99", "MF12", "BALLOON3"
    apid_value      : entier 0–63
    payload         : bytes de longueur arbitraire
    """
    ssid         = build_ssid(ssid_identifier)   # 10 bits
    apid         = apid_value & 0x3F             # 6 bits
    header_word  = (ssid << 6) | apid            # fusion en 16 bits
    payload_size = len(payload)                  # uint8
 
    header = struct.pack("<BHB", MAGIC_BYTE, header_word, payload_size)
    crc    = crc16_ccitt(header + payload)
    frame  = header + payload + struct.pack("<H", crc)
    return frame
```

## Côté Station
Vous pouvez vous référer au guide qui donne un exemple pour la station générique de **NectarMC** : [Nectar-RxStation-LoRa32](https://github.com/axpaul/Nectar-RxStation-LoRa32/blob/main/FRAME_GUIDE.md)

Dans les deux cas que le CRC soit géré à bord par l'électronique ou par le logiciel de vol, après une vérification d'une trame reçue depuis le bord, la station aura une trame de ce format :
```
┌───────────────────────────────────────────┬─────────────────┐
│                  HEADER                   │     PAYLOAD     |
├───────────────────────────────────────────┼─────────────────┤
│   MAGIC     │  Id_mission  │ payload_size │                 |
│   1 Byte    │   2 Bytes    │   1 Byte     |     N Byte      |
│    0xEB     │              │              |                 |
└─────────────┴──────────────┴──────────────┴─────────────────┘
```
On refait ensuite un calcul du CRC sur l'entièreté de la trame afin de pouvoir contrôle son intégrité côté **NectarMC**.

Voici un exemple avec les métadonnées disponibles actuellement
```
┌───────────────────────────────────────────┬───────────────────┬───────────────┐
│                 HEADER                    │      PAYLOAD      │     CONTROL   │
├───────────────────────────────────────────┼───────────────────┼───────────────┤
│   MAGIC     │  Id_mission  │ payload_size │      N data       │     CRC16     │
│   1 Byte    │   2 Bytes    │   1 Byte     │      bytes        │    2 Bytes    │
│    0xEB     │ (Little-End) │  (N bytes)   │                   │               │
└─────────────┴──────────────┴──────────────┴───────────────────┴───────────────┘
```

**C'est cette trame qui doit revenir dans NectarMC**

#### Création du fichier de décommutation
 
Les fichiers de decommutation sont des fichiers JSON de type **BDS**. Ils décrivent comment transformer une trame binaire en mesures lisibles.
 
Pour créer un fichier BDS depuis l'interface :
 
1. Ouvrir l'interface de création en cliquant sur le bouton "{ }"
2. Dans la selection du fichier **BDS File**, cliquer sur le bouton **Create BDS**.
3. Le module **Decomb Builder** s'ouvre.
4. Renseigner les informations générales de la trame : identifiant, nom, taille en octets, description si besoin.
5. Ajouter les champs de telemetrie un par un.
6. Pour chaque champ, définir son nom, sa taille en bits, son type de donnée, son unité et éventuellement une fonction de transfert.
7. Si certains champs représentent une position GPS, les marquer avec le rôle GPS correspondant.
8. Generer et telecharger le fichier JSON.
9. Revenir dans la creation du decom, importer ou selectionner ce fichier BDS, puis lancer le decom.
 
Le Decomb Builder calcule les offsets des champs et valide la structure avant de générer le JSON.

---
 
## Exemple concret — `FX99`, APID 7, payload 20 Bytes
 
**Calcul du header :**
 
```
MAGIC        = 0xEB
SSID  FX99   → type=00, num=99  → 0x063
APID  7                         → 0x07
Id_mission  = (0x063 << 6) | 0x07 = 0x18C7
payload_size = 20 = 0x14
```
 
**Octets du header (little-endian) :**
 
```
EB  C7  18  14
↑    ╰──┬──╯  ↑
magic uint16  uint8
       
```
 
**Trame complète :**
 
```
[ EB C7 18 14 ] [ <20 Bytes payload> ]  [ <2 Bytes CRC> ]
    header              payload               CRC
```
 
---
 
## Comportement de **NectarMC** en réception
 
**NectarMC** traite le buffer entrant selon la séquence suivante :
 
| Étape | État               | Action                                                                                     |
|:-----:|--------------------|-------------------------------------------------------------------------------------------|
| 1     | **Accumulation**   | Les octets entrants sont accumulés dans un buffer circulaire.                              |
| 2     | **Resync**         | Si `buffer[0] ≠ 0xEB`, avance jusqu'à la prochaine occurrence (`frames_lost += offset`). |
| 3     | **Parse header**   | Dès que ≥ 4 bytes sont alignés sur `0xEB`, le header est extrait.                        |
| 4     | **Faux magic**     | Header invalide malgré `0xEB` → avance d'1 octet et retente (`frames_lost++`).           |
| 5     | **Attente payload**| Si `buffer < 4 + payload_size + 2` bytes, le router attend.                              |
| 6     | **Validation CRC** | CRC reçu vs CRC recalculé — mismatch : trame consommée mais non transmise (`frames_crc_error++`). |
| 7     | **Envoi serveur**  | CRC valide → sérialisation JSON et envoi via WebSocket au serveur central.               |
 
### Objet JSON émis vers le serveur
 
```JSON

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
