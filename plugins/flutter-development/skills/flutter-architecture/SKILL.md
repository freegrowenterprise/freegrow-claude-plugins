---
name: flutter-clean-architecture
description: Flutter 클린 아키텍처 + Riverpod 3.0 기반 프로젝트 구조. Flutter 프로젝트 생성, 기능 구현, 리팩토링 시 사용.
---

# Flutter Clean Architecture (Riverpod 3.0)

Riverpod 3.0+, go_router, http 패키지 기반의 프로덕션 레디 Flutter 아키텍처 패턴.

## 기술 스택

| 패키지 | 버전 | 용도 |
|--------|------|------|
| flutter_riverpod | ^3.0.3 | 상태 관리 |
| riverpod_annotation | ^3.0.3 | 코드 생성 |
| hooks_riverpod | ^3.0.3 | Hook 지원 |
| go_router | latest | 라우팅 |
| http | latest | HTTP 클라이언트 |
| dartz | latest | Either 타입 |
| equatable | latest | 값 비교 |
| sentry_flutter | latest | 에러 모니터링 |
| shared_preferences | latest | 로컬 저장소 |

### Dev Dependencies
- build_runner: 코드 생성
- riverpod_generator: Riverpod 코드 생성

## 프로젝트 구조

```
lib/
├── main.dart                    # 앱 진입점
├── constant/
│   └── constant.dart            # 상수 정의 (API URL, Project ID 등)
├── core/                        # 공통 핵심 기능
│   ├── router/
│   │   ├── app_router.dart      # GoRouter 설정 + 인증 redirect
│   │   └── app_router.g.dart
│   ├── network/
│   │   ├── typedef.dart         # ResultFuture, ResultVoid 등 타입 정의
│   │   ├── usecase.dart         # UseCase 추상 클래스
│   │   ├── failure.dart         # Failure 클래스
│   │   └── exceptions.dart      # APIException 클래스
│   ├── utils/
│   │   ├── toast_utils.dart     # Toast 메시지 유틸
│   │   └── image_picker_utils.dart
│   └── error/
│       └── sentry_helper.dart   # Sentry 에러 리포팅
├── common/                      # 공통 UI 컴포넌트
│   ├── layouts/
│   │   └── main_layout.dart     # ShellRoute용 레이아웃
│   ├── widgets/
│   │   └── [공통_위젯].dart
│   └── providers/
│       └── [공통_provider].dart
└── src/                         # 기능별 모듈 (Feature Module)
    └── [feature_name]/
        ├── data/
        │   ├── datasources/
        │   │   └── xxx_remote_datasource.dart
        │   ├── models/
        │   │   └── xxx_model.dart
        │   └── repositories/
        │       └── xxx_repository_impl.dart
        ├── domain/
        │   ├── entities/
        │   │   └── xxx_entity.dart
        │   ├── repositories/
        │   │   └── xxx_repository.dart
        │   ├── usecases/
        │   │   └── get_xxx_usecase.dart
        │   └── request/
        │       └── create_xxx_params.dart
        └── presentation/
            ├── provider/
            │   ├── xxx_list_provider.dart
            │   └── xxx_list_provider.g.dart
            ├── screens/
            │   └── xxx_page.dart
            └── widgets/
                └── xxx_widget.dart
```

## 클린 아키텍처 레이어

| 레이어 | 역할 | 의존성 |
|--------|------|--------|
| **Domain** | 비즈니스 로직, Entity, UseCase | 외부 의존 없음 (순수 Dart) |
| **Data** | API 호출, 로컬 저장소, Model | Domain 레이어에 의존 |
| **Presentation** | UI, 상태 관리, Widget | Domain 레이어에 의존 |

```
┌─────────────────────────────────────┐
│         Presentation Layer          │
│  (Widgets, Pages, Providers)        │
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

## 코드 패턴

### 1. Core 타입 정의

```dart
// core/network/typedef.dart
import 'package:dartz/dartz.dart';
import 'failure.dart';

