# The Lucky Budget
> Personal Finance Manager

The Lucky Budget helps users manage funds, expenses, goals, and subscriptions — providing financial knowledge and insights to help achieve financial freedom.

---

## Collaborators

| Name | Role |
|---|---|
| Dillon Rinkwest | Lead Developer |
| Liam Swile | Developer |
| Patrick Shutte | Developer |

---

## Table of Contents

- [Features](#features)
- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Getting Started](#getting-started)
- [CI / CD](#ci--cd)
- [Database](#database)
- [Testing](#testing)
- [Permissions](#permissions)
- [Known Limitations](#known-limitations)
- [Demo Video](#demo-video)
- [References](#references)
- [Declaration of AI Usage](#declaration-of-ai-usage)

---

## Features

### Login / Register
- Email + password authentication via **Firebase Auth**
- **Google Sign-In** using the Android Credential Manager API — no manual token handling
- Strict password validation: minimum 8 characters, at least one uppercase letter, one lowercase letter, one digit, and one special character
- **Forgot password** flow sends a reset email via Firebase
- Auto-logout when the app goes to the background (configurable via `SessionManager`)

### Dashboard
- Personalised greeting using the authenticated user's display name
- **Category tiles** show spending per category for the current month at a glance
- **Recent transactions** list — the five most recent expense entries across all accounts
- **Savings goal snapshot** — a horizontally scrollable row of active goals showing progress bars and amounts remaining
- **Subscription tiles** — upcoming renewal reminders surfaced directly on the home screen

### Transactions
- Full chronological history of all expense entries
- Filter by date range, bank account, or category
- Each entry shows: amount (with currency symbol), category, account, date, and optional notes
- **Add Transaction** screen: pick a category from a dropdown, select a bank account, enter amount and date, attach an optional receipt image (camera or gallery)
- Transactions are written to Firestore and immediately reflected across all screens via LiveData observers

### Analytics
- **Date-range filter** — pick any start and end date using the Material date picker
- **Custom donut chart** (`DonutChartView`) — renders spending proportions per category using Android Canvas, with animated segment drawing and a centre-label showing total spend
- Per-account spending breakdown table with individual totals
- Per-category row list with amounts and percentage of total
- **PDF export** — `AnalyticsPdfExporter` generates a formatted PDF of the current date range's analytics and shares it via the system share sheet using `FileProvider`

### Category Totals
- Dedicated screen showing all categories and their total spend for the selected period
- Tappable rows navigate to a filtered transaction list for that category

### Bank Accounts
- Add multiple bank accounts (e.g. Cheque, Savings, Credit)
- Each account tracks its current balance and total spending
- Per-account spending is calculated from `expense_entries` linked to that account
- Accounts screen shows a summary card per account; tap to drill into that account's transactions

### Savings Goals
- Create a goal with a name, target amount, deadline date, and optional minimum / maximum daily spending bands
- **`SpendingGoalChartView`** — a custom bar chart that plots daily spending against the goal band, drawing a coloured "safe zone" band between the min and max thresholds
- Goals show a progress bar (amount saved vs. target) and a countdown to the deadline
- Mark a goal as complete; completed goals are archived separately

### Subscriptions
- Track recurring services (e.g. Netflix, Spotify, gym membership)
- Fields: service name, amount, billing cycle (monthly / yearly), start date, next renewal date, auto-pay toggle
- **Reminders** — opt-in notification reminders N days before renewal
- Dashboard subscription tiles surface the next-due subscription at a glance
- `SubscriptionAdapter` groups active vs. paused subscriptions

### Spending Insights (AI)
- Powered by **Google Gemini 2.5 Flash** via a direct OkHttp REST call to the `generateContent` endpoint
- The app assembles a structured prompt from the user's last 30 days of transactions, category totals, and savings goal progress, then sends it to Gemini
- The response is parsed and displayed as:
  - **Spending summary** — natural-language overview of where money went
  - **Overspending warnings** — categories that exceeded typical thresholds
  - **Budget suggestions** — category-level recommendations
  - **Savings advice** — personalised tips based on goal progress
- Falls back to a secondary API key (`GEMINI_API_KEY_2`) if the primary key is rate-limited
- Loading state shown with a progress indicator; errors display a user-friendly message

### Settings
- **Overspend alerts** — toggle push notifications when a category exceeds its budget
- **Subscription reminders** — global toggle for renewal reminder notifications
- **Multi-currency** — select a display currency; `CurrencyManager` fetches live exchange rates and converts all amounts on the fly
- **Profile management** — update display name and profile photo
- **Sign out** — clears the session and returns to the login screen

### Manage Categories
- Create custom expense categories with a name and colour
- Delete categories that have no linked transactions
- Categories are user-scoped in Firestore — each user has their own category list

---

## Architecture

Pulser DS follows the **MVVM (Model–View–ViewModel)** pattern recommended by Android Jetpack, layered as follows:

```
┌─────────────────────────────────────────────┐
│                   UI Layer                  │
│  Activities  ·  Adapters  ·  Custom Views   │
│         observe LiveData / StateFlow        │
└──────────────────────┬──────────────────────┘
                       │  calls
┌──────────────────────▼──────────────────────┐
│               ViewModel Layer               │
│  BankAccountViewModel  ·  CategoryViewModel │
│  ExpenseEntryViewModel  ·  SavingsGoalVM    │
│  SubscriptionViewModel  ·  UserViewModel    │
│  SpendingInsightViewModel                   │
│         exposes LiveData results            │
└──────────────────────┬──────────────────────┘
                       │  calls suspend fns
┌──────────────────────▼──────────────────────┐
│              Repository Layer               │
│  UserRepository  ·  CategoryRepository      │
│  ExpenseEntryRepository  ·  BankAccountRepo │
│  SavingsGoalRepository  ·  SubsRepository   │
│         reads / writes Firestore            │
└──────────────────────┬──────────────────────┘
                       │
┌──────────────────────▼──────────────────────┐
│               Data Sources                  │
│  Cloud Firestore  ·  Firebase Auth          │
│  Gemini REST API  ·  Currency Exchange API  │
└─────────────────────────────────────────────┘
```

### Key Design Decisions

| Decision | Rationale |
|---|---|
| **Firestore over Room** | Real-time cross-device sync without a manual sync layer; Part 2 used Room, Part 3 migrated to Firestore |
| **`SingleLiveEvent`** | Prevents one-shot UI events (toasts, navigation) from re-firing on screen rotation |
| **`ViewModelFactory`** | Repositories are injected into ViewModels via a factory, keeping ViewModels testable without Android framework dependencies |
| **`BaseActivity`** | All activities extend `BaseActivity`, which centralises the auto-logout logic and navigation bar setup |
| **`App.kt` lifecycle callbacks** | Registers `ActivityLifecycleCallbacks` to detect when the app moves to the background and trigger session timeout |
| **OkHttp over Retrofit for Gemini** | The Gemini endpoint requires a custom JSON body structure; a single OkHttp call is simpler than a Retrofit interface with one method |

### Data Flow Example — Adding a Transaction

```
AddTransactionActivity
  → user taps Save
  → ExpenseEntryViewModel.addEntry(entry)
      → ExpenseEntryRepository.addEntry(uid, entry)
          → Firestore.collection("expense_entries").add(doc)
  → LiveData<Result> emits Success
  → Activity observes → shows toast → finishes
  → TransactionsActivity re-observes its LiveData
      → ExpenseEntryRepository streams snapshot listener
          → list updates automatically
```

---

## Tech Stack

- **Language:** Kotlin
- **Min SDK:** 26 (Android 8.0 Oreo) · **Target / Compile SDK:** 36
- **Architecture:** MVVM (ViewModel + LiveData + Repository)
- **Database:** Cloud Firestore (Firebase) — real-time sync across devices
- **Authentication:** Firebase Auth — email/password + Google Sign-In (Credential Manager)
- **AI:** Google Gemini 2.5 Flash via OkHttp REST
- **UI:** XML layouts, ViewBinding, Material Design 3, custom `DonutChartView` & `SpendingGoalChartView`
- **Concurrency:** Kotlin Coroutines + `lifecycleScope`
- **Build:** Gradle 9.4.1 (Kotlin DSL) · AGP 9.2.1 · **CI:** GitHub Actions

### Key Dependencies

```toml
# Firebase
firebase-bom                        = "34.13.0"
firebase-auth                       (via BoM)
firebase-firestore                  (via BoM)
firebase-analytics                  (via BoM)

# Google Sign-In
credentials                         = "1.3.0"
credentials-play-services-auth      = "1.3.0"
googleid                            = "1.1.1"

# Networking
okhttp3                             = "4.12.0"

# Lifecycle / Coroutines
lifecycle-viewmodel-ktx             = "2.8.4"
kotlinx-coroutines-android          = "1.8.1"
kotlinx-coroutines-play-services    = "1.8.1"

# UI
material                            = "1.12.0"
constraintlayout                    = "2.2.0"
activity-ktx                        = "1.9.0"
```

---

## Project Structure

```
Pulserds_PROG7313/
├── app/
│   ├── google-services.json
│   ├── build.gradle.kts
│   └── src/main/
│       ├── AndroidManifest.xml
│       ├── java/com/example/pulserds_prog7313/
│       │   ├── App.kt                              # Application class — auto-logout lifecycle callbacks
│       │   ├── DonutChart.kt                       # Donut chart data model
│       │   ├── DonutChartView.kt                   # Custom Canvas donut chart view
│       │   ├── SpendingGoalChartView.kt            # Custom daily-spending bar chart with goal band
│       │   │
│       │   ├── data/                               # Firestore document models
│       │   │   ├── User.kt
│       │   │   ├── Category.kt
│       │   │   ├── ExpenseEntry.kt
│       │   │   ├── BankAccount.kt
│       │   │   ├── AccountSpending.kt
│       │   │   ├── SavingsGoal.kt
│       │   │   └── Subscription.kt
│       │   │
│       │   ├── repository/                         # Firestore data access layer
│       │   │   ├── UserRepository.kt
│       │   │   ├── CategoryRepository.kt
│       │   │   ├── ExpenseEntryRepository.kt
│       │   │   ├── BankAccountRepository.kt
│       │   │   ├── SavingsGoalRepository.kt
│       │   │   └── SubscriptionRepository.kt
│       │   │
│       │   ├── service/
│       │   │   └── GeminiSpendingService.kt        # Gemini 2.5 Flash REST integration
│       │   │
│       │   ├── ui/                                 # Activities
│       │   │   ├── BaseActivity.kt
│       │   │   ├── MainActivity.kt                 # Login
│       │   │   ├── CreateAccActivity.kt            # Registration
│       │   │   ├── ForgotPasswordActivity.kt
│       │   │   ├── DashBoardActivity.kt
│       │   │   ├── TransactionsActivity.kt
│       │   │   ├── AddTransactionActivity.kt
│       │   │   ├── AnalyticsActivity.kt
│       │   │   ├── CategoryTotalsActivity.kt
│       │   │   ├── AccountsActivity.kt
│       │   │   ├── AddAccountActivity.kt
│       │   │   ├── SavingsGoalsActivity.kt
│       │   │   ├── CreateSavingsGoalActivity.kt
│       │   │   ├── ManageSubscriptionsActivity.kt
│       │   │   ├── AddSubscriptionActivity.kt
│       │   │   ├── SpendingInsightActivity.kt      # AI spending analysis
│       │   │   ├── SettingsActivity.kt
│       │   │   ├── ManageCategoriesActivity.kt
│       │   │   └── adapters/
│       │   │       ├── AccountSpendingAdapter.kt
│       │   │       ├── CategoryRowAdapter.kt
│       │   │       ├── CategoryTileAdapter.kt
│       │   │       ├── DashSavingsAdapter.kt
│       │   │       ├── SpendingRowAdapter.kt
│       │   │       ├── SubscriptionAdapter.kt
│       │   │       ├── SubscriptionTileAdapter.kt
│       │   │       └── TransactionAdapter.kt
│       │   │
│       │   ├── utils/
│       │   │   ├── SessionManager.kt               # SharedPreferences session handling
│       │   │   ├── CurrencyManager.kt              # Multi-currency with live exchange rates
│       │   │   ├── AnalyticsPdfExporter.kt         # Analytics PDF export
│       │   │   └── SingleLiveEvent.kt
│       │   │
│       │   └── viewmodel/
│       │       ├── BankAccountViewModel.kt
│       │       ├── CategoryViewModel.kt
│       │       ├── ExpenseEntryViewModel.kt
│       │       ├── SavingsGoalViewModel.kt
│       │       ├── SubscriptionViewModel.kt
│       │       ├── SpendingInsightViewModel.kt
│       │       ├── UserViewModel.kt
│       │       └── ViewModelFactory.kt
│       │
│       └── res/
│           ├── layout/                             # 20 XML layout files
│           ├── values/                             # colors, strings, themes, dimens, attrs
│           ├── drawable/
│           ├── mipmap-anydpi/
│           └── xml/                               # backup_rules, data_extraction_rules, file_paths
│
├── gradle/
│   ├── libs.versions.toml
│   ├── wrapper/
│   │   └── gradle-wrapper.properties              # Gradle 9.4.1
│   └── gradle-daemon-jvm.properties
│
├── .github/
│   └── workflows/
│       └── ci.yml                                 # GitHub Actions CI pipeline
│
├── build.gradle.kts                               # Root build file (AGP 9.2.1, Kotlin 2.2.21)
├── settings.gradle.kts                            # Project name: Pulserds_PROG7313
├── gradle.properties
├── gradlew / gradlew.bat
└── local.properties                               # sdk.dir + GEMINI_API_KEY (not committed)
```

---

## Getting Started

### Prerequisites

| Requirement | Version |
|---|---|
| Android Studio | Ladybug (2024.2.1) or newer |
| JDK | 17 |
| Android SDK | API 36 (compile & target) |
| Gradle | 9.4.1 (via wrapper — no manual install needed) |
| Android Gradle Plugin | 9.2.1 |

### 1. Clone the Repository

```bash
git clone https://github.com/EMWCCN/prog7313-poe-g2-pulser-ds.git
cd prog7313-poe-g2-pulser-ds
```

### 2. Configure `local.properties`

Create (or edit) `local.properties` in the **project root** — this file is not committed to Git:

```properties
sdk.dir=/path/to/your/Android/Sdk
GEMINI_API_KEY=your_gemini_api_key_here
GEMINI_API_KEY_2=your_gemini_api_key_2_here
```

> **Where to get a Gemini API key:** Visit [Google AI Studio](https://aistudio.google.com/app/apikey) and generate a free key. The Spending Insights screen will not load without it.

> **`sdk.dir` on common platforms:**
> - **Windows:** `C\:\\Users\\YourName\\AppData\\Local\\Android\\Sdk`
> - **macOS:** `/Users/YourName/Library/Android/sdk`
> - **Linux:** `/home/YourName/Android/Sdk`
>
> Android Studio sets this automatically when you open the project — you only need to add the `GEMINI_API_KEY` lines.

### 3. Firebase Setup

A valid `app/google-services.json` is already included in the repository. No additional Firebase setup is required to run the app. The file configures:

- Firebase project ID and app ID
- Google Sign-In OAuth client IDs
- Firestore database URL
- Firebase Analytics

### 4. Open in Android Studio

1. Launch Android Studio.
2. Select **File → Open** and choose the cloned project folder.
3. Wait for Gradle sync to complete (this may take a few minutes on first run).
4. If prompted, install any missing SDK components.

### 5. Run the App

**Option A — Android Studio (recommended):**
1. Connect a physical device (API 26+) via USB with USB Debugging enabled, or start an emulator (API 26+) from the AVD Manager.
2. Select your device in the toolbar.
3. Click **Run ▶**.

**Option B — Command line:**
```bash
# Build debug APK
./gradlew assembleDebug

# Output location
app/build/outputs/apk/debug/app-debug.apk
```

To install directly to a connected device:
```bash
./gradlew installDebug
```

### 6. Run Unit Tests

```bash
./gradlew testDebugUnitTest
```

Test results are written to `app/build/reports/tests/testDebugUnitTest/`.

### Troubleshooting

| Problem | Fix |
|---|---|
| **Gradle sync fails with "SDK not found"** | Open `local.properties` and verify `sdk.dir` points to your Android SDK installation |
| **`GEMINI_API_KEY` build error** | Ensure both `GEMINI_API_KEY` and `GEMINI_API_KEY_2` are present in `local.properties` — the build reads them as `BuildConfig` fields |
| **Google Sign-In fails at runtime** | The `google-services.json` SHA-1 fingerprint must match your debug keystore. Run `./gradlew signingReport` and add the printed SHA-1 to the Firebase Console under your Android app |
| **Firestore permission denied** | The Firestore security rules require the user to be authenticated. Make sure you are signed in before making any read/write calls |
| **Emulator has no internet** | Gemini API and currency exchange calls require a network connection. Ensure the emulator has network access or test on a physical device |
| **`google-services.json` missing on CI** | The CI workflow writes the file from the `GOOGLE_SERVICES_JSON` repository secret — see [CI / CD](#ci--cd) for setup |

---

## CI / CD

A **GitHub Actions** workflow (`.github/workflows/ci.yml`) triggers automatically on every push and pull request to `main`, as well as manually via `workflow_dispatch`.

### Pipeline Steps

| Step | Action |
|---|---|
| 1 | Checkout repository |
| 2 | Set up JDK 17 (Temurin distribution) |
| 3 | Set up Android SDK |
| 4 | Generate `local.properties` from repository secrets (`GEMINI_API_KEY`, `GEMINI_API_KEY_2`, `sdk.dir`) |
| 5 | Cache Gradle packages |
| 6 | Grant execute permission for `gradlew` |
| 7 | Verify Gradle wrapper checksum |
| 8 | Write `google-services.json` from `GOOGLE_SERVICES_JSON` secret |
| 9 | Build debug APK (`assembleDebug`) |
| 10 | Run JUnit unit tests (`testDebugUnitTest`) |
| 11 | Run Android Lint (`lintDebug`) |
| 12–14 | Upload lint report, test results, and debug APK as build artifacts |

### Required Repository Secrets

| Secret | Description |
|---|---|
| `GEMINI_API_KEY` | Primary Gemini API key |
| `GEMINI_API_KEY_2` | Secondary / fallback Gemini API key |
| `GOOGLE_SERVICES_JSON` | Full contents of `google-services.json` (base64-encoded or raw JSON) |

To add secrets: **GitHub repo → Settings → Secrets and variables → Actions → New repository secret**.

### Build Artifacts

After each successful run, the following files are available for download from the workflow summary page:

- `app-debug.apk` — the compiled debug APK
- `lint-results-debug.html` — Android Lint report
- `test-results/` — JUnit XML test results

---

## Database

Pulser DS uses **Cloud Firestore** as its sole database. All data is stored per user under their Firebase UID, enabling real-time sync across devices.

### Collections

| Collection | Description |
|---|---|
| `users` | User profiles linked to Firebase Auth |
| `categories` | User-defined expense categories |
| `expense_entries` | Transaction records linked to a category and bank account |
| `bank_accounts` | Bank accounts with balance tracking |
| `savings_goals` | Savings targets with deadlines and min/max spending goal bands |
| `subscriptions` | Recurring subscriptions with billing cycle, renewal date, and reminder settings |

### Document Schemas

#### `users/{uid}`
```
displayName    : String
email          : String
photoUrl       : String?
createdAt      : Timestamp
currency       : String          // e.g. "ZAR", "USD"
overspendAlerts: Boolean
subReminders   : Boolean
```

#### `users/{uid}/categories/{categoryId}`
```
name           : String
colour         : String          // hex colour code
createdAt      : Timestamp
```

#### `users/{uid}/expense_entries/{entryId}`
```
amount         : Double
categoryId     : String
categoryName   : String
accountId      : String
accountName    : String
date           : Timestamp
notes          : String?
receiptUrl     : String?
```

#### `users/{uid}/bank_accounts/{accountId}`
```
name           : String          // e.g. "Cheque Account"
balance        : Double
createdAt      : Timestamp
```

#### `users/{uid}/savings_goals/{goalId}`
```
name           : String
targetAmount   : Double
currentAmount  : Double
deadline       : Timestamp
minDailySpend  : Double?
maxDailySpend  : Double?
isComplete     : Boolean
createdAt      : Timestamp
```

#### `users/{uid}/subscriptions/{subscriptionId}`
```
serviceName    : String
amount         : Double
billingCycle   : String          // "monthly" | "yearly"
startDate      : Timestamp
nextRenewal    : Timestamp
autoPay        : Boolean
reminder       : Boolean
reminderDays   : Int             // days before renewal to notify
isActive       : Boolean
```

> A local Room schema export is retained in `app/schemas/` from the Part 2 implementation (version 10). Room is no longer used at runtime.

---

## Testing

### Unit Tests

Unit tests live in `app/src/test/` and are run with:

```bash
./gradlew testDebugUnitTest
```

The test suite covers:

| Area | What is tested |
|---|---|
| **Password validation** | All combinations of missing character classes, length edge cases |
| **Date formatting** | Date range filter logic, boundary dates |
| **Currency conversion** | Exchange rate application, rounding behaviour |
| **Analytics calculations** | Category totals, percentage calculations, date range filtering |
| **Savings goal progress** | Progress percentage calculation, deadline proximity |

### Instrumented Tests

Instrumented tests (Espresso) live in `app/src/androidTest/`. These require a connected device or emulator and are not run on CI to avoid requiring an emulator in the GitHub Actions environment.

```bash
./gradlew connectedAndroidTest
```

### Test Reports

After running tests, open the HTML report in a browser:

```
app/build/reports/tests/testDebugUnitTest/index.html
```

---

## Permissions

| Permission | Reason |
|---|---|
| `INTERNET` | Firebase, Gemini API, and live currency exchange rate calls |
| `CAMERA` | Attach receipt photos to transactions (declared as not required) |
| `READ_MEDIA_IMAGES` (API 33+) | Select images from gallery |
| `READ_EXTERNAL_STORAGE` (API ≤ 32) | Select images from gallery on older devices |

Permissions are requested at runtime following the Android runtime permissions model. The camera and storage permissions are optional — the app functions fully without them.

---

## Known Limitations

| Limitation | Detail |
|---|---|
| **Offline support** | Firestore's local persistence cache allows reads while offline, but writes will queue and sync when connectivity is restored — no explicit offline indicator is shown to the user |
| **Receipt image storage** | Receipt photos are stored locally on the device; they are not uploaded to Firebase Storage, so they will not sync to other devices |
| **Currency rates** | Exchange rates are fetched once per session at app launch; they do not update in real time during a session |
| **Gemini rate limits** | The free tier of the Gemini API has per-minute rate limits; rapid successive calls to the Spending Insights screen may fall back to the secondary key or briefly fail |
| **No data export** | Only the Analytics screen supports PDF export; transactions cannot be exported to CSV or spreadsheet format |
| **Single-user device** | Sessions are stored in `SharedPreferences` and are not isolated per Android user account on shared devices |

---

## Demo Video

▶️ [https://youtu.be/pipU8XDkbBA](https://youtu.be/7QyxXqyV3cc)

---

## References

[1] Google, "Get started with Cloud Firestore," *Firebase Documentation*, 2024. [Online]. Available: https://firebase.google.com/docs/firestore/quickstart. [Accessed: 30 May 2026].

[2] Google, "Firebase Authentication," *Firebase Documentation*, 2024. [Online]. Available: https://firebase.google.com/docs/auth. [Accessed: 30 May 2026].

[3] Firebase, "Authenticate with Google on Android," Google, 27 May 2026. [Online]. Available: https://firebase.google.com/docs/auth/android/google-signin. [Accessed: 30 May 2026].

[4] Firebase, "Add multi-factor authentication to your Android app," Google, 27 May 2026. [Online]. Available: https://firebase.google.com/docs/auth/android/multi-factor. [Accessed: 30 May 2026].

[5] Google, "ViewModel overview," *Android Developers*, 2024. [Online]. Available: https://developer.android.com/topic/libraries/architecture/viewmodel. [Accessed: 28 Apr. 2026].

[6] Google, "LiveData overview," *Android Developers*, 2024. [Online]. Available: https://developer.android.com/topic/libraries/architecture/livedata. [Accessed: 28 Apr. 2026].

[7] JetBrains, "Coroutines guide," *Kotlin Documentation*, 2024. [Online]. Available: https://kotlinlang.org/docs/coroutines-guide.html. [Accessed: 28 Apr. 2026].

[8] Google, "Guide to app architecture," *Android Developers*, 2024. [Online]. Available: https://developer.android.com/topic/architecture. [Accessed: 28 Apr. 2026].

[9] Google, "View Binding," *Android Developers*, 2024. [Online]. Available: https://developer.android.com/topic/libraries/view-binding. [Accessed: 28 Apr. 2026].

[10] Google, "Credential Manager," *Android Developers*, 2024. [Online]. Available: https://developer.android.com/identity/sign-in/credential-manager. [Accessed: 30 May 2026].

[11] Google, "Gemini API — generateContent REST reference," *Google AI for Developers*, 2025. [Online]. Available: https://ai.google.dev/api/generate-content. [Accessed: 30 May 2026].

[12] Square Inc., *OkHttp*, ver. 4.12.0. [Software]. Available: https://square.github.io/okhttp.

[13] Google, "Canvas and drawables," *Android Developers*, 2024. [Online]. Available: https://developer.android.com/develop/ui/views/graphics/drawables. [Accessed: 28 Apr. 2026].

[14] Google, "SharedPreferences — save key-value data," *Android Developers*, 2024. [Online]. Available: https://developer.android.com/training/data-storage/shared-preferences. [Accessed: 28 Apr. 2026].

[15] Google, "Requesting permissions at runtime," *Android Developers*, 2024. [Online]. Available: https://developer.android.com/training/permissions/requesting. [Accessed: 28 Apr. 2026].

[16] Google, "Share files with FileProvider," *Android Developers*, 2024. [Online]. Available: https://developer.android.com/reference/androidx/core/content/FileProvider. [Accessed: 28 Apr. 2026].

[17] GitHub, "GitHub Actions documentation," *GitHub Docs*, 2024. [Online]. Available: https://docs.github.com/en/actions. [Accessed: 28 Apr. 2026].

[18] Gradle Inc., "Gradle Kotlin DSL primer," *Gradle Documentation*, 2024. [Online]. Available: https://docs.gradle.org/current/userguide/kotlin_dsl.html. [Accessed: 28 Apr. 2026].

[19] JUnit Team, *JUnit*, ver. 4.13.2. [Software]. Available: https://junit.org/junit4.

[20] Google, *AndroidX Test Espresso Core*, ver. 3.6.1. [Software]. Available: https://developer.android.com/training/testing/espresso.

[21] F. van Puffelen, "Save a Room Database file in Firebase," Stack Exchange Inc, 22 August 2021. [Online]. Available: https://stackoverflow.com/questions/68878022/save-a-room-database-file-in-firebase. [Accessed: 30 May 2026].

[22] A. Mamo, "Android Room and Firebase in a MVVM Architecture - Offline First Application," Stack Exchange Inc, 18 November 2022. [Online]. Available: https://stackoverflow.com/questions/74484239/android-room-and-firebase-in-a-mvvm-architecture-offline-first-application. [Accessed: 30 May 2026].

[23] S. Visweswaran, "Create PDF from layout XML in Android Studio using PDFDocument class," Medium, 27 August 2021. [Online]. Available: https://medium.com/@strawberryinc0531/convert-xml-to-pdf-in-android-studio-using-pdfdocument-class-757dee166d50. [Accessed: 30 May 2026].

[24] A. Van Gasse, "Generate PDF from Android," Stack Exchange Inc, 24 November 2016. [Online]. Available: https://stackoverflow.com/questions/40379451/generate-pdf-from-android. [Accessed: 30 May 2026].

---

## Declaration of AI Usage

### Usage 1 — Login page troubleshooting
**Tool:** Claude AI · **Date:** 10/04/2026
**Purpose:** Understand how login functionality should be structured and fix an ID reference issue between classes.
**Link:** https://claude.ai/share/dd1caa85-5884-4b9d-b899-88d3234fc2d8
**Action:** Generated code was not used — a different approach was found independently. The ID issue was also resolved manually.

---

### Usage 2 — Create account layout overlap
**Tool:** Claude AI · **Date:** 16/04/2026
**Purpose:** Fix a layout overlap issue between interactive UI elements.
**Link:** https://claude.ai/share/88a5d0cb-a044-414b-be2e-2aaac0ca8a9c
**Action:** Generated code was used only to fix the layout error; all other code was written manually.

---

### Usage 3 — Image upload feature investigation
**Tool:** Claude AI · **Date:** 27/04/2026
**Purpose:** Determine whether the image upload feature worked correctly (button was routing to Google Drive instead of the gallery).
**Link:** https://claude.ai/share/40b1dbf3-4c4f-4ff2-b609-a29acc39e7c0
**Action:** Response was used to understand and confirm the feature's behaviour. No generated code was adopted.

---

### Usage 4 — Week filter debugging
**Tool:** Claude AI · **Date:** 28/04/2026
**Purpose:** Identify why the week filter returned no results despite entries existing within the filtered period.
**Links:** https://claude.ai/share/061c7497-53d9-43e7-93b0-9900f54a475b · https://claude.ai/share/353e3527-cce5-4e5c-a5ab-ae4d3d0fca40
**Action:** A minor date-formatting change suggested by the tool was tested to verify whether it resolved the filter.

---

### Usage 5 — GitHub Actions CI workflow failure
**Tool:** Claude AI · **Date:** 28/04/2026
**Purpose:** Identify the cause of a workflow failure during the test step.
**Link:** https://claude.ai/share/9cdf5f21-1522-47c3-8d6d-92ae827bc0f5
**Action:** Output was used to evaluate which fix option best suited the workflow configuration.

---

### Usage 6 — GitHub CI Workflow Fix
**Tool:** Claude AI · **Date:** 30/05/2026
**Purpose:** Get an explanation of why the CI workflow failed in GitHub and steps on how to resolve it.
**Link:** https://claude.ai/share/472ba97e-5aee-4478-9f37-1f978c972253
**Action:** Followed the provided steps to fix the failing CI issue.

---

### Usage 7 — BaseActivity Class Build Error
**Tool:** Claude AI · **Date:** 30/05/2026
**Purpose:** Understand why the project build was failing due to an error in the BaseActivity class.
**Link:** https://claude.ai/share/9ed82f06-f6f3-485e-b99b-27fe4ebf307a
**Action:** Followed instructions on how to fix the error.

---

### Usage 8 — Correct Referencing Guide
**Tool:** Claude AI · **Date:** 30/05/2026
**Purpose:** Aid in referencing methods correctly without making modifications to any existing code.
**Link:** https://claude.ai/share/a916b1ac-ede1-4229-bcad-eca0de3a7885
**Action:** Used the allocations provided as a guide to manually allocate the references correctly.

---

### Usage 9 — Gemini API Fix
**Tool:** Claude AI · **Date:** 31/05/2026
**Purpose:** Identify why the AI insights feature was failing within the application and learn how to resolve it.
**Link:** https://claude.ai/share/f6701b7a-6d28-462c-a541-0ad39e45ad2c
**Action:** Followed the provided instructions to fix the API integration error.

---

### Usage 10 — Testing Document Error Debugging
**Tool:** Claude AI · **Date:** 31/05/2026
**Purpose:** Get an explanation for specific errors identified in the testing document and how to address them.
**Link:** https://claude.ai/share/89bf4806-22c6-4ec5-ab3a-25b63e93722d
**Action:** Followed instructions to fix the highlighted errors.

---

### Usage 11 — Dynamic Navigation Bar Implementation
**Tool:** Gemini · **Date:** 01/06/2026
**Purpose:** Assist in creating, structuring, and fixing a dynamic navigation bar within the application.
**Link:** https://g.co/gemini/share/faf4df7da882
**Action:** Followed the instructions to successfully implement and troubleshoot the navigation bar.

---

### Usage 12 — Database Duplicate Display Resolution
**Tool:** Gemini · **Date:** 01/06/2026
**Purpose:** Explain and fix a duplicate display issue across the database and application by ensuring only a single entry is displayed even if duplicates exist.
**Link:** https://g.co/gemini/share/bf95f422bd74
**Action:** Followed the instructions to handle the duplicate entries and fix the display output.
