# Projet : Gestion de Logements à Louer – Backend

## Stack technique
- **Framework** : NestJS TypeScript
- **ORM** : TypeORM (multi-driver — ne jamais hardcoder MySQL dans la logique métier)
- **Base de données** : MySQL (configurable via .env)
- **Auth** : @nestjs/passport, passport-jwt, @nestjs/jwt — guards RBAC
- **Validation** : class-validator + class-transformer — ValidationPipe global (whitelist:true, transform:true, forbidNonWhitelisted:true)
- **Documentation API** : @nestjs/swagger — Swagger accessible sur /api/docs
- **Planificateur** : @nestjs/schedule (routine notifications)
- **Email** : Nodemailer (SMTP configurable via .env)
- **Export** : exceljs (Excel), pdfkit (PDF)
- **Upload fichiers** : Multer
- **Sécurité** : bcrypt (hash mdp), Helmet, @nestjs/throttler (rate limiting)
- **Dates** : date-fns (obligatoire pour tous les calculs de dates)
- **Logging** : Logger NestJS natif (console + fichier en prod)

## Architecture des modules
Un module par domaine fonctionnel :
- AuthModule, UsersModule, BatimentsModule, LogementsModule
- LocatairesModule, OccupationsModule, PaiementsModule
- FilesModule, ExportModule, NotificationsModule, DashboardModule, CommonModule

Chaque module = `controller` + `service` + `module.ts` + dossier `dto/` + dossier `entities/`

## Conventions de code
- **Colonnes SQL / attributs entités** : snake_case (ex : `date_dernier_jour_couvert`)
- **Variables / fonctions TypeScript** : camelCase (ex : `getArrieres()`)
- **Classes / Interfaces** : PascalCase (ex : `OccupationService`, `CreatePaiementDto`)
- **Constantes** : UPPER_SNAKE_CASE (ex : `PERIODE_TYPES`)
- **Fichiers** : kebab-case (ex : `occupation.service.ts`, `create-paiement.dto.ts`)
- **Endpoints REST** : kebab-case (ex : `/reset-password`, `/admin-logement`)

## Structure des réponses API (toujours respecter)
```typescript
// Succès
{ statusCode: 200, message: "Opération réussie", data: { ... } }

// Erreur de validation
{ statusCode: 400, message: "Validation échouée", errors: [{ field: "montant", message: "doit être > 0" }] }

// Erreur métier
{ statusCode: 422, message: "Ce logement est actuellement occupé" }
```
Un `AllExceptionsFilter` global doit intercepter toutes les erreurs non gérées.

## Rôles utilisateur (Enum Role)
```
LOCATAIRE | ADMIN_LOGEMENT | ADMIN_BATIMENT | ADMIN_GLOBAL
```

## Règles métier critiques (NE JAMAIS dévier de ces règles)

### addPeriodToDate(date: Date, periode: Periode): Date
Utiliser date-fns. Ajouter à `date` le nombre d'unités de `periode` :
- JOUR → addDays(date, periode.nombre)
- SEMAINE → addWeeks(date, periode.nombre)
- MOIS → addMonths(date, periode.nombre) ← gérer les fins de mois
- ANNEE → addYears(date, periode.nombre)

### getLoyer(date: Date): Loyer
Parmi les Loyers du Logement dont `date_debut <= date`, retourner celui dont `date_debut` est la plus grande.

### isOccupied(logementId): boolean
Retourne true s'il existe une Occupation avec `date_fin = null` pour ce Logement.

