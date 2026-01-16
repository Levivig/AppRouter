# AppRouter

A generic, reusable navigation router for SwiftUI applications. Supports both simple single-stack navigation and complex tab-based navigation with independent navigation stacks and sheet presentation.

## Features

- 🎯 **Simple Router** - `SimpleRouter` for single NavigationStack apps
- 🏷️ **Tab Router** - `Router` for tab-based apps with independent navigation per tab
- 📄 **Sheet Management** - Built-in sheet presentation and dismissal
- 🔄 **SwiftUI Integration** - Uses `@Observable` for reactive state updates
- 🧵 **Thread Safe** - `@MainActor` implementation ensures UI safety
- 📱 **iOS 17+ Ready** - Built for modern SwiftUI patterns
- 🔗 **URL Deep Linking** - Navigate via URLs with automatic parameter parsing
- 🛤️ **Contextual Routing** - Same path components create different destinations based on context (e.g., `/users/detail` vs `/posts/detail`)

## Installation

### Swift Package Manager

Add this package to your project:

```swift
dependencies: [
    .package(url: "https://github.com/Levivig/AppRouter.git", from: "1.0.0")
]
```

## Quick Start

AppRouter provides two routers depending on your app's navigation needs:

- **`SimpleRouter`** - For apps with a single NavigationStack
- **`Router`** - For apps with tab-based navigation

### Option 1: Simple Navigation (Single Stack)

Perfect for apps that don't use tabs and just need a single navigation stack with sheet support.

```swift
import SwiftUI
import AppRouter

// 1. Define your destination and sheet types
enum Destination: DestinationType {
    case detail(id: String)
    case settings
    case profile(userId: String)
}

enum Sheet: SheetType {
    case compose
    case settings
    
    var id: Int { hashValue }
}

// 2. Use SimpleRouter
struct ContentView: View {
    @State private var router = SimpleRouter<Destination, Sheet>()
    
    var body: some View {
        NavigationStack(path: $router.path) {
            HomeView()
                .navigationDestination(for: Destination.self) { destination in
                    destinationView(for: destination)
                }
        }
        .sheet(item: $router.presentedSheet) { sheet in
            sheetView(for: sheet)
        }
        .environment(router)
    }
    
    @ViewBuilder
    private func destinationView(for destination: Destination) -> some View {
        switch destination {
        case .detail(let id):
            DetailView(id: id)
        case .settings:
            SettingsView()
        case .profile(let userId):
            ProfileView(userId: userId)
        }
    }
    
    @ViewBuilder
    private func sheetView(for sheet: Sheet) -> some View {
        switch sheet {
        case .compose:
            ComposeView()
        case .settings:
            SettingsSheet()
        }
    }
}

// 3. Navigate from anywhere in your app
struct HomeView: View {
    @Environment(SimpleRouter<Destination, Sheet>.self) private var router
    
    var body: some View {
        VStack {
            Button("Go to Detail") {
                router.navigateTo(.detail(id: "123"))
            }
            
            Button("Show Compose Sheet") {
                router.presentSheet(.compose)
            }
        }
    }
}
```

### Option 2: Tab-Based Navigation

For apps that use TabView with independent navigation stacks per tab.

#### 1. Define Your Tab Type

```swift
import AppRouter

enum AppTab: String, TabType, CaseIterable {
    case home, profile, settings
    
    var id: String { rawValue }
    
    var icon: String {
        switch self {
        case .home: return "house"
        case .profile: return "person"  
        case .settings: return "gear"
        }
    }
}
```

#### 2. Define Destination and Sheet Types

```swift
enum Destination: DestinationType {
    case detail(id: String)
    case list
    case profile(userId: String)
}

enum Sheet: SheetType {
    case settings
    case profile
    case compose
    
    var id: Int { hashValue }
}
```

#### 3. Use the Tab Router

