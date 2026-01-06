---
name: flutter-state-management
description: Master Flutter state management with Riverpod 2.x, BLoC, Provider, and GetX. Use when choosing state management solutions, implementing global state, or managing complex app state.
---

# Flutter State Management

Comprehensive guide to modern Flutter state management patterns with Riverpod, BLoC, Provider, and GetX for scalable applications.

## When to Use This Skill

- Choosing the right state management solution for your app
- Implementing Riverpod 2.x with code generation
- Setting up BLoC/Cubit for event-driven architecture
- Managing complex application state
- Implementing optimistic updates and caching
- Migrating between state management solutions

## Core Concepts

### 1. State Management Comparison

| Solution | Complexity | Learning Curve | Best For |
|----------|------------|----------------|----------|
| **Riverpod** | Medium | Medium | Modern apps, compile-time safety |
| **BLoC** | High | High | Large teams, complex business logic |
| **Provider** | Low | Low | Simple apps, beginners |
| **GetX** | Low | Low | Rapid prototyping, small apps |
| **Stacked** | Medium | Medium | MVVM architecture |

### 2. Selection Criteria

```
Simple app, few screens     → Provider
Medium app, type safety     → Riverpod
Large app, complex flows    → BLoC
Fast prototyping            → GetX
Enterprise, strict MVVM     → Stacked
```

## Quick Start

### Riverpod Setup

```bash
flutter pub add flutter_riverpod riverpod_annotation
flutter pub add dev:riverpod_generator dev:build_runner
```

### BLoC Setup

```bash
flutter pub add flutter_bloc bloc equatable
flutter pub add dev:bloc_test
```

## Patterns

### Pattern 1: Riverpod 2.x with Code Generation

```dart
// providers/user_provider.dart
import 'package:riverpod_annotation/riverpod_annotation.dart';
import '../domain/entities/user_entity.dart';
import '../domain/usecases/get_user.dart';
import '../di/injection.dart';

part 'user_provider.g.dart';

// Simple state provider
@riverpod
class Counter extends _$Counter {
  @override
  int build() => 0;

  void increment() => state++;
  void decrement() => state--;
}

// Async provider with family parameter
@riverpod
class UserDetail extends _$UserDetail {
  @override
  Future<UserEntity> build(String userId) async {
    return await _fetchUser(userId);
  }

  Future<UserEntity> _fetchUser(String userId) async {
    final getUser = getIt<GetUser>();
    final result = await getUser(userId);

    return result.fold(
      (failure) => throw Exception(failure.message),
      (user) => user,
    );
  }

  Future<void> refresh() async {
    state = const AsyncValue.loading();
    state = await AsyncValue.guard(() => _fetchUser(arg));
  }
}

// List provider with CRUD operations
@riverpod
class UserList extends _$UserList {
  @override
  Future<List<UserEntity>> build() async {
    return await _fetchUsers();
  }

  Future<List<UserEntity>> _fetchUsers() async {
    final getUsers = getIt<GetUsers>();
    final result = await getUsers();

    return result.fold(
      (failure) => throw Exception(failure.message),
      (users) => users,
    );
  }

  // Optimistic add
  Future<void> addUser(UserEntity user) async {
    final previousState = state.valueOrNull ?? [];
    state = AsyncValue.data([...previousState, user]);

    try {
      final createUser = getIt<CreateUser>();
      final result = await createUser(user);

      result.fold(
        (failure) {
          // Rollback on failure
          state = AsyncValue.data(previousState);
          throw Exception(failure.message);
        },
        (_) => null,
      );
    } catch (e) {
      state = AsyncValue.data(previousState);
      rethrow;
    }
  }

  // Optimistic remove
  void removeUser(String id) {
    final currentUsers = state.valueOrNull ?? [];
    state = AsyncValue.data(
      currentUsers.where((user) => user.id != id).toList(),
    );
  }

  // Optimistic update
  void updateUser(UserEntity updatedUser) {
    final currentUsers = state.valueOrNull ?? [];
    state = AsyncValue.data(
      currentUsers
          .map((user) => user.id == updatedUser.id ? updatedUser : user)
          .toList(),
    );
  }
}
```

