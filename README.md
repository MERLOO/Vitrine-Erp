# ERP - Système de Gestion d'Entreprise

Application ERP modulaire composée d'un backend NestJS et d'un frontend Next.js.

## Structure du projet

```
erp/
├── erp-backend/     # API REST (NestJS + Prisma + PostgreSQL)
└── erp-frontend/    # Interface web (Next.js + React + TailwindCSS)
```

---

## Backend

### Stack technique

| Technologie | Usage |
|-------------|-------|
| **NestJS 10** | Framework backend avec architecture modulaire |
| **Prisma 5** | ORM pour PostgreSQL avec migrations auto-générées |
| **PostgreSQL** | Base de données relationnelle |
| **JWT** | Authentification stateless avec refresh token |
| **Swagger/OpenAPI** | Documentation API automatique |
| **class-validator** | Validation des DTOs |
| **bcryptjs** | Hash des mots de passe |
| **Docker** | Conteneurisation pour développement et production |

### Architecture

**Monolithe modulaire** — chaque domaine métier est isolé dans son propre module.

```
src/
├── modules/
│   ├── auth/          # Authentification JWT
│   ├── users/         # Gestion utilisateurs & RBAC
│   ├── products/      # Catalogue & mouvements de stock
│   ├── contacts/      # Clients/fournisseurs
│   ├── orders/        # Commandes avec state machine
│   └── dashboard/     # KPIs et agrégations
├── common/
│   ├── decorators/    # @Roles() pour RBAC
│   └── guards/        # RolesGuard, JwtAuthGuard
├── prisma/            # PrismaService global
└── main.ts            # Bootstrap (Helmet, CORS, Swagger)
```

**Pattern Controller → Service → Repository** :
- **Controller** : routing, validation, transformation
- **Service** : logique métier
- **PrismaService** : accès aux données

### Modules et endpoints

#### Auth (`/api/v1/auth`)
| Endpoint | Méthode | Description |
|----------|---------|-------------|
| `/login` | POST | Authentification, retourne access + refresh tokens |
| `/refresh` | POST | Rotation du refresh token |
| `/logout` | POST | Révocation du refresh token |
| `/me` | GET | Informations de l'utilisateur connecté |

**Mécanisme JWT** :
- Access token : 15 minutes
- Refresh token : 7 jours avec rotation (un seul usage)
- Hash SHA-256 pour le stockage des refresh tokens

#### Users (`/api/v1/users`)
| Endpoint | Méthode | Rôles | Description |
|----------|---------|-------|-------------|
| `/` | GET | ADMIN, MANAGER | Liste paginée des utilisateurs |
| `/` | POST | ADMIN | Création d'utilisateur |
| `/:id` | GET | ADMIN, MANAGER | Détails d'un utilisateur |
| `/:id` | PATCH | ADMIN | Modification |
| `/:id` | DELETE | ADMIN | Soft delete |
| `/me` | GET | Tous | Profil courant |
| `/me` | PATCH | Tous | Modification de son propre profil |

**RBAC** : 3 rôles (`ADMIN`, `MANAGER`, `EMPLOYEE`)

#### Products (`/api/v1/products`)
| Endpoint | Méthode | Description |
|----------|---------|-------------|
| `/` | GET | Liste paginée avec filtres |
| `/` | POST | Création produit |
| `/:id` | GET | Détails avec fournisseur |
| `/:id` | PATCH | Modification |
| `/:id` | DELETE | Soft delete |
| `/low-stock` | GET | Produits sous seuil |
| `/:id/stock-movements` | GET | Historique des mouvements |
| `/:id/stock-movements` | POST | Nouveau mouvement (IN/OUT/ADJUSTMENT) |

**Gestion du stock** :
- Transactions Prisma pour garantie atomicité
- Vérification stock négatif impossible
- Mouvements immuables (append-only)

#### Contacts (`/api/v1/contacts`)
| Endpoint | Méthode | Description |
|----------|---------|-------------|
| `/` | GET | Liste avec filtres par type |
| `/` | POST | Création |
| `/:id` | GET | Détails |
| `/:id` | PATCH | Modification |
| `/:id` | DELETE | Soft delete |

**Types** : `CLIENT`, `SUPPLIER`, `BOTH`

#### Orders (`/api/v1/orders`)
| Endpoint | Méthode | Description |
|----------|---------|-------------|
| `/` | GET | Liste avec filtres |
| `/` | POST | Création (DRAFT) |
| `/:id` | GET | Détails avec lignes |
| `/:id` | PATCH | Modification lignes/contact |
| `/:id/status` | PATCH | Transition de statut |
| `/:id` | DELETE | Suppression (DRAFT uniquement) |

**State machine** :
```
DRAFT → CONFIRMED → SHIPPED → DELIVERED
  ↓          ↓
CANCELLED  CANCELLED
```

**Règles métier** :
- Snapshot du prix unitaire à la création
- Décrément stock à CONFIRMED, restauré à CANCELLED
- EMPLOYEE ne peut modifier que ses propres brouillons

#### Dashboard (`/api/v1/dashboard`)
| Endpoint | Méthode | Description |
|----------|---------|-------------|
| `/summary` | GET | KPIs globaux |
| `/revenue-chart` | GET | Données graphique revenus |
| `/low-stock` | GET | Alertes stock bas |
| `/top-products` | GET | Produits les plus vendus |

**Restriction** : KPIs financiers masqués pour EMPLOYEE

