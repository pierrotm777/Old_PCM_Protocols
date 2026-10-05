# Portage Graupner/JR SPCM 1024 sur ATmega2560 et intégration Protothreads

## 1. Objectif du portage

Après validation du générateur SPCM sur ESP32, l'objectif était de
porter le même protocole sur **ATmega2560**, cible correspondant à
l'environnement OpenAVRc.

Le point essentiel n'était plus seulement de produire les bons bits,
mais de reproduire un signal temporel suffisamment précis pour qu'un
véritable module HF Graupner accroche rapidement, conserve la liaison et
récupère immédiatement après une coupure.

Le second objectif était d'adapter progressivement le générateur au
fonctionnement **Protothreads (PT)** utilisé dans `PROTO_PCM.cpp`
d'OpenAVRc.

## 2. Pourquoi l'ATmega2560

Des essais avaient initialement été réalisés autour de l'ATmega328P/Uno.
Cette cible est rapidement devenue trop limitée en SRAM pour conserver
confortablement les doubles buffers et les données intermédiaires
nécessaires au générateur.

L'ATmega2560 dispose de suffisamment de RAM et correspond surtout à la
cible finale du projet.

La sortie retenue est :

`D12 = PB6 = OC1B`

Le signal est donc directement associé à la sortie matérielle **Output
Compare B de Timer1**.

## 3. Architecture temporelle finalement validée

La solution déterminante a été d'abandonner une génération logicielle
des fronts et de confier cette tâche au matériel Timer1.

Timer1 fonctionne avec un prescaler `/8` :

-   horloge CPU : 16 MHz ;
-   horloge Timer1 : 2 MHz ;
-   résolution : 0,5 µs par compteur.

Les constantes validées sont notamment :

``` cpp
static constexpr uint16_t SYNC_COUNTS  = 830;       // 415 us
static constexpr uint32_t FRAME_COUNTS = 88050UL;   // 44,025 ms
```

OC1B fonctionne en mode **toggle on compare match**. Le front est donc
produit directement par le périphérique matériel, sans dépendre du temps
d'exécution de l'ISR.

L'ISR se contente essentiellement de programmer l'événement suivant :

``` cpp
OCR1B += DurationBuf[a][idx];
```

Cette architecture s'est révélée beaucoup plus robuste que les premières
solutions.

## 4. Double buffering

Deux buffers de durées sont utilisés :

``` cpp
DurationBuf[2][MAX_DURATIONS]
```

Pendant qu'un buffer est transmis par Timer1, l'autre peut être préparé.

Les variables `ActiveBuf`, `PendingBuf` et `PendingReady` assurent la
synchronisation entre le producteur de la prochaine trame et l'ISR.

Cette séparation est fondamentale : **Timer1 émet, le constructeur
prépare**.

Le constructeur ne doit jamais intervenir directement dans le timing
d'un front déjà programmé.

## 5. Validation de la V11 non-PT

La version V11 FAILSAFE non-PT a constitué la première référence
pleinement stable sur ATmega2560.

Elle a validé :

-   l'accrochage du module HF ;
-   la période de trame ;
-   la stabilité temporelle ;
-   C1 à C8 ;
-   le mode `SWEEP` ;
-   C9 ;
-   le Fail-Safe ;
-   la récupération immédiate après coupure puis retour de
    l'alimentation.

Cette version doit être conservée comme référence de sécurité pour toute
évolution ultérieure.

## 6. Passage aux Protothreads

Le passage aux PT a été volontairement progressif.

Le principe recherché est le même que dans OpenAVRc : découper la
construction de la prochaine trame en petites opérations coopératives
afin de ne pas monopoliser le processeur.

Un point essentiel des Protothreads est qu'une variable automatique
locale ne doit pas être considérée comme persistante à travers un
`PT_YIELD()`.

Toutes les informations nécessaires après un Yield doivent donc être
stockées dans un contexte persistant.

Pour ce générateur, cet état est regroupé dans `GraPtCtx`.

## 7. PT1 à PT3 : identification des erreurs

Les premières adaptations PT ont été utiles pour identifier les limites
de l'architecture.

PT1 produisait un signal temporel très propre et le récepteur
accrochait, mais les servos restaient fixes.

PT2 a essayé de faire progresser le constructeur PT depuis l'ISR Timer1.
Cette direction s'est révélée mauvaise : les trames temporelles ont été
perturbées et les servos ne revenaient même plus correctement au neutre.

PT3 est revenu au principe sûr : PT hors ISR et Timer1 indépendant. Les
servos revenaient à 1500 µs, mais ne répondaient toujours pas.

Le diagnostic `SHOW` a alors fourni l'information décisive :

