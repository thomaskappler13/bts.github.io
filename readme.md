# ASM1 — Projet ELF64 / assembleur x86-64

**ESGI 3A — Projet assembleur**  
**Étudiants :** Thomas Kappler et Théo Robert  
**Environnement :** Ubuntu 24.04, x86-64, NASM

## 1. Présentation

Ce projet étudie les exécutables **ELF64**, leur chargement en mémoire et la transformation d'un Program Header **`PT_NOTE → PT_LOAD`**. Il comprend un programme témoin, un module expérimental de manipulation ELF et deux variantes assembleur consacrées aux modèles de communication réseau **bind** (connexion entrante) et **reverse** (connexion sortante) dans un laboratoire isolé.

Ce guide décrit les fichiers remis, le démarrage du programme témoin, les outils de compilation et les vérifications permettant de documenter les résultats de l'étude.

## 2. Répartition

| Membre | Partie |
| --- | --- |
| **Théo Robert** | Analyse ELF64, Program Headers et module de manipulation ELF. |
| **Thomas Kappler** | Variantes NASM bind/reverse, syscalls, PIC, processus et observations en VM. |
| **En commun** | `INTERFACE.md`, tests, documentation et présentation. |

## 3. Fichiers

| Fichier | Rôle |
| --- | --- |
| `victime.asm` | Programme témoin affichant un message de contrôle. |
| `injecteur.asm` | Module expérimental de transformation de métadonnées ELF. |
| `injecteur_reverse.asm` | Variante du module pour l'étude du mode reverse, lorsqu'elle est présente. |
| `bindshell.asm` | Source assembleur relative au modèle bind. |
| `reverseshell.asm` | Source assembleur relative au modèle reverse. |
| `INTERFACE.md` | Contrat entre les modules. |
| `Présentation ASM1.pptx` | Support de soutenance (nom à vérifier dans le bundle). |

Les principaux composants source utilisent **NASM x86-64**. GCC et GNU Binutils servent à la construction et à l'analyse.

## 4. Prérequis

- Ubuntu 24.04 x86-64 (environnement de référence).
- NASM, GCC et GNU Binutils (`ld`, `objcopy`, `readelf`, `objdump`).
- `xxd`, `file` et éventuellement GDB.
- Pour les observations réseau du projet : deux VM isolées sur un réseau *Host-only*.

Vérifier les outils :

```bash
nasm -v
gcc --version
ld --version
objcopy --version
readelf --version
objdump --version
file --version
```

## 5. Compiler, lancer et vérifier

### 5.1. Démarrage rapide du programme témoin

Dans un terminal Ubuntu, depuis le dossier du projet :

```bash
cd /home/thomas/esgi/ASM1
ls -la

# Produire le fichier objet
nasm -f elf64 victime.asm -o victime.o

# Lier en exécutable ELF64 non-PIE, sans fichiers de démarrage standard
gcc -no-pie -nostartfiles -o victime victime.o

# Lancer le programme témoin
./victime
```

**Sortie attendue :**

```text
Programme original execute avec succes !
```

`nasm -f elf64` produit un objet ELF64 ; `gcc -no-pie -nostartfiles` assure l'édition de liens sans PIE ni fichiers de démarrage habituels, car le programme définit son propre `_start` ; `./victime` lance l'exécutable produit.

### 5.2. Vérifier le programme ELF obtenu

```bash
file victime
readelf -h victime
readelf -l victime
readelf -S victime
objdump -d victime
```

Pour vérifier spécifiquement la présence d'une entrée `NOTE` :

```bash
readelf -l victime | grep NOTE
```

**Exemple observé lors des tests :**

```text
NOTE  0x0000000000000120  0x0000000000400120  0x0000000000400120
```

La présence de `PT_NOTE` dépend des caractéristiques de l'exécutable et de la configuration de l'éditeur de liens ; elle n'est pas garantie universellement par ces options.

### 5.3. Examiner les autres composants remis

Les fichiers `injecteur.asm`, `bindshell.asm` et `reverseshell.asm` appartiennent à l'étude expérimentale. Pour inventorier les sources et examiner les binaires déjà produits dans le laboratoire, sans les exécuter :

