# Architecture Documentation

Dokumen ini menjelaskan arsitektur aplikasi Lokapandu yang dibangun menggunakan Flutter dengan prinsip Clean Architecture.

## 📋 Daftar Isi

- [Overview](#overview)
- [Clean Architecture](#clean-architecture)
- [Layer Structure](#layer-structure)
- [Data Flow](#data-flow)
- [Dependency Injection](#dependency-injection)
- [State Management](#state-management)
- [Error Handling](#error-handling)
- [Code Generation](#code-generation)
- [External Services](#external-services)

## 🏗️ Overview

Lokapandu menggunakan **Clean Architecture** yang dikombinasikan dengan **Provider** untuk state management. Arsitektur ini memisahkan concerns ke dalam layer yang berbeda, membuat kode lebih maintainable, testable, dan scalable.

### Prinsip Utama

1. **Separation of Concerns** - Setiap layer memiliki tanggung jawab yang jelas
2. **Dependency Inversion** - Layer dalam tidak bergantung pada layer luar
3. **Testability** - Setiap komponen dapat ditest secara independen
4. **Maintainability** - Mudah untuk modify dan extend

## 🎯 Clean Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Presentation Layer                       │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │    Pages    │  │   Widgets   │  │      Providers      │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      Domain Layer                           │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │  Entities   │  │  Use Cases  │  │    Repositories     │  │
│  │             │  │             │  │   (Interfaces)      │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                       Data Layer                            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │   Models    │  │ Repositories│  │   Data Sources      │  │
│  │             │  │(Implements) │  │                     │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                   External Services                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │  Supabase   │  │  Firebase   │  │    Google Maps      │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

## 📁 Layer Structure

### 1. Presentation Layer (`lib/presentation/`)

Layer ini bertanggung jawab untuk UI dan interaksi user.

```
presentation/
├── pages/          # Screen/halaman aplikasi
├── widgets/        # Reusable UI components
└── provider/       # State management dengan Provider
```

#### Pages
```dart
class HomePage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('Lokapandu')),
      body: Consumer<TourismSpotNotifier>(
        builder: (context, notifier, child) {
          if (notifier.isLoading) {
            return CircularProgressIndicator();
          }
          return TourismSpotList(spots: notifier.tourismSpots);
        },
      ),
    );
  }
}
```

#### Providers
```dart
class TourismSpotNotifier extends ChangeNotifier {
  final GetTourismSpotList _getTourismSpotList;
  
  List<TourismSpot> _tourismSpots = [];
  bool _isLoading = false;
  
  // Getters
  List<TourismSpot> get tourismSpots => _tourismSpots;
  bool get isLoading => _isLoading;
  
  // Methods
  Future<void> loadTourismSpots() async {
    _isLoading = true;
    notifyListeners();
    
    final result = await _getTourismSpotList(NoParams());
    result.fold(
      (failure) => _handleError(failure),
      (spots) => _tourismSpots = spots,
    );
    
    _isLoading = false;
    notifyListeners();
  }
}
```

### 2. Domain Layer (`lib/domain/`)

Layer ini berisi business logic dan rules aplikasi.

```
domain/
├── entities/       # Business objects
├── repositories/   # Repository interfaces
└── usecases/       # Business logic operations
```

#### Entities
```dart
@freezed
class TourismSpot with _$TourismSpot {
  const factory TourismSpot({
    required String id,
    required String name,
    required String description,
    required String city,
    required String province,
    required String address,
    required double latitude,
    required double longitude,
    required String openTime,
    required String closeTime,
    required String googleMapsLink,
    required List<TourismImage> images,
    required List<Facilities> facilities,
  }) = _TourismSpot;
}
```

#### Use Cases
```dart
class GetTourismSpotList implements UseCase<List<TourismSpot>, NoParams> {
  final TourismSpotRepository repository;
  
  GetTourismSpotList(this.repository);
  
  @override
  Future<Either<Failure, List<TourismSpot>>> call(NoParams params) async {
    return await repository.getTourismSpots();
  }
}
```

#### Repository Interfaces
```dart
abstract class TourismSpotRepository {
  Future<Either<Failure, List<TourismSpot>>> getTourismSpots();
  Future<Either<Failure, TourismSpot>> getTourismSpotById(String id);
}
```

### 3. Data Layer (`lib/data/`)

Layer ini menangani data dari berbagai sumber.

```
data/
├── datasources/    # Remote & local data sources
├── models/         # Data transfer objects
└── repositories/   # Repository implementations
```

#### Models
```dart
@freezed
class TourismSpotModel with _$TourismSpotModel {
  const factory TourismSpotModel({
    required String id,
    required String name,
    required String description,
    required String city,
    required String province,
    required String address,
    required double latitude,
    required double longitude,
    required String openTime,
    required String closeTime,
    required String googleMapsLink,
    required List<TourismImageModel> images,
    required List<FacilitiesModel> facilities,
  }) = _TourismSpotModel;

  factory TourismSpotModel.fromJson(Map<String, dynamic> json) =>
      _$TourismSpotModelFromJson(json);
}

extension TourismSpotModelX on TourismSpotModel {
  TourismSpot toEntity() => TourismSpot(
    id: id,
    name: name,
    description: description,
    // ... other fields
  );
}
```

#### Data Sources
```dart
abstract class TourismSpotRemoteDataSource {
  Future<List<TourismSpotModel>> getTourismSpots();
}

class TourismSpotRemoteDataSourceImpl implements TourismSpotRemoteDataSource {
  final SupabaseService supabaseService;
  
  TourismSpotRemoteDataSourceImpl({required this.supabaseService});
  
  @override
  Future<List<TourismSpotModel>> getTourismSpots() async {
    try {
      final response = await supabaseService.client
          .from('tourism_spots')
          .select('*, images(*), facilities(*)');
      
      return response.map((json) => TourismSpotModel.fromJson(json)).toList();
    } catch (e) {
      throw ServerException();
    }
  }
}
```

#### Repository Implementation
```dart
class TourismSpotRepositoryImpl implements TourismSpotRepository {
  final TourismSpotRemoteDataSource remoteDataSource;
  
  TourismSpotRepositoryImpl({required this.remoteDataSource});
  
  @override
  Future<Either<Failure, List<TourismSpot>>> getTourismSpots() async {
    try {
      final result = await remoteDataSource.getTourismSpots();
      return Right(result.map((model) => model.toEntity()).toList());
    } on ServerException {
      return Left(ServerFailure('Server error occurred'));
    } on SocketException {
      return Left(ConnectionFailure('No internet connection'));
    }
  }
}
```

## 🔄 Data Flow

```
User Interaction
       │
       ▼
   UI Widget
       │
       ▼
   Provider (State Management)
       │
       ▼
   Use Case (Business Logic)
       │
       ▼
Repository Interface (Domain)
       │
       ▼
Repository Implementation (Data)
       │
       ▼
   Data Source
       │
       ▼
External Service (Supabase/Firebase)
```

### Contoh Flow: Mengambil Daftar Tempat Wisata

1. **User** membuka halaman home
2. **HomePage** memanggil `TourismSpotNotifier.loadTourismSpots()`
3. **TourismSpotNotifier** memanggil `GetTourismSpotList` use case
4. **GetTourismSpotList** memanggil `TourismSpotRepository.getTourismSpots()`
5. **TourismSpotRepositoryImpl** memanggil `TourismSpotRemoteDataSource.getTourismSpots()`
6. **TourismSpotRemoteDataSourceImpl** melakukan request ke Supabase
7. **Response** dikembalikan melalui chain yang sama dengan transformasi data

## 🔧 Dependency Injection

Menggunakan **GetIt** untuk dependency injection:

```dart
// injection.dart
final locator = GetIt.instance;

void init() {
  // Providers
  locator.registerFactory(() => TourismSpotNotifier(locator()));
  
  // Use Cases
  locator.registerLazySingleton(() => GetTourismSpotList(locator()));
  
  // Repositories
  locator.registerLazySingleton<TourismSpotRepository>(
    () => TourismSpotRepositoryImpl(remoteDataSource: locator()),
  );
  
  // Data Sources
  locator.registerLazySingleton<TourismSpotRemoteDataSource>(
    () => TourismSpotRemoteDataSourceImpl(supabaseService: locator()),
  );
  
  // Services
  locator.registerLazySingleton<SupabaseService>(() => SupabaseService());
}
```

### Dependency Graph

```
TourismSpotNotifier
       │
       ▼
GetTourismSpotList
       │
       ▼
TourismSpotRepository
       │
       ▼
TourismSpotRemoteDataSource
       │
       ▼
SupabaseService
```

## 📊 State Management

Menggunakan **Provider** pattern untuk state management:

### Provider Setup
```dart
// main.dart
runApp(
  MultiProvider(
    providers: [
      ChangeNotifierProvider(create: (_) => locator<TourismSpotNotifier>()),
      // ... other providers
    ],
    child: const App(),
  ),
);
```

### State Access
```dart
// Dalam widget
Consumer<TourismSpotNotifier>(
  builder: (context, notifier, child) {
    return ListView.builder(
      itemCount: notifier.tourismSpots.length,
      itemBuilder: (context, index) {
        return TourismSpotCard(spot: notifier.tourismSpots[index]);
      },
    );
  },
)

// Atau menggunakan context
final notifier = Provider.of<TourismSpotNotifier>(context);
```

## ❌ Error Handling

### Failure Classes
```dart
@freezed
abstract class Failure with _$Failure {
  const factory Failure.server(String message) = ServerFailure;
  const factory Failure.connection(String message) = ConnectionFailure;
  const factory Failure.cache(String message) = CacheFailure;
}
```

### Error Handling Pattern
```dart
final result = await useCase(params);
result.fold(
  (failure) => failure.when(
    server: (message) => _showError('Server error: $message'),
    connection: (message) => _showError('Connection error: $message'),
    cache: (message) => _showError('Cache error: $message'),
  ),
  (data) => _handleSuccess(data),
);
```

## 🔄 Code Generation

Menggunakan beberapa package untuk code generation:

### Freezed (Data Classes)
```dart
@freezed
class TourismSpot with _$TourismSpot {
  const factory TourismSpot({
    required String id,
    required String name,
  }) = _TourismSpot;
}
```

### JSON Serialization
```dart
@JsonSerializable()
class TourismSpotModel {
  final String id;
  final String name;
  
  factory TourismSpotModel.fromJson(Map<String, dynamic> json) =>
      _$TourismSpotModelFromJson(json);
}
```

### Environment Variables
```dart
@Envied(path: '.env')
abstract class Env {
  @EnviedField(varName: 'SUPABASE_URL')
  static const String supabaseUrl = _Env.supabaseUrl;
}
```

### Generate Command
```bash
flutter packages pub run build_runner build --delete-conflicting-outputs
```

## 🌐 External Services

### Supabase Integration
```dart
class SupabaseService {
  late final SupabaseClient client;
  
  SupabaseService() {
    client = Supabase.instance.client;
  }
  
  Future<List<Map<String, dynamic>>> getTourismSpots() async {
    final response = await client
        .from('tourism_spots')
        .select('*, images(*), facilities(*)')
        .order('created_at', ascending: false);
    
    return response;
  }
}
```

### Firebase AI Integration
```dart
class FirebaseAIService {
  Future<List<String>> getRecommendations(String userPreferences) async {
    // Implementation for AI recommendations
  }
}
```

## 🧪 Testing Strategy

### Unit Tests
- Test use cases dengan mock repositories
- Test repositories dengan mock data sources
- Test models dan entities

### Widget Tests
- Test UI components
- Test provider interactions
- Test navigation

### Integration Tests
- Test complete user flows
- Test external service integrations

### Test Structure
```
test/
├── unit/
│   ├── data/
│   │   ├── models/
│   │   ├── repositories/
│   │   └── datasources/
│   ├── domain/
│   │   ├── entities/
│   │   └── usecases/
│   └── presentation/
│       └── provider/
├── widget/
└── integration/
```

## 📈 Performance Considerations

1. **Lazy Loading** - Load data hanya saat dibutuhkan
2. **Caching** - Cache data untuk mengurangi network calls
3. **Image Optimization** - Compress dan cache images
4. **State Management** - Minimize rebuilds dengan proper provider usage
5. **Code Splitting** - Lazy load features yang tidak immediately needed

## 🔮 Future Enhancements

1. **Offline Support** - Local database dengan sync
2. **Push Notifications** - Firebase messaging
3. **Analytics** - Firebase analytics integration
4. **Crash Reporting** - Firebase crashlytics
5. **Performance Monitoring** - Firebase performance

---

Arsitektur ini dirancang untuk scalability dan maintainability jangka panjang. Setiap layer memiliki tanggung jawab yang jelas dan dapat ditest secara independen.