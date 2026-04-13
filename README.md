# Laravel Dynamic Forms

## JSON-Driven Fields & Dual-Side Validation

## Architecture Overview

### The approach has three parts: parse JSON to build HTML form fields dynamically, validate on the server

### using Laravel's validator with rules derived from the same JSON, and validate on the client before

### submission.

(^1) JSON config — fields + rules
(^2) Controller — parses schema, passes to view
(^3) Blade view — renders fields dynamically, embeds rules as data-attributes
(^4) Client-side JS — reads data-rules, validates before POST
(^5) FormRequest / Controller — server validates; errors redirect back

## Step 1 — The JSON Schema

### Store this in storage/app/form_schema.json or in a database column.

##### {

```
"fields": [
{
"name": "full_name",
"label": "Full Name",
"type": "text",
"rules": ["required", "min:3", "max:100"]
},
{
"name": "email",
"label": "Email Address",
"type": "email",
"rules": ["required", "email"]
},
```

##### {

```
"name": "age",
"label": "Age",
"type": "number",
"rules": ["required", "integer", "min:18", "max:120"]
},
{
"name": "gender",
"label": "Gender",
"type": "select",
"options": ["Male", "Female", "Other"],
"rules": ["required", "in:Male,Female,Other"]
},
{
"name": "bio",
"label": "Bio",
"type": "textarea",
"rules": ["nullable", "max:500"]
}
]
}
```
## Step 2 — The Controller

### The controller loads the JSON schema, passes it to the view, and on POST dynamically builds Laravel

### validation rules from the same schema.

```
// app/Http/Controllers/DynamicFormController.php
```
```
namespace App\Http\Controllers;
```
```
use Illuminate\Http\Request;
```

use Illuminate\Support\Facades\Storage;

