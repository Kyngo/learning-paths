---
title: "Jetpack Compose Fundamentals"
weight: 7
---

# Jetpack Compose Fundamentals

Jetpack Compose is Android's modern declarative UI toolkit. Instead of building views imperatively through XML layouts and `findViewById`, you describe what the UI should look like as a function of state, and the framework handles rendering and updates.

## Declarative vs Imperative UI

| Aspect | Imperative (XML + View) | Declarative (Compose) |
|--------|------------------------|-----------------------|
| UI definition | XML layouts + code wiring | Kotlin functions |
| State updates | Find view, call setters | Recompose with new state |
| View hierarchy | Mutable, held in memory | Immutable descriptions, rebuilt on change |
| Reusability | Custom views, includes | Composable functions |
| Preview | Layout editor | `@Preview` annotation |

In Compose, the UI is a function of state. When state changes, the framework re-executes (recomposes) only the affected composable functions.

## @Composable Functions

A composable function describes a piece of UI. It is annotated with `@Composable` and can call other composable functions:

```kotlin
@Composable
fun Greeting(name: String) {
    Text(text = "Hello, $name!")
}

@Composable
fun WelcomeScreen() {
    Column {
        Greeting("Alice")
        Greeting("Bob")
    }
}
```

### Rules of Composable Functions

- Must be annotated with `@Composable`
- Can only be called from other `@Composable` functions (or from `setContent {}`)
- Should be fast, idempotent, and free of side effects
- Can be called in any order, skipped, or re-executed by the runtime
- Should not depend on the order of execution of sibling composables

```kotlin
// Entry point — typically in an Activity
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            MyAppTheme {
                WelcomeScreen()
            }
        }
    }
}
```

## Modifiers

Modifiers decorate composables with layout, drawing, and interaction behaviour. They chain from left to right, and order matters:

```kotlin
@Composable
fun StyledCard() {
    Text(
        text = "Hello, Compose!",
        modifier = Modifier
            .fillMaxWidth()
            .padding(16.dp)
            .background(Color.LightGray, RoundedCornerShape(8.dp))
            .padding(24.dp)  // Inner padding (after background)
            .clickable { println("Tapped") }
    )
}
```

### Common Modifiers

| Modifier | Purpose |
|----------|---------|
| `.padding(dp)` | Add space inside or outside |
| `.fillMaxWidth()` / `.fillMaxHeight()` / `.fillMaxSize()` | Expand to fill parent |
| `.size(width, height)` | Fixed size |
| `.width(dp)` / `.height(dp)` | Fixed single dimension |
| `.background(color, shape)` | Background colour and shape |
| `.border(width, color, shape)` | Border line |
| `.clip(shape)` | Clip content to shape |
| `.clickable { }` | Handle click events |
| `.weight(float)` | Proportional sizing in Row/Column |
| `.align(alignment)` | Alignment within parent |
| `.offset(x, y)` | Position offset |

### Modifier Order Matters

```kotlin
// Background THEN padding — background extends behind padding
Modifier.background(Color.Red).padding(16.dp)

// Padding THEN background — background only covers content area
Modifier.padding(16.dp).background(Color.Red)
```

## Layouts

Compose provides three fundamental layout composables and lazy variants for scrollable content.

### Column, Row, and Box

```kotlin
@Composable
fun LayoutExamples() {
    // Column — vertical arrangement
    Column(
        modifier = Modifier.fillMaxWidth().padding(16.dp),
        verticalArrangement = Arrangement.spacedBy(8.dp),
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        Text("First")
        Text("Second")
        Text("Third")
    }

    // Row — horizontal arrangement
    Row(
        horizontalArrangement = Arrangement.SpaceBetween,
        verticalAlignment = Alignment.CenterVertically,
        modifier = Modifier.fillMaxWidth()
    ) {
        Text("Left")
        Text("Right")
    }

    // Box — stack children (like FrameLayout)
    Box(
        modifier = Modifier.size(200.dp),
        contentAlignment = Alignment.Center
    ) {
        Image(painter = painterResource(R.drawable.bg), contentDescription = null)
        Text("Overlay", color = Color.White)
    }
}
```

### Arrangement and Alignment

| Layout | Arrangement axis | Alignment axis |
|--------|-----------------|----------------|
| `Column` | `verticalArrangement` | `horizontalAlignment` |
| `Row` | `horizontalArrangement` | `verticalAlignment` |
| `Box` | — | `contentAlignment` |

Common arrangements: `Arrangement.Top`, `Arrangement.SpaceBetween`, `Arrangement.spacedBy(8.dp)`, `Arrangement.Center`.

### LazyColumn and LazyRow

For scrollable lists of items, use lazy layouts. They only compose and lay out visible items:

```kotlin
@Composable
fun UserList(users: List<User>) {
    LazyColumn(
        contentPadding = PaddingValues(16.dp),
        verticalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        items(
            items = users,
            key = { it.id }  // Stable key for recomposition efficiency
        ) { user ->
            UserCard(user)
        }
    }
}

// With headers and mixed content
LazyColumn {
    item { Header("Active Users") }

    items(activeUsers, key = { it.id }) { user ->
        UserCard(user)
    }

    item { Header("Inactive Users") }

    items(inactiveUsers, key = { it.id }) { user ->
        UserCard(user, dimmed = true)
    }
}
```

