# Référence complète – Entités, DTOs et Endpoints

> Ce fichier est la source de vérité pour la génération des entités TypeORM,
> des DTOs class-validator et des controllers NestJS.
> Il complète le CLAUDE.md pour les étapes B1.1→B1.7, B2.6 et B2.7.

---

## 1. Entités TypeORM

### 1.1 Utilisateur

**Table** : `utilisateur`

| Colonne | Type TypeScript | Type SQL | Contrainte / Défaut |
|---|---|---|---|
| id | number | INT AUTO_INCREMENT | PK |
| nom | string | VARCHAR(100) | NOT NULL |
| prenom | string | VARCHAR(100) | NOT NULL |
| telephone | string | VARCHAR(30) | NOT NULL, UNIQUE |
| email | string \| null | VARCHAR(150) | UNIQUE, NULL |
| username | string | VARCHAR(80) | NOT NULL, UNIQUE |
| password | string | VARCHAR(255) | NOT NULL (bcrypt) |
| role | Role (enum) | ENUM | NOT NULL |
| isActive | boolean | BOOLEAN | NOT NULL, DEFAULT true |
| createdAt | Date | DATETIME | Auto (@CreateDateColumn) |
| updatedAt | Date | DATETIME | Auto (@UpdateDateColumn) |

**Enum Role** :
```typescript
export enum Role {
  LOCATAIRE = 'LOCATAIRE',
  ADMIN_LOGEMENT = 'ADMIN_LOGEMENT',
  ADMIN_BATIMENT = 'ADMIN_BATIMENT',
  ADMIN_GLOBAL = 'ADMIN_GLOBAL',
}
```

**Relations** :
- `@OneToOne` → Locataire (optionnel, côté Locataire)
- `@ManyToMany` → Logement (table `admin_logement_logement`) pour ADMIN_LOGEMENT
- `@ManyToMany` → Batiment (table `admin_batiment_batiment`) pour ADMIN_BATIMENT

---

### 1.2 Batiment

**Table** : `batiment`

| Colonne | Type TypeScript | Type SQL | Contrainte / Défaut |
|---|---|---|---|
| id | number | INT AUTO_INCREMENT | PK |
| nom | string | VARCHAR(150) | NOT NULL |
| adresse | string | VARCHAR(255) | NOT NULL |
| description | string \| null | TEXT | NULL |
| createdAt | Date | DATETIME | Auto |
| updatedAt | Date | DATETIME | Auto |

**Relations** :
- `@OneToMany` → Logement (`logement.batimentId`)
- `@ManyToMany` → Utilisateur (table `admin_batiment_batiment`)

---

### 1.3 Logement

**Table** : `logement`

| Colonne | Type TypeScript | Type SQL | Contrainte / Défaut |
|---|---|---|---|
| id | number | INT AUTO_INCREMENT | PK |
| batimentId | number | INT | FK → batiment(id), NOT NULL |
| nom | string | VARCHAR(150) | NOT NULL |
| description | string \| null | TEXT | NULL |
| createdAt | Date | DATETIME | Auto |
| updatedAt | Date | DATETIME | Auto |

**Relations** :
- `@ManyToOne` → Batiment
- `@OneToMany` → Loyer
- `@OneToMany` → Occupation
- `@ManyToMany` → Utilisateur (table `admin_logement_logement`)

---

### 1.4 Loyer

**Table** : `loyer`

| Colonne | Type TypeScript | Type SQL | Contrainte / Défaut |
|---|---|---|---|
| id | number | INT AUTO_INCREMENT | PK |
| logementId | number | INT | FK → logement(id), NOT NULL |
| montant | number | DECIMAL(15,2) | NOT NULL |
| dateDebut | Date | DATE | NOT NULL, DEFAULT = date du jour |
| periodeNombre | number | INT | NOT NULL, >= 1 |
| periodeType | PeriodeType (enum) | ENUM | NOT NULL, DEFAULT 'MOIS' |
| createdAt | Date | DATETIME | Auto |

