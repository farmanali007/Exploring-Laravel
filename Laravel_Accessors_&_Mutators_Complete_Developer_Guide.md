# Laravel Accessors & Mutators: Complete Developer Guide

## Table of Contents

1. [Conceptual Foundation](#conceptual-foundation)
2. [Implementation Approaches](#implementation-approaches)
3. [Eloquent Integration](#eloquent-integration)
4. [Real-World Examples](#real-world-examples)
5. [Performance & Security](#performance--security)
6. [When NOT to Use Them](#when-not-to-use-them)
7. [Testing Strategies](#testing-strategies)
8. [Senior Developer Best Practices](#senior-developer-best-practices)
9. [Summary & Key Takeaways](#summary--key-takeaways)

---

## Conceptual Foundation

### What Are Accessors and Mutators?

Accessors and mutators are Eloquent's mechanism for transforming attribute values when they're retrieved from or saved to the database. They sit at the intersection of your domain model and data persistence layer, providing a clean way to handle data transformation without cluttering your business logic.

**Accessors** transform attribute values when reading from the model:

- Format data for display (currency, dates, names)
- Compute virtual attributes from existing data
- Decrypt sensitive information
- Apply business rules to raw data

**Mutators** transform attribute values before saving to the database:

- Normalize input data (email lowercase, phone formatting)
- Encrypt sensitive data
- Generate derived values (slugs from titles)
- Apply validation or sanitization

### Eloquent Lifecycle Integration

Accessors and mutators hook into Eloquent's attribute resolution system. Understanding their position in the lifecycle is crucial:

1. **Mutators**: Execute during `fill()`, direct assignment, or mass assignment
2. **Database Write**: Mutated values are persisted
3. **Database Read**: Raw values are retrieved
4. **Accessors**: Execute during attribute access via `$model->attribute`
5. **Serialization**: Accessed values appear in `toArray()` and JSON output

This lifecycle integration means accessors/mutators work seamlessly with relationships, eager loading, and Laravel's serialization system.

---

## Implementation Approaches

Laravel offers two primary approaches: classic method-based and modern Attribute class-based. Each has distinct advantages and use cases.

### Classic Method-Based Approach

The traditional approach uses naming conventions to define accessor/mutator methods.

```php
<?php

class User extends Model
{
    // Accessor: transforms data when reading
    public function getFullNameAttribute(): string
    {
        return trim($this->first_name . ' ' . $this->last_name);
    }

    // Mutator: transforms data when writing
    public function setEmailAttribute(string $value): void
    {
        $this->attributes['email'] = strtolower(trim($value));
    }
}
```

**Pros:**

- Explicit method signatures
- IDE-friendly with proper type hints
- Familiar to developers from older Laravel versions
- Clear separation between get/set logic

**Cons:**

- More verbose for simple transformations
- Two methods required for bidirectional transforms
- Method naming can become unwieldy

### Modern Attribute Class Approach

Laravel 9+ introduced the `Attribute` class for more concise accessor/mutator definitions.

```php
<?php

use Illuminate\Database\Eloquent\Casts\Attribute;

class User extends Model
{
    // Combined accessor/mutator
    protected function email(): Attribute
    {
        return Attribute::make(
            get: fn (string $value) => strtoupper($value),
            set: fn (string $value) => strtolower(trim($value))
        );
    }

    // Accessor-only
    protected function fullName(): Attribute
    {
        return Attribute::make(
            get: fn () => trim($this->first_name . ' ' . $this->last_name)
        );
    }
}
```

**Pros:**

- Concise syntax
- Single method for related logic
- Better performance (cached)
- Supports fluent method chaining

**Cons:**

- Less explicit return types
- Newer syntax (team familiarity)
- Closure-based (debugging considerations)

### When to Choose Each Approach

**Use Classic Methods When:**

- Complex transformation logic requiring multiple statements
- Team prefers explicit method signatures
- Working with legacy codebases
- Need maximum IDE support and debugging clarity

**Use Attribute Class When:**

- Simple transformations (formatting, normalization)
- Want to keep related get/set logic together
- Building new applications on Laravel 9+
- Performance is critical (cached attribute resolution)

---

## Eloquent Integration

### Working with Attribute Casting

Accessors/mutators integrate seamlessly with Eloquent's `$casts` property and custom cast classes.

```php
<?php

class Product extends Model
{
    protected $casts = [
        'metadata' => 'array',
        'price_cents' => 'integer',
        'published_at' => 'datetime',
        'status' => ProductStatus::class, // Enum cast
    ];

    // Accessor works with casted attributes
    protected function price(): Attribute
    {
        return Attribute::make(
            get: fn (int $value) => $value / 100, // Convert cents to dollars
            set: fn (float $value) => (int) ($value * 100) // Convert dollars to cents
        );
    }

    // Virtual attribute using casted data
    protected function isOnSale(): Attribute
    {
        return Attribute::make(
            get: fn () => $this->status === ProductStatus::Sale &&
                         $this->published_at?->isPast()
        );
    }
}
```

### Serialization Control

Control how accessors appear in JSON/array output using `$appends`, `$hidden`, and `$visible`.

```php
<?php

class User extends Model
{
    protected $appends = ['full_name', 'avatar_url'];
    protected $hidden = ['password', 'remember_token'];

    protected function fullName(): Attribute
    {
        return Attribute::make(
            get: fn () => trim($this->first_name . ' ' . $this->last_name)
        );
    }

    protected function avatarUrl(): Attribute
    {
        return Attribute::make(
            get: fn () => $this->avatar
                ? Storage::url($this->avatar)
                : asset('images/default-avatar.png')
        );
    }
}

// Usage
$user = User::first();
$user->toArray(); // Includes full_name and avatar_url, excludes password
```

**Key Integration Points:**

- Appended accessors appear in API responses automatically
- Hidden attributes are excluded even if accessed
- Accessors work with Eloquent relationships and eager loading
- Values are computed during serialization, not database queries

---

## Real-World Examples

### 1. Full Name Accessor with Localization

**Problem**: Display formatted names considering cultural differences and missing data.

**Before** (without accessor):

```php
<?php

// Controller logic scattered throughout application
public function show(User $user)
{
    $fullName = trim($user->first_name . ' ' . $user->last_name);
    if (empty($fullName)) {
        $fullName = $user->email;
    }

    return view('users.show', compact('user', 'fullName'));
}
```

**After** (with accessor):

```php
<?php

class User extends Model
{
    protected $appends = ['full_name'];

    protected function fullName(): Attribute
    {
        return Attribute::make(
            get: function (): string {
                // Handle different name formats
                $name = match (app()->getLocale()) {
                    'ja' => $this->last_name . ' ' . $this->first_name,
                    default => $this->first_name . ' ' . $this->last_name
                };

                $name = trim($name);

                // Fallback to email if no name available
                return $name ?: $this->email;
            }
        );
    }
}

// Controller becomes clean
public function show(User $user)
{
    return view('users.show', compact('user'));
}

// View usage
{{ $user->full_name }}
```

**Key Changes:**

- Centralized name formatting logic in the model
- Locale-aware formatting using match expressions
- Automatic fallback to email when name is empty
- Appended to serialization for API consistency

### 2. Money/Price with Cents Storage

**Problem**: Store currency as cents (integer) for precision, expose as decimal for business logic and APIs.

**Before** (manual conversion everywhere):

```php
<?php

// Scattered conversion logic
$product = Product::find(1);
$priceInDollars = $product->price_cents / 100;

// Error-prone saving
$product->price_cents = (int) ($request->price * 100);
```

**After** (with accessor/mutator):

```php
<?php

class Product extends Model
{
    protected $casts = [
        'price_cents' => 'integer',
    ];

    protected function price(): Attribute
    {
        return Attribute::make(
            get: fn (int $value) => number_format($value / 100, 2, '.', ''),
            set: fn (string|float $value) => (int) (floatval($value) * 100)
        );
    }

    // Formatted price for display
    protected function priceFormatted(): Attribute
    {
        return Attribute::make(
            get: fn () => '$' . $this->price
        );
    }
}

// Usage becomes clean and consistent
$product = Product::find(1);
echo $product->price; // "29.99"
echo $product->price_formatted; // "$29.99"

$product->price = "39.99"; // Automatically converts to 3999 cents
$product->save();
```

**Key Changes:**

- Automatic precision handling prevents floating-point errors
- Mutator accepts both string and float input
- Separate formatted accessor for display purposes
- Database stores integers (price_cents) for exact arithmetic

### 3. Slug Generation with Collision Handling

**Problem**: Automatically generate URL-friendly slugs from titles, handle uniqueness without database calls in mutators.

**Before** (manual slug creation):

```php
<?php

// Controller handling slug creation
public function store(Request $request)
{
    $slug = Str::slug($request->title);

    // Check for uniqueness (problematic in mutator)
    $originalSlug = $slug;
    $counter = 1;
    while (Post::where('slug', $slug)->exists()) {
        $slug = $originalSlug . '-' . $counter++;
    }

    Post::create([
        'title' => $request->title,
        'slug' => $slug,
        'content' => $request->content,
    ]);
}
```

**After** (with mutator + observer):

```php
<?php

class Post extends Model
{
    protected $fillable = ['title', 'content', 'slug'];

    // Mutator handles basic slug generation only
    protected function slug(): Attribute
    {
        return Attribute::make(
            set: fn (?string $value) => $value ?: Str::slug($this->title)
        );
    }
}

class PostObserver
{
    public function saving(Post $post): void
    {
        // Handle uniqueness check in observer, not mutator
        if ($post->isDirty('slug')) {
            $originalSlug = $post->slug;
            $counter = 1;

            while (Post::where('slug', $post->slug)
                       ->where('id', '!=', $post->id ?? 0)
                       ->exists()) {
                $post->slug = $originalSlug . '-' . $counter++;
            }
        }
    }
}

// Clean controller
public function store(Request $request)
{
    return Post::create($request->validated());
}
```

**Key Changes:**

- Mutator handles basic slug generation from title
- Observer manages database-dependent uniqueness logic
- Separation of concerns: transformation vs business rules
- Handles both create and update scenarios correctly

### 4. JSON Meta Accessor with Deep Merging

**Problem**: Store flexible metadata as JSON, provide type-safe access with default values and deep merging.

**Before** (manual JSON handling):

```php
<?php

// Repetitive JSON handling throughout codebase
$settings = json_decode($user->meta ?? '{}', true);
$theme = $settings['ui']['theme'] ?? 'light';
$notifications = array_merge(
    ['email' => true, 'sms' => false],
    $settings['notifications'] ?? []
);
```

**After** (with structured accessor):

```php
<?php

class User extends Model
{
    protected $casts = [
        'meta' => 'array',
    ];

    // Structured settings access
    protected function settings(): Attribute
    {
        return Attribute::make(
            get: function (): array {
                $defaults = [
                    'ui' => [
                        'theme' => 'light',
                        'language' => 'en',
                        'timezone' => 'UTC',
                    ],
                    'notifications' => [
                        'email' => true,
                        'sms' => false,
                        'push' => true,
                    ],
                    'privacy' => [
                        'profile_visible' => true,
                        'activity_visible' => false,
                    ],
                ];

                return array_merge_recursive($defaults, $this->meta ?? []);
            }
        );
    }

    // Convenience accessors for common settings
    protected function theme(): Attribute
    {
        return Attribute::make(
            get: fn () => $this->settings['ui']['theme']
        );
    }

    protected function notificationPreferences(): Attribute
    {
        return Attribute::make(
            get: fn () => $this->settings['notifications']
        );
    }
}

// Clean usage
$user = User::find(1);
echo $user->theme; // 'light' or user's preference
$preferences = $user->notification_preferences;

// Type-safe setting updates
$meta = $user->meta ?? [];
$meta['ui']['theme'] = 'dark';
$user->meta = $meta;
$user->save();
```

**Key Changes:**

- Deep merging provides sensible defaults for missing keys
- Structured access prevents JSON parsing errors
- Convenience accessors for frequently used settings
- Maintains type safety with proper casting

### 5. Encrypted Attributes

**Problem**: Transparently encrypt/decrypt sensitive data using Laravel's encryption.

**Before** (manual encryption handling):

```php
<?php

// Manual encryption in multiple places
$user = new User();
$user->ssn = encrypt($request->ssn);

// Manual decryption when accessing
$ssn = decrypt($user->ssn);
```

**After** (with encrypted cast):

```php
<?php

class User extends Model
{
    protected $casts = [
        'ssn' => 'encrypted',
        'credit_card' => 'encrypted',
    ];

    // Additional formatting for display
    protected function ssnMasked(): Attribute
    {
        return Attribute::make(
            get: fn () => $this->ssn ? 'XXX-XX-' . substr($this->ssn, -4) : null
        );
    }
}

// Usage becomes transparent
$user = User::create(['ssn' => '123-45-6789']); // Auto-encrypted
echo $user->ssn; // Auto-decrypted: "123-45-6789"
echo $user->ssn_masked; // Formatted: "XXX-XX-6789"
```

**Key Changes:**

- Built-in encrypted cast handles encryption/decryption
- Additional accessor provides masked display format
- Transparent usage throughout application
- No manual encrypt/decrypt calls needed

### 6. Modern Attribute Class Migration

**Before** (classic approach):

```php
<?php

class Product extends Model
{
    public function getPriceFormattedAttribute(): string
    {
        return '$' . number_format($this->price / 100, 2);
    }

    public function setPriceAttribute($value): void
    {
        $this->attributes['price'] = (int) ($value * 100);
    }

    public function getSlugAttribute(): ?string
    {
        return $this->attributes['slug'] ?? Str::slug($this->title);
    }
}
```

**After** (migrated to Attribute class):

```php
<?php

use Illuminate\Database\Eloquent\Casts\Attribute;

class Product extends Model
{
    // Combined price logic
    protected function price(): Attribute
    {
        return Attribute::make(
            get: fn (int $value) => $value / 100,
            set: fn (float|string $value) => (int) (floatval($value) * 100)
        );
    }

    // Formatted price as separate concern
    protected function priceFormatted(): Attribute
    {
        return Attribute::make(
            get: fn () => '$' . number_format($this->price, 2)
        );
    }

    // Slug with fallback logic
    protected function slug(): Attribute
    {
        return Attribute::make(
            get: fn (?string $value) => $value ?? Str::slug($this->title)
        );
    }
}
```

**Why Migrate?**

- **Performance**: Attribute classes are cached, reducing method call overhead
- **Consistency**: Related get/set logic stays together
- **Maintainability**: Less method pollution in model classes
- **Modern Laravel**: Aligns with current framework patterns

---

## Performance & Security

### Performance Considerations

**Avoid Database Calls in Accessors**

```php
<?php

// BAD: Database query in accessor
protected function postCount(): Attribute
{
    return Attribute::make(
        get: fn () => $this->posts()->count() // N+1 query problem!
    );
}

// GOOD: Use relationships with eager loading
public function posts()
{
    return $this->hasMany(Post::class);
}

// Controller: eager load with count
$users = User::withCount('posts')->get();
foreach ($users as $user) {
    echo $user->posts_count; // Uses eager-loaded count
}
```

**Expensive Computation Caching**

```php
<?php

class User extends Model
{
    // BAD: Recalculates on every access
    protected function expensiveComputation(): Attribute
    {
        return Attribute::make(
            get: fn () => $this->performComplexCalculation()
        );
    }

    // GOOD: Cache expensive operations
    protected function expensiveComputation(): Attribute
    {
        return Attribute::make(
            get: function () {
                return Cache::remember(
                    "user.{$this->id}.expensive_computation",
                    now()->addHour(),
                    fn () => $this->performComplexCalculation()
                );
            }
        );
    }
}
```

**Conditional Appending**

```php
<?php

class User extends Model
{
    // Don't append by default if expensive
    // protected $appends = ['full_name']; // Remove this

    protected function fullName(): Attribute
    {
        return Attribute::make(
            get: fn () => trim($this->first_name . ' ' . $this->last_name)
        );
    }
}

// Append selectively in controllers/resources
class UserController extends Controller
{
    public function index()
    {
        return User::all()->append('full_name'); // Only when needed
    }
}
```

### Security Considerations

**XSS Prevention in Accessors**

```php
<?php

class Post extends Model
{
    // BAD: Raw HTML without escaping
    protected function contentPreview(): Attribute
    {
        return Attribute::make(
            get: fn () => substr($this->content, 0, 200) . '...'
        );
    }

    // GOOD: Escape HTML for safe display
    protected function contentPreview(): Attribute
    {
        return Attribute::make(
            get: fn () => e(Str::limit(strip_tags($this->content), 200))
        );
    }
}
```

**Input Sanitization in Mutators**

```php
<?php

class User extends Model
{
    protected function email(): Attribute
    {
        return Attribute::make(
            set: function (string $value) {
                // Sanitize and validate
                $email = filter_var(trim($value), FILTER_SANITIZE_EMAIL);

                if (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
                    throw new InvalidArgumentException('Invalid email format');
                }

                return strtolower($email);
            }
        );
    }
}
```

**Sensitive Data Handling**

```php
<?php

class User extends Model
{
    protected $hidden = ['password', 'ssn', 'api_token'];

    // Never expose sensitive data accidentally
    protected function maskedSsn(): Attribute
    {
        return Attribute::make(
            get: fn () => $this->ssn ? 'XXX-XX-' . substr($this->ssn, -4) : null
        );
    }
}
```

---

## When NOT to Use Them

### Anti-Patterns and Better Alternatives

**Anti-Pattern 1: Heavy Business Logic in Accessors**

```php
<?php

// BAD: Complex business logic in accessor
class Order extends Model
{
    protected function canBeCancelled(): Attribute
    {
        return Attribute::make(
            get: function () {
                // Complex business rules don't belong here
                if ($this->status === 'shipped') return false;
                if ($this->created_at->diffInHours() > 24) return false;
                if ($this->payment_status !== 'completed') return false;

                // HTTP calls in accessors are terrible!
                $shippingStatus = Http::get("shipping-api/orders/{$this->id}");
                return $shippingStatus->json('can_cancel');
            }
        );
    }
}

// GOOD: Use dedicated service classes
class OrderCancellationService
{
    public function canCancel(Order $order): bool
    {
        return $this->checkStatusRequirements($order)
            && $this->checkTimeRequirements($order)
            && $this->checkPaymentRequirements($order)
            && $this->checkShippingStatus($order);
    }

    // Individual rule methods...
}

class Order extends Model
{
    public function canBeCancelled(): bool
    {
        return app(OrderCancellationService::class)->canCancel($this);
    }
}
```

**Anti-Pattern 2: Form Validation in Mutators**

```php
<?php

// BAD: Validation in mutators
class User extends Model
{
    protected function email(): Attribute
    {
        return Attribute::make(
            set: function (string $value) {
                // Validation doesn't belong in mutators!
                if (!filter_var($value, FILTER_VALIDATE_EMAIL)) {
                    throw new ValidationException('Invalid email');
                }
                return strtolower($value);
            }
        );
    }
}

// GOOD: Use Form Requests for validation
class CreateUserRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'email' => 'required|email|unique:users',
        ];
    }

    public function prepareForValidation(): void
    {
        $this->merge([
            'email' => strtolower($this->email),
        ]);
    }
}

class User extends Model
{
    // Simple normalization only
    protected function email(): Attribute
    {
        return Attribute::make(
            set: fn (string $value) => strtolower(trim($value))
        );
    }
}
```

**Anti-Pattern 3: Complex Data Transformations**

```php
<?php

// BAD: Complex transformations in accessors
class Report extends Model
{
    protected function aggregatedData(): Attribute
    {
        return Attribute::make(
            get: function () {
                // This belongs in a dedicated class
                $data = collect($this->raw_data);
                return $data->groupBy('category')
                           ->map(fn ($items) => $items->sum('amount'))
                           ->sortByDesc()
                           ->take(10)
                           ->map(fn ($amount, $category) => [
                               'category' => $category,
                               'amount' => $amount,
                               'percentage' => $amount / $data->sum('amount') * 100
                           ]);
            }
        );
    }
}

// GOOD: Use Data Transfer Objects or Value Objects
class ReportAggregator
{
    public function aggregate(array $rawData): ReportSummary
    {
        $data = collect($rawData);
        $total = $data->sum('amount');

        $topCategories = $data->groupBy('category')
            ->map(fn ($items) => $items->sum('amount'))
            ->sortByDesc()
            ->take(10)
            ->map(fn ($amount, $category) => new CategorySummary(
                category: $category,
                amount: $amount,
                percentage: $amount / $total * 100
            ));

        return new ReportSummary($topCategories, $total);
    }
}

class Report extends Model
{
    public function getAggregatedData(): ReportSummary
    {
        return app(ReportAggregator::class)->aggregate($this->raw_data);
    }
}
```

### When to Use Alternatives

**Use Custom Casts For:**

- Complex type transformations
- Reusable logic across models
- Value objects with validation

**Use DTOs/Value Objects For:**

- Input validation and transformation
- Complex data structures
- API request/response handling

**Use Observers For:**

- Database-dependent logic
- Side effects (emails, notifications)
- Complex model lifecycle events

**Use Service Classes For:**

- Business logic
- External API calls
- Multi-step processes

---

## Testing Strategies

### Unit Testing Accessors

Testing accessors focuses on transformation logic without database dependencies.

```php
<?php

use App\Models\User;
use PHPUnit\Framework\TestCase;

class UserAccessorTest extends TestCase
{
    /** @test */
    public function it_formats_full_name_correctly()
    {
        $user = new User([
            'first_name' => 'John',
            'last_name' => 'Doe'
        ]);

        $this->assertEquals('John Doe', $user->full_name);
    }

    /** @test */
    public function it_handles_missing_names_gracefully()
    {
        $user = new User([
            'first_name' => '',
            'last_name' => '',
            'email' => 'john@example.com'
        ]);

        $this->assertEquals('john@example.com', $user->full_name);
    }

    /** @test */
    public function it_trims_whitespace_from_names()
    {
        $user = new User([
            'first_name' => '  John  ',
            'last_name' => '  Doe  '
        ]);

        $this->assertEquals('John Doe', $user->full_name);
    }

    /** @test */
    public function it_formats_names_for_japanese_locale()
    {
        App::setLocale('ja');

        $user = new User([
            'first_name' => 'Taro',
            'last_name' => 'Yamada'
        ]);

        $this->assertEquals('Yamada Taro', $user->full_name);
    }
}
```

### Unit Testing Mutators

Testing mutators ensures input transformation works correctly before database storage.

```php
<?php

use App\Models\Product;
use PHPUnit\Framework\TestCase;

class ProductMutatorTest extends TestCase
{
    /** @test */
    public function it_converts_price_to_cents()
    {
        $product = new Product();
        $product->price = '29.99';

        $this->assertEquals(2999, $product->getAttributes()['price_cents']);
    }

    /** @test */
    public function it_handles_integer_prices()
    {
        $product = new Product();
        $product->price = 30;

        $this->assertEquals(3000, $product->getAttributes()['price_cents']);
    }

    /** @test */
    public function it_handles_float_prices()
    {
        $product = new Product();
        $product->price = 29.99;

        $this->assertEquals(2999, $product->getAttributes()['price_cents']);
    }

    /** @test */
    public function it_generates_slug_from_title()
    {
        $product = new Product(['title' => 'Amazing Product Name!']);
        $product->slug = null; // Trigger mutator

        $this->assertEquals('amazing-product-name', $product->getAttributes()['slug']);
    }

    /** @test */
    public function it_preserves_existing_slug()
    {
        $product = new Product(['title' => 'Amazing Product']);
        $product->slug = 'custom-slug';

        $this->assertEquals('custom-slug', $product->getAttributes()['slug']);
    }
}
```

### Integration Testing with Database

For complex logic involving observers or database constraints:

```php
<?php

use App\Models\Post;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class SlugGenerationTest extends TestCase
{
    use RefreshDatabase;

    /** @test */
    public function it_generates_unique_slugs_for_duplicate_titles()
    {
        // Create first post
        $post1 = Post::create(['title' => 'Amazing Post', 'content' => 'Content']);
        $this->assertEquals('amazing-post', $post1->slug);

        // Create second post with same title
        $post2 = Post::create(['title' => 'Amazing Post', 'content' => 'Content']);
        $this->assertEquals('amazing-post-1', $post2->slug);

        // Create third post
        $post3 = Post::create(['title' => 'Amazing Post', 'content' => 'Content']);
        $this->assertEquals('amazing-post-2', $post3->slug);
    }

    /** @test */
    public function it_updates_slug_when_title_changes()
    {
        $post = Post::create(['title' => 'Original Title', 'content' => 'Content']);
        $this->assertEquals('original-title', $post->slug);

        $post->update(['title' => 'Updated Title']);
        $this->assertEquals('updated-title', $post->slug);
    }
}
```

### Testing Edge Cases

Always test boundary conditions and error scenarios:

```php
<?php

class MoneyAttributeTest extends TestCase
{
    /** @test */
    public function it_handles_zero_values()
    {
        $product = new Product();
        $product->price = 0;

        $this->assertEquals(0, $product->getAttributes()['price_cents']);
        $this->assertEquals('0.00', $product->price);
    }

    /** @test */
    public function it_handles_negative_values()
    {
        $product = new Product();
        $product->price = -10.50;

        $this->assertEquals(-1050, $product->getAttributes()['price_cents']);
        $this->assertEquals('-10.50', $product->price);
    }

    /** @test */
    public function it_handles_very_small_amounts()
    {
        $product = new Product();
        $product->price = 0.01;

        $this->assertEquals(1, $product->getAttributes()['price_cents']);
        $this->assertEquals('0.01', $product->price);
    }
}
```

### What to Mock vs Test

**Mock External Dependencies:**

```php
<?php

// When testing accessors that use external services
class UserTest extends TestCase
{
    /** @test */
    public function it_generates_avatar_url()
    {
        Storage::fake('public');
        Storage::put('public/avatars/user1.jpg', 'fake-content');

        $user = new User(['id' => 1, 'avatar' => 'avatars/user1.jpg']);

        $this->assertStringContains('/storage/avatars/user1.jpg', $user->avatar_url);
    }
}
```

**Test Pure Logic Directly:**

```php
<?php

// No mocking needed for pure transformation logic
class EmailMutatorTest extends TestCase
{
    /** @test */
    public function it_normalizes_email_addresses()
    {
        $user = new User();
        $user->email = '  JOHN@EXAMPLE.COM  ';

        $this->assertEquals('john@example.com', $user->getAttributes()['email']);
    }
}
```

---

## Senior Developer Best Practices

### Code Review Checklist

When reviewing accessors and mutators, look for these items:

**Functional Correctness:**

- [ ] Are accessors pure functions without side effects?
- [ ] Do mutators only transform input without validation logic?
- [ ] Is error handling appropriate for the use case?
- [ ] Are edge cases (null, empty, extreme values) handled?

**Performance Concerns:**

- [ ] No database queries in accessors
- [ ] No HTTP calls or expensive operations
- [ ] Appropriate caching for complex calculations
- [ ] Conditional appending for expensive accessors

**Security Considerations:**

- [ ] Output is properly escaped for XSS prevention
- [ ] Input is sanitized but not validated
- [ ] Sensitive data is not accidentally exposed
- [ ] Proper use of `$hidden` and `$visible` attributes

**Architectural Alignment:**

- [ ] Business logic is in appropriate service classes
- [ ] Complex transformations use dedicated classes
- [ ] Database-dependent logic uses observers
- [ ] Reusable transformations use custom casts

### Sample PR Comments

```
❌ This accessor performs a database query on every access:

protected function postCount(): Attribute
{
    return Attribute::make(
        get: fn () => $this->posts()->count()
    );
}

✅ Consider using withCount() in the controller instead:

// Controller
$users = User::withCount('posts')->get();

// Then access via: $user->posts_count
```

```
❌ Complex business logic should be extracted:

protected function eligibleForDiscount(): Attribute
{
    return Attribute::make(
        get: function () {
            // 20 lines of complex business rules...
        }
    );
}

✅ Move to dedicated service:

class DiscountEligibilityService
{
    public function isEligible(User $user): bool
    {
        // Complex business rules here
    }
}
```

### Naming Conventions

**Consistent Naming Patterns:**

```php
<?php

class User extends Model
{
    // Computed attributes: use descriptive names
    protected function fullName(): Attribute { /* ... */ }
    protected function avatarUrl(): Attribute { /* ... */ }

    // Formatted versions: add suffix
    protected function priceFormatted(): Attribute { /* ... */ }
    protected function dateFormatted(): Attribute { /* ... */ }

    // Boolean accessors: use "is" or "has" prefix
    protected function isActive(): Attribute { /* ... */ }
    protected function hasSubscription(): Attribute { /* ... */ }

    // Masked/safe versions: add suffix
    protected function emailMasked(): Attribute { /* ... */ }
    protected function phonePartial(): Attribute { /* ... */ }
}
```

### Documentation Standards

**Document Breaking Changes:**

```php
<?php

class User extends Model
{
    /**
     * Full name accessor.
     *
     * @version 2.1.0 Changed format for Japanese locale (last name first)
     * @version 2.0.0 Added email fallback when names are empty
     * @version 1.0.0 Initial implementation
     */
    protected function fullName(): Attribute
    {
        return Attribute::make(
            get: function (): string {
                // Implementation...
            }
        );
    }
}
```

**Document Side Effects:**

```php
<?php

class Product extends Model
{
    /**
     * Price mutator converts dollars to cents for storage.
     *
     * Accepts: string, int, or float
     * Stores: integer (cents)
     *
     * @example
     * $product->price = '29.99'; // Stores 2999
     * $product->price = 30;      // Stores 3000
     */
    protected function price(): Attribute
    {
        return Attribute::make(
            get: fn (int $value) => $value / 100,
            set: fn (string|int|float $value) => (int) (floatval($value) * 100)
        );
    }
}
```

### Performance Optimization Patterns

**Eager Loading with Computed Attributes:**

```php
<?php

class PostController extends Controller
{
    public function index()
    {
        return Post::with(['author:id,first_name,last_name'])
                   ->withCount(['comments', 'likes'])
                   ->get()
                   ->append(['reading_time', 'author_name']);
    }
}

class Post extends Model
{
    protected function readingTime(): Attribute
    {
        return Attribute::make(
            get: fn () => ceil(str_word_count($this->content) / 200) . ' min read'
        );
    }

    protected function authorName(): Attribute
    {
        return Attribute::make(
            get: fn () => $this->author->full_name // Uses eager-loaded relation
        );
    }
}
```

**Selective Attribute Loading:**

```php
<?php

class UserResource extends JsonResource
{
    public function toArray($request): array
    {
        $data = parent::toArray($request);

        // Only append expensive attributes when specifically requested
        if ($request->query('include_stats')) {
            $this->resource->append('activity_score');
        }

        return $data;
    }
}
```

### Versioning Considerations

When updating accessors/mutators in production:

**Backward Compatible Changes:**

- Adding new optional parameters
- Adding fallback behavior
- Improving performance without changing output

**Breaking Changes:**

- Changing return type or format
- Removing fallback behavior
- Changing input requirements

**Migration Strategy:**

```php
<?php

class User extends Model
{
    protected function fullName(): Attribute
    {
        return Attribute::make(
            get: function (): string {
                // Support both old and new formats during migration
                if (config('app.legacy_name_format')) {
                    return $this->first_name . ' ' . $this->last_name;
                }

                // New locale-aware format
                return match (app()->getLocale()) {
                    'ja' => $this->last_name . ' ' . $this->first_name,
                    default => $this->first_name . ' ' . $this->last_name
                };
            }
        );
    }
}
```

---

## Summary & Key Takeaways

### Essential Recommendations

• **Keep accessors pure**: No database queries, HTTP calls, or side effects. Accessors should be predictable transformations that can be called multiple times safely.

• **Use mutators for normalization only**: Handle data formatting and sanitization, but leave validation to Form Requests and business logic to service classes.

• **Choose the right implementation style**: Use classic methods for complex logic requiring multiple statements, Attribute classes for simple transformations and better performance.

• **Leverage Eloquent integration**: Combine with `$casts`, `$appends`, and `$hidden` for powerful serialization control. Use conditional appending for expensive computations.

• **Optimize for performance**: Eager load relationships, cache expensive calculations, and avoid appending attributes globally when they're only needed in specific contexts.

• **Maintain security awareness**: Always escape output for display, sanitize input appropriately, and use `$hidden` to protect sensitive data from accidental exposure.

• **Extract complex logic**: Move business rules to service classes, use observers for database-dependent operations, and employ custom casts for reusable transformations across models.

• **Test thoroughly**: Write focused unit tests for transformation logic, integration tests for database-dependent features, and always test edge cases and boundary conditions.

The key to effective accessors and mutators is understanding they're data transformation tools, not business logic containers. When used appropriately, they provide a clean, maintainable way to handle the gap between your database schema and application needs while keeping your models focused and your code DRY.
