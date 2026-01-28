

# ARCHITECTURE DES EXÉCUTABLES - PROJET LRPPN/TRAV

**Date**: 28 janvier 2026  
**Projet**: Client Lourd LRPPN (C++ / MFC)  
**Workspace**: c:\clients\client-cpp\Trav

---

## TABLE DES MATIÈRES

1. [Applications Principales](#1-applications-principales)
2. [Applications Administration](#2-applications-administration)
3. [Utilitaires](#3-utilitaires)
4. [DLL Partagées](#4-dll-partagées)
5. [Bibliothèques Statiques](#5-bibliothèques-statiques)
6. [Orchestration des Applications](#6-orchestration-des-applications)
7. [Architecture Client Lourd](#7-architecture-client-lourd)
8. [Lancement Inter-Applications](#8-lancement-inter-applications)
9. [Dépendances de Chargement](#9-dépendances-de-chargement)
10. [Communication Applicative](#10-communication-applicative)
11. [Initialisation Type](#11-initialisation-type)
12. [Déploiement](#12-déploiement)

---

## 1. APPLICATIONS PRINCIPALES

### Exécutables Métier (.exe)

| Exécutable | Module | Rôle | ConfigurationType |
|------------|--------|------|-------------------|
| **RDPr.exe** | RDP | Rédaction de Procédure (Application principale) | Application |
| **MAN.exe** | MAN | Gestion de la Main Courante | Application |
| **RCJ.exe** | RCJ | Réquisitions Civiles Judiciaires | Application |
| **RPJ.exe** | RPJ | Réquisitions Parquet Judiciaire | Application |
| **GPJ.exe** | GPJ | Gestion Projet Judiciaire | Application |
| **HIS.exe** | HIS | Historique / Consultation | Application |
| **SIG.exe** | SIG | Module de Signalisation | Application |
| **TEC.exe** | TEC | Module Technique | Application |

**Caractéristiques communes:**
- Framework: MFC (Microsoft Foundation Classes)
- Architecture: Document/View
- Pattern: MVP (Model-View-Presenter)
- Communication: SOAP via gSOAP
- Base: Dérivées de `CArdoWinApp`

---

## 2. APPLICATIONS ADMINISTRATION

### Exécutables Administration (.exe)

| Exécutable | Module Source | Rôle |
|------------|---------------|------|
| **AdmParaR.exe** | ADMGESTION | Paramétrage général LRPPN |
| **AdmModR.exe** | ADMMOD | Gestion des modèles de documents |
| **AdmThLOr.exe** | AdmThLoc | Gestion des thésaurus locaux |
| **AdmThIEr.exe** | AdmThIE | Thésaurus Import/Export |
| **AdmThNVr.exe** | ADMTHNOMVOIE | Thésaurus nomenclature voies |
| **AdmThSEr.exe** | ADMTHSECTOR | Thésaurus secteurs géographiques |

**Fonctionnalités:**
- Paramétrage application
- Gestion référentiels locaux
- Configuration services aide
- Import/Export données thésaurus

---

## 3. UTILITAIRES

### Exécutables Utilitaires (.exe)

| Exécutable | Module | Rôle | Description |
|------------|--------|------|-------------|
| **UriExecLRP.exe** | UriExecLRP | Lanceur URI | Handler protocol `lrppn://` pour lancement applications |
| **SetArdDataNumVer.exe** | SetArdDataNumVer | Versioning | Mise à jour numéro de version base de données |
| **InstNomV.exe** | InstNomV | Installation | Installation nomenclature des voies |
| **Stic16.exe/dll** | STIC16 | Legacy | Support 16-bit (compatibilité) |

**UriExecLRP - Mapping URI:**
```
lrppn://RDP         → C:\CLIENTS\LRPPN3\RDPr.exe
lrppn://MAN         → C:\CLIENTS\LRPPN3\MANr.exe
lrppn://RCJ         → C:\CLIENTS\LRPPN3\RCJr.exe
lrppn://RPJ         → C:\CLIENTS\LRPPN3\rpjR.exe
lrppn://GPJ         → C:\CLIENTS\LRPPN3\GPJr.exe
lrppn://HIS         → C:\CLIENTS\LRPPN3\HISr.exe
lrppn://SIG         → C:\CLIENTS\LRPPN3\SIGr.exe
lrppn://TEC         → C:\CLIENTS\LRPPN3\TECr.exe
lrppn://AdmMod      → C:\CLIENTS\LRPPN3\AdmModr.exe
lrppn://AdmPara     → C:\CLIENTS\LRPPN3\AdmParar.exe
lrppn://AdmThLO     → C:\CLIENTS\LRPPN3\AdmThLOr.exe
lrppn://adminProfils → Administrateur.jar (Java)
lrppn://reference3   → loadFromReference3.jar
lrppn://updater      → LrppnUpdater.jar
lrppn://thlImpExp    → thLocImportExport.jar
```

---

## 4. DLL PARTAGÉES

### 4.1 DLL Métier

| DLL | Module | Rôle | Dépendances |
|-----|--------|------|-------------|
| **Ardo.dll** | Ardo | Logique métier principale Ardo | gSoap, FmlMap, SQLGestion |
| **ArdCom.dll** | ArdCom | Composants communs Ardo | STIC32, Tools |
| **ArdObj.dll** | ArdObj | Objets métier Ardo | SQLGestion |
| **ArdTxt.dll** | ARDTXT | Éditeur TX Text Control | TXTextControl.dll (externe) |
| **ProGestData.dll** | ProGestData | Gestion données procédures | Ardo, SQLGestion |

**Responsabilités:**
- `Ardo.dll`: 
  - `CArdoGestData` - Gestion communication Tuxedo
  - `CArdoWinApp` - Classe application de base
  - `gSoapFmlClient` - Client SOAP/FML32
  
- `ArdCom.dll`:
  - Dialogues communs
  - Listes de sélection
  - Composants UI réutilisables

- `ProGestData.dll`:
  - `CProGestData` - Logique spécifique procédures
  - Règles métier avancées

### 4.2 DLL Communication

| DLL | Module | Rôle | Standard |
|-----|--------|------|----------|
| **gSoap.dll** | gSoap | Client SOAP | gSOAP 2.8.127 |
| **FmlMap.dll** | FmlMap | Mapping FML32 Tuxedo | Tuxedo FML32 |

**Architecture Communication:**
```
Application
    ↓
gSoapFmlClient.cpp (Ardo.dll)
    ↓ ServiceGenerique()
gSoap.dll (tuxbridge::Proxy)
    ↓ HTTP/SOAP
FmlMap.dll (CFmlMap - buffers FML32)
    ↓ tpexport/tpimport
TuxBridge WebService
    ↓
Tuxedo Middleware
```

### 4.3 DLL Données

| DLL | Module | Rôle |
|-----|--------|------|
| **SQLGestion.dll** | SQLGestion | Accès SQL / ORM |
| **STIC32.dll** | STIC32 | Framework STIC (base MFC) |

**Fonctionnalités SQLGestion:**
- ORM propriétaire
- Repository Pattern (Thesaurus)
- Entités métier (Entity_*)
- Accès base locale + distante

### 4.4 DLL Utilitaires

| DLL | Module | Rôle |
|-----|--------|------|
| **Tools.dll** | Tools | Outils transverses |
| **RunLiquibaseDll.dll** | RunLiquibaseDll | Migration BDD (Liquibase) |

---

## 5. BIBLIOTHÈQUES STATIQUES

| Bibliothèque | Module | Type | Rôle |
|--------------|--------|------|------|
| **ZipArchive.lib** | ZipArchive | StaticLibrary | Compression/Décompression ZIP |
| **Codebase.lib** | Codebase | StaticLibrary | Accès fichiers DBF (legacy) |

**Note:** Les bibliothèques statiques sont liées directement dans les exécutables au moment de la compilation.

---

## 6. ORCHESTRATION DES APPLICATIONS

### 6.1 Lancement via URI (Point d'Entrée Principal)

```
┌─────────────────────────────────────┐
│  Navigateur Web / Application       │
│  (Interface Portail CheopsNG)       │
└─────────────┬───────────────────────┘
              │ URI Protocol Handler
              │ lrppn://RDP?TOKEN=xxx&MAT=xxx&...
              ↓
┌─────────────────────────────────────┐
│  UriExecLRP.exe                     │
│  • Parse URI                        │
│  • Extrait paramètres               │
│  • Crée ligne de commande           │
└─────────────┬───────────────────────┘
              │ CreateProcess()
              ↓
┌─────────────────────────────────────┐
│  RDPr.exe (ou autre app)            │
│  Lancé avec arguments:              │
│  • TOKEN=xxx                        │
│  • MAT=matricule                    │
│  • MAT_ANON=anonyme                 │
│  • PPM=profil                       │
│  • SVPR=service                     │
│  • etc.                             │
└─────────────────────────────────────┘
```

**Code source:** `UriExecLRP.cpp`

### 6.2 Flux de Lancement

```cpp
// 1. Handler URI enregistré dans Registry
HKEY_CLASSES_ROOT\lrppn
    URL Protocol = ""
    shell\open\command = "C:\...\UriExecLRP.exe" "%1"

// 2. Parse URL
string uri = "lrppn://RDP?TOKEN=xxx&MAT=yyy&...";
map<string, string> params = getParams(uri);

// 3. Mapping App
string appPath = appList["RDP"]; // → C:\CLIENTS\LRPPN3\RDPr.exe

// 4. Construction ligne commande
string cmdLine = appPath + " " + buildArguments(params);

// 5. Lancement
CreateProcess(NULL, cmdLine, ...);
```

---

## 7. ARCHITECTURE CLIENT LOURD

### 7.1 Architecture en Couches

```
┌─────────────────────────────────────────────────┐
│  COUCHE PRÉSENTATION (Application .exe)         │
│  • RDP.exe, MAN.exe, RCJ.exe, etc.              │
│  • Document/View MFC                            │
│  • Dialogues utilisateur                        │
│  • Menus, Barres d'outils                       │
├─────────────────────────────────────────────────┤
│  COUCHE MÉTIER (DLL)                            │
│  • Ardo.dll (CArdoGestData)                     │
│  • ArdCom.dll (Composants communs)              │
│  • ProGestData.dll (CProGestData)               │
│  • ArdObj.dll (Objets métier)                   │
│  • ArdTxt.dll (Éditeur documents)               │
├─────────────────────────────────────────────────┤
│  COUCHE COMMUNICATION                           │
│  • gSoap.dll (tuxbridge::Proxy)                 │
│  • FmlMap.dll (CFmlMap)                         │
│  • ServiceGenerique(URL, service, params)       │
├─────────────────────────────────────────────────┤
│  COUCHE DONNÉES                                 │
│  • SQLGestion.dll (ORM)                         │
│  • STIC32.dll (Framework)                       │
│  • Base de données locale (SQLite)              │
├─────────────────────────────────────────────────┤
│  COUCHE INFRASTRUCTURE                          │
│  • HTTP/SOAP (gSOAP 2.8.127)                    │
│  • WinInet API                                  │
├─────────────────────────────────────────────────┤
│  SERVEUR APPLICATION                            │
│  • TuxBridge WebService                         │
│  • Tuxedo Middleware                            │
│  • Services Métier (STICGESTION, etc.)          │
└─────────────────────────────────────────────────┘
```

### 7.2 Flux de Communication

```
RDP.exe
    ↓ Utilise
CArdoGestData::TraiterRequete(buffer, "STICGESTION", ...)
    ↓ Appelle (Ardo.dll)
ServiceGenerique(URL, "STICGESTION", matricule, token, inBuffer, outBuffer)
    ↓ Prépare (gSoapFmlClient.cpp)
    • Ajout métadonnées (F_CSV="STICGESTION", F_MAT, F_ADRIP, ...)
    • Conversion CFmlMap → FBFR32
    • Export tpexport() → Base64
    ↓ Invoque (gSoap.dll)
tuxbridge::Proxy::ServiceGenerique(sessionId, inBufferB64, outBufferB64)
    ↓ HTTP POST
    • Header: X-LLNG-TOKEN
    • Body: SOAP XML
    ↓ Réseau
TuxBridge WebService
    ↓ Décodage
    • Base64 → FML32
    • Extraction F_CSV
    ↓ Appel
tpcall("STICGESTION", buffer, ...)
    ↓ Tuxedo
Service Métier (traitement)
    ↓ Retour
SOAP Response (outBufferB64)
    ↓ Import
tpimport() → FBFR32 → CFmlMap
    ↓ Retour
Application traite réponse
```

---

## 8. LANCEMENT INTER-APPLICATIONS

### 8.1 Mécanismes de Lancement

Les applications peuvent lancer d'autres processus via plusieurs méthodes:

**1. CreateProcess() - Lancement Standard**
```cpp
// ArdoGestData.cpp - Ligne 7076
STARTUPINFO si;
PROCESS_INFORMATION pi;
ZeroMemory(&si, sizeof(si));
si.cb = sizeof(si);

bSuccess = CreateProcess(
    NULL,              // lpApplicationName
    szCmdline,         // lpCommandLine (ex: "AdmModR.exe /param")
    NULL,              // lpProcessAttributes
    NULL,              // lpThreadAttributes
    FALSE,             // bInheritHandles
    0,                 // dwCreationFlags
    NULL,              // lpEnvironment
    NULL,              // lpCurrentDirectory
    &si,               // lpStartupInfo
    &pi                // lpProcessInformation
);
```

**2. ShellExecute() - Ouverture Documents**
```cpp
// RdpBsLsDoc.cpp - Ligne 2542
// Ouverture avec application par défaut
ShellExecute(
    NULL,              // hwnd
    "open",            // lpOperation
    fichier.docx,      // lpFile
    NULL,              // lpParameters
    NULL,              // lpDirectory
    SW_SHOWNORMAL      // nShowCmd
);
```

### 8.2 Cas d'Usage

| Scénario | Mécanisme | Exemple |
|----------|-----------|---------|
| Édition modèle document | CreateProcess | `AdmModR.exe /edit /id=123` |
| Ouverture document Office | ShellExecute | `rapport.docx` |
| Export PDF | CreateProcess | `pdfcreator.exe input.rtf` |
| Administration | CreateProcess | `AdmParaR.exe /config` |
| Navigateur externe | ShellExecute | `https://portail.justice.fr` |

---

## 9. DÉPENDANCES DE CHARGEMENT

### 9.1 Graphe de Dépendances RDP.exe

```
RDP.exe (Application Principale)
│
├── Ardo.dll ★
│   ├── gSoap.dll
│   │   └── WinInet.dll (Windows)
│   ├── FmlMap.dll
│   │   └── atmi.dll (Tuxedo SDK)
│   ├── SQLGestion.dll
│   │   └── ODBC32.dll (Windows)
│   └── STIC32.dll
│       └── MFC140.dll (Visual Studio)
│
├── ArdCom.dll
│   ├── STIC32.dll (déjà chargé)
│   └── Tools.dll
│
├── ArdTxt.dll
│   ├── TXTextControl27.dll (Externe)
│   └── STIC32.dll (déjà chargé)
│
├── ProGestData.dll
│   ├── Ardo.dll ★ (déjà chargé)
│   ├── SQLGestion.dll (déjà chargé)
│   └── FmlMap.dll (déjà chargé)
│
├── ArdObj.dll
│   └── SQLGestion.dll (déjà chargé)
│
└── [Bibliothèques système Windows]
    ├── KERNEL32.dll
    ├── USER32.dll
    ├── GDI32.dll
    ├── COMCTL32.dll
    └── MSVCR140.dll
```

**★ Note:** `Ardo.dll` est la DLL centrale chargée par toutes les applications.

### 9.2 Ordre de Chargement

```cpp
// 1. Application démarre
int WINAPI WinMain(HINSTANCE hInstance, ...)
{
    // 2. Framework MFC s'initialise
    AfxWinInit(hInstance, ...);
    
    // 3. Constructeur application
    CRDPApp::CRDPApp() : CArdoWinApp()
    {
        // 4. Chargement explicite DLL
        mpArdoGestData = new CProGestData(); // → Charge ProGestData.dll
                                              // → Charge Ardo.dll
                                              // → Charge gSoap.dll
                                              // → Charge FmlMap.dll
    }
    
    // 5. InitInstance()
    BOOL CRDPApp::InitInstance()
    {
        // Chargement implicite via imports
        // ArdCom.dll, ArdTxt.dll chargées à la première utilisation
        
        InitArdoDll(1, sVersion); // Initialise Ardo.dll
    }
}
```

### 9.3 Résolution de Dépendances

**Chemins de recherche DLL (ordre):**
1. Répertoire de l'exécutable (`C:\CLIENTS\LRPPN3\`)
2. Répertoire système Windows (`C:\Windows\System32\`)
3. Répertoire Windows (`C:\Windows\`)
4. Répertoire courant
5. Variable PATH

**Gestion version:**
- Manifeste embarqué (Side-by-Side assemblies)
- Dépendances Visual C++ Runtime (MSVC 2015-2019)

---

## 10. COMMUNICATION APPLICATIVE

### 10.1 Mode Synchrone (Requête/Réponse)

**Architecture:**
```
Client → ServiceGenerique() → HTTP POST → TuxBridge → tpcall() → Service Tuxedo
                                                                         ↓
Client ← Réponse             ← HTTP 200 ← SOAP XML  ← return   ← Traitement
```

**Code type:**
```cpp
// Préparation buffer FML32
CFmlMap* inBuffer = GetFmlPtr(BUFFER_ENVOI);
inBuffer->SetValue("FPRPR_KPRO", sCleProc);
inBuffer->SetValue("FTCPC_FONCTION", "CHARGER");

// Appel synchrone
int iRet = mpArdoGestData->TraiterRequete(
    BUFFER_ENVOI,           // Buffer entrée
    "STICGESTION",          // Service Tuxedo
    "Chargement procédure", // Libellé (logs)
    BUFFER_RECEPTION        // Buffer sortie
);

// Vérification réponse
if (iRet == RDP_OK) {
    CString sStatut;
    GetValue(BUFFER_RECEPTION, "FTCPC_STATUT", sStatut);
}
```

### 10.2 Mode Asynchrone (Messages Windows)

**Communication inter-fenêtres:**
```cpp
// Fenêtre émettrice
PostMessage(hWndTarget, WM_ACTION, ACTION_MAJ_ITEM, lParam);

// Fenêtre réceptrice
LRESULT OnAction(WPARAM wParam, LPARAM lParam)
{
    switch (wParam) {
        case ACTION_MAJ_ITEM:
            RechargerDonnees();
            break;
        case ACTION_MAJ_DATA:
            SauvegarderModifications();
            break;
    }
}
```

**Messages personnalisés:**
- `WM_ACTION` : Actions métier
- `WM_KFBUTTON_KILLFOCUS` : Validation contrôles
- `WM_MODIF_STATUT` : Changement état

### 10.3 Partage de Données

**1. Base de Données Locale**
```
C:\CLIENTS\LRPPN3\Data\
├── reference.db (SQLite)
├── cache.db
└── config.db
```

**2. Fichiers DBF (Legacy)**
```cpp
// Via Codebase.lib
d4open("MAGISTRAT.DBF");
d4seek("MAT", matricule);
```

**3. Buffers FML32 Partagés**
```cpp
// Pool de buffers réutilisables
enum { 
    BUFFER_ENVOI = 0,
    BUFFER_RECEPTION = 1,
    BUFFER_REFERENCE = 2,
    BUFFER_TEMP = 3
};

CFmlMap* pBuffer = m_pTabBufferFml[BUFFER_ENVOI];
```

---

## 11. INITIALISATION TYPE

### 11.1 Séquence de Démarrage Complète

```cpp
// ═════════════════════════════════════════════════════
// PHASE 1: LANCEMENT
// ═════════════════════════════════════════════════════

// Portail Web → URI
lrppn://RDP?TOKEN=abc123&MAT=999999&PPM=RJ&...

// ═════════════════════════════════════════════════════
// PHASE 2: HANDLER URI
// ═════════════════════════════════════════════════════

UriExecLRP.exe
    → Parse URI
    → CreateProcess("RDPr.exe /TOKEN=abc123 /MAT=999999 ...")

// ═════════════════════════════════════════════════════
// PHASE 3: INITIALISATION APPLICATION
// ═════════════════════════════════════════════════════

int WINAPI WinMain(...)
{
    CRDPApp theApp;  // Constructeur
    
    // RDP.cpp - Ligne 74
    CRDPApp::CRDPApp() : CArdoWinApp()
    {
        // Création instance gestion données
        mpArdoGestData = new CProGestData();
        // → Charge ProGestData.dll
        // → Charge Ardo.dll
        // → Charge gSoap.dll, FmlMap.dll
    }
    
    // Initialisation MFC
    return theApp.Run();
}

// ═════════════════════════════════════════════════════
// PHASE 4: INITIALISATION DLL
// ═════════════════════════════════════════════════════

BOOL CRDPApp::InitInstance()
{
    // Parse arguments ligne commande
    ParseCommandLine(cmdInfo);
    
    // Extraction paramètres
    CString sToken    = GetParam("TOKEN");
    CString sMatricule = GetParam("MAT");
    CString sService   = GetParam("SVPR");
    
    // Stockage session
    SetCheopsNGSession("X-LLNG-TOKEN: " + sToken);
    SetMatricule(sMatricule);
    SetCodeService(sService);
    
    // Initialisation DLL Ardo
    CString sVersion = "03.00.00@99.99.99";
    InitArdoDll(1, sVersion);
    
    // Initialisation buffers FML32
    mpArdoGestData->InitBuffersFml();
    mpArdoGestData->InitTabCle();
}

// ═════════════════════════════════════════════════════
// PHASE 5: CONNEXION SERVEUR
// ═════════════════════════════════════════════════════

// Test connexion
CString sErrorMsg;
BOOL bConnexion = ServiceGenerique(
    GetUrlLRPPN(),              // "http://serveur/tuxbridge"
    "STICGESTION",              // Service test
    GetMatricule(),
    GetMatriculeAnon(),
    GetCheopsNGSession(),
    pBufferTest,
    pBufferResponse,
    sErrorMsg
);

if (!bConnexion) {
    SetModeConnexion(MODE_DEGRADE);
    AfxMessageBox("Connexion serveur impossible");
}

// ═════════════════════════════════════════════════════
// PHASE 6: CHARGEMENT CONTEXTE UTILISATEUR
// ═════════════════════════════════════════════════════

// Récupération profil utilisateur
pBuffer->SetValue("F_CSV", "ARDOPARAM");
pBuffer->SetValue("F_MAT", GetMatricule());
pBuffer->SetValue("FTCPC_FONCTION", "LOAD_PROFIL");

TraiterRequete(BUFFER_ENVOI, "ARDOPARAM", "", BUFFER_RECEPTION);

// Extraction droits
GetValue(BUFFER_RECEPTION, "FTCPC_DROITS", sDroits);
m_iUserStatus = ParseDroits(sDroits);

// ═════════════════════════════════════════════════════
// PHASE 7: CHARGEMENT RÉFÉRENTIELS
// ═════════════════════════════════════════════════════

// Thésaurus
Thesaurus::GetThServiceAide().SelectAll();
Thesaurus::GetThParametres().SelectAll();
Thesaurus::GetThNatinf().SelectAll();

// Cache local
ChargerCacheLocal();

// ═════════════════════════════════════════════════════
// PHASE 8: AFFICHAGE INTERFACE
// ═════════════════════════════════════════════════════

// Création fenêtre principale
CMainFrame* pMainFrame = new CMainFrame;
pMainFrame->LoadFrame(IDR_MAINFRAME);
pMainFrame->ShowWindow(SW_SHOW);
m_pMainWnd = pMainFrame;

// Document/View
CDocument* pDoc = pDocTemplate->OpenDocumentFile(NULL);
```

### 11.2 Diagramme Temporel

```
Temps   │ Action
════════╪═══════════════════════════════════════════════
0ms     │ Clic URI dans navigateur
        │
50ms    │ UriExecLRP.exe lance
        │
100ms   │ CreateProcess(RDPr.exe)
        │
150ms   │ WinMain() - Constructeur CRDPApp
        │ → new CProGestData()
        │
200ms   │ Chargement DLL (Ardo, gSoap, FmlMap...)
        │
300ms   │ InitInstance()
        │ → Parse arguments
        │
350ms   │ InitArdoDll()
        │ → InitBuffersFml()
        │
400ms   │ Test connexion ServiceGenerique()
        │ → HTTP POST
        │
1000ms  │ ← Réponse serveur (OK/KO)
        │
1200ms  │ Chargement profil utilisateur
        │ → ARDOPARAM
        │
1500ms  │ ← Profil reçu
        │
1600ms  │ Chargement thésaurus
        │ → Base locale + cache
        │
2000ms  │ Création fenêtre principale
        │ → LoadFrame()
        │
2200ms  │ ShowWindow(SW_SHOW)
        │ ✓ Application prête
```

---

## 12. DÉPLOIEMENT

### 12.1 Structure d'Installation

```
C:\CLIENTS\LRPPN3\
│
├── [Exécutables Métier]
│   ├── RDPr.exe
│   ├── MANr.exe
│   ├── RCJr.exe
│   ├── rpjR.exe
│   ├── GPJr.exe
│   ├── HISr.exe
│   ├── SIGr.exe
│   └── TECr.exe
│
├── [Exécutables Administration]
│   ├── AdmParar.exe
│   ├── AdmModr.exe
│   ├── AdmThLOr.exe
│   ├── AdmThIEr.exe
│   ├── AdmThNVr.exe
│   └── AdmThSEr.exe
│
├── [Utilitaires]
│   ├── UriExecLRP.exe
│   ├── SetArdDataNumVer.exe
│   ├── InstNomV.exe
│   ├── Administrateur.jar
│   ├── loadFromReference3.jar
│   ├── LrppnUpdater.jar
│   └── thLocImportExport.jar
│
├── [DLL Métier]
│   ├── Ardo.dll
│   ├── ArdCom.dll
│   ├── ArdObj.dll
│   ├── ArdTxt.dll
│   └── ProGestData.dll
│
├── [DLL Communication]
│   ├── gSoap.dll
│   └── FmlMap.dll
│
├── [DLL Données]
│   ├── SQLGestion.dll
│   └── STIC32.dll
│
├── [DLL Utilitaires]
│   ├── Tools.dll
│   ├── RunLiquibaseDll.dll
│   └── ZipArchive.dll (si dynamique)
│
├── [DLL Externes]
│   ├── TXTextControl27.dll
│   ├── atmi.dll (Tuxedo)
│   └── [Visual C++ Redistributable]
│
├── Data\
│   ├── reference.db
│   ├── cache.db
│   └── config.db
│
├── Logs\
│   ├── LRPPN.log
│   ├── UriExecLRP.log
│   └── Echanges.log
│
├── Aide\
│   ├── ADMDOC.chm
│   ├── RDP.chm
│   └── [Autres .chm]
│
├── Modeles\
│   ├── PV_Standard.rtf
│   ├── Requisition.docx
│   └── [Templates]
│
├── Config\
│   ├── LRPPN.ini
│   ├── Connexion.xml
│   └── Services.xml
│
└── Temp\
    └── [Fichiers temporaires]
```

### 12.2 Installation WiX

**Projet:** `LrppnInstallerWix.wixproj`

**Références projets:**
```xml
<ProjectReference Include="..\ADMGESTION\AdmGestion.vcxproj" />
<ProjectReference Include="..\ADMMOD\AdmMod.vcxproj" />
<ProjectReference Include="..\AdmThLoc\AdmThLoc.vcxproj" />
<ProjectReference Include="..\RDP\RDP.vcxproj" />
<ProjectReference Include="..\MAN\MAN.vcxproj" />
<!-- ... tous les projets ... -->
```

**Composants WiX (Projects.wxs):**
```xml
<Component Id="RDP" Guid="*">
    <File Id="RDPr.exe" Source="$(var.RDP.TargetPath)" />
</Component>
<Component Id="AdmModR" Guid="*">
    <File Id="AdmModR.exe" Source="$(var.AdmMod.TargetPath)" />
</Component>
<!-- ... -->
```

### 12.3 Enregistrement URI Handler

**Registry:**
```ini
[HKEY_CLASSES_ROOT\lrppn]
@="URL:LRPPN Protocol"
"URL Protocol"=""

[HKEY_CLASSES_ROOT\lrppn\shell\open\command]
@="\"C:\\CLIENTS\\LRPPN3\\UriExecLRP.exe\" \"%1\""
```

### 12.4 Configuration Post-Installation

**1. LRPPN.ini**
```ini
[Connexion]
URLServeur=http://serveur-lrppn/tuxbridge
Timeout=30000

[Logs]
NiveauLog=INFO
CheminLogs=C:\CLIENTS\LRPPN3\Logs

[Base]
CheminData=C:\CLIENTS\LRPPN3\Data
```

**2. Connexion.xml**
```xml
<configuration>
    <urlLRPPN>http://serveur/tuxbridge</urlLRPPN>
    <serviceSoap>ServiceGenerique</serviceSoap>
    <timeout>30</timeout>
</configuration>
```

### 12.5 Dépendances Système

**Prérequis:**
- Windows 7 SP1 / Windows 10 / Windows 11
- .NET Framework 4.7.2+ (pour composants Java)
- Visual C++ Redistributable 2015-2019 (x86)
- Java Runtime Environment 8+ (pour .jar)

**Composants optionnels:**
- TX Text Control (licence séparée)
- Microsoft Office (pour édition documents)

### 12.6 Mise à Jour

**Mécanisme:**
```
LrppnUpdater.jar
    ↓ Télécharge
Nouveau package (ZIP)
    ↓ Extrait
C:\CLIENTS\LRPPN3\Temp\
    ↓ Ferme applications
Arrêt processus RDP.exe, etc.
    ↓ Remplace fichiers
Copie vers C:\CLIENTS\LRPPN3\
    ↓ Redémarre
Relance application
```

---

## ANNEXES

### A. Taille Approximative Déploiement

| Composant | Taille |
|-----------|--------|
| Exécutables (.exe) | ~50 MB |
| DLL métier | ~30 MB |
| DLL système | ~20 MB |
| Base de données | ~100 MB |
| Aide (.chm) | ~10 MB |
| Modèles | ~5 MB |
| **TOTAL** | **~215 MB** |

### B. Performance Démarrage

| Étape | Durée Moyenne |
|-------|---------------|
| Lancement UriExecLRP | 50 ms |
| Chargement DLL | 200 ms |
| Connexion serveur | 600 ms |
| Chargement contexte | 500 ms |
| Affichage UI | 200 ms |
| **TOTAL** | **~1.5 secondes** |

### C. Références Techniques

- **gSOAP**: https://www.genivia.com/dev.html (v2.8.127)
- **TX Text Control**: https://www.textcontrol.com/
- **Tuxedo**: Oracle Tuxedo 12c
- **MFC**: Microsoft Foundation Classes (Visual Studio 2019)
- **WiX Toolset**: v3.11

---

**FIN DU DOCUMENT**


# ARCHITECTURE DES APPELS CLIENT LOURD → TUXEDO

**Date**: 28 janvier 2026  
**Composant**: Communication Client/Serveur via gSOAP  
**Framework**: Tuxedo Bridge + FML32

---

## 1. VUE D'ENSEMBLE

Le client lourd communique avec le middleware **Tuxedo** via un **Web Service SOAP** générique qui fait office de bridge/pont.

### Architecture Globale

```
┌──────────────────────────────────────────────────────────────┐
│  CLIENT LOURD (Applications MFC)                             │
│  • Ardo, RDP, RCJ, RPJ, MAN, TEC, SIG                        │
└────────────────────┬─────────────────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────────────────┐
│  COUCHE ACCÈS DONNÉES                                        │
│  • CArdoGestData::TraiterRequete()                           │
│  • CProGestData (dérivée)                                    │
└────────────────────┬─────────────────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────────────────┐
│  CLIENT SOAP (gSoapFmlClient)                                │
│  • ServiceGenerique()                                        │
│  • Conversion FML32 ↔ Base64                                 │
└────────────────────┬─────────────────────────────────────────┘
                     │
                     ▼ HTTP/SOAP
┌──────────────────────────────────────────────────────────────┐
│  WEB SERVICE TUXBRIDGE                                       │
│  • tuxbridge::Proxy (généré par gSOAP)                       │
│  • Endpoint: GetUrlLRPPN()                                   │
└────────────────────┬─────────────────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────────────────┐
│  MIDDLEWARE TUXEDO                                           │
│  • Routage vers services métier                              │
│  • Gestion transactions                                      │
└────────────────────┬─────────────────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────────────────┐
│  SERVICES MÉTIER TUXEDO                                      │
│  • Code service déterminé par F_CSV                          │
└──────────────────────────────────────────────────────────────┘
```

---

## 2. BIBLIOTHÈQUE GSOAP

### Localisation
**Dossier**: `gSoap/`

### Composants Principaux

| Fichier | Description |
|---------|-------------|
| `stdsoap2.h/cpp` | Runtime engine gSOAP 2.8.127 |
| `tuxBridgeProxy.h/cpp` | Proxy client généré |
| `tuxBridgeStub.h` | Stubs SOAP |
| `tuxBridgeH.h` | Types de données |
| `tuxBridgeC.cpp` | Sérialisation/Désérialisation |
| `gsoapWinInet.cpp` | Transport HTTP via WinInet |

### Namespace
```cpp
namespace tuxbridge {
    class Proxy {
        int ServiceGenerique(
            const char *sessionId,
            const xsd__base64Binary& inBuffer,
            xsd__base64Binary &outBuffer
        );
    };
}
```

---

## 3. POINT D'ENTRÉE : ServiceGenerique()

### Signature
**Fichier**: `Ardo/gSoapFmlClient.cpp` (ligne 72)

```cpp
BOOL ServiceGenerique(
    CString adresse,           // URL endpoint du web service
    CString sCodeService,      // Code du service Tuxedo à invoquer
    CString sMatricule,        // Matricule utilisateur connecté
    CString sMatriculeAnon,    // Matricule anonymisé (audit)
    CString sCheopsNGSession,  // Token d'authentification SSO
    CFmlMap* inBuffer,         // Buffer FML32 d'entrée (données métier)
    CFmlMap* outBuffer,        // Buffer FML32 de sortie (réponse)
    CString &sErrorMessage     // Message d'erreur si échec
);
```

### Rôle
1. **Enrichissement automatique** : Ajoute métadonnées de traçabilité
2. **Conversion** : FML32 → Base64 (via `tuxExport`)
3. **Appel SOAP** : Invoque le web service
4. **Conversion retour** : Base64 → FML32 (via `tuxImport`)
5. **Gestion erreurs** : Capture et remonte les faults SOAP

---

## 4. MÉTADONNÉES INJECTÉES AUTOMATIQUEMENT

Chaque appel est enrichi avec des champs FML32 de journalisation :

| Champ FML | Description | Source |
|-----------|-------------|--------|
| `F_CSV` | **Code Service Tuxedo** | Paramètre `sCodeService` |
| `F_MAT` | Matricule utilisateur | Session CSticWinApp |
| `F_IDI` | Matricule anonymisé | Pour audit/RGPD |
| `F_ADRIP` | Adresse IP poste | `gethostbyname()` |
| `F_NPT` | Nom poste client | `gethostname()` |
| `F_ATF` | Code application | **"ARD30000"** (constante) |
| `F_CTX` | Contexte technique | **"IARDSER3"** (constante) |
| `F_CSR` | Code service requête | **"RE"** (requête) |
| `F_NOSESS` | Numéro session | "1" |
| `F_SDM`, `F_PST`, `F_LSV` | Réservés | Espaces |
| `F_NOMFC`, `F_PFC`, `F_GFC`, `F_FFC` | Contrôle fonctionnel | Espaces |
| `F_SCOM` | Service complémentaire | Espace |
| `F_TMG` | Type message | **"D\0"** |

### Code Source
```cpp
// gSoapFmlClient.cpp lignes 79-98
inBuffer->SetValue("F_NOSESS", "1");
inBuffer->SetValue("F_ADRIP", ipAddress);
inBuffer->SetValue("F_ATF", "ARD30000");
inBuffer->SetValue("F_NPT", hostname);
inBuffer->SetValue("F_CTX", "IARDSER3");
inBuffer->SetValue("F_MAT", sMatricule);
inBuffer->SetValue("F_CSV", sCodeService);  // ← DÉTERMINE LE SERVICE TUXEDO
inBuffer->SetValue("F_CSR", "RE");
inBuffer->SetValue("F_IDI", sMatriculeAnon);
// ...
```

---

## 5. FLUX DE DONNÉES DÉTAILLÉ

### 5.1 Préparation Requête (Client → Serveur)

```
┌─────────────────────────────────────────────────────────┐
│ 1. Application métier prépare CFmlMap                   │
│    • Ajoute champs métier (FPRPR_*, FPADS_*, etc.)     │
└───────────────────┬─────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────┐
│ 2. ServiceGenerique() enrichit avec métadonnées        │
│    • F_CSV, F_MAT, F_ADRIP, F_NPT, etc.                │
└───────────────────┬─────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────┐
│ 3. tuxExport() : FML32 → Base64                         │
│    • tpalloc("FML32") → allocation buffer Tuxedo        │
│    • Fcpy32() → copie données                           │
│    • tpexport() → conversion endianness + base64        │
└───────────────────┬─────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────┐
│ 4. lrppnFML.ServiceGenerique() : Appel SOAP             │
│    • Sérialisation XML                                  │
│    • HTTP POST vers endpoint                            │
│    • Header: X-LLNG-TOKEN (CheopsNG SSO)                │
└───────────────────┬─────────────────────────────────────┘
                    │
                    ▼
                [RÉSEAU]
```

### 5.2 Traitement Réponse (Serveur → Client)

```
                [RÉSEAU]
                    │
                    ▼
┌─────────────────────────────────────────────────────────┐
│ 5. Réception réponse SOAP                               │
│    • Désérialisation XML                                │
│    • Extraction xsd__base64Binary                       │
└───────────────────┬─────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────┐
│ 6. tuxImport() : Base64 → FML32                         │
│    • tpalloc() → allocation buffer sortie               │
│    • tpimport() → décodage base64 + endianness          │
│    • Copie vers CFmlMap outBuffer                       │
└───────────────────┬─────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────┐
│ 7. Application métier récupère CFmlMap                  │
│    • Lit champs de réponse                              │
│    • Traite données métier                              │
└─────────────────────────────────────────────────────────┘
```

---

## 6. GESTION DES ERREURS

### Types d'Erreurs

```cpp
try {
    // Appel ServiceGenerique()
} catch(tuxError& e) {
    // Erreur Tuxedo : tpalloc, tpexport, tpimport
    // Message via tpstrerror(tperrno)
} catch(fmlError& e) {
    // Erreur FML32 : Fcpy32, Fget32, etc.
    // Message via Fstrerror32(Ferror32)
} catch(std::runtime_error& e) {
    // Erreur SOAP : timeout, parsing, réseau
    // Message via soap_stream_fault()
}
```

### Codes Retour

| Code | Signification |
|------|---------------|
| `SOAP_OK` | Succès |
| `SOAP_ERR` | Erreur générique |
| `SOAP_TCP_ERROR` | Erreur réseau |
| `SOAP_HTTP_ERROR` | Erreur HTTP (4xx, 5xx) |
| `SOAP_SSL_ERROR` | Erreur SSL/TLS |
| `SOAP_FAULT` | Fault SOAP métier |

---

## 7. EXEMPLES D'UTILISATION

### 7.1 Appel Générique depuis CArdoGestData

**Fichier**: `Ardo/ArdoGestData.cpp` (ligne 301)

```cpp
CString sErrorMessage;
bRet = ServiceGenerique(
    ((CArdoWinApp*) AfxGetApp())->GetUrlLRPPN(),        // URL du serveur
    ((CSticWinApp*) AfxGetApp())->GetCodeService(),     // Code service
    ((CSticWinApp*) AfxGetApp())->CSticWinApp::GetMatricule(),
    ((CSticWinApp*) AfxGetApp())->GetMatriculeAnon(),
    ((CSticWinApp*) AfxGetApp())->GetCheopsNGSession(),
    m_pTabBufferFml[numBuf],                            // Buffer entrée
    m_pTabBufferFml[numBufRcv],                         // Buffer sortie
    sErrorMessage
);

if (!bRet) {
    // Gestion erreur
    AfxMessageBox("Erreur de communication");
    ((CArdoWinApp*) AfxGetApp())->SetModeConnexion(MODE_DEGRADE);
}
```

### 7.2 Wrapper TraiterRequete()

**Fichier**: `Ardo/ArdoGestData.cpp`

```cpp
int CArdoGestData::TraiterRequete(
    int numBuf,           // Numéro buffer FML entrée
    CString sService,     // Code service Tuxedo
    CString sEtiquette,   // Libellé pour la waitbar
    int numBufRcv         // Numéro buffer FML sortie
) {
    // Prépare le buffer
    m_pTabBufferFml[numBuf]->SetValue("F_CSV", sService);
    
    // Appelle ServiceGenerique()
    bRet = ServiceGenerique(...);
    
    // Vérifie réponse
    if (!VerifReponse(numBufRcv, szCodeErreur, szMessage, ...)) {
        // Traite erreur métier
    }
}
```

---

## 8. SERVICES TUXEDO APPELÉS

Le **code service** (`F_CSV`) détermine quelle fonction Tuxedo est invoquée côté serveur.

### Exemples de Codes Services

| Code | Description Probable |
|------|---------------------|
| `STICGESTION` | Gestion générale STIC |
| `ARDOGEST` | Gestion documents Ardo |
| `ARDOPARAM` | Paramétrage Ardo |
| `ARDOLOAD` | Chargement procédures |
| `ARDOSAVE` | Sauvegarde documents |
| `RPJGEST` | Gestion RPJ |
| `RCJGEST` | Gestion RCJ |

**Note**: Les codes exacts sont définis dans les constantes applicatives (à documenter via analyse du code).

---

## 9. CONFIGURATION

### URL du Service
Récupérée via :
```cpp
((CArdoWinApp*) AfxGetApp())->GetUrlLRPPN()
```

Typiquement stockée dans :
- Registre Windows
- Fichier INI/configuration
- Variable d'environnement

Format : `http://serveur:port/tuxbridge/services/lrppnFML`

### Authentification
Header HTTP personnalisé :
```cpp
lrppnFML.soap->http_extra_header = sCheopsNGSession;
```

Format : `X-LLNG-TOKEN: <token-sso>`

---

## 10. BUFFERS FML32

### Gestion des Buffers

```cpp
class CFmlMap {
    FBFR32* GetBuffer();           // Pointeur buffer Tuxedo
    FBFR32** GetBufferAdr();       // Adresse pointeur (pour tuxImport)
    
    SetValue(field, value);        // Écriture champ
    GetValue(field, &value);       // Lecture champ
    FreeBuffer();                  // Libération mémoire
};
```

### Pool de Buffers (CArdoGestData)

```cpp
#define BUFFER_ENVOI      0
#define BUFFER_RECEPTION  1
#define BUFFER_REFERENCE  2
#define BUFFER_TEMP       3
// ...

CFmlMap* m_pTabBufferFml[NB_BUFFER_MAX];
```

---

## 11. PERFORMANCES & OPTIMISATIONS

### Réutilisation des Buffers
```cpp
// Évite allocations multiples
m_pTabBufferFml[BUFFER_ENVOI]->FreeBuffer();  // Vide sans désallouer
m_pTabBufferFml[BUFFER_ENVOI]->SetValue(...); // Réutilise
```

### Copie Temporaire
```cpp
// Si buffer envoi = buffer réception
if(numBuf == numBufRcv) {
    SetFmlPtr(BUFFER_TEMP, GetFmlPtr(numBuf));
    numBuf = BUFFER_TEMP;
}
```

### Logs des Échanges
Activable via `LOGS_ECHANGES` :
```cpp
if (bIsLogsEchanges) {
    EcrireBuff("Question", numBuf);
    // ... appel service ...
    EcrireBuff("Réponse", numBufRcv);
}
```

---

## 12. DÉPENDANCES

### Bibliothèques Externes
- **gSOAP 2.8.127** : Framework SOAP
- **Tuxedo FML32** : Manipulation buffers FML
- **WinInet** : Transport HTTP Windows
- **Winsock** : Réseau (gethostname)

### Fichiers Clés

| Fichier | Rôle |
|---------|------|
| `gSoap/stdsoap2.h` | Runtime gSOAP |
| `gSoap/tuxBridgeProxy.h` | Proxy client |
| `Ardo/gSoapFmlClient.h` | Interface ServiceGenerique |
| `Ardo/gSoapFmlClient.cpp` | Implémentation |
| `Ardo/ArdoGestData.h/cpp` | Couche accès données |
| `FmlMap/FmlMap.h` | Wrapper FML32 |
| `FmlMap/fml32.h` | API Tuxedo FML |
| `FmlMap/atmi.h` | API Tuxedo ATMI |

---

## 13. MODES DE CONNEXION

### Mode Normal
Appels directs vers Tuxedo via web service.

### Mode Dégradé
Activé si échec communication :
```cpp
((CArdoWinApp*) AfxGetApp())->SetModeConnexion(MODE_DEGRADE);
```

En mode dégradé :
- Pas d'appels serveur
- Données locales uniquement
- Fonctionnalités limitées

---

## 14. SÉCURITÉ

### Authentification
- **SSO CheopsNG** : Token dans header HTTP
- Validation côté serveur du token

### Traçabilité
- Matricule utilisateur (`F_MAT`)
- Matricule anonymisé (`F_IDI`)
- Adresse IP poste (`F_ADRIP`)
- Nom poste (`F_NPT`)
- Code application (`F_ATF`)

### Transport
- HTTP (ou HTTPS si configuré)
- Pas de chiffrement des buffers FML (responsabilité du transport)

---

## 15. DEBUGGING

### Logs gSOAP
```cpp
soap_set_test_logfile(soap, "soap.log");
soap->recv_timeout = 30;  // Timeout réseau
soap->send_timeout = 30;
```

### Traces FML
```cpp
OutputDebugString(msg);  // Traces buffers
LOGLRPPN("message");     // Logs applicatifs
```

### Analyse Réseau
- Wireshark sur port du service
- Fiddler pour inspecter SOAP

---

## 16. DIAGRAMME DE SÉQUENCE

```
Client MFC          gSoapFmlClient    tuxBridge    TuxBridge WS    Tuxedo
    |                     |                |              |            |
    |--TraiterRequete()-->|                |              |            |
    |                     |                |              |            |
    |                     |--ServiceGenerique()           |            |
    |                     |  (enrichit F_CSV, F_MAT...)   |            |
    |                     |                |              |            |
    |                     |--tuxExport()-->|              |            |
    |                     |  (FML32→B64)   |              |            |
    |                     |                |              |            |
    |                     |                |--SOAP/HTTP-->|            |
    |                     |                |  POST        |            |
    |                     |                |              |            |
    |                     |                |              |--tpcall()-->|
    |                     |                |              | (F_CSV)    |
    |                     |                |              |            |
    |                     |                |              |<-----------|
    |                     |                |              |  FML32     |
    |                     |                |              |            |
    |                     |                |<--SOAP resp--|            |
    |                     |                |   (B64)      |            |
    |                     |                |              |            |
    |                     |<--tuxImport()--|              |            |
    |                     |  (B64→FML32)   |              |            |
    |                     |                |              |            |
    |<--CFmlMap outBuf----|                |              |            |
    |                     |                |              |            |
```

---

## 17. POINTS D'ATTENTION

### Gestion Mémoire
- Les buffers FML32 doivent être libérés via `tpfree()`
- gSOAP gère sa propre mémoire via `soap_malloc()`
- Attention aux fuites en cas d'exception

### Endianness
`tpexport()` / `tpimport()` gèrent automatiquement la conversion big/little endian entre client Windows et serveur Unix.

### Timeout
Par défaut, pas de timeout configuré → peut bloquer indéfiniment.

### Thread Safety
- `ServiceGenerique()` n'est **PAS thread-safe**
- Utilisation mono-thread (UI thread MFC)

---

## 18. ÉVOLUTIONS POSSIBLES

### REST API
Remplacer SOAP par REST/JSON :
- Plus léger que SOAP
- Meilleure interopérabilité
- Moins verbeux

### Compression
Compresser les buffers base64 (gzip) pour réduire la bande passante.

### Cache Local
Mettre en cache certaines réponses pour mode déconnecté.

### Async/Await
Rendre les appels asynchrones pour ne pas bloquer l'UI.

---

**FIN DU DOCUMENT**



# MODULE ADMGESTION

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
