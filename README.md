# TWO — Client Project & Contract Management

A Flutter client application for managing the full client-side lifecycle of software projects and contracts. Clients can create project requests, track progress, review and electronically sign contracts, follow meetings, receive in-app notifications, and manage their profile from a single mobile-first interface.

This repository contains the client-facing app; all business data is served by an external REST API.

---

## Table of Contents

- [Overview](#overview)
- [Key Features](#key-features)
- [Tech Stack](#tech-stack)
- [Architecture](#architecture)
- [Directory Structure](#directory-structure)
- [Navigation & Routes](#navigation--routes)
- [Usage Guide](#usage-guide)
- [Quality & Tooling](#quality--tooling)
- [Localization](#localization)
- [Production Checklist](#production-checklist)
- [Known Limitations](#known-limitations)

---

## Overview

**TWO** (internally named *Team Work Organizer*) is a Flutter application built around a client workflow:

1. A client signs in with credentials issued by the backend.
2. The client creates a project request and uploads supporting documents.
3. The client tracks project status, meetings, and edit requests.
4. When a contract is issued, the client can view the PDF, request edits, or sign it directly in the app.
5. Notifications and a profile dashboard keep the client informed.

The app is structured with a feature-first, Clean Architecture-inspired layout and uses BLoC for state management, `go_router` for navigation, `get_it` for dependency injection, and `dartz` for functional error handling.

---

## Key Features

### Authentication & Session Management
- Splash screen with token-based automatic routing.
- Email/password login against the REST backend.
- Bearer-token authentication for protected endpoints.
- Session persistence with `shared_preferences`.
- Logout with confirmation dialog and token cleanup.

### Onboarding
- Three-page onboarding carousel with custom illustrations, animations, and skip/next actions.
- Automatic routing to onboarding or home based on stored authentication state.

### Client Dashboard (Home)
- Personalized greeting using the authenticated user's profile.
- Recent projects carousel with status and visibility indicators.
- Recent contracts list.
- Unread notification count/list access.
- Pull-to-refresh for dashboard data.
- Skeleton loading placeholders and empty/error states.

### Project Management
- Browse all client projects with search-style UI and status cards.
- View project details including type, description, requirements, team, visibility, and contract status.
- Create projects with a two-step stepper:
  - Project type, description, key requirements, and optional PDF document upload.
  - Cooperation type and preferred contact time.
- Update existing project details and visibility.
- Rate completed projects on a 0–10 scale.
- Send edit requests with a message.
- View project edit-request history.
- View project meetings and meeting details.

### Contract Management
- Browse contracts with status filters.
- View contract details and approval/signature status.
- Open contract PDFs in an embedded viewer using Syncfusion PDF Viewer.
- Request contract edits.
- Capture a handwritten signature in-app and submit it as a Base64 PNG.
- Contract status indicators for manager approval, project manager approval, admin signature, and client signature.

### Notifications
- Unread notification list with project context.
- Notification cards for updates, meetings, and project events.
- Deep-link style navigation from notifications to related project content.

### Profile & Settings
- User profile fetch and display (name, email, avatar).
- Language selection UI for English and Arabic.
- Notification toggle UI placeholder.
- Logout action with session cleanup.

### UI & Developer Experience
- Centralized design system: colors, typography, spacing, radii, and reusable widgets.
- Custom buttons, cards, containers, dialogs, form fields, status boxes, and bottom navigation.
- Skeleton loading via `skeletonizer`.
- Lottie animations for loading, empty, success, error, and unauthorized states.
- SVG icons and illustrations.
- PDF/file picker integration with permission handling.
- Launcher icon generation configured for Android and iOS.
- Platform scaffolding for Android, iOS, Web, Windows, macOS, and Linux (mobile-first UI).

---

## Tech Stack

| Category | Technology / Package | Version |
|---|---|---|
| Framework | Flutter / Dart SDK | Dart `^3.8.1` |
| State Management | `flutter_bloc`, `bloc` | `^9.1.1`, `^9.0.0` |
| Routing | `go_router` | `^16.1.0` |
| Dependency Injection | `get_it` | `^8.2.0` |
| Functional Error Handling | `dartz` | `^0.10.1` |
| HTTP Client | `http` | `^1.4.0` |
| Local Storage | `shared_preferences` | `^2.5.3` |
| Networking Utilities | `internet_connection_checker` | `^3.0.1` |
| PDF Viewing | `syncfusion_flutter_pdfviewer` | `^27.1.48` |
| PDF Generation/Processing | `syncfusion_flutter_pdf` | `^27.1.48` |
| E-Signature | `signature` | `^5.4.1` |
| File Selection | `file_picker` | `^10.2.1` |
| File System | `path_provider` | `^2.1.5` |
| Permissions | `permission_handler` | `^12.0.1` |
| SVG Rendering | `flutter_svg` | `^2.2.0` |
| Lottie Animations | `lottie` | `^3.3.1` |
| Icons | `iconsax` | `^0.0.8` |
| Skeleton Loading | `skeletonizer` | `^2.1.0+1` |
| Launcher Icons | `flutter_launcher_icons` | `^0.14.4` |
| Linting | `flutter_lints` | `^5.0.0` |
| Fonts | Poppins, Almarai | Bundled in `assets/fonts/` |

> Note: The UI layer also references `EasyLocalization` APIs (`.tr()`, `context.setLocale`) and translation files under `assets/lang/`, but the `easy_localization` package is not currently declared in `pubspec.yaml`. See [Localization](#localization).

---

## Architecture

The codebase follows a **feature-first, layered Clean Architecture** pattern:

| Layer | Responsibility | Location |
|---|---|---|
| Presentation | Screens, widgets, dialogs, UI state rendering | `lib/features/*/pages`, `lib/features/*/widgets`, `lib/core/widgets` |
| Application / State | BLoC events, state, and UI-triggered actions | `lib/features/app/bloc` |
| Domain | Repository contracts and use cases | `lib/features/app/domain` |
| Data | Remote datasource, DTOs/models, repository implementation | `lib/features/app/data` |
| Core | API clients, error/failure handling, network info, storage, use-case base types | `lib/core` |
| Configuration | Constants, asset paths, routes, theme, localization strings | `lib/config` |
| Dependency Injection | Service locator registrations | `lib/injection_container.dart` |

### Data Flow

```text
UI Widget
   │  add(Event)
   ▼
AppBloc (flutter_bloc)
   │  call(Param)
   ▼
UseCase
   │
   ▼
AppRepo (abstract)
   │
   ▼
AppRepoImpl ──► AppRemoteDatasource ──► HTTP API
   │                                      │
   └──────── Either<Failure, T> ◄─────────┘
                    │
                    ▼
              AppState (CasualStatus)
                    │
                    ▼
             BlocBuilder / BlocListener
```

### State Model

- A single `AppBloc` coordinates all feature state.
- `AppState` exposes status flags per operation using `CasualStatus`:
  `initial`, `loading`, `success`, `failure`, `noToken`, `notAuthorized`.
- UI classes (`BlocBuilder` / `BlocListener`) call helpers in `AppBlocStateHandling` to render loading, empty, error, success, and unauthorized states consistently.
- Authentication tokens are read from `SharedPreferencesServices` inside BLoC handlers and attached to authorized requests as `Authorization: Bearer <token>`.

### Error Handling

- HTTP clients (`GetApi`, `GetWithTokenApi`, `PostApi`, `PostWithTokenApi`, `DeleteApi`) check for HTTP 200 and throw a `ServerException` parsed from the backend `message` field otherwise.
- Repository methods wrap calls with `HandlingExceptionManager.wrapHandling`, returning `Either<Failure, T>`.
- BLoC folds the result into success or failure state, and the UI displays an error dialog or error status box.

### Dependency Injection

`lib/injection_container.dart` registers:

- `AppBloc` as a factory.
- All use cases and repositories as lazy singletons.
- `AppRemoteDatasource`, `NetworkInfo`, `SharedPreferences`, and `InternetConnectionChecker`.

---

## Directory Structure

```text
two_client_app/
├── assets/
│   ├── animation/               # Lottie JSON animations
│   ├── fonts/                   # Poppins and Almarai font families
│   ├── icons/
│   │   ├── auth/                # Auth-related SVG icons
│   │   └── home/                # Navigation and home SVG icons
│   ├── images/
│   │   ├── png/                 # Backgrounds, illustrations, logo
│   │   └── svg/                 # Splash and logo SVGs
│   └── lang/                    # English and Arabic translation JSON
├── lib/
│   ├── config/
│   │   ├── constants/           # API base URI, padding, sizes, radii
│   │   ├── paths/               # Asset paths and centralized strings
│   │   ├── routes/              # go_router router and route names
│   │   └── theme/               # Colors, typography, ThemeData
│   ├── core/
│   │   ├── api/                 # HTTP GET/POST/DELETE clients
│   │   ├── error/               # Exceptions, failures, validators
│   │   ├── functions/           # Helpers and BLoC state rendering
│   │   ├── models/              # Shared response models
│   │   ├── network/             # Connectivity checker and status enum
│   │   ├── param/               # Request parameter objects
│   │   ├── services/            # Token storage service
│   │   ├── usecases/            # UseCase base interfaces
│   │   └── widgets/             # Reusable UI components
│   ├── features/
│   │   ├── app/
│   │   │   ├── bloc/            # AppBloc, events, state
│   │   │   ├── data/            # Datasource, repository impl, models
│   │   │   └── domain/          # Repository contract and use cases
│   │   ├── auth/                # Login page, form, auth widgets
│   │   ├── contract/            # Contract list, details, PDF, signature
│   │   ├── home/                # Home dashboard and notifications
│   │   ├── intro/               # Splash and onboarding
│   │   ├── profile/             # Profile and settings
│   │   └── projects/            # Project list, create, update, meetings
│   ├── injection_container.dart # GetIt service locator setup
│   └── main.dart                # App entry point
├── pubspec.yaml
├── analysis_options.yaml
└── README.md
```

> The `test/` directory is not present in this repository yet. See [Quality & Tooling](#quality--tooling).

---

## Navigation & Routes

Navigation is managed with `go_router`. The authenticated area uses a `StatefulShellRoute.indexedStack` with five persistent branches:

| Branch | Path | Screen |
|---|---|---|
| Home | `/home` | Dashboard |
| Home | `/notification` | Unread notifications |
| Home | `/recent-project` | Recent projects |
| Projects | `/project` | Project list |
| Projects | `/project-detailes` | Project details (`extra: ProjectModel`) |
| Projects | `/project-edit-requests` | Project edit requests (`extra: projectId`) |
| Projects | `/project-meetings` | Project meetings (`extra: projectId`) |
| Projects | `/update-project` | Update project (`extra: ProjectModel`) |
| Create | `/create-project` | Create project stepper |
| Contracts | `/contract` | Contract list |
| Contracts | `/contract-details` | Contract details (`extra: ContractModel`) |
| Contracts | `/pdf-viewer/:pdfUrl/:id` | PDF viewer |
| Contracts | `/signature/:pdfUrl/:id` | Signature capture |
| Profile | `/profile` | Profile and settings |

Unauthenticated routes:

| Path | Screen |
|---|---|
| `/` | Splash |
| `/onboard` | Onboarding |
| `/login` | Login |

Route names are centralized in `lib/config/routes/app_route_config.dart`.

---

## Usage Guide

### First Launch

1. Start the app. The splash screen checks for a stored token.
2. Without a token, the onboarding flow opens.
3. Skip or complete onboarding to reach the login screen.
4. Sign in with a backend-issued client account.

### Dashboard

- The home screen loads the user's profile, recent projects, recent contracts, and unread notifications.
- Pull down to refresh dashboard data.
- Tap the avatar/greeting area to open the profile.
- Use **See All** to open the recent projects list.

### Create a Project

1. Tap the central **+** action in the bottom navigation.
2. In **Project Request**:
   - Choose a project type.
   - Select or add a project description.
   - Select or add one or more key requirements.
   - Optionally upload a PDF document.
3. In **Work Data**:
   - Choose the cooperation type.
   - Enter the preferred contact day/time.
4. Confirm to submit the project request.

### Manage a Project

From the project list, open a project card to view details. The overflow menu provides:

- **Rate Project** — submit a 0–10 rating.
- **Edit Project** — update description, requirements, type, cooperation type, and visibility.
- **View Meetings** — list scheduled project meetings.
- **View Edit Requests** — review previously submitted edit request messages.
- **Send Edit Message** — submit a new edit request to the project team.

### Review & Sign a Contract

1. Open the **Contract** tab.
2. Select a contract to view its details and approval statuses.
3. Open the PDF to read the contract.
4. From the PDF viewer, tap the signature action.
5. Draw a signature, clear it if needed, and save to submit it as a Base64 PNG.
6. Use the contract overflow menu to send an edit request if changes are required.

### Notifications & Profile

- Open the notification icon on the dashboard to view unread notifications.
- In **Profile**, view account details, switch the displayed language (scaffolded), toggle notifications (UI placeholder), or log out.

---

## Quality & Tooling

### Formatting

```bash
dart format .
```

### Static Analysis

```bash
flutter analyze
```

The project uses `flutter_lints` with `deprecated_member_use` ignored in `analysis_options.yaml`.

### Tests

```bash
flutter test
```

There is currently no `test/` directory. Recommended first additions:

- Unit tests for use cases and repository error mapping.
- BLoC tests for login, project creation, and contract signing.
- Widget tests for the create-project stepper and login form validation.
- Mock HTTP tests for datasource response parsing.

### Linting Notes

- `analysis_options.yaml` extends `package:flutter_lints/flutter.yaml`.
- Deprecated API usage is suppressed to accommodate existing Flutter APIs.

---

## Localization

The project includes translation catalogs:

- `assets/lang/en.json` — English
- `assets/lang/ar.json` — Arabic

The app entry point and profile page are written to support runtime locale switching through `EasyLocalization` and `.tr()`. However, the current repository is in a partial localization state:

- `easy_localization` is **not declared** in `pubspec.yaml`.
- The `EasyLocalization` import is missing from the files that reference it.
- `assets/lang/` is **not registered** in the `flutter.assets` section of `pubspec.yaml`.
- Many UI labels are still centralized as hardcoded English constants in `lib/config/paths/text_strings.dart`.

To complete localization:

1. Add the `easy_localization` package to `pubspec.yaml`.
2. Add the missing `easy_localization` import to `lib/main.dart` and `lib/features/profile/pages/profile_page.dart`.
3. Register `assets/lang/` under `flutter.assets`.
4. Replace `TextStrings.*` usages with `.tr()` keys where runtime translation is required.

Because the `EasyLocalization` symbols are currently unresolved, a fresh checkout will not compile until these steps are completed. The English string catalog remains available through the centralized `TextStrings` constants in the meantime.

---

## Production Checklist

Before shipping a release build:

- [ ] Replace the development `baseUri` with a production **HTTPS** endpoint.
- [ ] Add `android.permission.INTERNET` to `android/app/src/main/AndroidManifest.xml` (it is currently only present in the debug/profile manifests).
- [ ] Configure Android cleartext/network security if HTTP remains in use for any environment.
- [ ] Change the Android `applicationId` (`com.example.two_client_app`) and iOS bundle identifier to a production identifier.
- [ ] Configure a release signing key for Android; release currently uses the debug signing config.
- [ ] Update the web app title, description, and manifest branding (`web/index.html`, `web/manifest.json`).
- [ ] Replace the default launcher icons and application labels.
- [ ] Add the `easy_localization` dependency and complete the localization wiring if localization is required.
- [ ] Move authentication tokens to secure storage (`flutter_secure_storage`) instead of plain `SharedPreferences`.
- [ ] Add automated tests and a CI pipeline.
- [ ] Add an explicit project license.

---

## Known Limitations

- **Single global BLoC**: `AppBloc` owns all feature state. This is straightforward for a small app but should be split into feature-level BLoCs/Cubits as the application grows.
- **Hardcoded API URL**: There is no environment-based configuration; the base URI is compiled into the app.
- **Localization incomplete**: See [Localization](#localization).
- **No automated tests or CI**: The `test/` directory and workflow definitions are absent.
- **No push notifications**: Notifications are fetched from the REST API and displayed in-app.
- **No offline mode**: The connectivity checker is registered in DI but is not yet used to block or queue requests.
- **Placeholder settings**: The notification toggle in the profile screen is UI-only and does not persist or affect backend behavior.
- **Backend dependency**: Login, projects, contracts, meetings, notifications, and images all require the external backend to be reachable.

