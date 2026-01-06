---
name: flutter-architecture
description: Build production Flutter apps with Clean Architecture, feature-driven development, and dependency injection. Use when architecting new Flutter projects, refactoring existing apps, or implementing scalable patterns.
---

# Flutter Architecture Patterns

Production-ready architecture patterns for Flutter applications with Clean Architecture, feature-driven development, and robust dependency injection.

## When to Use This Skill

- Starting a new Flutter project with scalable architecture
- Refactoring a monolithic Flutter app into modular structure
- Implementing Clean Architecture with proper layer separation
- Setting up dependency injection with GetIt or Riverpod
- Designing repository patterns for data abstraction
- Building maintainable, testable Flutter applications

## Core Concepts

### 1. Project Structure (Feature-First)

```
lib/
├── main.dart                    # App entry point
├── app/
│   ├── app.dart                 # MaterialApp configuration
│   ├── router/
│   │   ├── app_router.dart      # GoRouter setup
│   │   └── route_guards.dart    # Auth guards
│   └── di/
│       └── injection.dart       # Dependency injection setup
├── core/
│   ├── constants/
│   │   └── api_constants.dart
│   ├── errors/
│   │   ├── exceptions.dart      # Custom exceptions
│   │   └── failures.dart        # Failure classes
│   ├── network/
│   │   ├── api_client.dart      # Dio client setup
│   │   └── interceptors/
│   ├── utils/
│   │   └── extensions.dart
│   └── usecases/
│       └── usecase.dart         # Base usecase class
├── shared/
│   ├── widgets/                 # Reusable widgets
│   ├── theme/                   # App theming
│   └── providers/               # Shared providers
└── features/
    └── [feature_name]/
        ├── data/
        │   ├── datasources/
        │   │   ├── feature_local_datasource.dart
        │   │   └── feature_remote_datasource.dart
        │   ├── models/
        │   │   └── feature_model.dart
        │   └── repositories/
        │       └── feature_repository_impl.dart
        ├── domain/
        │   ├── entities/
        │   │   └── feature_entity.dart
        │   ├── repositories/
        │   │   └── feature_repository.dart
        │   └── usecases/
        │       ├── get_feature.dart
        │       └── create_feature.dart
        └── presentation/
            ├── providers/
            │   └── feature_provider.dart
            ├── pages/
            │   └── feature_page.dart
            └── widgets/
                └── feature_widget.dart
```

### 2. Clean Architecture Layers

| Layer | Purpose | Dependencies |
|-------|---------|--------------|
| **Domain** | Business logic, entities, usecases | None (pure Dart) |
| **Data** | API calls, local storage, models | Domain layer |
| **Presentation** | UI, state management, widgets | Domain layer |

```
┌─────────────────────────────────────┐
│         Presentation Layer          │
│  (Widgets, Pages, State Management) │
└──────────────────┬──────────────────┘
                   │ depends on
┌──────────────────▼──────────────────┐
│           Domain Layer              │
│   (Entities, UseCases, Contracts)   │
└──────────────────▲──────────────────┘
                   │ implements
┌──────────────────┴──────────────────┐
│            Data Layer               │
│ (Models, DataSources, Repositories) │
└─────────────────────────────────────┘
```

## Quick Start

```bash
# Create new Flutter project
flutter create --org com.example my_app
cd my_app

# Add essential dependencies
flutter pub add dartz go_router dio get_it injectable
flutter pub add flutter_riverpod riverpod_annotation
flutter pub add dev:build_runner dev:injectable_generator dev:riverpod_generator
```

```dart
// lib/main.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'app/di/injection.dart';
import 'app/app.dart';

Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await configureDependencies();

  runApp(
    const ProviderScope(
      child: MyApp(),
    ),
  );
}
```

## Patterns

### Pattern 1: Domain Layer - Entities & UseCases

