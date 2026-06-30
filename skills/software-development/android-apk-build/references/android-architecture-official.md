# Android Architecture Best Practices — Official Reference

**Source:** [developer.android.com/topic/architecture](https://developer.android.com/topic/architecture)

## 5 Core Principles (Google Official)

1. **Pemisahan Fokus (Separation of Concerns)** — UI hanya container, jangan simpan state di Activity/Fragment
2. **Menjalankan UI dari Model Data** — Drive UI from persistent models (Room DB)
3. **Sumber Ketetapan Tunggal (SSOT)** — Room DB = canonical source, network hanya update SSOT
4. **Aliran Data Searah (UDF)** — State flow DOWN (StateFlow), events flow UP (function calls)
5. **Lapisan Arsitektur** — UI Layer (Compose + ViewModel) → Domain Layer (UseCases, optional) → Data Layer (Repository + Room + API)

## Compose Screen Pattern (Every Screen Must Follow)

```kotlin
// 1. UiState — immutable data class
data class MyUiState(val items: List<Item> = emptyList(), val isLoading: Boolean = true)

// 2. ViewModel — state holder, @HiltViewModel
@HiltViewModel
class MyViewModel @Inject constructor(private val repo: Repository) : ViewModel() {
    private val _uiState = MutableStateFlow(MyUiState())
    val uiState: StateFlow<MyUiState> = _uiState.asStateFlow()
}

// 3. Screen — stateless composable
@Composable
fun MyScreen(vm: MyViewModel = hiltViewModel()) {
    val uiState by vm.uiState.collectAsStateWithLifecycle()
    Scaffold { ... }
}
```

## Key Rules

| ✅ MUST | ❌ MUST NOT |
|---------|------------|
| `collectAsStateWithLifecycle()` | `collectAsState()` without lifecycle |
| Single `UiState` data class per screen | Multiple `mutableStateOf` scattered |
| ViewModel holds all business logic | API calls directly in Composable |
| `StateFlow<UiState>` | `LiveData` (legacy) |
| `ExposedDropdownMenuBox` for dropdowns | Custom dropdown wrappers |
| `NumberFormat.getNumberInstance(Locale("id","ID"))` for Rupiah | Hardcoded formatting |

## Anti-Patterns That Break Builds

- Importing custom components that don't exist (`MoneyInputField`, `AppDropdownField`) → use `OutlinedTextField` + `ExposedDropdownMenuBox`
- Importing `UiFormatters` that doesn't exist → use `java.text.NumberFormat` directly
- Missing `kotlinx.coroutines.launch` import in ViewModel → always import explicitly
- Entity classes referenced without importing package (`BudgetTemplateEntity`, `MasterLocationEntity`, `TaskExpenseEntity`)

## Offline-First Pattern (Already in FundManager V2)

- Room DB = SSOT with Flow-based reactive reads
- WorkManager periodic sync (15 min) + one-time sync after login
- Outbox pattern for offline writes (`SyncOutboxRepository`)
- Repository = cache-first, network refreshes update Room → Flow auto-emits to UI
