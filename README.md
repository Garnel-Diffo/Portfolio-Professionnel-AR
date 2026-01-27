
# Portfolio Professionnel AR — MVP

**Résumé**  
Projet : Portfolio Professionnel en Réalité Augmentée (AR).  
Objectif : transformer une carte de visite / document physique en portail vers une expérience AR (modèles 3D PBR, vidéos, texte), pilotable en temps réel via un dashboard web avec analytics.

---

## Table des matières
1. Contexte & objectifs
2. Architecture système (diagramme)
3. Arborescence GitHub (premier push)
4. Quickstart — Prérequis & Installation
5. Configuration Firebase (Firestore, Storage, Auth, Functions)
6. Structure des données (Firestore)
7. Analytics & événements
8. Mode Présentation / Contrôle à distance
9. Pipeline d'assets & optimisation
10. Règles Firestore & Storage (exemples)
11. CI / CD & bonnes pratiques
12. Release & builds
13. Contribution
14. Licence

---

## 1. Contexte & objectifs
Ce dépôt contient le code source du MVP du **Portfolio Professionnel AR** :
- Application mobile (Unity) : scan d'image (carte) → rendu AR (GLB/glTF, animations).
- Admin Web (Next.js / React) : gestion des portfolios, markers, contenus, analytics.
- Backend léger : Firebase (Firestore, Storage, Auth, Functions, Analytics).

Objectifs du MVP :
- Scan d'une carte → affichage d'un modèle 3D (.glb) avec UI basique.
- Admin web pour uploader modèles & lier à un marker.
- Tracking analytics des scans et vues.
- Mode présentation (remote control via Firestore sessions).

---

## 2. Architecture système (diagramme)

```
+-----------------+       HTTPS/Realtime       +------------------+
|  Admin Web (UI) | <------------------------> |  Firebase (Auth) |
|  React.js       |        Firestore / Storage |  Firestore       |
+-----------------+                            |  Storage         |
       |                                       |  Functions       |
       |                                       +------------------+
       |                                               ^
       |                                               |
       v                                               |
+-----------------+     HTTP(S) / Storage URLs        +-----------------+
| Mobile AR App   |<--------------------------------> | Firebase Storage|
| Unity + Vuforia |          (download models)        | (GLB, thumbs)   |
+-----------------+                                   +-----------------+
       ^
       |
 Firestore snapshot listeners (presentation control / session)
```

Explication :  
- L'admin met à jour Firestore et upload les modèles dans Storage.  
- L'application Unity interroge Firestore pour récupérer la configuration d'un marker et télécharge les assets depuis Storage.  
- Les analytics sont envoyées à Firebase Analytics et/ou Firestore (événements bruts puis agrégations).

---

## 3. Arborescence GitHub (premier push)
Structure recommandée pour le repo (monorepo simple) :

```
/README.md
/.gitignore
/.github/
  workflows/
    ci.yml
/unity-app/
  .gitignore
  README_UNITY.md
  ProjectSettings/
  Assets/
  Packages/
  README_DEV_NOTES.md
/admin-web/
  package.json
  next.config.js
  /public
  /src
    /components
    /pages
    /lib
    /styles
  README_WEB.md
/infrastructure/
  firebase.json
  firestore.rules
  storage.rules
  functions/
    package.json
    index.js
/docs/
  architecture.md
  assets_pipeline.md
/demo/
  demo_video.mp4
  slides.pdf
```

