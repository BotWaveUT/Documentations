# Documentation

Sommaire:


### Organistion du projet :

- **boot**      -> dossier à copier sur la carte SD pour booter sur la raspberry pi 3B
- **src**       -> dossier contenant les fichiers sources du projet
- **include**   -> dossier contenant les headers du projet
- **Makefile**  -> permet de compiler avec les fichiers présents dans src et include et génère l'image kernel à copier dans la carte SD

## Matériel

### Kernel

---

### Activation de la FPU

Pour pouvoir utiliser des valeurs flottantes et double il faut activer la *FPU* présente sur le CPU (cortex A53) de la raspberry pi.

Dans le fichier boot.S après le eret qui permet de passer en privilège EL1 lorsque le code entre dans *EL1_entry* les lignes suivantes ont été rajoutées : 

```asm
mov x0, #3
lsl x0, x0, #20
msr CPACR_EL1, x0
```
Ces lignes permettent de de mettre la valeur (0b11 << 20) dans le registre système `CPACR_EL1` pour activer la FPU et le SIMD.

---

### Fonctionnement du PCM

Le PCM est un module présent dans la raspberry pi pour communiquer via le protocole I²S.

Le code en rapport avec l'initialisation et l'utilisation de ce module est présent dans les fichiers *i2s.c* et *i2s.h*.

Pour initialiser le PCM, il faut suivre plusieurs étapes dans un ordre précis afin de s'assurer de la bonne initialisation.

1. Initialiser les broches GPIO concernant le PCM
2. Choisir la bonne horloge et choisir la bonne fréquence
3. Activer le PCM
4. Paramétrer les registres matérielles du PCM
5. Vider la FIFO TX et attendre 2 cycle de la clock d'entrée du PCM
6. Activer et paramétrer l'envoie de signaux vers la DMA

Ensuite il faut paramétrer et activer la DMA et ensuite nous pouvons activer la transmissions des données stockées dans la FIFO TX vers les broches.

```c
static void pcm_init_gpio()
{
    unsigned int FSEL1 = *(volatile unsigned*)GPFSEL1; //GPIO 18 - 19
    unsigned int FSEL2 = *(volatile unsigned*)GPFSEL2; //GPIO 20 - 21

    FSEL1 &= ~(((0b111) << 24) | ((0b111) << 27));
    FSEL2 &= ~((0b111) | ((0b111) << 3));

    FSEL1 |= ((0b100 << 24) | (0b100 << 27));
    FSEL2 |= ((0b100) | (0b100 << 3));

    *(volatile unsigned*)GPFSEL1 = FSEL1;
    *(volatile unsigned*)GPFSEL2 = FSEL2;
    
    *(volatile unsigned*)GPPUD = 0;
    delay(150);
    *(volatile unsigned*)GPPUDCLK0 = (1 << 18 | 1 << 19 | 1 << 20 | 1 << 21);
    delay(150);
    *(volatile unsigned*)GPPUDCLK0 = 0;
}
```

La fonction `pcm_init_gpio()` permet d'initialiser les broches 18, 19, 20, 21 en mode ALT0 associé à la valeur 4.

```c
FSEL1 &= ~(((0b111) << 24) | ((0b111) << 27));
FSEL2 &= ~((0b111) | ((0b111) << 3));

FSEL1 |= ((0b100 << 24) | (0b100 << 27));
FSEL2 |= ((0b100) | (0b100 << 3));
```
Cette partie permet de masquer l'ancienne valeur en mettant que des 0 dans les parties de GPFSEL1 et GPFSEL2 pour y écrire la nouvelle valeur dans les bits associés aux broches.

Par la suite nous précisons que nous n'utilisons aucune résistance de pull-up ou pull-down interne à la raspberry pi en écrivant 0 dans le registre GPPUD.

Nous devons attendre 150 cycles CPU pour prendre en compte la modification avant d'écrire dans le registre GPPUDCLK0 pour préciser sur quelles broches nous voulons ces modifications.

Maintenant que les broches sont bien paramétrées, nous allons passer à la gestion des clocks.

```c
static void pcm_init_clock() {
    *(volatile unsigned*)CM_PCMCTL = (CM_PASSWORD | (*(volatile unsigned*)CM_PCMCTL & (~ 0x10)));  //disable clock

    while(*(volatile unsigned*)CM_PCMCTL & (1 << BUSY));    //wait the clock is not busy

    *(volatile unsigned*)CM_PCMDIV = (CM_PASSWORD | (0x162 << 12) | 0x4EF); //set the divider

    *(volatile unsigned*)CM_PCMCTL = (CM_PASSWORD) | (0b01 << 9) | 0x10 | 0b0110; //set the clock with 1 division, PLLD used and enable
}
```
Cette fonction permet de paramétrer la clock utilisée par le module PCM via le registre CM_PCMCTL.

L'écriture dans chacun des registres en rapport avec les clocks nécessite de mettre une valeur dans les 8bits de poids forts appelé **CM_PASSWORD** qui est une sécurité matérielle pour prendre en compte chaque modification. Sans ce CM_PASSWORD, aucune modification ne sera prise en compte à chaque écriture.

Dans un premier temps, il faut désactiver la clock du PCM sur le bit 1.

Il faut attendre que tout soit bien désactivé en attendant que le bit associé à BUSY nous informe que le changement s'est bien établi.





### Fonctionnement de la DMA

## Traitement du signal