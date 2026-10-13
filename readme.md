# ASM1 — Projet ELF64 / assembleur x86-64

**ESGI 3A — Projet assembleur**  
**Étudiants :** Thomas Kappler et Théo Robert  
**Environnement :** Ubuntu 24.04, x86-64, NASM

## 1. Présentation

Ce projet porte sur le format des exécutables **ELF64**, leur chargement en mémoire et l'étude de la transformation d'un Program Header **`PT_NOTE → PT_LOAD`**. Il comprend un programme de démonstration, un module de manipulation ELF et deux variantes de code assembleur permettant d'étudier les modèles de communication réseau **bind** (connexion entrante) et **reverse** (connexion sortante) dans un laboratoire isolé.

Ce guide décrit les fichiers remis, la compilation et l'inspection du programme de démonstration, les outils employés et les résultats observés. Les essais réseau sont présentés par leur objectif et leurs observations.

## 2. Répartition

| Membre | Partie |
| --- | --- |
| **Théo Robert** | Lecture et analyse ELF64, Program Headers, architecture du module de modification ELF. |
| **Thomas Kappler** | Variantes NASM bind/reverse, étude des syscalls, du PIC, des processus et tests en VM. |
| **En commun** | `INTERFACE.md`, tests, documentation et présentation. |

## 3. Fichiers

| Fichier | Rôle |
| --- | --- |
| `victime.asm` | Programme témoin en assembleur affichant un message de contrôle. |
| `injecteur.asm` | Module NASM de manipulation des métadonnées ELF. |
| `injecteur_reverse.asm` | Variante du module destinée aux essais reverse, générée à partir de la source principale dans le laboratoire. |
| `bindshell.asm` | Variante étudiant le modèle de communication bind. |
| `reverseshell.asm` | Variante étudiant le modèle de communication reverse. |
| `INTERFACE.md` | Contrat de format et de coopération entre les composants. |
| `Présentation ASM1.pptx` | Support de soutenance (vérifier le nom exact dans le bundle). |

Les principaux composants source sont écrits en **NASM x86-64**. GCC et GNU Binutils interviennent lors de la construction et de l'analyse.

## 4. Prérequis

- Système Linux x86-64 ; environnement de référence : **Ubuntu 24.04**.
- **NASM**, **GCC**, **GNU Binutils** (`ld`, `objcopy`, `readelf`, `objdump`).
- Outils d'inspection : **xxd**, **file** et, si nécessaire, **GDB**.
- Deux VM en réseau isolé *Host-only* pour les observations réseau consignées par le binôme.

Contrôler la présence des outils :

```bash
nasm -v
gcc --version
ld --version
objcopy --version
readelf --version
objdump --version
file --version
```

## 5. Compilation, exécution et vérifications

### 5.1. Compiler et exécuter `victime.asm`

Depuis la racine du projet :

```bash
nasm -f elf64 victime.asm -o victime.o
gcc -no-pie -nostartfiles -o victime victime.o
./victime
```

Sortie attendue :

```text
Programme original execute avec succes !
```

- `nasm -f elf64` assemble le code en **fichier objet ELF64**.
- `gcc -no-pie -nostartfiles` réalise l'édition de liens en demandant un exécutable **non-PIE**, sans les fichiers de démarrage standards ; la source possède son propre `_start`.
- `./victime` exécute le programme témoin.

### 5.2. Inspecter le format ELF

```bash
file victime
readelf -h victime
readelf -l victime
readelf -S victime
objdump -d victime
```

- `file` identifie le type de fichier.
- `readelf -h` affiche l'**ELF Header**, notamment son point d'entrée.
- `readelf -l` affiche les **Program Headers** (`LOAD`, `NOTE`, etc.).
- `readelf -S` affiche les **Section Headers**.
- `objdump -d` désassemble le code machine.

Vérification ciblée effectuée pendant les tests :

```bash
readelf -l victime | grep NOTE
```

Extrait de sortie communiqué :

```text
NOTE  0x0000000000000120  0x0000000000400120  0x0000000000400120
```

**Attention :** les options d'édition de liens ne garantissent pas, à elles seules, la présence d'un segment `PT_NOTE` dans tous les environnements.

### 5.3. Inventaire des binaires générés dans le laboratoire

Lorsque les composants ont déjà été construits dans l'environnement de test, ces commandes permettent d'en vérifier l'existence et les caractéristiques, sans établir de connexion réseau :

```bash
ls -lh victime *.o *.bin injecteur injecteur_reverse victime_infectee 2>/dev/null
file victime victime_infectee 2>/dev/null
readelf -h victime_infectee 2>/dev/null
readelf -l victime_infectee 2>/dev/null
```

