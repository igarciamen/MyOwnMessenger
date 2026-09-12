# Messenger

A WhatsApp-style real-time messaging application for Android, built as a full-stack mobile learning project covering the complete feature set of a modern chat app: real-time one-to-one messaging, user presence, push notifications, and peer-to-peer audio/video calling with a custom Firestore-based signaling layer.

# Demo
https://github.com/user-attachments/assets/c95522ec-aa93-48c4-b71b-eca88c1d0c73

## Features

- 🔐 Email/password authentication (register, login, logout)
- 💬 Real-time one-to-one text messaging, no manual refresh required
- 🟢 Online/offline presence indicator, updated live
- 🔔 Push notifications with sound and app icon badge, delivered promptly even in Doze mode
- 📞 Audio calls and video calls (WebRTC), working over both Wi-Fi and mobile data
- 🕒 Message timestamps (date + time)
- 🔒 Firestore & Realtime Database security rules — each user can only access their own data and conversations they participate in

Text, audio, and video are the only supported communication modes, file/attachment sharing is intentionally out of scope.

## Tech Stack

| Layer | Technology |
|---|---|
| Language | Kotlin |
| UI | Jetpack Compose, Material 3 |
| Architecture | MVVM, Hilt (dependency injection), Coroutines & Flow |
| Navigation | Jetpack Navigation Compose |
| Backend (BaaS) | Firebase Authentication, Cloud Firestore, Realtime Database, Cloud Messaging (FCM) |
| Real-time calling | WebRTC (Stream's `stream-webrtc-android` fork) |
| NAT traversal | STUN (Google, public) + TURN (Metered / Open Relay Project, own account) |
| Networking | OkHttp, Google Auth Library (OAuth2) for server-to-server FCM HTTP v1 calls |
| Background reliability | Foreground Service + WakeLock (keeps call audio alive with the screen off) |
| Testing | JUnit (unit), Espresso + Compose UI Testing (instrumented) |

## Architecture Overview

```
com.igarciamen.messenger/
├── data/         Repositories: Auth, Chat, Call, Presence, User
├── domain/       Models & pure logic: User, Chat, Message, Call,
│                 ChatIdBuilder, DateFormatter, AuthValidator,
│                 UserPresence, PresenceFormatter, FcmPayloadBuilder,
│                 IceCandidateData, IceCandidateMapper
├── di/           Hilt modules (FirebaseModule)
├── service/      NotificationHelper, MessengerMessagingService (FCM), FcmSender
├── webrtc/       WebRtcClient, CallForegroundService
├── ui/
│   ├── navigation/  NavGraph
│   ├── login/       AuthViewModel, LoginScreen, RegisterScreen
│   ├── home/        HomeViewModel, HomeScreen
│   ├── contacts/    ContactsViewModel, ContactsScreen
│   ├── chat/        ChatViewModel, ChatScreen
│   ├── call/        CallViewModel, CallScreen, IncomingCallViewModel
│   └── theme/
├── MessengerApplication.kt
└── MainActivity.kt
```

Each screen follows a ViewModel + repository pattern, with Firestore/Realtime Database listeners exposed as Kotlin `Flow`s and collected reactively in Compose via `collectAsState()`.

## Firebase Data Model

**Firestore:**
```
users/{uid}                      — profile, fcmToken
chats/{chatId}                   — participants, lastMessage, lastMessageTimestamp
chats/{chatId}/messages/{msgId}  — senderId, text, timestamp
calls/{callId}                   — callerId, calleeId, type, status, offer, answer
calls/{callId}/callerCandidates
calls/{callId}/calleeCandidates
```

**Realtime Database:**
```
status/{uid}                     — state ("online"/"offline"), lastChanged
```

Security rules restrict every read/write to the authenticated user's own data or conversations/calls they participate in, see `firestore.rules` and `database.rules.json` for the exact rule set.

## Setup

### Prerequisites
- Android Studio (latest stable)
- A Firebase project with **Authentication** (Email/Password), **Cloud Firestore**, **Realtime Database**, and **Cloud Messaging** enabled
- A [Metered](https://www.metered.ca) free account for TURN server credentials (500 MB/month)

### Steps
1. Clone the repository.
2. Add your `google-services.json` to `app/`.
3. Publish the security rules from `firestore.rules` and `database.rules.json` to your Firebase project.
4. Generate your own TURN credentials via the Metered REST API (never embed the secret key in the app — see `WebRtcClient.kt` comments for the exact flow) and update `WebRtcClient.kt` with your `username`/`password`.
5. Add your FCM service account key under `app/src/main/assets/` (excluded from version control) for server-to-server push notifications.
6. Build and run:
   ```
   ./gradlew assembleDebug
   ```

### Running tests
```
./gradlew test                        # 31 unit tests (JVM)
./gradlew connectedAndroidTest         # 4 instrumented tests (device/emulator)
```

## Known Limitations

- **MIUI devices**: aggressive battery/autostart restrictions can delay FCM token refresh and, in rare cases, block automated test installation (`INSTALL_FAILED_USER_RESTRICTED`). Enable "Autostart" and set battery usage to "No restrictions" for the app on affected devices.
- **TURN quota**: the free Metered plan provides 500 MB/month of relayed traffic, renewing automatically; sufficient for development and testing, not sized for production-scale usage.
- **No file/media attachments**: by design, only text, audio calls, and video calls are supported.

