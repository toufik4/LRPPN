# ARCHITECTURE GLOBALE DU PROJET TRAV - MODULE ADMGESTION

**Date**: 26 janvier 2026  
**Module**: AdmBdParpv (Paramétrage des Procédures PV)  
**Langage**: C++ / MFC

---

## 1. STACK TECHNOLOGIQUE

### Frameworks & Bibliothèques
- **MFC (Microsoft Foundation Classes)** - Framework UI Windows
- **STIC Framework** - Framework métier propriétaire
- **ORM Propriétaire** - Couche d'abstraction base de données

### Technologies
- C++ (Standard MFC)
- Win32 API
- SQL (via ORM)

---

## 2. ARCHITECTURE EN COUCHES

```
┌─────────────────────────────────────────────────┐
│        COUCHE PRÉSENTATION (UI)                 │
│  • CAdmBdParpv (Dialogue principal)             │
│  • CSticDetailDlg (Classe parente)              │
│  • CSticStdDialog (Dialogue standard)           │
├─────────────────────────────────────────────────┤
│        COUCHE CONTRÔLES MÉTIER                  │
│  • CSticEdit (Champs de saisie)                 │
│  • CKfButton (Cases à cocher)                   │
│  • CDetectButton (Boutons avec détection)       │
├─────────────────────────────────────────────────┤
│        COUCHE LOGIQUE MÉTIER                    │
│  • Validation des données                       │
│  • Règles métier (CheckOblig, CheckValidation)  │
│  • Gestion des modifications (ModifDonnee)      │
├─────────────────────────────────────────────────┤
│        COUCHE ACCÈS AUX DONNÉES                 │
│  • CData (Objets de données)                    │
│  • CTabData (Collections)                       │
│  • Thesaurus (Repository Pattern)               │
├─────────────────────────────────────────────────┤
│        COUCHE ENTITÉS (ORM)                     │
│  • Entity_ListeServicesAide                     │
│  • Entity_ListeParametres                       │
├─────────────────────────────────────────────────┤
│        BASE DE DONNÉES                          │
│  • Tables Paramètres                            │
│  • Tables Services Aide                         │
└─────────────────────────────────────────────────┘
```

---

## 3. COMPOSANTS PRINCIPAUX

### 3.1 Module CAdmBdParpv

**Responsabilités:**
- Gestion des paramètres de procédures PV
- Configuration de la numérotation automatique
- Paramétrage des services d'aide aux victimes
- Gestion de la procédure pivot

**Données Gérées:**
| Code | Description | Type |
|------|-------------|------|
| NPRO | Numérotation automatique | DTCCL_AUTC |
| CLOT | Période de clôture | DTCCL_AUTC |
| AIDV | Service d'Aide aux Victimes (SAV) | DTCCL_ADRES |
| BUAV | Bureau d'Aide aux Victimes (BAV) | DTCCL_ADRES |
| AVOC | Permanence Avocats | DTCCL_ADRES |
| AIDJ | Bureau d'Aide Juridictionnelle (BAJ) | DTCCL_ADRES |
| CIVI | Commission d'Indemnisation | DTCCL_ADRES |
| IMPR | Impression numéro procédure | DTCCL_IMPR |
| ANPV | Année procédure pivot | DTCCL_ANPV |
| NMPV | Numéro procédure pivot | DTCCL_NMPV |

### 3.2 Services d'Aide aux Victimes

**Gestion via CAdmBsAidev:**
- SAV - Service d'Aide aux Victimes
- BAV - Bureau d'Aide aux Victimes  
- AVOC - Permanence Avocats
- BAJ - Bureau d'Aide Juridictionnelle
- CIVI - Commission d'Indemnisation

---

## 4. PATTERNS DE CONCEPTION

### 4.1 MVC/MVP (Model-View-Presenter)
```
View: CAdmBdParpv (Dialog)
Model: CData + Entity_*
Presenter: Méthodes de contrôle (CheckOblig, CheckValidation)
```

### 4.2 Repository Pattern
```cpp
Thesaurus::GetThServiceAide()
Thesaurus::GetThParametres()
```

### 4.3 Data Transfer Object (DTO)
```cpp
Entity_ListeServicesAide
Entity_ListeParametres
```

### 4.4 Factory Pattern
```cpp
CreateData() // Création des objets CData
```

### 4.5 Observer Pattern
- Messages Windows (WM_ACTION, WM_KFBUTTON_KILLFOCUS)
- Communication inter-composants

---

## 5. FLUX DE DONNÉES

### 5.1 Chargement des Données
```
OnInitDialog()
    └─> CreateData()
    └─> ChargeItem()
        └─> Thesaurus::GetThParametres().SelectAll()
        └─> Thesaurus::GetThServiceAide().SelectAll()
            └─> Affichage dans les contrôles UI
```

### 5.2 Sauvegarde des Données
```
OnMajItem(ACTION_MAJ_DATA)
    └─> ModifDonnee(iFlag, sParam)
        └─> Thesaurus::GetThParametres().UpdateInsert()
            └─> Base de données
```

### 5.3 Validation
```
KillFocus Event
    └─> PostUpdateAndValidate()
        └─> CheckValidation()
            └─> Validation métier
                └─> Mise à jour UI
```

