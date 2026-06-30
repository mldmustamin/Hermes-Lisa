# Pre-Implementation Research — Android Official Docs

## User Mandate
> "Gunakan semua sumberdaya, jangan malas buat research."
> "Pelajari dan pahami semua dokumentasi Android dan library Jetpack. Itu akan sangat membantu."

## Mandatory Research Before Android Coding

Before writing ANY Kotlin/Compose code, consult these official sources (Indonesian versions preferred when available):

### Architecture
- [Panduan arsitektur aplikasi](https://developer.android.com/topic/architecture?hl=id)
- [Lapisan UI](https://developer.android.com/topic/architecture/ui-layer?hl=id)
- [Lapisan Data](https://developer.android.com/topic/architecture/data-layer?hl=id)
- [Lapisan Domain](https://developer.android.com/topic/architecture/domain-layer?hl=id)
- [Offline-first](https://developer.android.com/topic/architecture/data-layer/offline-first?hl=id)
- [Rekomendasi arsitektur](https://developer.android.com/topic/architecture/recommendations?hl=id)

### Compose
- [State dan Jetpack Compose](https://developer.android.com/develop/ui/compose/state?hl=id)
- [State Hoisting](https://developer.android.com/develop/ui/compose/state-hoisting?hl=id)
- [Arsitektur UI Compose](https://developer.android.com/develop/ui/compose/architecture?hl=id)

### Jetpack
- [Jetpack overview](https://developer.android.com/jetpack?hl=id)
- [AndroidX Explorer](https://developer.android.com/jetpack/androidx/explorer?hl=id)
- [Navigation](https://developer.android.com/guide/navigation?hl=id)

## Key Patterns to Internalize

1. **Single Activity + Compose Navigation** — official recommendation
2. **ViewModel + StateFlow + UiState** — standard pattern per screen
3. **collectAsStateWithLifecycle()** — NOT collectAsState()
4. **State hoisting**: hoist to lowest common ancestor (ViewModel for business logic)
5. **UDF**: state DOWN via StateFlow, events UP via function calls
6. **Room as SSOT**: UI reads Room (never network directly)
7. **Repository pattern**: cache-first, sync in background
8. **Hilt DI**: @HiltViewModel + @Inject constructor

## Anti-Patterns to Avoid
- API calls inside Composable → move to ViewModel/Repository
- collectAsState() without lifecycle → use collectAsStateWithLifecycle()
- Multiple mutableStateOf in ViewModel → single UiState data class
- Custom UI components reinventing Material3 → use standard Composables
- Skipping docs and coding from memory → patterns change across versions
