# Laravel Model Conventions: Complete Masterclass

## 1. Introduction to Models

Laravel models are the **M** in MVC - they represent your application's data and business logic. Built on top of the Eloquent ORM, models provide an elegant ActiveRecord implementation for working with your database.

### What Models Represent

Models are **more than just database representations**. They encapsulate:

- **Data structure** and validation rules
- **Business logic** related to the entity
- **Relationships** between different entities
- **Query scopes** for common database operations
- **Attribute casting** and transformation

### The Eloquent Philosophy

```php
// Eloquent embraces convention over configuration
$user = User::find(1);              // Find user by ID
$user->posts()->create($data);      // Create related post
$activeUsers = User::active()->get(); // Use query scopes
```

Laravel's conventions make code predictable and reduce boilerplate, but understanding **when and how** to override these conventions is what separates mid-level from senior developers.

---

## 2. Naming Conventions

### 2.1 Model Class Names

**Convention**: Singular, PascalCase

```php
✅ Good Examples:
User.php          → class User
BlogPost.php      → class BlogPost
OrderItem.php     → class OrderItem
PaymentMethod.php → class PaymentMethod

❌ Bad Examples:
Users.php         → class Users     (plural)
blog_post.php     → class blog_post (snake_case)
orderitem.php     → class orderitem (no separation)
```

### 2.2 File and Directory Organization

```php
// Basic structure
app/Models/User.php
app/Models/Post.php
app/Models/Category.php

// Large applications - organize by domain
app/Models/User/User.php
app/Models/User/Profile.php
app/Models/User/Permission.php

app/Models/Blog/Post.php
app/Models/Blog/Category.php
app/Models/Blog/Tag.php

app/Models/Ecommerce/Product.php
app/Models/Ecommerce/Order.php
app/Models/Ecommerce/Cart.php
```

### 2.3 Senior-Level Naming Strategy

```php
// For complex domains, use descriptive names
✅ Good:
CustomerSubscription.php
ProductVariant.php
ShippingAddress.php
PaymentTransaction.php

❌ Avoid generic names:
Data.php
Info.php
Item.php
Record.php
```

---

## 3. Database Table Conventions

### 3.1 Default Table Naming

Laravel automatically pluralizes and snake_cases your model name:

```php
Model Name     → Table Name
User           → users
BlogPost       → blog_posts
OrderItem      → order_items
PaymentMethod  → payment_methods
```

### 3.2 When to Override Table Names

```php
// Override when legacy database doesn't follow conventions
class User extends Model
{
    protected $table = 'customer_accounts'; // Legacy table name
}

// Or when using prefixes
class BlogPost extends Model
{
    protected $table = 'cms_posts';
}

// Multiple databases with different schemas
class Product extends Model
{
    protected $connection = 'inventory';
    protected $table = 'product_catalog';
}
```

### 3.3 Advanced Table Configuration

```php
class AuditLog extends Model
{
    protected $connection = 'audit_db';
    protected $table = 'system_audit_logs';

    // Partition-specific logic for large datasets
    public function getTable()
    {
        $month = date('Y_m');
        return "audit_logs_{$month}";
    }
}
```

---

## 4. Primary Keys & Conventions

### 4.1 Default Primary Key Convention

```php
// Laravel assumes:
// - Column name: 'id'
// - Type: auto-incrementing integer
// - Unsigned: true

class User extends Model
{
    // No configuration needed for standard setup
}
```

### 4.2 Customizing Primary Keys

```php
// Non-standard primary key name
class Product extends Model
{
    protected $primaryKey = 'product_id';
}

// UUID primary keys
class Order extends Model
{
    protected $primaryKey = 'uuid';
    protected $keyType = 'string';
    public $incrementing = false;

    protected static function boot()
    {
        parent::boot();

        static::creating(function ($model) {
            $model->uuid = (string) Str::uuid();
        });
    }
}

// Composite primary keys (advanced)
class UserRole extends Model
{
    protected $primaryKey = ['user_id', 'role_id'];
    public $incrementing = false;

    // Override getKey() for composite keys
    public function getKey()
    {
        return [$this->user_id, $this->role_id];
    }
}
```

### 4.3 Senior-Level Primary Key Strategies

