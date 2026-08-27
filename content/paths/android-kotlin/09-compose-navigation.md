---
title: "Compose Navigation"
weight: 9
---

# Compose Navigation

Navigation in Jetpack Compose is handled by the Navigation component, which manages the back stack, argument passing, deep links, and transitions between screens. The type-safe navigation API (introduced with Navigation 2.8+) uses serializable route objects instead of raw strings.

## Navigation Component Setup

Navigation in Compose centres on three elements: `NavController`, `NavHost`, and route definitions.

```kotlin
@Composable
fun MyApp() {
    val navController = rememberNavController()

    NavHost(
        navController = navController,
        startDestination = Home
    ) {
        composable<Home> {
            HomeScreen(
                onNavigateToProfile = { navController.navigate(Profile) },
                onNavigateToDetail = { id -> navController.navigate(Detail(id)) }
            )
        }
        composable<Profile> {
            ProfileScreen(onBack = { navController.popBackStack() })
        }
        composable<Detail> { backStackEntry ->
            val detail: Detail = backStackEntry.toRoute()
            DetailScreen(itemId = detail.id)
        }
    }
}
```

### NavController

The `NavController` manages the navigation back stack:

```kotlin
val navController = rememberNavController()

// Navigate to a destination
navController.navigate(Profile)

// Navigate and clear back stack
navController.navigate(Home) {
    popUpTo(Home) { inclusive = true }
}

// Pop back
navController.popBackStack()

// Navigate with single-top (avoid duplicate on back stack)
navController.navigate(Profile) {
    launchSingleTop = true
}
```

## Type-Safe Routes

Define routes as serializable data objects and data classes. This approach eliminates string-based route building and provides compile-time safety:

```kotlin
import kotlinx.serialization.Serializable

// Route with no arguments
@Serializable
data object Home

// Route with arguments
@Serializable
data class Detail(val id: Long)

// Route with optional arguments
@Serializable
data class Search(val query: String = "", val page: Int = 1)

// Nested graph route
@Serializable
data object Settings
```

### Extracting Route Arguments

```kotlin
composable<Detail> { backStackEntry ->
    val route: Detail = backStackEntry.toRoute()
    DetailScreen(
        itemId = route.id,
        onBack = { navController.popBackStack() }
    )
}

composable<Search> { backStackEntry ->
    val route: Search = backStackEntry.toRoute()
    SearchScreen(
        initialQuery = route.query,
        initialPage = route.page
    )
}
```

### Argument Types

| Type | Supported | Notes |
|------|-----------|-------|
| `Int`, `Long`, `Float`, `Boolean` | Yes | Primitive types |
| `String` | Yes | URL-encoded automatically |
| `Enum` | Yes | Serialized by name |
| `@Serializable` data class | Yes | As route itself |
| `List`, `Array` | Partial | Use custom `NavType` |
| `Parcelable`, `Serializable` | Via custom `NavType` | Requires explicit type mapping |

## Deep Links

Deep links allow external URLs or intents to navigate directly to a screen:

```kotlin
composable<Detail>(
    deepLinks = listOf(
        navDeepLink {
            uriPattern = "https://myapp.example.com/items/{id}"
        }
    )
) { backStackEntry ->
    val route: Detail = backStackEntry.toRoute()
    DetailScreen(itemId = route.id)
}
```

Register deep links in the `AndroidManifest.xml`:

```xml
<activity android:name=".MainActivity">
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="https" android:host="myapp.example.com" />
    </intent-filter>
</activity>
```

## Nested Navigation Graphs

Group related screens into nested graphs to keep the navigation structure manageable:

```kotlin
@Serializable data object AuthGraph      // Graph route
@Serializable data object Login
@Serializable data object Register
@Serializable data object ForgotPassword

@Serializable data object MainGraph      // Graph route
@Serializable data object Home
@Serializable data object Profile

@Composable
fun AppNavigation() {
    val navController = rememberNavController()

    NavHost(navController = navController, startDestination = AuthGraph) {

        navigation<AuthGraph>(startDestination = Login) {
            composable<Login> {
                LoginScreen(
                    onLogin = {
                        navController.navigate(MainGraph) {
                            popUpTo(AuthGraph) { inclusive = true }
                        }
                    },
                    onRegister = { navController.navigate(Register) },
                    onForgotPassword = { navController.navigate(ForgotPassword) }
                )
            }
            composable<Register> {
                RegisterScreen(onBack = { navController.popBackStack() })
            }
            composable<ForgotPassword> {
                ForgotPasswordScreen(onBack = { navController.popBackStack() })
            }
        }

        navigation<MainGraph>(startDestination = Home) {
            composable<Home> {
                HomeScreen(onProfile = { navController.navigate(Profile) })
            }
            composable<Profile> {
                ProfileScreen(
                    onLogout = {
                        navController.navigate(AuthGraph) {
                            popUpTo(MainGraph) { inclusive = true }
                        }
                    }
                )
            }
        }
    }
}
```

