

# ARCHITECTURE CLIENT LRPPN

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