typedef ResultFuture<T> = Future<Either<Failure, T>>;
typedef ResultVoid = ResultFuture<void>;
typedef DataMap = Map<String, dynamic>;
```

```dart
// core/network/usecase.dart
import 'typedef.dart';

abstract class UseCaseWithParams<Type, Params> {
  const UseCaseWithParams();
  ResultFuture<Type> call(Params params);
}

abstract class UseCaseWithParamsTwo<Type, Params, Param2> {
  const UseCaseWithParamsTwo();
  ResultFuture<Type> call(Params params, Param2 param2);
}

abstract class UseCaseWithoutParams<Type> {
  const UseCaseWithoutParams();
  ResultFuture<Type> call();
}
```

### 2. Domain Layer - Entity

```dart
// domain/entities/xxx_entity.dart
class XxxEntity {
  final int id;
  final String title;
  final String content;
  final int projectId;
  final bool? isActive;

  const XxxEntity({
    required this.id,
    required this.title,
    required this.content,
    required this.projectId,
    this.isActive,
  });
}
```

### 3. Domain Layer - Repository Interface

```dart
// domain/repositories/xxx_repository.dart
import 'package:flutter/material.dart';
import 'package:your_app/core/network/typedef.dart';
import '../entities/xxx_entity.dart';

abstract class XxxRepository {
  ResultFuture<List<XxxEntity>> getList(BuildContext context, int projectId);
  ResultFuture<XxxEntity> getDetail(BuildContext context, int id);
  ResultVoid create(BuildContext context, CreateXxxParams params);
  ResultVoid update(BuildContext context, int id, UpdateXxxParams params);
  ResultVoid delete(BuildContext context, int id);
}
```

### 4. Domain Layer - UseCase

```dart
// domain/usecases/get_xxx_list_usecase.dart
import 'package:flutter/material.dart';
import 'package:your_app/core/network/typedef.dart';
import 'package:your_app/core/network/usecase.dart';
import '../entities/xxx_entity.dart';
import '../repositories/xxx_repository.dart';

class GetXxxListUseCase extends UseCaseWithParamsTwo<List<XxxEntity>, BuildContext, int> {
  final XxxRepository _repository;

  GetXxxListUseCase(this._repository);

  @override
  ResultFuture<List<XxxEntity>> call(BuildContext context, int projectId) async {
    return await _repository.getList(context, projectId);
  }
}
```

### 5. Data Layer - Model (Entity 상속)

```dart
// data/models/xxx_model.dart
import '../../domain/entities/xxx_entity.dart';

class XxxModel extends XxxEntity {
  const XxxModel({
    required super.id,
    required super.title,
    required super.content,
    required super.projectId,
    super.isActive,
  });

  factory XxxModel.fromJson(Map<String, dynamic> json) {
    return XxxModel(
      id: json['id'],
      title: json['title'] ?? '',
      content: json['content'] ?? '',
      projectId: json['projectId'],
      isActive: json['isActive'],
    );
  }

  Map<String, dynamic> toJson() {
    return {
      'id': id,
      'title': title,
      'content': content,
      'projectId': projectId,
      'isActive': isActive,
    };
  }
}
```

### 6. Data Layer - DataSource

```dart
// data/datasources/xxx_remote_datasource.dart
import 'dart:convert';
import 'package:flutter/foundation.dart';
import 'package:flutter/material.dart';
import 'package:http/http.dart' as http;
import 'package:your_app/constant/constant.dart';
import 'package:your_app/core/error/sentry_helper.dart';
import 'package:your_app/core/network/exceptions.dart';
import '../models/xxx_model.dart';

abstract class XxxDataSource {
  Future<List<XxxModel>> getList(BuildContext context, int projectId);
  Future<XxxModel> getDetail(BuildContext context, int id);
  Future<void> create(BuildContext context, Map<String, dynamic> data);
}

class XxxRemoteDataSourceImpl implements XxxDataSource {
  final http.Client _client;

  XxxRemoteDataSourceImpl(this._client);

  final String _baseEndpoint = '/api/xxx';