```php
// Strategy pattern for different ID types
abstract class BaseModel extends Model
{
    public static function boot()
    {
        parent::boot();

        if (static::usesUuids()) {
            static::creating(function ($model) {
                $model->{$model->getKeyName()} = (string) Str::uuid();
            });
        }
    }

    protected static function usesUuids(): bool
    {
        return false;
    }
}

class User extends BaseModel
{
    protected $keyType = 'string';
    public $incrementing = false;

    protected static function usesUuids(): bool
    {
        return true;
    }
}
```

---

## 5. Timestamps

### 5.1 Default Timestamp Behavior

```php
// Laravel automatically manages these columns:
created_at TIMESTAMP NULL
updated_at TIMESTAMP NULL

class User extends Model
{
    // Timestamps are enabled by default
    // public $timestamps = true; // Not needed
}
```

### 5.2 Customizing Timestamps

```php
// Disable timestamps
class ViewLog extends Model
{
    public $timestamps = false;
}

// Custom timestamp column names
class Product extends Model
{
    const CREATED_AT = 'date_created';
    const UPDATED_AT = 'date_modified';
}

// Only track creation time
class AccessLog extends Model
{
    const UPDATED_AT = null; // Disable updated_at
}
```

### 5.3 Advanced Timestamp Handling

```php
// Multiple timestamp tracking
class Document extends Model
{
    protected $dates = [
        'created_at',
        'updated_at',
        'published_at',
        'archived_at',
        'deleted_at'
    ];

    // Custom timestamp for specific events
    public function markAsPublished()
    {
        $this->update([
            'published_at' => now(),
            'status' => 'published'
        ]);
    }
}

// Timezone-aware timestamps
class Event extends Model
{
    protected $casts = [
        'created_at' => 'datetime:Y-m-d H:i:s T',
        'starts_at' => 'datetime',
        'ends_at' => 'datetime',
    ];

    // Convert to user's timezone
    public function getStartsAtInTimezone($timezone)
    {
        return $this->starts_at->setTimezone($timezone);
    }
}
```

---

## 6. Fillable & Guarded Attributes

### 6.1 Understanding Mass Assignment Protection

Mass assignment protection prevents malicious users from modifying unintended model attributes.

```php
❌ Vulnerable without protection:
$user = User::create($request->all()); // Dangerous!
// User could inject: {'is_admin': true, 'email_verified_at': '2023-01-01'}

✅ Safe with proper protection:
class User extends Model
{
    protected $fillable = ['name', 'email', 'password'];
}
$user = User::create($request->only(['name', 'email', 'password']));
```

### 6.2 Fillable vs Guarded Strategies

```php
// Strategy 1: Whitelist approach (Recommended)
class User extends Model
{
    protected $fillable = [
        'name',
        'email',
        'password',
        'phone',
        'address'
    ];
}

// Strategy 2: Blacklist approach
class User extends Model
{
    protected $guarded = [
        'id',
        'is_admin',
        'email_verified_at',
        'remember_token'
    ];
}

❌ Never do this in production:
class User extends Model
{
    protected $guarded = []; // Disables ALL protection
}
```

### 6.3 Advanced Mass Assignment Patterns

```php
// Role-based fillable attributes
class User extends Model
{
    protected $fillable = ['name', 'email'];

    public function getFillableForRole($role): array
    {
        $baseFillable = $this->getFillable();

        if ($role === 'admin') {
            return array_merge($baseFillable, ['is_admin', 'email_verified_at']);
        }

        return $baseFillable;
    }

    public function fillForRole(array $attributes, string $role): self
    {
        $this->fillable($this->getFillableForRole($role));
        return $this->fill($attributes);
    }
}

// Context-specific fillable
class Product extends Model
{
    protected $fillable = ['name', 'description', 'price'];

    // Admin can modify sensitive fields
    public function fillAsAdmin(array $attributes): self
    {
        $this->fillable(array_merge($this->fillable, [
            'cost_price',
            'supplier_id',
            'internal_notes'
        ]));

        return $this->fill($attributes);
    }
}
```

---

## 7. Relationships Conventions

### 7.1 Relationship Naming Conventions

```php
// Method names should describe the relationship
class User extends Model
{
    // One-to-One: singular noun
    public function profile() // user_id in profiles table
    {
        return $this->hasOne(Profile::class);
    }

    // One-to-Many: plural noun
    public function posts() // user_id in posts table
    {
        return $this->hasMany(Post::class);
    }

    // Many-to-Many: plural noun
    public function roles() // user_role pivot table
    {
        return $this->belongsToMany(Role::class);
    }

    // Belongs To: singular noun (matches foreign key minus _id)
    public function company() // company_id in users table
    {
        return $this->belongsTo(Company::class);
    }
}
```

