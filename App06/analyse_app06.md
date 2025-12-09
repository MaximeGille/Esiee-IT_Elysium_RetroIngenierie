# Analyse du binaire `app06`

> **Type:** ELF 32-bit LSB executable, Intel 80386  
> **Linkage:** Dynamically linked  
> **Stripped:** No  
> **Protection:** Stack canary (SSP)

---

## 1. Prologue et Épilogue

### main() - Prologue
```asm
push   ebp
mov    ebp, esp
sub    esp, 0x50              ; réserve 80 bytes
mov    eax, gs:0x14
mov    DWORD PTR [ebp-0x4], eax
xor    eax, eax
```

### main() - Épilogue
```asm
mov    eax, 0x0
mov    edx, DWORD PTR [ebp-0x4]
sub    edx, DWORD PTR gs:0x14
je     8049325
call   __stack_chk_fail
leave
ret
```

### check_username/check_password/check_totp - Prologue/Épilogue minimal
```asm
push   ebp
mov    ebp, esp
; ... comparaison ...
leave
ret
```

---

## 2. Calls et Arguments

### main()
| Adresse | Fonction | Arguments | Description |
|---------|----------|-----------|-------------|
| `0x8049247` | `printf` | `"Username? "` | Prompt username |
| `0x804925b` | `fgets` | `username`, `31`, `stdin` | Lit username |
| `0x8049268` | `printf` | `"Password? "` | Prompt password |
| `0x804927c` | `fgets` | `password`, `31`, `stdin` | Lit password |
| `0x8049289` | `printf` | `"TOTP? "` | Prompt TOTP |
| `0x804929d` | `fgets` | `totp`, `7`, `stdin` | Lit TOTP (6 chars + \0) |
| `0x80492a9` | `check_username` | `username` | Vérifie username |
| `0x80492b9` | `check_password` | `password` | Vérifie password |
| `0x80492c9` | `check_totp` | `totp` | Vérifie TOTP |
| `0x80492da` | `puts` | `"Good!"` | Succès total |
| `0x80492e9` | `puts` | `"Bad TOTP :("` | Échec TOTP |
| `0x80492f8` | `puts` | `"Bad password :("` | Échec password |
| `0x8049307` | `puts` | `"Bad username :("` | Échec username |

### Fonctions de vérification
| Fonction | Comparaison | Longueur | Valeur attendue |
|----------|-------------|----------|-----------------|
| `check_username` | `strncmp(input, "fdupont", 7)` | 7 | `fdupont` |
| `check_password` | `strncmp(input, "irh0u803YdRB", 12)` | 12 | `irh0u803YdRB` |
| `check_totp` | `strncmp(input, "817201", 6)` | 6 | `817201` |

---

## 3. Jumps et Conditions

### Structure imbriquée (3 niveaux)
```asm
; Vérification username
call   check_username
test   eax, eax
je     8049302            ; → "Bad username :("

; Vérification password (si username OK)
call   check_password
test   eax, eax
je     80492f3            ; → "Bad password :("

; Vérification TOTP (si password OK)
call   check_totp
test   eax, eax
je     80492e4            ; → "Bad TOTP :("

; Succès total
puts("Good!")
```

### Flux de contrôle
```
          ┌─────────────────┐
          │ printf(Username)│
          │ fgets(username) │
          │ printf(Password)│
          │ fgets(password) │
          │ printf(TOTP)    │
          │ fgets(totp)     │
          └────────┬────────┘
                   │
                   ▼
        check_username() == 1 ?
                   │
         ┌────NO───┴───YES────┐
         │                    │
         ▼                    ▼
  "Bad username :("   check_password() == 1 ?
                              │
                    ┌────NO───┴───YES────┐
                    │                    │
                    ▼                    ▼
            "Bad password :("    check_totp() == 1 ?
                                         │
                               ┌────NO───┴───YES────┐
                               │                    │
                               ▼                    ▼
                        "Bad TOTP :("           "Good!"
```

---

## 4. Variables locales (main)

| Offset | Taille | Usage |
|--------|--------|-------|
| `[ebp-0x44]` | 32 bytes | Buffer username |
| `[ebp-0x24]` | 32 bytes | Buffer password |
| `[ebp-0x4c]` | 8 bytes | Buffer TOTP |
| `[ebp-0x4]` | 4 bytes | Stack canary |

---

## 5. Fonctionnement général

Programme d'**authentification multi-facteurs** (MFA) avec :
- Vérification du nom d'utilisateur
- Vérification du mot de passe
- Vérification d'un code TOTP (Time-based One-Time Password)

### Code C reconstitué

```c
#include <stdio.h>
#include <string.h>

int check_username(char *input) {
    if (strncmp("fdupont", input, 7) == 0)
        return 1;
    return 0;
}

int check_password(char *input) {
    if (strncmp("irh0u803YdRB", input, 12) == 0)
        return 1;
    return 0;
}

int check_totp(char *input) {
    if (strncmp("817201", input, 6) == 0)
        return 1;
    return 0;
}

int main(int argc, char **argv) {
    char username[32];
    char password[32];
    char totp[8];
    
    printf("Username? ");
    fgets(username, 31, stdin);
    printf("Password? ");
    fgets(password, 31, stdin);
    printf("TOTP? ");
    fgets(totp, 7, stdin);
    
    if (check_username(username)) {
        if (check_password(password)) {
            if (check_totp(totp)) {
                puts("Good!");
            } else {
                puts("Bad TOTP :(");
            }
        } else {
            puts("Bad password :(");
        }
    } else {
        puts("Bad username :(");
    }
    return 0;
}
```

---

## 6. Informations extraites

| Élément | Valeur |
|---------|--------|
| **Username** | `fdupont` |
| **Password** | `irh0u803YdRB` |
| **TOTP** | `817201` |
| **Fonctions** | `main`, `check_username`, `check_password`, `check_totp` |
| **Protection** | Stack canary |

---

## 7. Évolution par rapport aux apps précédentes

| Aspect | app05 | app06 |
|--------|-------|-------|
| Inputs | 1 (password) | 3 (username, password, TOTP) |
| Vérifications | 1 fonction | 3 fonctions séparées |
| Structure | Linéaire | Imbriquée (if/else) |
| Messages d'erreur | Générique | Spécifique par champ |
| Simulation | Simple password check | Authentification MFA |
