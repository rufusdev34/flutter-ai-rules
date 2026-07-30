---
name: solid-principles
description: "Use when designing classes, refactoring tight coupling, implementing interfaces, applying Single Responsibility, Open-Closed, Liskov Substitution, Interface Segregation, or Dependency Inversion principles in Dart and Flutter."
---

# SOLID Principles in Dart & Flutter

This skill provides comprehensive guidance for applying the **SOLID** design principles in Dart and Flutter applications to write maintainable, testable, and scalable object-oriented code.

---

## When to Use

Use this skill when:
* Designing new classes, services, or modules in Dart/Flutter.
* Refactoring legacy code with tight coupling or monolithic classes.
* Establishing interface contracts and abstract classes.
* Writing unit tests that require mocking external dependencies.
* Evaluating code architecture during reviews.

---

## The 5 SOLID Principles Summary

| Principle | Name | Core Idea |
|---|---|---|
| **S** | Single Responsibility (SRP) | A class should have one, and only one, reason to change. |
| **O** | Open / Closed (OCP) | Code should be open for extension, but closed for modification. |
| **L** | Liskov Substitution (LSP) | Derived classes must be substitutable for their base classes. |
| **I** | Interface Segregation (ISP) | Clients should not be forced to depend on methods they do not use. |
| **D** | Dependency Inversion (DIP) | Depend on abstractions, not concrete implementations. |

---

## 1. Single Responsibility Principle (SRP)

> **"A class should have only one reason to change."**

### Guidelines:
- Separate UI, business logic, data parsing, and network communication into distinct classes.
- A widget should only render UI; a controller/cubit should only handle business state; a repository should only orchestrate data.

### ❌ Bad (Violates SRP)
```dart
// Bad: UserProfileManager manages state, parses JSON, calls HTTP, and saves to storage
class UserProfileManager {
  Future<void> fetchAndSaveUser(String id) async {
    final response = await http.get(Uri.parse('https://api.com/users/$id'));
    final json = jsonDecode(response.body);
    final user = User(name: json['name'], email: json['email']);
    final prefs = await SharedPreferences.getInstance();
    await prefs.setString('user_$id', jsonEncode({'name': user.name}));
  }
}
```

### ✅ Good (Follows SRP)
```dart
// Dedicated API Service for HTTP fetching
class UserApiClient {
  final http.Client _client;
  UserApiClient(this._client);

  Future<Map<String, dynamic>> getUserJson(String id) async {
    final response = await _client.get(Uri.parse('https://api.com/users/$id'));
    return jsonDecode(response.body) as Map<String, dynamic>;
  }
}

// Dedicated Local Cache Service
class UserLocalCache {
  Future<void> saveUser(String id, String jsonString) async {
    final prefs = await SharedPreferences.getInstance();
    await prefs.setString('user_$id', jsonString);
  }
}

// Repository orchestrating operations with single purpose
class UserRepository {
  final UserApiClient _apiClient;
  final UserLocalCache _localCache;

  UserRepository({required UserApiClient apiClient, required UserLocalCache localCache})
      : _apiClient = apiClient,
        _localCache = localCache;

  Future<User> getUser(String id) async {
    final json = await _apiClient.getUserJson(id);
    final user = User.fromJson(json);
    await _localCache.saveUser(id, jsonEncode(json));
    return user;
  }
}
```

---

## 2. Open / Closed Principle (OCP)

> **"Software entities should be open for extension, but closed for modification."**

### Guidelines:
- Add new features by adding new classes, strategies, or mixins—never by modifying existing `switch` / `if-else` chains in core domain logic.
- Use polymorphism, abstract classes, strategy patterns, or Dart extensions.

### ❌ Bad (Violates OCP)
```dart
// Modifying this method every time a new payment type is added violates OCP
class PaymentProcessor {
  void processPayment(String type, double amount) {
    if (type == 'credit_card') {
      // Process credit card
    } else if (type == 'paypal') {
      // Process paypal
    } else if (type == 'apple_pay') { // Added later by modifying class
      // Process apple pay
    }
  }
}
```

### ✅ Good (Follows OCP)
```dart
abstract class PaymentStrategy {
  Future<bool> process(double amount);
}

class CreditCardPayment implements PaymentStrategy {
  @override
  Future<bool> process(double amount) async {
    // Process credit card payment logic
    return true;
  }
}

class PaypalPayment implements PaymentStrategy {
  @override
  Future<bool> process(double amount) async {
    // Process PayPal payment logic
    return true;
  }
}

// Open for new payment methods without modifying PaymentProcessor
class PaymentProcessor {
  Future<bool> executePayment(PaymentStrategy strategy, double amount) {
    return strategy.process(amount);
  }
}
```

---

## 3. Liskov Substitution Principle (LSP)

> **"Subtypes must be substitutable for their base types without altering program correctness."**