### 7.2 Foreign Key Conventions

```php
// Laravel automatically assumes:
Model Name → Foreign Key
User       → user_id
BlogPost   → blog_post_id
Company    → company_id

// Override when needed:
class Post extends Model
{
    public function author() // Custom foreign key
    {
        return $this->belongsTo(User::class, 'author_id');
    }

    public function category() // Custom keys for both tables
    {
        return $this->belongsTo(Category::class, 'cat_id', 'category_id');
    }
}
```

### 7.3 Pivot Table Conventions

```php
// Naming: alphabetical order, singular, snake_case
role_user (not user_role)
category_post
product_tag

// With timestamps and extra data:
class User extends Model
{
    public function roles()
    {
        return $this->belongsToMany(Role::class)
                    ->withTimestamps()
                    ->withPivot(['assigned_by', 'assigned_at', 'notes']);
    }
}

// Custom pivot model for complex relationships
class UserRole extends Pivot
{
    protected $table = 'user_roles';

    protected $casts = [
        'assigned_at' => 'datetime',
        'permissions' => 'array'
    ];

    public function assignedBy()
    {
        return $this->belongsTo(User::class, 'assigned_by');
    }
}

// Using custom pivot
class User extends Model
{
    public function roles()
    {
        return $this->belongsToMany(Role::class)
                    ->using(UserRole::class)
                    ->withPivot(['assigned_by', 'assigned_at', 'permissions']);
    }
}
```

### 7.4 Advanced Relationship Patterns

```php
// Polymorphic relationships
class Comment extends Model
{
    // Convention: {relation}_type and {relation}_id
    public function commentable() // commentable_type, commentable_id
    {
        return $this->morphTo();
    }
}

class Post extends Model
{
    public function comments()
    {
        return $this->morphMany(Comment::class, 'commentable');
    }
}

// Self-referencing relationships
class Category extends Model
{
    public function parent()
    {
        return $this->belongsTo(Category::class, 'parent_id');
    }

    public function children()
    {
        return $this->hasMany(Category::class, 'parent_id');
    }

    // Get all descendants recursively
    public function descendants()
    {
        return $this->children()->with('descendants');
    }
}

// Through relationships
class User extends Model
{
    // Get posts through company
    public function companyPosts()
    {
        return $this->hasManyThrough(
            Post::class,     // Final model
            Company::class,  // Intermediate model
            'user_id',       // Foreign key on companies table
            'company_id',    // Foreign key on posts table
            'id',           // Local key on users table
            'id'            // Local key on companies table
        );
    }
}
```

---

## 8. Attribute Casting & Mutators

### 8.1 Basic Casting

```php
class User extends Model
{
    protected $casts = [
        'email_verified_at' => 'datetime',
        'settings' => 'array',
        'is_admin' => 'boolean',
        'salary' => 'decimal:2',
        'birth_date' => 'date',
        'metadata' => 'json'
    ];
}

// Usage:
$user = User::find(1);
$user->is_admin;        // Returns boolean, not string
$user->settings;        // Returns array, not JSON string
$user->email_verified_at; // Returns Carbon instance
```

### 8.2 Custom Casts (Laravel 9+)

```php
// Create custom cast
class MoneyCast implements CastsAttributes
{
    public function get($model, string $key, $value, array $attributes)
    {
        return $value ? new Money($value) : null;
    }

    public function set($model, string $key, $value, array $attributes)
    {
        return $value instanceof Money ? $value->getAmount() : $value;
    }
}

// Use in model
class Product extends Model
{
    protected $casts = [
        'price' => MoneyCast::class,
        'cost' => MoneyCast::class,
    ];
}
```

### 8.3 Attribute Accessors & Mutators (Laravel 9+ Syntax)

