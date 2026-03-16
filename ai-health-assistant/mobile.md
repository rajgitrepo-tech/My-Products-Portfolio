# Mobile App — Deep Dive

> 26 production screens. Native Android performance. Built with Flutter for a market of 600M+ Android users in India.

[![Flutter](https://img.shields.io/badge/Flutter-Dart-02569B?style=flat&logo=flutter&logoColor=white)](https://flutter.dev)
[![Android](https://img.shields.io/badge/Platform-Android-3DDC84?style=flat&logo=android&logoColor=white)](https://developer.android.com)
[![Cognito](https://img.shields.io/badge/Auth-AWS_Amplify_Cognito-FF9900?style=flat&logo=amazonaws&logoColor=white)](https://aws.amazon.com/cognito)

---

## Why Flutter

| Consideration | Decision |
|--------------|---------|
| Target market | India — Android dominates at 95%+ market share |
| Performance | Flutter compiles to native ARM code — no JavaScript bridge |
| UI fidelity | Custom widgets, smooth 60fps animations, Material Design 3 |
| Future | Single codebase ready for iOS expansion without rewrite |
| Ecosystem | Rich package ecosystem for Razorpay, Amplify, audio, camera |

---

## Architecture

### State Management — Provider

Provider was chosen over Bloc/Riverpod for its simplicity and testability at this product stage.

```
main.dart
  └── MultiProvider
      ├── AuthService (Cognito tokens, user session)
      ├── ApiClient (HTTP + JWT injection)
      ├── SubscriptionApiClient (billing state)
      └── Screen-level ChangeNotifiers (local UI state)
```

### Service Layer

| Service | Responsibility |
|---------|---------------|
| AuthService | Cognito token management, login/logout, session persistence |
| CognitoService | Amplify SDK wrapper — OTP flow, token refresh |
| ApiClient | HTTP client with automatic JWT injection and retry logic |
| SubscriptionApiClient | Billing API calls, tier state management |
| VoiceService | Audio recording, S3 upload, voice query API |
| AudioCacheService | Local MP3 cache (MD5 key, 24h TTL, 50MB LRU eviction) |
| AudioRecorder | Microphone capture (16kHz mono, AAC/m4a) |
| RazorpayService | Payment SDK wrapper, success/failure callbacks |
| WellnessNotificationService | Local notification scheduling, channel management |
| StepCounterService | Pedometer integration, daily step tracking |
| CalorieDatabaseService | Indian food calorie lookup (local JSON database) |
| UserScopedStorage | SharedPreferences wrapper with user_id prefix isolation |
| DataFreshnessTracker | Cache invalidation — tracks when remote data needs refresh |
| ScreenRefreshController | Coordinates cross-screen data refresh after mutations |
| AppLifecycleManager | Handles foreground/background transitions, token refresh |

---

## Screen Inventory (26 Screens)

### Authentication Flow
| Screen | Purpose |
|--------|---------|
| login_screen | Phone number entry with country code |
| otp_screen | 6-digit OTP input with countdown timer and resend |
| register_screen | New user onboarding — name, age, gender, dietary preference, health goal |

### Core App
| Screen | Purpose |
|--------|---------|
| main_navigation_screen | Bottom navigation shell — routes to primary sections |
| home_screen | Dashboard — today's calories, shelf count, meals planned, quick actions |
| chat_screen_enhanced | Context-aware AI health coach — the primary feature |
| voice_chat_screen | Voice assistant interface — record, transcribe, play response |

### Nutrition & Food
| Screen | Purpose |
|--------|---------|
| food_log_screen | Daily nutrition log with calorie totals |
| food_capture_screen | Camera/gallery → Bedrock Vision → auto-populated log entry |
| bulk_review_screen | Review multiple AI-identified food items before logging |
| select_recipe_for_log_screen | Pick a saved recipe to log as a meal |

### Recipes
| Screen | Purpose |
|--------|---------|
| recipes_screen | Recipe library — user-created + AI-generated + pre-seeded |
| generate_recipe_screen | AI recipe generation from shelf inventory |
| recipe_detail_screen | Full recipe view — ingredients, steps, nutrition |

### Planning & Shelf
| Screen | Purpose |
|--------|---------|
| planner_screen | Weekly meal planner with nutrition summary |
| add_meal_screen | Add a meal to the plan for a specific day/time |
| shelf_screen | Pantry inventory — items, quantities, expiry dates |
| add_shelf_item_screen | Add item to pantry with quantity and expiry |

### Health & Settings
| Screen | Purpose |
|--------|---------|
| health_insights_screen | Nutrition trends, macro breakdown, activity data |
| wellness_reminder_screen | Set up scheduled wellness notifications |
| wellness_reminders_screen | Manage all active reminders |
| profile_screen | User health profile — goals, dietary preference, stats |
| settings_screen | App settings, account management |
| subscription_screen | Current subscription status and management |
| subscription_selection_screen | Tier comparison and upgrade flow |
| update_required_screen | Force update gate for breaking API changes |

---

## Key UX Patterns

### User-Scoped Storage
Every piece of locally cached data is prefixed with the user's Cognito `user_id`.
Switching accounts on the same device clears the previous user's data automatically.
This was a critical fix — early versions had cross-user data leakage on shared devices.

### Auto-Refresh Mixin
Screens that display live data (home, chat, shelf, planner) use `AutoRefreshMixin`.
When the app returns to foreground, `DataFreshnessTracker` checks if remote data is stale and triggers a background refresh — users always see current data without manual pull-to-refresh.

### Glassmorphic UI
Custom `GlassmorphicCard` widget used throughout — frosted glass effect with blur and transparency. Consistent visual language across all data cards.

### Floating Quick Access
`FloatingQuickAccess` widget provides persistent shortcuts to the most-used actions (add food, add shelf item, generate recipe) from any screen — reduces navigation depth for frequent tasks.

### Subscription Error Handling
`SubscriptionErrorHandler` utility intercepts 429 (usage limit) and 402 (subscription required) API responses and surfaces contextual upgrade prompts — users understand why they hit a limit and how to resolve it.

### Upgrade Prompt Dialog
`UpgradePromptDialog` widget shown when a usage limit is reached — displays current tier, what the limit is, and a direct CTA to the subscription selection screen.

---

## Voice UI Components

### VoiceButton Widget
Five distinct states with animated transitions:
- Idle — microphone icon, sage green
- Recording — stop icon, red, pulsing animation
- Processing — loading spinner, orange
- Playing — speaker icon, blue, pulsing animation
- Error — error icon, red with message

Usage counter displayed inline — warns when queries are running low (≤5 remaining).

### LanguageSelector Widget
Language selection dialog with native script display:
- 🇮🇳 हिंदी (Hindi)
- 🇬🇧 English
- 🇮🇳 தமிழ் (Tamil)
- 🇮🇳 తెలుగు (Telugu)
- 🇮🇳 മലയാളം (Malayalam)
- 🇮🇳 ಕನ್ನಡ (Kannada)

Selection persists across sessions via UserScopedStorage.

### AudioCacheService
Local MP3 cache for voice responses:
- Cache key: MD5(response_text + language_code)
- TTL: 24 hours
- Max size: 50MB (LRU eviction)
- ~70% cache hit rate on common health questions
- Instant playback for cached responses — zero API call

---

## Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| flutter | SDK | Framework |
| provider | ^6.1.1 | State management |
| http | ^1.1.0 | HTTP client |
| amplify_flutter | ^1.5.0 | AWS Amplify SDK |
| amplify_auth_cognito | ^1.5.0 | Cognito auth |
| razorpay_flutter | ^1.3.6 | Payment gateway |
| image_picker | ^1.0.7 | Camera + gallery |
| flutter_image_compress | ^2.1.0 | Image optimisation |
| audioplayers | ^5.2.1 | Audio playback |
| permission_handler | ^11.2.0 | Runtime permissions |
| flutter_local_notifications | ^17.0.0 | Wellness reminders |
| timezone | ^0.9.2 | Notification scheduling |
| pedometer | ^4.0.1 | Step counter |
| shared_preferences | ^2.2.2 | Local storage |
| connectivity_plus | ^5.0.2 | Network state |
| package_info_plus | ^5.0.1 | Version tracking |
| device_info_plus | ^9.0.0 | Device metadata |
| intl | ^0.18.1 | Date/number formatting |
| url_launcher | ^6.2.4 | External links |
| crypto | ^3.0.3 | MD5 for cache keys |
| path_provider | ^2.1.2 | File system paths |

---

## Build & Release

### Android Signing
- Keystore stored securely (not in repo)
- Signing config in `android/app/build.gradle`
- GitHub Actions secret for CI builds

### Version Strategy
- Version format: `major.minor.patch+buildNumber`
- Current: `1.0.92+97`
- 90+ production build iterations shipped

### Build Command
```bash
flutter build apk --release --split-per-abi
```
Produces separate APKs for arm64-v8a, armeabi-v7a, x86_64 — smaller download size per device.
