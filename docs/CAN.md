# CAN sur autoradio Junsun « hcn » + appli CarCan — ce qu'on a appris

Relevé sur un Junsun V1 (MT8163, Android 9 affiché « 15 », firmware hcn) avec CarCan `com.hcn.autocan` v43 et un boîtier
SimpleSoft profil VW. D'autres firmwares fonctionnent autrement : utilise la méthode en fin de page.

Vu aussi sur un poste hcn **Rockchip RK3326** (Android 9, CarCan v5.0.1.20251014, liaison `/dev/ttyS3`, Seat Ibiza 2014) :
mêmes propriétés `persist.vendor.cem.cardoor1..5` et `persist.vendor.cem.angle`, ordinateur de bord lu par la méthode A.
Sur ce poste, le nom Bluetooth bascule sans cesse dans `Settings.Secure` (ignoré par le journal depuis la 1.18.1).

**Retard des propriétés** : sur ce poste, `cardoor*` passe à 2 trois à cinq secondes après l'ouverture (profil Raise
`VW/Skoda/Seat ALL_R01`, et encore 3 s juste après un changement de profil). La fenêtre portes de CarCan (réglage
« display door info » = Open, mise en page `door_layout`) s'affiche, elle, tout de suite : depuis la 1.18.3 le lanceur s'en
sert comme avance (calques `leftfrontdoor`, `rightfrontdoor`, `leftreardoor`, `rightreardoor`, `reardoor` = coffre,
`frontdoor` = capot ; `carbg` et `closedoor` = fond). Version du boîtier lue par CarCan : `VW-RZ-08-0043.212.04-HSE`.

## 1. Lu en direct, sans autorisation : propriétés système `persist.vendor.cem.*`

| Propriété | Valeur |
|---|---|
| `cardoor1..5` | 1 = fermé, 2 = ouvert — ordre : avant gauche, avant droite, arrière gauche, arrière droite, coffre |
| `light0` / `light1` | clignotant gauche / droit (2 = allumé) |
| `speed` | vitesse CAN (km/h) si `persist.sys.cem.speed.from.can` = 1 |

Lecture : `android.os.SystemProperties.get()` par réflexion, toutes les 0,5 s (`CanBridge.java`). Correspondance modifiable
dans `ibiza_map.json` (`"cemDoors": [0,1,2,3,4]`).

## 2. Ordinateur de bord : affiché seulement par CarCan

CarCan lit le boîtier sur la liaison série (`/dev/ttyMT2`) et garde la liaison pour lui (impossible de la partager).
Son service AIDL `ICustomDataServices` n'envoie aucune donnée. Il publie seulement dans `Settings.System` :
`car_speed_value`, `key_can_average_speed`, `key_can_gear`.

**Méthode A (sans autorisation)** : ouvrir CarCan directement sur la page des infos, lire l'écran par l'accessibilité,
revenir au lanceur, le tout sous une image figée du lanceur (fenêtre `TYPE_ACCESSIBILITY_OVERLAY`, `Curtain.java`).
- Ouverture directe : extra `ToFragment` = `com.hcn.autocan.SimpleSoft.VW.VWInfoFragment`
  (+ `FLAG_ACTIVITY_CLEAR_TOP | FLAG_ACTIVITY_SINGLE_TOP`, sinon une instance déjà lancée ne reçoit pas l'extra).
- Les cases du menu CarCan ne réagissent qu'au **toucher** (`OnTouchListener`) : clic d'accessibilité sans effet,
  il faut `dispatchGesture` (capacité `canPerformGestures`).
- Identifiants des vues : `odbtextName` / `odbtextValue`, `textValue` / `textUnit` / `textName` (`CarText.parseHcn`).

**Méthode B (en direct, permission READ_LOGS par ADB ou root)** : CarCan écrit chaque morceau reçu du boîtier dans le
journal : `I CanService: rev: 2E41…` → décodé par `LogCan.java`.

## 3. Trames du boîtier SimpleSoft, profil VW (d'après `VWDataDecorder` de CarCan)

Format : `2E | cmd | len | données (len octets) | chk`, chk = (cmd + len + données) XOR 0xFF.

| cmd | Contenu |
|---|---|
| `0x14` | éclairage tableau de bord |
| `0x20` | boutons au volant |
| `0x21` | climatisation |
| `0x22` / `0x23` | radars arrière / avant |
| `0x24` | portes |
| `0x25` | coupure son radars |
| `0x26` | angle du volant |
| `0x27` | ampli / égaliseur |
| `0x30` | version du boîtier |
| `0x41` | infos véhicule (ci-dessous) |

`0x41` (index dans le tableau [cmd, len, d…] reçu par le décodeur) :
- sous-type 1 (`[2]` = 1) : `[3]` bit 6 lave-glace bas, bit 7 ceinture, bits 0–3 portes
- sous-type 2 : régime `[3..4]`, vitesse `[5..6]` × 0,01 km/h, batterie `[7..8]` × 0,01 V,
  extérieur (signé) `[9..10]` × 0,1 °C, kilométrage `[11..13]`, carburant `[14]` L
- sous-type 3 : `[3]` bit 7 alerte carburant, bit 6 alerte batterie

Attention : le dépôt « simplesoft-canbus-box-reverse-engineer » décrit le boîtier **Toyota** (RP5-TY-101) : numéros et
formats différents du profil VW.

## 4. Méthode pour un autre autoradio / une autre appli CAN

1. Lanceur installé : Réglages > Diagnostic CAN > *Exporter le journal* (ouvre/ferme des portes) et *Copier les APK CAN*.
2. Le journal liste les propriétés système, fenêtres et textes vus : chercher les clés qui changent avec les portes.
3. Décompiler l'APK CAN (androguard / jadx) : chercher le port série, le format des trames (classe `PacketManager` ou
   équivalent), le décodeur du profil véhicule, les `Log.*` qui écrivent les trames, les extras d'ouverture d'activité.