  @override
  Future<List<XxxModel>> getList(BuildContext context, int projectId) async {
    try {
      final uri = Uri.https(
        kDebugMode ? Constants.kBaseUrlTest : Constants.kBaseUrlProd,
        '$_baseEndpoint/by-project/$projectId',
      );
      final response = await _client.get(
        uri,
        headers: {
          'Content-Type': 'application/json',
          'Authorization': SharedPreferencesUtil.getString(SharedPreferencesKey.accessToken) ?? '',
        },
      );

      if (response.statusCode != 200) {
        throw APIException(
          message: '데이터를 불러오는데 실패했습니다.',
          statusCode: response.statusCode,
        );
      }

      // 중요: utf8 디코딩 필수
      final decodedResponse = jsonDecode(utf8.decode(response.bodyBytes));
      return (decodedResponse as List)
          .map((json) => XxxModel.fromJson(json))
          .toList();
    } on APIException {
      rethrow;
    } catch (e, st) {
      await SentryHelper.captureApiError(
        exception: e,
        stackTrace: st,
        endpoint: '$_baseEndpoint/by-project/$projectId',
        message: '목록 조회 실패',
      );
      throw APIException(message: e.toString(), statusCode: 505);
    }
  }

  // getDetail, create 구현...
}
```

### 7. Data Layer - Repository 구현체

```dart
// data/repositories/xxx_repository_impl.dart
import 'package:dartz/dartz.dart';
import 'package:flutter/material.dart';
import 'package:your_app/core/network/exceptions.dart';
import 'package:your_app/core/network/failure.dart';
import 'package:your_app/core/network/typedef.dart';
import '../datasources/xxx_remote_datasource.dart';
import '../../domain/entities/xxx_entity.dart';
import '../../domain/repositories/xxx_repository.dart';

class XxxRepositoryImpl implements XxxRepository {
  final XxxDataSource _remoteDataSource;

  XxxRepositoryImpl(this._remoteDataSource);

  @override
  ResultFuture<List<XxxEntity>> getList(BuildContext context, int projectId) async {
    try {
      final models = await _remoteDataSource.getList(context, projectId);
      return Right(models); // Model은 Entity를 상속하므로 그대로 반환
    } on APIException catch (e) {
      return Left(APIFailure.fromException(e));
    }
  }

  // 나머지 메서드 구현...
}
```

### 8. Presentation Layer - Provider (Riverpod 3.0+)

**중요: Riverpod 3.0 변경사항**
```dart
// ❌ 이전 방식 (Riverpod 2.x)
@riverpod
XxxDataSource xxxDataSource(XxxDataSourceRef ref) { ... }

// ✅ 새로운 방식 (Riverpod 3.0+)
@riverpod
XxxDataSource xxxDataSource(Ref ref) { ... }
```

```dart
// presentation/provider/xxx_list_provider.dart
import 'package:riverpod_annotation/riverpod_annotation.dart';

part 'xxx_list_provider.g.dart';

@riverpod
class XxxList extends _$XxxList {
  @override
  Future<List<XxxModel>> build(int projectId) async {
    return await _loadData(projectId);
  }

  Future<List<XxxModel>> _loadData(int projectId) async {
    try {
      final service = ref.read(xxxServiceProvider);
      return await service.getList(projectId);
    } catch (e) {
      throw Exception('데이터를 불러오는데 실패했습니다: $e');
    }
  }

  Future<void> refresh(int projectId) async {
    state = const AsyncValue.loading();
    state = await AsyncValue.guard(() => _loadData(projectId));
  }

  void addToList(XxxModel item) {
    final currentItems = state.value ?? [];
    state = AsyncValue.data([item, ...currentItems]);
  }

  void removeFromList(int id) {
    final currentItems = state.value ?? [];
    state = AsyncValue.data(
      currentItems.where((item) => item.id != id).toList(),
    );
  }
}
```

### 9. Presentation Layer - Screen (HookConsumerWidget)

```dart
// presentation/screens/xxx_page.dart
import 'package:flutter/material.dart';
import 'package:flutter_hooks/flutter_hooks.dart';
import 'package:hooks_riverpod/hooks_riverpod.dart';