```dart
// Usage in widget
import 'package:flutter/material.dart';
import 'package:flutter_hooks/flutter_hooks.dart';
import 'package:hooks_riverpod/hooks_riverpod.dart';

class UserListPage extends HookConsumerWidget {
  const UserListPage({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final usersAsync = ref.watch(userListProvider);
    final searchController = useTextEditingController();

    return Scaffold(
      appBar: AppBar(title: const Text('Users')),
      body: usersAsync.when(
        loading: () => const Center(child: CircularProgressIndicator()),
        error: (error, stack) => Center(
          child: Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              Text('Error: $error'),
              ElevatedButton(
                onPressed: () => ref.invalidate(userListProvider),
                child: const Text('Retry'),
              ),
            ],
          ),
        ),
        data: (users) => RefreshIndicator(
          onRefresh: () async => ref.invalidate(userListProvider),
          child: ListView.builder(
            itemCount: users.length,
            itemBuilder: (context, index) {
              final user = users[index];
              return ListTile(
                title: Text(user.name),
                subtitle: Text(user.email),
                trailing: IconButton(
                  icon: const Icon(Icons.delete),
                  onPressed: () {
                    ref.read(userListProvider.notifier).removeUser(user.id);
                  },
                ),
              );
            },
          ),
        ),
      ),
    );
  }
}
```

### Pattern 2: BLoC Pattern with Events & States

```dart
// bloc/user/user_event.dart
import 'package:equatable/equatable.dart';

abstract class UserEvent extends Equatable {
  const UserEvent();

  @override
  List<Object?> get props => [];
}

class LoadUsers extends UserEvent {
  const LoadUsers();
}

class LoadUserDetail extends UserEvent {
  final String userId;

  const LoadUserDetail(this.userId);

  @override
  List<Object?> get props => [userId];
}

class CreateUser extends UserEvent {
  final String name;
  final String email;

  const CreateUser({required this.name, required this.email});

  @override
  List<Object?> get props => [name, email];
}

class DeleteUser extends UserEvent {
  final String userId;

  const DeleteUser(this.userId);

  @override
  List<Object?> get props => [userId];
}
```

```dart
// bloc/user/user_state.dart
import 'package:equatable/equatable.dart';
import '../../domain/entities/user_entity.dart';

abstract class UserState extends Equatable {
  const UserState();

  @override
  List<Object?> get props => [];
}

class UserInitial extends UserState {
  const UserInitial();
}

class UserLoading extends UserState {
  const UserLoading();
}

class UsersLoaded extends UserState {
  final List<UserEntity> users;

  const UsersLoaded(this.users);

  @override
  List<Object?> get props => [users];
}

class UserDetailLoaded extends UserState {
  final UserEntity user;

  const UserDetailLoaded(this.user);

  @override
  List<Object?> get props => [user];
}

class UserError extends UserState {
  final String message;

  const UserError(this.message);

  @override
  List<Object?> get props => [message];
}

class UserCreated extends UserState {
  const UserCreated();
}

class UserDeleted extends UserState {
  const UserDeleted();
}
```

```dart
// bloc/user/user_bloc.dart
import 'package:flutter_bloc/flutter_bloc.dart';
import '../../domain/usecases/get_users.dart';
import '../../domain/usecases/get_user.dart';
import '../../domain/usecases/create_user.dart';
import '../../domain/usecases/delete_user.dart';
import 'user_event.dart';
import 'user_state.dart';

class UserBloc extends Bloc<UserEvent, UserState> {
  final GetUsers _getUsers;
  final GetUser _getUser;
  final CreateUserUseCase _createUser;
  final DeleteUserUseCase _deleteUser;

  UserBloc({
    required GetUsers getUsers,
    required GetUser getUser,
    required CreateUserUseCase createUser,
    required DeleteUserUseCase deleteUser,
  })  : _getUsers = getUsers,
        _getUser = getUser,
        _createUser = createUser,
        _deleteUser = deleteUser,
        super(const UserInitial()) {
    on<LoadUsers>(_onLoadUsers);
    on<LoadUserDetail>(_onLoadUserDetail);
    on<CreateUser>(_onCreateUser);
    on<DeleteUser>(_onDeleteUser);
  }

  Future<void> _onLoadUsers(
    LoadUsers event,
    Emitter<UserState> emit,
  ) async {
    emit(const UserLoading());

    final result = await _getUsers();

    result.fold(
      (failure) => emit(UserError(failure.message)),
      (users) => emit(UsersLoaded(users)),
    );
  }

  Future<void> _onLoadUserDetail(
    LoadUserDetail event,
    Emitter<UserState> emit,
  ) async {
    emit(const UserLoading());

    final result = await _getUser(event.userId);

    result.fold(
      (failure) => emit(UserError(failure.message)),
      (user) => emit(UserDetailLoaded(user)),
    );
  }

  Future<void> _onCreateUser(
    CreateUser event,
    Emitter<UserState> emit,
  ) async {
    emit(const UserLoading());

    final result = await _createUser(
      CreateUserParams(name: event.name, email: event.email),
    );

    result.fold(
      (failure) => emit(UserError(failure.message)),
      (_) {
        emit(const UserCreated());
        add(const LoadUsers()); // Refresh list
      },
    );
  }

  Future<void> _onDeleteUser(
    DeleteUser event,
    Emitter<UserState> emit,
  ) async {
    final currentState = state;

    // Optimistic update
    if (currentState is UsersLoaded) {
      emit(UsersLoaded(
        currentState.users.where((u) => u.id != event.userId).toList(),
      ));
    }

    final result = await _deleteUser(event.userId);

    result.fold(
      (failure) {
        // Rollback on failure
        if (currentState is UsersLoaded) {
          emit(currentState);
        }
        emit(UserError(failure.message));
      },
      (_) => null,
    );
  }
}
```