```bash
ls -lh *.asm *.bin *.o 2>/dev/null
file victime victime_infectee 2>/dev/null
readelf -h victime_infectee 2>/dev/null
readelf -l victime_infectee 2>/dev/null
xxd bindshell.bin 2>/dev/null
xxd reverseshell.bin 2>/dev/null
```

La procédure d'installation d'un shell distant dans un autre exécutable et son utilisation réseau ne figurent pas dans ce guide. Les tests sont consignés ci-dessous comme observations de laboratoire.

### 5.4. Comprendre les outils

| Commande | Rôle |
| --- | --- |
| `nasm -f elf64` | Assemble une source NASM en fichier objet ELF64. |
| `gcc -no-pie -nostartfiles` | Lie sans produire de PIE et sans fichiers de démarrage standard. |
| `ld -o sortie objet.o` | Effectue l'édition de liens d'un fichier objet. |
| `objcopy -O binary -j .text` | Extrait la section `.text` en octets binaires bruts. |
| `readelf -h` | Affiche l'en-tête ELF, notamment le point d'entrée. |
| `readelf -l` | Affiche les Program Headers (`LOAD`, `NOTE`, etc.). |
| `readelf -S` | Affiche les Section Headers. |
| `objdump -d` | Désassemble les instructions. |
| `xxd` | Affiche le contenu d'un fichier en hexadécimal. |
| `grep` | Filtre des lignes ; sur `xxd`, ne garantit pas le nombre d'occurrences d'une séquence. |

## 6. Résultats communiqués

| Observation | Résultat communiqué |
| --- | --- |
| Compilation et exécution de `victime` | Réussies ; message de contrôle affiché. |
| Présence initiale de `PT_NOTE` | Observée avec `readelf -l`. |
| Taille de `bindshell.bin` | **313 octets**. |
| Taille de `reverseshell.bin` | **211 octets**. |
| Construction de l'outil de manipulation ELF | Réussie dans le journal terminal. |
| Production d'un fichier `victime_infectee` | Observée dans le journal terminal. |
| Comportement du programme témoin après modification | Message initial observé sur le cas testé. |
| Point d'entrée et Program Headers | Des changements ont été observés lors d'essais antérieurs ; les valeurs dépendent du binaire utilisé. |
| Modèles bind et reverse | Essais réseau rapportés dans le laboratoire isolé à deux VM. |
| `ptrace` sous GDB | Comportement rapporté ; à interpréter selon le protocole d'essai. |

Ces observations ne garantissent pas une compatibilité avec tous les exécutables ELF64 ni avec tous les environnements réseau.

## 7. Contrat entre les modules

Le fichier **`INTERFACE.md`** détaille les responsabilités, les formats échangés, les contraintes NASM et les conventions retenues par le binôme. Il constitue la référence du projet pour la coopération entre les composants.

## 8. Limites et dépannage

**Limites connues :** cible ELF64 x86-64 dans un environnement contrôlé, support non universel des exécutables PIE, buffers de taille fixe, validation partielle des en-têtes et absence de garantie de lecture/écriture complète en une seule opération.

| Problème | Vérification utile |
| --- | --- |
| `nasm: command not found` | Vérifier l'installation de NASM (`nasm -v`). |
| Fichier source introuvable | Vérifier le dossier courant avec `pwd` et `ls -la`. |
| Échec de l'édition de liens | Relire les messages de GCC/LD et confirmer la présence de `victime.o`. |
| `Permission denied` à l'exécution | Examiner les permissions avec `ls -l victime`. |
| `readelf` signale un format invalide | Vérifier le type du fichier avec `file` et reconstruire le programme témoin. |
| Pas de `PT_NOTE` | Consigner ce résultat : il dépend du fichier produit et de l'éditeur de liens. |

## 9. Références

- Supports *Shellcode class — Day 1* et *Asm dev 2*, Maxime Boury, ESGI.
- `INTERFACE.md`, contrat d'interface du binôme.
- Documentation des outils NASM et GNU Binutils (`readelf`, `objdump`, `ld`, `objcopy`).
