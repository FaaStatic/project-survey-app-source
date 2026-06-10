# Synapsis Survey App

An **offline-first survey application** built with Flutter, designed for field surveyors who need to collect data with or without an internet connection.

## Key Features

- **Offline-first**: surveys stored locally with SQLite (sqflite), synced when connectivity returns (connectivity_plus + WorkManager background sync)
- **Biometric login** via local_auth
- Dynamic survey detail & submission flow
- Environment-based config with flutter_dotenv

## Architecture

Feature-based **Clean Architecture** with Riverpod:

```
lib/
├── core/
│   ├── api/        # Dio client setup
│   ├── database/   # SQLite layer
│   ├── di/         # Dependency injection
│   ├── router/     # go_router config
│   └── util/
└── features/survey/
    ├── domain/     # Entities, repository contracts, use cases
    ├── data/       # Models (freezed), repository impl, remote/local sources
    └── presentation/  # Splash, Login, Menu, Survey Detail
```

- **Functional error handling** with `dartz` (`Either<Failure, T>`)
- **Code generation**: freezed + json_serializable + riverpod_generator
- **Lint discipline**: riverpod_lint + custom_lint
- **Testing**: mockito + flutter_test

## Tech Stack

| Concern | Library |
|---|---|
| State management | Riverpod 2 (codegen) |
| Networking | Dio 5 |
| Local DB | sqflite |
| Background sync | workmanager |
| Navigation | go_router 13 |
| Models | freezed + json_serializable |
| Auth | local_auth (biometric) |

## Running

```bash
flutter pub get
dart run build_runner build --delete-conflicting-outputs
# create .env (see .env.example)
flutter run
```