Fichiers de démarrage (premier commit) :
- README.md (celui-ci)
- .gitignore (racine)
- /admin-web/ initial Next.js scaffold (ou un README expliquant comment l'initialiser)
- /unity-app/ dossier initial avec README_UNITY.md expliquant la configuration Unity et les packages nécessaires
- /infrastructure/firestore.rules et storage.rules

---

## 4. Quickstart — Prérequis & Installation

### Prérequis locaux
- Node.js LTS (>=18 recommandé)
- Yarn ou npm
- Unity 2020/2021 LTS (ou version compatible), avec modules Android et iOS si builds mobiles
- Vuforia Engine (ou ARFoundation + ARCore/ARKit selon choix)
- Firebase CLI (`npm install -g firebase-tools`)
- Git

### Cloner le repo
```bash
git clone git@github.com:TON_COMPTE/portfolio-ar.git
cd portfolio-ar
```

### Installer l'admin web (exemple Next.js)
```bash
cd admin-web
npm install
# ou
yarn install
```

### Lancer l'admin en dev
```bash
npm run dev
# ou
yarn dev
# puis visiter http://localhost:3000
```

### Unity — notes rapides
- Ouvrir `/unity-app` dans Unity Hub.
- Installer Vuforia (ou ARFoundation) via Package Manager.
- Importer UniGLTF ou GLTFUtility pour charger les GLB runtime.
- Ajouter Firebase SDK for Unity (Auth, Firestore, Storage analytics).

---

## 5. Configuration Firebase (pratique)
1. Créer un projet Firebase dans la console Firebase.
2. Activer Firestore (mode "test" pour dev, **changer en production** avant déploiement).
3. Activer Firebase Storage.
4. Activer Firebase Auth (Google sign-in et Email/Password pour admin).
5. Activer Firebase Analytics.
6. Installer Firebase CLI et initialiser :
```bash
firebase login
firebase init
# choisir Firestore, Functions (optionnel), Hosting
```

7. Déployer règles (après vérif) :
```bash
firebase deploy --only firestore:rules,storage
```

---

## 6. Structure des données (Firestore) — exemples (MVP)
Collections principales :
- `/portfolios/{portfolioId}`
- `/markers/{markerId}`
- `/contents/{contentId}`
- `/sessions/{sessionId}` (pour présentation)
- `analytics_events` (option pour analytics bruts)

Exemple `markers/marker_001` :
```json
{
  "name": "Carte - Architecte",
  "imageTargetUrl": "https://.../marker_001.jpg",
  "contents": ["c_001","c_002"],
  "defaultContentId": "c_001",
  "createdAt": 167xxx,
  "ownerId": "uid_admin"
}
```

Exemple `contents/c_001` :
```json
{
  "type": "model",
  "title": "Immeuble A",
  "modelUrl": "https://firebasestorage.googleapis.com/.../immeuble_a_v1.glb",
  "thumbnailUrl": "https://.../thumbs/immeuble_a.png",
  "meta": {
    "triangles": 15000,
    "textures": ["albedo.jpg","normal.jpg"],
    "version": 1
  },
  "createdAt": 167xxx
}
```

---

## 7. Analytics & événements (noms recommandés)
- `portfolio_scan` : { markerId, portfolioId }
- `content_view` : { contentId, duration }
- `model_loaded` : { contentId, loadTimeMs }
- `share_action` : { contentId, platform }
- `presentation_command` : { sessionId, command }

Ces events peuvent être envoyés à Firebase Analytics et/ou stockés dans `analytics_events` pour agrégation.

---

## 8. Mode Présentation / Contrôle à distance
Utilisation : admin crée une `session` dans Firestore :
`/sessions/{sessionId}` :
```json
{
  "markerId":"marker_001",
  "presenterId":"uid_admin",
  "command":"idle", // next, prev, play, pause, goto:contentId
  "updatedAt": 167xxx
}
```
L'app Unity écoute ce document et applique la commande à la scène AR.

---

## 9. Pipeline d'assets & optimisation (MVP)
- Formats : GLB (glTF binaire)
- Compression : Draco (mesh), KTX2/Basis pour textures
- Textures : max 2048 → idéal 1024/512 pour mobile
- LOD : prévoir 2 niveaux (high/low)
- Outils : Blender, gltf-pack, gltf-transform (`gltf-transform`, `gltfpack`), BasisU

Processus recommandé à l'upload :
1. Upload glb via admin
2. Cloud Function `onModelUpload` : vérifie taille, génère thumbnail, optimise (optionnel)
3. Mettre à jour `contents` doc avec `modelUrl` et `thumbnailUrl`

---

## 10. Règles Firestore & Storage (exemples)
> **Attention** : ces règles sont des exemples. Adapte-les selon ton modèle d'auth & ownership.

**firestore.rules**
```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    // Portfolios: public read, write only owner
    match /portfolios/{portfolioId} {
      allow read: if true;
      allow write: if request.auth != null && request.auth.uid == request.resource.data.ownerId;
    }
    // Markers: public read, write for authenticated users (admin)
    match /markers/{markerId} {
      allow read: if true;
      allow write: if request.auth != null && request.auth.token.admin == true;
    }
    // Contents: public read, write for admins
    match /contents/{contentId} {
      allow read: if true;
      allow write: if request.auth != null && request.auth.token.admin == true;
    }
    // Sessions: read public, write by authenticated presenter
    match /sessions/{sessionId} {
      allow read: if true;
      allow write: if request.auth != null;
    }
    // Analytics: write allowed (client), read only admin
    match /analytics_events/{eventId} {
      allow create: if true;
      allow read: if request.auth != null && request.auth.token.admin == true;
    }
  }
}
```

**storage.rules**
```javascript
rules_version = '2';
service firebase.storage {
  match /b/{bucket}/o {
    // Public read for assets (models, thumbs)
    match /public/{allPaths=**} {
      allow read: if true;
      allow write: if request.auth != null && request.auth.token.admin == true;
    }
    // Private uploads (if needed)
    match /private/{allPaths=**} {
      allow read, write: if request.auth != null && request.auth.token.admin == true;
    }
  }
}
```

---

## 11. CI / CD & bonnes pratiques
- Linting : ESLint pour admin, Prettier pour format.  
- Tests : Jest (admin).  
- GH Actions : build & deploy admin on push to `main` branch.  
- Unity : stocker `Library` dans `.gitignore`. Utiliser LFS si vous stockez assets lourds.  
- Versionner les assets GLB via Storage bucket (pas dans Git).

---

## 12. Release & builds
- Android : build APK -> signer (keystore) -> upload au Play Store (si production).  
- iOS : build Xcode archive -> code signing -> TestFlight / App Store. (iOS recommandé après MVP).  
- Admin : déployer sur Vercel ou Firebase Hosting.

---

## 13. Contribution
- Workflow : `feature/*` branches; PR -> review -> merge to `develop` -> release via `main`.  
- Templates : ISSUE_TEMPLATE.md / PR_TEMPLATE.md (à ajouter).

---

## 14. Licence
- Par défaut : MIT. Adapter selon besoin.

---

## Fichiers fournis dans ce repo
- `README.md` (ce document)
- `infrastructure/firestore.rules` (exemple)
- `infrastructure/storage.rules` (exemple)
- `admin-web/` : scaffold Next.js (à initialiser)
- `unity-app/` : README_UNITY.md pour instructions Unity

---

Bonne base pour commencer l'implémentation.  
Si tu veux, je peux maintenant :
1. Générer les issues GitHub (backlog) prêtes à être importées ;  
2. Préparer les snippets de code pour l'admin (upload glb -> storage -> create content doc) ;  
3. Générer la fonction Cloud Function `onModelUpload` en Node.js ;  
4. Préparer README_UNITY.md détaillé pour la configuration Unity + packages.

Dis ce que tu veux que je fasse **tout de suite**.
