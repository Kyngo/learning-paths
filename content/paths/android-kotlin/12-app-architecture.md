---
title: "App Architecture"
weight: 12
---

# App Architecture

Building a production Android app requires more than individual features — it requires a coherent architecture that separates concerns, manages dependencies, supports testing, and handles the Android lifecycle gracefully. This section covers MVVM with Compose, Hilt dependency injection, multi-module project structure, testing strategies, lifecycle management, and release configuration.

## MVVM with Compose

The Model-View-ViewModel pattern is the standard architecture for Compose apps. The ViewModel holds UI state and business logic; the Composable renders that state and forwards user events.

```text
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  Composable  │────▶│  ViewModel   │────▶│  Repository  │
│  (View)      │◀────│  (UI State)  │◀────│  (Data)      │
└──────────────┘     └──────────────┘     └──────────────┘
  Events go up        State goes down       Data sources
```

### Complete MVVM Example

```kotlin
// --- Domain model ---
data class Task(val id: Long, val title: String, val completed: Boolean)

// --- UI State ---
sealed interface TaskListUiState {
    data object Loading : TaskListUiState
    data class Success(
        val tasks: List<Task>,
        val filter: TaskFilter = TaskFilter.ALL
    ) : TaskListUiState
    data class Error(val message: String) : TaskListUiState
}

enum class TaskFilter { ALL, ACTIVE, COMPLETED }

// --- ViewModel ---
class TaskListViewModel(
    private val repository: TaskRepository
) : ViewModel() {

    private val filter = MutableStateFlow(TaskFilter.ALL)

    val uiState: StateFlow<TaskListUiState> = combine(
        repository.observeTasks(),
        filter
    ) { tasks, currentFilter ->
        val filtered = when (currentFilter) {
            TaskFilter.ALL -> tasks
            TaskFilter.ACTIVE -> tasks.filter { !it.completed }
            TaskFilter.COMPLETED -> tasks.filter { it.completed }
        }
        TaskListUiState.Success(filtered, currentFilter)
    }
        .catch { emit(TaskListUiState.Error(it.message ?: "Unknown error")) }
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), TaskListUiState.Loading)

    fun setFilter(filter: TaskFilter) {
        this.filter.value = filter
    }

    fun toggleTask(taskId: Long) {
        viewModelScope.launch { repository.toggleTask(taskId) }
    }

    fun deleteTask(taskId: Long) {
        viewModelScope.launch { repository.deleteTask(taskId) }
    }
}

// --- Composable ---
@Composable
fun TaskListScreen(viewModel: TaskListViewModel = hiltViewModel()) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    when (val state = uiState) {
        TaskListUiState.Loading -> CircularProgressIndicator()
        is TaskListUiState.Error -> ErrorView(state.message)
        is TaskListUiState.Success -> TaskListContent(
            tasks = state.tasks,
            filter = state.filter,
            onFilterChange = viewModel::setFilter,
            onToggle = viewModel::toggleTask,
            onDelete = viewModel::deleteTask
        )
    }
}
```

### MVVM Rules

| Rule | Rationale |
|------|-----------|
| ViewModel never references Composables or Android Views | Survives configuration changes; testable without UI |
| State flows down, events flow up | Unidirectional data flow prevents inconsistencies |
| UI state is a single sealed interface | Exhaustive handling, no invalid combinations |
| ViewModel exposes `StateFlow`, not `MutableStateFlow` | Read-only from the UI layer |
| Business logic lives in repository/use case layers | ViewModel orchestrates, doesn't implement |

## Hilt Dependency Injection

Hilt is the standard DI framework for Android. It generates the Dagger component hierarchy and provides scoped injection with minimal boilerplate:

### Setup

```kotlin
// Application class
@HiltAndroidApp
class MyApplication : Application()

// Activity
@AndroidEntryPoint
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent { MyApp() }
    }
}
```

