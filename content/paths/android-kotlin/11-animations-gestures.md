---
title: "Animations and Gestures"
weight: 11
---

# Animations and Gestures

Compose provides a layered animation system ranging from simple one-shot animations to complex choreographed transitions. Gestures are handled declaratively through modifier functions, making touch interactions composable and testable.

## animate*AsState

The simplest animation API. It animates a single value toward a target, recomposing smoothly as the value transitions:

```kotlin
@Composable
fun ExpandableCard(expanded: Boolean) {
    val height by animateDpAsState(
        targetValue = if (expanded) 200.dp else 80.dp,
        animationSpec = tween(durationMillis = 300),
        label = "card height"
    )
    val alpha by animateFloatAsState(
        targetValue = if (expanded) 1f else 0.6f,
        animationSpec = tween(durationMillis = 300),
        label = "card alpha"
    )
    val backgroundColor by animateColorAsState(
        targetValue = if (expanded) MaterialTheme.colorScheme.primaryContainer
            else MaterialTheme.colorScheme.surface,
        label = "card background"
    )

    Card(
        modifier = Modifier
            .fillMaxWidth()
            .height(height)
            .alpha(alpha),
        colors = CardDefaults.cardColors(containerColor = backgroundColor)
    ) {
        // Content
    }
}
```

### Available animate*AsState Functions

| Function | Animates |
|----------|----------|
| `animateFloatAsState` | Float values (alpha, scale, rotation) |
| `animateDpAsState` | Dp values (size, padding, offset) |
| `animateColorAsState` | Color transitions |
| `animateIntAsState` | Integer values |
| `animateOffsetAsState` | Offset (x, y) position |
| `animateSizeAsState` | Size (width, height) |
| `animateIntOffsetAsState` | IntOffset position |

### Animation Specs

Animation specs control the timing and easing of animations:

```kotlin
// Tween — duration-based with easing
tween<Float>(durationMillis = 500, easing = FastOutSlowInEasing)

// Spring — physics-based
spring<Float>(dampingRatio = Spring.DampingRatioMediumBouncy, stiffness = Spring.StiffnessLow)

// Keyframes — multi-step
keyframes<Float> {
    durationMillis = 1000
    0f at 0 using LinearEasing
    0.5f at 500 using FastOutSlowInEasing
    1f at 1000
}

// Repeatable
repeatable<Float>(iterations = 3, animation = tween(300), repeatMode = RepeatMode.Reverse)

// Infinite repeating
infiniteRepeatable<Float>(animation = tween(1000), repeatMode = RepeatMode.Reverse)

// Snap — instant, no animation
snap<Float>(delayMillis = 100)
```

| Spec | Use case |
|------|----------|
| `tween` | Fixed-duration transitions (most common) |
| `spring` | Natural-feeling motion (bouncy, snappy) |
| `keyframes` | Multi-step choreographed animations |
| `repeatable` | Finite looping (pulse, shake) |
| `infiniteRepeatable` | Continuous animation (loading spinner, heartbeat) |
| `snap` | Instant change with optional delay |

## AnimatedVisibility

Controls the appearance and disappearance of content with enter and exit animations:

```kotlin
@Composable
fun NotificationBanner(visible: Boolean, message: String) {
    AnimatedVisibility(
        visible = visible,
        enter = slideInVertically(initialOffsetY = { -it }) + fadeIn(),
        exit = slideOutVertically(targetOffsetY = { -it }) + fadeOut()
    ) {
        Surface(
            color = MaterialTheme.colorScheme.primaryContainer,
            modifier = Modifier.fillMaxWidth()
        ) {
            Text(
                text = message,
                modifier = Modifier.padding(16.dp),
                style = MaterialTheme.typography.bodyMedium
            )
        }
    }
}
```

### Enter and Exit Transitions

Enter and exit transitions can be combined with `+`:

| Enter | Exit | Effect |
|-------|------|--------|
| `fadeIn()` | `fadeOut()` | Opacity change |
| `slideInVertically()` | `slideOutVertically()` | Vertical slide |
| `slideInHorizontally()` | `slideOutHorizontally()` | Horizontal slide |
| `expandVertically()` | `shrinkVertically()` | Expand/collapse height |
| `expandHorizontally()` | `shrinkHorizontally()` | Expand/collapse width |
| `expandIn()` | `shrinkOut()` | Expand/shrink in both dimensions |
| `scaleIn()` | `scaleOut()` | Scale from/to zero |

