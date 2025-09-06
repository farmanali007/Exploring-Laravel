# Laravel Eloquent One-to-One Relationships: Complete Guide

## Table of Contents

1. [Definition & Conceptual Understanding](#definition--conceptual-understanding)
2. [Database Schema & Visual Diagrams](#database-schema--visual-diagrams)
3. [Basic Implementation](#basic-implementation)
4. [Real-World Examples](#real-world-examples)
5. [Advanced Implementation Techniques](#advanced-implementation-techniques)
6. [Performance Optimization & Best Practices](#performance-optimization--best-practices)
7. [Common Mistakes & Recommended Practices](#common-mistakes--recommended-practices)
8. [Advanced Scenarios](#advanced-scenarios)
9. [Testing Strategies](#testing-strategies)
10. [Security & Maintainability](#security--maintainability)

---

## Definition & Conceptual Understanding

### What is a One-to-One Relationship?

A **One-to-One relationship** in Laravel Eloquent represents a database relationship where:

- **One record** in table A is associated with **exactly one record** in table B
- **One record** in table B is associated with **exactly one record** in table A
- The relationship is **bidirectional** but **mutually exclusive**

### Key Characteristics:

- **Uniqueness**: Each foreign key value appears at most once
- **Bidirectionality**: Can be navigated from both sides
- **Optional or Required**: Can be nullable or non-nullable
- **Data Normalization**: Often used to split large tables or separate concerns

### Laravel Eloquent Methods:

- `hasOne()`: Defines the "owning" side of the relationship
- `belongsTo()`: Defines the "owned" side of the relationship

---

## Database Schema & Visual Diagrams

### Basic Schema Structure

```
┌─────────────────┐         ┌─────────────────┐
│     users       │         │   user_profiles │
├─────────────────┤         ├─────────────────┤
│ id (PK)         │────────→│ id (PK)         │
│ email           │         │ user_id (FK)    │
│ password        │         │ first_name      │
│ created_at      │         │ last_name       │
│ updated_at      │         │ bio             │
└─────────────────┘         │ avatar          │
                            │ created_at      │
                            │ updated_at      │
                            └─────────────────┘

Relationship: User (1) ←→ (1) UserProfile
```

### Eloquent Relationship Mapping

```
User Model (Parent/Owner)
├── hasOne(UserProfile::class)
└── Foreign Key: user_profiles.user_id → users.id

UserProfile Model (Child/Owned)
├── belongsTo(User::class)
└── References: user_profiles.user_id → users.id
```

### Advanced Schema with Custom Keys

```
┌─────────────────┐         ┌─────────────────┐
│    companies    │         │   company_hqs   │
├─────────────────┤         ├─────────────────┤
│ id (PK)         │         │ id (PK)         │
│ company_code    │────────→│ company_ref     │
│ name            │         │ address         │
│ industry        │         │ city            │
└─────────────────┘         │ country         │
                            │ phone           │
                            └─────────────────┘

Custom Relationship: companies.company_code ←→ company_hqs.company_ref
```

---

## Basic Implementation

### Step 1: Database Migrations

```php
// Create users table
Schema::create('users', function (Blueprint $table) {
    $table->id();
    $table->string('email')->unique();
    $table->string('password');
    $table->timestamps();
});

// Create user_profiles table
Schema::create('user_profiles', function (Blueprint $table) {
    $table->id();
    $table->foreignId('user_id')->constrained()->onDelete('cascade');
    $table->string('first_name');
    $table->string('last_name');
    $table->text('bio')->nullable();
    $table->string('avatar')->nullable();
    $table->timestamps();

    // Ensure one-to-one relationship
    $table->unique('user_id');
});
```

### Step 2: Eloquent Models

```php
// app/Models/User.php
class User extends Authenticatable
{
    protected $fillable = ['email', 'password'];

    /**
     * Get the user's profile
     */
    public function profile(): HasOne
    {
        return $this->hasOne(UserProfile::class);
    }
}

// app/Models/UserProfile.php
class UserProfile extends Model
{
    protected $fillable = [
        'user_id', 'first_name', 'last_name', 'bio', 'avatar'
    ];

    /**
     * Get the user that owns the profile
     */
    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }
}
```

### Step 3: Basic Usage Examples

```php
// Creating relationships
$user = User::create([
    'email' => 'john@example.com',
    'password' => Hash::make('password')
]);

$profile = $user->profile()->create([
    'first_name' => 'John',
    'last_name' => 'Doe',
    'bio' => 'Software Developer'
]);

// Accessing relationships
$user = User::find(1);
$profile = $user->profile; // Returns UserProfile or null

$profile = UserProfile::find(1);
$user = $profile->user; // Returns User

// Checking relationship existence
if ($user->profile) {
    echo $user->profile->first_name;
}

// Using relationship queries
$usersWithProfiles = User::has('profile')->get();
$usersWithoutProfiles = User::doesntHave('profile')->get();
```

---

## Real-World Examples

### Example 1: User-Profile System

```php
// Migration
Schema::create('user_profiles', function (Blueprint $table) {
    $table->id();
    $table->foreignId('user_id')->constrained()->onDelete('cascade');
    $table->string('first_name');
    $table->string('last_name');
    $table->date('date_of_birth')->nullable();
    $table->enum('gender', ['male', 'female', 'other'])->nullable();
    $table->string('phone')->nullable();
    $table->text('bio')->nullable();
    $table->string('avatar')->nullable();
    $table->json('social_links')->nullable();
    $table->timestamps();

    $table->unique('user_id');
});

// Models with advanced features
class User extends Authenticatable
{
    public function profile(): HasOne
    {
        return $this->hasOne(UserProfile::class);
    }

    // Accessor for full name through relationship
    public function getFullNameAttribute(): string
    {
        return $this->profile
            ? "{$this->profile->first_name} {$this->profile->last_name}"
            : 'No Profile';
    }

    // Scope for users with complete profiles
    public function scopeWithCompleteProfile($query)
    {
        return $query->whereHas('profile', function ($q) {
            $q->whereNotNull(['first_name', 'last_name', 'date_of_birth']);
        });
    }
}

class UserProfile extends Model
{
    protected $fillable = [
        'user_id', 'first_name', 'last_name', 'date_of_birth',
        'gender', 'phone', 'bio', 'avatar', 'social_links'
    ];

    protected $casts = [
        'date_of_birth' => 'date',
        'social_links' => 'array'
    ];

    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }

    // Accessor for age
    public function getAgeAttribute(): ?int
    {
        return $this->date_of_birth?->age;
    }
}
```

### Example 2: Company-Headquarters System

```php
// Migration
Schema::create('company_headquarters', function (Blueprint $table) {
    $table->id();
    $table->string('company_code'); // Custom foreign key
    $table->string('address');
    $table->string('city');
    $table->string('state');
    $table->string('country');
    $table->string('postal_code');
    $table->decimal('latitude', 10, 8)->nullable();
    $table->decimal('longitude', 11, 8)->nullable();
    $table->timestamps();

    $table->unique('company_code');
    $table->foreign('company_code')->references('code')->on('companies');
});

// Models with custom keys
class Company extends Model
{
    protected $primaryKey = 'code';
    protected $keyType = 'string';
    public $incrementing = false;

    public function headquarters(): HasOne
    {
        return $this->hasOne(CompanyHeadquarters::class, 'company_code', 'code');
    }
}

class CompanyHeadquarters extends Model
{
    protected $fillable = [
        'company_code', 'address', 'city', 'state',
        'country', 'postal_code', 'latitude', 'longitude'
    ];

    public function company(): BelongsTo
    {
        return $this->belongsTo(Company::class, 'company_code', 'code');
    }

    // Accessor for full address
    public function getFullAddressAttribute(): string
    {
        return "{$this->address}, {$this->city}, {$this->state} {$this->postal_code}, {$this->country}";
    }
}
```

---

## Advanced Implementation Techniques

### Custom Foreign Keys and Local Keys

```php
class User extends Model
{
    // Custom foreign key
    public function profile(): HasOne
    {
        return $this->hasOne(UserProfile::class, 'user_uuid', 'uuid');
    }
}

class UserProfile extends Model
{
    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class, 'user_uuid', 'uuid');
    }
}
```

### Default Models and Null Object Pattern

```php
class User extends Model
{
    public function profile(): HasOne
    {
        return $this->hasOne(UserProfile::class)->withDefault([
            'first_name' => 'Guest',
            'last_name' => 'User',
            'bio' => 'No profile information available'
        ]);
    }

    // Or with a factory method
    public function profile(): HasOne
    {
        return $this->hasOne(UserProfile::class)->withDefault(function ($profile, $user) {
            $profile->first_name = 'User';
            $profile->last_name = $user->id;
        });
    }
}

// Usage - always returns a profile object
$user = User::find(1);
echo $user->profile->first_name; // Never throws null error
```

### Polymorphic One-to-One Relationships

```php
// Migration for polymorphic addresses
Schema::create('addresses', function (Blueprint $table) {
    $table->id();
    $table->morphs('addressable'); // Creates addressable_id and addressable_type
    $table->string('street');
    $table->string('city');
    $table->string('country');
    $table->timestamps();
});

// Models
class User extends Model
{
    public function address(): MorphOne
    {
        return $this->morphOne(Address::class, 'addressable');
    }
}

class Company extends Model
{
    public function address(): MorphOne
    {
        return $this->morphOne(Address::class, 'addressable');
    }
}

class Address extends Model
{
    public function addressable(): MorphTo
    {
        return $this->morphTo();
    }
}

// Usage
$user = User::find(1);
$userAddress = $user->address;

$company = Company::find(1);
$companyAddress = $company->address;
```

### Conditional Relationships

```php
class User extends Model
{
    public function activeProfile(): HasOne
    {
        return $this->hasOne(UserProfile::class)->where('is_active', true);
    }

    public function primaryAddress(): HasOne
    {
        return $this->hasOne(Address::class)->where('is_primary', true);
    }
}
```

---

## Performance Optimization & Best Practices

### Eager Loading Strategies

```php
// ❌ N+1 Query Problem
$users = User::all();
foreach ($users as $user) {
    echo $user->profile->first_name; // Executes query for each user
}

// ✅ Eager Loading Solution
$users = User::with('profile')->get();
foreach ($users as $user) {
    echo $user->profile->first_name; // Single query for all profiles
}

// ✅ Conditional Eager Loading
$users = User::when($needsProfiles, function ($query) {
    return $query->with('profile');
})->get();

// ✅ Lazy Eager Loading
$users = User::all();
$users->load('profile'); // Load relationships after initial query
```

### Advanced Eager Loading Techniques

```php
// Select specific columns to reduce memory usage
$users = User::with(['profile:id,user_id,first_name,last_name'])->get();

// Nested eager loading with constraints
$users = User::with(['profile' => function ($query) {
    $query->where('is_public', true)
          ->orderBy('updated_at', 'desc');
}])->get();

// Eager loading with counts
$users = User::withCount('profile as has_profile')->get();

// Multiple relationship loading
$users = User::with(['profile', 'orders', 'preferences'])->get();
```

### Database Indexing Strategies

```php
// In migration
Schema::create('user_profiles', function (Blueprint $table) {
    $table->id();
    $table->foreignId('user_id')->constrained();

    // Essential indexes for one-to-one
    $table->unique('user_id'); // Enforces one-to-one constraint
    $table->index('user_id');  // Speeds up lookups

    // Composite indexes for filtered queries
    $table->index(['user_id', 'is_active']);
    $table->index(['created_at', 'user_id']);
});
```

### Query Optimization Techniques

```php
// Use exists() for checking relationship existence
$hasProfile = User::whereExists(function ($query) {
    $query->select(DB::raw(1))
          ->from('user_profiles')
          ->whereColumn('user_profiles.user_id', 'users.id');
})->count();

// Efficient relationship queries
$usersWithProfiles = User::whereHas('profile')->get();
$usersWithoutProfiles = User::whereDoesntHave('profile')->get();

// Join queries for better performance
$users = User::join('user_profiles', 'users.id', '=', 'user_profiles.user_id')
             ->select('users.*', 'user_profiles.first_name', 'user_profiles.last_name')
             ->get();
```

### Memory Management

```php
// Use chunk() for large datasets
User::with('profile')->chunk(1000, function ($users) {
    foreach ($users as $user) {
        // Process user and profile
    }
});

// Use cursor() for memory-efficient iteration
foreach (User::with('profile')->cursor() as $user) {
    // Process without loading entire collection into memory
}

// Clean up relationships after processing
$users = User::with('profile')->get();
// Process users...
unset($users); // Free memory
```

---

## Common Mistakes & Recommended Practices

### ❌ Common Mistakes (Don'ts)

#### 1. Forgetting Unique Constraints

```php
// ❌ Missing unique constraint allows multiple profiles per user
Schema::create('user_profiles', function (Blueprint $table) {
    $table->foreignId('user_id')->constrained();
    // Missing: $table->unique('user_id');
});
```

#### 2. Not Handling Null Relationships

```php
// ❌ Will throw error if profile doesn't exist
$user = User::find(1);
echo $user->profile->first_name; // Error if profile is null
```

#### 3. Creating N+1 Queries

```php
// ❌ N+1 query problem
$users = User::all();
foreach ($users as $user) {
    if ($user->profile) {
        echo $user->profile->first_name;
    }
}
```

#### 4. Improper Cascade Handling

```php
// ❌ Not considering cascade effects
Schema::create('user_profiles', function (Blueprint $table) {
    $table->foreignId('user_id')->constrained(); // No cascade rules
});
```

#### 5. Inconsistent Relationship Definitions

```php
// ❌ Mismatched relationship definitions
class User extends Model
{
    public function profile(): HasOne
    {
        return $this->hasOne(UserProfile::class, 'wrong_key');
    }
}
```

### ✅ Recommended Practices (Do's)

#### 1. Always Add Unique Constraints

```php
// ✅ Proper one-to-one constraint
Schema::create('user_profiles', function (Blueprint $table) {
    $table->foreignId('user_id')->constrained()->onDelete('cascade');
    $table->unique('user_id'); // Enforces one-to-one
});
```

#### 2. Safe Null Handling

```php
// ✅ Safe null handling
class User extends Model
{
    public function profile(): HasOne
    {
        return $this->hasOne(UserProfile::class)->withDefault();
    }

    public function getDisplayNameAttribute(): string
    {
        return $this->profile?->first_name ?? 'Guest User';
    }
}
```

#### 3. Use Eager Loading

```php
// ✅ Proper eager loading
$users = User::with('profile')->get();
foreach ($users as $user) {
    echo $user->profile->first_name ?? 'No Profile';
}
```

#### 4. Implement Proper Cascading

```php
// ✅ Proper cascade configuration
Schema::create('user_profiles', function (Blueprint $table) {
    $table->foreignId('user_id')
          ->constrained()
          ->onUpdate('cascade')
          ->onDelete('cascade');
});
```

#### 5. Consistent Model Definitions

```php
// ✅ Consistent relationship definitions
class User extends Model
{
    public function profile(): HasOne
    {
        return $this->hasOne(UserProfile::class);
    }
}

class UserProfile extends Model
{
    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }
}
```

---

## Advanced Scenarios

### When to Use One-to-One vs Other Relationships

#### One-to-One Use Cases:

- **User Profile**: User ↔ UserProfile (personal information)
- **Product Details**: Product ↔ ProductSpecification (extended attributes)
- **Company Headquarters**: Company ↔ CompanyHeadquarters (single main office)
- **Payment Method**: User ↔ DefaultPaymentMethod (single default)

#### vs One-to-Many:

```php
// ✅ One-to-One: User has one primary address
public function primaryAddress(): HasOne
{
    return $this->hasOne(Address::class)->where('is_primary', true);
}

// ✅ One-to-Many: User has many addresses
public function addresses(): HasMany
{
    return $this->hasMany(Address::class);
}
```

#### vs Polymorphic:

```php
// ✅ Polymorphic: Multiple models can have addresses
class User extends Model
{
    public function address(): MorphOne
    {
        return $this->morphOne(Address::class, 'addressable');
    }
}

class Company extends Model
{
    public function address(): MorphOne
    {
        return $this->morphOne(Address::class, 'addressable');
    }
}
```

### Handling Nullable Relationships

```php
class User extends Model
{
    public function profile(): HasOne
    {
        return $this->hasOne(UserProfile::class);
    }

    // Helper method for nullable checks
    public function hasProfile(): bool
    {
        return $this->profile !== null;
    }

    // Safe accessor
    public function getProfileNameAttribute(): string
    {
        return $this->profile?->first_name ?? 'Unknown';
    }
}

// Query patterns for nullable relationships
$usersWithProfiles = User::whereNotNull('profile_id')->get();
$usersWithoutProfiles = User::whereNull('profile_id')->get();
$usersWithCompleteProfiles = User::whereHas('profile', function ($query) {
    $query->whereNotNull(['first_name', 'last_name']);
})->get();
```

### Cascading Deletes and Updates

```php
// Database-level cascading
Schema::create('user_profiles', function (Blueprint $table) {
    $table->foreignId('user_id')
          ->constrained()
          ->onUpdate('cascade')    // Update profile when user ID changes
          ->onDelete('cascade');   // Delete profile when user is deleted
});

// Model-level cascading
class User extends Model
{
    protected static function booted()
    {
        static::deleting(function ($user) {
            // Custom logic before deleting
            $user->profile?->delete();
        });
    }
}

// Service-level cascading
class UserService
{
    public function deleteUser(User $user): bool
    {
        DB::transaction(function () use ($user) {
            // Delete related data
            $user->profile?->delete();
            $user->preferences?->delete();

            // Delete user
            $user->delete();
        });

        return true;
    }
}
```

### Advanced Query Scenarios

```php
// Complex filtering through relationships
$users = User::whereHas('profile', function ($query) {
    $query->where('age', '>=', 18)
          ->where('country', 'US')
          ->whereNotNull('phone');
})->with(['profile' => function ($query) {
    $query->select('id', 'user_id', 'first_name', 'last_name', 'age');
}])->get();

// Conditional relationships
$users = User::with([
    'profile' => function ($query) use ($includePrivate) {
        if (!$includePrivate) {
            $query->where('is_public', true);
        }
    }
])->get();

// Aggregations through relationships
$avgAge = User::join('user_profiles', 'users.id', '=', 'user_profiles.user_id')
              ->avg('user_profiles.age');

// Subquery relationships
$users = User::addSelect([
    'profile_created_at' => UserProfile::select('created_at')
        ->whereColumn('user_id', 'users.id')
        ->limit(1)
])->get();
```

---

## Testing Strategies

### Model Testing

```php
// tests/Unit/UserTest.php
class UserTest extends TestCase
{
    use RefreshDatabase;

    /** @test */
    public function user_can_have_a_profile()
    {
        $user = User::factory()->create();
        $profile = UserProfile::factory()->create(['user_id' => $user->id]);

        $this->assertInstanceOf(UserProfile::class, $user->profile);
        $this->assertEquals($profile->id, $user->profile->id);
    }

    /** @test */
    public function profile_belongs_to_user()
    {
        $user = User::factory()->create();
        $profile = UserProfile::factory()->create(['user_id' => $user->id]);

        $this->assertInstanceOf(User::class, $profile->user);
        $this->assertEquals($user->id, $profile->user->id);
    }

    /** @test */
    public function user_can_exist_without_profile()
    {
        $user = User::factory()->create();

        $this->assertNull($user->profile);
    }

    /** @test */
    public function deleting_user_deletes_profile()
    {
        $user = User::factory()->create();
        $profile = UserProfile::factory()->create(['user_id' => $user->id]);

        $user->delete();

        $this->assertDatabaseMissing('user_profiles', ['id' => $profile->id]);
    }
}
```

### Factory Definitions

```php
// database/factories/UserFactory.php
class UserFactory extends Factory
{
    public function definition()
    {
        return [
            'email' => $this->faker->unique()->safeEmail(),
            'password' => Hash::make('password'),
        ];
    }

    public function withProfile(array $profileAttributes = [])
    {
        return $this->afterCreating(function (User $user) use ($profileAttributes) {
            UserProfile::factory()->create(
                array_merge(['user_id' => $user->id], $profileAttributes)
            );
        });
    }
}

// database/factories/UserProfileFactory.php
class UserProfileFactory extends Factory
{
    public function definition()
    {
        return [
            'user_id' => User::factory(),
            'first_name' => $this->faker->firstName(),
            'last_name' => $this->faker->lastName(),
            'bio' => $this->faker->paragraph(),
        ];
    }
}

// Usage in tests
$userWithProfile = User::factory()->withProfile()->create();
$profile = UserProfile::factory()->create();
```

### Integration Testing

```php
// tests/Feature/UserProfileTest.php
class UserProfileTest extends TestCase
{
    use RefreshDatabase;

    /** @test */
    public function user_can_create_profile()
    {
        $user = User::factory()->create();

        $response = $this->actingAs($user)->post('/profile', [
            'first_name' => 'John',
            'last_name' => 'Doe',
            'bio' => 'Developer'
        ]);

        $response->assertRedirect('/profile');
        $this->assertDatabaseHas('user_profiles', [
            'user_id' => $user->id,
            'first_name' => 'John'
        ]);
    }

    /** @test */
    public function user_cannot_create_multiple_profiles()
    {
        $user = User::factory()->withProfile()->create();

        $response = $this->actingAs($user)->post('/profile', [
            'first_name' => 'Jane',
            'last_name' => 'Smith'
        ]);

        $response->assertStatus(422);
        $this->assertEquals(1, UserProfile::where('user_id', $user->id)->count());
    }
}
```

---

## Security & Maintainability

### Security Considerations

#### 1. Mass Assignment Protection

```php
class UserProfile extends Model
{
    protected $fillable = [
        'first_name', 'last_name', 'bio', 'avatar'
    ];

    protected $guarded = [
        'user_id', 'is_verified', 'created_at', 'updated_at'
    ];
}
```

#### 2. Authorization Checks

```php
// Policy for profile access
class UserProfilePolicy
{
    public function view(User $user, UserProfile $profile): bool
    {
        return $user->id === $profile->user_id || $profile->is_public;
    }

    public function update(User $user, UserProfile $profile): bool
    {
        return $user->id === $profile->user_id;
    }
}

// Usage in controller
public function show(UserProfile $profile)
{
    $this->authorize('view', $profile);
    return view('profile.show', compact('profile'));
}
```

#### 3. Input Validation

```php
class UpdateProfileRequest extends FormRequest
{
    public function rules()
    {
        return [
            'first_name' => 'required|string|max:255',
            'last_name' => 'required|string|max:255',
            'bio' => 'nullable|string|max:1000',
            'avatar' => 'nullable|image|max:2048',
        ];
    }

    public function authorize()
    {
        return $this->user()->can('update', $this->route('profile'));
    }
}
```

### Maintainability Best Practices

#### 1. Use Resource Classes

```php
class UserProfileResource extends JsonResource
{
    public function toArray($request)
    {
        return [
            'id' => $this->id,
            'full_name' => $this->first_name . ' ' . $this->last_name,
            'bio' => $this->bio,
            'avatar_url' => $this->avatar ? Storage::url($this->avatar) : null,
            'user' => new UserResource($this->whenLoaded('user')),
        ];
    }
}
```

#### 2. Service Layer Pattern

```php
class UserProfileService
{
    public function createProfile(User $user, array $data): UserProfile
    {
        if ($user->profile) {
            throw new ValidationException('User already has a profile');
        }

        return DB::transaction(function () use ($user, $data) {
            $profile = $user->profile()->create($data);

            // Additional business logic
            event(new ProfileCreated($profile));

            return $profile;
        });
    }

    public function updateProfile(UserProfile $profile, array $data): UserProfile
    {
        return DB::transaction(function () use ($profile, $data) {
            $profile->update($data);

            // Additional business logic
            Cache::forget("user_profile_{$profile->user_id}");

            return $profile->fresh();
        });
    }
}
```

#### 3. Repository Pattern

```php
interface UserProfileRepositoryInterface
{
    public function findByUser(User $user): ?UserProfile;
    public function createForUser(User $user, array $data): UserProfile;
    public function updateProfile(UserProfile $profile, array $data): UserProfile;
}

class EloquentUserProfileRepository implements UserProfileRepositoryInterface
{
    public function findByUser(User $user): ?UserProfile
    {
        return $user->profile;
    }

    public function createForUser(User $user, array $data): UserProfile
    {
        return $user->profile()->create($data);
    }

    public function updateProfile(UserProfile $profile, array $data): UserProfile
    {
        $profile->update($data);
        return $profile->fresh();
    }
}
```

#### 4. Caching Strategies

```php
class User extends Model
{
    public function profile(): HasOne
    {
        return $this->hasOne(UserProfile::class);
    }

    public function getCachedProfile(): ?UserProfile
    {
        return Cache::remember(
            "user_profile_{$this->id}",
            3600,
            fn() => $this->profile
        );
    }
}

// Cache invalidation
class UserProfile extends Model
{
    protected static function booted()
    {
        static::saved(function ($profile) {
            Cache::forget("user_profile_{$profile->user_id}");
        });

        static::deleted(function ($profile) {
            Cache::forget("user_profile_{$profile->user_id}");
        });
    }
}
```

---

## Conclusion

Laravel Eloquent's One-to-One relationships provide a powerful way to model exclusive relationships between entities. Key takeaways for senior developers:

1. **Always enforce uniqueness** at the database level with unique constraints
2. **Use eager loading** to prevent N+1 query problems
3. **Handle null relationships gracefully** with proper null checks or default models
4. **Implement proper cascading** for data integrity
5. **Consider performance implications** when designing relationships
6. **Use appropriate indexes** for optimal query performance
7. **Test relationships thoroughly** including edge cases
8. **Apply security best practices** with proper authorization and validation

By following these patterns and practices, you'll build maintainable, performant, and secure one-to-one relationships in your Laravel applications.