### Module Definition

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {

    @Provides
    @Singleton
    fun provideJson(): Json = Json { ignoreUnknownKeys = true }

    @Provides
    @Singleton
    fun provideRetrofit(json: Json): Retrofit = Retrofit.Builder()
        .baseUrl("https://api.example.com/")
        .addConverterFactory(json.asConverterFactory("application/json".toMediaType()))
        .build()

    @Provides
    @Singleton
    fun provideTaskApi(retrofit: Retrofit): TaskApi =
        retrofit.create(TaskApi::class.java)
}

@Module
@InstallIn(SingletonComponent::class)
object DatabaseModule {

    @Provides
    @Singleton
    fun provideDatabase(@ApplicationContext context: Context): AppDatabase =
        Room.databaseBuilder(context, AppDatabase::class.java, "app-db").build()

    @Provides
    fun provideTaskDao(db: AppDatabase): TaskDao = db.taskDao()
}

@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {
    @Binds
    @Singleton
    abstract fun bindTaskRepository(impl: TaskRepositoryImpl): TaskRepository
}
```

### Injecting into ViewModel

```kotlin
@HiltViewModel
class TaskListViewModel @Inject constructor(
    private val repository: TaskRepository
) : ViewModel() {
    // ...
}

// In Compose — automatically scoped to navigation destination
@Composable
fun TaskListScreen(viewModel: TaskListViewModel = hiltViewModel()) {
    // ...
}
```

### Hilt Scopes

| Scope | Component | Lifetime |
|-------|-----------|----------|
| `@Singleton` | `SingletonComponent` | Application lifetime |
| `@ActivityScoped` | `ActivityComponent` | Activity lifetime |
| `@ViewModelScoped` | `ViewModelComponent` | ViewModel lifetime |
| `@ActivityRetainedScoped` | `ActivityRetainedComponent` | Survives config changes |
| No scope | Created each time | New instance per injection |

## Multi-Module Projects

As an app grows, splitting into modules improves build times, enforces boundaries, and enables parallel development:

```text
:app                          # Application module — Hilt setup, navigation, theming
├── :feature:tasks            # Task list and detail screens
├── :feature:profile          # Profile screens
├── :feature:settings         # Settings screens
├── :core:data                # Repositories, data sources
├── :core:network             # Retrofit, API definitions
├── :core:database            # Room, DAOs, entities
├── :core:model               # Domain models (shared across features)
├── :core:common              # Shared utilities, extensions
└── :core:ui                  # Shared Compose components, theme
```

### Module Dependency Rules

| Module type | Can depend on | Cannot depend on |
|-------------|--------------|------------------|
| `:app` | All modules | — |
| `:feature:*` | `:core:*` | Other `:feature:*` modules |
| `:core:data` | `:core:network`, `:core:database`, `:core:model` | `:feature:*`, `:app` |
| `:core:model` | Nothing (pure Kotlin) | Any other module |
| `:core:ui` | `:core:model`, `:core:common` | `:feature:*`, `:core:data` |

### Module build.gradle.kts

```kotlin
// :core:model — pure Kotlin, no Android dependency
plugins {
    id("java-library")
    alias(libs.plugins.kotlin.jvm)
    alias(libs.plugins.kotlin.serialization)
}

// :feature:tasks — Android + Compose
plugins {
    alias(libs.plugins.android.library)
    alias(libs.plugins.kotlin.android)
    alias(libs.plugins.hilt)
    alias(libs.plugins.ksp)
}

android {
    namespace = "com.example.feature.tasks"
    buildFeatures { compose = true }
}

dependencies {
    implementation(project(":core:model"))
    implementation(project(":core:data"))
    implementation(project(":core:ui"))
    implementation(libs.hilt.android)
    ksp(libs.hilt.compiler)
    testImplementation(libs.junit)
    testImplementation(libs.mockk)
}
```

## Testing

### Unit Tests (ViewModel + Repository)

```kotlin
class TaskListViewModelTest {

    @get:Rule
    val mainDispatcherRule = MainDispatcherRule()  // Replaces Dispatchers.Main in tests

    private val repository = FakeTaskRepository()
    private lateinit var viewModel: TaskListViewModel

    @Before
    fun setup() {
        viewModel = TaskListViewModel(repository)
    }

