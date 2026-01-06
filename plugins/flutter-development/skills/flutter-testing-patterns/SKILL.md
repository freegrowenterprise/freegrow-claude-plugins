---
name: flutter-testing-patterns
description: Implement comprehensive Flutter testing with unit tests, widget tests, integration tests, and golden tests. Use when setting up test infrastructure, writing testable code, or achieving high test coverage.
---

# Flutter Testing Patterns

Production-ready testing patterns for Flutter applications including unit tests, widget tests, integration tests, and golden (snapshot) tests.

## When to Use This Skill

- Setting up testing infrastructure for Flutter projects
- Writing unit tests for business logic and usecases
- Creating widget tests for UI components
- Implementing integration tests for user flows
- Setting up golden tests for visual regression
- Achieving and maintaining high test coverage

## Core Concepts

### 1. Testing Pyramid

```
        ┌───────────────┐
        │  Integration  │  Slow, Expensive, High Confidence
        │     Tests     │
        └───────┬───────┘
        ┌───────▼───────┐
        │    Widget     │  Medium Speed, Medium Cost
        │     Tests     │
        └───────┬───────┘
        ┌───────▼───────┐
        │     Unit      │  Fast, Cheap, Low Integration
        │     Tests     │
        └───────────────┘
```

### 2. Test Types

| Type | Purpose | Speed | Tools |
|------|---------|-------|-------|
| **Unit** | Logic, usecases, models | Fast | `test`, `mockito` |
| **Widget** | UI components in isolation | Medium | `flutter_test` |
| **Golden** | Visual regression | Medium | `golden_toolkit` |
| **Integration** | Full app flows | Slow | `integration_test`, `patrol` |

## Quick Start

```yaml
# pubspec.yaml
dev_dependencies:
  flutter_test:
    sdk: flutter
  integration_test:
    sdk: flutter
  test: ^1.24.0
  mockito: ^5.4.0
  build_runner: ^2.4.0
  bloc_test: ^9.1.0  # For BLoC testing
  golden_toolkit: ^0.15.0
  patrol: ^3.0.0  # Advanced integration testing
```

```bash
# Run all tests
flutter test

# Run with coverage
flutter test --coverage

# Run integration tests
flutter test integration_test/

# Update golden files
flutter test --update-goldens
```

## Patterns

### Pattern 1: Unit Testing - UseCases & Repositories

```dart
// test/domain/usecases/get_user_test.dart
import 'package:dartz/dartz.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:mockito/annotations.dart';
import 'package:mockito/mockito.dart';
import 'package:my_app/core/errors/failures.dart';
import 'package:my_app/features/user/domain/entities/user_entity.dart';
import 'package:my_app/features/user/domain/repositories/user_repository.dart';
import 'package:my_app/features/user/domain/usecases/get_user.dart';

import 'get_user_test.mocks.dart';

@GenerateMocks([UserRepository])
void main() {
  late GetUser usecase;
  late MockUserRepository mockRepository;

  setUp(() {
    mockRepository = MockUserRepository();
    usecase = GetUser(mockRepository);
  });

  const tUserId = 'user-123';
  const tUserEntity = UserEntity(
    id: 'user-123',
    email: 'test@example.com',
    name: 'Test User',
  );

  group('GetUser', () {
    test('should get user from repository', () async {
      // Arrange
      when(mockRepository.getUser(any))
          .thenAnswer((_) async => const Right(tUserEntity));

      // Act
      final result = await usecase(tUserId);

      // Assert
      expect(result, const Right(tUserEntity));
      verify(mockRepository.getUser(tUserId));
      verifyNoMoreInteractions(mockRepository);
    });

    test('should return ServerFailure when repository fails', () async {
      // Arrange
      const tFailure = ServerFailure(message: 'Server error', statusCode: 500);
      when(mockRepository.getUser(any))
          .thenAnswer((_) async => const Left(tFailure));

      // Act
      final result = await usecase(tUserId);

      // Assert
      expect(result, const Left(tFailure));
      verify(mockRepository.getUser(tUserId));
    });
  });
}
```

