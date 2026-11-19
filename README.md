# Synapsis Survey App

A Flutter app for field surveyors who need to fill in assessments with or without a connection. Built as a take-home assignment for Synapsis in 2024.

Surveyors log in, browse the assessment list, download an assessment for offline use, answer it question by question, and submit. When the device is offline, the app reads assessments from SQLite and saves answers locally so a background job can send them once the network is back.

## Features

- **Login with "remember me" and biometric unlock.** Credentials are checked against the API; a saved login can be unlocked with fingerprint or Face ID through `local_auth`.
- **Assessment list with pagination and pull to refresh.** More pages load as you scroll; each item shows when it was last downloaded.
- **Download for offline use.** One tap stores an assessment and all its questions in SQLite.
- **Question flow.** Questions render in a `PageView`, with a question picker dialog to jump between them. Supports multiple choice and other question types from the API.
- **Online or offline submit.** With a connection, answers go straight to the API. Without one, they are saved to a local table and a WorkManager task picks them up when the device reconnects.

## How the offline path works

Every repository call checks connectivity first (`NetworkInfo`, backed by `connectivity_plus`):

| Action | Online | Offline |
|---|---|---|
| Load assessment list | API (`ALL-SURVEY`) | Last downloaded list from SQLite |
| Load assessment detail | API (`DETAIL_SURVEY`) | Stored questions from SQLite |
| Submit answers | API (`SEND_SURVEY`) | Saved to the `TABLE_SAVE_ANSWER` table |
| Login | API (`LOGIN_USER`) | Blocked with "No Internet Connection." |

On startup, `main.dart` registers a one-off WorkManager task with a `NetworkType.connected` constraint. When it runs, it reads the saved answers, posts them to `SEND_SURVEY`, and clears the table on success.

Errors cross layers as `Either<ErrorFetch, T>` (`dartz`), with `FetchApiFail` and `FetchLocalFail` telling the UI where a failure came from.

## Architecture

Feature-based Clean Architecture with Riverpod for state and dependency injection:

```
lib/
├── main.dart                  # Bootstrap, WorkManager sync task, router
├── core/
│   ├── api/                   # Dio client, cookie auth interceptor, ApiResult
│   ├── database/              # SQLite schema and connection (sqflite)
│   ├── di/                    # Riverpod providers (API, DB, DAO, repository, router, prefs)
│   ├── params/                # Request params (freezed)
│   ├── router/                # go_router routes: /, /login, /home, /assesment
│   └── util/                  # Connectivity, errors, messages, shared widgets
└── features/survey/
    ├── domain/                # Repository contract and use case
    ├── data/
    │   ├── model/             # freezed + json_serializable models
    │   ├── repositories/      # Online/offline switch per call
    │   └── sources/           # Remote (Dio) and local (SQLite DAO) sources
    └── presentation/          # Splash, Login, Menu, Survey detail
        └── */controller/      # StateNotifier controllers exposing AsyncValue
```

## Tech stack

| Concern | Package |
|---|---|
| State and DI | flutter_riverpod 2 (`StateNotifier`, `AsyncValue`) |
| Networking | dio 5 |
| Local storage | sqflite, shared_preferences |
| Background sync | workmanager |
| Connectivity | connectivity_plus |
| Navigation | go_router 13 |
| Models | freezed, json_serializable |
| Errors | dartz (`Either`) |
| Biometrics | local_auth |
| Config | flutter_dotenv |
| Layout | sizer, gap |

## Getting started

Requirements: Flutter with Dart SDK 3.3.1 or newer, and an Android or iOS device or emulator with biometrics set up if you want to try fingerprint login.

```bash
flutter pub get
cp .env.example .env        # then fill in the values, see below
dart run build_runner build --delete-conflicting-outputs
flutter run
```

`.env` is bundled as an asset, so the app will not build without it. It holds:

- `BASE_URL` and the endpoint paths `LOGIN_USER`, `ALL-SURVEY`, `DETAIL_SURVEY`, `SEND_SURVEY`
- `DB_NAME` and the SQLite table names (`TABLE_*`)
- `REMEMBER_KEY` and `REMEMBER_KEY_VALUE` for the shared preferences keys
- `TOKEN` and `REFRESH_TOKEN`, sent as cookies by `ApiInterceptor`

Never commit `.env`; it is listed in `.gitignore`.

## Known limitations

This was a time-boxed assignment, and a few things are unfinished:

- **No automated tests yet.** `test/widget_test.dart` is still the Flutter template. `mockito`, `riverpod_generator`, and `riverpod_lint` are in `dev_dependencies` but not used.
- **Background sync needs fixing before it can be trusted.** In `callbackDispatcher`, the request body is built as a Dart `Set` (`{ {...} }`) rather than a `List`, and the cleanup query deletes `where: 'id = 0'` while the table is keyed by `id_save`. Answers saved offline may not be sent or cleared as expected.
- **Auth tokens come from `.env`**, not from the login response, so the session does not refresh on its own.
- **"Remember me" stores the username and password as plain JSON in `SharedPreferences`.** A production build should keep only a token, in secure storage (Keychain / Android Keystore).
