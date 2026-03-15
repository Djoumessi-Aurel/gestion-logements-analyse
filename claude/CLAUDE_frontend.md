# Projet : Gestion de Logements à Louer – Frontend

## Stack technique
- **Framework** : Next.js 15 TypeScript – App Router (jamais Pages Router)
- **État global** : Redux Toolkit + redux-persist chiffré (CryptoJS)
- **Styles** : TailwindCSS (utilitaires) + SCSS (cas complexes)
- **Composants UI** : PrimeReact (composant par défaut pour tout) + PrimeIcons
- **Client HTTP** : Axios avec intercepteurs JWT (refresh auto sur 401)
- **Graphiques** : PrimeReact Charts ou recharts
- **Validation formulaires** : react-hook-form + zod (ou yup)

## URL Backend
```
NEXT_PUBLIC_API_URL=http://localhost:3000   ← à adapter selon l'environnement
```

## Couleurs principales (charte graphique)
- Bleu principal : `#1e3a8a`
- Bleu moyen : `#3b82f6`
- Bleu clair (fond) : `#dbeafe`
- Blanc : `#ffffff`
- Texte sombre : `#1e293b`
- Succès : `#166534` / fond `#dcfce7`
- Erreur : `#991b1b` / fond `#fee2e2`
- Avertissement : `#92400e` / fond `#fef3c7`

## Structure des dossiers (App Router)
```
app/
  layout.tsx                  ← Layout racine (Redux Provider, thème)
  (auth)/
    login/page.tsx
    layout.tsx                ← Layout public (sans sidebar)
  (dashboard)/
    layout.tsx                ← Layout protégé (sidebar, header, session expirée)
    page.tsx                  ← Dashboard principal
    batiments/
    logements/
    occupations/
    paiements/
    locataires/
    utilisateurs/
    export/
middleware.ts                 ← Protection des routes (vérif JWT + rôle)
store/                        ← Slices Redux (auth, ui, ...)
services/                     ← Wrappers Axios par entité
components/                   ← Composants réutilisables
hooks/                        ← Hooks custom
```

## Conventions de code
- **Composants / Pages** : PascalCase (ex : `OccupationCard.tsx`)
- **Hooks** : camelCase préfixé `use` (ex : `useArrieres.ts`)
- **Services API** : kebab-case (ex : `occupations.api.ts`)
- **Types / Interfaces** : PascalCase (ex : `CreatePaiementDto`)
- **Variables / fonctions** : camelCase

## Règles UX impératives (TOUJOURS respecter)

### Feedback visuel
- Toute action déclenchant une requête API → afficher un spinner ou skeleton pendant le chargement
- Succès → toast/notification visible 3-5 secondes
- Erreur → message clair indiquant ce qui a échoué (exploiter le champ `message` de la réponse API)
- Erreur de validation → message sous le champ concerné, formulaire non soumis

### Gestion du token JWT
- Stocker l'access_token dans Redux (chiffré via CryptoJS dans redux-persist)
- **Jamais** stocker le token directement dans localStorage en clair
- Refresh token : Cookie HttpOnly (géré par le backend)
- Intercepteur Axios : relancer automatiquement la requête après refresh sur 401
- Si refresh expiré → afficher modal/toast "Session expirée" + redirect /login

### Formulaire Paiement (règle critique)
- Option 1 (nombre de loyers) : afficher en **temps réel** (avant validation) :
  - `fin_periode` calculée
  - `montant_paye` calculé
- Toggle entre Option 1 et Option 2 dans le même formulaire
- Option 2 : validation `fin_periode >= debut_periode` côté client

### RBAC côté client
- Décoder le JWT pour extraire le rôle
- Afficher **uniquement** les boutons/menus autorisés selon le rôle
- Le middleware.ts vérifie la présence et la validité du JWT pour chaque route protégée
- Ne pas faire confiance uniquement au frontend : le backend est la source de vérité

### Tri et filtrage
- Activer tri et filtrage sur **toutes** les listes de données (DataTable PrimeReact)

### Responsive
- Mobile (375px), Tablette (768px), Desktop (1280px+) — toujours vérifier les trois
- Sidebar : repliée sur mobile/tablette, déployée sur desktop

## Rôles utilisateur
```
LOCATAIRE | ADMIN_LOGEMENT | ADMIN_BATIMENT | ADMIN_GLOBAL
```

### Périmètre d'accès par rôle (pour le routing et l'affichage conditionnel)
- **LOCATAIRE** : lecture seule, uniquement ses propres occupations et paiements
- **ADMIN_LOGEMENT** : gestion des logements attribués (pas CRUD logement lui-même)
- **ADMIN_BATIMENT** : CRUD logements pour ses bâtiments + capacités ADMIN_LOGEMENT
- **ADMIN_GLOBAL** : accès total sans restriction

