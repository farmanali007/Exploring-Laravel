# Senior Laravel Validation Masterclass

_From Mid-Level to Senior-Level Understanding_

## Table of Contents

1. [Core Concepts: Under the Hood](#1-core-concepts-under-the-hood)
2. [Built-in Rules Deep Dive](#2-built-in-rules-deep-dive)
3. [Complex Scenarios](#3-complex-scenarios)
4. [Custom Rules Mastery](#4-custom-rules-mastery)
5. [Error Handling & UX](#5-error-handling--ux)
6. [Performance & Scaling](#6-performance--scaling)
7. [Security Aspects](#7-security-aspects)
8. [Best Practices](#8-best-practices)
9. [Common Mistakes](#9-common-mistakes)
10. [Laravel 10/11+ Updates](#10-laravel-1011-updates)

---

## 1. Core Concepts: Under the Hood

### How Laravel Validation Actually Works

**Mid-level thinking:** "I use `$request->validate()` and it works"
**Senior-level thinking:** "I understand the Validator lifecycle and can extend it"

```php
// What happens when you do this:
$request->validate(['email' => 'required|email']);

// Laravel actually does:
$validator = Validator::make($request->all(), ['email' => 'required|email']);
if ($validator->fails()) {
    throw new ValidationException($validator);
}
```

### The Validator Class Ecosystem

```php
use Illuminate\Validation\Validator;
use Illuminate\Validation\Factory as ValidationFactory;

// The ValidationFactory creates Validator instances
$factory = app(ValidationFactory::class);
$validator = $factory->make($data, $rules, $messages, $attributes);

// Each Validator instance has:
// - Rules collection
// - Data being validated
// - Error message bag
// - Custom validators/extensions
```

### Three Validation Approaches Compared

#### 1. Inline Validation (Controller)

```php
// ❌ Bad: Mixed concerns, hard to test, not reusable
public function store(Request $request)
{
    $validated = $request->validate([
        'name' => 'required|string|max:255',
        'email' => 'required|email|unique:users',
        'password' => 'required|min:8|confirmed'
    ]);

    User::create($validated);
}
```

#### 2. Form Request Classes

```php
// ✅ Good: Separated concerns, reusable, testable
class CreateUserRequest extends FormRequest
{
    public function authorize(): bool
    {
        return auth()->user()->can('create-users');
    }

    public function rules(): array
    {
        return [
            'name' => 'required|string|max:255',
            'email' => 'required|email|unique:users',
            'password' => 'required|min:8|confirmed'
        ];
    }

    // Senior tip: Override prepareForValidation for data transformation
    protected function prepareForValidation(): void
    {
        $this->merge([
            'slug' => Str::slug($this->name),
            'email' => strtolower($this->email)
        ]);
    }
}

// Controller becomes clean
public function store(CreateUserRequest $request)
{
    User::create($request->validated());
}
```

#### 3. Manual Validator Creation

```php
// ✅ Best for complex logic, services, or when you need full control
class UserService
{
    public function createUser(array $data): User
    {
        $validator = Validator::make($data, [
            'name' => 'required|string|max:255',
            'email' => 'required|email|unique:users'
        ]);

        // Add conditional rules based on business logic
        if ($this->isEnterpriseUser($data)) {
            $validator->addRules(['company_id' => 'required|exists:companies,id']);
        }

        if ($validator->fails()) {
            throw new ValidationException($validator);
        }

        return User::create($validator->validated());
    }
}
```

**Senior Insight:** Choose Form Requests for HTTP validation, manual Validator for service layer validation.

---

## 2. Built-in Rules Deep Dive

### Advanced Conditional Rules

#### `sometimes` - The Misunderstood Rule

```php
// ❌ Common misconception: "sometimes means optional"
'field' => 'sometimes|required|email'  // This is redundant!

// ✅ Correct usage: "Apply these rules only if field is present"
'optional_bio' => 'sometimes|string|max:1000',
'api_key' => 'sometimes|required|alpha_num|size:32'  // If provided, must be valid
```

#### Conditional Rules Mastery

```php
// Real-world example: User profile update
public function rules(): array
{
    return [
        'email' => [
            'required',
            'email',
            Rule::unique('users')->ignore($this->user()->id)
        ],

        // Only validate password if provided (for updates)
        'password' => 'sometimes|min:8|confirmed',

        // Require company details for business accounts
        'company_name' => 'required_if:account_type,business',
        'tax_id' => 'required_with:company_name|nullable|string',

        // Exclude certain fields based on conditions
        'personal_info' => 'exclude_if:account_type,business',

        // Prohibit fields in certain scenarios
        'admin_notes' => 'prohibited_unless:role,admin'
    ];
}
```

### Database Rules: `exists` and `unique` Deep Dive

#### Advanced `exists` Usage

```php
// ❌ Basic usage most developers know
'user_id' => 'exists:users,id'

// ✅ Senior-level usage with complex conditions
'user_id' => [
    'required',
    Rule::exists('users', 'id')->where(function ($query) {
        $query->where('active', true)
              ->where('organization_id', auth()->user()->organization_id);
    })
],

// Multiple column validation
'role_permission' => [
    Rule::exists('role_permissions')->where(function ($query) {
        $query->where('role_id', $this->input('role_id'))
              ->where('permission_id', $this->input('permission_id'));
    })
]
```

#### Mastering `unique` with Edge Cases

```php
// Real-world scenario: Handling soft deletes and complex uniqueness
public function rules(): array
{
    return [
        'email' => [
            'required',
            'email',
            Rule::unique('users')
                ->ignore($this->route('user')?->id)
                ->where('organization_id', $this->input('organization_id'))
                ->whereNull('deleted_at')  // Handle soft deletes
        ],

        // Composite uniqueness
        'sku' => [
            Rule::unique('products')
                ->where('category_id', $this->input('category_id'))
                ->where('brand_id', $this->input('brand_id'))
        ]
    ];
}
```

**Senior Tip:** Always consider soft deletes, multi-tenancy, and composite keys when using database rules.

---

## 3. Complex Scenarios

### Array and Nested Validation Mastery

#### Basic Array Validation

```php
// ❌ Beginner approach - missing edge cases
'tags' => 'array',
'tags.*' => 'string'

// ✅ Senior approach - comprehensive validation
'tags' => 'required|array|min:1|max:10',
'tags.*' => 'required|string|max:50|distinct',

// Nested objects
'addresses' => 'required|array|min:1',
'addresses.*.type' => 'required|in:home,work,billing',
'addresses.*.street' => 'required|string|max:255',
'addresses.*.city' => 'required|string|max:100',
'addresses.*.postal_code' => [
    'required',
    'string',
    function ($attribute, $value, $fail) {
        // Custom validation for postal code based on country
        $index = explode('.', $attribute)[1];
        $country = request()->input("addresses.{$index}.country");

        if (!$this->isValidPostalCodeForCountry($value, $country)) {
            $fail("Invalid postal code for {$country}");
        }
    }
]
```

#### Dynamic Array Validation

```php
// Real-world: E-commerce cart validation
class CartValidationRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'items' => 'required|array|min:1',
            'items.*.product_id' => 'required|exists:products,id',
            'items.*.quantity' => 'required|integer|min:1',
            'items.*.variant_id' => 'nullable|exists:product_variants,id',
        ];
    }

    public function withValidator($validator): void
    {
        $validator->after(function ($validator) {
            // Cross-item validation
            foreach ($this->input('items', []) as $index => $item) {
                $product = Product::find($item['product_id']);

                if ($product && $item['quantity'] > $product->stock) {
                    $validator->errors()->add(
                        "items.{$index}.quantity",
                        "Insufficient stock for {$product->name}"
                    );
                }
            }
        });
    }
}
```

### JSON and API Validation

```php
// Validating JSON strings
'metadata' => [
    'required',
    'json',
    function ($attribute, $value, $fail) {
        $decoded = json_decode($value, true);

        // Validate JSON structure
        if (!isset($decoded['version']) || !isset($decoded['settings'])) {
            $fail('Metadata JSON must contain version and settings keys');
        }
    }
],

// For API endpoints - validate deeply nested JSON
'config' => 'required|array',
'config.features' => 'required|array',
'config.features.*.enabled' => 'required|boolean',
'config.features.*.settings' => 'sometimes|array',
'config.features.*.settings.threshold' => 'sometimes|numeric|min:0'
```

### Multi-Step Form Validation

```php
class MultiStepFormRequest extends FormRequest
{
    public function rules(): array
    {
        $step = $this->route('step');

        return match($step) {
            1 => $this->stepOneRules(),
            2 => $this->stepTwoRules(),
            3 => $this->stepThreeRules(),
            default => []
        };
    }

    private function stepOneRules(): array
    {
        return [
            'first_name' => 'required|string|max:255',
            'last_name' => 'required|string|max:255',
            'email' => 'required|email|unique:users'
        ];
    }

    private function stepTwoRules(): array
    {
        // Validate that step 1 was completed
        session()->has('form_step_1') || abort(422, 'Complete step 1 first');

        return [
            'company' => 'required|string|max:255',
            'industry' => 'required|exists:industries,id'
        ];
    }
}
```

---

## 4. Custom Rules Mastery

### When to Use Each Approach

#### 1. Closure Rules (Quick & Simple)

```php
// ✅ Good for: Simple, one-off validations
public function rules(): array
{
    return [
        'username' => [
            'required',
            'string',
            function ($attribute, $value, $fail) {
                if (str_contains(strtolower($value), 'admin')) {
                    $fail('Username cannot contain "admin"');
                }
            }
        ]
    ];
}
```

#### 2. Rule Objects (Reusable & Testable)

```php
// ✅ Good for: Complex logic, reusable across app, needs testing
class StrongPassword implements Rule
{
    private int $minLength;
    private bool $requireSpecialChars;

    public function __construct(int $minLength = 12, bool $requireSpecialChars = true)
    {
        $this->minLength = $minLength;
        $this->requireSpecialChars = $requireSpecialChars;
    }

    public function passes($attribute, $value): bool
    {
        if (strlen($value) < $this->minLength) {
            return false;
        }

        if ($this->requireSpecialChars && !preg_match('/[!@#$%^&*(),.?":{}|<>]/', $value)) {
            return false;
        }

        // Check against common passwords
        return !in_array(strtolower($value), $this->getCommonPasswords());
    }

    public function message(): string
    {
        return 'Password must be at least :min characters and contain special characters.';
    }

    private function getCommonPasswords(): array
    {
        return ['password123', '123456789', 'qwerty123'];
    }
}

// Usage
'password' => ['required', new StrongPassword(12, true)]
```

#### 3. Validator Extensions (Global Rules)

```php
// In AppServiceProvider::boot()
Validator::extend('phone', function ($attribute, $value, $parameters, $validator) {
    return preg_match('/^\+?[1-9]\d{1,14}$/', $value);
});

// Custom message
Validator::replacer('phone', function ($message, $attribute, $rule, $parameters) {
    return str_replace(':attribute', $attribute, 'The :attribute must be a valid phone number');
});

// Usage anywhere
'phone' => 'required|phone'
```

### Advanced Custom Rule Patterns

#### Database-Dependent Rule with Caching

```php
class ValidCouponCode implements Rule
{
    private $userId;
    private $orderTotal;

    public function __construct($userId, $orderTotal)
    {
        $this->userId = $userId;
        $this->orderTotal = $orderTotal;
    }

    public function passes($attribute, $value): bool
    {
        // Use cache to avoid repeated DB hits
        $coupon = Cache::remember("coupon:{$value}", 300, function () use ($value) {
            return Coupon::where('code', $value)
                         ->where('active', true)
                         ->where('expires_at', '>', now())
                         ->first();
        });

        if (!$coupon) {
            return false;
        }

        // Check usage limits
        if ($coupon->max_uses && $coupon->used_count >= $coupon->max_uses) {
            return false;
        }

        // Check minimum order amount
        if ($coupon->minimum_amount && $this->orderTotal < $coupon->minimum_amount) {
            return false;
        }

        return true;
    }

    public function message(): string
    {
        return 'The coupon code is invalid or has expired.';
    }
}
```

---

## 5. Error Handling & UX

### Custom Error Messages Done Right

#### Message Hierarchy (Laravel chooses in this order)

1. Custom messages in Form Request
2. Custom attribute names
3. Language files
4. Default Laravel messages

```php
class UserRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'email' => 'required|email|unique:users',
            'password' => 'required|min:8|confirmed'
        ];
    }

    // ✅ Specific field messages
    public function messages(): array
    {
        return [
            'email.required' => 'We need your email address to create your account.',
            'email.unique' => 'This email is already registered. Try logging in instead.',
            'password.min' => 'Your password must be at least 8 characters for security.',
            'password.confirmed' => 'Password confirmation doesn\'t match.'
        ];
    }

    // ✅ Custom attribute names (affects all messages for these fields)
    public function attributes(): array
    {
        return [
            'email' => 'email address',
            'password_confirmation' => 'password confirmation'
        ];
    }
}
```

### Advanced Error Message Patterns

#### Context-Aware Messages

```php
public function messages(): array
{
    $isUpdate = $this->isMethod('PUT') || $this->isMethod('PATCH');

    return [
        'email.required' => $isUpdate
            ? 'Email address is required to update your profile'
            : 'Email address is required to create your account',

        'password.required' => $isUpdate
            ? 'Enter your current password to confirm changes'
            : 'Create a secure password for your account'
    ];
}
```

#### Localized Messages with Parameters

```php
// In resources/lang/en/validation.php
'custom' => [
    'age' => [
        'min' => 'You must be at least :min years old to register.',
        'max' => 'Maximum age for registration is :max years.'
    ]
]

// In your form request
'age' => 'required|integer|min:18|max:120'
```

### API vs Web Validation Strategies

#### API-Optimized Validation

```php
class ApiUserRequest extends FormRequest
{
    protected function failedValidation(Validator $validator)
    {
        throw new HttpResponseException(
            response()->json([
                'success' => false,
                'message' => 'Validation failed',
                'errors' => $validator->errors(),
                'error_code' => 'VALIDATION_FAILED'
            ], 422)
        );
    }

    // Return structured data for API clients
    protected function passedValidation()
    {
        // Transform data after validation
        $this->merge([
            'normalized_phone' => $this->normalizePhone($this->phone),
            'email' => strtolower($this->email)
        ]);
    }
}
```

#### Web Form UX Enhancement

```php
// In your Blade template
@error('email')
    <div class="error-message" role="alert">
        <svg class="error-icon">...</svg>
        {{ $message }}
        @if($message === 'This email is already registered. Try logging in instead.')
            <a href="{{ route('login') }}" class="login-link">Go to Login</a>
        @endif
    </div>
@enderror
```

---

## 6. Performance & Scaling

### Avoiding N+1 Queries in Validation

#### ❌ Bad: Multiple DB hits

```php
// This runs a query for EACH item
'items.*.product_id' => 'exists:products,id'
```

#### ✅ Good: Batch validation

```php
public function withValidator($validator): void
{
    $validator->after(function ($validator) {
        $productIds = collect($this->input('items', []))->pluck('product_id')->unique();

        // Single query to get all products
        $existingProducts = Product::whereIn('id', $productIds)->pluck('id');

        foreach ($this->input('items', []) as $index => $item) {
            if (!$existingProducts->contains($item['product_id'])) {
                $validator->errors()->add("items.{$index}.product_id", 'Product not found');
            }
        }
    });
}
```

### Caching Expensive Validations

```php
class ExpensiveValidationRule implements Rule
{
    public function passes($attribute, $value): bool
    {
        $cacheKey = "validation:expensive:{$attribute}:" . md5($value);

        return Cache::remember($cacheKey, 300, function () use ($value) {
            // Expensive operation (external API call, complex calculation, etc.)
            return $this->performExpensiveValidation($value);
        });
    }
}
```

### Validation in Background Jobs

```php
class ProcessBulkImportJob implements ShouldQueue
{
    public function handle(): void
    {
        foreach ($this->importData as $rowIndex => $rowData) {
            try {
                $validator = Validator::make($rowData, $this->getRules());

                if ($validator->fails()) {
                    // Store errors for later review, don't fail the entire job
                    ImportError::create([
                        'import_id' => $this->importId,
                        'row' => $rowIndex,
                        'errors' => $validator->errors()->toArray()
                    ]);
                    continue;
                }

                // Process valid row
                $this->processRow($validator->validated());

            } catch (Exception $e) {
                // Handle individual row failures gracefully
                Log::error("Row {$rowIndex} failed: " . $e->getMessage());
            }
        }
    }
}
```

---

## 7. Security Aspects

### Input Sanitization vs Validation

**Key Principle:** Validate input format, sanitize on output

```php
// ✅ Good: Validate structure, don't modify input
public function rules(): array
{
    return [
        'bio' => 'required|string|max:1000',  // Validate length and type
        'website' => 'nullable|url',          // Validate format
        'social_links.*' => 'url'            // Validate each URL
    ];
}

// ❌ Don't do this in validation
protected function prepareForValidation(): void
{
    $this->merge([
        'bio' => strip_tags($this->bio),  // Don't sanitize here!
    ]);
}

// ✅ Sanitize when displaying
// In your Blade template or API response
{{ Str::limit(strip_tags($user->bio), 100) }}
```

### File Upload Validation (Security Critical)

```php
// ✅ Comprehensive file validation
public function rules(): array
{
    return [
        'avatar' => [
            'required',
            'file',
            'image',                    // Only image files
            'mimes:jpeg,png,jpg,webp',  // Specific formats
            'max:2048',                 // Max 2MB
            'dimensions:min_width=100,min_height=100,max_width=2000,max_height=2000'
        ],

        'document' => [
            'required',
            'file',
            'mimes:pdf,doc,docx',
            'max:10240',  // 10MB
            function ($attribute, $value, $fail) {
                // Additional security check
                if ($this->isSuspiciousFile($value)) {
                    $fail('File appears to be malicious');
                }
            }
        ]
    ];
}

private function isSuspiciousFile($file): bool
{
    // Check actual file content, not just extension
    $finfo = finfo_open(FILEINFO_MIME_TYPE);
    $mimeType = finfo_file($finfo, $file->getPathname());
    finfo_close($finfo);

    // Ensure MIME type matches expected types
    $allowedMimes = ['application/pdf', 'application/msword'];
    return !in_array($mimeType, $allowedMimes);
}
```

### Mass Assignment Protection

```php
// ✅ Whitelist approach in Form Request
public function rules(): array
{
    return [
        'name' => 'required|string|max:255',
        'email' => 'required|email',
        // Only validate fields that should be mass assignable
    ];
}

// In your controller
User::create($request->validated()); // Only validated fields

// ❌ Dangerous: Using all input
User::create($request->all()); // Could include unexpected fields
```

### SQL Injection Prevention

```php
// ✅ Safe: Using Laravel's query builder
Rule::exists('users')->where(function ($query) {
    $query->where('organization_id', $this->user()->organization_id);
})

// ❌ Dangerous: Raw SQL without proper escaping
Rule::exists('users')->whereRaw("organization_id = {$this->user()->organization_id}")
```

---

## 8. Best Practices

### Validation Architecture Patterns

#### Single Responsibility Form Requests

```php
// ✅ Good: Specific, focused requests
class CreateUserRequest extends FormRequest { /* ... */ }
class UpdateUserRequest extends FormRequest { /* ... */ }
class UpdateUserPasswordRequest extends FormRequest { /* ... */ }

// ❌ Bad: Generic, catch-all request
class UserRequest extends FormRequest
{
    public function rules(): array
    {
        // Trying to handle all user operations
        if ($this->isMethod('POST')) {
            // Create rules
        } elseif ($this->isMethod('PUT')) {
            // Update rules
        }
        // This becomes unwieldy
    }
}
```

#### Reusable Validation Logic

```php
// Create trait for common validation patterns
trait HasUserValidation
{
    protected function userRules(bool $requirePassword = true): array
    {
        $rules = [
            'name' => 'required|string|max:255',
            'email' => 'required|email|unique:users,email'
        ];

        if ($requirePassword) {
            $rules['password'] = 'required|min:8|confirmed';
        }

        return $rules;
    }

    protected function userMessages(): array
    {
        return [
            'name.required' => 'Please provide your full name',
            'email.unique' => 'This email is already registered'
        ];
    }
}

// Use in multiple Form Requests
class CreateUserRequest extends FormRequest
{
    use HasUserValidation;

    public function rules(): array
    {
        return $this->userRules(true);
    }

    public function messages(): array
    {
        return $this->userMessages();
    }
}
```

### Service Layer Validation

```php
// ✅ Best practice: Validate at service boundaries
class UserService
{
    public function createUser(array $userData): User
    {
        // Always validate in service layer, even if validated in controller
        $this->validateUserData($userData);

        DB::beginTransaction();
        try {
            $user = User::create($userData);

            // Additional business logic
            $this->assignDefaultRole($user);
            $this->sendWelcomeEmail($user);

            DB::commit();
            return $user;
        } catch (Exception $e) {
            DB::rollback();
            throw $e;
        }
    }

    private function validateUserData(array $data): void
    {
        $validator = Validator::make($data, [
            'name' => 'required|string|max:255',
            'email' => 'required|email|unique:users',
        ]);

        if ($validator->fails()) {
            throw new ValidationException($validator);
        }
    }
}
```

### Testing Validation Logic

```php
class CreateUserRequestTest extends TestCase
{
    /** @test */
    public function it_validates_required_fields(): void
    {
        $response = $this->postJson('/users', []);

        $response->assertStatus(422)
                 ->assertJsonValidationErrors(['name', 'email']);
    }

    /** @test */
    public function it_validates_email_uniqueness(): void
    {
        User::factory()->create(['email' => 'test@example.com']);

        $response = $this->postJson('/users', [
            'name' => 'John Doe',
            'email' => 'test@example.com'
        ]);

        $response->assertJsonValidationErrors(['email']);
    }

    /** @test */
    public function it_accepts_valid_data(): void
    {
        $response = $this->postJson('/users', [
            'name' => 'John Doe',
            'email' => 'john@example.com',
            'password' => 'password123',
            'password_confirmation' => 'password123'
        ]);

        $response->assertStatus(201);
        $this->assertDatabaseHas('users', ['email' => 'john@example.com']);
    }
}
```

---

## 9. Common Mistakes

### The `nullable` vs `sometimes` Confusion

```php
// ❌ Wrong understanding
'bio' => 'nullable|required|max:500'  // Contradictory!

// ✅ Correct usage
'bio' => 'nullable|string|max:500'        // Can be null OR valid string
'bio' => 'sometimes|required|string|max:500'  // If present, must be valid string
```

### Array Validation Mistakes

```php
// ❌ Common mistakes
'items' => 'array',              // Allows empty array
'items.*' => 'string'            // Doesn't validate individual items properly
'items.*.id' => 'exists:products' // Missing validation for empty arrays

// ✅ Correct approach
'items' => 'required|array|min:1|max:10',  // Must have 1-10 items
'items.*' => 'required|array',              // Each item must be an array
'items.*.id' => 'required|exists:products,id',
'items.*.quantity' => 'required|integer|min:1|max:999'
```

### Unique Rule Edge Cases

```php
// ❌ Problematic unique rules
'email' => 'unique:users'  // Doesn't handle updates correctly
'slug' => 'unique:posts'   // Doesn't consider soft deletes

// ✅ Robust unique rules
'email' => [
    'required',
    'email',
    Rule::unique('users')->ignore($this->route('user')?->id)->whereNull('deleted_at')
],
'slug' => [
    'required',
    Rule::unique('posts')->where('published_at', '!=', null)  // Only check published posts
]
```

### File Validation Oversights

```php
// ❌ Insecure file validation
'upload' => 'file|max:1024'  // No type restriction!

// ✅ Secure file validation
'upload' => [
    'required',
    'file',
    'mimes:pdf,doc,docx',  // Whitelist allowed types
    'max:10240',           // Size limit
    function ($attribute, $value, $fail) {
        // Check actual file content
        $realMimeType = mime_content_type($value->getPathname());
        $allowedMimes = ['application/pdf', 'application/msword'];

        if (!in_array($realMimeType, $allowedMimes)) {
            $fail('File type not allowed');
        }
    }
]
```

### Performance Mistakes

```php
// ❌ N+1 query problem
foreach ($request->input('user_ids') as $userId) {
    // This runs a query for each user_id
    'user_ids.*' => 'exists:users,id'
}

// ✅ Efficient validation
public function withValidator($validator): void
{
    $validator->after(function ($validator) {
        $userIds = $this->input('user_ids', []);
        $existingUsers = User::whereIn('id', $userIds)->pluck('id');

        foreach ($userIds as $index => $userId) {
            if (!$existingUsers->contains($userId)) {
                $validator->errors()->add("user_ids.{$index}", 'User not found');
            }
        }
    });
}
```

---

## 10. Laravel 10/11+ Updates

### New Features in Laravel 10+

#### List Rule for Arrays

```php
// New in Laravel 10.x
'users' => ['required', 'list'],  // Must be a list (numerically indexed array)
'users.*' => 'email'
```

#### Enhanced Error Messages

```php
// Better default error messages with more context
'items.0.name' => 'The name field is required for item #1'
'items.1.price' => 'The price must be a number for item #2'
```

### Laravel 11 Improvements

#### Rule Objects as Invokable Classes

```php
// New syntax in Laravel 11
class ValidatePostalCode
{
    public function __invoke($attribute, $value, $fail)
    {
        if (!$this->isValidPostalCode($value)) {
            $fail('Invalid postal code format');
        }
    }
}

// Usage
'postal_code' => [new ValidatePostalCode]
```

#### Improved Validation Pipeline

```php
// Better error handling and custom validation pipeline
public function rules(): array
{
    return [
        'data' => [
            'required',
            'array',
            new ValidateJsonStructure(['required_keys' => ['name', 'email']])
        ]
    ];
}
```

### Deprecation Warnings

#### Avoid These Patterns

```php
// ⚠️ Deprecated: String-based custom rules
Validator::extend('foo', 'FooValidator@validate');

// ✅ Use: Rule objects or closures
Validator::extend('foo', function ($attribute, $value, $parameters, $validator) {
    return $this->validateFoo($value);
});
```

---

## Senior-Level Rules of Thumb

### 🎯 The Golden Rules

1. **Validate Early, Validate Often**: Validate at system boundaries (HTTP requests, service layer inputs, external API responses)

2. **Be Explicit**: `'email' => 'required|email|max:255'` is better than `'email' => 'email'`

3. **Think Security First**: Always validate file uploads, use whitelisting over blacklisting, never trust client-side validation alone

4. **Performance Matters**: Batch database validation, cache expensive rules, avoid N+1 queries

5. **User Experience**: Provide clear, actionable error messages; validate related fields together

6. **Test Everything**: Your validation logic is critical business logic - test it thoroughly

### 🚀 Advanced Patterns to Master

- **Conditional Rule Building**: Dynamically build rules based on business logic
- **Cross-Field Validation**: Validate relationships between multiple fields
- **Contextual Messages**: Different messages for different user types or scenarios
- **Validation Middleware**: Custom middleware for complex multi-step validation
- **Event-Driven Validation**: Trigger validation based on model events

### 📈 From Mid-Level to Senior Checklist

- [ ] Can explain how Laravel's Validator class works internally
- [ ] Know when to use Form Requests vs manual validation vs inline validation
- [ ] Master complex array and nested object validation
- [ ] Understand database rule optimization and caching strategies
- [ ] Can write secure file upload validation
- [ ] Design reusable validation components and traits
- [ ] Handle validation in background jobs and service layers
- [ ] Write comprehensive tests for validation logic
- [ ] Optimize validation performance in high-traffic applications
- [ ] Create custom rules that are maintainable and testable

**Remember**: Senior-level validation isn't just about knowing more rules - it's about understanding the security implications, performance considerations, user experience impact, and maintainability of your validation logic. Always think about the bigger picture!

---

_This masterclass represents the culmination of validation best practices. Practice these concepts in real projects, and you'll develop the intuition that separates senior developers from mid-level ones._