```dart
// test/data/repositories/user_repository_impl_test.dart
import 'package:dartz/dartz.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:mockito/annotations.dart';
import 'package:mockito/mockito.dart';
import 'package:my_app/core/errors/exceptions.dart';
import 'package:my_app/core/errors/failures.dart';
import 'package:my_app/features/user/data/datasources/user_remote_datasource.dart';
import 'package:my_app/features/user/data/models/user_model.dart';
import 'package:my_app/features/user/data/repositories/user_repository_impl.dart';

import 'user_repository_impl_test.mocks.dart';

@GenerateMocks([UserRemoteDataSource])
void main() {
  late UserRepositoryImpl repository;
  late MockUserRemoteDataSource mockRemoteDataSource;

  setUp(() {
    mockRemoteDataSource = MockUserRemoteDataSource();
    repository = UserRepositoryImpl(mockRemoteDataSource);
  });

  const tUserModel = UserModel(
    id: 'user-123',
    email: 'test@example.com',
    name: 'Test User',
  );

  group('getUser', () {
    test('should return UserEntity when remote call is successful', () async {
      // Arrange
      when(mockRemoteDataSource.getUser(any))
          .thenAnswer((_) async => tUserModel);

      // Act
      final result = await repository.getUser('user-123');

      // Assert
      expect(result, const Right(tUserModel));
      verify(mockRemoteDataSource.getUser('user-123'));
    });

    test('should return ServerFailure when remote call throws ServerException',
        () async {
      // Arrange
      when(mockRemoteDataSource.getUser(any)).thenThrow(
        const ServerException(message: 'Server error', statusCode: 500),
      );

      // Act
      final result = await repository.getUser('user-123');

      // Assert
      expect(
        result,
        const Left(ServerFailure(message: 'Server error', statusCode: 500)),
      );
    });
  });
}
```

### Pattern 2: Widget Testing

```dart
// test/features/user/presentation/widgets/user_card_test.dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:my_app/features/user/domain/entities/user_entity.dart';
import 'package:my_app/features/user/presentation/widgets/user_card.dart';

void main() {
  const tUser = UserEntity(
    id: 'user-123',
    email: 'test@example.com',
    name: 'Test User',
  );

  Widget createWidgetUnderTest({
    UserEntity user = tUser,
    VoidCallback? onTap,
    VoidCallback? onDelete,
  }) {
    return MaterialApp(
      home: Scaffold(
        body: UserCard(
          user: user,
          onTap: onTap ?? () {},
          onDelete: onDelete ?? () {},
        ),
      ),
    );
  }

  group('UserCard', () {
    testWidgets('should display user name and email', (tester) async {
      // Arrange & Act
      await tester.pumpWidget(createWidgetUnderTest());

      // Assert
      expect(find.text('Test User'), findsOneWidget);
      expect(find.text('test@example.com'), findsOneWidget);
    });

    testWidgets('should call onTap when card is tapped', (tester) async {
      // Arrange
      var tapped = false;
      await tester.pumpWidget(createWidgetUnderTest(
        onTap: () => tapped = true,
      ));

      // Act
      await tester.tap(find.byType(UserCard));
      await tester.pump();

      // Assert
      expect(tapped, isTrue);
    });

    testWidgets('should call onDelete when delete icon is tapped',
        (tester) async {
      // Arrange
      var deleted = false;
      await tester.pumpWidget(createWidgetUnderTest(
        onDelete: () => deleted = true,
      ));

      // Act
      await tester.tap(find.byIcon(Icons.delete));
      await tester.pump();

      // Assert
      expect(deleted, isTrue);
    });

    testWidgets('should show confirmation dialog before delete', (tester) async {
      // Arrange
      await tester.pumpWidget(createWidgetUnderTest());

      // Act
      await tester.tap(find.byIcon(Icons.delete));
      await tester.pumpAndSettle();

      // Assert
      expect(find.text('Delete User'), findsOneWidget);
      expect(find.text('Are you sure?'), findsOneWidget);
    });
  });
}
```

