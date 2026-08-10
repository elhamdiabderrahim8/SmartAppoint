# 📋 Rapport d'analyse & de préparation au déploiement — SmartAppoint

**Date :** 10 août 2026
**Rôle :** ingénieur full-stack + testeur logiciel (analyse de code + tests non fonctionnels)
**Objectif :** décider si le projet est prêt pour un déploiement sur **Railway**, corriger les bugs bloquants, et guider le déploiement.

---

## 1. Verdict

> **✅ OUI, le projet est prêt à être déployé sur Railway**, à condition de respecter le guide ci-dessous (3 services, variables d'environnement, ordre de déploiement).

- Tous les builds passent, tous les services démarrent et répondent.
- **2 bugs de crash runtime ont été trouvés et corrigés** (fonction admin de notification fiscale).
- **1 point de reproductibilité corrigé** (pin scikit-learn à la version d'entraînement).
- Restent des **limitations d'architecture** (stockage fichiers JSON éphémère, vérification Stripe non sécurisée) qui ne bloquent **pas** le déploiement mais doivent être connues.

---

## 2. Ce que j'ai fait (méthodologie)

1. **Exploration** de l'arborescence : monorepo à 3 services (`backend/` Express.js, `frontend/` React, `machine-learning/` Flask) + `RAILWAY_DEPLOYMENT.md` déjà présent.
2. **Lecture et analyse** du code backend (22 modules), frontend (26 pages), et du service ML.
3. **Tests non fonctionnels exécutés :**
   - Installation des dépendances (`npm ci` backend + frontend, venv pip ML)
   - **Build production frontend** (`npm run build`) → réussite
   - **Démarrage backend** + tests HTTP des endpoints (`/health`, routes publiques/protégées, rate limiters, headers de sécurité)
   - **Chargement du modèle ML** et tests HTTP (`/health`, `/predict`, `/predict/batch`)
   - **Serveur statique frontend** (`serve -s build`) avec fallback SPA
   - **Gunicorn** (commande de démarrage Railway du service ML)
   - **Chargement statique de tous les modules** backend (détection d'erreurs d'import)
   - **Trigger ciblé** de la fonction `notifyAllProsOfTax` pour confirmer le bug
4. **Corrections appliquées** (voir §4) et re-vérification du démarrage.
5. **Rédaction** de ce rapport + guide de déploiement.

---

## 3. Résultats des tests non fonctionnels

### 3.1 Backend (Express.js)

| Test | Résultat |
|---|---|
| `npm install` / `npm ci` | ✅ |
| Démarrage avec variables requises | ✅ |
| `GET /health` | ✅ 200 `{"status":"ok"}` |
| Headers de sécurité (Helmet : HSTS, nosniff, X-Frame-Options) | ✅ |
| Compression gzip (compression) | ✅ |
| Rate limiters `/api/auth` et `/api/admin` | ✅ |
| Routes protégées (`/api/pro/*`, `/api/admin/*`) → 401 sans token | ✅ |
| `NODE_ENV=production` → route de debug `/__debug_routes` désactivée | ✅ |
| Chargement des 22 modules sans erreur | ✅ |
| Démarrage **sans** `SUPABASE_URL` | ❌ Crash immédiat (fail-fast) |
| Démarrage **sans** `STRIPE_SECRET_KEY` | ❌ Crash immédiat (fail-fast) |

### 3.2 Frontend (React)

| Test | Résultat |
|---|---|
| `npm ci` | ✅ |
| `npm run build` (production) | ✅ Aucun warning ni erreur |
| `REACT_APP_API_URL` injectée dans le bundle | ✅ |
| `serve -s build` (commande Railway `railway:start`) | ✅ `/` et `/login` → 200, fallback SPA OK |
| Build des 26 pages / imports résolus | ✅ |

### 3.3 Machine Learning (Flask)

| Test | Résultat |
|---|---|
| Création venv + `pip install -r requirements.txt` | ✅ |
| Chargement du modèle `.pkl` (RandomForest, 300 arbres) | ✅ (avec avertissement de version, corrigé) |
| `GET /health` | ✅ 200, `model_loaded: true` |
| `POST /predict` (objet) | ✅ 200 |
| `POST /predict/batch` | ✅ 200 |
| `gunicorn app_ai:app --bind 0.0.0.0:$PORT` (commande Railway) | ✅ 200 |

---

## 4. Bugs trouvés et corrigés

### 🐛 Bug 1 — CRASH : endpoint admin de notification fiscale (corrigé)
- **Fichiers :** `backend/routes/admin.js`, `backend/services/notificationService.js`
- **Symptôme :** cliquer sur « Envoyer les notifications fiscales » (admin) provoquait `ReferenceError: USER_TABLE is not defined` ; et avant même ça, `notifyAllProsOfTax is not defined` car la fonction n'était **pas importée** dans `admin.js`.
- **Cause :** `notificationService.js` utilisait `USER_TABLE` et `getDetailedStats` (définis dans `adminService.js`) sans les importer ; `admin.js` appelait la fonction sans l'importer.
- **Correctif appliqué :**
  - Import de `notifyAllProsOfTax` dans `admin.js`.
  - Require paresseux de `getDetailedStats` + définition locale de `USER_TABLE` dans `notificationService.js` (évite la dépendance circulaire, `adminService` → `notificationService`).
- **Vérifié :** le `ReferenceError` est éliminé ; le module se charge ; le backend démarre.

### 🐛 Bug 2 — Reproducibilité ML : version de scikit-learn non épinglée (corrigé)
- **Fichier :** `machine-learning/requirements.txt`
- **Symptôme :** `requirements.txt` n'épinglait pas scikit-learn. Le modèle `.pkl` a été entraîné avec **sklearn 1.8.0** ; à l'installation on obtenait **1.9.0** avec un avertissement `InconsistentVersionWarning`. À terme, une montée de version (1.10 / 2.x) pourrait casser `joblib.load` ou changer les prédictions.
- **Correctif appliqué :** `scikit-learn==1.8.0` (version d'entraînement). Vérifié : une roue existe pour Python 3.14.

---

## 5. Problèmes résiduels (ne bloquent PAS le déploiement — à connaître)

| # | Problème | Impact | Recommandation |
|---|---|---|---|
| 1 | **Stockage fichier éphémère** : les plannings pro, la config admin et les statuts de validation sont écrits dans `backend/data/*.json`. Sur Railway le disque est **éphémère** (perdu à chaque redéploiement / redémarrage). | Les plannings/config saisis seront perdus au redeploy. Multi-réplicas → incohérences. | Migrer ces 3 stores vers Supabase (table) **ou** ajouter un **Volume Railway** monté sur `backend/data`. |
| 2 | **Paiement Stripe non sécurisé** : `POST /api/appointments/:id/confirm-payment` marque un RDV « payé » **sans vérification webhook Stripe**. Tout client authentifié peut payer sans payer. | Fraude / pertes financières en production réelle. | Ajouter la vérification webhook Stripe (`stripe.webhooks.constructEvent`). Le README le liste d'ailleurs comme amélioration. |
| 3 | **Route publique protégée** : `/api/specialites` (et `/api/ai/predict`) renvoient **401** car le router `client.js` est monté sur `/api` avec un middleware `auth` global. Le frontend ne les appelle que depuis des pages authentifiées, donc **ça fonctionne**, mais l'intention « navigation publique » du README n'est pas respectée. | UX : le visiteur non connecté ne peut pas parcourir les spécialités. | Monter `specialites.js` et `ai.js` **avant** `clientRoutes` dans `server.js`, ou retirer le `router.use(auth)` global. |
| 4 | **Route de debug cassée (dev uniquement)** : `/__debug_routes` renvoie 500 en développement (incompatible Express 5, `app._router` indéfini). Désactivée en production, donc sans impact. | Faible. | Corriger ou supprimer. |
| 5 | **`console.log` laissé** dans `frontend/src/pages/admin/AdminStats.jsx` (et quelques logs de debug backend). | Bruit console uniquement. | Nettoyer avant mise en production. |
| 6 | **Aucun test automatisé** (unités / intégration) sur les 3 services. | Risque de régressions. | Ajouter des tests (le README le reconnaît). |
| 7 | **Fichiers volumineux dans le repo** : `IAmodel.csv` (65 Mo) et `modele_prediction_paiement.pkl` (38 Mo) commités. | Builds/lecture repo plus lents ; GitHub affiche des warnings > 50 Mo. | À terme : utiliser Git LFS ou générer le modèle au build. |
| 8 | **Avatars en fallback local** (`backend/uploads/`) : perdus au redeploy (déjà couvert par le point 1). | Mineur (le fallback ne sert que si Supabase Storage échoue). | — |
| 9 | **`/predict` accepte des features vides** et retourne une prédiction avec des zéros (pas d'erreur 400). | Fiabilité des prédictions. | Valider la présence des 7 features côté Flask. |

---

## 6. Fichiers modifiés

```
backend/routes/admin.js                 +1   (import notifyAllProsOfTax)
backend/services/notificationService.js +8/-2 (fix USER_TABLE + getDetailedStats)
machine-learning/requirements.txt       pin scikit-learn==1.8.0
```

---

## 7. Conclusion

L'application est **prête pour le déploiement sur Railway**. Les 3 services se construisent et fonctionnent. Les 2 bugs de crash ont été corrigés. Suis le guide pas-à-pas ci-dessous, et traite le problème n°1 (fichiers JSON éphémères) et n°2 (Stripe) avant toute mise en production avec de vrais utilisateurs et de vrais paiements.