```dart
// Usage with BlocProvider
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';

class UserListPage extends StatelessWidget {
  const UserListPage({super.key});

  @override
  Widget build(BuildContext context) {
    return BlocProvider(
      create: (context) => getIt<UserBloc>()..add(const LoadUsers()),
      child: const UserListView(),
    );
  }
}

class UserListView extends StatelessWidget {
  const UserListView({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Users')),
      body: BlocBuilder<UserBloc, UserState>(
        builder: (context, state) {
          if (state is UserLoading) {
            return const Center(child: CircularProgressIndicator());
          }

          if (state is UserError) {
            return Center(
              child: Column(
                mainAxisAlignment: MainAxisAlignment.center,
                children: [
                  Text(state.message),
                  ElevatedButton(
                    onPressed: () {
                      context.read<UserBloc>().add(const LoadUsers());
                    },
                    child: const Text('Retry'),
                  ),
                ],
              ),
            );
          }

          if (state is UsersLoaded) {
            return ListView.builder(
              itemCount: state.users.length,
              itemBuilder: (context, index) {
                final user = state.users[index];
                return ListTile(
                  title: Text(user.name),
                  subtitle: Text(user.email),
                  trailing: IconButton(
                    icon: const Icon(Icons.delete),
                    onPressed: () {
                      context.read<UserBloc>().add(DeleteUser(user.id));
                    },
                  ),
                );
              },
            );
          }

          return const SizedBox.shrink();
        },
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () => _showCreateDialog(context),
        child: const Icon(Icons.add),
      ),
    );
  }
}
```

### Pattern 3: Cubit (Simplified BLoC)

```dart
// cubit/counter_cubit.dart
import 'package:flutter_bloc/flutter_bloc.dart';

class CounterCubit extends Cubit<int> {
  CounterCubit() : super(0);

  void increment() => emit(state + 1);
  void decrement() => emit(state - 1);
  void reset() => emit(0);
}

// cubit/theme_cubit.dart
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';

class ThemeCubit extends Cubit<ThemeMode> {
  ThemeCubit() : super(ThemeMode.system);

  void setLight() => emit(ThemeMode.light);
  void setDark() => emit(ThemeMode.dark);
  void setSystem() => emit(ThemeMode.system);
  void toggle() {
    emit(state == ThemeMode.dark ? ThemeMode.light : ThemeMode.dark);
  }
}
```

```dart
// cubit/user_list_cubit.dart
import 'package:equatable/equatable.dart';
import 'package:flutter_bloc/flutter_bloc.dart';
import '../../domain/entities/user_entity.dart';
import '../../domain/usecases/get_users.dart';

part 'user_list_state.dart';

class UserListCubit extends Cubit<UserListState> {
  final GetUsers _getUsers;

  UserListCubit(this._getUsers) : super(const UserListState());

  Future<void> loadUsers() async {
    emit(state.copyWith(status: UserListStatus.loading));

    final result = await _getUsers();

    result.fold(
      (failure) => emit(state.copyWith(
        status: UserListStatus.failure,
        errorMessage: failure.message,
      )),
      (users) => emit(state.copyWith(
        status: UserListStatus.success,
        users: users,
      )),
    );
  }

  void filterUsers(String query) {
    if (query.isEmpty) {
      emit(state.copyWith(filteredUsers: state.users));
      return;
    }

    final filtered = state.users
        .where((user) =>
            user.name.toLowerCase().contains(query.toLowerCase()) ||
            user.email.toLowerCase().contains(query.toLowerCase()))
        .toList();

    emit(state.copyWith(filteredUsers: filtered));
  }
}

// user_list_state.dart
part of 'user_list_cubit.dart';

enum UserListStatus { initial, loading, success, failure }

class UserListState extends Equatable {
  final UserListStatus status;
  final List<UserEntity> users;
  final List<UserEntity> filteredUsers;
  final String? errorMessage;

  const UserListState({
    this.status = UserListStatus.initial,
    this.users = const [],
    this.filteredUsers = const [],
    this.errorMessage,
  });

  UserListState copyWith({
    UserListStatus? status,
    List<UserEntity>? users,
    List<UserEntity>? filteredUsers,
    String? errorMessage,
  }) {
    return UserListState(
      status: status ?? this.status,
      users: users ?? this.users,
      filteredUsers: filteredUsers ?? this.filteredUsers,
      errorMessage: errorMessage ?? this.errorMessage,
    );
  }

  @override
  List<Object?> get props => [status, users, filteredUsers, errorMessage];
}
```