```dart
// test/features/user/presentation/pages/user_list_page_test.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:mockito/annotations.dart';
import 'package:mockito/mockito.dart';
import 'package:my_app/features/user/domain/entities/user_entity.dart';
import 'package:my_app/features/user/presentation/pages/user_list_page.dart';
import 'package:my_app/features/user/presentation/providers/user_list_provider.dart';

import 'user_list_page_test.mocks.dart';

@GenerateMocks([UserListNotifier])
void main() {
  late MockUserListNotifier mockNotifier;

  setUp(() {
    mockNotifier = MockUserListNotifier();
  });

  const tUsers = [
    UserEntity(id: '1', email: 'user1@example.com', name: 'User 1'),
    UserEntity(id: '2', email: 'user2@example.com', name: 'User 2'),
  ];

  Widget createWidgetUnderTest(AsyncValue<List<UserEntity>> state) {
    return ProviderScope(
      overrides: [
        userListProvider.overrideWith((ref) {
          when(mockNotifier.build()).thenReturn(state);
          return mockNotifier;
        }),
      ],
      child: const MaterialApp(
        home: UserListPage(),
      ),
    );
  }

  group('UserListPage', () {
    testWidgets('should show loading indicator when state is loading',
        (tester) async {
      // Arrange & Act
      await tester.pumpWidget(
        createWidgetUnderTest(const AsyncValue.loading()),
      );

      // Assert
      expect(find.byType(CircularProgressIndicator), findsOneWidget);
    });

    testWidgets('should show user list when state has data', (tester) async {
      // Arrange & Act
      await tester.pumpWidget(
        createWidgetUnderTest(const AsyncValue.data(tUsers)),
      );

      // Assert
      expect(find.text('User 1'), findsOneWidget);
      expect(find.text('User 2'), findsOneWidget);
    });

    testWidgets('should show error message when state has error',
        (tester) async {
      // Arrange & Act
      await tester.pumpWidget(
        createWidgetUnderTest(
          AsyncValue.error('Failed to load users', StackTrace.current),
        ),
      );

      // Assert
      expect(find.text('Failed to load users'), findsOneWidget);
      expect(find.text('Retry'), findsOneWidget);
    });

    testWidgets('should show empty state when list is empty', (tester) async {
      // Arrange & Act
      await tester.pumpWidget(
        createWidgetUnderTest(const AsyncValue.data([])),
      );

      // Assert
      expect(find.text('No users found'), findsOneWidget);
    });
  });
}
```

### Pattern 3: BLoC Testing

```dart
// test/features/user/presentation/bloc/user_bloc_test.dart
import 'package:bloc_test/bloc_test.dart';
import 'package:dartz/dartz.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:mockito/annotations.dart';
import 'package:mockito/mockito.dart';
import 'package:my_app/core/errors/failures.dart';
import 'package:my_app/features/user/domain/entities/user_entity.dart';
import 'package:my_app/features/user/domain/usecases/get_users.dart';
import 'package:my_app/features/user/presentation/bloc/user_bloc.dart';
import 'package:my_app/features/user/presentation/bloc/user_event.dart';
import 'package:my_app/features/user/presentation/bloc/user_state.dart';

import 'user_bloc_test.mocks.dart';

@GenerateMocks([GetUsers])
void main() {
  late UserBloc bloc;
  late MockGetUsers mockGetUsers;

  setUp(() {
    mockGetUsers = MockGetUsers();
    bloc = UserBloc(getUsers: mockGetUsers);
  });

  tearDown(() {
    bloc.close();
  });

  const tUsers = [
    UserEntity(id: '1', email: 'user1@example.com', name: 'User 1'),
    UserEntity(id: '2', email: 'user2@example.com', name: 'User 2'),
  ];

  test('initial state should be UserInitial', () {
    expect(bloc.state, const UserInitial());
  });

  group('LoadUsers', () {
    blocTest<UserBloc, UserState>(
      'emits [UserLoading, UsersLoaded] when LoadUsers is successful',
      build: () {
        when(mockGetUsers()).thenAnswer((_) async => const Right(tUsers));
        return bloc;
      },
      act: (bloc) => bloc.add(const LoadUsers()),
      expect: () => [
        const UserLoading(),
        const UsersLoaded(tUsers),
      ],
      verify: (_) {
        verify(mockGetUsers()).called(1);
      },
    );

    blocTest<UserBloc, UserState>(
      'emits [UserLoading, UserError] when LoadUsers fails',
      build: () {
        when(mockGetUsers()).thenAnswer(
          (_) async => const Left(ServerFailure(message: 'Server error')),
        );
        return bloc;
      },
      act: (bloc) => bloc.add(const LoadUsers()),
      expect: () => [
        const UserLoading(),
        const UserError('Server error'),
      ],
    );

    blocTest<UserBloc, UserState>(
      'does not emit new states when already loading',
      build: () {
        when(mockGetUsers()).thenAnswer(
          (_) async => Future.delayed(
            const Duration(seconds: 1),
            () => const Right(tUsers),
          ),
        );
        return bloc;
      },
      seed: () => const UserLoading(),
      act: (bloc) => bloc.add(const LoadUsers()),
      expect: () => [],
    );
  });
}
```

