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
