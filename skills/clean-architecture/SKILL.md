---
name: clean-architecture
description: "Use when structuring enterprise applications, defining Domain/Data/Presentation layers, implementing Use Cases, isolating entities, mapping DTOs, or writing clean architecture boundaries in Dart and Flutter."
---

# Clean Architecture in Dart & Flutter

This skill provides complete, production-ready guidance for implementing **Clean Architecture** in Dart and Flutter applications.

---

## When to Use

Use this skill when:
* Building or refactoring enterprise-scale Flutter applications.
* Defining architectural boundaries between Domain, Data, and Presentation layers.
* Implementing Use Cases / Interactors for business workflows.
* Enforcing strict dependency rules (inward dependency flow).
* Decoupling business logic from Flutter UI, state management packages, and external SDKs.
* Establishing robust data transformation boundaries (Data Transfer Objects `DTO` $\leftrightarrow$ Domain Entities `Entity` $\leftrightarrow$ Presentation Models `ViewState`).

---

## Clean Architecture Layers & Dependency Rule

Clean Architecture organizes code into concentric concentric layers with a single strict rule: **Dependencies point inward only**.

```
┌─────────────────────────────────────────────────────────────┐
│ 1. Presentation Layer (Outer)                               │
│    - Widgets, Screens, UI Layouts                           │
│    - State Holders (Bloc / Cubit / Riverpod / ViewModel)    │
└──────────────────────────────┬──────────────────────────────┘
                               │ depends on
                               ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. Data Layer (Outer / Framework boundary)                   │
│    - Data Sources (REST HTTP Clients, Local DBs, SharedPrefs)│
│    - Repository Implementations                             │
│    - DTOs / JSON Serialization Models                       │
└──────────────────────────────┬──────────────────────────────┘
                               │ implements interfaces from
                               ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. Domain Layer (Inner Core — Pure Dart)                    │
│    - Business Entities & Value Objects                      │
│    - Use Cases / Interactors                                │
│    - Abstract Repository Contracts                          │
│    - Failure / Exception Abstractions                       │
└─────────────────────────────────────────────────────────────┘
```

> **Crucial Rule**: The **Domain Layer** is pure Dart. It must never import `flutter/material.dart`, third-party packages (`dio`, `sqflite`, `firebase`), or code from Data/Presentation layers.

---

## Standard Directory Structure (Feature-First Clean Architecture)

Combine Clean Architecture layers with Feature-First organization for maximum scalability:

```
lib/
├── core/                                    # Global core shared across features
│   ├── error/
│   │   ├── failures.dart                    # Domain Failures
│   │   └── exceptions.dart                  # Data Exceptions
│   ├── network/
│   │   └── api_client.dart
│   └── usecase/
│       └── usecase.dart                     # Base UseCase interface
├── features/
│   └── authentication/                      # Feature Directory
│       ├── domain/                          # INNER LAYER
│       │   ├── entities/
│       │   │   └── user_entity.dart
│       │   ├── repositories/
│       │   │   └── auth_repository.dart     # Abstract interface
│       │   └── usecases/
│       │       ├── login_usecase.dart
│       │       └── get_current_user.dart
│       ├── data/                            # MIDDLE LAYER
│       │   ├── datasources/
│       │   │   ├── auth_remote_data_source.dart
│       │   │   └── auth_local_data_source.dart
│       │   ├── models/
│       │   │   └── user_model.dart          # DTO with toJson/fromJson
│       │   └── repositories/
│       │       └── auth_repository_impl.dart# Implements domain auth_repository
│       └── presentation/                    # OUTER LAYER
│           ├── bloc/ / controllers/
│           │   ├── auth_bloc.dart
│           │   ├── auth_event.dart
│           │   └── auth_state.dart
│           └── pages/ / widgets/
│               ├── login_page.dart
│               └── widgets/
```

---

## Step-by-Step Layer Implementation Guide

### 1. Domain Layer (Core Business Rules)

#### A. Base UseCase Interface (`lib/core/usecase/usecase.dart`)
```dart
import 'package:dartz/dartz.dart'; // Or custom Result/Either class
import '../error/failures.dart';

abstract class UseCase<Type, Params> {
  Future<Either<Failure, Type>> call(Params params);
}

class NoParams {}
```

#### B. Entity (`lib/features/auth/domain/entities/user_entity.dart`)
```dart
import 'package:equatable/equatable.dart';

class UserEntity extends Equatable {
  final String id;
  final String email;
  final String name;

  const UserEntity({
    required this.id,
    required this.email,
    required this.name,
  });

  @override
  List<Object?> get props => [id, email, name];
}
```

#### C. Abstract Repository Interface (`lib/features/auth/domain/repositories/auth_repository.dart`)
```dart
import 'package:dartz/dartz.dart';
import '../../../../core/error/failures.dart';
import '../entities/user_entity.dart';

abstract class AuthRepository {
  Future<Either<Failure, UserEntity>> login({
    required String email,
    required String password,
  });
}
```