### Pattern 4: Golden Tests (Visual Regression)

```dart
// test/golden/user_card_golden_test.dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:golden_toolkit/golden_toolkit.dart';
import 'package:my_app/features/user/domain/entities/user_entity.dart';
import 'package:my_app/features/user/presentation/widgets/user_card.dart';

void main() {
  const tUser = UserEntity(
    id: 'user-123',
    email: 'test@example.com',
    name: 'Test User',
  );

  group('UserCard Golden Tests', () {
    testGoldens('UserCard - default state', (tester) async {
      final builder = GoldenBuilder.column()
        ..addScenario(
          'Default',
          UserCard(user: tUser, onTap: () {}, onDelete: () {}),
        )
        ..addScenario(
          'Long name',
          UserCard(
            user: const UserEntity(
              id: '2',
              email: 'longname@example.com',
              name: 'This Is A Very Long User Name That Should Be Truncated',
            ),
            onTap: () {},
            onDelete: () {},
          ),
        );

      await tester.pumpWidgetBuilder(
        builder.build(),
        surfaceSize: const Size(400, 300),
        wrapper: materialAppWrapper(theme: ThemeData.light()),
      );

      await screenMatchesGolden(tester, 'user_card_default');
    });

    testGoldens('UserCard - dark theme', (tester) async {
      await tester.pumpWidgetBuilder(
        UserCard(user: tUser, onTap: () {}, onDelete: () {}),
        surfaceSize: const Size(400, 100),
        wrapper: materialAppWrapper(theme: ThemeData.dark()),
      );

      await screenMatchesGolden(tester, 'user_card_dark');
    });

    testGoldens('UserCard - multi-device', (tester) async {
      final builder = DeviceBuilder()
        ..overrideDevicesForAllScenarios(devices: [
          Device.phone,
          Device.iphone11,
          Device.tabletLandscape,
        ])
        ..addScenario(
          widget: UserCard(user: tUser, onTap: () {}, onDelete: () {}),
          name: 'default',
        );

      await tester.pumpDeviceBuilder(builder);
      await screenMatchesGolden(tester, 'user_card_multi_device');
    });
  });
}

// test/golden/flutter_test_config.dart
import 'dart:async';
import 'package:golden_toolkit/golden_toolkit.dart';

Future<void> testExecutable(FutureOr<void> Function() testMain) async {
  await loadAppFonts();
  return testMain();
}
```

### Pattern 5: Integration Testing with Patrol

```dart
// integration_test/app_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:integration_test/integration_test.dart';
import 'package:patrol/patrol.dart';
import 'package:my_app/main.dart' as app;

void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  group('App Integration Tests', () {
    patrolTest(
      'User can login and view user list',
      ($) async {
        app.main();
        await $.pumpAndSettle();

        // Login screen
        expect($('Login'), findsOneWidget);

        await $(TextField).at(0).enterText('test@example.com');
        await $(TextField).at(1).enterText('password123');
        await $('Login').tap();

        await $.pumpAndSettle();

        // User list screen
        expect($('Users'), findsOneWidget);
        expect($(ListView), findsOneWidget);
      },
    );

    patrolTest(
      'User can create a new user',
      ($) async {
        app.main();
        await $.pumpAndSettle();

        // Navigate to user list (assuming logged in)
        await $('Users').tap();
        await $.pumpAndSettle();

        // Tap FAB to create user
        await $(FloatingActionButton).tap();
        await $.pumpAndSettle();

        // Fill form
        await $('Name').enterText('New User');
        await $('Email').enterText('newuser@example.com');
        await $('Save').tap();

        await $.pumpAndSettle();

        // Verify user was added
        expect($('New User'), findsOneWidget);
      },
    );

    patrolTest(
      'User can delete a user',
      ($) async {
        app.main();
        await $.pumpAndSettle();

        // Find delete button and tap
        await $(Icons.delete).first.tap();
        await $.pumpAndSettle();

        // Confirm deletion
        expect($('Delete User'), findsOneWidget);
        await $('Confirm').tap();

        await $.pumpAndSettle();

        // Verify deletion success
        expect($('User deleted'), findsOneWidget);
      },
    );
  });
}
```

