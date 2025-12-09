# Analyse du binaire `app02`

> **Type:** ELF 32-bit LSB executable, Intel 80386  
> **Linkage:** Dynamically linked  
> **Stripped:** No

---

## 1. Prologue et Épilogue

### Prologue
```asm
push   ebp           ; sauvegarde ancien base pointer
mov    ebp, esp      ; nouveau frame pointer
sub    esp, 0xc      ; réserve 12 bytes (3 variables locales)
```

### Épilogue
```asm
mov    eax, 0x0      ; return 0
leave                ; esp = ebp; pop ebp
ret
```

---

## 2. Calls et Arguments

| Adresse | Fonction | Arguments | Description |
|---------|----------|-----------|-------------|
| `0x804919a` | `printf` | `"J'ai %02i ans.\n"`, `i` | Affiche l'âge formaté sur 2 chiffres |

---

## 3. Jumps et Conditions

### Structure de la boucle
```asm
mov    DWORD PTR [ebp-0xc], eax   ; i = 0
jmp    80491a6                     ; saute à la condition

; Corps de la boucle (0x8049192)
push   DWORD PTR [ebp-0xc]         ; push i
push   0x804a008                   ; push format string
call   printf
add    DWORD PTR [ebp-0xc], 0x1    ; i++

; Condition (0x80491a6)
mov    eax, DWORD PTR [ebp-0xc]    ; eax = i
cmp    eax, DWORD PTR [ebp-0x4]    ; compare i avec 67
jl     8049192                      ; si i < 67 → continue la boucle
```

### Flux de contrôle
```
i = 0
    │
    ▼
┌──► i < 67 ? ───NON──► fin
│       │
│      OUI
│       │
│       ▼
│   printf("J'ai %02i ans.\n", i)
│       │
│       ▼
│     i++
└───────┘
```

---

## 4. Variables locales

| Offset | Valeur initiale | Rôle |
|--------|-----------------|------|
| `[ebp-0x4]` | `0x43` (67) | Limite de la boucle |
| `[ebp-0x8]` | `0x0` | Valeur de départ |
| `[ebp-0xc]` | `0x0` | Compteur `i` |

---

## 5. Fonctionnement général

Programme qui affiche une **boucle d'âges** de 0 à 66 ans.

### Code C reconstitué

```c
#include <stdio.h>

int main() {
    int start = 0;
    int end = 67;
    for (int i = start; i < end; i++) {
        printf("J'ai %02i ans.\n", i);
    }
    return 0;
}
```

### Sortie attendue
```
J'ai 00 ans.
J'ai 01 ans.
J'ai 02 ans.
...
J'ai 66 ans.
```

---

## 6. Informations extraites

| Élément | Valeur |
|---------|--------|
| **Type de boucle** | `for` |
| **Début** | 0 |
| **Fin** | 66 (< 67) |
| **Itérations** | 67 |
| **Format** | `%02i` (entier sur 2 chiffres) |
