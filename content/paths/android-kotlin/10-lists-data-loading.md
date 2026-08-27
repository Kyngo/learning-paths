---
title: "Lists and Data Loading"
weight: 10
---

# Lists and Data Loading

Most Android apps revolve around fetching data, storing it locally, and displaying it in scrollable lists. This section covers efficient list rendering with `LazyColumn` and `LazyGrid`, pagination with Paging 3, networking with Retrofit, local persistence with Room, and the repository pattern that ties everything together.

## LazyColumn and LazyGrid

### LazyColumn Patterns

`LazyColumn` only composes and lays out items currently visible on screen. Always provide stable keys for correct recomposition and animation:

```kotlin
@Composable
fun ArticleList(articles: List<Article>, onArticleClick: (Long) -> Unit) {
    LazyColumn(
        contentPadding = PaddingValues(16.dp),
        verticalArrangement = Arrangement.spacedBy(12.dp)
    ) {
        items(
            items = articles,
            key = { it.id }      // Stable identity
        ) { article ->
            ArticleCard(
                article = article,
                onClick = { onArticleClick(article.id) },
                modifier = Modifier.animateItem()  // Animate add/remove/reorder
            )
        }
    }
}
```

### Sticky Headers

```kotlin
@OptIn(ExperimentalFoundationApi::class)
@Composable
fun GroupedList(grouped: Map<String, List<Contact>>) {
    LazyColumn {
        grouped.forEach { (letter, contacts) ->
            stickyHeader {
                Text(
                    text = letter,
                    modifier = Modifier
                        .fillMaxWidth()
                        .background(MaterialTheme.colorScheme.surfaceVariant)
                        .padding(horizontal = 16.dp, vertical = 8.dp),
                    style = MaterialTheme.typography.titleSmall
                )
            }
            items(contacts, key = { it.id }) { contact ->
                ContactRow(contact)
            }
        }
    }
}
```

### LazyVerticalGrid

For grid layouts (image galleries, product catalogs):

```kotlin
@Composable
fun PhotoGrid(photos: List<Photo>) {
    LazyVerticalGrid(
        columns = GridCells.Adaptive(minSize = 120.dp),
        contentPadding = PaddingValues(8.dp),
        horizontalArrangement = Arrangement.spacedBy(8.dp),
        verticalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        items(photos, key = { it.id }) { photo ->
            AsyncImage(
                model = photo.url,
                contentDescription = photo.description,
                contentScale = ContentScale.Crop,
                modifier = Modifier
                    .aspectRatio(1f)
                    .clip(RoundedCornerShape(8.dp))
            )
        }
    }
}
```

| Grid Column Type | Behaviour |
|-----------------|-----------|
| `GridCells.Fixed(n)` | Exactly `n` columns |
| `GridCells.Adaptive(minSize)` | As many columns as fit with minimum width |
| `GridCells.FixedSize(size)` | Columns of exact width, count varies |

## Paging 3

The Paging 3 library loads data incrementally as the user scrolls, handling pagination logic, loading states, and caching.

### PagingSource

```kotlin
class ArticlePagingSource(
    private val api: ArticleApi
) : PagingSource<Int, Article>() {

    override fun getRefreshKey(state: PagingState<Int, Article>): Int? {
        return state.anchorPosition?.let { anchor ->
            state.closestPageToPosition(anchor)?.prevKey?.plus(1)
                ?: state.closestPageToPosition(anchor)?.nextKey?.minus(1)
        }
    }

    override suspend fun load(params: LoadParams<Int>): LoadResult<Int, Article> {
        val page = params.key ?: 1
        return try {
            val response = api.getArticles(page = page, size = params.loadSize)
            LoadResult.Page(
                data = response.items,
                prevKey = if (page == 1) null else page - 1,
                nextKey = if (response.items.isEmpty()) null else page + 1
            )
        } catch (e: Exception) {
            LoadResult.Error(e)
        }
    }
}
```

### ViewModel with Pager

```kotlin
class ArticleListViewModel(
    private val api: ArticleApi
) : ViewModel() {

    val articles: Flow<PagingData<Article>> = Pager(
        config = PagingConfig(
            pageSize = 20,
            enablePlaceholders = false,
            prefetchDistance = 5
        ),
        pagingSourceFactory = { ArticlePagingSource(api) }
    ).flow.cachedIn(viewModelScope)
}
```

