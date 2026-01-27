# Portfolio Professionnel AR — MVP (Version complète des outils & versions)

## Résumé
**Titre :** Portfolio Professionnel en Réalité Augmentée (AR)  
**Objectif :** Transformer une carte de visite ou un document physique en portail vers une expérience AR immersive (modèles 3D PBR, vidéos, texte), pilotable en temps réel via une interface d'administration web et doté d'outils d'analyse.

**Répertoire distant :** [https://github.com/Garnel-Diffo/Portfolio-Professionnel-AR.git](https://github.com/Garnel-Diffo/Portfolio-Professionnel-AR.git)

---

## Table des matières
1. [Objectifs du MVP](#1-objectifs-du-mvp)
2. [Liste exhaustive des outils & versions (à utiliser)](#2-liste-exhaustive-des-outils--versions-utiliser-strictement)
3. [Architecture système (schéma)](#3-architecture-système-schéma)
4. [Structure recommandée du dépôt](#4-structure-recommandée-du-dépôt)
5. [Installation & Lancement](#5-installation--lancement)
    *   [Clonage du projet](#51-clonage-du-projet)
    *   [Lancement Admin Web](#52-lancement-admin-web)
    *   [Lancement Unity App](#53-lancement-unity-app)
6. [Configuration Unity 6.3 LTS — détail pas-à-pas](#6-configuration-unity-63-lts--guide-détaillé)
7. [Packages Unity : installation & versions compatibles](#7-packages-unity--installation--bonnes-pratiques)
8. [Configuration Firebase — résumé opérationnel + règles](#8-configuration-firebase--règles-et-déploiement)
9. [Schéma de données Firestore (exemples)](#9-schéma-de-données-firestore-exemples)
10. [Pipeline d'assets et optimisations](#10-pipeline-dassets--optimisation)
11. [CI / CD, sécurité, Git & bonnes pratiques](#11-ci--cd-sécurité--git)
12. [Release & builds (Android)](#12-release--builds-android)
13. [Annexes : liens officiels & notes](#13-annexes--liens-officiels)

---

## 1. Objectifs du MVP
Livrer un MVP fonctionnel qui :
- Detecte une image-cible (carte) et affiche un modèle 3D (.glb).  
- Permet via l'admin web d'uploader des modèles et lier des marqueurs.  
- Fournit analytics (scans, vues, durée).  
- Permet le mode présentation (contrôle à distance via Firestore).  
- Produit un APK Android de démonstration.

---

## 2. Liste exhaustive des outils & versions (UTILISER STRICTEMENT)

### Environnement général
- Système d'exploitation : Windows 10/11 64-bit ou macOS 12+
- Git : version stable (>=2.30)
- Python (optionnel pour outils) : 3.8+

### Front / Admin Web
- Node.js : **LTS ≥ 18.x**
- npm : fourni avec Node.js
- Vite : utilisé via `npm create vite@latest`
- React : **18.x**
- Tailwind CSS : **3.x** (`tailwindcss@3`)
- Dépendances front : `firebase` (SDK v9 modular), `react-router-dom`, `react-hook-form`, `chart.js` ou `recharts`, `axios`

### Outils CLI
- Firebase CLI : **latest** (`npm install -g firebase-tools`)
- Git LFS : **recommandé** pour gros fichiers (optionnel)

### Unity (MUST)
- **Unity 6.3 LTS (6000.3.2f1)** — version officielle du projet
- Android build modules : SDK, NDK, OpenJDK via Unity Hub
- Paramètres Android (Player Settings) :
  - **Scripting Backend** : IL2CPP
  - **Target Architectures** : ARM64
  - **Minimum API Level** : Android 8.0 (API 26)
  - **Target API Level** : Android 14 (API 34)
  - **Internet Access** : Require

### Packages Unity & librairies (versions compatibles)
- **Vuforia Engine** : **11.x** (ex. 11.4.x) — télécharger depuis developer.vuforia.com
- **UniGLTF** : v0.120+ (GitHub)
- **Firebase Unity SDK** : 11.x+ (Auth, Firestore, Storage, Analytics)
- (Option) **UniTask** pour async utilities

### Backend / Cloud
- Firebase (Firestore, Storage, Auth, Functions, Analytics)
- Node.js 18+ pour Cloud Functions (si utilisées)

---

## 3. Architecture système (schéma)

```
[Admin Web (React + Vite + Tailwind)]  <--HTTPS-->  [Firebase : Firestore / Storage / Auth / Functions]
         ^                                                      ^
         |                                                      |
[Developer/Admin Browser]                                 [Unity Mobile App (Android)]
         |                                                      |
         |----------------Realtime (Firestore listeners)--------|
```

---

## 4. Structure recommandée du dépôt

```
/README.md
/.gitignore
/.github/workflows/ci.yml
/unity-app/
  README_UNITY.md
  ProjectSettings/
  Assets/
  Packages/   (ne PAS committer les .tgz)
  Scripts/
  .gitignore
/admin-web/
  package.json
  vite.config.ts
  tailwind.config.cjs
  postcss.config.cjs
  /public
  /src
    /components
    /pages
    /services
    /styles
/infrastructure/
  firebase.json
  firestore.rules
  storage.rules
  functions/
    package.json
    index.js
/docs/
demo/
```

---

## 5. Installation & Lancement

### 5.1 Clonage du projet

Ouvrez un terminal et exécutez la commande suivante pour récupérer les sources :

```bash
git clone https://github.com/Garnel-Diffo/Portfolio-Professionnel-AR.git
cd Portfolio-Professionnel-AR
```

*(Optionnel) Si vous utilisez Git LFS :*
```bash
git lfs install
git lfs pull
```

### 5.2 Lancement Admin Web

Le panneau d'administration est une application React (Vite).

1.  Accédez au dossier :
    ```bash
    cd admin-web
    ```
2.  Installez les dépendances :
    ```bash
    npm install
    ```
3.  Lancez le serveur de développement :
    ```bash
    npm run dev
    ```
4.  Ouvrez votre navigateur à l'adresse indiquée (généralement `http://localhost:5173`).

### 5.3 Lancement Unity App

L'application mobile AR est développée sous Unity 6.3 LTS.

1.  Ouvrez **Unity Hub**.
2.  Cliquez sur **Add** (Ajouter) -> **Add project from disk**.
3.  Sélectionnez le dossier `unity-app` présent à la racine du dépôt cloné.
4.  Assurez-vous que la version d'Editor sélectionnée est bien **6000.3.2f1** (ou compatible 6.3 LTS).
5.  Ouvrez le projet.
6.  (Au premier lancement) Unity peut demander d'importer ou de résoudre des packages ; acceptez.

---

## 6. Configuration Unity 6.3 LTS — Guide détaillé

### 6.1 Ouvrir le projet
- Ouvrir **Unity Hub**
- Ajouter ou ouvrir le projet du dossier `unity-app/`

### 6.2 Installer modules Android via Unity Hub
- Unity Hub -> Installs -> (sur la version 6.3) -> Add Modules -> cocher :
  - Android Build Support
  - Android SDK & NDK Tools
  - OpenJDK

### 6.3 Paramétrer Player Settings (Android)
- Menu : **Edit > Project Settings...**
- Section : **Player** → choisir **Android**
- Sous **Other Settings** :
  - **Scripting Backend** : **IL2CPP**
  - **Target Architectures** : cocher **ARM64** (désactiver ARMv7)
  - **Minimum API Level** : **Android 8.0 (API 26)**
  - **Target API Level** : **Android 14 (API 34)**
  - **Internet Access** : **Require**
- Sous **Publishing Settings** : configurer keystore pour builds signés (ajouter keystore si disponible)

### 6.4 Vérifier le manifest Android si besoin (Plugins/Android/AndroidManifest.xml)
- S’assurer que les permissions INTERNET sont présentes si l’app en a besoin.

---

## 7. Packages Unity : installation & bonnes pratiques

### Vuforia Engine (11.x)
- Téléchargement : https://developer.vuforia.com/
- Import : Unity -> **Assets -> Import Package -> Custom Package...** -> sélectionner le `.unitypackage`
- Vérifier l’import et le menu **GameObject -> Vuforia Engine**

### UniGLTF
- Source : https://github.com/vrm-c/UniGLTF
- Import via `.unitypackage` ou copier les fichiers `Assets/UniGLTF`

### Firebase Unity SDK
- Télécharger depuis : https://firebase.google.com/download/unity
- Importer packages nécessaires (Auth, Firestore, Storage, Analytics)

**Ne pas committer** les packages `.tgz` ou `.unitypackage` dans Git ; conserver les packages localement et/ou documenter la procédure d’import.

---

## 8. Configuration Firebase — Règles et déploiement

### Initialiser Firebase CLI
```bash
npm install -g firebase-tools
firebase login
firebase init
```

### Déployer règles
```bash
firebase deploy --only firestore:rules,storage
```

### Exemple de règles (mettre dans `infrastructure/firestore.rules` et `storage.rules`)
- `firestore.rules` : lectures publiques, écritures contrôlées par Auth et claims.admin
- `storage.rules` : dossier `public/` (read public), dossier `private/` (admin only)

---

## 9. Schéma de données Firestore (exemples)
Collections : `portfolios`, `markers`, `contents`, `sessions`, `analytics_events`

Exemple `contents` :
```json
{
  "type":"model",
  "title":"Immeuble A",
  "modelUrl":"https://firebasestorage.googleapis.com/....glb",
  "thumbnailUrl":"https://...png",
  "meta":{"triangles":12000,"version":1},
  "createdAt":1660000000
}
```

---

## 10. Pipeline d'assets & optimisation

- Format : **GLB**
- Compression : **Draco**
- Textures : **Basis/KTX2** ou PNG/JPEG optimisés
- Résolutions : 1024 → 512
- Outils : Blender, `gltf-transform`, `gltfpack`, `BasisU`
- Workflow : Admin upload -> Cloud Function optimise -> Stockage public

---

## 11. CI / CD, sécurité & Git

### .gitignore conseillé (extrait)
- `unity-app/Library/`
- `unity-app/Temp/`
- `unity-app/Build/`
- `admin-web/node_modules/`
- `admin-web/dist/`
- `*.tgz`
- `.firebase/`

### Git LFS (optionnel)
- `git lfs install`
- `git lfs track "*.glb"`

### Sécurité
- NE PAS laisser Firestore en mode test en prod
- Utiliser Firebase Auth + custom claims pour admin

---

## 12. Release & builds (Android)

### Build steps
- File -> Build Settings -> select Android -> Switch Platform
- Add Scenes -> Build
- Configure keystore in Player Settings -> Publishing Settings
- Build -> Generate APK or AAB (AAB preferred for Play Store)

---

## 13. Annexes & liens officiels

- Unity : https://unity.com/
- Vuforia : https://developer.vuforia.com/
- UniGLTF : https://github.com/vrm-c/UniGLTF
- Firebase Unity SDK : https://firebase.google.com/download/unity
- Firebase CLI : https://firebase.google.com/docs/cli
- gltf-transform : https://gltf-transform.donmccurdy.com/