## 5. Autoradios TopWay (TS10 / TS18, firmware « tw »)

Vu sur un poste « Plateforme d'informations : TS-10 », puce Unisoc UIS8581 (SC9863A), Android 10, dans une Ford Fiesta.
Pas de CarCan, pas de `persist.vendor.cem.*` : les propriétés système sont `persist.tw.*` (`persist.tw.reverse`, `persist.tw.park`…).

| Élément | Où |
|---|---|
| Appli système | `com.tw.service` (uid système) : fenêtres portes, climatisation, radars ; `com.tw.service.xt` : commandes (BT, musique, radio, marche arrière) pour les applis tierces |
| Fenêtre portes | `com.tw.service`, layout `door` : vues `door_fl`, `door_fr`, `door_bl`, `door_br` visibles = porte ouverte. `door_b` = silhouette toujours visible ; coffre et capot sont dans le niveau de son image (`setLevel((état >> 4) & 3)`), **invisibles pour l'accessibilité** |
| Microcontrôleur | classe système `android.tw.john.TWUtil` : `open(short[] numéros)`, `start()`, `addHandler(nom, Handler)`, `removeHandler`, `stop`, `close`. Messages : `what` = numéro, `arg1`, `arg2`, `obj` (octets ou texte) |
| Trames CAN | message `0x0501` : `arg1` = commande du protocole du boîtier, `obj` = octets. Le décodage dépend du protocole (`ma.dg`) et du modèle (`ma.mModel`), lus dans la version MCU (message `0x010A`, texte `xxx-xxx-<modèle>-…`) |
| Marche arrière | message `0x9F1C`, `arg1` = 1 en marche arrière |

Ce que fait le lanceur (1.22.1) : portes lues dans la fenêtre portes (accessibilité, sans retard notable) ; Réglages > Données CAN du poste >
*Sonder le CAN du poste TopWay (3 min)* ouvre TWUtil en lecture seule et note dans le journal chaque message qui change (plafonné),
puis un résumé. Ces trames servent à décoder coffre, vitesse, température, carburant pour le boîtier concerné. Si le poste refuse
l'accès à TWUtil à une appli non système, le journal l'indique (« TWUtil inaccessible »).

### Portes TopWay lues sur le CAN (1.24.2)

Relevé avec le sondage sur une Ford Fiesta 2010 5 portes (TS10, Unisoc UIS8581, boîtier Raise, profil
`Raise/ford/09-12 fiesta / highconfig`), en comparant avec la fenêtre portes de `com.tw.service` :

| Message | Contenu |
|---|---|
| `0x0505`, `arg1` = 0, 1 octet | état des portes (même octet que ci-dessous), arrive en premier |
| `0x0501`, `arg1` = `0x24`, 6 octets | octet 0 = portes ; octet 1 bit 0 = marche arrière |
| `0x0501`, `arg1` = `0x2A`, 16 octets | octet 2 = portes, autre codage (avant gauche `20`, avant droite `10`, arrière gauche `08`, arrière droite `04`) |
| `0x9F1C` | `arg1` = 1 en marche arrière (répété toutes les 0,5 s) |

Octet des portes : `40` avant gauche, `80` avant droite, `10` arrière gauche, `20` arrière droite, `00` tout fermé ;
`08` coffre et `04` capot supposés (non ouverts pendant le relevé). Pas de radars remontés sur cette voiture.
Le lanceur garde TWUtil ouvert en lecture seule et décode `0x0505/0` et `0x0501/0x24` (`Ts10Probe.doors`) ;
bits modifiables dans `ibiza_map.json` (`"ts10DoorBits"`).