### Compose Integration

```kotlin
@Composable
fun PaginatedArticleList(viewModel: ArticleListViewModel = viewModel()) {
    val articles = viewModel.articles.collectAsLazyPagingItems()

    LazyColumn(
        contentPadding = PaddingValues(16.dp),
        verticalArrangement = Arrangement.spacedBy(12.dp)
    ) {
        items(
            count = articles.itemCount,
            key = articles.itemKey { it.id }
        ) { index ->
            val article = articles[index]
            if (article != null) {
                ArticleCard(article)
            }
        }

        // Loading state at the bottom
        when (articles.loadState.append) {
            is LoadState.Loading -> item { LoadingRow() }
            is LoadState.Error -> item {
                ErrorRow(
                    message = "Failed to load more",
                    onRetry = { articles.retry() }
                )
            }
            else -> {}
        }
    }

    // Full-screen loading/error for initial load
    when (articles.loadState.refresh) {
        is LoadState.Loading -> FullScreenLoading()
        is LoadState.Error -> FullScreenError(onRetry = { articles.refresh() })
        else -> {}
    }
}
```

## Retrofit + Kotlin Serialization

Retrofit handles HTTP networking. Combined with `kotlinx.serialization`, it provides type-safe API calls:

### Setup

```kotlin
// build.gradle.kts dependencies
// implementation("com.squareup.retrofit2:retrofit:2.11.0")
// implementation("com.squareup.retrofit2:converter-kotlinx-serialization:2.11.0")
// implementation("com.squareup.okhttp3:logging-interceptor:4.12.0")
// implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.7.0")

val json = Json { ignoreUnknownKeys = true }

val retrofit = Retrofit.Builder()
    .baseUrl("https://api.example.com/")
    .addConverterFactory(json.asConverterFactory("application/json".toMediaType()))
    .client(
        OkHttpClient.Builder()
            .addInterceptor(HttpLoggingInterceptor().apply {
                level = HttpLoggingInterceptor.Level.BODY
            })
            .build()
    )
    .build()
```

### API Interface

```kotlin
interface ArticleApi {
    @GET("articles")
    suspend fun getArticles(
        @Query("page") page: Int,
        @Query("size") size: Int
    ): PaginatedResponse<Article>

    @GET("articles/{id}")
    suspend fun getArticle(@Path("id") id: Long): Article

    @POST("articles")
    suspend fun createArticle(@Body article: CreateArticleRequest): Article

    @DELETE("articles/{id}")
    suspend fun deleteArticle(@Path("id") id: Long)
}

@Serializable
data class PaginatedResponse<T>(
    val items: List<T>,
    val total: Int,
    val page: Int
)

@Serializable
data class Article(
    val id: Long,
    val title: String,
    val body: String,
    @SerialName("created_at") val createdAt: String
)

@Serializable
data class CreateArticleRequest(
    val title: String,
    val body: String
)
```

## Room Database

Room provides an abstraction layer over SQLite, offering compile-time verification of SQL queries and seamless integration with Kotlin coroutines and Flow.

### Entity

```kotlin
@Entity(tableName = "articles")
data class ArticleEntity(
    @PrimaryKey val id: Long,
    val title: String,
    val body: String,
    val createdAt: String,
    val cachedAt: Long = System.currentTimeMillis()
)
```

### DAO (Data Access Object)

```kotlin
@Dao
interface ArticleDao {
    @Query("SELECT * FROM articles ORDER BY createdAt DESC")
    fun observeAll(): Flow<List<ArticleEntity>>  // Reactive — emits on changes

    @Query("SELECT * FROM articles WHERE id = :id")
    suspend fun getById(id: Long): ArticleEntity?

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertAll(articles: List<ArticleEntity>)

    @Delete
    suspend fun delete(article: ArticleEntity)

    @Query("DELETE FROM articles WHERE cachedAt < :threshold")
    suspend fun deleteOlderThan(threshold: Long)

    @Query("SELECT COUNT(*) FROM articles")
    suspend fun count(): Int
}
```

### Database

```kotlin
@Database(entities = [ArticleEntity::class], version = 1)
abstract class AppDatabase : RoomDatabase() {
    abstract fun articleDao(): ArticleDao
}

// Create instance (typically via DI)
val db = Room.databaseBuilder(
    context.applicationContext,
    AppDatabase::class.java,
    "app-database"
).build()
```

