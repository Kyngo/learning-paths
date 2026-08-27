---
title: "Compose State Management"
weight: 8
---

# Compose State Management

State management is the central challenge in Compose. The framework re-executes composable functions whenever state changes (recomposition), so understanding how to hold, hoist, and observe state determines whether your UI is correct, performant, and testable.

## remember and mutableStateOf

`mutableStateOf` creates an observable state holder. When its value changes, Compose recomposes any composable that reads it. `remember` preserves the value across recompositions:

```kotlin
@Composable
fun Counter() {
    var count by remember { mutableStateOf(0) }

    Column(horizontalAlignment = Alignment.CenterHorizontally) {
        Text("Count: $count", style = MaterialTheme.typography.headlineMedium)
        Button(onClick = { count++ }) {
            Text("Increment")
        }
    }
}
```

### remember vs rememberSaveable

| Function | Survives recomposition | Survives configuration change | Survives process death |
|----------|----------------------|------------------------------|----------------------|
| `remember` | Yes | No | No |
| `rememberSaveable` | Yes | Yes | Yes (if using Saver) |

```kotlin
// Survives screen rotation
var text by rememberSaveable { mutableStateOf("") }

// Custom saver for complex types
data class SearchState(val query: String, val page: Int)

val SearchStateSaver = Saver<SearchState, List<Any>>(
    save = { listOf(it.query, it.page) },
    restore = { SearchState(it[0] as String, it[1] as Int) }
)

var searchState by rememberSaveable(stateSaver = SearchStateSaver) {
    mutableStateOf(SearchState("", 1))
}
```

### State Delegation Patterns

```kotlin
// Delegation with 'by' — read/write directly
var name by remember { mutableStateOf("") }
name = "Alice"     // Direct assignment
println(name)      // Direct read

// Without delegation — explicit .value
val nameState = remember { mutableStateOf("") }
nameState.value = "Alice"
println(nameState.value)

// Destructuring
val (name, setName) = remember { mutableStateOf("") }
setName("Alice")   // Setter function
```

## State Hoisting

State hoisting moves state up to a caller, making the composable stateless and reusable. The pattern is: state goes down as parameters, events go up as callbacks:

```kotlin
// Stateless — receives state and callbacks from parent
@Composable
fun SearchBar(
    query: String,
    onQueryChange: (String) -> Unit,
    onSearch: () -> Unit,
    modifier: Modifier = Modifier
) {
    OutlinedTextField(
        value = query,
        onValueChange = onQueryChange,
        modifier = modifier.fillMaxWidth(),
        placeholder = { Text("Search...") },
        trailingIcon = {
            IconButton(onClick = onSearch) {
                Icon(Icons.Default.Search, contentDescription = "Search")
            }
        }
    )
}

// Stateful parent — owns the state
@Composable
fun SearchScreen() {
    var query by remember { mutableStateOf("") }
    var results by remember { mutableStateOf<List<Item>>(emptyList()) }

    Column {
        SearchBar(
            query = query,
            onQueryChange = { query = it },
            onSearch = { results = performSearch(query) }
        )
        ResultList(results)
    }
}
```

### When to Hoist State

| Scenario | Keep local | Hoist |
|----------|-----------|-------|
| Animation state, scroll position | ✓ | |
| Form field used by one composable | ✓ | |
| State shared between composables | | ✓ |
| State needed by parent for logic | | ✓ |
| State that should survive navigation | | ✓ (to ViewModel) |

## ViewModel + StateFlow

For state that should survive configuration changes and belongs to screen-level logic, use a `ViewModel` with `StateFlow`:

```kotlin
data class ProfileUiState(
    val user: User? = null,
    val isLoading: Boolean = false,
    val error: String? = null
)

class ProfileViewModel(
    private val userRepository: UserRepository
) : ViewModel() {

    private val _uiState = MutableStateFlow(ProfileUiState())
    val uiState: StateFlow<ProfileUiState> = _uiState.asStateFlow()

    init {
        loadProfile()
    }

    fun loadProfile() {
        viewModelScope.launch {
            _uiState.update { it.copy(isLoading = true, error = null) }
            try {
                val user = userRepository.getCurrentUser()
                _uiState.update { it.copy(user = user, isLoading = false) }
            } catch (e: Exception) {
                _uiState.update { it.copy(error = e.message, isLoading = false) }
            }
        }
    }

    fun logout() {
        viewModelScope.launch {
            userRepository.logout()
            _uiState.update { it.copy(user = null) }
        }
    }
}
```

### Collecting StateFlow in Compose

```kotlin
@Composable
fun ProfileScreen(viewModel: ProfileViewModel = viewModel()) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    when {
        uiState.isLoading -> LoadingIndicator()
        uiState.error != null -> ErrorMessage(uiState.error!!, onRetry = viewModel::loadProfile)
        uiState.user != null -> ProfileContent(uiState.user!!, onLogout = viewModel::logout)
    }
}
```

`collectAsStateWithLifecycle()` is lifecycle-aware — it stops collecting when the composable leaves the screen, preventing wasted work and potential crashes.

### UI State Pattern

Model all possible screen states in a single sealed interface:

```kotlin
sealed interface HomeUiState {
    data object Loading : HomeUiState
    data class Success(val items: List<Item>) : HomeUiState
    data class Error(val message: String) : HomeUiState
}

class HomeViewModel(private val repository: ItemRepository) : ViewModel() {

    val uiState: StateFlow<HomeUiState> = repository
        .observeItems()
        .map<List<Item>, HomeUiState> { HomeUiState.Success(it) }
        .catch { emit(HomeUiState.Error(it.message ?: "Unknown error")) }
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5_000),
            initialValue = HomeUiState.Loading
        )
}

@Composable
fun HomeScreen(viewModel: HomeViewModel = viewModel()) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    when (val state = uiState) {
        HomeUiState.Loading -> CircularProgressIndicator()
        is HomeUiState.Success -> ItemList(state.items)
        is HomeUiState.Error -> ErrorView(state.message)
    }
}
```

## Side Effects

Side effects are operations that escape the scope of a composable function — network calls, logging, timers, or subscriptions. Compose provides effect handlers to manage them safely.

### LaunchedEffect

Runs a suspend function when the composable enters the composition. Cancels and restarts when the key changes:

```kotlin
@Composable
fun MessageScreen(userId: String) {
    var messages by remember { mutableStateOf<List<Message>>(emptyList()) }

    LaunchedEffect(userId) {
        // Runs when userId changes; cancelled when composable leaves
        messages = repository.getMessages(userId)
    }

    LaunchedEffect(Unit) {
        // Runs once when composable enters composition
        analytics.trackScreenView("messages")
    }

    MessageList(messages)
}
```

### DisposableEffect

For effects that require cleanup (subscriptions, listeners, callbacks):

```kotlin
@Composable
fun LocationTracker(onLocationUpdate: (Location) -> Unit) {
    val context = LocalContext.current

    DisposableEffect(Unit) {
        val locationManager = context.getSystemService<LocationManager>()
        val listener = LocationListener { location ->
            onLocationUpdate(location)
        }
        locationManager?.requestLocationUpdates(
            LocationManager.GPS_PROVIDER, 1000L, 10f, listener
        )

        onDispose {
            locationManager?.removeUpdates(listener)
        }
    }
}
```

### SideEffect

Runs after every successful recomposition. Used to update non-Compose state:

```kotlin
@Composable
fun AnalyticsScreen(screenName: String) {
    SideEffect {
        // Runs after every recomposition
        analytics.setCurrentScreen(screenName)
    }
}
```

### Effect Selection Guide

| Effect | When it runs | Cleanup | Use case |
|--------|-------------|---------|----------|
| `LaunchedEffect(key)` | On enter + when key changes | Auto-cancelled | Async operations, one-shot loads |
| `DisposableEffect(key)` | On enter + when key changes | `onDispose { }` | Subscriptions, listeners, callbacks |
| `SideEffect` | After every recomposition | None | Sync non-Compose state |
| `rememberCoroutineScope()` | Manual launch | Tied to composition | Event-driven coroutines |
| `produceState(initial)` | On enter | Auto-cancelled | Convert non-Compose state to Compose state |

### rememberCoroutineScope

For launching coroutines in response to events (not composition):

```kotlin
@Composable
fun SnackbarExample() {
    val scope = rememberCoroutineScope()
    val snackbarHostState = remember { SnackbarHostState() }

    Scaffold(snackbarHost = { SnackbarHost(snackbarHostState) }) { padding ->
        Button(
            onClick = {
                scope.launch {
                    snackbarHostState.showSnackbar("Action completed")
                }
            },
            modifier = Modifier.padding(padding)
        ) {
            Text("Show Snackbar")
        }
    }
}
```

## derivedStateOf

Creates state that is computed from other state values. The derived state only triggers recomposition when its computed value actually changes:

```kotlin
@Composable
fun FilteredList(items: List<Item>) {
    var query by remember { mutableStateOf("") }

    // Only recomposes consumers when the filtered result changes
    val filteredItems by remember(items) {
        derivedStateOf {
            if (query.isBlank()) items
            else items.filter { it.name.contains(query, ignoreCase = true) }
        }
    }

    Column {
        TextField(value = query, onValueChange = { query = it })
        LazyColumn {
            items(filteredItems, key = { it.id }) { item ->
                ItemRow(item)
            }
        }
    }
}
```

### When to Use derivedStateOf

| Scenario | Use `derivedStateOf`? |
|----------|-----------------------|
| Filtering a list based on a query | Yes — avoids recomposition when filter result is unchanged |
| Computed value from one state | Usually not — a simple `val x = state.value + 1` suffices |
| State that changes less frequently than its inputs | Yes |
| "Show scroll-to-top" button based on list position | Yes |

## Key Takeaways

- `remember` preserves state across recompositions; `rememberSaveable` also survives configuration changes and process death
- State hoisting moves state up and events down — stateless composables are reusable, testable, and previewable
- `ViewModel` + `StateFlow` is the standard pattern for screen-level state — collect with `collectAsStateWithLifecycle()` for lifecycle awareness
- Model UI state as a sealed interface (`Loading`/`Success`/`Error`) for exhaustive handling
- Use `LaunchedEffect` for async operations, `DisposableEffect` for subscriptions requiring cleanup, and `SideEffect` for post-recomposition synchronisation
- `derivedStateOf` prevents unnecessary recompositions when a computed value changes less frequently than its inputs