```swift
import SwiftUI
import AppRouter

struct ContentView: View {
    @State private var router = Router<AppTab, Destination, Sheet>(initialTab: .home)
    
    var body: some View {
        TabView(selection: $router.selectedTab) {
            ForEach(AppTab.allCases) { tab in
                NavigationStack(path: $router[tab]) {
                    HomeView()
                        .navigationDestination(for: Destination.self) { destination in
                            destinationView(for: destination)
                        }
                }
                .tabItem {
                    Label(tab.rawValue.capitalized, systemImage: tab.icon)
                }
                .tag(tab)
            }
        }
        .sheet(item: $router.presentedSheet) { sheet in
            sheetView(for: sheet)
        }
    }
    
    @ViewBuilder
    private func destinationView(for destination: Destination) -> some View {
        switch destination {
        case .detail(let id):
            DetailView(id: id)
        case .list:
            ListView()
        case .profile(let userId):
            ProfileView(userId: userId)
        }
    }
    
    @ViewBuilder  
    private func sheetView(for sheet: Sheet) -> some View {
        switch sheet {
        case .settings:
            SettingsView()
        case .profile:
            ProfileSheet()
        case .compose:
            ComposeView()
        }
    }
}
```

## API Reference

### SimpleRouter

For single NavigationStack apps:

```swift
@Observable @MainActor
public final class SimpleRouter<Destination: DestinationType, Sheet: SheetType>
```

#### Properties
- `path: [Destination]` - Navigation path
- `presentedSheet: Sheet?` - Currently presented sheet

#### Methods
- `navigateTo(_:)` - Navigate to a destination
- `popNavigation()` - Pop last destination from stack
- `popToRoot()` - Clear navigation stack
- `presentSheet(_:)` - Present a sheet
- `dismissSheet()` - Dismiss current sheet
- `navigate(to:)` - Navigate using a URL or URL string

### Router

For tab-based apps with independent navigation per tab:

```swift
@Observable @MainActor
public final class Router<Tab: TabType, Destination: DestinationType, Sheet: SheetType>
```

#### Properties
- `selectedTab: Tab` - Currently selected tab
- `presentedSheet: Sheet?` - Currently presented sheet
- `selectedTabPath: [Destination]` - Navigation path for current tab

#### Methods
- `navigateTo(_:for:)` - Navigate to a destination  
- `popNavigation(for:)` - Pop last destination from stack
- `popToRoot(for:)` - Clear navigation stack for tab
- `presentSheet(_:)` - Present a sheet
- `dismissSheet()` - Dismiss current sheet
- `navigate(to:)` - Navigate using a URL or URL string
- `push(to:)` - Push URL destinations onto the current stack

### Protocols

#### DestinationType  
```swift
public protocol DestinationType: Hashable {
    /// Creates a destination from a URL path component with full path context and query parameters
    static func from(path: String, fullPath: [String], parameters: [String: String]) -> Self?
}
```

#### SheetType
```swift
public protocol SheetType: Hashable, Identifiable {}
```

#### TabType
```swift
public protocol TabType: Hashable, CaseIterable, Identifiable, Sendable {
    var icon: String { get }
}
```
*Only needed for tab-based navigation*

## URL Deep Linking

AppRouter supports URL-based deep linking, allowing you to navigate to specific screens in your app using URLs. This works with both `Router` and `SimpleRouter`.

### Setting Up Deep Linking

#### 1. Implement URL Parsing in Your Destination Type