### Sécurité

- **Guards** : `JwtAuthGuard` → `RolesGuard` (chaînage obligatoire)
- **Rate limiting** : 100 req/min global, 5/min sur `/auth/login`
- **Validation** : `whitelist: true, forbidNonWhitelisted: true`
- **Protection ADMIN** : auto-protection contre auto-désactivation/rétrogradation

### Modèle de données

| Modèle | Description |
|--------|-------------|
| `User` | Comptes utilisateurs avec rôle |
| `RefreshToken` | Sessions (hash SHA-256, rotation) |
| `Product` | Catalogue avec prix Decimal |
| `StockMovement` | Historique immuable |
| `Contact` | Clients/fournisseurs unifiés |
| `Order` | Commandes avec state machine |
| `OrderLine` | Lignes avec snapshot prix |

---

## Frontend

### Stack technique

| Technologie | Usage |
|-------------|-------|
| **Next.js 16** | Framework React avec App Router |
| **React 19** | UI library |
| **TailwindCSS 4** | Styling |
| **shadcn/ui** | Composants UI accessibles |
| **Zustand** | State management |
| **React Hook Form** | Gestion des formulaires |
| **Zod** | Validation côté client |
| **Axios** | Client HTTP |
| **Recharts** | Graphiques et visualisations |

### Architecture

```
src/
├── app/                    # App Router (pages)
│   ├── (dashboard)/        # Layout protégé
│   ├── login/              # Page de connexion
│   ├── layout.tsx          # Layout racine
│   └── page.tsx            # Page d'accueil
├── components/
│   ├── layout/             # Header, Sidebar
│   └── ui/                 # Composants shadcn
├── lib/
│   ├── api/                # Services API par domaine
│   │   ├── auth.ts
│   │   ├── users.ts
│   │   ├── products.ts
│   │   ├── contacts.ts
│   │   └── orders.ts
│   ├── api-client.ts       # Instance Axios + interceptors
│   └── utils.ts            # Utilitaires
├── stores/
│   └── auth-store.ts       # État d'authentification
└── types/
    └── index.ts            # Types TypeScript (DTOs)
```

### Gestion d'état

**Zustand avec persist** :
```typescript
// auth-store.ts
interface AuthState {
  user: User | null;
  isAuthenticated: boolean;
  isLoading: boolean;
  login: (email, password) => Promise<void>;
  logout: () => Promise<void>;
  refreshUser: () => Promise<void>;
}
```

Persistance localStorage pour `user` et `isAuthenticated`.

### Client API

**Intercepteurs Axios** :
- **Request** : injection automatique du `Authorization: Bearer`
- **Response** : gestion du 401 avec refresh token automatique
- **Queue** : mise en file des requêtes pendant le refresh

```typescript
// api-client.ts
api.interceptors.response.use(
  (response) => response,
  async (error) => {
    if (error.response?.status === 401 && refreshToken) {
      // Tentative de refresh automatique
      // Si échec → redirection /login
    }
  }
);
```

### Composants UI

Composants shadcn/ui utilisés :
- `Button`, `Input`, `Label`, `Select`
- `Card`, `Dialog`, `Sheet`
- `Table`, `Badge`, `Avatar`
- `DropdownMenu`, `Toast` (Sonner)

### Types TypeScript

Types synchronisés avec les modèles Prisma backend :
- `User`, `Product`, `Contact`, `Order`, `OrderLine`
- `StockMovement`, `RefreshToken`
- DTOs : `CreateUserDto`, `UpdateProductDto`, etc.
- Query params : `ListUsersQuery`, `ListProductsQuery`, etc.

---

## Démarrage

### Prérequis

- Docker et Docker Compose
- Node.js 20+ (pour développement local)

### Backend

```bash
cd erp-backend
cp .env.example .env
docker compose up --build
```

- API : http://localhost:3001/api/v1
- Swagger : http://localhost:3001/api/docs

**Initialisation** :
```bash
docker compose exec backend npx prisma migrate deploy
docker compose exec backend npm run prisma:seed          # Admin initial
docker compose exec backend npm run prisma:seed-demo     # Données démo
```

Compte admin : `admin@erp.local` / `ChangeMe123!`

### Frontend

```bash
cd erp-frontend
npm install
npm run dev
```

Interface : http://localhost:3000

Configurer `.env.local` :
```
NEXT_PUBLIC_API_URL=http://localhost:3001/api/v1
```

---

## Tests

### Backend

```bash
npm run test        # Tests unitaires (Jest)
npm run test:e2e    # Tests d'intégration (Supertest)
npm run lint        # ESLint + Prettier
```

---

## Scripts disponibles

### Backend

| Commande | Description |
|----------|-------------|
| `start:dev` | Développement avec hot-reload |
| `build` | Build production |
| `test` | Tests unitaires |
| `test:e2e` | Tests d'intégration |
| `lint` | Linting |
| `prisma:migrate` | Nouvelle migration |
| `prisma:seed` | Création admin initial |
| `prisma:seed-demo` | Données de démonstration |

### Frontend

| Commande | Description |
|----------|-------------|
| `dev` | Serveur de développement |
| `build` | Build production |
| `start` | Serveur production |
| `lint` | ESLint |

---

## Documentation

- [Documentation backend détaillée](./erp-backend/DOCUMENTATION.md)
- [API Swagger](http://localhost:3001/api/docs) (backend lancé)