```php
class User extends Model
{
    // Accessor - modify data when retrieving
    protected function firstName(): Attribute
    {
        return Attribute::make(
            get: fn (string $value) => ucfirst($value),
            set: fn (string $value) => strtolower($value),
        );
    }

    // Computed attribute
    protected function fullName(): Attribute
    {
        return Attribute::make(
            get: fn () => "{$this->first_name} {$this->last_name}",
        );
    }

    // Complex transformation
    protected function avatar(): Attribute
    {
        return Attribute::make(
            get: function ($value) {
                if (!$value) {
                    return "https://ui-avatars.com/api/?name=" . urlencode($this->name);
                }
                return Storage::url($value);
            }
        );
    }
}
```

### 8.4 Legacy Accessor/Mutator Syntax (Still Valid)

```php
class User extends Model
{
    // Accessor
    public function getFullNameAttribute()
    {
        return "{$this->first_name} {$this->last_name}";
    }

    // Mutator
    public function setPasswordAttribute($value)
    {
        $this->attributes['password'] = Hash::make($value);
    }

    // Date mutator
    public function setBirthDateAttribute($value)
    {
        $this->attributes['birth_date'] = Carbon::parse($value)->format('Y-m-d');
    }
}
```

---

## 9. Model Events & Observers

### 9.1 Model Lifecycle Events

```php
class User extends Model
{
    protected static function boot()
    {
        parent::boot();

        // Before creating
        static::creating(function ($user) {
            $user->uuid = (string) Str::uuid();
        });

        // After creating
        static::created(function ($user) {
            // Send welcome email
            Mail::to($user)->send(new WelcomeEmail($user));
        });

        // Before updating
        static::updating(function ($user) {
            if ($user->isDirty('email')) {
                $user->email_verified_at = null;
            }
        });

        // Before deleting
        static::deleting(function ($user) {
            // Clean up related data
            $user->posts()->delete();
            $user->comments()->delete();
        });
    }
}
```

### 9.2 Model Observers (Cleaner Approach)

```php
// Create observer
php artisan make:observer UserObserver --model=User

// app/Observers/UserObserver.php
class UserObserver
{
    public function creating(User $user): void
    {
        $user->uuid = (string) Str::uuid();
    }

    public function created(User $user): void
    {
        // Dispatch job instead of sending email directly
        SendWelcomeEmail::dispatch($user);
    }

    public function updating(User $user): void
    {
        if ($user->isDirty('email')) {
            $user->email_verified_at = null;

            // Log email change for security
            activity()
                ->performedOn($user)
                ->log('email_changed');
        }
    }

    public function deleting(User $user): void
    {
        // Soft delete related data instead of hard delete
        $user->posts()->delete();
        $user->comments()->delete();
    }
}

// Register observer in AppServiceProvider
public function boot()
{
    User::observe(UserObserver::class);
}
```

### 9.3 Advanced Event Patterns

```php
// Conditional events
class Post extends Model
{
    protected static function boot()
    {
        parent::boot();

        static::saving(function ($post) {
            // Only auto-generate slug if not provided
            if (!$post->slug && $post->title) {
                $post->slug = Str::slug($post->title);
            }

            // Auto-set published_at for published posts
            if ($post->status === 'published' && !$post->published_at) {
                $post->published_at = now();
            }
        });
    }
}

// Event-driven architecture
class OrderObserver
{
    public function created(Order $order): void
    {
        // Dispatch multiple events for different services
        ProcessPayment::dispatch($order);
        UpdateInventory::dispatch($order);
        SendOrderConfirmation::dispatch($order);

        // Broadcast real-time update
        broadcast(new OrderCreated($order));
    }

    public function updated(Order $order): void
    {
        if ($order->wasChanged('status')) {
            match($order->status) {
                'confirmed' => ConfirmOrder::dispatch($order),
                'shipped' => ShipOrder::dispatch($order),
                'delivered' => CompleteOrder::dispatch($order),
                'cancelled' => CancelOrder::dispatch($order),
                default => null,
            };
        }
    }
}
```

---

## 10. Scopes

### 10.1 Local Scopes

```php
class User extends Model
{
    // Convention: scope + PascalCase method name
    public function scopeActive($query)
    {
        return $query->where('is_active', true);
    }

    public function scopeAdmins($query)
    {
        return $query->where('role', 'admin');
    }

    // Parameterized scopes
    public function scopeCreatedBetween($query, $startDate, $endDate)
    {
        return $query->whereBetween('created_at', [$startDate, $endDate]);
    }

    public function scopeWithRole($query, $role)
    {
        return $query->whereHas('roles', function ($q) use ($role) {
            $q->where('name', $role);
        });
    }
}

// Usage:
$activeUsers = User::active()->get();
$adminUsers = User::active()->admins()->get();
$recentUsers = User::createdBetween('2023-01-01', '2023-12-31')->get();
```