```swift
enum Destination: DestinationType {
    case detail(id: String)
    case list
    case profile(userId: String)
    case userDetail(id: String)     // Different from generic detail
    case postDetail(id: String)     // Different from generic detail
    
    // Required for URL deep linking with contextual routing support
    static func from(path: String, fullPath: [String], parameters: [String: String]) -> Destination? {
        // Find current position in the path for context
        guard let currentIndex = fullPath.firstIndex(of: path) else {
            return nil
        }
        
        let previousComponent = currentIndex > 0 ? fullPath[currentIndex - 1] : nil
        
        switch (previousComponent, path) {
        // Contextual routing - same path component, different destinations
        case ("users", "detail"):
            let id = parameters["id"] ?? "unknown"
            return .userDetail(id: id)
        case ("posts", "detail"):
            let id = parameters["id"] ?? "unknown"
            return .postDetail(id: id)
        // Standard routing
        case (_, "detail"):
            let id = parameters["id"] ?? "default"
            return .detail(id: id)
        case (_, "list"):
            return .list
        case (_, "profile"):
            let userId = parameters["userId"] ?? "unknown"
            return .profile(userId: userId)
        case (nil, "users"), (nil, "posts"):
            return nil // These are path segments, not destinations
        default:
            return nil
        }
    }
}
```

#### 2. Handle Incoming URLs with SwiftUI's .openURL

```swift
struct ContentView: View {
    @State private var router = SimpleRouter<Destination, Sheet>()
    
    var body: some View {
        NavigationStack(path: $router.path) {
            HomeView()
                .navigationDestination(for: Destination.self) { destination in
                    destinationView(for: destination)
                }
        }
        .environment(router)
        .onOpenURL { url in
            // Handle deep links
            router.navigate(to: url)
        }
    }
}
```

#### 3. Configure Your App's URL Scheme

Add your URL scheme to your app's `Info.plist`:

```xml
<key>CFBundleURLTypes</key>
<array>
    <dict>
        <key>CFBundleURLName</key>
        <string>myapp.deeplink</string>
        <key>CFBundleURLSchemes</key>
        <array>
            <string>myapp</string>
        </array>
    </dict>
</array>
```

### URL Format

URLs follow this format: `scheme://destination1/destination2?param1=value1&param2=value2`

#### Examples

```swift
// Navigate to a single destination
"myapp://list"

// Navigate to a destination with parameters
"myapp://detail?id=123"

// Navigate through multiple destinations (navigation stack)
"myapp://list/detail?id=456"

// Contextual routing - same path, different destinations
"myapp://users/detail?id=user123"  // → userDetail(id: "user123")
"myapp://posts/detail?id=post456"  // → postDetail(id: "post456")
"myapp://detail?id=generic789"     // → detail(id: "generic789")

// Complex navigation with context
"myapp://list/users/detail?id=john&tab=profile"
```

### Contextual Routing

AppRouter supports **contextual routing** where the same path component can create different destinations based on the preceding path. This mirrors web routing patterns where `/users/detail` and `/posts/detail` are different routes.

```swift
// Different destinations from the same "detail" path:
"myapp://users/detail?id=123"  // Creates userDetail(id: "123")
"myapp://posts/detail?id=456"  // Creates postDetail(id: "456") 
"myapp://detail?id=789"        // Creates detail(id: "789")
```

This enables more natural URL structures that match REST API patterns and web conventions.

### Using Deep Links Programmatically

#### Create URLs for Sharing

```swift
// Using the URL helper extension
let url = URL.deepLink(
    scheme: "myapp",
    destinations: [Destination.detail(id: "123")],
    parameters: ["source": "share"]
)

// Share the URL
if let url = url {
    let activityVC = UIActivityViewController(activityItems: [url], applicationActivities: nil)
    // Present activity controller
}
```

#### Navigate Programmatically

```swift
struct HomeView: View {
    @Environment(SimpleRouter<Destination, Sheet>.self) private var router
    
    var body: some View {
        VStack {
            Button("Deep Link to Detail") {
                router.navigate(to: "myapp://detail?id=456")
            }
            
            Button("Navigate with URL") {
                let url = URL(string: "myapp://list/detail?id=789")!
                router.navigate(to: url)
            }

            Button("Push URL Destinations") {
                let url = URL(string: "myapp://list/detail?id=789")!
                router.push(to: url)
            }
        }
    }
}
```