class XxxPage extends HookConsumerWidget {
  final int projectId;

  const XxxPage({super.key, required this.projectId});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final dataAsync = ref.watch(xxxListProvider(projectId));
    final isLoading = useState<bool>(false);
    final searchController = useTextEditingController();

    return dataAsync.when(
      loading: () => const Center(child: CircularProgressIndicator()),
      error: (error, stack) => Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Icon(Icons.error_outline, size: 64, color: Colors.red[400]),
            const SizedBox(height: 16),
            Text('오류: $error'),
            const SizedBox(height: 16),
            ElevatedButton(
              onPressed: () => ref.refresh(xxxListProvider(projectId)),
              child: const Text('다시 시도'),
            ),
          ],
        ),
      ),
      data: (items) {
        return SingleChildScrollView(
          padding: const EdgeInsets.all(24.0),
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              // 헤더
              Row(
                mainAxisAlignment: MainAxisAlignment.spaceBetween,
                children: [
                  Text(
                    '제목',
                    style: Theme.of(context).textTheme.headlineMedium?.copyWith(
                      fontWeight: FontWeight.bold,
                    ),
                  ),
                  ElevatedButton.icon(
                    onPressed: () => _showCreateDialog(context, ref),
                    icon: const Icon(Icons.add),
                    label: const Text('추가'),
                  ),
                ],
              ),
              const SizedBox(height: 24),
              // 리스트
              ListView.builder(
                shrinkWrap: true,
                physics: const NeverScrollableScrollPhysics(),
                itemCount: items.length,
                itemBuilder: (context, index) {
                  final item = items[index];
                  return _buildItemCard(context, ref, item);
                },
              ),
            ],
          ),
        );
      },
    );
  }
}
```

### 10. Dialog 패턴 (HookConsumerWidget)

```dart
class XxxDialog extends HookConsumerWidget {
  final XxxModel? existingItem;

  const XxxDialog({super.key, this.existingItem});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final formKey = useMemoized(() => GlobalKey<FormState>());
    final titleController = useTextEditingController(
      text: existingItem?.title ?? '',
    );
    final isLoading = useState<bool>(false);

    Future<void> save() async {
      if (!formKey.currentState!.validate()) return;

      isLoading.value = true;
      try {
        // API 호출
        Navigator.of(context).pop(true);
      } catch (e) {
        ToastUtils.showError(context, '저장 실패: $e');
      } finally {
        isLoading.value = false;
      }
    }

    return Dialog(
      shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(16)),
      child: Container(
        width: 600,
        padding: const EdgeInsets.all(24),
        child: Form(
          key: formKey,
          child: Column(
            mainAxisSize: MainAxisSize.min,
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              Text(
                existingItem == null ? '추가' : '수정',
                style: Theme.of(context).textTheme.titleLarge?.copyWith(
                  fontWeight: FontWeight.bold,
                ),
              ),
              const SizedBox(height: 24),
              TextFormField(
                controller: titleController,
                decoration: const InputDecoration(
                  labelText: '제목 *',
                  border: OutlineInputBorder(),
                ),
                validator: (value) {
                  if (value == null || value.isEmpty) {
                    return '제목을 입력해주세요';
                  }
                  return null;
                },
              ),
              const SizedBox(height: 24),
              Row(
                mainAxisAlignment: MainAxisAlignment.end,
                children: [
                  TextButton(
                    onPressed: isLoading.value ? null : () => Navigator.of(context).pop(),
                    child: const Text('취소'),
                  ),
                  const SizedBox(width: 12),
                  ElevatedButton(
                    onPressed: isLoading.value ? null : save,
                    child: isLoading.value
                        ? const SizedBox(
                            width: 20,
                            height: 20,
                            child: CircularProgressIndicator(strokeWidth: 2),
                          )
                        : Text(existingItem == null ? '추가' : '수정'),
                  ),
                ],
              ),
            ],
          ),
        ),
      ),
    );
  }
}
```

### 11. Router (GoRouter + Riverpod)

```dart
// core/router/app_router.dart
import 'package:go_router/go_router.dart';
import 'package:riverpod_annotation/riverpod_annotation.dart';