    @Test
    fun `initial state is loading then success`() = runTest {
        val states = mutableListOf<TaskListUiState>()

        val job = launch {
            viewModel.uiState.toList(states)
        }

        advanceUntilIdle()
        assertThat(states.last()).isInstanceOf(TaskListUiState.Success::class.java)
        job.cancel()
    }

    @Test
    fun `toggle task updates completion state`() = runTest {
        advanceUntilIdle()
        viewModel.toggleTask(1L)
        advanceUntilIdle()

        val state = viewModel.uiState.value as TaskListUiState.Success
        val task = state.tasks.first { it.id == 1L }
        assertThat(task.completed).isTrue()
    }
}

// Test dispatcher rule
class MainDispatcherRule(
    private val dispatcher: TestDispatcher = UnconfinedTestDispatcher()
) : TestWatcher() {
    override fun starting(description: Description) {
        Dispatchers.setMain(dispatcher)
    }
    override fun finished(description: Description) {
        Dispatchers.resetMain()
    }
}
```

### Compose UI Tests

```kotlin
class TaskListScreenTest {

    @get:Rule
    val composeRule = createComposeRule()

    @Test
    fun `displays task list`() {
        val tasks = listOf(
            Task(1, "Buy groceries", false),
            Task(2, "Write tests", true)
        )

        composeRule.setContent {
            TaskListContent(
                tasks = tasks,
                filter = TaskFilter.ALL,
                onFilterChange = {},
                onToggle = {},
                onDelete = {}
            )
        }

        composeRule.onNodeWithText("Buy groceries").assertIsDisplayed()
        composeRule.onNodeWithText("Write tests").assertIsDisplayed()
    }

    @Test
    fun `tap task calls onToggle`() {
        var toggledId: Long? = null
        val tasks = listOf(Task(1, "Test task", false))

        composeRule.setContent {
            TaskListContent(
                tasks = tasks,
                filter = TaskFilter.ALL,
                onFilterChange = {},
                onToggle = { toggledId = it },
                onDelete = {}
            )
        }

        composeRule.onNodeWithText("Test task").performClick()
        assertThat(toggledId).isEqualTo(1L)
    }
}
```

### MockK for Mocking

```kotlin
class TaskRepositoryTest {

    private val api = mockk<TaskApi>()
    private val dao = mockk<TaskDao>(relaxed = true)
    private val repository = TaskRepositoryImpl(api, dao)

    @Test
    fun `refresh fetches from API and caches locally`() = runTest {
        val remoteTasks = listOf(Task(1, "Remote task", false))
        coEvery { api.getTasks() } returns remoteTasks

        repository.refresh()

        coVerify { api.getTasks() }
        coVerify { dao.insertAll(any()) }
    }

    @Test
    fun `observe tasks returns flow from DAO`() = runTest {
        val localTasks = listOf(Task(1, "Local task", false))
        every { dao.observeAll() } returns flowOf(localTasks.map { it.toEntity() })

        val result = repository.observeTasks().first()

        assertThat(result).hasSize(1)
        assertThat(result.first().title).isEqualTo("Local task")
    }
}
```

## App Lifecycle

### ViewModel Lifecycle

ViewModels survive configuration changes (rotation, dark mode toggle) but not process death. Use `SavedStateHandle` for state that must survive process death:

```kotlin
@HiltViewModel
class SearchViewModel @Inject constructor(
    private val savedStateHandle: SavedStateHandle,
    private val repository: SearchRepository
) : ViewModel() {

    val query = savedStateHandle.getStateFlow("query", "")

    fun setQuery(newQuery: String) {
        savedStateHandle["query"] = newQuery
    }
}
```

### Lifecycle-Aware Collection

Always use `collectAsStateWithLifecycle()` in Compose, not `collectAsState()`:

```kotlin
// Correct — stops collecting when the screen is not visible
val state by viewModel.uiState.collectAsStateWithLifecycle()

