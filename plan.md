# Tên dự án: AI Fitness Journal (Nhật ký Tập luyện & Dinh dưỡng Thông minh)

## 1. Định hướng & Công nghệ Chủ đạo
- **Nền tảng:** Android Native (Kotlin 100% · Jetpack Compose · Material 3).
- **Kiến trúc:** Modern Android Architecture (MVVM + Clean Architecture + Repository Pattern + Single Source of Truth).
- **Local Persistence:** Room Database (Offline-first) + DataStore (Preferences).
- **Backend & Cloud Services:** Firebase (Auth, Cloud Firestore, Cloud Storage, Cloud Functions, App Check, FCM, Crashlytics).
- **AI Stack:** Google Gemini 1.5 Flash (via Firebase Vertex AI SDK / Cloud Functions) + `text-embedding-004` + Firestore Vector Search (Hybrid RAG).

---

## 2. Tầm nhìn & Nguyên tắc Nghiệp vụ Cốt lõi
### Giá trị cốt lõi
1. **Ghi nhận dễ dàng:** Nhật ký tập luyện, dinh dưỡng, hình ảnh tiến trình và cảm nhận cá nhân.
2. **Chính xác tuyệt đối về con số:** Calories/Macros và khối lượng tạ được tính toán bằng thuật toán chuẩn xác trong code, không dựa hoàn toàn vào văn bản sinh ngẫu nhiên từ AI.
3. **Cá nhân hóa bằng Hybrid RAG:** AI Coach tư vấn chính xác dựa trên dữ liệu nhật ký thực tế của từng người dùng kết hợp với kiến thức thể hình chuẩn.
4. **Trải nghiệm Offline-First:** Toàn bộ nhật ký, lịch tập và dữ liệu dinh dưỡng cốt lõi phải xem và chỉnh sửa mượt mà ngay cả khi không có mạng.

### Các nguyên tắc kỹ thuật bắt buộc
- **AI chỉ đưa ra đề xuất:** Ảnh món ăn do AI Vision nhận diện hoặc giáo án do AI Coach gợi ý bắt buộc phải qua bước người dùng xác nhận/chỉnh sửa trước khi lưu chính thức.
- **Client-Side Image Compression:** Toàn bộ ảnh chụp từ CameraX (ảnh bữa ăn, ảnh body) phải được nén/resize xuống dưới 500KB (max resolution 1080p) tại Client trước khi lưu local hoặc upload Cloud Storage.
- **Offline Sync Engine:** Ghi vào Room trước (`syncStatus = PENDING`), dùng WorkManager đẩy lên Firestore ngầm theo chiến lược *Last-Write-Wins* dựa trên timestamp UTC.
- **Bảo mật & Kiểm soát Chi phí AI:** API Key AI bảo vệ hoàn toàn ở Backend/Firebase Vertex AI SDK với **Firebase App Check** và cơ chế **Rate Limiting/Quota** (giới hạn số lượt dùng AI/ngày) để tránh rủi ro chi phí.

---

## 3. Phạm vi Chức năng & Phân kỳ Triển khai

### GIAI ĐOẠN 1: Core App & Offline-First (MVP Cốt lõi · Tuần 1–8)
| Phân hệ | Chi tiết chức năng |
| :--- | :--- |
| **Authentication & Profile** | Đăng nhập Firebase Auth (Email/Google), thiết lập TDEE, BMR, mục tiêu (Tăng cơ/Giảm mỡ). |
| **Nutrition (Thủ công + Việt Nam)** | Bộ dữ liệu món ăn Việt Nam nạp sẵn (Pre-populated Room DB) + Tạo món cá nhân + Tính Calories/Macros chuẩn xác. |
| **Workout Logging** | Thư viện bài tập chuẩn + Ghi log Sets/Reps/Weight + Đếm thời gian nghỉ + Tính tổng Volume/Calories tiêu hao. |
| **Dashboard & Timeline** | Tổng quan ngày (Calorie In/Out, Progress Bar Macros, Buổi tập gần nhất) + Timeline nhật ký. |
| **Local DB & Sync Engine** | Room Local Database + WorkManager đẩy dữ liệu PENDING lên Cloud Firestore khi có mạng. |
| **Media & Storage** | CameraX chụp ảnh bữa ăn/tiến trình + Nén ảnh tại chỗ (Client Compression) + Coil hiển thị. |

### GIAI ĐOẠN 2: AI Integration & Analytics (Tuần 9–14)
| Phân hệ | Chi tiết chức năng |
| :--- | :--- |
| **AI Vision (Nhận diện món ăn)** | Chụp ảnh bữa ăn -> Gửi Gemini 1.5 Flash -> Trả về danh sách món & khối lượng ước tính -> Người dùng xác nhận/chỉnh sửa. |
| **AI Coach (Hybrid RAG Chatbot)** | Chatbot tư vấn ngữ cảnh. Kết hợp **Structured Context** (số liệu tập/ăn từ Room) + **Vector Search** (Firestore Vector Search & `text-embedding-004`). Hỗ trợ UI Streaming qua Compose Flow. |
| **AI Quota & Security** | Áp dụng Firebase App Check + Rate Limit (vd: Max 10 lượt Vision, 20 lượt AI Chat/ngày). |
| **Progress & Analytics** | Biểu đồ theo dõi cân nặng, tổng khối lượng tạ (Volume) tăng trưởng theo thời gian (dùng Compose Canvas / Vico Chart). |

