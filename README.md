# omt-h264

Proposition non adoptée pour ajouter **H.264** comme second codec vidéo au
protocole [OMT (Open Media Transport)](https://github.com/openmediatransport),
en complément de VMX1 — pensée pour les liaisons Wi-Fi partagées où le
budget de bande passante est la contrainte dominante (ex. plusieurs
téléphones diffusant simultanément sur une même box).

**Ce dépôt est de la documentation, pas du code.** Il décrit le format de
trame, le FourCC, le modèle de débit et surtout la question de négociation
de capacités qui n'a pas de réponse dans le protocole actuel — voir
[SPEC.md](SPEC.md) pour le détail technique complet.

## Pourquoi pas de code ici

L'implémentation d'émission vit dans un SDK Android propriétaire
(`blue-broadcast/omt-android`), qui n'est pas un dépôt public. Ce que ce
dépôt donne à la place :

- la spécification du format de trame (suffisante pour ré-implémenter le
  côté émission ou réception indépendamment),
- une **release APK** signée de l'app de démonstration
  (`blue-broadcast/sample-app`) pour tester l'émission H.264 en conditions
  réelles, sans avoir accès au code du SDK,
- la partie réception, elle, **est** ouverte : fork MIT de `libomtnet` +
  fork GPL-2.0 du plugin OBS OMT, tous deux dans l'organisation
  `blue-broadcast`.

## Tester

1. Récupérer la dernière release de ce dépôt (APK).
2. L'installer sur un téléphone Android (minSdk 24).
3. Dans les réglages de l'app, choisir le codec **H.264** et une
   résolution/qualité.
4. Démarrer la diffusion — la source apparaît comme une source OMT
   standard sur le réseau local.
5. Côté récepteur : OBS + le plugin OMT forké
   (`blue-broadcast/omtplugin` + `blue-broadcast/libomtnet`) décode le
   flux H.264. Un récepteur qui ne connaît que VMX1 (vMix, par exemple)
   verra la source sans pouvoir la décoder — c'est précisément le problème
   de négociation de capacités que [SPEC.md](SPEC.md) documente.

## Liens

- Discussion amont : *(lien ajouté une fois postée sur
  [openmediatransport/discussions](https://github.com/orgs/openmediatransport/discussions))*
- [blue-broadcast/libomtnet](https://github.com/blue-broadcast/libomtnet) — réception H.264 (MIT)
- [blue-broadcast/omtplugin](https://github.com/blue-broadcast/omtplugin) — plugin OBS (GPL-2.0)
- [blue-broadcast/omt-android](https://github.com/blue-broadcast/omt-android) — SDK d'émission (propriétaire, non public)

## Licence

Texte de ce dépôt (README, SPEC) sous licence CC-BY. L'APK distribué en
release reste soumis à la licence du SDK propriétaire qui l'a produit —
voir les conditions d'utilisation dans l'app elle-même.