### Pattern 4: GetX (Simple & Reactive)

```dart
// controllers/user_controller.dart
import 'package:get/get.dart';
import '../domain/entities/user_entity.dart';
import '../services/user_service.dart';

class UserController extends GetxController {
  final UserService _userService;

  UserController(this._userService);

  final _users = <UserEntity>[].obs;
  final _isLoading = false.obs;
  final _error = RxnString();

  List<UserEntity> get users => _users;
  bool get isLoading => _isLoading.value;
  String? get error => _error.value;

  @override
  void onInit() {
    super.onInit();
    loadUsers();
  }

  Future<void> loadUsers() async {
    _isLoading.value = true;
    _error.value = null;

    try {
      final result = await _userService.getUsers();
      _users.assignAll(result);
    } catch (e) {
      _error.value = e.toString();
    } finally {
      _isLoading.value = false;
    }
  }

  Future<void> createUser(String name, String email) async {
    try {
      final user = await _userService.createUser(name, email);
      _users.add(user);
    } catch (e) {
      Get.snackbar('Error', e.toString());
    }
  }

  void deleteUser(String id) {
    _users.removeWhere((user) => user.id == id);
  }
}

// Binding
class UserBinding extends Bindings {
  @override
  void dependencies() {
    Get.lazyPut<UserService>(() => UserServiceImpl());
    Get.lazyPut<UserController>(() => UserController(Get.find()));
  }
}

// Usage
class UserListPage extends GetView<UserController> {
  const UserListPage({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Users')),
      body: Obx(() {
        if (controller.isLoading) {
          return const Center(child: CircularProgressIndicator());
        }

        if (controller.error != null) {
          return Center(child: Text(controller.error!));
        }

        return ListView.builder(
          itemCount: controller.users.length,
          itemBuilder: (context, index) {
            final user = controller.users[index];
            return ListTile(
              title: Text(user.name),
              subtitle: Text(user.email),
            );
          },
        );
      }),
    );
  }
}
```

### Pattern 5: Combined Approach (Recommended)

```dart
// Use Riverpod for dependency injection and local state
// Use specific state management for complex features

// app/di/providers.dart
import 'package:riverpod_annotation/riverpod_annotation.dart';
import '../domain/repositories/user_repository.dart';
import '../data/repositories/user_repository_impl.dart';

part 'providers.g.dart';

@Riverpod(keepAlive: true)
UserRepository userRepository(Ref ref) {
  return UserRepositoryImpl(ref.watch(userRemoteDataSourceProvider));
}

// Feature-specific provider
@riverpod
class UserNotifier extends _$UserNotifier {
  @override
  AsyncValue<List<UserEntity>> build() {
    _loadUsers();
    return const AsyncValue.loading();
  }

  Future<void> _loadUsers() async {
    final repository = ref.read(userRepositoryProvider);
    final result = await repository.getUsers();

    state = result.fold(
      (failure) => AsyncValue.error(failure.message, StackTrace.current),
      (users) => AsyncValue.data(users),
    );
  }

  Future<void> refresh() async {
    state = const AsyncValue.loading();
    await _loadUsers();
  }
}
```

## Best Practices

### Do's
- **Choose based on team size** - BLoC for large teams, Riverpod for medium
- **Keep state minimal** - Only store what can't be derived
- **Use immutable state** - Prevents unexpected mutations
- **Separate UI from business logic** - State management shouldn't know about widgets
- **Test state changes** - Each state transition should be testable
- **Handle loading and error states** - Always show feedback to users

### Don'ts
- **Don't mix solutions** - Pick one primary solution per feature
- **Don't store derived data** - Compute it instead
- **Don't put API calls in widgets** - Use usecases or services
- **Don't ignore error handling** - Always handle failure cases
- **Don't make everything global** - Keep state as local as possible
- **Don't skip the async wrapper** - Use AsyncValue/AsyncState patterns

## Resources

- [Riverpod Documentation](https://riverpod.dev/)
- [BLoC Library](https://bloclibrary.dev/)
- [Provider Package](https://pub.dev/packages/provider)
- [GetX Documentation](https://pub.dev/packages/get)
- [Flutter State Management Guide](https://docs.flutter.dev/development/data-and-backend/state-mgmt)