### getArrieres(date?: Date): Arriere | undefined
1. date = date du jour si non fournie
2. Si date <= date_dernier_jour_couvert → retourner undefined (pas d'arriéré)
3. debut_periode_due = date_dernier_jour_couvert + 1 jour
4. loyer = logement.getLoyer(date)
5. fin_periode_due = addPeriodToDate(date_dernier_jour_couvert, loyer.periode), nombre_loyers_du = 1
6. Tant que fin_periode_due < date → fin_periode_due = addPeriodToDate(fin_periode_due, loyer.periode), nombre_loyers_du++
7. montant_du = nombre_loyers_du × loyer.montant
8. Retourner { debut_periode_due, fin_periode_due, montant_du, nombre_loyers_du }

### Paiement Option 1 (par nombre de loyers)
```
debut_periode = occupation.date_dernier_jour_couvert + 1 jour
fin_periode   = addPeriodToDate(debut_periode, { nombre: nombre_de_loyers * loyer.periode_nombre, type: loyer.periode_type }) - 1 jour
montant_paye  = nombre_de_loyers * occupation.getLogement().getLoyer(debut_periode).montant
date_attendue_paiement = debut_periode
```

### Paiement Option 2 (montant + fin_periode libres)
```
debut_periode = occupation.date_dernier_jour_couvert + 1 jour
Validation : fin_periode >= debut_periode (sinon 400)
nombre_de_loyers = null
date_attendue_paiement = debut_periode
```

### Post-validation paiement (OBLIGATOIRE, commun aux deux options)
```
occupation.date_dernier_jour_couvert = paiement.fin_periode
```

### Règles de suppression (retourner 422 avec message explicite)
- RG-01 : Impossible supprimer Locataire si Occupation liée
- RG-02 : Impossible supprimer Logement si Occupation liée
- RG-03 : Impossible supprimer Batiment si Logement lié
- RG-04 : Impossible supprimer Utilisateur → retourner 403/405, désactivation uniquement (isActive)
- RG-05 : Impossible créer Occupation si isOccupied() = true
- RG-06 : Impossible modifier Occupation si au moins 1 Paiement existe (sauf date_fin)
- RG-07 : Seul le dernier Paiement d'une Occupation est modifiable/supprimable intégralement
- RG-08 : Pour les paiements non-derniers, seule la preuve (fichiers) est modifiable
- RG-09 : Unicité (logement_id, date_debut) dans la table loyer
- RG-10 : À la création d'une Occupation : date_fin = null, date_dernier_jour_couvert = date_debut - 1 jour
- RG-11 : Après validation d'un Paiement : occupation.date_dernier_jour_couvert = paiement.fin_periode
- RG-12 : Option 2 : fin_periode >= debut_periode obligatoire

## Sécurité
- Mots de passe : bcrypt, salt rounds >= 12
- JWT : access_token (durée configurable .env) + refresh_token (Cookie HttpOnly)
- Guards : JwtAuthGuard (global) + RolesGuard (par endpoint) + ScopeGuard (périmètre)
- Rate limiting : @nestjs/throttler sur /auth/login et /auth/refresh
- Validation uploads : whitelist MIME, taille max (contrat: 10Mo, preuve: 5Mo)
- Ne jamais logger les mots de passe ni les tokens

## Variables d'environnement requises (.env)
```
DB_HOST, DB_PORT, DB_NAME, DB_USER, DB_PASSWORD
JWT_SECRET, JWT_EXPIRES_IN, REFRESH_TOKEN_SECRET, REFRESH_TOKEN_EXPIRES_IN
SMTP_HOST, SMTP_PORT, SMTP_USER, SMTP_PASS, SMTP_FROM
STORAGE_PATH
NOTIFICATION_CRON (ex: "0 8 * * *")
FRONTEND_URL
```

## Base de données
- TypeORM : synchronize=false en prod, toujours utiliser les migrations
- Indexer : logement_id, date_debut, date_fin, date_dernier_jour_couvert
- Ne jamais utiliser QueryBuilder sans .leftJoinAndSelect() explicite (éviter les N+1)

---

## Plan de réalisation – Étapes Backend

### Phase 0 – Initialisation (partie backend)
- P0.1 : Créer repo Git backend + README + .gitignore
- P0.2 : Initialiser projet NestJS TypeScript (`nest new backend`)
- P0.4 : Configurer variables d'environnement (.env + .env.example documenté)
- P0.5 : Configurer TypeORM + connexion MySQL (TypeOrmModule.forRootAsync)
- P0.6 : Installer toutes les dépendances backend

### Phase 1 – Entités & Modules
- B1.1 : Entité Utilisateur (enum Role, isActive, bcrypt)
- B1.2 : Entités Batiment et Logement (relations @OneToMany/@ManyToOne)
- B1.3 : Entité Loyer (periode_nombre + periode_type embedded, UNIQUE logement+date)
- B1.4 : Entités Locataire et Fichier
- B1.5 : Entité Occupation (date_fin nullable, date_dernier_jour_couvert)
- B1.6 : Entité Paiement + table paiement_fichier (@ManyToMany)
- B1.7 : Tables de liaison attributions (admin_logement_logement, admin_batiment_batiment)
- B1.8 : Générer et exécuter les migrations TypeORM
- B1.9 : Créer la structure des modules NestJS (tous les modules vides)

### Phase 2 – Logique métier
- B2.1 : CommonService – addPeriodToDate + tests Jest (4 types × fins de mois)
- B2.2 : LogementService – addLoyer(), getLoyer(date)
- B2.3 : OccupationService – isOccupied(), getArrieres() + tests Jest (4 scénarios)
- B2.4 : PaiementService – createOption1(), createOption2() + tests Jest
- B2.5 : PaiementService – getEquivalentNombreMois(), getEquivalentMontantMois()
- B2.6 : DTOs et validations class-validator (tous les modules)
- B2.7 : Controllers CRUD de base + AllExceptionsFilter global
- B2.8 : Règles de contrainte de suppression (RG-01 à RG-09)
- B2.9 : DashboardService – agrégations (getBatimentStats, getLogementStats, getLocataireStats)

### Phase 3 – Auth & Sécurité
- B3.1 : AuthModule – Login + génération access_token + refresh_token
- B3.2 : Refresh token rotation + endpoint /auth/me
- B3.3 : JwtAuthGuard (global) + RolesGuard + décorateur @Roles()
- B3.4 : ScopeGuard – vérification périmètre par rôle
- B3.5 : FilesModule – Multer sécurisé (MIME whitelist, taille, UUID filename)
- B3.6 : Rate limiting (@nestjs/throttler) + Helmet
- B3.7 : Finaliser documentation Swagger (@ApiBearerAuth, @ApiResponse sur tous les endpoints)

### Phase 4 – Fonctionnalités avancées
- B4.1 : ExportService – Excel via exceljs (6 rapports, mise en forme charte bleue)
- B4.2 : ExportService – PDF via pdfkit (rapports tabulaires)
- B4.3 : ExportController – 7 endpoints /export avec filtres (format, dateDebut, dateFin, périmètre)
- B4.4 : NotificationsService – routine @Cron() + algo filtrage par rôle
- B4.5 : Templates HTML email (4 templates selon rôle)
- B4.6 : Tests d'intégration Jest (flux critiques : auth, getArrieres, paiements, export)

---

## Étape en cours
<!-- Mettre à jour après chaque étape validée -->
P0.1