## Previews

The `@Preview` annotation renders composables directly in Android Studio without running the app:

```kotlin
@Preview(showBackground = true)
@Composable
fun GreetingPreview() {
    MyAppTheme {
        Greeting("Preview")
    }
}

@Preview(
    name = "Dark Mode",
    uiMode = Configuration.UI_MODE_NIGHT_YES,
    showBackground = true
)
@Composable
fun GreetingDarkPreview() {
    MyAppTheme {
        Greeting("Dark Preview")
    }
}

// Multiple previews with different configurations
@Preview(device = Devices.PIXEL_7)
@Preview(device = Devices.PIXEL_TABLET)
@Preview(fontScale = 2f)
@Composable
fun ResponsivePreview() {
    MyAppTheme {
        WelcomeScreen()
    }
}
```

### Preview Best Practices

- Create preview functions for every reusable composable
- Provide sample data through preview parameters or `@PreviewParameter`
- Preview both light and dark themes
- Preview different screen sizes and font scales
- Keep previews fast — avoid network calls or database access

## Material 3 Components

Compose integrates with Material 3 (Material You) for consistent, themed UI components:

```kotlin
@Composable
fun FormExample() {
    var text by remember { mutableStateOf("") }

    Column(modifier = Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(12.dp)) {
        // Text fields
        OutlinedTextField(
            value = text,
            onValueChange = { text = it },
            label = { Text("Username") },
            leadingIcon = { Icon(Icons.Default.Person, contentDescription = null) }
        )

        // Buttons
        Button(onClick = { /* submit */ }) {
            Text("Submit")
        }
        OutlinedButton(onClick = { /* cancel */ }) {
            Text("Cancel")
        }
        TextButton(onClick = { /* link */ }) {
            Text("Forgot Password?")
        }

        // Card
        ElevatedCard(modifier = Modifier.fillMaxWidth()) {
            Column(modifier = Modifier.padding(16.dp)) {
                Text("Card Title", style = MaterialTheme.typography.titleMedium)
                Text("Card content here", style = MaterialTheme.typography.bodyMedium)
            }
        }
    }
}
```

### Theming

Material 3 themes define colours, typography, and shapes:

```kotlin
@Composable
fun MyAppTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    content: @Composable () -> Unit
) {
    val colorScheme = if (darkTheme) darkColorScheme(
        primary = Color(0xFFBB86FC),
        secondary = Color(0xFF03DAC6),
        background = Color(0xFF121212)
    ) else lightColorScheme(
        primary = Color(0xFF6200EE),
        secondary = Color(0xFF03DAC6),
        background = Color(0xFFFFFBFE)
    )

    MaterialTheme(
        colorScheme = colorScheme,
        typography = Typography(),
        content = content
    )
}

// Access theme values in any composable
@Composable
fun ThemedText() {
    Text(
        text = "Themed",
        color = MaterialTheme.colorScheme.primary,
        style = MaterialTheme.typography.headlineMedium
    )
}
```

### Scaffold and Top-Level Layout

`Scaffold` provides the standard Material 3 app structure:

```kotlin
@Composable
fun MainScreen() {
    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text("My App") },
                actions = {
                    IconButton(onClick = { /* settings */ }) {
                        Icon(Icons.Default.Settings, contentDescription = "Settings")
                    }
                }
            )
        },
        floatingActionButton = {
            FloatingActionButton(onClick = { /* add item */ }) {
                Icon(Icons.Default.Add, contentDescription = "Add")
            }
        },
        bottomBar = {
            NavigationBar {
                NavigationBarItem(
                    icon = { Icon(Icons.Default.Home, contentDescription = null) },
                    label = { Text("Home") },
                    selected = true,
                    onClick = { }
                )
                NavigationBarItem(
                    icon = { Icon(Icons.Default.Person, contentDescription = null) },
                    label = { Text("Profile") },
                    selected = false,
                    onClick = { }
                )
            }
        }
    ) { paddingValues ->
        // Content — must apply paddingValues to avoid overlapping with bars
        Column(modifier = Modifier.padding(paddingValues)) {
            Text("Screen content")
        }
    }
}
```

## Key Takeaways

- Compose replaces XML layouts with `@Composable` functions that describe UI as a function of state
- Modifiers chain declaratively and order matters — padding before background differs from background before padding
- `Column`, `Row`, and `Box` are the fundamental layout primitives; `LazyColumn`/`LazyRow` handle scrollable lists efficiently
- `@Preview` renders composables in Android Studio without running the app — use it liberally for rapid iteration
- Material 3 integration provides themed components (`Button`, `Card`, `TextField`, `Scaffold`) out of the box
- `Scaffold` with `TopAppBar`, `NavigationBar`, and `FloatingActionButton` provides the standard Android app shell