**Contrainte UNIQUE** : `(logementId, dateDebut)` — ajouter `@Unique(['logementId', 'dateDebut'])` sur l'entité.

**Enum PeriodeType** :
```typescript
export enum PeriodeType {
  JOUR = 'JOUR',
  SEMAINE = 'SEMAINE',
  MOIS = 'MOIS',
  ANNEE = 'ANNEE',
}
```

**Type Periode** (value object, pas une entité) :
```typescript
export interface Periode {
  nombre: number; // >= 1
  type: PeriodeType;
}
```

**Relations** :
- `@ManyToOne` → Logement

---

### 1.5 Locataire

**Table** : `locataire`

| Colonne | Type TypeScript | Type SQL | Contrainte / Défaut |
|---|---|---|---|
| id | number | INT AUTO_INCREMENT | PK |
| nom | string | VARCHAR(100) | NOT NULL |
| prenom | string | VARCHAR(100) | NOT NULL |
| telephone | string | VARCHAR(30) | NOT NULL |
| email | string \| null | VARCHAR(150) | NULL |
| utilisateurId | number \| null | INT | FK → utilisateur(id), NULL, UNIQUE |
| createdAt | Date | DATETIME | Auto |
| updatedAt | Date | DATETIME | Auto |

**Relations** :
- `@OneToOne` → Utilisateur (nullable)
- `@OneToMany` → Occupation

---

### 1.6 Fichier

**Table** : `fichier`

| Colonne | Type TypeScript | Type SQL | Contrainte / Défaut |
|---|---|---|---|
| id | number | INT AUTO_INCREMENT | PK |
| nomOriginal | string | VARCHAR(255) | NOT NULL |
| cheminStockage | string | VARCHAR(500) | NOT NULL |
| mimeType | string | VARCHAR(100) | NOT NULL |
| taille | number | INT | NOT NULL (octets) |
| uploadedById | number | INT | FK → utilisateur(id), NOT NULL |
| createdAt | Date | DATETIME | Auto |

---

### 1.7 Occupation

**Table** : `occupation`

| Colonne | Type TypeScript | Type SQL | Contrainte / Défaut |
|---|---|---|---|
| id | number | INT AUTO_INCREMENT | PK |
| logementId | number | INT | FK → logement(id), NOT NULL |
| locataireId | number | INT | FK → locataire(id), NOT NULL |
| dateDebut | Date | DATE | NOT NULL |
| dateFin | Date \| null | DATE | NULL (null = occupation en cours) |
| dateDernierJourCouvert | Date | DATE | NOT NULL (= dateDebut - 1 jour à la création) |
| contratFichierId | number \| null | INT | FK → fichier(id), NULL |
| createdAt | Date | DATETIME | Auto |
| updatedAt | Date | DATETIME | Auto |

**Relations** :
- `@ManyToOne` → Logement
- `@ManyToOne` → Locataire
- `@OneToMany` → Paiement
- `@ManyToOne` → Fichier (contrat, nullable)

---

### 1.8 Paiement

**Table** : `paiement`

| Colonne | Type TypeScript | Type SQL | Contrainte / Défaut |
|---|---|---|---|
| id | number | INT AUTO_INCREMENT | PK |
| occupationId | number | INT | FK → occupation(id), NOT NULL |
| debutPeriode | Date | DATE | NOT NULL (calculé auto) |
| finPeriode | Date | DATE | NOT NULL, >= debutPeriode |
| montantPaye | number | DECIMAL(15,2) | NOT NULL |
| nombreDeLoyers | number \| null | INT | NULL (Option 2), >= 1 si défini |
| datePaiement | Date | DATE | NOT NULL, DEFAULT = date du jour |
| dateAttenduePaiement | Date | DATE | NOT NULL (= debutPeriode) |
| commentaire | string \| null | TEXT | NULL |
| createdAt | Date | DATETIME | Auto |
| updatedAt | Date | DATETIME | Auto |

**Relations** :
- `@ManyToOne` → Occupation
- `@ManyToMany` → Fichier (preuves) via table `paiement_fichier`