```kotlin
// Combine transitions
enter = fadeIn(tween(300)) + expandVertically(expandFrom = Alignment.Top)
exit = fadeOut(tween(200)) + shrinkVertically(shrinkTowards = Alignment.Top)
```

## Crossfade

Crossfade transitions between composable content based on a target state:

```kotlin
@Composable
fun ContentSwitcher(currentTab: Tab) {
    Crossfade(
        targetState = currentTab,
        animationSpec = tween(400),
        label = "tab crossfade"
    ) { tab ->
        when (tab) {
            Tab.HOME -> HomeContent()
            Tab.SEARCH -> SearchContent()
            Tab.PROFILE -> ProfileContent()
        }
    }
}
```

### AnimatedContent

For more control over the transition between content states, use `AnimatedContent`:

```kotlin
@Composable
fun CounterDisplay(count: Int) {
    AnimatedContent(
        targetState = count,
        transitionSpec = {
            if (targetState > initialState) {
                // Counting up — slide up
                slideInVertically { it } + fadeIn() togetherWith
                    slideOutVertically { -it } + fadeOut()
            } else {
                // Counting down — slide down
                slideInVertically { -it } + fadeIn() togetherWith
                    slideOutVertically { it } + fadeOut()
            }.using(SizeTransform(clip = false))
        },
        label = "counter"
    ) { targetCount ->
        Text(
            text = "$targetCount",
            style = MaterialTheme.typography.displayLarge
        )
    }
}
```

## updateTransition

`updateTransition` manages multiple animated values that should change together, driven by a single state:

```kotlin
enum class CardState { Collapsed, Expanded }

@Composable
fun AnimatedCard(cardState: CardState) {
    val transition = updateTransition(targetState = cardState, label = "card")

    val height by transition.animateDp(
        transitionSpec = { spring(stiffness = Spring.StiffnessLow) },
        label = "height"
    ) { state ->
        when (state) {
            CardState.Collapsed -> 80.dp
            CardState.Expanded -> 240.dp
        }
    }

    val cornerRadius by transition.animateDp(
        transitionSpec = { tween(300) },
        label = "corner"
    ) { state ->
        when (state) {
            CardState.Collapsed -> 16.dp
            CardState.Expanded -> 8.dp
        }
    }

    val elevation by transition.animateDp(
        transitionSpec = { tween(300) },
        label = "elevation"
    ) { state ->
        when (state) {
            CardState.Collapsed -> 2.dp
            CardState.Expanded -> 8.dp
        }
    }

    Card(
        modifier = Modifier
            .fillMaxWidth()
            .height(height),
        shape = RoundedCornerShape(cornerRadius),
        elevation = CardDefaults.cardElevation(defaultElevation = elevation)
    ) {
        // Content
    }
}
```

### InfiniteTransition

For animations that run indefinitely (loading indicators, pulsing elements):

```kotlin
@Composable
fun PulsingDot() {
    val infiniteTransition = rememberInfiniteTransition(label = "pulse")

    val scale by infiniteTransition.animateFloat(
        initialValue = 0.8f,
        targetValue = 1.2f,
        animationSpec = infiniteRepeatable(
            animation = tween(800, easing = FastOutSlowInEasing),
            repeatMode = RepeatMode.Reverse
        ),
        label = "scale"
    )

    val alpha by infiniteTransition.animateFloat(
        initialValue = 0.5f,
        targetValue = 1f,
        animationSpec = infiniteRepeatable(
            animation = tween(800),
            repeatMode = RepeatMode.Reverse
        ),
        label = "alpha"
    )

    Box(
        modifier = Modifier
            .size(24.dp)
            .scale(scale)
            .alpha(alpha)
            .background(MaterialTheme.colorScheme.primary, CircleShape)
    )
}
```

## Gesture Detection

Compose handles gestures through modifier functions and the `pointerInput` modifier.

### detectTapGestures