```dart
// domain/entities/user_entity.dart
import 'package:equatable/equatable.dart';

class UserEntity extends Equatable {
  final String id;
  final String email;
  final String name;
  final DateTime? createdAt;

  const UserEntity({
    required this.id,
    required this.email,
    required this.name,
    this.createdAt,
  });

  @override
  List<Object?> get props => [id, email, name, createdAt];
}
```

```dart
// core/usecases/usecase.dart
import 'package:dartz/dartz.dart';
import '../errors/failures.dart';

typedef ResultFuture<T> = Future<Either<Failure, T>>;
typedef ResultVoid = ResultFuture<void>;

abstract class UseCase<Type, Params> {
  ResultFuture<Type> call(Params params);
}

abstract class UseCaseNoParams<Type> {
  ResultFuture<Type> call();
}

class NoParams {}
```

```dart
// domain/usecases/get_user.dart
import 'package:dartz/dartz.dart';
import 'package:injectable/injectable.dart';
import '../../../core/errors/failures.dart';
import '../../../core/usecases/usecase.dart';
import '../entities/user_entity.dart';
import '../repositories/user_repository.dart';

@injectable
class GetUser implements UseCase<UserEntity, String> {
  final UserRepository _repository;

  GetUser(this._repository);

  @override
  ResultFuture<UserEntity> call(String userId) async {
    return await _repository.getUser(userId);
  }
}
```

```dart
// domain/repositories/user_repository.dart
import 'package:dartz/dartz.dart';
import '../../../core/errors/failures.dart';
import '../entities/user_entity.dart';

abstract class UserRepository {
  ResultFuture<UserEntity> getUser(String id);
  ResultFuture<List<UserEntity>> getUsers();
  ResultVoid createUser(UserEntity user);
  ResultVoid updateUser(UserEntity user);
  ResultVoid deleteUser(String id);
}
```

### Pattern 2: Data Layer - Models & DataSources

```dart
// data/models/user_model.dart
import '../../domain/entities/user_entity.dart';

class UserModel extends UserEntity {
  const UserModel({
    required super.id,
    required super.email,
    required super.name,
    super.createdAt,
  });

  factory UserModel.fromJson(Map<String, dynamic> json) {
    return UserModel(
      id: json['id'] as String,
      email: json['email'] as String,
      name: json['name'] as String,
      createdAt: json['created_at'] != null
          ? DateTime.parse(json['created_at'] as String)
          : null,
    );
  }

  Map<String, dynamic> toJson() {
    return {
      'id': id,
      'email': email,
      'name': name,
      'created_at': createdAt?.toIso8601String(),
    };
  }

  factory UserModel.fromEntity(UserEntity entity) {
    return UserModel(
      id: entity.id,
      email: entity.email,
      name: entity.name,
      createdAt: entity.createdAt,
    );
  }
}
```

```dart
// data/datasources/user_remote_datasource.dart
import 'package:dio/dio.dart';
import 'package:injectable/injectable.dart';
import '../../../core/errors/exceptions.dart';
import '../models/user_model.dart';

abstract class UserRemoteDataSource {
  Future<UserModel> getUser(String id);
  Future<List<UserModel>> getUsers();
  Future<void> createUser(Map<String, dynamic> data);
}

@Injectable(as: UserRemoteDataSource)
class UserRemoteDataSourceImpl implements UserRemoteDataSource {
  final Dio _dio;

  UserRemoteDataSourceImpl(this._dio);

  @override
  Future<UserModel> getUser(String id) async {
    try {
      final response = await _dio.get('/users/$id');
      return UserModel.fromJson(response.data);
    } on DioException catch (e) {
      throw ServerException(
        message: e.message ?? 'Server error',
        statusCode: e.response?.statusCode ?? 500,
      );
    }
  }

  @override
  Future<List<UserModel>> getUsers() async {
    try {
      final response = await _dio.get('/users');
      return (response.data as List)
          .map((json) => UserModel.fromJson(json))
          .toList();
    } on DioException catch (e) {
      throw ServerException(
        message: e.message ?? 'Server error',
        statusCode: e.response?.statusCode ?? 500,
      );
    }
  }

  @override
  Future<void> createUser(Map<String, dynamic> data) async {
    try {
      await _dio.post('/users', data: data);
    } on DioException catch (e) {
      throw ServerException(
        message: e.message ?? 'Server error',
        statusCode: e.response?.statusCode ?? 500,
      );
    }
  }
}
```

