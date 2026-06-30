# Laravel Blade + Alpine.js Form Patterns

## Money Input Formatting (IDR — Rp)

User expects Indonesian-style money input: `40,000`, `100,000`, `1,000,000`.

### Alpine.js Pattern

```html
<input type="text" inputmode="numeric"
       x-model="amount"
       x-on:input="amount = $el.value.replace(/[^0-9]/g,''); $el.value = amount ? parseInt(amount).toLocaleString('id-ID') : ''"
       placeholder="0">
<input type="hidden" name="field_name" x-bind:value="amount">
```

### Backend: Strip Commas Before Validation

```php
// In controller store method, before validation:
if ($request->has('items')) {
    $items = $request->input('items');
    foreach ($items as $i => $item) {
        if (isset($item['estimated_amount'])) {
            $items[$i]['estimated_amount'] = str_replace(
                ['Rp', '.', ',', ' '], '', $item['estimated_amount']
            );
        }
    }
    $request->merge(['items' => $items]);
}
```

## Searchable Dropdown (Alpine.js)

For 35+ options, a native `<select>` is too slow. Use search + filtered dropdown:

```html
<div x-data="{ open: false, search: '', selectedId: null, selected: '' }" @click.away="open = false">
    <input type="text" x-model="search" @focus="open = true"
           :placeholder="selected || 'Cari kategori...'">
    <input type="hidden" name="field" x-bind:value="selectedId">

    <div x-show="open" class="absolute z-50 bg-white border rounded shadow max-h-48 overflow-y-auto">
        <template x-for="opt in allOptions.filter(o => o.name.toLowerCase().includes(search.toLowerCase()))">
            <div @click="selectedId = opt.id; selected = opt.name; open = false; search = ''">
                <span x-text="opt.name"></span>
            </div>
        </template>
    </div>
</div>
```

## Blade + Alpine.js Template Loops

When using `json_encode` inside Alpine.js x-data, avoid `@json()` for inline arrays. Use `{!! json_encode() !!}` instead:

```blade
<!-- ❌ BROKEN - Blade confuses @json([...]) -->
x-data="{ items: @json($items->toArray()) }"

<!-- ✅ CORRECT -->
x-data="{ items: {!! json_encode($items->toArray()) !!} }"
```

## Auto-Calculation (Hotel = rate × days)

```html
<div x-data="{
    qty: 1,
    ratePerUnit: 200000,
    get amount() { return this.ratePerUnit * this.qty; },
    get isHotel() { return true; } // detect from selected template
}">
    <input x-show="isHotel" type="number" x-model.number="qty" min="1">
    <input type="text" x-model="amount" :readonly="isHotel"
           x-on:input="$el.value = parseInt(amount).toLocaleString('id-ID')">
</div>
```

## Pitfalls

| Issue | Cause | Fix |
|-------|-------|-----|
| `@json([...])` syntax error | Blade parser can't handle `@json(array)` with inline arrays | Always use `{!! json_encode() !!}` |
| Money input sends "1,000,000" to server | Frontend formatted, backend expects integer | Strip commas in controller before validation |
| Alpine scope issues in nested templates | Child `x-data` can't access parent methods directly | Use `$parent.method()` for parent access |
| Search dropdown not filtering | `allTemplates` loaded as string, not parsed | Verify `json_encode` output is valid JS array literal |