### Benefits of Nested Graphs

| Benefit | Detail |
|---------|--------|
| Encapsulation | Auth flow is self-contained |
| Back stack control | `popUpTo(AuthGraph) { inclusive = true }` clears the entire auth stack |
| Scoped ViewModels | Use `hiltViewModel()` scoped to a graph for shared state |
| Readability | Each graph is a logical feature boundary |

## Bottom Navigation

Bottom navigation tabs are a common Android pattern. Each tab typically has its own nested graph:

```kotlin
@Serializable data object HomeTab
@Serializable data object SearchTab
@Serializable data object ProfileTab

data class BottomNavItem(
    val label: String,
    val icon: ImageVector,
    val route: Any   // Type-safe route object
)

val bottomNavItems = listOf(
    BottomNavItem("Home", Icons.Default.Home, HomeTab),
    BottomNavItem("Search", Icons.Default.Search, SearchTab),
    BottomNavItem("Profile", Icons.Default.Person, ProfileTab)
)

@Composable
fun MainScreen() {
    val navController = rememberNavController()
    val currentBackStackEntry by navController.currentBackStackEntryAsState()

    Scaffold(
        bottomBar = {
            NavigationBar {
                bottomNavItems.forEach { item ->
                    NavigationBarItem(
                        icon = { Icon(item.icon, contentDescription = item.label) },
                        label = { Text(item.label) },
                        selected = currentBackStackEntry?.destination?.route ==
                            item.route::class.qualifiedName,
                        onClick = {
                            navController.navigate(item.route) {
                                // Avoid building up a large back stack
                                popUpTo(navController.graph.findStartDestination().id) {
                                    saveState = true
                                }
                                launchSingleTop = true
                                restoreState = true
                            }
                        }
                    )
                }
            }
        }
    ) { padding ->
        NavHost(
            navController = navController,
            startDestination = HomeTab,
            modifier = Modifier.padding(padding)
        ) {
            composable<HomeTab> { HomeScreen() }
            composable<SearchTab> { SearchScreen() }
            composable<ProfileTab> { ProfileScreen() }
        }
    }
}
```

### Back Stack Behaviour with Tabs

The navigation options in the `onClick` handler ensure:

- `popUpTo(startDestination) { saveState = true }` — avoids accumulating multiple copies of tabs on the back stack while saving each tab's state
- `launchSingleTop = true` — tapping the already-selected tab does not create a duplicate
- `restoreState = true` — switching back to a tab restores its previous scroll position and back stack

## Navigation Patterns

### Passing Results Back

When screen B needs to return a result to screen A:

```kotlin
// Screen A — observe the result
composable<ScreenA> {
    val result = navController.currentBackStackEntry
        ?.savedStateHandle
        ?.getStateFlow<String>("selected_item", "")
        ?.collectAsStateWithLifecycle()

    ScreenAContent(
        selectedItem = result?.value,
        onOpenPicker = { navController.navigate(ItemPicker) }
    )
}

// Screen B — set the result and pop back
composable<ItemPicker> {
    ItemPickerScreen(
        onItemSelected = { item ->
            navController.previousBackStackEntry
                ?.savedStateHandle
                ?.set("selected_item", item)
            navController.popBackStack()
        }
    )
}
```

### Conditional Navigation

```kotlin
@Composable
fun RootNavigation(isLoggedIn: Boolean) {
    val navController = rememberNavController()

    val startDestination = if (isLoggedIn) MainGraph else AuthGraph

    NavHost(navController = navController, startDestination = startDestination) {
        navigation<AuthGraph>(startDestination = Login) { /* ... */ }
        navigation<MainGraph>(startDestination = Home) { /* ... */ }
    }
}
```

## Key Takeaways

- Type-safe routes using `@Serializable` data objects and data classes eliminate string-based route building and provide compile-time argument safety
- `NavHost` defines the navigation graph; `NavController` manages the back stack and triggers navigation
- Deep links connect external URLs to specific screens — register URI patterns in both the composable and the manifest
- Nested navigation graphs group related screens (auth, settings, profile) for encapsulation and back stack control
- Bottom navigation uses `popUpTo` with `saveState`/`restoreState` to maintain tab state while keeping the back stack shallow
- Pass results between screens using `savedStateHandle` on back stack entries rather than shared ViewModels or global state