// Wrong — continues collecting even when the app is in the background
val state by viewModel.uiState.collectAsState()
```

### Process Death and Restoration

| State location | Survives rotation | Survives process death |
|---------------|-------------------|----------------------|
| `remember { }` | No | No |
| `rememberSaveable { }` | Yes | Yes |
| ViewModel property | Yes | No |
| `SavedStateHandle` | Yes | Yes |
| Room / DataStore | Yes | Yes |

## Release and ProGuard

### Build Configuration

```kotlin
// build.gradle.kts (app module)
android {
    buildTypes {
        release {
            isMinifyEnabled = true
            isShrinkResources = true
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
            signingConfig = signingConfigs.getByName("release")
        }
    }
}
```

### ProGuard / R8 Rules

```text
# Keep kotlinx.serialization classes
-keepattributes *Annotation*, InnerClasses
-dontnote kotlinx.serialization.AnnotationsKt

-keepclassmembers @kotlinx.serialization.Serializable class ** {
    *** Companion;
    kotlinx.serialization.KSerializer serializer(...);
}

# Keep Hilt-generated classes
-keep class dagger.hilt.** { *; }

# Keep Retrofit API interfaces
-keep,allowobfuscation interface * {
    @retrofit2.http.* <methods>;
}

# Keep Room entities
-keep @androidx.room.Entity class * { *; }
```

### Version Catalogue (libs.versions.toml)

Centralise dependency versions in a TOML catalogue:

```toml
[versions]
kotlin = "2.0.20"
compose-bom = "2024.09.00"
hilt = "2.51.1"
room = "2.6.1"
retrofit = "2.11.0"
navigation = "2.8.0"
paging = "3.3.2"
mockk = "1.13.12"

[libraries]
compose-bom = { group = "androidx.compose", name = "compose-bom", version.ref = "compose-bom" }
compose-material3 = { group = "androidx.compose.material3", name = "material3" }
compose-ui-tooling = { group = "androidx.compose.ui", name = "ui-tooling" }
hilt-android = { group = "com.google.dagger", name = "hilt-android", version.ref = "hilt" }
hilt-compiler = { group = "com.google.dagger", name = "hilt-android-compiler", version.ref = "hilt" }
room-runtime = { group = "androidx.room", name = "room-runtime", version.ref = "room" }
room-ktx = { group = "androidx.room", name = "room-ktx", version.ref = "room" }
room-compiler = { group = "androidx.room", name = "room-compiler", version.ref = "room" }
retrofit = { group = "com.squareup.retrofit2", name = "retrofit", version.ref = "retrofit" }
navigation-compose = { group = "androidx.navigation", name = "navigation-compose", version.ref = "navigation" }
mockk = { group = "io.mockk", name = "mockk", version.ref = "mockk" }

[plugins]
android-application = { id = "com.android.application", version = "8.6.0" }
kotlin-android = { id = "org.jetbrains.kotlin.android", version.ref = "kotlin" }
kotlin-serialization = { id = "org.jetbrains.kotlin.plugin.serialization", version.ref = "kotlin" }
compose-compiler = { id = "org.jetbrains.kotlin.plugin.compose", version.ref = "kotlin" }
hilt = { id = "com.google.dagger.hilt.android", version.ref = "hilt" }
ksp = { id = "com.google.devtools.ksp", version = "2.0.20-1.0.25" }
```

## Key Takeaways

- MVVM with unidirectional data flow (state down, events up) is the standard architecture — model all screen states as a sealed interface
- Hilt provides scoped dependency injection with minimal boilerplate — use `@HiltViewModel`, `@Inject`, and module bindings to wire the dependency graph
- Multi-module projects enforce architectural boundaries and improve build times — feature modules depend on core modules, never on each other
- Test ViewModels with fakes and `runTest`; test Compose UI with `createComposeRule` and semantic assertions; use MockK for mocking dependencies
- Use `collectAsStateWithLifecycle()` (not `collectAsState()`) to stop collection when the screen is not visible, saving resources and preventing crashes
- R8/ProGuard shrinks and obfuscates release builds — maintain rules for serialization, Hilt, Retrofit, and Room to prevent runtime crashes