```dart
// data/repositories/user_repository_impl.dart
import 'package:dartz/dartz.dart';
import 'package:injectable/injectable.dart';
import '../../../core/errors/exceptions.dart';
import '../../../core/errors/failures.dart';
import '../../domain/entities/user_entity.dart';
import '../../domain/repositories/user_repository.dart';
import '../datasources/user_remote_datasource.dart';
import '../models/user_model.dart';

@Injectable(as: UserRepository)
class UserRepositoryImpl implements UserRepository {
  final UserRemoteDataSource _remoteDataSource;

  UserRepositoryImpl(this._remoteDataSource);

  @override
  ResultFuture<UserEntity> getUser(String id) async {
    try {
      final result = await _remoteDataSource.getUser(id);
      return Right(result);
    } on ServerException catch (e) {
      return Left(ServerFailure(message: e.message, statusCode: e.statusCode));
    }
  }

  @override
  ResultFuture<List<UserEntity>> getUsers() async {
    try {
      final result = await _remoteDataSource.getUsers();
      return Right(result);
    } on ServerException catch (e) {
      return Left(ServerFailure(message: e.message, statusCode: e.statusCode));
    }
  }

  @override
  ResultVoid createUser(UserEntity user) async {
    try {
      await _remoteDataSource.createUser(UserModel.fromEntity(user).toJson());
      return const Right(null);
    } on ServerException catch (e) {
      return Left(ServerFailure(message: e.message, statusCode: e.statusCode));
    }
  }

  @override
  ResultVoid updateUser(UserEntity user) async {
    // Implementation...
    return const Right(null);
  }

  @override
  ResultVoid deleteUser(String id) async {
    // Implementation...
    return const Right(null);
  }
}
```

### Pattern 3: Dependency Injection with GetIt + Injectable

```dart
// app/di/injection.dart
import 'package:get_it/get_it.dart';
import 'package:injectable/injectable.dart';
import 'injection.config.dart';

final getIt = GetIt.instance;

@InjectableInit(preferRelativeImports: true)
Future<void> configureDependencies() async => getIt.init();
```

```dart
// app/di/register_module.dart
import 'package:dio/dio.dart';
import 'package:injectable/injectable.dart';
import 'package:shared_preferences/shared_preferences.dart';

@module
abstract class RegisterModule {
  @preResolve
  Future<SharedPreferences> get prefs => SharedPreferences.getInstance();

  @lazySingleton
  Dio dio() {
    return Dio(BaseOptions(
      baseUrl: 'https://api.example.com',
      connectTimeout: const Duration(seconds: 10),
      receiveTimeout: const Duration(seconds: 10),
      headers: {'Content-Type': 'application/json'},
    ))
      ..interceptors.add(LogInterceptor(
        requestBody: true,
        responseBody: true,
      ));
  }
}
```

```bash
# Generate injection code
dart run build_runner build --delete-conflicting-outputs
```

### Pattern 4: Error Handling

```dart
// core/errors/exceptions.dart
class ServerException implements Exception {
  final String message;
  final int statusCode;

  const ServerException({
    required this.message,
    required this.statusCode,
  });
}

class CacheException implements Exception {
  final String message;

  const CacheException({required this.message});
}

class NetworkException implements Exception {
  final String message;

  const NetworkException({required this.message});
}
```

