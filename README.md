# AutoWeb Studio — Backend minimal (P2)

Backend Node.js/Express qui complète le fichier HTML `AutoWeb_Studio_v5.0_corrige_complet.html`.
Il apporte tout ce qu'un fichier unique ne peut pas faire : clé API protégée, stockage fiable des projets, publication/hébergement réel des sites, réception des formulaires de contact et comptage d'usage.

---

## 1. Schéma d'architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│  NAVIGATEUR (fichier HTML unique — AutoWeb Studio admin)            │
│  · Édition du site · Aperçu · Undo/Redo · Autosave (localStorage)   │
└───────────────┬───────────────────────────────────┬─────────────────┘
                │ POST /api/gemini (contenu/clonage)│ POST /api/publish (HTML)
                │ POST /api/projects (CRUD)         │ POST /api/contact
                ▼                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│  SERVEUR Node.js + Express (server.js)                              │
│  · Proxy Gemini : la clé API reste côté serveur (variable env)      │
│  · Rate limiting par IP · Token admin pour les écritures            │
└───────┬──────────────────┬───────────────────────┬──────────────────┘
        │                  │                       │
        ▼                  ▼                       ▼
┌──────────────┐  ┌──────────────────┐  ┌────────────────────────────┐
│ data/db.json │  │ public/s/:slug/  │  │ API Google Gemini          │
│ projets +    │  │ index.html       │  │ (gemini-3-flash-preview)   │
│ contacts +   │  │ → sites publiés  │  └────────────────────────────┘
│ compteur     │  │   servis en      │
│ usage IA     │  │   GET /s/:slug   │
└──────────────┘  └──────────────────┘
```

Fichiers :
- `server.js` — toute la logique (1 fichier, ~180 lignes)
- `package.json` — dépendance unique : `express`
- `.env.example` — variables à configurer
- `data/` — créé automatiquement (JSON, remplaçable par SQLite/Postgres en production)
- `public/s/` — sites publiés (HTML statique)

---

## 2. Installation

```bash
cd autoweb-backend
npm install
cp .env.example .env   # puis éditez .env avec votre clé Gemini et un token admin
npm start              # http://localhost:3000
```

Prérequis : Node.js ≥ 18 (fetch natif).

---

## 3. Points de terminaison (API)

| Méthode | Route | Rôle | Sécurité |
|---|---|---|---|
| POST | `/api/gemini` | Proxy Gemini : `{ contents, generationConfig?, model? }` | rate limit 30/min/IP |
| GET | `/api/projects` | Liste des projets (id, nom, date) | public |
| POST | `/api/projects` | Créer/mettre à jour un projet `{ id?, name, data }` | `X-Admin-Token` |
| GET | `/api/projects/:id` | Charger un projet | public |
| DELETE | `/api/projects/:id` | Supprimer un projet | `X-Admin-Token` |
| POST | `/api/publish` | Publier un site `{ slug?, html }` → `{ url }` | `X-Admin-Token` + rate limit |
| GET | `/s/:slug` | Consulter le site publié (statique) | public |
| POST | `/api/contact` | Réception formulaire `{ name, email, message, site? }` | rate limit 10/min/IP |
| GET | `/api/usage` | Compteurs : appels Gemini, projets, contacts | public |

Toutes les réponses sont en JSON. En production, remplacez `Access-Control-Allow-Origin: *` par votre domaine exact.

---

## 4. Brancher le frontend HTML sur ce backend

Dans le fichier HTML, trois adaptations suffisent (recherchez les fonctions correspondantes) :

1. **Clé Gemini** — remplacez l'appel direct à `generativelanguage.googleapis.com` (dans `triggerAiSiteCloning`, `triggerFullAiGeneration`, `handleAiPromptAdjustment`) par :
   ```js
   const API = 'https://votre-domaine.com'; // ou http://localhost:3000 en dev
   const resp = await fetch(API + '/api/gemini', {
     method:'POST', headers:{'Content-Type':'application/json'},
     body: JSON.stringify({ contents:[{ parts:[{ text: prompt }] }], generationConfig:{ responseMimeType:'application/json' } })
   });
   ```
   La clé n'est plus jamais exposée au navigateur.

2. **Projets** — remplacez le stockage localStorage des projets par `GET/POST/DELETE /api/projects` (avec l'en-tête `X-Admin-Token`).

3. **Publication** — au lieu du simple téléchargement, ajoutez un bouton « Publier » qui envoie le HTML généré par `openExportModal` à `POST /api/publish` et affiche l'URL retournée (`/s/:slug`).

4. **Formulaire de contact** — dans `handleContactSubmit`, utilisez `POST /api/contact` au lieu de Formspree/mailto.

---

## 5. Déploiement

**Option simple (PaaS gratuit/peu cher) :** Render, Railway, Fly.io ou Clever Cloud. Poussez ce dossier, définissez les variables d'environnement (`GEMINI_API_KEY`, `ADMIN_TOKEN`), le port est fourni automatiquement.

**Option VPS (ex. 5€/mois) :** Ubuntu + Node.js + Caddy/Nginx en reverse-proxy avec HTTPS. Le stockage JSON convient pour quelques milliers de projets ; au-delà, migrez vers SQLite (`better-sqlite3`) ou Postgres.

**Évolutions recommandées :**
- Authentification utilisateurs réelle (JWT) au lieu du token partagé
- Stockage des images uploadées sur S3/Cloudflare R2 (évite le base64)
- Domaines personnalisés pour les sites publiés
- Webhook d'email (Resend/Postmark) pour les contacts

---

## 6. Vérification rapide

```bash
curl http://localhost:3000/api/usage
# → {"geminiCalls":0,"projects":0,"contacts":0}
```
# atoweb
