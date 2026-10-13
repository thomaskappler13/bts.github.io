# ASM1 — Guide utilisateur ELF64 / Assembleur x86-64

**ESGI 3A — Projet assembleur**  
**Étudiants :** Thomas Kappler et Théo Robert  
**Environnement :** Ubuntu 24.04, x86-64, NASM

---

## 1. Présentation du projet

Le projet ASM1 porte sur l'étude des exécutables ELF64, leur chargement en mémoire et la transformation d'un Program Header `PT_NOTE → PT_LOAD`.

Il comprend :

- Un programme témoin : `victime.asm`.
- Un module expérimental de manipulation ELF : `injecteur.asm`.
- Une variante du module : `injecteur_reverse.asm`.
- Deux programmes assembleur dédiés aux modèles réseau bind et reverse : `bindshell.asm` et `reverseshell.asm`.
- Un fichier `INTERFACE.md` définissant le contrat entre les modules.

Ce guide explique comment préparer l'environnement, compiler les fichiers, exécuter le programme témoin, analyser les exécutables et vérifier les communications réseau.

## 2. Répartition du travail

| Étudiant | Responsabilités |
|---|---|
| Théo Robert | Analyse ELF64, Program Headers et manipulation des métadonnées ELF. |
| Thomas Kappler | NASM, bind/reverse, syscalls, PIC, processus et observations réseau. |
| En commun | Tests, documentation, interface entre modules et soutenance. |

## 3. Prérequis

Le projet utilise Ubuntu 24.04 sur une architecture x86-64.

Installer les outils :

```bash
sudo apt update
sudo apt install -y nasm gcc binutils xxd file gdb netcat-openbsd iproute2 iputils-ping psmisc
```

Vérifier les installations :

```bash
nasm -v
gcc --version
ld --version
objcopy --version
readelf --version
objdump --version
file --version
nc -h
```

Pour les tests réseau, prévoir deux machines virtuelles Ubuntu sur un réseau Host-only.

## 4. Accéder au projet

Ouvrir un terminal dans le dossier du projet puis vérifier les fichiers disponibles :

```bash
pwd
ls -la
```

Si nécessaire, accéder au dossier :

```bash
cd /home/thomas/esgi/ASM1
```

Arborescence attendue :

```text
ASM1/
├── victime.asm
├── injecteur.asm
├── injecteur_reverse.asm
├── bindshell.asm
├── reverseshell.asm
└── INTERFACE.md
```

Certains fichiers binaires et objets seront générés pendant la compilation.

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

### 5.3. Vérification du fichier ELF64

```bash
file victime
readelf -h victime
readelf -l victime
readelf -S victime
objdump -d victime
```

Afficher uniquement le point d'entrée :

```bash
readelf -h victime | grep -Ei 'entry|entrée'
```

Vérifier la présence du segment `PT_NOTE` :

```bash
readelf -l victime | grep NOTE
```

Exemple de résultat :

```text
NOTE  0x0000000000000120  0x0000000000400120  0x0000000000400120
```

Les valeurs dépendent de l'exécutable généré et des options de compilation.

## 6. Compiler les composants du modèle bind

### 6.1. Compilation du programme assembleur

```bash
nasm -f elf64 bindshell.asm -o bindshell.o
objcopy -O binary -j .text bindshell.o bindshell.bin
```

Fichiers générés :

- `bindshell.o` : fichier objet ELF64.
- `bindshell.bin` : contenu binaire brut de la section `.text`.

### 6.2. Compilation du module ELF

```bash
nasm -f elf64 injecteur.asm -o injecteur.o
ld -o injecteur injecteur.o
```

### 6.3. Vérification des fichiers

Afficher les fichiers générés :

```bash
ls -lh bindshell.o bindshell.bin injecteur
```

Examiner les premiers octets :

```bash
xxd bindshell.bin | head -20
```

Examiner les derniers octets :

```bash
xxd bindshell.bin | tail -3
```

Rechercher visuellement la séquence `48 b8` :

```bash
xxd -g 1 bindshell.bin | grep -i '48 b8'
```