``` text
frame=1974 swaps=0 aborts=0 active=0 pending=1 ready=0
```

Timer1 continuait donc à transmettre les trames, mais aucune nouvelle
trame construite par le PT n'était publiée. Le buffer initial à 1500 µs
était simplement répété.

## 8. PT4 : première version PT totalement fonctionnelle

PT4 a volontairement simplifié le Protothread.

Le PT assure :

1.  la photographie des voies ;
2.  la construction logique ;
3.  un Yield ;
4.  la conversion physique ;
5.  un Yield ;
6.  l'appel au `buildDurations()` déjà validé de V11 ;
7.  la publication du buffer ;
8.  l'attente de sa consommation par l'ISR.

Cette version a immédiatement fonctionné : neutre, commandes et Sweep
ont été retrouvés.

PT4 constitue donc la **référence PT stable hybride**.

Elle utilise bien les Protothreads pour séquencer la préparation des
trames, mais `buildDurations()` reste exécuté en une seule fois.

## 9. PT5 FULL-PT

PT5 a été développée à partir de PT4 sans modifier Timer1, OC1B ni
l'ISR.

La différence importante est que la construction des durées est
elle-même devenue coopérative.

L'état nécessaire est conservé dans `GraPtCtx`, notamment :

``` cpp
uint16_t bitIndex;
uint16_t ones;
uint32_t previous;
uint32_t transition;
uint32_t delta;
```

La construction traite une transition physique, sauvegarde son état,
effectue un `PT_YIELD()`, puis reprend à la transition suivante.

Il n'y a donc plus d'appel monolithique à `buildDurations()` pour
préparer toute la trame.

Après correction d'une simple double déclaration de variables dans le
contexte, **PT5 FULL-PT FIX1 a été validée sur le matériel réel**.

Le récepteur accroche et les servos répondent correctement.

## 10. Architecture finale PT5

L'organisation finale peut être résumée ainsi :

``` text
loop()
  |
  +-- consoleTask()
  +-- updateSweep()
  |
  +-- GraBuildNextFramePt()
        |
        +-- snapshot C1..C10 + Fail-Safe
        +-- construction logique
        +-- PT_YIELD
        +-- conversion physique
        +-- PT_YIELD
        |
        +-- transition physique
        +-- PT_YIELD
        +-- transition physique
        +-- PT_YIELD
        +-- ...
        |
        +-- fin de trame
        +-- publication du buffer complet

Timer1 / OC1B
  |
  +-- génération matérielle des fronts
  +-- ISR courte
  +-- lecture du buffer actif
  +-- bascule vers le buffer prêt à la frontière de trame
```

Le timing critique et la construction de données sont ainsi clairement
séparés.

## 11. Fail-Safe

Le support Fail-Safe de V11 a été conservé pendant le passage PT.

Il reste désactivable à la compilation :

``` cpp
#define SPCM_FAILSAFE_ENABLED 1
```

Cette possibilité est volontaire, car OpenAVRc ne gère pas encore
nécessairement le Fail-Safe de la manière attendue par ce générateur.

Avec la valeur `0`, les champs concernés restent en HOLD/OFF et les
commandes Fail-Safe peuvent être bloquées.

Avec la valeur `1`, les positions Fail-Safe C1 à C8 sont injectées dans
le cycle de service SPCM.

## 12. Références à conserver

Trois étapes sont particulièrement importantes et ne doivent pas être
écrasées pendant l'intégration finale :

**V11 FAILSAFE non-PT**\
Référence du protocole et du timing Timer1/OC1B.

**V11 FAILSAFE PT4**\
Référence Protothreads simple et fonctionnelle.

**V11 FAILSAFE PT5 FULL-PT FIX1**\
Référence actuelle FULL-PT validée sur le matériel.

La règle pour la suite est de ne plus modifier le moteur Timer1/OC1B
validé sans raison mesurée. Les évolutions destinées à OpenAVRc doivent
porter principalement sur l'intégration du constructeur PT dans son
infrastructure existante.

## 13. Résultat du portage

Le portage a permis d'obtenir sur ATmega2560 un générateur Graupner/JR
SPCM :

-   compatible avec le véritable module HF testé ;
-   capable de piloter C1 à C8 et C9 ;
-   compatible avec le cycle de service et le Fail-Safe étudiés ;
-   stable à environ 44,025 ms par trame ;
-   basé sur Timer1/OC1B pour les fronts critiques ;
-   utilisant un double buffer ;
-   et désormais entièrement découpé en Protothreads pour la
    construction de la trame dans PT5 FULL-PT.

Cette version constitue une base directement exploitable pour la
prochaine étape : l'intégration propre du générateur SPCM dans
l'architecture `PROTO_PCM.cpp` d'OpenAVRc.