### 10.2 Global Scopes

```php
// Create global scope
class ActiveScope implements Scope
{
    public function apply(Builder $builder, Model $model)
    {
        $builder->where('is_active', true);
    }
}

// Apply to model
class User extends Model
{
    protected static function boot()
    {
        parent::boot();
        static::addGlobalScope(new ActiveScope);
    }
}

// Or use closure for simple scopes
class Post extends Model
{
    protected static function boot()
    {
        parent::boot();

        static::addGlobalScope('published', function (Builder $builder) {
            $builder->where('status', 'published');
        });
    }
}

// Remove global scope when needed
$allPosts = Post::withoutGlobalScope('published')->get();
$allUsers = User::withoutGlobalScopes()->get();
```

### 10.3 Advanced Scope Patterns

```php
class Post extends Model
{
    // Chainable scopes with relationships
    public function scopeWithAuthor($query)
    {
        return $query->with(['author' => function ($q) {
            $q->select('id', 'name', 'email');
        }]);
    }

    public function scopePopular($query, $threshold = 100)
    {
        return $query->where('views', '>=', $threshold)
                    ->orWhere('likes', '>=', $threshold / 2);
    }

    // Complex filtering scope
    public function scopeFilter($query, array $filters)
    {
        $query->when($filters['category'] ?? null, function ($q, $category) {
            $q->whereHas('categories', function ($q) use ($category) {
                $q->where('slug', $category);
            });
        });

        $query->when($filters['author'] ?? null, function ($q, $author) {
            $q->whereHas('author', function ($q) use ($author) {
                $q->where('username', $author);
            });
        });

        $query->when($filters['date_from'] ?? null, function ($q, $date) {
            $q->where('published_at', '>=', $date);
        });

        $query->when($filters['search'] ?? null, function ($q, $search) {
            $q->where('title', 'like', "%{$search}%")
              ->orWhere('content', 'like', "%{$search}%");
        });

        return $query;
    }
}

// Usage:
$posts = Post::filter([
    'category' => 'technology',
    'author' => 'john-doe',
    'date_from' => '2023-01-01',
    'search' => 'laravel'
])->popular()->withAuthor()->get();
```

---

## 11. Best Practices & Advanced Tips

### 11.1 Keep Models Focused

❌ **Fat Model (Bad Practice)**

```php
class User extends Model
{
    // DON'T put all logic in the model
    public function processPayment($amount, $paymentMethod)
    {
        // 50+ lines of payment processing logic
        $stripe = new StripeGateway();
        $paypal = new PayPalGateway();
        // Complex payment logic...
    }

    public function sendNotification($type, $data)
    {
        // 30+ lines of notification logic
        // Email, SMS, push notification logic...
    }

    public function generateReport($type)
    {
        // 100+ lines of report generation
    }
}
```

✅ **Thin Model (Good Practice)**

```php
class User extends Model
{
    protected $fillable = ['name', 'email', 'password'];

    protected $casts = [
        'email_verified_at' => 'datetime',
        'settings' => 'array'
    ];

    // Only model-specific logic
    public function isActive(): bool
    {
        return $this->is_active && $this->email_verified_at;
    }

    public function getAvatarUrlAttribute(): string
    {
        return $this->avatar
            ? Storage::url($this->avatar)
            : "https://ui-avatars.com/api/?name=" . urlencode($this->name);
    }

    // Relationships
    public function posts()
    {
        return $this->hasMany(Post::class);
    }

    public function roles()
    {
        return $this->belongsToMany(Role::class);
    }
}

// Move complex logic to services
class PaymentService
{
    public function processPayment(User $user, $amount, $paymentMethod)
    {
        // Payment processing logic
    }
}

class NotificationService
{
    public function sendNotification(User $user, $type, $data)
    {
        // Notification logic
    }
}
```

### 11.2 Strategic Model Organization

```php
// Large applications: organize by domain
app/
├── Models/
│   ├── User/
│   │   ├── User.php
│   │   ├── Profile.php
│   │   ├── Permission.php
│   │   └── Role.php
│   ├── Blog/
│   │   ├── Post.php
│   │   ├── Category.php
│   │   ├── Tag.php
│   │   └── Comment.php
│   ├── Ecommerce/
│   │   ├── Product.php
│   │   ├── Order.php
│   │   ├── Cart.php
│   │   └── Payment.php
│   └── Shared/
│       ├── Address.php
│       ├── Media.php
│       └── Setting.php
```

