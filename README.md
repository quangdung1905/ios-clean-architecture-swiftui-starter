# iOS Clean Architecture Starter

A production-ready iOS starter project demonstrating **Clean Architecture** with **MVVM** using **SwiftUI**. This project serves as a reference implementation for scalable iOS applications.

## 📋 Table of Contents

- [Project Overview](#project-overview)
- [Tech Stack](#tech-stack)
- [Architecture](#architecture)
- [Navigation Architecture](#navigation-architecture)
- [Deep Linking](#deep-linking)
- [Project Structure](#project-structure)
- [Dependency Flow](#dependency-flow)
- [Getting Started](#getting-started)
- [Architecture Decisions](#architecture-decisions)
- [Testing](#testing)
- [Scaling for Production](#scaling-for-production)

## 🎯 Project Overview

This project demonstrates senior-level iOS architecture patterns and best practices:

- **Clean Architecture**: Strict separation of concerns with clear dependency rules
- **MVVM**: Model-View-ViewModel pattern for SwiftUI
- **Protocol-Based Dependency Injection**: Testable and maintainable code
- **Advanced Navigation**: Generic routing system with deep linking support
- **Async/Await**: Modern Swift concurrency
- **Pure Native**: No third-party dependencies

The goal is to showcase **architecture, code quality, scalability, and best practices**, not UI polish.

## 🛠 Tech Stack

- **Language**: Swift 5+
- **UI Framework**: SwiftUI
- **Concurrency**: async/await
- **Architecture**: Clean Architecture + MVVM
- **Dependency Injection**: Protocol-based DI
- **Networking**: URLSession
- **State Management**: ObservableObject
- **Testing**: XCTest
- **Minimum iOS Version**: iOS 16.0+ (required for NavigationStack)

## 🏗 Architecture

### Clean Architecture Layers

The project follows Clean Architecture principles with strict dependency rules:

```
┌─────────────────────────────────────────┐
│         Presentation Layer              │
│  ┌──────────┐      ┌──────────────┐   │
│  │   View   │──────│  ViewModel   │   │
│  └──────────┘      └──────────────┘   │
└─────────────────────────────────────────┘
                  ↓ depends on
┌─────────────────────────────────────────┐
│            Domain Layer                  │
│  ┌──────────┐  ┌──────────┐  ┌──────┐  │
│  │ Entities │  │ UseCases │  │ Repo │  │
│  │          │  │          │  │ Proto│  │
│  └──────────┘  └──────────┘  └──────┘  │
└─────────────────────────────────────────┘
                  ↑ implements
┌─────────────────────────────────────────┐
│            Data Layer                    │
│  ┌──────────┐  ┌──────────┐  ┌──────┐  │
│  │   DTOs   │  │   Repo   │  │  API │  │
│  │          │  │   Impl   │  │Client│  │
│  └──────────┘  └──────────┘  └──────┘  │
└─────────────────────────────────────────┘
```

### Dependency Rules

1. **Presentation** depends on **Domain** only
2. **Data** depends on **Domain** only
3. **Domain** has **no dependencies** on outer layers
4. **Domain** must not import SwiftUI or any UI frameworks

### MVVM Pattern

- **View**: SwiftUI views (declarative UI)
- **ViewModel**: ObservableObject managing state and coordinating with use cases
- **Model**: Domain entities (pure business models)

## 🧭 Navigation Architecture

### AppRoute: Single Source of Truth

All navigation in the app goes through `AppRoute` enum:

```swift
enum AppRoute: Hashable {
    case userList
    case userDetail(userId: Int)
    case userPosts(userId: Int)
    case postDetail(postId: Int)
}
```

**Key Principles:**
- ✅ No hardcoded `NavigationLink(destination:)` with views
- ✅ All routes are serializable for deep linking
- ✅ Type-safe navigation with associated values

### NavigationService

Centralized navigation management through protocol-based service:

```swift
protocol NavigationService {
    var path: Binding<[AppRoute]> { get }
    func navigate(to route: AppRoute)
    func setRoutes(_ routes: [AppRoute])
    func pop()
    func popToRoot()
}
```

**Benefits:**
- Decouples navigation logic from views
- ViewModels call `NavigationService`, not views
- Easy to test and mock
- Supports programmatic navigation

### NavigationStack Integration

Uses SwiftUI's `NavigationStack` with route-based navigation:

```swift
NavigationStack(path: navigationService.path) {
    routeResolver.resolve(route: .userList)
        .navigationDestination(for: AppRoute.self) { route in
            routeResolver.resolve(route: route)
        }
}
```

### RouteResolver

Maps `AppRoute` to SwiftUI views in a single location:

```swift
struct RouteResolver {
    let container: AppContainer
    
    @ViewBuilder
    func resolve(route: AppRoute) -> some View {
        switch route {
        case .userList:
            UserListView(viewModel: container.makeUserListViewModel())
        case .userDetail(let userId):
            UserDetailView(viewModel: container.makeUserDetailViewModel(userId: userId))
        // ...
        }
    }
}
```

## 🔗 Deep Linking

### URL Scheme

The app supports custom URL scheme: `myapp://`

### Supported Deep Links

- `myapp://users` - Navigate to user list
- `myapp://users/{id}` - Navigate to user detail
- `myapp://users/{id}/posts` - Navigate to user posts
- `myapp://posts/{postId}` - Navigate to post detail

### DeepLinkHandler

Parses URLs and converts them to navigation routes:

```swift
protocol DeepLinkHandler {
    func parse(url: URL) -> [AppRoute]
}
```

**Features:**
- Cold-start deep linking (app launch)
- Hot deep linking (app already running)
- Invalid URL handling (returns empty array)

### AppCoordinator

Manages app initialization and deep link handling:

```swift
final class AppCoordinator {
    func start(url: URL?)  // Cold start
    func handleDeepLink(url: URL)  // Hot deep link
}
```

**Behavior:**
- No URL → starts at `.userList`
- Valid URL → resolves routes and sets navigation path
- Invalid URL → shows default route

### Example Usage

```swift
// In App file
.onOpenURL { url in
    Task { @MainActor in
        container.appCoordinator.handleDeepLink(url: url)
    }
}
```

## 📁 Project Structure

```
ios-clean-architecture-swiftui/
├── Domain/                          # Core business logic (no dependencies)
│   ├── Entities/
│   │   ├── User.swift              # Domain entity
│   │   └── Post.swift              # Domain entity
│   ├── Repositories/
│   │   ├── UserRepository.swift    # Repository protocol
│   │   └── PostRepository.swift    # Repository protocol
│   ├── UseCases/
│   │   ├── FetchUsersUseCase.swift # Business logic
│   │   ├── FetchUserUseCase.swift
│   │   ├── FetchPostsUseCase.swift
│   │   └── FetchPostUseCase.swift
│   └── Errors/
│       └── AppError.swift          # Domain errors
│
├── Data/                            # Data access layer
│   ├── Network/
│   │   └── APIClient.swift         # Generic API client
│   ├── DTOs/
│   │   ├── UserDTO.swift           # Data Transfer Objects
│   │   └── PostDTO.swift
│   └── Repositories/
│       ├── UserRepositoryImpl.swift # Repository implementation
│       └── PostRepositoryImpl.swift
│
├── Presentation/                    # UI layer (feature-based)
│   └── Features/
│       ├── Users/
│       │   ├── UserListView.swift
│       │   ├── UserListViewModel.swift
│       │   ├── UserDetailView.swift
│       │   ├── UserDetailViewModel.swift
│       │   ├── UserPostsView.swift
│       │   └── UserPostsViewModel.swift
│       ├── Posts/
│       │   ├── PostDetailView.swift
│       │   └── PostDetailViewModel.swift
│       └── RouteResolver.swift     # Route to view mapping
│
├── Core/                            # Shared infrastructure
│   ├── Navigation/
│   │   ├── AppRoute.swift          # Navigation routes
│   │   ├── NavigationService.swift # Navigation protocol & impl
│   │   └── DeepLinkHandler.swift  # Deep link parsing
│   ├── Coordinator/
│   │   └── AppCoordinator.swift    # App initialization
│   └── DependencyInjection/
│       └── AppContainer.swift      # DI container
│
└── Tests/                           # Unit tests
    ├── Mocks/
    │   ├── MockUserRepository.swift
    │   └── MockFetchUsersUseCase.swift
    ├── Domain/
    │   └── UseCases/
    │       └── FetchUsersUseCaseTests.swift
    ├── Presentation/
    │   └── ViewModels/
    │       └── UserListViewModelTests.swift
    └── Core/
        └── Navigation/
            ├── DeepLinkHandlerTests.swift
            └── NavigationServiceTests.swift
```

## 🔄 Dependency Flow

### Data Flow Example (Fetching Users)

```
1. UserListView
   ↓ calls
2. UserListViewModel.loadUsers()
   ↓ calls
3. FetchUsersUseCase.execute()
   ↓ calls
4. UserRepository.fetchUsers()
   ↓ implemented by
5. UserRepositoryImpl.fetchUsers()
   ↓ calls
6. APIClient.get()
   ↓ returns DTOs
7. UserDTO.toDomain() → User
   ↓ returns
8. [User] flows back up through layers
```

### Dependency Injection Flow

```
AppContainer
  ├── APIClient
  ├── UserRepositoryImpl (uses APIClient)
  ├── FetchUsersUseCaseImpl (uses UserRepository)
  └── UserListViewModel (uses FetchUsersUseCase)
```

## 🚀 Getting Started

### Prerequisites

- Xcode 15.0 or later
- iOS 16.0+ deployment target (required for NavigationStack)
- Swift 5.9+

### Running the Project

1. Open `ios-clean-architecture-swiftui.xcodeproj` in Xcode
2. Select a simulator or device
3. Build and run (⌘R)

The app will fetch users from [JSONPlaceholder](https://jsonplaceholder.typicode.com) and display them in a list.

### Running Tests

1. Press `⌘U` to run all tests
2. Or use the Test Navigator (⌘6) to run individual tests

## 🎓 Architecture Decisions

### Why Clean Architecture?

1. **Testability**: Each layer can be tested independently
2. **Independence**: Business logic is independent of frameworks
3. **Flexibility**: Easy to swap implementations (e.g., different data sources)
4. **Maintainability**: Clear separation of concerns
5. **Scalability**: Structure supports growth

### Why Protocol-Based DI?

- **Testability**: Easy to create mocks for testing
- **Flexibility**: Swap implementations without changing dependent code
- **No Singleton Abuse**: Dependencies are explicit and manageable
- **Compile-Time Safety**: Protocols ensure correct implementations

### Why No Third-Party Libraries?

- **Learning**: Understand how things work under the hood
- **Control**: Full control over implementation
- **Performance**: No external dependencies to manage
- **Production Ready**: Native solutions are often sufficient

### Why Async/Await?

- **Modern Swift**: Official concurrency model
- **Readability**: Cleaner than completion handlers
- **Error Handling**: Natural error propagation
- **Performance**: Efficient and safe

## 🧪 Testing

### Testing Strategy

1. **Unit Tests**: Test use cases and view models in isolation
2. **Mocking**: Use protocol-based mocks (no third-party frameworks)
3. **No Network Calls**: All tests use mocked dependencies

### Example Test Structure

```swift
// Test UseCase with mocked Repository
func testExecute_Success_ReturnsUsers() async throws {
    let mockRepository = MockUserRepository()
    let useCase = FetchUsersUseCaseImpl(userRepository: mockRepository)
    // ... test implementation
}

// Test ViewModel with mocked UseCase
func testLoadUsers_Success_UpdatesUsers() async {
    let mockUseCase = MockFetchUsersUseCase()
    let viewModel = UserListViewModel(fetchUsersUseCase: mockUseCase)
    // ... test implementation
}
```

## 📈 Scaling for Production

This architecture scales well for real-world applications:

### Adding New Features

1. **New Entity**: Add to `Domain/Entities/`
2. **New Use Case**: Add to `Domain/UseCases/`
3. **New Repository**: Add protocol to `Domain/Repositories/`, implementation to `Data/Repositories/`
4. **New Route**: Add case to `AppRoute` enum
5. **New View**: Add to `Presentation/Features/{FeatureName}/` with corresponding ViewModel
6. **Update RouteResolver**: Add route mapping in `resolve(route:)` method
7. **Update DeepLinkHandler**: Add URL parsing logic if needed
8. **Update DI Container**: Add factory methods for new ViewModels

### Extending Functionality

- **Caching**: Add caching layer in Data layer
- **Offline Support**: Implement local repository alongside remote
- **Authentication**: Add auth use cases and repository
- **Pagination**: Extend use cases and repositories
- **Error Recovery**: Enhance error handling in ViewModels

### Best Practices for Scaling

1. **Keep Use Cases Focused**: One use case = one responsibility
2. **Repository Pattern**: Abstract data sources (API, local DB, cache)
3. **Error Handling**: Consistent error propagation through layers
4. **State Management**: Keep ViewModels focused and testable
5. **Code Organization**: Group related files by feature when project grows

## 📝 Code Quality Principles

This project follows:

- **SOLID Principles**: Especially Single Responsibility and Dependency Inversion
- **Small, Focused Files**: Each file has a single, clear purpose
- **Clear Naming**: Self-documenting code
- **No Force Unwraps**: Safe optional handling
- **No Hard-coded Strings**: Constants for business logic strings
- **Protocol-Oriented**: Leverage Swift's protocol capabilities

## 🔍 Key Components Explained

### Domain Layer

- **Pure business logic** with no external dependencies
- **Framework-independent** (no SwiftUI, UIKit, Foundation networking)
- **Testable** in isolation

### Data Layer

- **Implements** Domain protocols
- **Handles** networking, DTOs, and mapping
- **Isolated** from presentation concerns

### Presentation Layer

- **Depends only** on Domain layer
- **Manages UI state** through ViewModels
- **Declarative UI** with SwiftUI

## 🤝 Contributing

This is a starter template. Feel free to:

- Extend with more features
- Add more comprehensive tests
- Implement additional use cases
- Enhance error handling
- Add documentation

## 📄 License

This project is provided as a reference implementation for educational purposes.

## 🙏 Acknowledgments

- Clean Architecture principles by Robert C. Martin
- JSONPlaceholder for providing a free API for testing

---

**Built with ❤️ following Clean Architecture principles**