```dart
// integration_test/robots/user_robot.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:patrol/patrol.dart';

class UserRobot {
  final PatrolIntegrationTester $;

  UserRobot(this.$);

  Future<void> navigateToUserList() async {
    await $('Users').tap();
    await $.pumpAndSettle();
  }

  Future<void> tapCreateUser() async {
    await $(FloatingActionButton).tap();
    await $.pumpAndSettle();
  }

  Future<void> fillUserForm({
    required String name,
    required String email,
  }) async {
    await $('Name').enterText(name);
    await $('Email').enterText(email);
  }

  Future<void> submitForm() async {
    await $('Save').tap();
    await $.pumpAndSettle();
  }

  Future<void> deleteUser(String userName) async {
    await $(userName).scrollTo().tap();
    await $(Icons.delete).tap();
    await $.pumpAndSettle();
    await $('Confirm').tap();
    await $.pumpAndSettle();
  }

  void verifyUserExists(String userName) {
    expect($(userName), findsOneWidget);
  }

  void verifyUserNotExists(String userName) {
    expect($(userName), findsNothing);
  }
}

// Usage
patrolTest('Create user flow', ($) async {
  app.main();
  await $.pumpAndSettle();

  final userRobot = UserRobot($);

  await userRobot.navigateToUserList();
  await userRobot.tapCreateUser();
  await userRobot.fillUserForm(name: 'Test User', email: 'test@example.com');
  await userRobot.submitForm();

  userRobot.verifyUserExists('Test User');
});
```

### Pattern 6: Test Utilities & Helpers

```dart
// test/helpers/test_helpers.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_test/flutter_test.dart';

/// Wrap widget with MaterialApp for testing
Widget testableWidget(Widget child, {List<Override>? overrides}) {
  return ProviderScope(
    overrides: overrides ?? [],
    child: MaterialApp(
      home: child,
    ),
  );
}

/// Wrap widget with Scaffold for testing
Widget testableScaffold(Widget child, {List<Override>? overrides}) {
  return ProviderScope(
    overrides: overrides ?? [],
    child: MaterialApp(
      home: Scaffold(body: child),
    ),
  );
}

/// Pump widget and wait for all animations
extension WidgetTesterExtension on WidgetTester {
  Future<void> pumpApp(Widget widget, {List<Override>? overrides}) async {
    await pumpWidget(testableWidget(widget, overrides: overrides));
    await pumpAndSettle();
  }

  Future<void> tapAndSettle(Finder finder) async {
    await tap(finder);
    await pumpAndSettle();
  }
}

/// Custom finders
Finder findByTooltip(String tooltip) => find.byTooltip(tooltip);
Finder findByTextContaining(String text) => find.textContaining(text);
```

```dart
// test/fixtures/user_fixtures.dart
import 'package:my_app/features/user/domain/entities/user_entity.dart';
import 'package:my_app/features/user/data/models/user_model.dart';

class UserFixtures {
  static const user1 = UserEntity(
    id: 'user-1',
    email: 'user1@example.com',
    name: 'User One',
  );

  static const user2 = UserEntity(
    id: 'user-2',
    email: 'user2@example.com',
    name: 'User Two',
  );

  static const users = [user1, user2];

  static const userModel1 = UserModel(
    id: 'user-1',
    email: 'user1@example.com',
    name: 'User One',
  );

  static Map<String, dynamic> get userJson1 => {
        'id': 'user-1',
        'email': 'user1@example.com',
        'name': 'User One',
      };
}
```

## Best Practices

### Do's
- **Test behavior, not implementation** - Focus on what, not how
- **Use AAA pattern** - Arrange, Act, Assert
- **Mock external dependencies** - Keep tests isolated
- **Name tests descriptively** - `should_returnUser_when_idIsValid`
- **Use fixtures** - Reuse test data across tests
- **Run tests in CI** - Catch regressions early

### Don'ts
- **Don't test Flutter/Dart internals** - Trust the framework
- **Don't over-mock** - Some integration is good
- **Don't skip edge cases** - Test error states too
- **Don't ignore flaky tests** - Fix or remove them
- **Don't hardcode expectations** - Use constants/fixtures
- **Don't test private methods** - Test through public API

## Resources

- [Flutter Testing Documentation](https://docs.flutter.dev/testing)
- [Mockito Package](https://pub.dev/packages/mockito)
- [bloc_test Package](https://pub.dev/packages/bloc_test)
- [Golden Toolkit](https://pub.dev/packages/golden_toolkit)
- [Patrol](https://patrol.leancode.co/)
