# ASM1 — Guide utilisateur ELF64 / Assembleur x86-64

**ESGI 3A — Projet assembleur**  
**Étudiants :** Thomas Kappler et Théo Robert  
**Environnement :** Ubuntu 24.04, x86-64, NASM

---

## 1. Présentation du projet

Le projet ASM1 porte sur l'étude des exécutables ELF64, de leur chargement en mémoire et de la transformation d'un Program Header `PT_NOTE → PT_LOAD`.

Il est composé de plusieurs éléments :

- Un programme témoin nommé `victime.asm`.
- Un module expérimental de manipulation ELF nommé `injecteur.asm`.
- Une variante du module nommée `injecteur_reverse.asm`.
- Deux programmes assembleur consacrés aux communications réseau : `bindshell.asm` et `reverseshell.asm`.
- Un fichier `INTERFACE.md` définissant les échanges entre les modules.

Ce guide explique comment préparer l'environnement, compiler les fichiers, lancer le programme témoin et effectuer les vérifications techniques nécessaires.

## 2. Répartition du travail

| Étudiant | Responsabilités |
|---|---|
| Théo Robert | Analyse ELF64, Program Headers, manipulation des métadonnées ELF. |
| Thomas Kappler | Assembleur NASM, modèles bind et reverse, syscalls, PIC, processus et observations réseau. |
| En commun | Tests, documentation, interface entre modules et soutenance. |

## 3. Prérequis

Le projet utilise Ubuntu 24.04 sur une architecture x86-64.

Installer les outils nécessaires :

```bash
sudo apt update
sudo apt install -y nasm gcc binutils xxd file gdb netcat-openbsd iproute2 iputils-ping
```

Vérifier leur installation :

```bash
nasm -v
gcc --version
ld --version
objcopy --version
readelf --version
objdump --version
file --version
```

Pour les observations réseau, utiliser deux VM Ubuntu configurées sur un réseau Host-only.

## 4. Accéder au projet

Ouvrir un terminal Ubuntu et accéder au répertoire :

```bash
cd /home/thomas/esgi/ASM1
```

Afficher les fichiers disponibles :

```bash
ls -la
```

Les fichiers attendus sont :

```text
ASM1/
├── victime.asm
├── injecteur.asm
├── injecteur_reverse.asm
├── bindshell.asm
├── reverseshell.asm
└── INTERFACE.md
```

Le fichier `injecteur_reverse.asm` peut être absent du dossier initial.

## 5. Compiler et exécuter le programme témoin

### 5.1. Compilation

```bash
nasm -f elf64 victime.asm -o victime.o
gcc -no-pie -nostartfiles -o victime victime.o
```

### 5.2. Exécution

```bash
./victime
```

Sortie attendue :

```text
Programme original execute avec succes !
```

### 5.3. Vérification ELF64

```bash
file victime
readelf -h victime
readelf -l victime
readelf -S victime
objdump -d victime
```

Vérifier la présence du segment `PT_NOTE` :

```bash
readelf -l victime | grep NOTE
```

Exemple de résultat observé :

```text
NOTE  0x0000000000000120  0x0000000000400120  0x0000000000400120
```

Le résultat peut varier selon les paramètres de compilation.

## 6. Compiler les composants du modèle bind

### 6.1. Compilation du programme assembleur

Depuis le dossier du projet :

```bash
cd /home/thomas/esgi/ASM1

nasm -f elf64 bindshell.asm -o bindshell.o
objcopy -O binary -j .text bindshell.o bindshell.bin
```

Le fichier `bindshell.o` est un fichier objet ELF64. Le fichier `bindshell.bin` contient les octets bruts de la section `.text`.

### 6.2. Compilation du module ELF

```bash
nasm -f elf64 injecteur.asm -o injecteur.o
ld -o injecteur injecteur.o
```

### 6.3. Vérifications

Contrôler les fichiers générés :

```bash
ls -lh bindshell.o bindshell.bin injecteur
```

Examiner le fichier binaire :

```bash
xxd bindshell.bin | head -20
```

Rechercher visuellement une séquence particulière :

```bash
xxd bindshell.bin | grep "48 b8"
```

Cette dernière commande sert uniquement au repérage visuel et ne garantit pas le comptage exact des occurrences.

Désassembler le fichier objet :

```bash
objdump -d -M intel bindshell.o
```

Inspecter le module ELF :

```bash
readelf -h injecteur
objdump -d -M intel injecteur
```

## 7. Compiler les composants du modèle reverse

### 7.1. Compilation du programme assembleur

```bash
cd /home/thomas/esgi/ASM1

nasm -f elf64 reverseshell.asm -o reverseshell.o
objcopy -O binary -j .text reverseshell.o reverseshell.bin
```

### 7.2. Compilation du module reverse

Si le fichier `injecteur_reverse.asm` est déjà présent :

```bash
nasm -f elf64 injecteur_reverse.asm -o injecteur_reverse.o
ld -o injecteur_reverse injecteur_reverse.o
```

Si le projet prévoit une variante construite à partir d'un fichier commun, vérifier que les références et les paramètres utilisés correspondent bien au module reverse avant de le compiler.

### 7.3. Vérifications

```bash
ls -lh reverseshell.o reverseshell.bin
```

Afficher les octets :

```bash
xxd reverseshell.bin | head -20
```

Rechercher la séquence utilisée lors de l'analyse :

```bash
xxd reverseshell.bin | grep "48 b8"
```