---

### 1.9 Tables de liaison

**`admin_logement_logement`** :
| Colonne | Type SQL |
|---|---|
| utilisateur_id | INT FK → utilisateur(id) |
| logement_id | INT FK → logement(id) |

**`admin_batiment_batiment`** :
| Colonne | Type SQL |
|---|---|
| utilisateur_id | INT FK → utilisateur(id) |
| batiment_id | INT FK → batiment(id) |

**`paiement_fichier`** :
| Colonne | Type SQL |
|---|---|
| paiement_id | INT FK → paiement(id) |
| fichier_id | INT FK → fichier(id) |

---

## 2. DTOs (class-validator)

### 2.1 Auth

```typescript
// LoginDto
username: string          // @IsString() @IsNotEmpty()
password: string          // @IsString() @IsNotEmpty()

// RefreshTokenDto
refreshToken: string      // @IsString() @IsNotEmpty()
```

---

### 2.2 Utilisateur

```typescript
// CreateUserDto
nom: string               // @IsString() @IsNotEmpty()
prenom: string            // @IsString() @IsNotEmpty()
telephone: string         // @IsString() @IsNotEmpty()
email?: string            // @IsEmail() @IsOptional()
username: string          // @IsString() @IsNotEmpty()
password: string          // @IsString() @MinLength(8)
role: Role                // @IsEnum(Role) @IsNotEmpty()

// UpdateUserDto — tous les champs optionnels (PartialType de CreateUserDto sauf password/role)
// ResetPasswordDto
newPassword: string       // @IsString() @MinLength(8)

// ToggleActiveDto
isActive: boolean         // @IsBoolean()
```

---

### 2.3 Batiment

```typescript
// CreateBatimentDto
nom: string               // @IsString() @IsNotEmpty()
adresse: string           // @IsString() @IsNotEmpty()
description?: string      // @IsString() @IsOptional()

// UpdateBatimentDto — PartialType(CreateBatimentDto)
```

---

### 2.4 Logement

```typescript
// CreateLogementDto
batimentId: number        // @IsInt() @IsPositive()
nom: string               // @IsString() @IsNotEmpty()
description?: string      // @IsString() @IsOptional()
// Loyer initial (obligatoire à la création)
loyerMontant: number      // @IsNumber() @IsPositive()
loyerDateDebut?: Date     // @IsDateString() @IsOptional() (défaut = aujourd'hui)
loyerPeriodeNombre: number // @IsInt() @Min(1)
loyerPeriodeType: PeriodeType // @IsEnum(PeriodeType)

// UpdateLogementDto — PartialType sans les champs loyer

// AddLoyerDto
montant: number           // @IsNumber() @IsPositive()
dateDebut?: Date          // @IsDateString() @IsOptional() (défaut = aujourd'hui)
periodeNombre: number     // @IsInt() @Min(1)
periodeType: PeriodeType  // @IsEnum(PeriodeType)
```

---

### 2.5 Locataire

```typescript
// CreateLocataireDto
nom: string               // @IsString() @IsNotEmpty()
prenom: string            // @IsString() @IsNotEmpty()
telephone: string         // @IsString() @IsNotEmpty()
email?: string            // @IsEmail() @IsOptional()
utilisateurId?: number    // @IsInt() @IsOptional()

// UpdateLocataireDto — PartialType(CreateLocataireDto)
```

---

### 2.6 Occupation

```typescript
// CreateOccupationDto
logementId: number        // @IsInt() @IsPositive()
locataireId: number       // @IsInt() @IsPositive()
dateDebut: Date           // @IsDateString() @IsNotEmpty()

// UpdateOccupationDto (uniquement si aucun paiement)
dateDebut?: Date          // @IsDateString() @IsOptional()
locataireId?: number      // @IsInt() @IsOptional()

// EndOccupationDto
dateFin: Date             // @IsDateString() @IsNotEmpty()
```

---

### 2.7 Paiement

