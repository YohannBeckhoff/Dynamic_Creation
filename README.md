---
title: TwinCAT 3 – Instanciation dynamique de blocs fonction (__NEW / __DELETE)
description: Exemple TwinCAT 3 montrant la création et la destruction dynamiques d'instances de FB avec __NEW et __DELETE, mono-instance et multi-instances (jusqu'à 1000), avec surveillance de la mémoire du router.
author: Yohann
date: 2026-11-02
version: 1.0.0
language: fr
tags:
  - TwinCAT3
  - IEC-61131-3
  - Structured-Text
  - __NEW
  - __DELETE
  - dynamic-creation
  - Beckhoff
environment: TwinCAT PLC Control 3.5.21.61
library: Tc2_Utilities
---

# TwinCAT 3 – Instanciation dynamique avec `__NEW` et `__DELETE`

Projet d'exemple TwinCAT 3 illustrant la **création** et la **destruction dynamiques** d'instances de bloc fonction (FB) pendant l'exécution du PLC, avec les opérateurs `__NEW` et `__DELETE`.

Deux cas sont présentés :

| Programme | Objectif |
|---|---|
| `monoInstance` | Créer / détruire **une seule** instance de `FB_Dynamic` |
| `multiInstances` | Gérer un **nombre dynamique** d'instances (de 0 à `cMaxInstances`), ajustable en ligne, avec suivi de la mémoire du router |

---

## Contenu du projet

| Fichier / POU | Type | Rôle |
|---|---|---|
| `FB_Dynamic.xml` | PLCopen XML | Projet à importer dans TwinCAT (menu *Projet > Importer PLCopenXML*) |
| `FB_Dynamic` | Function Block | Bloc instancié dynamiquement (attribut `enable_dynamic_creation`) |
| `FB_Dynamic.Increase` | Méthode | Incrémente le compteur interne et retourne sa valeur |
| `MAIN` | Programme | Appelle `monoInstance` ou `multiInstances` |
| `monoInstance` | Programme | Exemple avec un seul pointeur |
| `multiInstances` | Programme | Exemple avec un tableau de pointeurs |

## Prérequis

- TwinCAT 3 (XAE) – projet généré avec TwinCAT PLC Control 3.5.21.61
- Bibliothèque `Tc2_Utilities` (pour `FB_GetRouterStatusInfo` et `ST_TcRouterStatusInfo`)

## Import

1. Ouvrir un projet PLC TwinCAT 3.
2. Clic droit sur le projet PLC, puis *Add > Existing Item* ou *Projet > Importer PLCopenXML*, et sélectionner `FB_Dynamic.xml`.
3. Vérifier la référence à `Tc2_Utilities`.
4. Compiler, activer la configuration, puis passer en Run.

---

## 1. Le bloc fonction `FB_Dynamic`

Pour pouvoir être créé avec `__NEW`, le FB **doit** porter l'attribut `enable_dynamic_creation`.

```iecst
{attribute 'enable_dynamic_creation'}
FUNCTION_BLOCK FB_Dynamic
VAR
    nCounts : INT;
END_VAR
```

Méthode `Increase` :

```iecst
METHOD Increase : INT
VAR_INPUT
    nCountStep : INT;
END_VAR

nCounts  := nCounts + nCountStep;
Increase := nCounts;
```

## 2. Programme `MAIN`

```iecst
//monoInstance();
multiInstances();
```

Il suffit de commenter / décommenter l'appel voulu.

---

## 3. `monoInstance` – une seule instance

Principe : un pointeur `pFB : POINTER TO FB_Dynamic`.

- `__NEW(FB_Dynamic)` alloue l'instance et retourne son adresse (0 en cas d'échec).
- `__DELETE(pFB)` libère la mémoire.

```iecst
IF bNew AND (pFB = 0) THEN
    pFB  := __NEW(FB_Dynamic);
    bNew := FALSE;
END_IF

IF (pFB <> 0) THEN
    nResult := pFB^.Increase(nCountStep := nIncrease);
END_IF

IF bDelete THEN
    __DELETE(pFB);
    bDelete := FALSE;
END_IF
```

> **Attention** : dans cette version, `pFB` n'est pas remis à 0 après `__DELETE`. Au cycle suivant, `pFB <> 0` reste vrai et `Increase` est appelé sur une instance détruite (accès mémoire invalide). Version corrigée :
>
> ```iecst
> IF bDelete AND (pFB <> 0) THEN
>     __DELETE(pFB);
>     pFB     := 0;      // indispensable
>     bDelete := FALSE;
> END_IF
> ```

---

## 4. `multiInstances` – nombre dynamique d'instances

### Principe

- Un **tableau de pointeurs** `aFB : ARRAY[1..cMaxInstances] OF POINTER TO FB_Dynamic` : aucune instance n'est créée statiquement.
- Le nombre d'instances souhaité est piloté par `nTarget` (modifiable en ligne).
- À chaque cycle, le programme **réconcilie** l'état réel avec `nTarget` :
  - `i <= nTarget` et pointeur à 0 → `__NEW`
  - `i > nTarget` et pointeur non nul → `__DELETE` puis pointeur remis à 0