## Affichage Locataire (vue espace personnel)
- Si arriéré → badge rouge + montant dû + période due
- Si à jour → badge vert + date attendue prochain paiement (= date_dernier_jour_couvert + 1 jour)
- Tout en lecture seule, aucun bouton de modification

## Composants partagés à créer (Phase F5.7)
- `PageHeader` : titre + breadcrumb + bouton d'action contextuel
- `DataTableWrapper` : PrimeReact DataTable avec tri/filtre/pagination préconfigurés
- `StatusBadge` : badge coloré (Occupé/Libre, À jour/Arriéré, Actif/Inactif)
- `ConfirmDialog` : confirmation avant suppression
- `LoadingSpinner` / `SkeletonLoader`
- `ErrorMessage` : affichage erreurs API
- `FileUploader` : upload avec prévisualisation + validation MIME/taille
- `ExportModal` : choix format (Excel/PDF) + période + périmètre + bouton télécharger

## Export (téléchargement côté navigateur)
```typescript
// Pattern à utiliser pour déclencher le téléchargement d'un Blob
const response = await apiClient.get('/export/paiements', {
  params: { format: 'excel', dateDebut, dateFin },
  responseType: 'blob'
});
const url = URL.createObjectURL(new Blob([response.data]));
const link = document.createElement('a');
link.href = url;
link.download = `paiements_${date}.xlsx`;
link.click();
URL.revokeObjectURL(url);
```

## Variables d'environnement requises (.env.local)
```
NEXT_PUBLIC_API_URL=http://localhost:3000
NEXT_PUBLIC_CRYPTO_SECRET=...    ← clé de chiffrement redux-persist
```

---

## Plan de réalisation – Étapes Frontend

### Phase 0 – Initialisation (partie frontend)
- P0.1 : Créer repo Git frontend + README + .gitignore
- P0.3 : Initialiser projet Next.js 15 TypeScript (`create-next-app@latest` – App Router, TailwindCSS)
- P0.4 : Configurer variables d'environnement (.env.local + .env.example documenté)
- P0.7 : Installer toutes les dépendances frontend

### Phase 5 – Fondations Frontend
- F5.1 : Redux Toolkit + redux-persist chiffré (CryptoJS) + Provider dans app/layout.tsx
- F5.2 : apiClient Axios + intercepteur request (Bearer token) + intercepteur response (refresh auto sur 401)
- F5.3 : Services API frontend typés (un fichier par entité dans services/)
- F5.4 : middleware.ts – protection des routes + vérification rôle JWT
- F5.5 : Layouts (racine, (auth), (dashboard) avec sidebar + header + notification session expirée)
- F5.6 : Page Login – formulaire, appel /auth/login, redirect selon rôle, messages d'erreur
- F5.7 : Composants partagés de base (PageHeader, DataTableWrapper, StatusBadge, ConfirmDialog, LoadingSpinner, ErrorMessage, FileUploader, ExportModal)

### Phase 6 – Pages métier (CRUD)
- F6.1 : Pages CRUD Bâtiments (liste, création/modification modal, suppression avec confirm)
- F6.2 : Pages CRUD Logements + historique loyers + formulaire ajout loyer
- F6.3 : Pages CRUD Locataires (avec badge arriéré dans la liste)
- F6.4 : Pages CRUD Occupations (vérif logement libre, blocage si paiements, upload contrat, mise en fin)
- F6.5 : Formulaire Paiement – Options 1 & 2 avec calcul temps réel, toggle, upload preuves
- F6.6 : Pages CRUD Utilisateurs + gestion rôles + attribution logements/bâtiments + activation/désactivation
- F6.7 : Vue espace Locataire (lecture seule, arriéré ou prochain paiement, historique paiements)

### Phase 7 – Dashboards & Export
- F7.1 : Dashboard principal – KPIs globaux + liste logements triée par date_dernier_jour_couvert avec code couleur
- F7.2 : Dashboard Bâtiment – graphique loyers perçus + donut occupé/libre + liste arriérés
- F7.3 : Dashboard Logement – statut actuel + timeline occupations + historique paiements
- F7.4 : Dashboard Locataire – montant total, nb loyers, assiduité, arriérés actuels
- F7.5 : Composant ExportModal intégré sur toutes les pages liste et dashboards + téléchargement Blob
- F7.6 : Responsive design & polish UI (vérif 375px / 768px / 1280px+, cohérence visuelle)

### Phase 8 – Intégration & Finalisation
- I8.1 : Tests E2E flux critiques (login → logement → occupation → paiement → export)
- I8.2 : Vérification règles métier RG-01 à RG-12 (messages d'erreur côté frontend)
- I8.3 : Tests RBAC – vérifier que chaque rôle ne voit que son périmètre
- I8.5 : Vérification logs et gestion des erreurs non gérées (ErrorBoundary React)
- I8.6 : Configuration déploiement (Dockerfile frontend, variables d'env de prod)

---

## Étape en cours
<!-- Mettre à jour après chaque étape validée -->
P0.1