```typescript
// CreatePaiementOption1Dto
occupationId: number      // @IsInt() @IsPositive()
nombreDeLoyers: number    // @IsInt() @Min(1)
datePaiement?: Date       // @IsDateString() @IsOptional() (défaut = aujourd'hui)
commentaire?: string      // @IsString() @IsOptional()

// CreatePaiementOption2Dto
occupationId: number      // @IsInt() @IsPositive()
montantPaye: number       // @IsNumber() @IsPositive()
finPeriode: Date          // @IsDateString() @IsNotEmpty()
datePaiement?: Date       // @IsDateString() @IsOptional()
commentaire?: string      // @IsString() @IsOptional()

// UpdatePaiementDto (dernier paiement uniquement)
datePaiement?: Date       // @IsDateString() @IsOptional()
commentaire?: string      // @IsString() @IsOptional()
montantPaye?: number      // @IsNumber() @IsPositive() @IsOptional()
finPeriode?: Date         // @IsDateString() @IsOptional()
```

---

### 2.8 Export

```typescript
// ExportQueryDto
format: 'excel' | 'pdf'   // @IsIn(['excel','pdf'])
dateDebut?: Date          // @IsDateString() @IsOptional()
dateFin?: Date            // @IsDateString() @IsOptional()
logementId?: number       // @IsInt() @IsOptional()
batimentId?: number       // @IsInt() @IsOptional()
locale?: 'fr' | 'en'     // @IsIn(['fr','en']) @IsOptional() (défaut: 'fr')
```

---

## 3. Endpoints REST complets

> Format : `MÉTHODE /chemin` — Description — Rôle minimum requis

### 3.1 Auth `/auth`
```
POST   /auth/login              Connexion → access_token + refresh_token     PUBLIC
POST   /auth/logout             Déconnexion                                   AUTH
POST   /auth/refresh            Rafraîchir le token                           AUTH
GET    /auth/me                 Profil utilisateur connecté                   AUTH
```

### 3.2 Utilisateurs `/users`
```
GET    /users                   Liste (filtrée par périmètre)                 ADMIN_LOGEMENT
POST   /users                   Créer un utilisateur                          ADMIN_LOGEMENT
GET    /users/:id               Détail                                        ADMIN_LOGEMENT
PATCH  /users/:id               Modifier                                      ADMIN_LOGEMENT
PATCH  /users/:id/activate      Activer / Désactiver (ToggleActiveDto)        ADMIN_LOGEMENT
PATCH  /users/:id/reset-password  Réinitialiser mot de passe                 ADMIN_LOGEMENT
```

### 3.3 Bâtiments `/batiments`
```
GET    /batiments                Liste (filtrée par périmètre)                ADMIN_BATIMENT
POST   /batiments                Créer                                         ADMIN_GLOBAL
GET    /batiments/:id            Détail                                        ADMIN_BATIMENT
PATCH  /batiments/:id            Modifier                                      ADMIN_BATIMENT
DELETE /batiments/:id            Supprimer (si aucun logement lié)            ADMIN_GLOBAL
GET    /batiments/:id/dashboard  KPIs analytiques                             ADMIN_BATIMENT
```

### 3.4 Logements `/logements`
```
GET    /logements                Liste (filtrée)                               ADMIN_LOGEMENT
POST   /logements                Créer + loyer initial                        ADMIN_BATIMENT
GET    /logements/:id            Détail                                        ADMIN_LOGEMENT
PATCH  /logements/:id            Modifier                                      ADMIN_BATIMENT
DELETE /logements/:id            Supprimer (si aucune occupation liée)        ADMIN_BATIMENT
POST   /logements/:id/loyers     Ajouter un loyer (AddLoyerDto)               ADMIN_LOGEMENT
GET    /logements/:id/loyers     Historique des loyers                         ADMIN_LOGEMENT
GET    /logements/:id/dashboard  KPIs analytiques                             ADMIN_LOGEMENT
```

