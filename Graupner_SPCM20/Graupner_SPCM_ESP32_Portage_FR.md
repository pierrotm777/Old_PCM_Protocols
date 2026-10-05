# Portage Graupner/JR SPCM 1024 sur ESP32

## 1. Objectif

Le travail sur ESP32 avait pour but de reconstruire un signal
**Graupner/JR SPCM 1024** compatible avec un véritable module HF
Graupner et ses récepteurs, à partir d'une génération entièrement
logicielle et instrumentable.

Cette version a servi de référence pour comprendre le protocole avant
son portage sur AVR ATmega2560. Elle a notamment permis de valider la
structure des trames, le codage des voies C1 à C8, les voies
supplémentaires C9/C10, le cycle des champs de service et le Fail-Safe.

## 2. Structure de la trame SPCM

Une trame logique est constituée de **32 quintets de 5 bits**, organisés
en quatre groupes de huit quintets.

Chaque groupe contient six quintets de données et deux quintets de
contrôle. Les huit voies principales sont codées sur 10 bits.

Le placement retenu et validé est :

-   C1 = q16 × 32 + q17
-   C2 = q1 × 32 + q2
-   C3 = q19 × 32 + q20
-   C4 = q4 × 32 + q5
-   C5 = q24 × 32 + q25
-   C6 = q9 × 32 + q10
-   C7 = q27 × 32 + q28
-   C8 = q12 × 32 + q13

Les voies C9 et C10 sont des voies supplémentaires codées sur **8
bits**. Elles utilisent les champs AUX disponibles dans le cycle de
service.

Il n'existe pas, dans le format SPCM20 étudié, de C11/C12 équivalentes.

## 3. Conversion des valeurs servo

Les valeurs servo utilisées par le programme sont exprimées en
microsecondes. Elles sont converties vers la représentation SPCM avant
la construction de la trame.

C1 à C8 utilisent la résolution 10 bits du protocole. C9 et C10
utilisent une valeur 8 bits dérivée de la valeur SPCM.

Cette séparation a été importante pendant les essais : C9 a été vérifiée
avec le récepteur disponible, qui ne possède que neuf voies. C10 a donc
été générée par le programme mais n'a pas pu être vérifiée physiquement
sur ce récepteur.

## 4. Codage physique

Les quintets logiques sont transformés en motifs physiques à l'aide de
la table de correspondance SPCM de 32 entrées.

La trame physique obtenue est ensuite transformée en transitions
temporelles constituant le signal envoyé au module HF.

Les essais à l'analyseur logique ont permis de vérifier la répétition
correcte des trames et la cohérence des intervalles avec le tick SPCM
mesuré.

## 5. CRC

Les quatre groupes de la trame utilisent les polynômes CRC suivants :

-   Groupe A : `0x31`
-   Groupe B : `0x13`
-   Groupe C : `0x91`
-   Groupe D : `0x19`

Le calcul est effectué MSB en premier, avec initialisation et XOR final
à zéro.

Cette partie a été validée avant le portage AVR et n'a plus eu besoin
d'être remise en cause pendant les derniers essais.

## 6. Champs de service et Fail-Safe

L'étude de la version ESP32 a permis de comprendre le rôle des champs
AUX dans la transmission du Fail-Safe.

Une valeur AUX dont le bit 4 vaut 1 transporte une partie de la valeur
Fail-Safe. Les bits 3:2 sélectionnent le fragment transmis et les bits
1:0 transportent successivement les fragments de la valeur.

La valeur Fail-Safe est quantifiée de manière à conserver les bits 1:0 à
zéro.

Une valeur Fail-Safe nulle signifie **HOLD/OFF**.

L'ordre de construction retenu est :

1.  chargement des champs de service ;
2.  insertion des fragments Fail-Safe dans les champs concernés ;
3.  insertion de C9/C10 lorsque le champ AUX reste disponible ;
4.  insertion de C1 à C8 ;
5.  calcul des CRC ;
6.  conversion vers la trame physique.

Cette organisation permet de faire cohabiter C9/C10 et le Fail-Safe.

## 7. Commandes de test

La version de développement ESP32 comportait une console permettant de
modifier directement les voies et de tester le protocole sans dépendre
d'OpenAVRc.

Un mode `SWEEP` fait varier automatiquement C1 à C8. Il s'est révélé
particulièrement utile pour vérifier visuellement que les trames reçues
sont réellement renouvelées et qu'un récepteur n'est pas simplement
verrouillé sur une trame fixe.

Les commandes Fail-Safe permettent de définir individuellement la
position de C1 à C8, de remettre une voie en HOLD et de remettre toutes
les voies en HOLD.

## 8. Validation du Fail-Safe

Le comportement Fail-Safe a été vérifié sur le matériel réel.

Pendant un Sweep, la coupure de l'alimentation du lien provoque le
passage du récepteur vers les positions Fail-Safe programmées. Après
rétablissement de l'alimentation, la réception reprend immédiatement.

Cette validation a été importante car elle a fourni une référence fiable
pour le portage ATmega2560.

## 9. Rôle de la version ESP32

La version ESP32 doit être considérée comme la **référence fonctionnelle
du protocole** qui a permis de stabiliser :

-   le format des 32 quintets ;
-   le mapping C1 à C8 ;
-   C9/C10 ;
-   les CRC ;
-   le cycle de service ;
-   le codage Fail-Safe ;
-   les outils de test, notamment `SWEEP`.

Le portage vers ATmega2560 n'a donc pas consisté à réinventer le
protocole, mais à reproduire ce comportement avec une génération
temporelle adaptée à l'architecture AVR et, finalement, à l'intégration
OpenAVRc.