```kotlin
@Composable
fun TapExample() {
    var lastAction by remember { mutableStateOf("None") }

    Box(
        modifier = Modifier
            .size(200.dp)
            .background(Color.LightGray)
            .pointerInput(Unit) {
                detectTapGestures(
                    onTap = { offset -> lastAction = "Tap at $offset" },
                    onDoubleTap = { lastAction = "Double tap" },
                    onLongPress = { lastAction = "Long press" },
                    onPress = { /* called immediately on press */ }
                )
            },
        contentAlignment = Alignment.Center
    ) {
        Text(lastAction)
    }
}
```

### detectDragGestures

```kotlin
@Composable
fun DraggableBox() {
    var offset by remember { mutableStateOf(Offset.Zero) }

    Box(modifier = Modifier.fillMaxSize()) {
        Box(
            modifier = Modifier
                .offset { IntOffset(offset.x.roundToInt(), offset.y.roundToInt()) }
                .size(80.dp)
                .background(MaterialTheme.colorScheme.primary, RoundedCornerShape(16.dp))
                .pointerInput(Unit) {
                    detectDragGestures { change, dragAmount ->
                        change.consume()
                        offset += dragAmount
                    }
                }
        )
    }
}
```

### Swipe to Dismiss

```kotlin
@Composable
fun SwipeableItem(
    onDismiss: () -> Unit,
    content: @Composable () -> Unit
) {
    val dismissState = rememberSwipeToDismissBoxState(
        confirmValueChange = { value ->
            if (value == SwipeToDismissBoxValue.EndToStart) {
                onDismiss()
                true
            } else false
        }
    )

    SwipeToDismissBox(
        state = dismissState,
        backgroundContent = {
            Box(
                modifier = Modifier
                    .fillMaxSize()
                    .background(Color.Red)
                    .padding(16.dp),
                contentAlignment = Alignment.CenterEnd
            ) {
                Icon(Icons.Default.Delete, contentDescription = "Delete", tint = Color.White)
            }
        }
    ) {
        content()
    }
}
```

### Gesture Modifier Summary

| Modifier / Detector | Gesture |
|---------------------|---------|
| `clickable { }` | Simple tap |
| `combinedClickable` | Tap, double-tap, long press |
| `detectTapGestures` | Tap, double-tap, long press, press with offset |
| `detectDragGestures` | Free-form drag |
| `detectHorizontalDragGestures` | Horizontal drag only |
| `detectVerticalDragGestures` | Vertical drag only |
| `detectTransformGestures` | Pinch, zoom, rotate |
| `scrollable` | Scroll with fling |
| `draggable` | One-axis drag with state |

## Shared Element Transitions

Shared element transitions animate elements smoothly between screens (available from Compose 1.7+):

```kotlin
@Composable
fun ListScreen(
    items: List<Item>,
    sharedTransitionScope: SharedTransitionScope,
    animatedContentScope: AnimatedContentScope,
    onItemClick: (Item) -> Unit
) {
    with(sharedTransitionScope) {
        LazyColumn {
            items(items, key = { it.id }) { item ->
                Row(
                    modifier = Modifier
                        .clickable { onItemClick(item) }
                        .padding(16.dp)
                ) {
                    AsyncImage(
                        model = item.imageUrl,
                        contentDescription = null,
                        modifier = Modifier
                            .size(60.dp)
                            .sharedElement(
                                state = rememberSharedContentState(key = "image-${item.id}"),
                                animatedVisibilityScope = animatedContentScope
                            )
                    )
                    Text(
                        text = item.title,
                        modifier = Modifier
                            .sharedElement(
                                state = rememberSharedContentState(key = "title-${item.id}"),
                                animatedVisibilityScope = animatedContentScope
                            )
                    )
                }
            }
        }
    }
}
```

## Key Takeaways

- `animate*AsState` is the simplest animation API — use it for single-value transitions driven by state changes
- Animation specs (`tween`, `spring`, `keyframes`) control timing and feel — `spring` gives natural motion, `tween` gives precise control
- `AnimatedVisibility` handles enter/exit transitions; `AnimatedContent` handles transitions between different content states
- `updateTransition` coordinates multiple animated values driven by a single state change — use it when multiple properties must animate together
- Gestures are handled through `pointerInput` with detectors (`detectTapGestures`, `detectDragGestures`, `detectTransformGestures`) — they compose cleanly with other modifiers
- Shared element transitions animate elements between screens for visual continuity — use `sharedElement` modifier with `SharedTransitionScope`
