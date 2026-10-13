# ASM1 — Projet ELF64 / assembleur x86-64

**ESGI 3A — Projet assembleur**  
**Étudiants :** Thomas Kappler et Théo Robert  
**Environnement :** Ubuntu 24.04, x86-64, NASM

## 1. Présentation

Ce projet porte sur le format des exécutables **ELF64**, leur chargement en mémoire et l'étude de la transformation d'un Program Header **`PT_NOTE → PT_LOAD`**. Il comprend un programme de démonstration, un module de manipulation ELF, et deux variantes de code assembleur permettant d'étudier les modèles de communication réseau **bind** (connexion entrante) et **reverse** (connexion sortante) dans un laboratoire isolé.

Le README décrit les fichiers remis, la compilation et l'inspection du programme de démonstration, les outils employés et les résultats communiqués. Les essais réseau sont documentés par leur objectif et leurs observations, sans instructions de déploiement d'un shell distant.

## 2. Répartition

| Membre | Partie |
|---|---|
| **Théo Robert** | Lecture et analyse ELF64, Program Headers, architecture du module de modification ELF. |
| **Thomas Kappler** | Variantes NASM bind/reverse, étude des syscalls, PIC, processus et tests en VM. |
| **En commun** | `INTERFACE.md`, tests, documentation et présentation. |

## 3. Fichiers

| Fichier | Rôle |
|---|---|
| `victime.asm` | Petit exécutable assembleur qui affiche un message de contrôle. |
| `injecteur.asm` | Module assembleur étudiant les modifications des métadonnées ELF. |
| `injecteur_reverse.asm` | Module assembleur étudiant les modifications des métadonnées ELF pour le reverse. |
| `bindshell.asm` | Variante d'étude du modèle réseau bind. |
| `reverseshell.asm` | Variante d'étude du modèle réseau reverse. |
| `INTERFACE.md` | Contrat de format entre les composants. |
| `Présentation ASM1.pptx` | Support de soutenance. |

**Note :** Les quatre principaux composants source sont écrits en **NASM x86-64**. GCC et GNU Binutils sont utilisés pour les étapes de construction et d'analyse.

## 4. Prérequis

- Linux x86-64 ; environnement utilisé : **Ubuntu 24.04**.
- **NASM**, **GCC**, **GNU Binutils** (`ld`, `objcopy`, `readelf`, `objdump`).
- Utilitaires d'inspection : **xxd**, **file** ; **GDB** si nécessaire.
- Deux VM isolées (Host-only) pour les observations réseau documentées dans le rapport.

Vérifier les outils présents :

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

### 5.1. Compiler et lancer `victime.asm`

```bash
nasm -f elf64 victime.asm -o victime.o
gcc -no-pie -nostartfiles -o victime victime.o
./victime
```

Sortie attendue :

```text
Programme original execute avec succes !
```

- `nasm -f elf64` : traduit la source NASM en **fichier objet ELF64**.
- `gcc -no-pie -nostartfiles` : réalise l'édition de liens en demandant un exécutable **non-PIE**, sans les fichiers de démarrage standards (la source définit son propre `_start`).
- `./victime` : exécute le programme de démonstration.

### 5.2. Inspecter le format ELF de la victime

```bash
file victime
readelf -h victime
readelf -l victime
readelf -S victime
objdump -d victime
```

- `file` : identifie le type du fichier.
- `readelf -h` : affiche l'**ELF Header**, notamment `Entry point address`.
- `readelf -l` : affiche les **Program Headers** (`LOAD`, `NOTE`, etc.).
- `readelf -S` : affiche les **Section Headers**.
- `objdump -d` : désassemble les instructions du programme.

Vérification ciblée utilisée durant les tests :

```bash
readelf -l victime | grep NOTE
```

Le journal communiqué montre notamment :

```text
NOTE  0x0000000000000120  0x0000000000400120  0x0000000000400120
```

**Important :** la présence d'un `PT_NOTE` dépend du binaire produit et de la configuration de l'éditeur de liens : les options de compilation ne garantissent pas à elles seules sa présence.

### 5.3. Contrôler les fichiers construits dans le laboratoire

Après leur génération dans votre environnement de travail, les fichiers peuvent être **inspectés sans être lancés** :

```bash
ls -lh victime *.o *.bin injecteur injecteur_reverse victime_infectee 2>/dev/null
file victime victime_infectee 2>/dev/null
readelf -h victime_infectee 2>/dev/null
readelf -l victime_infectee 2>/dev/null
```

Les variantes `bindshell.asm`, `reverseshell.asm` et le module `injecteur.asm` sont décrits dans le tableau des fichiers et le contrat `INTERFACE.md`. Les procédures d'intégration d'un shell distant dans un exécutable et d'accès distant ne sont pas détaillées ici.

### 5.4. Explication des commandes complémentaires

| Commande | Fonction |
|---|---|
| `ld -o <sortie> <objet>` | Effectue l'édition de liens d'un fichier objet vers un exécutable. |
| `objcopy -O binary -j .text` | Extrait les octets de la section `.text` dans un fichier binaire brut. |
| `xxd <fichier>` | Affiche le contenu binaire sous forme hexadécimale. |
| `grep` | Filtre les lignes affichées ; un `grep` sur `xxd` ne prouve pas à lui seul l'unicité d'une séquence d'octets. |


### 5.5. Vérifications ciblées des fichiers générés

Les commandes ci-dessous servent à examiner les fichiers **déjà construits** et à documenter le test. Elles ne lancent aucune connexion réseau.

```bash
# Examiner les tailles des variantes
ls -la bindshell.bin reverseshell.bin

# Afficher les octets des fichiers pour inspection
xxd bindshell.bin
xxd reverseshell.bin

# Vérifier les informations ELF avant / après dans le labo
readelf -h victime
readelf -l victime
readelf -h victime_infectee
readelf -l victime_infectee

# Vérifier le comportement du programme témoin
./victime
```

**Note :** les sorties de `xxd | grep "48 b8"` constituent des indices visuels, pas un contrôle fiable du nombre exact d'occurrences. Une commande `readelf` n'atteste que de la structure affichée, pas de la fiabilité générale du programme.

## 6. Résultats observés et limites

### Résultats relevés

| Vérification | Observation communiquée |
|---|---|
| Assemblage et exécution de `victime` | Réussite ; message affiché. |
| Présence initiale de `PT_NOTE` | Confirmée avec `readelf -l`. |
| Taille de `bindshell.bin` | **313 octets**. |
| Taille de `reverseshell.bin` | **211 octets**. |
| Création de `victime_infectee` | Observée dans le journal. |
| Exécution du fichier résultant | Message initial affiché sur le cas testé. |
| Point d'entrée et Program Headers modifiés | Observations signalées dans les essais précédents ; vérifier la correspondance avec la version finale. |
| Communication bind/reverse entre les VM | Essais rapportés par le binôme ; résultats propres à l'environnement isolé. |
| Anti-debug | Comportement rapporté lors de tests ; ne pas généraliser sans validation supplémentaire. |