### Tab-Based Apps

For tab-based apps using `Router`, deep links navigate to the **currently selected tab**:

```swift
struct TabContentView: View {
    @State private var router = Router<AppTab, Destination, Sheet>(initialTab: .home)
    
    var body: some View {
        TabView(selection: $router.selectedTab) {
            // ... tab content
        }
        .onOpenURL { url in
            // This will navigate in the currently active tab
            router.navigate(to: url)
        }
    }
}

You can also push URL destinations on top of the current stack:

```swift
router.push(to: "myapp://list/detail?id=789")
```
```

### Advanced URL Handling

#### Custom URL Processing

```swift
// Handle URLs manually for custom logic
@Environment(AppRouter.self) private var router

func handleCustomURL(_ url: URL) {
    // Add custom pre-processing
    guard url.scheme == "myapp" else { return }
    
    // Log analytics
    Analytics.track("deep_link_opened", parameters: ["url": url.absoluteString])
    
    // Navigate using the router
    let success = router.navigate(to: url)
    
    if !success {
        // Handle failed navigation
        showErrorAlert("Invalid deep link")
    }
}
```

#### URL Validation

```swift
extension Destination {
    static func from(path: String, fullPath: [String], parameters: [String: String]) -> Destination? {
        guard let currentIndex = fullPath.firstIndex(of: path) else {
            return nil
        }
        
        let previousComponent = currentIndex > 0 ? fullPath[currentIndex - 1] : nil
        
        switch (previousComponent, path) {
        case ("users", "detail"):
            // Validate required parameters for user detail
            guard let id = parameters["id"], !id.isEmpty else {
                return nil
            }
            return .userDetail(id: id)
        case (_, "profile"):
            guard let userId = parameters["userId"], 
                  userId.count >= 3 else {
                return nil
            }
            return .profile(userId: userId)
        default:
            return nil
        }
    }
}
```

### Testing Deep Links

#### iOS Simulator
```bash
# Open deep link in simulator
xcrun simctl openurl booted "myapp://detail?id=123"

# Test contextual routing
xcrun simctl openurl booted "myapp://users/detail?id=user123"
xcrun simctl openurl booted "myapp://posts/detail?id=post456"
```

#### Xcode Debugging
1. Edit your scheme
2. Go to "Run" → "Arguments" → "Arguments Passed On Launch"
3. Add: `-FIRDebugEnabled`
4. Go to "Options" → "URL Arguments"
5. Add your test URL: `myapp://detail?id=test`

## Examples

### Type Aliases for Cleaner Syntax

To avoid verbose generic syntax throughout your app, create a type alias:

```swift
// Define once in your app
typealias AppRouter = Router<AppTab, Destination, Sheet>
typealias AppSimpleRouter = SimpleRouter<Destination, Sheet>

// Then use the cleaner syntax everywhere
@Environment(AppRouter.self) private var router
@State private var router = AppRouter(initialTab: .home)
```

### Navigate Programmatically

```swift
struct HomeView: View {
    @Environment(AppRouter.self) private var router
    
    var body: some View {
        VStack {
            Button("Go to Detail") {
                router.navigateTo(.detail(id: "123"))
            }
            
            Button("Show Settings") {
                router.presentSheet(.settings)
            }
            
            Button("Go to Profile Tab") {
                router.selectedTab = .profile
                router.navigateTo(.profile(userId: "user123"), for: .profile)
            }
        }
    }
}
```

### Environment Integration

```swift
struct App: View {
    @State private var router = Router<AppTab, Destination, Sheet>(initialTab: .home)
    
    var body: some View {
        ContentView()
            .environment(router)
            .environment(\.currentTab, router.selectedTab)
    }
}
```

## Requirements

- iOS 17.0+
- macOS 14.0+
- tvOS 17.0+  
- watchOS 10.0+
- Swift 5.9+

## License

MIT License - see LICENSE file for details