### 3.5 Locataires `/locataires`
```
GET    /locataires               Liste (filtrée)                               ADMIN_LOGEMENT
POST   /locataires               Créer                                         ADMIN_LOGEMENT
GET    /locataires/:id           Détail                                        ADMIN_LOGEMENT
PATCH  /locataires/:id           Modifier                                      ADMIN_LOGEMENT
DELETE /locataires/:id           Supprimer (si aucune occupation liée)        ADMIN_LOGEMENT
GET    /locataires/:id/dashboard KPIs solvabilité / assiduité                ADMIN_LOGEMENT
```

### 3.6 Occupations `/occupations`
```
GET    /occupations              Liste (filtrée)                               ADMIN_LOGEMENT
POST   /occupations              Créer (vérifie isOccupied)                   ADMIN_LOGEMENT
GET    /occupations/:id          Détail                                        LOCATAIRE (la sienne)
PATCH  /occupations/:id          Modifier (interdit si paiement existant)     ADMIN_LOGEMENT
PATCH  /occupations/:id/fin      Mettre fin (EndOccupationDto)                ADMIN_LOGEMENT
DELETE /occupations/:id          Supprimer (interdit si paiement existant)    ADMIN_LOGEMENT
GET    /occupations/:id/arrieres Calcul arriéré (?date=YYYY-MM-DD optionnel)  LOCATAIRE (la sienne)
POST   /occupations/:id/contrat  Upload contrat de bail                       ADMIN_LOGEMENT
GET    /occupations/:id/contrat  Télécharger contrat                          LOCATAIRE (la sienne)
```

### 3.7 Paiements `/paiements`
```
GET    /paiements                Liste (filtrée)                               ADMIN_LOGEMENT
POST   /paiements/option1        Créer paiement Option 1                      ADMIN_LOGEMENT
POST   /paiements/option2        Créer paiement Option 2                      ADMIN_LOGEMENT
GET    /paiements/:id            Détail                                        LOCATAIRE (le sien)
PATCH  /paiements/:id            Modifier (dernier uniquement)                ADMIN_LOGEMENT
DELETE /paiements/:id            Supprimer (dernier uniquement)               ADMIN_LOGEMENT
PATCH  /paiements/:id/preuves    Remplacer preuves (tous paiements)           ADMIN_LOGEMENT
POST   /paiements/:id/preuves    Uploader preuves                             ADMIN_LOGEMENT
```

### 3.8 Export `/export`
```
GET    /export/occupations       Export occupations + paiements               ADMIN_LOGEMENT
GET    /export/paiements         Export historique paiements                  ADMIN_LOGEMENT
GET    /export/locataires        Export liste locataires                      ADMIN_LOGEMENT
GET    /export/logements         Export logements + statuts                   ADMIN_LOGEMENT
GET    /export/batiments         Export bâtiments                             ADMIN_BATIMENT
GET    /export/arrieres          Export rapport arriérés                      ADMIN_LOGEMENT
GET    /export/dashboard/:id     Export dashboard bâtiment                   ADMIN_BATIMENT
```
> Tous les endpoints /export acceptent les query params du ExportQueryDto.

### 3.9 Notifications `/notifications`
```
POST   /notifications/trigger    Déclencher manuellement la routine (DEV only) ADMIN_GLOBAL
```

### 3.10 Dashboard `/dashboard`
```
GET    /dashboard/global         KPIs globaux (nb bâtiments, logements, arriérés…) ADMIN_LOGEMENT
```

---

## 4. Codes HTTP de réponse

| Code | Situation |
|---|---|
| 200 | Succès GET / PATCH / DELETE |
| 201 | Succès POST (création) |
| 400 | Validation DTO échouée |
| 401 | Token absent, invalide ou expiré |
| 403 | Rôle insuffisant ou hors périmètre |
| 404 | Ressource introuvable |
| 409 | Violation contrainte unique (ex : date_debut Loyer en doublon) |
| 422 | Règle métier violée (ex : logement déjà occupé, occupation non modifiable) |
| 429 | Rate limit dépassé |
| 500 | Erreur serveur non gérée |