```dart
// core/errors/failures.dart
import 'package:equatable/equatable.dart';

abstract class Failure extends Equatable {
  final String message;
  final int? statusCode;

  const Failure({
    required this.message,
    this.statusCode,
  });

  @override
  List<Object?> get props => [message, statusCode];
}

class ServerFailure extends Failure {
  const ServerFailure({
    required super.message,
    super.statusCode,
  });
}

class CacheFailure extends Failure {
  const CacheFailure({required super.message});
}

class NetworkFailure extends Failure {
  const NetworkFailure({required super.message});
}
```

### Pattern 5: Router with GoRouter

```dart
// app/router/app_router.dart
import 'package:flutter/material.dart';
import 'package:go_router/go_router.dart';
import 'package:riverpod_annotation/riverpod_annotation.dart';
import '../../features/auth/presentation/providers/auth_provider.dart';
import '../../features/auth/presentation/pages/login_page.dart';
import '../../features/home/presentation/pages/home_page.dart';
import '../../shared/widgets/main_layout.dart';

part 'app_router.g.dart';

@Riverpod(keepAlive: true)
GoRouter appRouter(Ref ref) {
  final authState = ref.watch(authProvider);

  return GoRouter(
    initialLocation: '/',
    debugLogDiagnostics: true,
    redirect: (context, state) {
      final isAuthenticated = authState.isAuthenticated;
      final isLoginRoute = state.matchedLocation == '/login';

      if (!isAuthenticated && !isLoginRoute) {
        return '/login';
      }
      if (isAuthenticated && isLoginRoute) {
        return '/';
      }
      return null;
    },
    routes: [
      GoRoute(
        path: '/login',
        builder: (context, state) => const LoginPage(),
      ),
      ShellRoute(
        builder: (context, state, child) => MainLayout(child: child),
        routes: [
          GoRoute(
            path: '/',
            builder: (context, state) => const HomePage(),
            routes: [
              GoRoute(
                path: 'users/:id',
                builder: (context, state) {
                  final id = state.pathParameters['id']!;
                  return UserDetailPage(userId: id);
                },
              ),
            ],
          ),
        ],
      ),
    ],
  );
}
```

### Pattern 6: Feature Module Template

```dart
// features/[feature]/domain/entities/[feature]_entity.dart
// features/[feature]/domain/repositories/[feature]_repository.dart
// features/[feature]/domain/usecases/get_[feature].dart

// features/[feature]/data/models/[feature]_model.dart
// features/[feature]/data/datasources/[feature]_remote_datasource.dart
// features/[feature]/data/repositories/[feature]_repository_impl.dart

// features/[feature]/presentation/providers/[feature]_provider.dart
// features/[feature]/presentation/pages/[feature]_page.dart
// features/[feature]/presentation/widgets/[feature]_widget.dart
```

## Best Practices

### Do's
- **Separate layers strictly** - Domain layer should have zero external dependencies
- **Use abstractions** - Repository interfaces in domain, implementations in data
- **Inject dependencies** - Use GetIt/Injectable for loose coupling
- **Handle errors consistently** - Either type for all repository methods
- **Keep entities immutable** - Use Equatable and const constructors
- **Test each layer independently** - Mock dependencies at boundaries

### Don'ts
- **Don't import data layer in domain** - Violates dependency rule
- **Don't put business logic in widgets** - Use usecases instead
- **Don't skip repository pattern** - Even for simple data access
- **Don't hardcode dependencies** - Always use injection
- **Don't mix UI state with domain state** - Keep them separate
- **Don't ignore error cases** - Handle Left side of Either

## Resources

- [Clean Architecture by Uncle Bob](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- [Flutter Clean Architecture - ResoCoder](https://resocoder.com/flutter-clean-architecture-tdd/)
- [Injectable Package](https://pub.dev/packages/injectable)
- [GoRouter Documentation](https://pub.dev/packages/go_router)
- [Dartz Package](https://pub.dev/packages/dartz)