### 11.3 Performance Considerations

```php
// ❌ N+1 Query Problem
$users = User::all();
foreach ($users as $user) {
    echo $user->posts->count(); // New query for each user
}

// ✅ Eager Loading
$users = User::withCount('posts')->get();
foreach ($users as $user) {
    echo $user->posts_count; // No additional queries
}

// ✅ Strategic Eager Loading
class Post extends Model
{
    protected $with = ['author']; // Always load author

    // Or use selective loading
    public function scopeWithFullData($query)
    {
        return $query->with([
            'author:id,name,email',
            'categories:id,name,slug',
            'tags:id,name'
        ])->withCount(['comments', 'likes']);
    }
}

// ✅ Chunking for large datasets
User::chunk(1000, function ($users) {
    foreach ($users as $user) {
        // Process user
    }
});
```

### 11.4 Advanced Model Patterns

```php
// Repository Pattern for complex queries
interface UserRepositoryInterface
{
    public function findActiveUsersWithPosts(): Collection;
    public function findUsersByRole(string $role): Collection;
}

class EloquentUserRepository implements UserRepositoryInterface
{
    public function findActiveUsersWithPosts(): Collection
    {
        return User::active()
                   ->has('posts')
                   ->with(['posts' => function ($query) {
                       $query->published()->latest();
                   }])
                   ->get();
    }
}

// Factory Pattern for model creation
class UserFactory
{
    public static function createAdmin(array $data): User
    {
        $user = User::create($data);
        $user->roles()->attach(Role::where('name', 'admin')->first());
        return $user;
    }

    public static function createCustomer(array $data): User
    {
        $user = User::create($data);
        $user->roles()->attach(Role::where('name', 'customer')->first());
        $user->profile()->create(['type' => 'customer']);
        return $user;
    }
}
```

---

## 12. Common Mistakes

### 12.1 Naming Mistakes

```php
❌ Common Naming Errors:
class Users extends Model {} // Should be User (singular)
class blog_post extends Model {} // Should be BlogPost (PascalCase)
class product_items extends Model {} // Should be ProductItem

// Database table issues:
User model → user table (should be users)
BlogPost model → blogpost table (should be blog_posts)
```

### 12.2 Mass Assignment Vulnerabilities

```php
❌ Dangerous Mass Assignment:
class User extends Model
{
    protected $guarded = []; // NEVER do this!
}

// Allows: User::create(['is_admin' => true, 'email_verified_at' => now()])

✅ Secure Mass Assignment:
class User extends Model
{
    protected $fillable = ['name', 'email', 'password'];

    // Or use guarded selectively
    protected $guarded = ['id', 'is_admin', 'email_verified_at', 'remember_token'];
}
```

### 12.3 Relationship Mistakes

```php
❌ Incorrect Foreign Key Assumptions:
class Post extends Model
{
    public function author()
    {
        // Assumes author_id exists, but column is user_id
        return $this->belongsTo(User::class);
    }
}

✅ Explicit Foreign Key:
class Post extends Model
{
    public function author()
    {
        return $this->belongsTo(User::class, 'user_id');
    }
}

❌ Missing Indexes:
// Migration without foreign key index
Schema::create('posts', function (Blueprint $table) {
    $table->id();
    $table->unsignedBigInteger('user_id'); // Missing index!
    $table->string('title');
});

✅ Proper Indexing:
Schema::create('posts', function (Blueprint $table) {
    $table->id();
    $table->foreignId('user_id')->constrained(); // Automatic index
    // Or manually:
    // $table->unsignedBigInteger('user_id')->index();
    $table->string('title');
});
```

### 12.4 Performance Mistakes

```php
❌ Not Using Database-Level Operations:
// Inefficient: load all records into memory
$posts = Post::all();
foreach ($posts as $post) {
    $post->increment('views');
}

✅ Database-Level Operations:
// Efficient: single query
Post::query()->increment('views');

// Or for specific conditions:
Post::where('featured', true)->increment('views');
```

---

## 13. Laravel 10/11 Updates