#### D. Use Case (`lib/features/auth/domain/usecases/login_usecase.dart`)
```dart
import 'package:dartz/dartz.dart';
import '../../../../core/error/failures.dart';
import '../../../../core/usecase/usecase.dart';
import '../entities/user_entity.dart';
import '../repositories/auth_repository.dart';

class LoginParams {
  final String email;
  final String password;

  const LoginParams({required this.email, required this.password});
}

class LoginUseCase implements UseCase<UserEntity, LoginParams> {
  final AuthRepository repository;

  LoginUseCase(this.repository);

  @override
  Future<Either<Failure, UserEntity>> call(LoginParams params) {
    return repository.login(
      email: params.email,
      password: params.password,
    );
  }
}
```

---

### 2. Data Layer (Data Sources & Mappers)

#### A. DTO Model (`lib/features/auth/data/models/user_model.dart`)
```dart
import '../../domain/entities/user_entity.dart';

class UserModel extends UserEntity {
  const UserModel({
    required super.id,
    required super.email,
    required super.name,
  });

  factory UserModel.fromJson(Map<String, dynamic> json) {
    return UserModel(
      id: json['id'] as String,
      email: json['email'] as String,
      name: json['name'] as String,
    );
  }

  Map<String, dynamic> toJson() {
    return {
      'id': id,
      'email': email,
      'name': name,
    };
  }

  // Conversion method to pure Domain Entity
  UserEntity toEntity() => UserEntity(id: id, email: email, name: name);
}
```

#### B. Data Source (`lib/features/auth/data/datasources/auth_remote_data_source.dart`)
```dart
abstract class AuthRemoteDataSource {
  Future<UserModel> login(String email, String password);
}

class AuthRemoteDataSourceImpl implements AuthRemoteDataSource {
  final HttpClient _client;

  AuthRemoteDataSourceImpl(this._client);

  @override
  Future<UserModel> login(String email, String password) async {
    final response = await _client.post('/auth/login', data: {
      'email': email,
      'password': password,
    });
    return UserModel.fromJson(response.data);
  }
}
```

#### C. Repository Implementation (`lib/features/auth/data/repositories/auth_repository_impl.dart`)
```dart
import 'package:dartz/dartz.dart';
import '../../../../core/error/exceptions.dart';
import '../../../../core/error/failures.dart';
import '../../domain/entities/user_entity.dart';
import '../../domain/repositories/auth_repository.dart';
import '../datasources/auth_remote_data_source.dart';

class AuthRepositoryImpl implements AuthRepository {
  final AuthRemoteDataSource remoteDataSource;

  AuthRepositoryImpl({required this.remoteDataSource});

  @override
  Future<Either<Failure, UserEntity>> login({
    required String email,
    required String password,
  }) async {
    try {
      final userModel = await remoteDataSource.login(email, password);
      return Right(userModel.toEntity());
    } on ServerException catch (e) {
      return Left(ServerFailure(e.message));
    } on NetworkException {
      return Left(NetworkFailure('No internet connection'));
    }
  }
}
```

---

### 3. Presentation Layer (State & UI)

#### B. State Controller / Cubit (`lib/features/auth/presentation/bloc/auth_cubit.dart`)
```dart
import 'package:flutter_bloc/flutter_bloc.dart';
import '../../domain/usecases/login_usecase.dart';
import 'auth_state.dart';

class AuthCubit extends Cubit<AuthState> {
  final LoginUseCase _loginUseCase;

  AuthCubit(this._loginUseCase) : super(AuthInitial());

  Future<void> login(String email, String password) async {
    emit(AuthLoading());
    final result = await _loginUseCase(LoginParams(email: email, password: password));
    result.fold(
      (failure) => emit(AuthError(failure.message)),
      (user) => emit(AuthAuthenticated(user)),
    );
  }
}
```

---

## Best Practices & Anti-Patterns

### ✅ Best Practices
1. **Strict Layers**: Data sources throw `Exception`s; Repositories catch them and return domain `Failure`s.
2. **DTO $\leftrightarrow$ Entity separation**: Keep JSON annotations (`@JsonKey`, `fromJson`) in DTO Models, never on Domain Entities.
3. **Use Cases handle business logic**: Place validations or orchestration combining multiple repositories inside Use Cases.
4. **Dependency Injection**: Use `get_it` or `Provider` to register contracts (`AuthRepository`) mapped to implementations (`AuthRepositoryImpl`).

### ❌ Anti-Patterns
* **Leaking Flutter into Domain**: Importing `BuildContext`, `Color`, or `Widget` inside a UseCase or Entity.
* **Bypassing Use Cases**: Calling Repositories directly from UI Widgets or ViewModels in complex applications.
* **Exposing DTOs to UI**: Returning raw API models with JSON logic straight to widgets.
* **Mutating Entities directly**: Entities should be immutable (using `const` constructors and `Equatable`).