- `nMaxOpsPerCycle` limite le nombre de créations / suppressions par cycle afin d'**étaler la charge** et d'éviter un pic sur le temps de cycle.
- Si `__NEW` retourne 0 (mémoire épuisée), `nNewFailed` est incrémenté et `bAllocError` passe à TRUE.

### Variables principales

| Variable | Type | Rôle |
|---|---|---|
| `cMaxInstances` | `UINT` (constante) | Taille du tableau (1000 dans le projet, à adapter : 10, 100, 1000...) |
| `nTarget` | `UINT` | Nombre d'instances souhaité |
| `nMaxOpsPerCycle` | `UINT` | Nombre max de `__NEW` / `__DELETE` par cycle |
| `nIncrease` | `INT` | Pas d'incrémentation passé à `Increase` |
| `nActive` | `UINT` | Nombre d'instances actuellement créées |
| `nNewFailed` | `UDINT` | Nombre d'échecs de `__NEW` |
| `bAllocError` | `BOOL` | Indique qu'une allocation a échoué |
| `nUsage` | `REAL` | Taux d'occupation de la mémoire du router (%) |

### Code

```iecst
// --- 1. Ajustement du nombre d'instances vers nTarget ---
nTarget := LIMIT(0, nTarget, cMaxInstances);
nOps    := 0;

FOR i := 1 TO cMaxInstances DO
    IF nOps >= nMaxOpsPerCycle THEN
        EXIT;                              // on étale la charge sur plusieurs cycles
    END_IF

    IF i <= nTarget THEN
        // Instance souhaitée : on la crée si elle n'existe pas
        IF aFB[i] = 0 THEN
            aFB[i] := __NEW(FB_Dynamic);
            nOps := nOps + 1;
            IF aFB[i] = 0 THEN             // __NEW renvoie 0 si plus de mémoire
                nNewFailed  := nNewFailed + 1;
                bAllocError := TRUE;
                EXIT;
            END_IF
        END_IF
    ELSE
        // Instance en trop : on la détruit
        IF aFB[i] <> 0 THEN
            __DELETE(aFB[i]);
            aFB[i] := 0;                   // indispensable
            nOps := nOps + 1;
        END_IF
    END_IF
END_FOR

// --- 2. Appel cyclique des instances existantes ---
nActive := 0;
FOR i := 1 TO cMaxInstances DO
    IF aFB[i] <> 0 THEN
        aResult[i] := aFB[i]^.Increase(nCountStep := nIncrease);
        nActive    := nActive + 1;
    END_IF
END_FOR

// --- 3. Surveillance du router ---
fbGetRouterStatusInfo(
    sNetId   := '',
    bExecute := NOT TonGetRouterStatus.Q,
    tTimeout := DEFAULT_ADS_TIMEOUT,
    bBusy    => ,
    bError   => bError,
    nErrID   => nErrorID,
    info     => stTcRouterStatusInfo);

TonGetRouterStatus(IN := NOT fbGetRouterStatusInfo.bBusy, PT := T#1S);

nUsage := (TO_REAL(stTcRouterStatusInfo.maxMem - stTcRouterStatusInfo.maxMemAvail)
           / TO_REAL(stTcRouterStatusInfo.maxMem)) * 100.0;
```

### Utilisation

1. Passer en Run et ouvrir `multiInstances` en ligne.
2. Modifier `nTarget` (par exemple 10, 100, 1000) puis écrire la valeur.
3. Observer `nActive`, `nUsage`, `nNewFailed` et `bAllocError`.
4. Repasser `nTarget` à 0 pour libérer toutes les instances.

---

## 5. Tester les limites du système

Les instances dynamiques sont allouées dans la **mémoire du router TwinCAT**. Pour explorer les limites :

| Point à observer | Comment |
|---|---|
| Mémoire consommée | `nUsage` (calculé depuis `ST_TcRouterStatusInfo`) |
| Échec d'allocation | `nNewFailed`, `bAllocError` |
| Impact sur le temps de cycle | Temps d'exécution de la tâche PLC, avec différentes valeurs de `nMaxOpsPerCycle` |
| Impact du nombre d'instances | Comparer 10 / 100 / 1000 instances |

Tableau de résultats à compléter :

| `cMaxInstances` / `nTarget` | `nActive` | `nUsage` (%) | Temps d'exécution tâche | `bAllocError` |
|---|---|---|---|---|
| 10 | | | | |
| 100 | | | | |
| 1000 | | | | |

## 6. Points d'attention

- `{attribute 'enable_dynamic_creation'}` est **obligatoire** sur le FB à instancier avec `__NEW`.
- Toujours tester le retour de `__NEW` (0 = échec).
- Toujours remettre le pointeur à 0 après `__DELETE`.
- Les instances dynamiques sont perdues lors d'un Reset ou d'un Download du PLC.
- Pour de grands nombres d'instances, étaler les créations et suppressions sur plusieurs cycles (`nMaxOpsPerCycle`).

