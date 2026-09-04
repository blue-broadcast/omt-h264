# H.264 comme codec vidéo OMT — proposition

Statut : **proposition**, non adoptée en amont. Implémentée et testée de
bout en bout (émission Android + réception OBS) dans l'écosystème
`blue-broadcast`, distincte du SDK propriétaire lui-même (voir
[README.md](README.md) pour ce qui est et n'est pas ouvert ici).

## Motivation

Le codec vidéo natif OMT, **VMX1**, est intra-frame et vise un débit
constant en bits par pixel, indépendant du contenu. C'est le bon choix par
défaut pour un lien local à forte capacité (Ethernet, Wi-Fi 6 dédié) : très
faible latence de décompression, aucune dépendance à un décodeur matériel.

Sur un lien Wi-Fi **partagé** entre plusieurs sources (le cas visé ici :
plusieurs téléphones diffusant simultanément sur une même box), le budget
de bande passante devient la contrainte dominante. H.264 avec un encodeur
matériel (`MediaCodec` sur Android) offre, à qualité perçue comparable,
un débit très inférieur à VMX1 pour le même contenu — au prix d'une
compression inter-frame (latence de décodage plus élevée, dépendance à un
décodeur H.264 côté récepteur).

Cette proposition ne remplace pas VMX1 : elle ajoute H.264 comme **second
codec vidéo possible**, à négocier ou détecter selon les capacités du
récepteur.

## FourCC

```c
constexpr int32_t H264 = 0x34363248;  // ASCII "H264", little-endian
```

Valeur choisie par analogie avec les FourCC vidéo déjà définis dans le
protocole OMT (`VMX1`, `UYVY`, `YUY2`, `NV12`, `BGRA`) — même convention
4 caractères ASCII empaquetés en `int32_t` little-endian. **Non réservée
officiellement** par le projet amont ; à confirmer ou faire arbitrer avant
adoption, pour éviter toute collision avec un usage existant ou futur.

## Format de trame

Le frame vidéo OMT standard (`FrameHeader` 16 octets + `VideoExtHeader`
32 octets, transport TCP existant, ports 6400-6600) est réutilisé sans
changement structurel. Seul le contenu du payload et un champ diffèrent :

```c
struct VideoExtHeader {
    int32_t codec;         // = FourCC H264 au lieu de VMX1/UYVY/etc.
    int32_t width;
    int32_t height;
    int32_t frameRateN;
    int32_t frameRateD;
    float   aspectRatio;
    int32_t flags;
    int32_t colorSpace;
};
```

Le **payload** n'est plus une image décompressée (planaire ou entrelacée)
mais une **unité d'accès H.264 Annex B** déjà compressée par l'encodeur
matériel — telle quelle, sans ré-encapsulation supplémentaire (pas de
conteneur type MP4/fMP4, juste le flux Annex B start-code délimité que
produit `MediaCodec` en sortie).

Il s'agit d'un **pass-through** : l'émetteur ne fait *aucun* traitement du
flux H.264 au-delà de l'encapsuler dans l'en-tête OMT existant — il ne
décode jamais lui-même ce qu'il vient d'encoder. Le coût CPU côté émission
reste donc celui du seul encodage matériel.

## Négociation de capacités — l'angle le plus utile de cette proposition

**Le protocole OMT actuel n'a pas de mécanisme pour qu'un récepteur
annonce, avant connexion, quels codecs vidéo il sait décoder.** Le
récepteur découvre le codec effectivement utilisé en lisant
`VideoExtHeader.codec` **après** avoir reçu la première trame vidéo — il
n'y a pas d'étape de négociation antérieure comparable à celle qui existe
déjà pour les canaux (`<OMTSubscribe video="true" audio="true" .../>`).

Conséquence pratique observée pendant l'implémentation : un récepteur qui
ne sait pas décoder H.264 (ex. vMix, qui n'attend que VMX1) reçoit quand
même les trames et ne peut que les rejeter silencieusement — pas
d'indication à l'émetteur, pas de repli automatique.

Ce que l'implémentation de référence (fork `libomtnet`) a dû ajouter pour
rester robuste face à ce manque :

- Détection de disponibilité du décodeur H.264 côté récepteur avant
  d'accepter la première trame (`OMTH264Codec.IsAvailable`, qui dépend de
  la présence de FFmpeg) — sans négociation, ce test arrive après
  connexion, pas avant.
- Repli explicite en cas d'échec de décodage (`H.264 decoder waiting for
  a keyframe` tant que le premier keyframe n'est pas arrivé), plutôt qu'un
  crash ou un flux corrompu affiché.

**Proposition ouverte à la discussion** (c'est le point sur lequel des
retours seraient les plus utiles) : étendre `<OMTSubscribe .../>` avec un
attribut de capacités vidéo, par exemple :

```xml
<OMTSubscribe Video="true" Audio="true" VideoCodecs="VMX1,H264" />
```

L'émetteur choisirait alors le premier codec de la liste qu'il sait
produire, dans l'ordre de préférence du récepteur — rétrocompatible avec
un récepteur qui n'envoie pas cet attribut (comportement actuel : VMX1
implicite).

## Modèle de débit (implémentation de référence)

Le débit cible suit un modèle « bits par pixel » linéaire en résolution
et en fps, séparé de celui utilisé pour VMX1 :

```
kbps = bpp[qualité] × largeur × hauteur × fps / 1000
```

| Qualité | bpp H.264 | bpp VMX1 (référence) |
|---|---|---|
| Basse   | 0.08 | 0.20 |
| Moyenne | 0.15 | 0.36 |
| Haute   | 0.24 | 0.65 |

Ancrages mesurés (30 fps) : 720p Haute ≈ 6,5 Mbit/s, 1080p Haute ≈
15 Mbit/s — contre respectivement ≈ 18 et ≈ 40 Mbit/s pour VMX1 aux mêmes
réglages. Le ratio (~2,5×) reflète le gain inter-frame typique de H.264
face à un codec intra-frame, à qualité perçue comparable sur un contenu
caméra live (mouvement continu, pas d'animation graphique).

## Ce qui n'est pas couvert par cette proposition

- Pas de HEVC/AV1 : H.264 a été choisi pour sa disponibilité quasi
  universelle en encodage matériel sur Android (`MediaCodec`) et en
  décodage logiciel/matériel côté récepteurs desktop (FFmpeg).
- Pas de renégociation dynamique en cours de flux (changement de codec à
  chaud) : le codec est fixé au démarrage de la diffusion.
- Pas de gestion de B-frames : l'implémentation de référence encode en
  IPPP (pas de trames bidirectionnelles), pour garder une latence de
  décodage minimale malgré la compression inter-frame.

## Implémentation de référence

- **Émission** : SDK Android propriétaire (`blue-broadcast/omt-android`,
  non ouvert ici) — `MediaCodec` H.264 matériel, pass-through vers le
  frame OMT.
- **Réception** : fork MIT de `libomtnet` — décodage FFmpeg
  (`OMTH264Codec`), câblé dans un fork du plugin OBS OMT officiel.
- **Démonstration** : APK Android signé (voir [README.md](README.md)),
  installable sur un téléphone pour tester l'émission en conditions
  réelles face à n'importe quel récepteur OMT existant.