### 13.1 New Attribute Features (Laravel 10+)

```php
// Enhanced attribute casting
class User extends Model
{
    protected function avatar(): Attribute
    {
        return Attribute::make(
            get: fn (?string $value) => $value
                ? Storage::disk('s3')->url($value)
                : $this->generateGravatar(),
            set: fn (UploadedFile $file) => $this->storeAvatar($file),
        );
    }

    // Encrypted attributes (Laravel 10)
    protected $casts = [
        'social_security' => 'encrypted',
        'credit_card' => 'encrypted:array',
    ];
}
```

### 13.2 Improved Query Builder (Laravel 11)

```php
// New whereLike method
User::whereLike('name', 'john')->get();

// Enhanced when() methods
$query = User::query()
    ->when($request->filled('name'),
        fn ($q) => $q->whereLike('name', $request->name))
    ->when($request->filled('active'),
        fn ($q) => $q->where('is_active', $request->boolean('active')));
```

### 13.3 Model Factories Improvements

```php
// Enhanced factory methods (Laravel 11)
class UserFactory extends Factory
{
    public function configure()
    {
        return $this->afterCreating(function (User $user) {
            $user->profile()->create([
                'bio' => $this->faker->paragraph,
            ]);
        });
    }

    // Conditional states
    public function admin()
    {
        return $this->state(fn () => ['role' => 'admin'])
                    ->afterCreating(fn (User $user) =>
                        $user->permissions()->attach(Permission::all()));
    }
}
```

---

## 14. Model Conventions Checklist

### ✅ Naming & Structure Do's

- Use singular PascalCase for model names (`User`, `BlogPost`)
- Follow snake_case pluralization for table names (`users`, `blog_posts`)
- Organize models by domain in large applications
- Use descriptive names that reflect business concepts
- Follow Laravel's foreign key conventions (`user_id`, `blog_post_id`)
- Name pivot tables alphabetically (`role_user`, not `user_role`)
- Use meaningful relationship method names

### ❌ Naming & Structure Don'ts

- Don't use plural model names (`Users` ❌)
- Don't use snake_case for model classes (`blog_post` ❌)
- Don't use generic names (`Data`, `Info`, `Item` ❌)
- Don't ignore foreign key naming conventions
- Don't create overly nested directory structures
- Don't mix naming conventions within the same project

### ✅ Security & Data Do's

- Always use `$fillable` for mass assignment protection
- Validate data with Form Requests, not in models
- Use appropriate casts for data types
- Implement proper authorization with policies
- Use observers for complex model events
- Index foreign keys and frequently queried columns
- Use database-level constraints where appropriate

### ❌ Security & Data Don'ts

- Never use `$guarded = []` in production
- Don't put validation logic in models
- Don't ignore mass assignment protection
- Don't store sensitive data without encryption
- Don't bypass Eloquent for data integrity operations
- Don't forget to index relationship columns
- Don't mix business logic with data persistence

### ✅ Performance Do's

- Use eager loading to prevent N+1 queries
- Implement appropriate database indexes
- Use chunking for large dataset operations
- Cache expensive computed attributes
- Use database-level operations for bulk updates
- Implement query scopes for reusable logic
- Consider using readonly replicas for reports

### ❌ Performance Don'ts

- Don't load unnecessary relationships
- Don't perform complex calculations in accessors without caching
- Don't use loops where database operations could be used
- Don't ignore query optimization
- Don't forget to paginate large result sets
- Don't mix presentation logic with model logic

### ✅ Architecture Do's

- Keep models thin and focused
- Move complex business logic to services
- Use repositories for complex queries
- Implement model events and observers appropriately
- Follow single responsibility principle
- Use interfaces for dependency injection
- Implement proper error handling

### ❌ Architecture Don'ts

- Don't create fat models with excessive logic
- Don't mix different concerns in the same model
- Don't bypass the repository pattern for complex queries
- Don't ignore separation of concerns
- Don't put presentation logic in models
- Don't create tight coupling between models and external services

---

**Remember**: Laravel's conventions exist to make your code predictable and maintainable. Master these conventions first, then learn when and how to break them strategically. The goal is not just working code, but code that scales with your team and application over time.

The journey from mid-level to senior developer is about understanding not just the "how" but the "why" behind these conventions, and knowing when to apply patterns that serve your specific business needs while maintaining Laravel's elegant simplicity.
