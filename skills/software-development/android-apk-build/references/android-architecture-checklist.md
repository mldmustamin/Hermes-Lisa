# Official Android Architecture Compliance Checklist

**Source:** [Android Developers Guide to App Architecture](https://developer.android.com/topic/architecture?hl=id)
**Language:** Indonesian

## Pre-Coding Verification
Before writing ANY Android Kotlin/Compose code, verify against these 8 principles.

### 1. Separation of Concerns (Pemisahan Fokus)
- [ ] Activity NOT used for business logic (only UI container)
- [ ] ViewModel for state management, not Activity
- [ ] Repository for data access, never direct network calls from UI
- [ ] NO app data stored in Activity/Fragment (they get destroyed)

### 2. Single Source of Truth (SSOT)
- [ ] Room DB is the canonical data source (for offline-first apps)
- [ ] Network updates Room, Room updates UI
- [ ] UI NEVER reads directly from network
- [ ] ONE SSOT per data type

### 3. Unidirectional Data Flow (UDF)
- [ ] State flows DOWN via `StateFlow<UiState>`
- [ ] Events flow UP via function calls (`onEvent()`)
- [ ] UiState is immutable `data class` with `val` properties
- [ ] ViewModel uses `_uiState.update { it.copy(...) }` for mutations

### 4. ViewModel + StateFlow Pattern
- [ ] Single `UiState` data class per screen
- [ ] `StateFlow<UiState>` exposed from ViewModel
- [ ] `collectAsStateWithLifecycle()` in Compose (NOT `collectAsState()`)
- [ ] `viewModelScope.launch` for coroutines
- [ ] `hiltViewModel()` for navigation destination ViewModels

### 5. Repository Pattern
- [ ] Repository interface in domain layer
- [ ] Implementation in data layer
- [ ] LocalDataSource (Room) + RemoteDataSource (Ktor HTTP)
- [ ] Expose reactive data via `Flow<List<T>>`
- [ ] One-shot operations via `suspend` functions

### 6. Room Database
- [ ] Entity with `@Entity`, `@PrimaryKey`
- [ ] DAO with `@Dao`, Flow-based queries
- [ ] TypeConverters for enums
- [ ] Migration for version bumps
- [ ] Hilt DI for DAO providers

### 7. Dependency Injection (Hilt)
- [ ] `@HiltAndroidApp` on Application
- [ ] `@AndroidEntryPoint` on Activity
- [ ] `@HiltViewModel` on ViewModels
- [ ] `@Module` + `@InstallIn(SingletonComponent::class)` for providers

### 8. Compose Best Practices
- [ ] Stateless composable pattern: `fun MyScreen(state: UiState, onEvent: (Event) -> Unit)`
- [ ] State hoisting to lowest common ancestor
- [ ] `remember` only for UI-local state (expanded, text field value)
- [ ] `rememberSaveable` for configuration change survival
- [ ] No `var` definitions in top-level composable (hoist to ViewModel)

## Common Anti-Patterns to Reject

| Anti-Pattern | Correct Pattern |
|--------------|-----------------|
| Direct API call in Composable | ViewModel → Repository → API |
| Multiple `mutableStateOf` in ViewModel | Single `UiState` data class |
| `collectAsState()` | `collectAsStateWithLifecycle()` |
| Hardcoded values in Composables | Hoist to ViewModel UiState |
| Writing code before reading official docs | Research developer.android.com FIRST |

## Pre-Build Checklist
1. `./gradlew assembleDebug --no-daemon`
2. Fix ALL compilation errors before commit
3. Verify imports exist (don't reference non-existent classes)
4. Test APK size reasonable (<30MB for debug)