Inspecter les instructions :

```bash
objdump -d -M intel reverseshell.o
```

Vérifier une instruction réseau présente dans le code source, si elle existe :

```bash
grep -F 'mov dword [rsp+4]' reverseshell.asm
```

Cette commande permet de consulter le paramètre défini dans le fichier source. Elle ne garantit pas à elle seule la validité de la configuration réseau.

## 8. Étudier les modifications ELF64

Les deux modules expérimentaux s'appuient sur l'étude de la structure ELF64.

### 8.1. Vérifier le point d'entrée initial

```bash
readelf -h victime | grep -i 'entry'
```

Sur un système en français, utiliser :

```bash
readelf -h victime | grep -i 'entrée'
```

### 8.2. Examiner les Program Headers

```bash
readelf -l victime
```

Afficher uniquement les segments `LOAD` et `NOTE` :

```bash
readelf -l victime | grep -E "LOAD|NOTE"
```

### 8.3. Comparer deux exécutables ELF

Lorsqu'un fichier expérimental distinct a été produit et validé dans le laboratoire, comparer ses métadonnées avec le programme témoin.

```bash
readelf -h victime
readelf -l victime
```

Puis effectuer les mêmes commandes sur le fichier expérimental concerné.

Les éléments à comparer sont :

| Élément | Vérification |
|---|---|
| `e_entry` | Adresse du point d'entrée. |
| `p_type` | Type des segments ELF. |
| `p_offset` | Décalage dans le fichier. |
| `p_vaddr` | Adresse virtuelle. |
| `p_filesz` | Taille du segment dans le fichier. |
| `p_memsz` | Taille en mémoire. |
| `p_flags` | Permissions. |
| `p_align` | Alignement. |

Une transformation `PT_NOTE → PT_LOAD` doit être analysée à partir des valeurs effectivement obtenues. La présence de quatre segments `LOAD`, par exemple, n'est pas systématique.

## 9. Vérifier les communications réseau

Les variantes bind et reverse sont étudiées dans un laboratoire composé de deux machines virtuelles isolées.

- **VM1 :** machine hébergeant le programme étudié.
- **VM2 :** machine servant à observer les communications.

### 9.1. Vérifier les adresses IP

Sur VM1 :

```bash
ip -br addr
```

Sur VM2 :

```bash
ip -br addr
```

Relever les adresses IP des interfaces Host-only.

### 9.2. Vérifier la connectivité entre les VM

Depuis VM1 :

```bash
ping -c 4 IP_VM2
```

Depuis VM2 :

```bash
ping -c 4 IP_VM1
```

Remplacer `IP_VM1` et `IP_VM2` par les adresses réelles.

### 9.3. Vérifier les ports en écoute

```bash
ss -tlnp
```

Pour rechercher un port utilisé pendant les observations :

```bash
ss -tlnp | grep 4444
```

Si aucun processus n'écoute sur ce port, la commande ne renvoie aucune ligne.

### 9.4. Vérifier les connexions établies

```bash
ss -tn
```

Ces observations permettent d'identifier le sens de communication, les adresses utilisées et l'état des connexions TCP.

Pour la soutenance, documenter les différences entre une connexion entrante (bind) et une connexion sortante (reverse), à l'aide d'un échange de messages de test sans interface de commande distante.

## 10. Nettoyage et nouvelle compilation

Pour supprimer les fichiers intermédiaires générés pendant la compilation :

```bash
rm -f victime.o bindshell.o reverseshell.o
rm -f injecteur.o injecteur_reverse.o
rm -f bindshell.bin reverseshell.bin
```

Les fichiers `.asm` et `INTERFACE.md` restent conservés.

Il est ensuite possible de relancer les étapes de compilation des sections précédentes.

## 11. Récapitulatif des commandes

| Action | Commande |
|---|---|
| Ouvrir le projet | `cd /home/thomas/esgi/ASM1` |
| Compiler le programme témoin | `nasm -f elf64 victime.asm -o victime.o` |
| Lier le programme témoin | `gcc -no-pie -nostartfiles -o victime victime.o` |
| Lancer le témoin | `./victime` |
| Compiler bind | `nasm -f elf64 bindshell.asm -o bindshell.o` |
| Compiler reverse | `nasm -f elf64 reverseshell.asm -o reverseshell.o` |
| Produire le binaire brut | `objcopy -O binary -j .text fichier.o fichier.bin` |
| Compiler le module ELF | `nasm -f elf64 injecteur.asm -o injecteur.o` |
| Lier le module ELF | `ld -o injecteur injecteur.o` |
| Vérifier les Program Headers | `readelf -l victime` |
| Vérifier le point d'entrée | `readelf -h victime` |
| Examiner les octets | `xxd fichier.bin` |
| Désassembler | `objdump -d -M intel fichier.o` |
| Afficher les interfaces réseau | `ip -br addr` |
| Vérifier les ports TCP | `ss -tlnp` |

## 12. Conclusion

Ce guide permet de reproduire les principales étapes de préparation et d'analyse du projet ASM1 sur Ubuntu 24.04.

Il couvre la compilation du programme témoin, des variantes bind et reverse, l'inspection des fichiers ELF64 et l'observation de communications entre deux machines virtuelles.

La validation finale du projet doit s'appuyer sur les résultats réellement obtenus lors des tests, notamment les informations fournies par `readelf`, `objdump`, `xxd` et les outils de diagnostic réseau.