Les fichiers `bindshell.asm`, `reverseshell.asm` et `injecteur.asm` sont décrits en section 3 et dans `INTERFACE.md`. Le présent guide ne détaille pas la chaîne d'injection et de déploiement d'un shell distant.

### 5.4. Commandes et outils complémentaires

| Commande | Fonction |
| --- | --- |
| `ld -o sortie objet.o` | Effectue l'édition de liens vers un exécutable. |
| `objcopy -O binary -j .text` | Extrait les octets de la section `.text` vers un binaire brut. |
| `xxd fichier` | Affiche les octets d'un fichier sous forme hexadécimale. |
| `grep` | Filtre des lignes ; appliqué à `xxd`, il ne prouve pas à lui seul l'unicité d'une séquence d'octets. |

### 5.5. Vérifier les résultats de la démonstration

Ces commandes inspectent les fichiers existants :

```bash
# Tailles des binaires bruts
ls -la bindshell.bin reverseshell.bin

# Inspection hexadécimale
xxd bindshell.bin
xxd reverseshell.bin

# Comparaison des informations ELF
readelf -h victime
readelf -l victime
readelf -h victime_infectee
readelf -l victime_infectee

# Exécution du programme témoin original
./victime
```

Les résultats de `readelf` décrivent la structure du fichier examiné ; ils ne garantissent pas à eux seuls sa fiabilité générale. Une sortie de `xxd | grep "48 b8"` ne permet pas de compter rigoureusement les occurrences d'une séquence d'octets.

## 6. Résultats observés

| Vérification | Observation communiquée |
| --- | --- |
| Assemblage et exécution de `victime` | Réussis ; message attendu affiché. |
| Segment `PT_NOTE` dans `victime` | Présence confirmée avec `readelf -l`. |
| Taille de `bindshell.bin` | **313 octets**. |
| Taille de `reverseshell.bin` | **211 octets**. |
| Compilation de `injecteur` | Exécutable obtenu (**9 432 octets** dans le journal fourni). |
| Compilation de `injecteur_reverse` | Exécutable obtenu (**9 448 octets** dans le journal fourni). |
| Création de `victime_infectee` | Confirmée ; **12 499 octets** dans l'essai reverse communiqué. |
| Exécution du fichier résultant | Message du programme original affiché lors du test. |
| Valeur du point d'entrée après modification | `0x403000` rapporté pour un essai antérieur ; comparer avec les fichiers finaux. |
| Modification des Program Headers | Signalée lors des essais ; vérifier les sorties `readelf` correspondant à la version finale. |
| Communication bind et reverse | Essais local et entre deux VM rapportés par le binôme. |
| Observation sous GDB | Test anti-debug rapporté ; portée limitée au scénario observé. |

Ces observations concernent les cas de test du laboratoire ; elles ne garantissent pas la prise en charge de tous les fichiers ELF64.

## 7. Contrat d'interface

Le fichier **`INTERFACE.md`** définit la répartition des composants, le format binaire brut utilisé pour les variantes assembleur, les contraintes générales (x86-64, PIC, syscalls Linux) et les informations nécessaires à leur intégration.

**Point à actualiser :** le contrat indique une taille maximale approximative de 250 octets, alors que `bindshell.bin` mesure **313 octets** dans le journal transmis. La documentation doit refléter cette différence.

## 8. Limites et dépannage

### Limites

- Environnement de référence restreint à **Linux x86-64** et à des ELF64 adaptés aux essais.
- Prise en charge générale des exécutables PIE et d'autres configurations ELF non démontrée.
- Buffers de taille fixe et vérifications incomplètes des entrées.
- Gestion partielle des lectures/écritures courtes et des erreurs système.
- Observations réseau dépendantes de la configuration des VM et de leurs permissions.

### Dépannage

| Symptôme | Contrôle utile |
| --- | --- |
| `nasm: command not found` | Vérifier l'installation de NASM avec `nasm -v`. |
| Fichier `victime` absent | Vérifier la sortie des étapes NASM et GCC. |
| `Permission denied` | Vérifier les droits avec `ls -l victime`. |
| Aucun `NOTE` affiché | Vérifier `readelf -l victime` et la configuration de l'édition de liens. |
| Fichier résultant illisible | Comparer `file`, `readelf -h` et `readelf -l` des fichiers disponibles. |
| Différences entre les résultats du README et ceux du PC | Vérifier les versions de sources et d'outils utilisées. |

## 9. Références

- Supports de cours *Shellcode class — Day 1* et *Asm dev 2*, Maxime Boury, ESGI.
- Documentation locale : `man nasm`, `man readelf`, `man objdump`, `man ld`.
- Contrat interne du projet : [`INTERFACE.md`](INTERFACE.md).