---

## 6. GESTION DES MESSAGES WINDOWS

### Messages Personnalisés
- **WM_ACTION** : Actions de mise à jour
  - ACTION_MAJ_ITEM : Rechargement des données
  - ACTION_MAJ_DATA : Sauvegarde des modifications

- **WM_KFBUTTON_KILLFOCUS** : Perte de focus des cases à cocher

- **WM_MODIF_STATUT** : Modification du statut (messages d'erreur)

---

## 7. CONTRÔLES DE VALIDATION

### 7.1 CheckValidation()
- Validation de la procédure pivot (année + numéro cohérents)
- Format du numéro de procédure (6 caractères)
- Cohérence des données liées

### 7.2 CheckOblig()
- Vérification des champs obligatoires
- Validation de la complétude de la procédure pivot
- Messages d'erreur contextuels

### 7.3 CheckRemplissage()
- Détection de saisie dans la boîte de dialogue

---

## 8. SYSTÈME DE LOGGING

```cpp
LOGLRPPN("begin") // Entrée de fonction
LOGLRPPN("end")   // Sortie de fonction
```

---

## 9. STRUCTURE DES DONNÉES

### 9.1 Tableau de Correspondance
```cpp
mTabCorresData[NPRO] = DTCCL_AUTC;
mTabCorresData[CLOT] = DTCCL_AUTC;
mTabCorresData[AIDV] = DTCCL_ADRES;
// ... etc
```

### 9.2 Entités Persistantes

**Entity_ListeParametres:**
- numAuto : Numérotation automatique
- secteur : Secteurs d'adresse
- periode : Période de clôture
- indProcPv : Indicateur impression
- anneePivot : Année pivot
- numeroPivot : Numéro pivot

**Entity_ListeServicesAide:**
- service_SAV : Service Aide Victimes
- service_BAV : Bureau Aide Victimes
- service_AVOC : Permanence Avocats
- service_BAJ : Bureau Aide Juridictionnelle
- service_CIVI : Commission Indemnisation

---

## 10. INTERACTIONS UTILISATEUR

### 10.1 Boutons F2 (Détail)
- **OnParpvAidvF2()** → Ouvre CAdmBsAidev pour SAV
- **OnParpvBuavF2()** → Ouvre CAdmBsAidev pour BAV
- **OnParpvAvocF2()** → Ouvre CAdmBsAidev pour AVOC
- **OnParpvAidjF2()** → Ouvre CAdmBsAidev pour BAJ
- **OnParpvCiviF2()** → Ouvre CAdmBsAidev pour CIVI

### 10.2 Spin Button
- **OnDeltaposParpvClotBt()** : Incrémentation/Décrémentation période (0-99)

### 10.3 Cases à Cocher
- **m_SECT_Cc** : Saisie secteurs obligatoire (O/N)
- **m_IMPR_Cc** : Impression numéro procédure (O/N)

---

## 11. RÈGLES MÉTIER

### Numérotation Automatique
- "O" : Obligatoire automatique
- "N" : Facultative

### Période de Clôture
- Valeur numérique 0-99
- Incrémentation/Décrémentation via Spin Button

### Procédure Pivot
- Année ET Numéro doivent être renseignés ensemble
- Numéro formaté sur 6 caractères

### Secteurs d'Adresse
- Activation/Désactivation via case à cocher

---

## 12. DÉPENDANCES EXTERNES

### Fichiers Inclus
```cpp
#include "stdafx.h"
#include "AdmGestion.h"
#include "AdmBdParpv.h"
#include "AdmBsAidev.h"
```

### Modules Liés
- **AdmGestion** : Application principale
- **AdmBsAidev** : Saisie détaillée services aide

---

## 13. CYCLE DE VIE DU DIALOGUE

```
Construction
    └─> CAdmBdParpv()
        └─> Initialisation pointeurs
        └─> Configuration tableaux

Initialisation
    └─> OnInitDialog()
        └─> CreateData()
        └─> SubclassDlgItem()
        └─> ChargeItem()

Utilisation
    └─> Saisie utilisateur
    └─> Validation temps réel
    └─> Messages WM_ACTION

Validation
    └─> CheckOblig()
    └─> CheckValidation()

Sauvegarde
    └─> OnMajItem()
        └─> ModifDonnee()

Destruction
    └─> ~CAdmBdParpv()
```

---

## 14. POINTS D'EXTENSION

### Ajout d'un Nouveau Paramètre
1. Ajouter constante dans enum
2. Créer CData* membre
3. Ajouter dans mTabCorresData[]
4. Implémenter dans CreateData()
5. Ajouter contrôle UI
6. Gérer dans ModifDonnee()

---

## 15. CONSIDÉRATIONS TECHNIQUES

### Gestion Mémoire
- Allocation dynamique (new/delete)
- Nettoyage dans le destructeur
- Gestion des pointeurs NULL

### Thread Safety
- Application mono-thread (MFC UI thread)
- Pas de synchronisation nécessaire

### Performance
- Chargement en bloc via SelectAll()
- Mise à jour unitaire via UpdateInsert()

---

**FIN DU DOCUMENT**
