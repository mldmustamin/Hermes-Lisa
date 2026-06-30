# IDR Money Input Format — Web + Android

## Web (Blade + Alpine.js)

### Component: `<x-money-input>`
Blade component at `resources/views/components/money-input.blade.php`:
- Accepts `name`, `label`, `value` props
- Shows "Rp" prefix
- Auto-formats: `40000` → `40,000` while typing
- Hidden input stores raw integer value for form submission

### Inline Alpine.js (for dynamic item templates)
```html
<input type="text" inputmode="numeric"
   x-model="amount"
   x-on:input="let n=$el.value.replace(/[^0-9]/g,''); amount=n; $el.value=n?parseInt(n).toLocaleString('id-ID'):''"
   placeholder="Estimasi (Rp)"
   class="...text-right...">
```

### Backend: Strip Commas Before Validation
```php
// In controller store/update methods:
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
This converts `"Rp 1,000,000"` → `"1000000"` before validation.

### Display (read-only)
```blade
Rp {{ number_format($value, 0, ',', '.') }}
```

## Android (Kotlin/Compose)

### Input field with formatting
```kotlin
var amountText by remember { mutableStateOf("") }

OutlinedTextField(
    value = amountText,
    onValueChange = { raw ->
        val digits = raw.replace(Regex("[^0-9]"), "")
        amountText = if (digits.isEmpty()) "" 
            else NumberFormat.getNumberInstance(Locale("id", "ID")).format(digits.toLong())
        // Store raw value separately
        onAmountChanged(digits.toLongOrNull() ?: 0)
    },
    keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Number),
    ...
)
```

### Display
```kotlin
val fmt = remember { NumberFormat.getNumberInstance(Locale("id", "ID")) }
Text("Rp ${fmt.format(amount)}")
```

## Key Rules
- Always use `toLocaleString('id-ID')` for Indonesian format (not `en-US`)
- Strip non-digits before sending to backend
- Display raw numbers with `number_format(value, 0, ',', '.')` in Blade