### GIAI ĐOẠN 3: Health Integration & Polishing (Tuần 15–16)
| Phân hệ | Chi tiết chức năng |
| :--- | :--- |
| **Health Connect** | Tích hợp Android Health Connect (Đếm bước chân, theo dõi giấc ngủ, nhịp tim). |
| **Notifications** | Nhắc nhở tập luyện, uống nước và nạp dinh dưỡng via FCM / AlarmManager. |
| **Polishing & Release** | Test offline edge-cases, tối ưu hiệu năng Compose (Baseline Profiles), chuẩn bị phát hành Beta. |

---

## 4. Kiến trúc Hệ thống & Tech Stack

```
                               ┌────────────────────────────────────────────────────────┐
                               │                    PRESENTATION LAYER                  │
                               │   Jetpack Compose · Material 3 · Navigation · ViewModels│
                               └───────────────────────────┬────────────────────────────┘
                                                           │ StateFlow / UiState
                               ┌───────────────────────────▼────────────────────────────┐
                               │                       DOMAIN LAYER                     │
                               │             Use Cases · Domain Models · Business Rules │
                               └───────────────────────────┬────────────────────────────┘
                                                           │
                               ┌───────────────────────────▼────────────────────────────┐
                               │                        DATA LAYER                      │
                               │        Repositories · Data Mappers · Sync Engine       │
                               └───────────────┬────────────────────────┬───────────────┘
                                               │                        │
                       ┌───────────────────────▼──────┐        ┌────────▼──────────────────────┐
                       │          LOCAL DATA          │        │          REMOTE DATA          │
                       │ Room DB (Single Source)      │        │ Cloud Firestore               │
                       │ Encrypted DataStore          │        │ Cloud Storage                 │
                       │ Client Image Compressor      │        │ Firebase Vertex AI (Gemini)   │
                       └──────────────────────────────┘        └───────────────────────────────┘
```

### Danh mục công nghệ & thư viện
- **Core:** Kotlin 100%, Coroutines, Flow.
- **UI:** Jetpack Compose, Material 3, Navigation Compose, Coil (Image Loading).
- **Dependency Injection:** Hilt (Dagger).
- **Database Local:** Room (với Flow support), DataStore.
- **Background Task:** WorkManager (Sync Engine).
- **Camera:** CameraX.
- **Network & AI:** Retrofit, OkHttp, Firebase SDKs (Auth, Firestore, Storage, Vertex AI, App Check), `text-embedding-004`.

---

## 5. Cấu trúc Thư mục Dự án Android (Package-by-Feature)

```
app/src/main/kotlin/com/example/trainingdiary/
├── app/
│   ├── TrainingDiaryApp.kt
│   └── navigation/
├── core/
│   ├── designsystem/     // Colors, Typography, Shapes, Common Components
│   ├── database/         // Room Entities, DAOs, AppDatabase
│   ├── network/          // Retrofit instances, Firebase clients, AI Services
│   ├── common/           // Result wrappers, Extensions, Utils (ImageCompressor)
│   └── sync/             // SyncWorker, Outbox Queue Manager
├── data/
│   ├── repository/       // Implementations of repositories
│   └── mapper/           // DB <-> Domain <-> Remote mappers
├── domain/
│   ├── model/            // Pure Kotlin data classes (Workout, Meal, UserProfile)
│   ├── repository/       // Repository Interfaces
│   └── usecase/          // Business logic (CalculateTDEEUseCase, SyncDataUseCase)
└── feature/
    ├── auth/             // Login, Register, Profile Setup
    ├── dashboard/        // Today Summary, Daily Progress
    ├── journal/          // Timeline View, Photos
    ├── nutrition/        // Food Catalog, Meal Logger, Custom Food Form
    ├── workout/          // Exercise List, Active Workout Tracker, Sets/Reps Logger
    ├── aicoach/          // RAG Chatbot UI, Message Stream, Gemini Vision Dialog
    └── progress/         // Weight & Strength Progress Charts
```

---

## 6. Mô hình Dữ liệu Firestore & Security Rules
### Structure:
- `users/{uid}`: Thông tin hồ sơ, chỉ số TDEE/BMR, mục tiêu.
- `users/{uid}/journalEntries/{entryId}`: Nhật ký hàng ngày.
- `users/{uid}/meals/{mealId}`: Ghi nhận bữa ăn.
- `users/{uid}/workoutSessions/{sessionId}`: Chi tiết buổi tập.
- `users/{uid}/coachConversations/{conversationId}/messages/{messageId}`: Lịch sử chat AI.
- `foodCatalog/{foodId}`: Thư viện thực phẩm dùng chung (Read-only for clients).
- `exercises/{exerciseId}`: Thư viện bài tập dùng chung (Read-only for clients).

### Security & Quota Rules:
- Mọi tài nguyên dưới `users/{uid}` yêu cầu `request.auth.uid == uid`.
- Không lưu ảnh Base64 trong Firestore (bắt buộc dùng Cloud Storage).
- Áp dụng Firebase App Check bảo vệ Cloud Functions / Vertex AI Gateway.

---

## 7. Lộ trình Thực hiện 16 Tuần Chi tiết

```
[Tuần 1-2]  Base Setup (Hilt, Room, Compose Theme, Navigation)
     │
[Tuần 3-5]  Module Dinh dưỡng (Food DB Việt Nam + Custom Food + Calculator)
     │
[Tuần 6-8]  Module Tập luyện (Thư viện bài tập + Logger Sets/Reps + Volume)
     │
[Tuần 9-11] Sync Engine (Room <-> Firestore with WorkManager) & Auth/Profile
     │
[Tuần 12-14] Hybrid RAG AI Coach (Gemini 1.5 Flash + Firestore Vector Search + Vision)
     │
[Tuần 15-16] Health Connect, Analytics Charts, Performance & Beta Release
```
