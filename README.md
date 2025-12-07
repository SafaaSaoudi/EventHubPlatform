# Workshop2 - EventHub Application

This project was generated with [Angular CLI](https://github.com/angular/angular-cli) version 18.2.20.

## Overview

EventHub is an Angular application for managing events, tickets, and user feedback. The application includes a complete authentication system with sign-in, sign-up, route guards, and HTTP interceptors.

## Table of Contents

- [Development Setup](#development-setup)
- [Authentication System](#authentication-system)
  - [Architecture](#architecture)
  - [Components](#components)
  - [Usage Examples](#usage-examples)
  - [Configuration](#configuration)
- [Project Structure](#project-structure)
- [Running the Application](#running-the-application)

---

## Development Setup

### Prerequisites

- Node.js (v18 or higher)
- Angular CLI: `npm install -g @angular/cli`
- json-server (for mock backend): `npm install -g json-server`

### Installation

```bash
npm install
```

### Start the Mock Backend

Before running the application, start the json-server to provide the mock API:

```bash
json-server -w src/db.json 
```

The API will be available at `http://localhost:3000`

---

## Authentication System

### Architecture

The authentication system is built with the following components:

```
src/app/
├── core/
│   ├── services/
│   │   └── auth.service.ts          # Authentication service
│   ├── guards/
│   │   ├── auth.guard.ts           # Protects authenticated routes
│   │   └── guest.guard.ts          # Protects auth pages from logged-in users
│   └── interceptors/
│       └── auth.interceptor.ts      # Adds token to HTTP requests
├── models/
│   └── user.ts                      # User and auth interfaces
└── features/
    └── auth/
        ├── sign-in/                 # Sign-in component
        ├── sign-up/                 # Sign-up component
        └── auth.module.ts           # Auth feature module
```

### Components

#### 1. AuthService (`src/app/core/services/auth.service.ts`)

The central service managing authentication state and operations.

**Key Features:**
- User registration (`signUp()`)
- User login (`signIn()`)
- User logout (`logout()`)
- Token management
- Observable-based state management

**Public API:**
```typescript
// Observables
currentUser$: Observable<User | null>      // Current user data
isAuthenticated$: Observable<boolean>      // Authentication status

// Methods
signUp(registerData: RegisterRequest): Observable<AuthResponse>
signIn(loginData: LoginRequest): Observable<AuthResponse>
logout(): void
getToken(): string | null
getCurrentUser(): User | null
isLoggedIn(): boolean
```

**How it works:**
- Uses `BehaviorSubject` to manage authentication state
- Stores token and user data in `localStorage`
- Automatically notifies subscribers when auth state changes
- Generates simple base64 tokens (for development; use JWT in production)

#### 2. Auth Guard (`src/app/core/guards/auth.guard.ts`)

Protects routes that require authentication.

**Functionality:**
- Checks if user is logged in
- Redirects to sign-in page if not authenticated
- Preserves the requested URL for redirect after login

**Usage:**
```typescript
import { authGuard } from './core/guards/auth.guard';

const routes: Routes = [
  {
    path: 'tickets',
    component: TicketsComponent,
    canActivate: [authGuard]  // Route protected
  }
];
```

#### 3. Guest Guard (`src/app/core/guards/guest.guard.ts`)

Prevents authenticated users from accessing login/signup pages.

**Functionality:**
- Checks if user is NOT logged in
- Redirects to home if already authenticated
- Used on `/auth/sign-in` and `/auth/sign-up` routes

**Usage:**
```typescript
import { guestGuard } from './core/guards/guest.guard';

const routes: Routes = [
  {
    path: 'sign-in',
    component: SignInComponent,
    canActivate: [guestGuard]  // Only accessible to non-authenticated users
  }
];
```

#### Why Both Guards Are Needed

**`authGuard`** and **`guestGuard`** serve complementary purposes:

| Guard | Purpose | Logic | Redirects to |
|-------|---------|-------|--------------|
| **authGuard** | Protects **private routes** | ✅ Allows if authenticated<br>❌ Blocks if not authenticated | `/auth/sign-in` |
| **guestGuard** | Protects **auth pages** | ✅ Allows if NOT authenticated<br>❌ Blocks if already authenticated | `/home` |

**Real-world scenarios:**

1. **User tries to access `/tickets` while logged out:**
   - `authGuard` checks: "Is user logged in?" → NO
   - Result: Redirects to `/auth/sign-in?returnUrl=/tickets`

2. **User tries to access `/auth/sign-in` while logged in:**
   - `guestGuard` checks: "Is user logged in?" → YES
   - Result: Redirects to `/home` (prevents unnecessary access to login page)

3. **User tries to access `/auth/sign-in` while logged out:**
   - `guestGuard` checks: "Is user logged in?" → NO
   - Result: Allows access (normal behavior)

**Benefits:**
- ✅ Better UX: Prevents logged-in users from seeing login pages
- ✅ Clear separation of concerns: Each guard has a specific role
- ✅ Security: Proper access control for both public and private routes
- ✅ Maintainability: Easy to understand and modify

#### 4. Auth Interceptor (`src/app/core/interceptors/auth.interceptor.ts`)

Automatically adds authentication token to HTTP requests.

**Functionality:**
- Intercepts all HTTP requests
- Adds `Authorization: Bearer <token>` header if token exists
- No manual token management needed in components

**Configuration:**
Registered in `app.module.ts`:
```typescript
providers: [
  provideHttpClient(withInterceptors([authInterceptor]))
]
```

#### 5. Sign-In Component (`src/app/features/auth/sign-in/`)

Login form component with validation and error handling.

**Features:**
- Email and password validation
- Loading states
- Error message display
- Automatic redirect after successful login
- Preserves return URL for post-login navigation

#### 6. Sign-Up Component (`src/app/features/auth/sign-up/`)

Registration form component.

**Features:**
- Form validation (email, password, confirmation)
- Password strength check (minimum 6 characters)
- Email uniqueness check
- Loading states
- Error handling

### Usage Examples

#### Using AuthService in a Component

```typescript
import { Component, OnInit, OnDestroy } from '@angular/core';
import { AuthService } from './core/services/auth.service';
import { User } from './models/user';
import { Subscription } from 'rxjs';

@Component({
  selector: 'app-my-component',
  templateUrl: './my-component.html'
})
export class MyComponent implements OnInit, OnDestroy {
  currentUser: User | null = null;
  isAuthenticated: boolean = false;
  private subscriptions = new Subscription();

  constructor(private authService: AuthService) {}

  ngOnInit(): void {
    // Subscribe to current user
    this.subscriptions.add(
      this.authService.currentUser$.subscribe(user => {
        this.currentUser = user;
      })
    );

    // Subscribe to authentication status
    this.subscriptions.add(
      this.authService.isAuthenticated$.subscribe(isAuth => {
        this.isAuthenticated = isAuth;
      })
    );
  }

  ngOnDestroy(): void {
    this.subscriptions.unsubscribe();
  }

  logout(): void {
    this.authService.logout();
  }
}
```

#### Using Auth State in Templates

```html
<!-- Show content only for authenticated users -->
<div *ngIf="isAuthenticated">
  <p>Welcome, {{ currentUser?.firstName }}!</p>
  <button (click)="logout()">Logout</button>
</div>

<!-- Show content only for guests -->
<div *ngIf="!isAuthenticated">
  <a routerLink="/auth/sign-in">Sign In</a>
</div>
```

#### Protecting Routes

```typescript
// app-routing.module.ts
import { authGuard } from './core/guards/auth.guard';

const routes: Routes = [
  { path: 'home', component: HomeComponent },
  { 
    path: 'tickets', 
    component: TicketsComponent,
    canActivate: [authGuard]  // Protected route
  },
  { 
    path: 'events', 
    loadChildren: () => import('./features/events/events.module').then(m => m.EventsModule),
    canActivate: [authGuard]  // Protected lazy-loaded route
  }
];
```

#### Making Authenticated API Calls

The interceptor automatically adds the token, so you don't need to do anything special:

```typescript
// The interceptor automatically adds: Authorization: Bearer <token>
this.http.get('http://localhost:3000/api/protected-data').subscribe(data => {
  // Handle response
});
```

### Configuration

#### Test Users

The following test users are available in `src/db.json`:

| Email | Password | Role |
|-------|----------|------|
| admin@example.com | admin123 | admin |
| user@example.com | user123 | user |
| contact@aurianeparishotel.com | AZERTY | user |
| safa.saoudi@esprit.tn | 123456789 | user |

#### API Configuration

The API URL is configured in `auth.service.ts`:

```typescript
private readonly API_URL = 'http://localhost:3000';
```

To change the API endpoint, update this constant.

#### Token Storage

Tokens are stored in `localStorage` with the following keys:
- `auth_token`: The authentication token
- `current_user`: The current user object (without password)

#### Routes

- `/auth/sign-in` - Sign in page (protected by `guestGuard`)
- `/auth/sign-up` - Sign up page (protected by `guestGuard`)
- `/home` - Home page (public)
- `/events` - Events page (protected by `authGuard`)
- Other routes can be protected by adding `canActivate: [authGuard]`

---

## Project Structure

```
src/
├── app/
│   ├── core/                    # Core functionality
│   │   ├── services/           # Shared services
│   │   ├── guards/             # Route guards
│   │   └── interceptors/      # HTTP interceptors
│   ├── features/               # Feature modules
│   │   ├── auth/              # Authentication feature
│   │   ├── events/            # Events feature
│   │   ├── tickets/           # Tickets feature
│   │   └── feedback/          # Feedback feature
│   ├── Layout/                # Layout components
│   │   ├── header/            # Header with auth state
│   │   ├── footer/            # Footer
│   │   └── home/              # Home page
│   └── models/                # Data models
│       ├── user.ts            # User interfaces
│       └── event.ts          # Event interfaces
├── db.json                    # Mock database (json-server)
└── assets/                    # Static assets
```

---

## Running the Application

### Development Server

1. **Start the mock backend:**
   ```bash
   json-server --watch src/db.json --port 3000
   ```

2. **Start the Angular development server:**
   ```bash
   ng serve
   ```

3. **Navigate to:** `http://localhost:4200`

The application will automatically reload if you change any of the source files.

### Build

```bash
ng build
```

The build artifacts will be stored in the `dist/` directory.

### Running Unit Tests

```bash
ng test
```

Executes the unit tests via [Karma](https://karma-runner.github.io).

---

## Code Scaffolding

Run `ng generate component component-name` to generate a new component. You can also use:

```bash
ng generate directive|pipe|service|class|guard|interface|enum|module
```

---

## Further Help

- [Angular Documentation](https://angular.dev)
- [Angular CLI Overview](https://angular.dev/tools/cli)
- [RxJS Documentation](https://rxjs.dev)

---

## Authentication Flow

1. **User Registration:**
   - User fills sign-up form
   - System checks if email exists
   - Creates new user in database
   - Generates token and stores in localStorage
   - Redirects to home page

2. **User Login:**
   - User enters credentials
   - System validates email and password
   - Generates token and stores in localStorage
   - Redirects to requested page or home

3. **Protected Route Access:**
   - **Private routes** (e.g., `/tickets`, `/events`):
     - User navigates to protected route
     - `authGuard` checks authentication
     - If not authenticated, redirects to `/auth/sign-in?returnUrl=<original-url>`
     - After login, redirects to originally requested page
   - **Auth pages** (e.g., `/auth/sign-in`, `/auth/sign-up`):
     - User navigates to auth page
     - `guestGuard` checks if user is already authenticated
     - If authenticated, redirects to `/home` (prevents unnecessary access)
     - If not authenticated, allows access to auth page

4. **API Requests:**
   - Component makes HTTP request
   - Interceptor adds Authorization header
   - Server validates token
   - Response returned to component

5. **Logout:**
   - User clicks logout
   - Token and user data removed from localStorage
   - Observables notify subscribers
   - User redirected to home page

---

## Security Notes

⚠️ **Important:** This implementation uses a simple base64 token for development purposes. For production:

1. **Use JWT tokens** from a secure backend
2. **Implement token refresh** mechanism
3. **Add token expiration** handling
4. **Use HTTPS** for all API calls
5. **Implement proper password hashing** (bcrypt, etc.)
6. **Add CSRF protection**
7. **Implement rate limiting** for login attempts
8. **Store sensitive data** in httpOnly cookies instead of localStorage

---

## Troubleshooting

### Token not being added to requests
- Verify the interceptor is registered in `app.module.ts`
- Check that `provideHttpClient` includes `withInterceptors([authInterceptor])`

### User state not updating
- Ensure you're subscribing to `currentUser$` and `isAuthenticated$` observables
- Check that `setAuthData()` is called after successful login/signup

### Guard not working
- Verify `canActivate: [authGuard]` is added to route configuration
- Check that `isLoggedIn()` returns true when user is authenticated

### Redirect issues
- Ensure RouterModule is imported in your module
- Check that route paths match exactly (case-sensitive)

---

## License

This project is for educational purposes.
#   E v e n t H u b P l a t f o r m 
 
 