### Guidelines:
- Overridden methods in subclasses must satisfy the contract of the superclass.
- Never throw `UnimplementedError` or narrow precondition parameters in subclasses.
- Ensure return types and async behavior are consistent with base expectations.

### ❌ Bad (Violates LSP)
```dart
abstract class FileStorage {
  Future<void> writeFile(String path, String data);
  Future<String> readFile(String path);
}

class ReadOnlyCloudStorage implements FileStorage {
  @override
  Future<String> readFile(String path) async => "cloud data";

  @override
  Future<void> writeFile(String path, String data) {
    // Violates LSP: Subtype breaks supertype contract by throwing runtime error
    throw UnimplementedError('Read-only storage cannot write files');
  }
}
```

### ✅ Good (Follows LSP)
```dart
// Separate capabilities so subtypes don't break contracts
abstract class ReadableStorage {
  Future<String> readFile(String path);
}

abstract class WritableStorage {
  Future<void> writeFile(String path, String data);
}

class ReadOnlyCloudStorage implements ReadableStorage {
  @override
  Future<String> readFile(String path) async => "cloud data";
}

class LocalLocalStorage implements ReadableStorage, WritableStorage {
  @override
  Future<String> readFile(String path) async => "local data";

  @override
  Future<void> writeFile(String path, String data) async {
    // Save file locally
  }
}
```

---

## 4. Interface Segregation Principle (ISP)

> **"Clients should not be forced to depend on methods they do not use."**

### Guidelines:
- Prefer multiple specific interfaces over one bloated interface.
- In Dart, every class implicitly defines an interface. Keep `abstract class` declarations focused and minimal.

### ❌ Bad (Violates ISP)
```dart
// Monolithic interface forcing all clients to implement irrelevant methods
abstract class MultiFunctionDevice {
  void printDocument();
  void scanDocument();
  void faxDocument();
}

class BasicPrinter implements MultiFunctionDevice {
  @override
  void printDocument() => print('Printing...');

  @override
  void scanDocument() => throw UnimplementedError(); // Irrelevant to BasicPrinter

  @override
  void faxDocument() => throw UnimplementedError();  // Irrelevant to BasicPrinter
}
```

### ✅ Good (Follows ISP)
```dart
abstract class Printer {
  void printDocument();
}

abstract class Scanner {
  void scanDocument();
}

abstract class Fax {
  void faxDocument();
}

class BasicPrinter implements Printer {
  @override
  void printDocument() => print('Printing...');
}

class SmartCopier implements Printer, Scanner {
  @override
  void printDocument() => print('Printing...');

  @override
  void scanDocument() => print('Scanning...');
}
```

---

## 5. Dependency Inversion Principle (DIP)

> **"High-level modules should not depend on low-level modules. Both should depend on abstractions."**

### Guidelines:
- High-level business logic (Use Cases, View Models, Repositories) must depend on abstract classes / interfaces, not concrete implementations (`Dio`, `http`, `SharedPreferences`).
- Pass dependencies in constructors (Constructor Injection).

### ❌ Bad (Violates DIP)
```dart
class AuthenticationRepository {
  // Hardcoded dependency on concrete Firebase SDK
  final FirebaseAuth _firebaseAuth = FirebaseAuth.instance;

  Future<void> login(String email, String password) async {
    await _firebaseAuth.signInWithEmailAndPassword(email: email, password: password);
  }
}
```

### ✅ Good (Follows DIP)
```dart
// Abstraction owned by high-level layer
abstract class AuthRemoteDataSource {
  Future<UserDto> signIn(String email, String password);
}

// Concrete low-level implementation
class FirebaseAuthDataSource implements AuthRemoteDataSource {
  final FirebaseAuth _firebaseAuth;
  FirebaseAuthDataSource(this._firebaseAuth);

  @override
  Future<UserDto> signIn(String email, String password) async {
    final credential = await _firebaseAuth.signInWithEmailAndPassword(
      email: email,
      password: password,
    );
    return UserDto.fromFirebaseUser(credential.user!);
  }
}

// High-level module depending only on abstraction
class AuthenticationRepository {
  final AuthRemoteDataSource _remoteDataSource;

  AuthenticationRepository(this._remoteDataSource);

  Future<User> login(String email, String password) async {
    final dto = await _remoteDataSource.signIn(email, password);
    return dto.toDomain();
  }
}
```

---

## SOLID Rules Checklist for Code Reviews

- [ ] Does every class have a single, clearly stated responsibility?
- [ ] Can new variations of behavior be added via new classes without editing existing `if`/`switch` blocks?
- [ ] Can every subclass replace its parent class without breaking execution or throwing unexpected runtime exceptions?
- [ ] Are interfaces minimal and cohesive, containing only required methods for their consumers?
- [ ] Do high-level domain classes depend strictly on abstract interfaces rather than concrete third-party SDKs?
