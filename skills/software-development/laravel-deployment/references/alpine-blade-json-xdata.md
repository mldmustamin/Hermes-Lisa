# Alpine.js + Blade: JSON in x-data Attributes

## Pitfall: Double Quotes Break x-data

When embedding JSON data inside Alpine.js `x-data="..."` using Blade's `{!! json_encode() !!}` or `@json()`, the JSON double quotes break the HTML attribute, causing the entire Alpine code to appear as raw text on the page.

### Wrong (breaks)
```blade
<form x-data="{ items: @json($items), ... }">
```
Output: `x-data="{ items: JSON.parse('[{\"id\":...}]'), ... }"` — broken HTML attribute

### Wrong (also breaks)
```blade
<form x-data="{ items: {!! json_encode($items) !!}, ... }">
```

### Right: Use Alpine.data() in a <script> tag
```blade
<form x-data="myForm()">
<script>
document.addEventListener('alpine:init', () => {
    Alpine.data('myForm', () => ({
        items: <?= json_encode($items) ?>,
        // ... methods and state
    }))
})
</script>
```

### Why
- `<?= json_encode($items) ?>` outputs raw JSON directly in JavaScript context — no quoting issues
- `Alpine.data()` registers reusable component state
- The `<script>` tag keeps JSON in proper JavaScript context
- `x-data="myForm()"` references the registered component by name

## Alternative: Pass Data via Server-Rendered Script
```blade
<script>
window.formData = @json($items);
</script>
<form x-data="{ items: window.formData, ... }">
```

## Note
`{{ Js::from($items) }}` wraps in `JSON.parse(...)` — useful for single values but adds overhead for large datasets. Direct `@json()` / `json_encode()` is cleaner for Alpine.js.