part 'app_router.g.dart';

@Riverpod(keepAlive: true)
GoRouter appRouter(Ref ref) {
  final authState = ref.watch(authProvider);

  return GoRouter(
    initialLocation: '/home',
    debugLogDiagnostics: true,
    redirect: (context, state) {
      final isAuthenticated = authState is AuthAuthenticated;
      final isLoginPage = state.matchedLocation == '/login';

      if (!isAuthenticated && !isLoginPage) {
        return '/login';
      }

      if (isAuthenticated && isLoginPage) {
        return '/home';
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
            path: '/home',
            builder: (context, state) => const HomePage(),
          ),
          GoRoute(
            path: '/feature/:id',
            builder: (context, state) {
              final id = int.parse(state.pathParameters['id']!);
              return FeaturePage(id: id);
            },
          ),
        ],
      ),
    ],
  );
}
```

### 12. main.dart

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:sentry_flutter/sentry_flutter.dart';

Future<void> main() async {
  await SentryFlutter.init(
    (options) {
      options.dsn = 'YOUR_SENTRY_DSN';
      options.sendDefaultPii = true;
      options.tracesSampleRate = 1.0;
    },
    appRunner: () => runApp(
      const ProviderScope(child: MyApp()),
    ),
  );
}

class MyApp extends ConsumerWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final router = ref.watch(appRouterProvider);

    return MaterialApp.router(
      title: 'App Title',
      theme: ThemeData(
        primarySwatch: Colors.blue,
        useMaterial3: true,
      ),
      routerConfig: router,
      debugShowCheckedModeBanner: false,
    );
  }
}
```

## 네이밍 컨벤션

| 레이어 | 타입 | 네이밍 | 예시 |
|--------|------|--------|------|
| Domain | Entity | XxxEntity | StoryEntity |
| Domain | Repository Interface | XxxRepository (abstract) | StoryRepository |
| Domain | UseCase | GetXxxUseCase, CreateXxxUseCase | GetStoriesUseCase |
| Domain | Request Params | XxxParams | CreateStoryParams |
| Data | Model | XxxModel (extends Entity) | StoryModel |
| Data | DataSource Interface | XxxDataSource (abstract) | StoryDataSource |
| Data | DataSource Impl | XxxRemoteDataSourceImpl | StoryRemoteDataSourceImpl |
| Data | Repository Impl | XxxRepositoryImpl | StoryRepositoryImpl |

**파일명**: snake_case (예: story_repository.dart)
**클래스명**: PascalCase (예: StoryRepository)
**변수/함수명**: camelCase (예: getStories)
**상수**: kCamelCase (예: kBaseUrl)
**Provider**: xxxProvider (예: storyListProvider)

## 코드 생성

Provider와 Service 작성 후 반드시 실행:

```bash
flutter pub run build_runner build --delete-conflicting-outputs
```

## 주의사항

1. **utf8 디코딩**: API 응답은 항상 `utf8.decode(response.bodyBytes)` 사용
2. **에러 처리**: 모든 API 호출에 try-catch + Sentry 리포팅
3. **상태 업데이트**: 서버 호출 후 로컬 상태 갱신 (refresh 또는 로컬 업데이트)
4. **코드 생성**: `.g.dart` 파일은 수동 편집 금지
5. **Provider 파라미터**: Family provider 사용 시 build(int param) 패턴 사용
6. **클린 아키텍처 의존성**: Domain → 외부 의존 X, Data → Domain 의존, Presentation → Domain 의존
7. **Model은 Entity 상속**: Data 레이어의 Model은 항상 Domain의 Entity를 상속
8. **ResultFuture 사용**: Repository는 항상 `Either<Failure, T>` 타입 반환