class DynamicFormController extends Controller
{
private function getSchema(): array
{
$json = Storage::get('form_schema.json');
return json_decode($json, true);
}

public function show()
{
$schema = $this->getSchema();
return view('dynamic-form', compact('schema'));
}

public function store(Request $request)
{
$schema = $this->getSchema();

// Build Laravel validation rules from JSON
$rules = [];
$labels = [];
foreach ($schema['fields'] as $field) {
$rules[$field['name']] = $field['rules'];
$labels[$field['name']] = $field['label'];
}

$validated = $request->validate($rules, [], $labels);

// Process $validated data...


```
return redirect()->back()->with('success', 'Form submitted!');
}
}
```
## Step 3 — The Blade View

### The view loops over the schema fields and renders each one. Rules are embedded as a data-rules

### attribute so the JS validator can read them without a second server call.

```
{{-- resources/views/dynamic-form.blade.php --}}
```
```
@if(session('success'))
&lt;div class="alert alert-success"&gt;{{ session('success') }}&lt;/div&gt;
@endif
```
```
&lt;form id="dynamic-form" method="POST" action="{{ route('form.store') }}"&gt;
@csrf
```
```
@foreach($schema['fields'] as $field)
&lt;div class="mb-3"&gt;
&lt;label&gt;{{ $field['label'] }}&lt;/label&gt;
```
```
@switch($field['type'])
@case('textarea')
&lt;textarea
name="{{ $field['name'] }}"
data-rules="{{ json_encode($field['rules']) }}"
class="form-control @error($field['name']) is-invalid @enderror"
&gt;{{ old($field['name']) }}&lt;/textarea&gt;
@break
```
```
@case('select')
&lt;select
```

name="{{ $field['name'] }}"
data-rules="{{ json_encode($field['rules']) }}"
class="form-control @error($field['name']) is-invalid @enderror"
&gt;
&lt;option value=""&gt;-- Select --&lt;/option&gt;
@foreach($field['options'] as $opt)
&lt;option value="{{ $opt }}"
{{ old($field['name']) == $opt? 'selected' : '' }}&gt;
{{ $opt }}
&lt;/option&gt;
@endforeach
&lt;/select&gt;
@break

@default
&lt;input
type="{{ $field['type'] }}"
name="{{ $field['name'] }}"
value="{{ old($field['name']) }}"
data-rules="{{ json_encode($field['rules']) }}"
class="form-control @error($field['name']) is-invalid @enderror"
&gt;
@endswitch

@error($field['name'])
&lt;div class="invalid-feedback"&gt;{{ $message }}&lt;/div&gt;
@enderror
&lt;/div&gt;
@endforeach


```
&lt;button type="submit"&gt;Submit&lt;/button&gt;
&lt;/form&gt;
```
## Step 4 — Client-Side Validation (JavaScript)

### This reads the data-rules attribute embedded from JSON and validates before the form posts.

```
document.getElementById('dynamic-form').addEventListener('submit', function (e) {
let isValid = true;
```
```
// Clear previous client errors
document.querySelectorAll('.client-error').forEach(el => el.remove());
document.querySelectorAll('.is-invalid').forEach(el => {
el.classList.remove('is-invalid');
});
```
```
document.querySelectorAll('[data-rules]').forEach(function (field) {
const rules = JSON.parse(field.dataset.rules);
const value = field.value.trim();
let error = null;
```
```
for (const rule of rules) {
if (rule === 'required' && !value) {
error = `${field.name} is required.`; break;
}
if (rule === 'email' && value &&
!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value)) {
error = 'Enter a valid email address.'; break;
}
if (rule.startsWith('min:')) {
const min = parseInt(rule.split(':')[1]);
```

if (field.type === 'number' && value && parseInt(value) < min) {
error = `Minimum value is ${min}.`; break;
}
if (field.type !== 'number' && value.length < min) {
error = `Minimum ${min} characters required.`; break;
}
}
if (rule.startsWith('max:')) {
const max = parseInt(rule.split(':')[1]);
if (field.type === 'number' && value && parseInt(value) > max) {
error = `Maximum value is ${max}.`; break;
}
if (field.type !== 'number' && value.length > max) {
error = `Maximum ${max} characters allowed.`; break;
}
}
if (rule.startsWith('in:')) {
const allowed = rule.split(':')[1].split(',');
if (value && !allowed.includes(value)) {
error = 'Invalid selection.'; break;
}
}
}

if (error) {
isValid = false;
field.classList.add('is-invalid');
const msg = document.createElement('div');


```
msg.className = 'invalid-feedback client-error';
msg.textContent = error;
field.insertAdjacentElement('afterend', msg);
}
});
```
```
if (!isValid) e.preventDefault();
});
```
## Step 5 — Routes

```
// routes/web.php
```
```
use App\Http\Controllers\DynamicFormController;
```
```
Route::get('/form', [DynamicFormController::class, 'show'])->name('form.show');
Route::post('/form', [DynamicFormController::class, 'store'])->name('form.store');
```
## Key Points to Remember

### Server validationalways wins Client-side JS is a convenience — never rely on it alone. Laravel's validate() will catchanything bypassed or submitted directly.

### Use a FormRequestclass Create one with php artisan make:request DynamicFormRequest and override rules() to loadfrom JSON. Cleaner for larger forms.

### old() helper Repopulates fields after failed server validation — important for good UX. Always useold($field['name']) as the field value.

### Sanitize your JSON If the schema comes from user input (e.g. an admin panel), never pass unvalidated rulestrings into Laravel's validator. Rules like exists:users,id could probe your database.

## Supported JSON Rules — Quick Reference

### Rule Client-side Server-side (Laravel)

#### required Checks non-empty value required

#### email Regex format check email

#### min:N Length or numeric min min:N

#### max:N Length or numeric max max:N


#### integer

#### Not implemented

#### (pass-through) integer

#### in:a,b,c Checks against allowed list in:a,b,c

#### nullable Skips all checks if empty nullable

```
Laravel Dynamic Forms Guide • Generated by Claude
```