Cette séquence peut correspondre au début d'une instruction `mov rax, imm64` en x86-64. La recherche sur la sortie de `xxd` est indicative : les octets recherchés peuvent se trouver à cheval sur deux lignes.

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
nasm -f elf64 reverseshell.asm -o reverseshell.o
objcopy -O binary -j .text reverseshell.o reverseshell.bin
```

### 7.2. Compilation du module reverse

Si `injecteur_reverse.asm` est déjà présent :

```bash
nasm -f elf64 injecteur_reverse.asm -o injecteur_reverse.o
ld -o injecteur_reverse injecteur_reverse.o
```

Si le fichier est absent, il doit être préparé conformément à l'interface du projet. Une simple substitution de nom de fichier ne garantit pas la compatibilité des deux variantes.

### 7.3. Vérifications

Afficher les fichiers produits :

```bash
ls -lh reverseshell.o reverseshell.bin
```

Afficher les premiers et les derniers octets :

```bash
xxd reverseshell.bin | head -20
xxd reverseshell.bin | tail -3
```

Rechercher la séquence `48 b8` :

```bash
xxd -g 1 reverseshell.bin | grep -i '48 b8'
```

Désassembler les instructions :

```bash
objdump -d -M intel reverseshell.o
```

Examiner une instruction de configuration réseau présente dans le source, si elle existe :

```bash
grep -F 'mov dword [rsp+4]' reverseshell.asm
```

Cette commande affiche uniquement l'instruction trouvée. Elle ne valide pas la configuration réseau complète.

## 8. Étudier les modifications ELF64

### 8.1. Vérifier le point d'entrée initial

```bash
readelf -h victime | grep -Ei 'entry|entrée'
```

### 8.2. Examiner les Program Headers

```bash
readelf -l victime
```

Filtrer les segments `LOAD` et `NOTE` :

```bash
readelf -l victime | grep -E 'LOAD|NOTE'
```

### 8.3. Comparer deux exécutables ELF

Lorsqu'un exécutable expérimental est disponible, comparer ses métadonnées avec celles du programme témoin.

Pour le programme original :

```bash
readelf -h victime
readelf -l victime
```

Pour un fichier de comparaison nommé `victime_modifiee` :

```bash
readelf -h victime_modifiee
readelf -l victime_modifiee
```

Afficher uniquement les points d'entrée :

```bash
readelf -h victime | grep -Ei 'entry|entrée'
readelf -h victime_modifiee | grep -Ei 'entry|entrée'
```

Comparer les types de segments :

```bash
readelf -l victime | grep -E 'LOAD|NOTE'
readelf -l victime_modifiee | grep -E 'LOAD|NOTE'
```

Comparer la taille des fichiers :

```bash
stat -c '%s %n' victime victime_modifiee
```

Désassembler les deux exécutables :

```bash
objdump -d -M intel victime
objdump -d -M intel victime_modifiee
```

Les éléments à comparer :

| Élément | Description |
|---|---|
| `e_entry` | Adresse du point d'entrée. |
| `p_type` | Type du segment. |
| `p_offset` | Position dans le fichier. |
| `p_vaddr` | Adresse virtuelle. |
| `p_filesz` | Taille dans le fichier. |
| `p_memsz` | Taille en mémoire. |
| `p_flags` | Permissions. |
| `p_align` | Alignement. |

La disparition d'un segment `NOTE` ou l'apparition d'un segment `LOAD` doit être confirmée avec les résultats réels de `readelf`.

### 8.4. Vérifier le résultat de la compilation

Contrôler le code de retour de chaque commande immédiatement après son exécution :

```bash
echo "Code de retour : $?"
```

Un code `0` indique généralement que la commande précédente s'est terminée avec succès. Il ne garantit pas que le programme compilé fonctionnera correctement.

## 9. Vérifier les communications réseau

Les observations sont réalisées entre deux VM Ubuntu sur un réseau Host-only.

- VM1 : machine hébergeant le programme étudié.
- VM2 : machine permettant les observations réseau.

### 9.1. Vérifier les adresses IP

Sur chaque VM :

```bash
ip -br addr
```

Relever l'adresse IP de l'interface Host-only.

### 9.2. Vérifier la connectivité entre les VM

Depuis VM1 :

```bash
ping -c 4 IP_VM2
```

Depuis VM2 :

```bash
ping -c 4 IP_VM1
```

Remplacer les noms par les adresses IP réelles.

### 9.3. Vérifier les ports TCP en écoute

```bash
ss -tlnp
```

Filtrer sur le port étudié :

```bash
ss -tlnp | grep 4444
```

Si aucun service n'écoute sur ce port, la commande ne retourne aucune ligne.

### 9.4. Vérifier les connexions établies

```bash
ss -tn
```

Cette commande affiche les connexions TCP et leur état.

Elle permet notamment d'observer les états `ESTAB`, `SYN-SENT` ou `TIME-WAIT`, selon les communications présentes.

### 9.5. Arrêter les processus de test

Pour interrompre un programme lancé au premier plan, utiliser :

```text
Ctrl+C
```

Pour rechercher un processus par son nom :

```bash
ps aux | grep '[n]om_du_programme'
```

Pour arrêter un processus dont le PID est connu :

```bash
kill PID
```

Pour arrêter une instance de Netcat au premier plan, utiliser également `Ctrl+C`.

En présence de plusieurs processus portant le même nom, vérifier leur rôle avant de les arrêter.

## 10. Workflow de vérification

Cette procédure rassemble les principales commandes de compilation et d'inspection dans un ordre logique.

### Étape 1 — Compiler le programme témoin

```bash
nasm -f elf64 victime.asm -o victime.o
gcc -no-pie -nostartfiles -o victime victime.o
./victime
```

### Étape 2 — Compiler la variante bind

```bash
nasm -f elf64 bindshell.asm -o bindshell.o
objcopy -O binary -j .text bindshell.o bindshell.bin
```

### Étape 3 — Compiler la variante reverse

```bash
nasm -f elf64 reverseshell.asm -o reverseshell.o
objcopy -O binary -j .text reverseshell.o reverseshell.bin
```

### Étape 4 — Compiler le module ELF

```bash
nasm -f elf64 injecteur.asm -o injecteur.o
ld -o injecteur injecteur.o
```

### Étape 5 — Vérifier les résultats

```bash
file victime
readelf -h victime
readelf -l victime
objdump -d -M intel victime
```

Puis :

```bash
ls -lh *.o *.bin
```

### Étape 6 — Vérifier le réseau des VM

```bash
ip -br addr
ss -tlnp
ss -tn
```

Le guide distingue ainsi la compilation, l'analyse des exécutables et les observations réseau.

## 11. Nettoyage du projet

Supprimer les fichiers objets :

```bash
rm -f victime.o bindshell.o reverseshell.o
rm -f injecteur.o injecteur_reverse.o
```

Supprimer les binaires bruts :

```bash
rm -f bindshell.bin reverseshell.bin
```

Les fichiers sources `.asm` et la documentation sont conservés.

## 12. Récapitulatif

| Action | Commande |
|---|---|
| Compiler un source NASM | `nasm -f elf64 fichier.asm -o fichier.o` |
| Lier la victime | `gcc -no-pie -nostartfiles -o victime victime.o` |
| Lier le module ELF | `ld -o injecteur injecteur.o` |
| Lancer le programme témoin | `./victime` |
| Extraire la section `.text` | `objcopy -O binary -j .text fichier.o fichier.bin` |
| Afficher l'en-tête ELF | `readelf -h fichier` |
| Afficher les Program Headers | `readelf -l fichier` |
| Afficher les Section Headers | `readelf -S fichier` |
| Désassembler | `objdump -d -M intel fichier` |
| Examiner les octets | `xxd fichier.bin` |
| Comparer les tailles | `stat -c '%s %n' fichier1 fichier2` |
| Identifier les interfaces | `ip -br addr` |
| Tester la connectivité | `ping -c 4 IP` |
| Afficher les ports en écoute | `ss -tlnp` |
| Afficher les connexions TCP | `ss -tn` |
| Rechercher un processus | `ps aux` |
| Arrêter un processus | `kill PID` |

## 13. Conclusion

Ce guide utilisateur permet de préparer l'environnement ASM1, de compiler les sources assembleur et d'inspecter les exécutables ELF64.

Il fournit également les commandes nécessaires à la comparaison des métadonnées ELF, à la vérification de la connectivité entre les machines virtuelles et au nettoyage de l'environnement de travail.

Les résultats présentés pendant la soutenance doivent provenir des essais réellement effectués sur les machines virtuelles du projet.