### Room + Flow

Room DAOs returning `Flow` automatically re-emit when the underlying table changes:

```kotlin
class ArticleLocalDataSource(private val dao: ArticleDao) {
    fun observeArticles(): Flow<List<Article>> =
        dao.observeAll().map { entities ->
            entities.map { it.toDomain() }
        }

    suspend fun cacheArticles(articles: List<Article>) {
        dao.insertAll(articles.map { it.toEntity() })
    }
}
```

## Offline-First Patterns

The offline-first approach serves cached data immediately while refreshing from the network in the background:

```kotlin
class ArticleRepository(
    private val api: ArticleApi,
    private val dao: ArticleDao
) {
    fun observeArticles(): Flow<Resource<List<Article>>> = flow {
        // 1. Emit cached data immediately
        val cached = dao.observeAll().first()
        if (cached.isNotEmpty()) {
            emit(Resource.Success(cached.map { it.toDomain() }))
        } else {
            emit(Resource.Loading)
        }

        // 2. Fetch from network
        try {
            val remote = api.getArticles(page = 1, size = 50)
            dao.insertAll(remote.items.map { it.toEntity() })
            // Room Flow will automatically emit the updated data
        } catch (e: Exception) {
            if (cached.isEmpty()) {
                emit(Resource.Error(e.message ?: "Network error"))
            }
            // If we have cached data, silently fail — user sees cached content
        }
    }
}

sealed class Resource<out T> {
    data object Loading : Resource<Nothing>()
    data class Success<T>(val data: T) : Resource<T>()
    data class Error(val message: String) : Resource<Nothing>()
}
```

### networkBoundResource Pattern

A reusable strategy for cache-then-network loading:

```kotlin
inline fun <ResultType, RequestType> networkBoundResource(
    crossinline query: () -> Flow<ResultType>,
    crossinline fetch: suspend () -> RequestType,
    crossinline saveFetchResult: suspend (RequestType) -> Unit,
    crossinline shouldFetch: (ResultType) -> Boolean = { true }
): Flow<Resource<ResultType>> = flow {
    val data = query().first()

    val flow = if (shouldFetch(data)) {
        emit(Resource.Loading)
        try {
            saveFetchResult(fetch())
            query().map { Resource.Success(it) }
        } catch (e: Exception) {
            query().map { Resource.Error(e.message ?: "Unknown error") }
        }
    } else {
        query().map { Resource.Success(it) }
    }

    emitAll(flow)
}
```

## Repository Pattern

The repository mediates between data sources, exposing a clean API to the ViewModel:

```kotlin
class ArticleRepository(
    private val remoteDataSource: ArticleRemoteDataSource,
    private val localDataSource: ArticleLocalDataSource
) {
    fun observeArticles(): Flow<List<Article>> =
        localDataSource.observeArticles()

    suspend fun refreshArticles() {
        val articles = remoteDataSource.fetchArticles()
        localDataSource.cacheArticles(articles)
    }

    suspend fun getArticle(id: Long): Article {
        return localDataSource.getArticle(id)
            ?: remoteDataSource.fetchArticle(id).also {
                localDataSource.cacheArticle(it)
            }
    }
}
```

| Layer | Responsibility |
|-------|---------------|
| API (Retrofit) | HTTP requests, JSON parsing |
| DAO (Room) | Local SQL queries, reactive Flows |
| Repository | Coordinates remote and local, caching strategy |
| ViewModel | UI state, user actions, calls repository |
| Composable | Renders UI from ViewModel state |

## Key Takeaways

- `LazyColumn` and `LazyVerticalGrid` only compose visible items — always provide stable `key` values for correct diffing and animation
- Paging 3 handles incremental loading with `PagingSource`, `Pager`, and `collectAsLazyPagingItems()` — handle `LoadState` for loading, error, and empty states
- Retrofit with `kotlinx.serialization` provides type-safe networking — define API interfaces with suspend functions
- Room provides compile-time SQL verification, Flow-based reactivity, and automatic re-emission when data changes
- The offline-first pattern serves cached data immediately and refreshes from the network in the background, providing a responsive user experience
- The repository pattern abstracts data sources behind a single API, making ViewModels independent of data origin
