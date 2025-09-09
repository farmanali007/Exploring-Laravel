# Laravel Query Scopes: Complete Developer Guide

## Table of Contents

1. [Introduction to Query Scopes](#introduction-to-query-scopes)
2. [Local Scopes](#local-scopes)
3. [Global Scopes](#global-scopes)
4. [Advanced Usage Patterns](#advanced-usage-patterns)
5. [Real-World Use Cases](#real-world-use-cases)
6. [When NOT to Use Scopes](#when-not-to-use-scopes)
7. [Senior Developer Best Practices](#senior-developer-best-practices)
8. [Testing Strategies](#testing-strategies)
9. [Summary & Key Takeaways](#summary--key-takeaways)

---

## Introduction to Query Scopes

### What Are Query Scopes?

Query scopes are Laravel's mechanism for extracting common query constraints into reusable, chainable methods on Eloquent models. They encapsulate frequently used WHERE clauses, JOINs, and other query modifications, promoting code reusability and maintaining clean, expressive query syntax.

**Two Types of Scopes:**

**Local Scopes** - Explicitly called methods that modify queries when invoked:

- Defined as `scope{Name}` methods on models
- Called explicitly in queries: `User::active()->get()`
- Optional application - only run when specifically called

**Global Scopes** - Automatically applied constraints that affect all queries for a model:

- Implemented as classes or closures
- Applied automatically to every query
- Can be removed when needed: `User::withoutGlobalScope(ActiveScope::class)->get()`

### Why Use Query Scopes?

Query scopes solve several common problems in application development:

**Code Reusability**: Eliminate repetitive WHERE clauses across controllers and services
**Expressiveness**: Create domain-specific query language that reads like business requirements
**Maintainability**: Centralize query logic in models where it belongs
**Consistency**: Ensure the same filtering logic is applied uniformly across the application
**Testability**: Isolate query constraints for focused unit testing

---

## Local Scopes

### Defining Local Scopes

Local scopes are methods prefixed with `scope` that accept a query builder instance and optional parameters.

**Problem**: Repeatedly filtering active users across multiple controllers and services.

**Before** (repetitive query constraints):

```php
<?php

// UserController.php
class UserController extends Controller
{
    public function index()
    {
        // Repeated constraint in multiple places
        $users = User::where('is_active', true)
                    ->where('email_verified_at', '!=', null)
                    ->get();

        return view('users.index', compact('users'));
    }
}

// DashboardController.php
class DashboardController extends Controller
{
    public function stats()
    {
        // Same constraint repeated
        $activeUserCount = User::where('is_active', true)
                              ->where('email_verified_at', '!=', null)
                              ->count();

        return view('dashboard', compact('activeUserCount'));
    }
}

// ReportService.php
class ReportService
{
    public function generateUserReport()
    {
        // Yet another repetition
        return User::where('is_active', true)
                  ->where('email_verified_at', '!=', null)
                  ->with('profile')
                  ->get();
    }
}
```

**After** (with local scope):

```php
<?php

// User.php
class User extends Model
{
    // Define the local scope
    public function scopeActive($query)
    {
        return $query->where('is_active', true)
                    ->where('email_verified_at', '!=', null);
    }

    // Additional scopes for common filters
    public function scopeVerified($query)
    {
        return $query->whereNotNull('email_verified_at');
    }

    public function scopeRecent($query)
    {
        return $query->where('created_at', '>=', now()->subDays(30));
    }
}

// UserController.php
class UserController extends Controller
{
    public function index()
    {
        // Clean, expressive query
        $users = User::active()->get();

        return view('users.index', compact('users'));
    }
}

// DashboardController.php
class DashboardController extends Controller
{
    public function stats()
    {
        // Consistent logic, readable code
        $activeUserCount = User::active()->count();

        return view('dashboard', compact('activeUserCount'));
    }
}

// ReportService.php
class ReportService
{
    public function generateUserReport()
    {
        // Chainable with other query methods
        return User::active()
                  ->with('profile')
                  ->get();
    }
}
```

**Key Improvements:**

- **DRY Principle**: Query logic defined once, used everywhere
- **Readability**: `User::active()` is self-documenting and business-focused
- **Maintainability**: Changing "active" criteria requires updating only the scope method
- **Chainability**: Scopes work seamlessly with other Eloquent methods
- **Consistency**: Ensures identical filtering logic across the application

### Parameterized Local Scopes

**Problem**: Need flexible filtering with dynamic parameters for different scenarios.

**Before** (rigid, non-reusable constraints):

```php
<?php

// Multiple specific methods for different time ranges
class PostController extends Controller
{
    public function recent()
    {
        return Post::where('created_at', '>=', now()->subDays(7))->get();
    }

    public function lastMonth()
    {
        return Post::where('created_at', '>=', now()->subDays(30))->get();
    }

    public function lastQuarter()
    {
        return Post::where('created_at', '>=', now()->subDays(90))->get();
    }
}
```

**After** (flexible parameterized scope):

```php
<?php

// Post.php
class Post extends Model
{
    // Parameterized scope for flexible date filtering
    public function scopeCreatedWithinDays($query, int $days)
    {
        return $query->where('created_at', '>=', now()->subDays($days));
    }

    // Scope with multiple parameters
    public function scopeByStatus($query, string $status, bool $published = true)
    {
        $query = $query->where('status', $status);

        if ($published) {
            $query->where('published_at', '<=', now());
        }

        return $query;
    }

    // Scope with array parameter for flexible filtering
    public function scopeInCategories($query, array $categoryIds)
    {
        return $query->whereIn('category_id', $categoryIds);
    }

    // Scope with optional parameters and defaults
    public function scopePopular($query, int $minViews = 1000, int $minLikes = 50)
    {
        return $query->where('view_count', '>=', $minViews)
                    ->where('likes_count', '>=', $minLikes);
    }
}

// PostController.php
class PostController extends Controller
{
    public function recent()
    {
        return Post::createdWithinDays(7)->get();
    }

    public function lastMonth()
    {
        return Post::createdWithinDays(30)->get();
    }

    public function lastQuarter()
    {
        return Post::createdWithinDays(90)->get();
    }

    public function popularInCategories(Request $request)
    {
        $categoryIds = $request->input('categories', []);
        $minViews = $request->input('min_views', 1000);

        return Post::popular($minViews)
                  ->inCategories($categoryIds)
                  ->byStatus('published')
                  ->get();
    }
}
```

**Advanced Features:**

- **Parameter Flexibility**: Single scope handles multiple use cases
- **Default Values**: Provide sensible defaults while allowing customization
- **Type Hints**: Ensure parameter type safety and better IDE support
- **Chainability**: Multiple parameterized scopes work together seamlessly
- **Business Logic Encapsulation**: Complex filtering rules contained in descriptive methods

---

## Global Scopes

### Understanding Global Scopes

Global scopes automatically apply constraints to every query for a model. They're invisible to calling code but consistently enforce business rules across the application.

**When to Use Global Scopes:**

- Multi-tenant applications (filter by tenant automatically)
- Soft deletes (Laravel's built-in example)
- Published/active status that should always be enforced
- Company-wide data access rules

### Implementing Global Scopes

**Problem**: Multi-tenant application where users should only see data belonging to their tenant, but developers keep forgetting to add tenant filtering.

**Before** (manual tenant filtering everywhere):

```php
<?php

// Inconsistent tenant filtering across controllers
class PostController extends Controller
{
    public function index()
    {
        // Easy to forget tenant filtering
        $posts = Post::where('tenant_id', auth()->user()->tenant_id)->get();

        return view('posts.index', compact('posts'));
    }

    public function show(Post $post)
    {
        // Security vulnerability - no tenant check!
        return view('posts.show', compact('post'));
    }
}

class ReportController extends Controller
{
    public function sales()
    {
        // Another place where tenant filtering might be forgotten
        $sales = Sale::where('tenant_id', auth()->user()->tenant_id)
                    ->sum('amount');

        return view('reports.sales', compact('sales'));
    }
}
```

**After** (automatic tenant filtering with global scope):

```php
<?php

// TenantScope.php
use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Scope;

class TenantScope implements Scope
{
    public function apply(Builder $builder, Model $model): void
    {
        // Only apply if user is authenticated and has a tenant
        if (auth()->check() && auth()->user()->tenant_id) {
            $builder->where($model->getTable() . '.tenant_id', auth()->user()->tenant_id);
        }
    }

    public function extend(Builder $builder): void
    {
        // Add methods to builder for scope manipulation
        $this->addWithoutTenant($builder);
        $this->addOnlyTenant($builder);
    }

    protected function addWithoutTenant(Builder $builder): void
    {
        $builder->macro('withoutTenant', function (Builder $builder) {
            return $builder->withoutGlobalScope($this);
        });
    }

    protected function addOnlyTenant(Builder $builder): void
    {
        $builder->macro('onlyTenant', function (Builder $builder, int $tenantId) {
            return $builder->withoutGlobalScope($this)
                          ->where('tenant_id', $tenantId);
        });
    }
}

// HasTenant trait for reusable global scope application
trait HasTenant
{
    protected static function bootHasTenant(): void
    {
        static::addGlobalScope(new TenantScope);
    }
}

// Post.php
class Post extends Model
{
    use HasTenant;

    protected $fillable = ['title', 'content', 'tenant_id'];
}

// Sale.php
class Sale extends Model
{
    use HasTenant;

    protected $fillable = ['amount', 'tenant_id', 'customer_id'];
}

// PostController.php
class PostController extends Controller
{
    public function index()
    {
        // Tenant filtering applied automatically - no security risk
        $posts = Post::all();

        return view('posts.index', compact('posts'));
    }

    public function show(Post $post)
    {
        // Route model binding respects global scope - secure by default
        return view('posts.show', compact('post'));
    }

    public function adminIndex()
    {
        // Admins can see posts from all tenants when needed
        $allPosts = Post::withoutTenant()->get();

        return view('admin.posts.index', compact('allPosts'));
    }
}
```

**Security and Consistency Benefits:**

- **Automatic Protection**: Tenant filtering applied to every query without manual intervention
- **Route Model Binding**: Laravel's route model binding respects global scopes, providing automatic security
- **Impossible to Forget**: Developers cannot accidentally expose cross-tenant data
- **Flexible Override**: Admin functions can still access all data when explicitly needed
- **Consistent Behavior**: Same filtering logic across all models using the trait

### Removing Global Scopes

Sometimes you need to bypass global scopes for specific operations:

```php
<?php

class AdminController extends Controller
{
    public function globalStats()
    {
        // Remove specific global scope
        $allUsers = User::withoutGlobalScope(TenantScope::class)->count();

        // Remove all global scopes
        $totalPosts = Post::withoutGlobalScopes()->count();

        // Remove multiple specific scopes
        $adminData = Post::withoutGlobalScopes([
            TenantScope::class,
            PublishedScope::class
        ])->get();

        return view('admin.stats', compact('allUsers', 'totalPosts', 'adminData'));
    }

    public function migrateData()
    {
        // For data migration, work with all records regardless of scopes
        Post::withoutGlobalScopes()
            ->chunk(100, function ($posts) {
                foreach ($posts as $post) {
                    // Process posts from all tenants
                    $this->processPost($post);
                }
            });
    }
}
```

### Built-in Global Scope Example: Soft Deletes

Laravel's soft delete functionality is implemented using a global scope:

```php
<?php

// Laravel's SoftDeletes trait adds a global scope automatically
class User extends Model
{
    use SoftDeletes;

    protected $dates = ['deleted_at'];
}

// Usage examples showing how the global scope works
class UserService
{
    public function getActiveUsers()
    {
        // Only non-deleted users (global scope applied automatically)
        return User::all();
    }

    public function getAllUsersIncludingDeleted()
    {
        // Include soft-deleted records
        return User::withTrashed()->get();
    }

    public function getOnlyDeletedUsers()
    {
        // Only soft-deleted records
        return User::onlyTrashed()->get();
    }

    public function permanentlyDeleteOldUsers()
    {
        // Force delete old soft-deleted records
        User::onlyTrashed()
            ->where('deleted_at', '<', now()->subYear())
            ->forceDelete();
    }
}
```

---

## Advanced Usage Patterns

### Combining Multiple Scopes

Scopes are designed to be chainable and composable for complex filtering scenarios:

```php
<?php

// User.php
class User extends Model
{
    public function scopeActive($query)
    {
        return $query->where('is_active', true);
    }

    public function scopeVerified($query)
    {
        return $query->whereNotNull('email_verified_at');
    }

    public function scopeSubscribed($query)
    {
        return $query->where('subscription_status', 'active');
    }

    public function scopeInRole($query, string $role)
    {
        return $query->whereHas('roles', function ($q) use ($role) {
            $q->where('name', $role);
        });
    }

    public function scopeJoinedAfter($query, Carbon $date)
    {
        return $query->where('created_at', '>', $date);
    }
}

// Complex queries using multiple scopes
class UserAnalyticsService
{
    public function getRecentActiveSubscribers(): Collection
    {
        return User::active()
                  ->verified()
                  ->subscribed()
                  ->joinedAfter(now()->subMonths(3))
                  ->with(['subscription', 'profile'])
                  ->get();
    }

    public function getAdminUsers(): Collection
    {
        return User::active()
                  ->verified()
                  ->inRole('admin')
                  ->orderBy('last_login_at', 'desc')
                  ->get();
    }

    public function getUserSegmentation(): array
    {
        $baseQuery = User::verified();

        return [
            'active_subscribers' => $baseQuery->clone()->active()->subscribed()->count(),
            'inactive_subscribers' => $baseQuery->clone()->where('is_active', false)->subscribed()->count(),
            'active_free_users' => $baseQuery->clone()->active()->where('subscription_status', '!=', 'active')->count(),
            'recent_signups' => $baseQuery->clone()->joinedAfter(now()->subDays(7))->count(),
        ];
    }
}
```

### Dynamic Scopes with Closures

For ad-hoc filtering that doesn't warrant a dedicated scope method:

```php
<?php

class ProductService
{
    public function getProductsByComplexCriteria(array $criteria): Collection
    {
        $query = Product::query();

        // Apply dynamic scope based on criteria
        if (isset($criteria['price_range'])) {
            $query->where(function ($q) use ($criteria) {
                $q->where('price', '>=', $criteria['price_range']['min'])
                  ->where('price', '<=', $criteria['price_range']['max']);
            });
        }

        // Conditional scope application
        if (isset($criteria['categories'])) {
            $query->whereHas('categories', function ($q) use ($criteria) {
                $q->whereIn('id', $criteria['categories']);
            });
        }

        // Complex rating filter
        if (isset($criteria['min_rating'])) {
            $query->where(function ($q) use ($criteria) {
                $q->where('average_rating', '>=', $criteria['min_rating'])
                  ->orWhere(function ($subQ) use ($criteria) {
                      $subQ->whereNull('average_rating')
                           ->where('reviews_count', 0);
                  });
            });
        }

        return $query->with(['categories', 'reviews'])
                    ->orderBy('created_at', 'desc')
                    ->get();
    }
}
```

### Performance Considerations

Understanding the performance implications of scopes:

```php
<?php

class PerformanceAwareScopes
{
    // GOOD: Efficient scope using proper indexing
    public function scopePublished($query)
    {
        // Assumes index on (published_at, status)
        return $query->where('published_at', '<=', now())
                    ->where('status', 'published');
    }

    // CAREFUL: Scope that might cause N+1 problems
    public function scopeWithActiveComments($query)
    {
        // This could be inefficient without proper eager loading
        return $query->whereHas('comments', function ($q) {
            $q->where('is_active', true);
        });
    }

    // BETTER: Provide efficient alternative
    public function scopeWithActiveCommentsOptimized($query)
    {
        return $query->withCount(['comments' => function ($q) {
            $q->where('is_active', true);
        }])->having('comments_count', '>', 0);
    }

    // GOOD: Scope that enables efficient eager loading
    public function scopeWithRecentActivity($query)
    {
        return $query->with(['latestPost', 'recentComments' => function ($q) {
            $q->where('created_at', '>=', now()->subDays(7))
              ->limit(5);
        }]);
    }
}

// Usage with performance monitoring
class OptimizedController extends Controller
{
    public function efficientListing()
    {
        // Monitor query count and execution time
        \DB::enableQueryLog();

        $posts = Post::published()
                    ->withActiveCommentsOptimized()
                    ->withRecentActivity()
                    ->paginate(20);

        // Log query performance in development
        if (app()->environment('local')) {
            \Log::info('Query count: ' . count(\DB::getQueryLog()));
        }

        return view('posts.index', compact('posts'));
    }
}
```

---

## Real-World Use Cases

### Multi-Tenant Application Architecture

**Problem**: Building a SaaS application where each customer's data must be completely isolated.

```php
<?php

// TenantAwareScope.php - Comprehensive tenant isolation
class TenantAwareScope implements Scope
{
    public function apply(Builder $builder, Model $model): void
    {
        if ($this->shouldApplyTenantScope()) {
            $tenantId = $this->getCurrentTenantId();
            $builder->where($model->getTable() . '.tenant_id', $tenantId);
        }
    }

    protected function shouldApplyTenantScope(): bool
    {
        // Don't apply in console commands or when explicitly disabled
        return !app()->runningInConsole()
               && !app()->has('tenant.scope.disabled')
               && auth()->check()
               && auth()->user()->tenant_id;
    }

    protected function getCurrentTenantId(): int
    {
        // Multiple ways to determine tenant - flexible resolution
        if (request()->has('tenant_override') && auth()->user()->hasRole('super_admin')) {
            return request()->input('tenant_override');
        }

        return auth()->user()->tenant_id;
    }

    public function extend(Builder $builder): void
    {
        $this->addWithoutTenant($builder);
        $this->addForTenant($builder);
        $this->addCrossTenant($builder);
    }

    protected function addWithoutTenant(Builder $builder): void
    {
        $builder->macro('withoutTenant', function (Builder $builder) {
            return $builder->withoutGlobalScope($this);
        });
    }

    protected function addForTenant(Builder $builder): void
    {
        $builder->macro('forTenant', function (Builder $builder, int $tenantId) {
            return $builder->withoutGlobalScope($this)
                          ->where('tenant_id', $tenantId);
        });
    }

    protected function addCrossTenant(Builder $builder): void
    {
        $builder->macro('crossTenant', function (Builder $builder, array $tenantIds) {
            return $builder->withoutGlobalScope($this)
                          ->whereIn('tenant_id', $tenantIds);
        });
    }
}

// BaseTenantModel.php - Base class for tenant-aware models
abstract class BaseTenantModel extends Model
{
    protected static function bootBaseTenantModel(): void
    {
        static::addGlobalScope(new TenantAwareScope);

        // Automatically set tenant_id when creating records
        static::creating(function (Model $model) {
            if (auth()->check() && !$model->tenant_id) {
                $model->tenant_id = auth()->user()->tenant_id;
            }
        });
    }

    // Relationship that respects tenant boundaries
    public function tenant()
    {
        return $this->belongsTo(Tenant::class);
    }
}

// Concrete models using tenant awareness
class Order extends BaseTenantModel
{
    protected $fillable = ['customer_id', 'total', 'status', 'tenant_id'];

    public function scopePending($query)
    {
        return $query->where('status', 'pending');
    }

    public function scopeCompleted($query)
    {
        return $query->where('status', 'completed');
    }

    public function customer()
    {
        return $this->belongsTo(Customer::class);
    }
}

class Customer extends BaseTenantModel
{
    protected $fillable = ['name', 'email', 'tenant_id'];

    public function scopeActive($query)
    {
        return $query->where('is_active', true);
    }

    public function orders()
    {
        return $this->hasMany(Order::class);
    }
}

// Usage in controllers - tenant isolation is automatic
class OrderController extends Controller
{
    public function index()
    {
        // Only shows orders for current user's tenant
        $orders = Order::with('customer')
                      ->pending()
                      ->latest()
                      ->paginate(20);

        return view('orders.index', compact('orders'));
    }

    public function store(Request $request)
    {
        // tenant_id set automatically via model boot method
        $order = Order::create($request->validated());

        return redirect()->route('orders.show', $order);
    }
}

// Admin functionality that works across tenants
class AdminOrderController extends Controller
{
    public function globalStats()
    {
        $this->authorize('view-global-stats');

        $stats = [
            'total_orders' => Order::withoutTenant()->count(),
            'pending_orders' => Order::withoutTenant()->pending()->count(),
            'completed_orders' => Order::withoutTenant()->completed()->count(),
        ];

        return view('admin.orders.stats', compact('stats'));
    }

    public function tenantReport(Tenant $tenant)
    {
        $this->authorize('view-tenant-data', $tenant);

        $orders = Order::forTenant($tenant->id)
                      ->with('customer')
                      ->get();

        return view('admin.orders.tenant-report', compact('orders', 'tenant'));
    }
}
```

### Content Management with Publishing States

**Problem**: Blog/CMS where content has multiple states (draft, published, archived) and published content should be shown by default to public users.

```php
<?php

// PublishedScope.php - Automatically filter to published content
class PublishedScope implements Scope
{
    public function apply(Builder $builder, Model $model): void
    {
        // Only apply to non-admin users
        if (!$this->isAdminContext()) {
            $builder->where($model->getTable() . '.status', 'published')
                   ->where($model->getTable() . '.published_at', '<=', now());
        }
    }

    protected function isAdminContext(): bool
    {
        return auth()->check()
               && (auth()->user()->hasRole(['admin', 'editor'])
                   || request()->is('admin/*'));
    }

    public function extend(Builder $builder): void
    {
        $builder->macro('withDrafts', function (Builder $builder) {
            return $builder->withoutGlobalScope($this);
        });

        $builder->macro('onlyDrafts', function (Builder $builder) {
            return $builder->withoutGlobalScope($this)
                          ->where('status', 'draft');
        });

        $builder->macro('onlyPublished', function (Builder $builder) {
            return $builder->withoutGlobalScope($this)
                          ->where('status', 'published')
                          ->where('published_at', '<=', now());
        });
    }
}

// Post.php
class Post extends Model
{
    protected $fillable = ['title', 'content', 'status', 'published_at'];
    protected $casts = ['published_at' => 'datetime'];

    protected static function booted(): void
    {
        static::addGlobalScope(new PublishedScope);
    }

    // Local scopes for additional filtering
    public function scopeFeatured($query)
    {
        return $query->where('is_featured', true);
    }

    public function scopeInCategory($query, string $categorySlug)
    {
        return $query->whereHas('category', function ($q) use ($categorySlug) {
            $q->where('slug', $categorySlug);
        });
    }

    public function scopeByAuthor($query, User $author)
    {
        return $query->where('author_id', $author->id);
    }

    public function scopeScheduled($query)
    {
        return $query->where('status', 'published')
                    ->where('published_at', '>', now());
    }

    public function scopeRecent($query, int $days = 30)
    {
        return $query->where('published_at', '>=', now()->subDays($days));
    }

    // Relationships
    public function author()
    {
        return $this->belongsTo(User::class, 'author_id');
    }

    public function category()
    {
        return $this->belongsTo(Category::class);
    }
}

// Public-facing controllers - only published content shown
class BlogController extends Controller
{
    public function index()
    {
        // Global scope ensures only published posts are shown
        $posts = Post::with(['author', 'category'])
                    ->featured()
                    ->latest('published_at')
                    ->paginate(10);

        return view('blog.index', compact('posts'));
    }

    public function category(Category $category)
    {
        $posts = Post::inCategory($category->slug)
                    ->with('author')
                    ->latest('published_at')
                    ->paginate(15);

        return view('blog.category', compact('posts', 'category'));
    }

    public function author(User $author)
    {
        $posts = Post::byAuthor($author)
                    ->with('category')
                    ->latest('published_at')
                    ->paginate(10);

        return view('blog.author', compact('posts', 'author'));
    }
}

// Admin controllers - can see all content states
class AdminPostController extends Controller
{
    public function index(Request $request)
    {
        $query = Post::withDrafts()->with(['author', 'category']);

        // Apply filters based on request parameters
        if ($request->filled('status')) {
            match($request->status) {
                'published' => $query->onlyPublished(),
                'draft' => $query->onlyDrafts(),
                'scheduled' => $query->scheduled(),
                default => $query
            };
        }

        if ($request->filled('author')) {
            $query->byAuthor(User::find($request->author));
        }

        $posts = $query->latest('updated_at')->paginate(20);

        return view('admin.posts.index', compact('posts'));
    }

    public function create()
    {
        return view('admin.posts.create');
    }

    public function store(Request $request)
    {
        $validated = $request->validate([
            'title' => 'required|max:255',
            'content' => 'required',
            'status' => 'required|in:draft,published',
            'published_at' => 'nullable|date',
            'category_id' => 'required|exists:categories,id',
        ]);

        $validated['author_id'] = auth()->id();

        // Set published_at to now if status is published and no date specified
        if ($validated['status'] === 'published' && !$validated['published_at']) {
            $validated['published_at'] = now();
        }

        $post = Post::create($validated);

        return redirect()->route('admin.posts.show', $post);
    }
}
```

### Role-Based Data Access

**Problem**: Users should only see data relevant to their role and permissions within an organization.

```php
<?php

// RoleBasedAccessScope.php
class RoleBasedAccessScope implements Scope
{
    public function apply(Builder $builder, Model $model): void
    {
        if (!auth()->check()) {
            return;
        }

        $user = auth()->user();

        // Super admins see everything
        if ($user->hasRole('super_admin')) {
            return;
        }

        // Apply role-specific constraints
        match(true) {
            $user->hasRole('manager') => $this->applyManagerConstraints($builder, $model, $user),
            $user->hasRole('team_lead') => $this->applyTeamLeadConstraints($builder, $model, $user),
            $user->hasRole('employee') => $this->applyEmployeeConstraints($builder, $model, $user),
            default => $this->applyGuestConstraints($builder, $model)
        };
    }

    protected function applyManagerConstraints(Builder $builder, Model $model, User $user): void
    {
        // Managers see all data in their department
        $builder->whereHas('department', function ($q) use ($user) {
            $q->whereIn('id', $user->managedDepartments->pluck('id'));
        });
    }

    protected function applyTeamLeadConstraints(Builder $builder, Model $model, User $user): void
    {
        // Team leads see their team's data
        $builder->whereHas('assignedUsers', function ($q) use ($user) {
            $q->whereIn('user_id', $user->teamMembers->pluck('id'));
        });
    }

    protected function applyEmployeeConstraints(Builder $builder, Model $model, User $user): void
    {
        // Employees only see their own data
        $builder->where('user_id', $user->id);
    }

    protected function applyGuestConstraints(Builder $builder, Model $model): void
    {
        // Guests see nothing
        $builder->whereRaw('1 = 0');
    }
}

// Project.php
class Project extends Model
{
    protected static function booted(): void
    {
        static::addGlobalScope(new RoleBasedAccessScope);
    }

    public function scopeActive($query)
    {
        return $query->where('status', 'active');
    }

    public function scopeCompleted($query)
    {
        return $query->where('status', 'completed');
    }

    public function scopeOverdue($query)
    {
        return $query->where('due_date', '<', now())
                    ->where('status', '!=', 'completed');
    }

    public function department()
    {
        return $this->belongsTo(Department::class);
    }

    public function assignedUsers()
    {
        return $this->belongsToMany(User::class, 'project_user');
    }
}

// ProjectController.php
class ProjectController extends Controller
{
    public function index()
    {
        // Users automatically see only projects they have access to
        $projects = Project::active()
                          ->with(['department', 'assignedUsers'])
                          ->latest()
                          ->paginate(15);

        return view('projects.index', compact('projects'));
    }

    public function myProjects()
    {
        // Even more restrictive - only projects user is assigned to
        $projects = Project::whereHas('assignedUsers', function ($q) {
            $q->where('user_id', auth()->id());
        })->active()
          ->with('department')
          ->get();

        return view('projects.my-projects', compact('projects'));
    }
}
```

---

## When NOT to Use Scopes

### Business Logic That Doesn't Belong in Models

**Problem**: Putting complex business logic in model scopes leads to fat models and tight coupling.

```php
<?php

// BAD: Complex business logic in model scope
class Order extends Model
{
    // This scope is doing too much - it's business logic, not a query constraint
    public function scopeEligibleForDiscount($query)
    {
        return $query->where(function ($q) {
            $q->where('total', '>', 100)
              ->where('customer_type', 'premium')
              ->where('created_at', '>', now()->subDays(30))
              ->whereHas('customer', function ($customerQuery) {
                  $customerQuery->where('loyalty_points', '>', 1000)
                              ->where('is_vip', true)
                              ->whereDoesntHave('activeDiscounts');
              });
        });
    }
}

// GOOD: Move business logic to dedicated service
class DiscountEligibilityService
{
    public function getEligibleOrders(): Collection
    {
        return Order::where('total', '>', 100)
                   ->where('customer_type', 'premium')
                   ->recent(30)
                   ->whereHas('customer', function ($q) {
                       $q->where('loyalty_points', '>', 1000)
                         ->vip()
                         ->withoutActiveDiscounts();
                   })
                   ->get();
    }

    public function isOrderEligible(Order $order): bool
    {
        return $order->total > 100
               && $order->customer_type === 'premium'
               && $order->created_at > now()->subDays(30)
               && $this->isCustomerEligible($order->customer);
    }

    protected function isCustomerEligible(Customer $customer): bool
    {
        return $customer->loyalty_points > 1000
               && $customer->is_vip
               && $customer->activeDiscounts()->count() === 0;
    }
}

// Simple, focused scopes in models
class Order extends Model
{
    public function scopeRecent($query, int $days = 30)
    {
        return $query->where('created_at', '>', now()->subDays($days));
    }

    public function scopeMinimumTotal($query, float $amount)
    {
        return $query->where('total', '>=', $amount);
    }

    public function scopePremiumCustomers($query)
    {
        return $query->where('customer_type', 'premium');
    }
}

class Customer extends Model
{
    public function scopeVip($query)
    {
        return $query->where('is_vip', true);
    }

    public function scopeWithoutActiveDiscounts($query)
    {
        return $query->whereDoesntHave('activeDiscounts');
    }

    public function scopeMinimumLoyaltyPoints($query, int $points)
    {
        return $query->where('loyalty_points', '>=', $points);
    }
}
```

### Complex Queries Better Suited for Query Builders

**Problem**: Some complex analytical queries are too specialized for model scopes and should use dedicated query builders or raw SQL.

```php
<?php

// BAD: Complex analytical query as a scope
class User extends Model
{
    // This is too complex and specialized for a general-purpose scope
    public function scopeComplexAnalytics($query)
    {
        return $query->select([
            'users.*',
            DB::raw('COUNT(orders.id) as order_count'),
            DB::raw('SUM(orders.total) as total_spent'),
            DB::raw('AVG(orders.total) as avg_order_value'),
            DB::raw('MAX(orders.created_at) as last_order_date'),
            DB::raw('COUNT(CASE WHEN orders.created_at > DATE_SUB(NOW(), INTERVAL 30 DAY) THEN 1 END) as recent_orders'),
        ])
        ->leftJoin('orders', 'users.id', '=', 'orders.user_id')
        ->leftJoin('order_items', 'orders.id', '=', 'order_items.order_id')
        ->leftJoin('products', 'order_items.product_id', '=', 'products.id')
        ->where('users.created_at', '>', now()->subYear())
        ->groupBy('users.id')
        ->having('order_count', '>', 0)
        ->orderBy('total_spent', 'desc');
    }
}

// GOOD: Dedicated analytics query builder
class UserAnalyticsQueryBuilder
{
    protected Builder $query;

    public function __construct()
    {
        $this->query = User::query();
    }

    public function withOrderStatistics(): self
    {
        $this->query->select([
            'users.*',
            DB::raw('COUNT(orders.id) as order_count'),
            DB::raw('COALESCE(SUM(orders.total), 0) as total_spent'),
            DB::raw('COALESCE(AVG(orders.total), 0) as avg_order_value'),
            DB::raw('MAX(orders.created_at) as last_order_date'),
        ])
        ->leftJoin('orders', 'users.id', '=', 'orders.user_id')
        ->groupBy('users.id');

        return $this;
    }

    public function withRecentActivity(int $days = 30): self
    {
        $this->query->addSelect([
            DB::raw("COUNT(CASE WHEN orders.created_at > DATE_SUB(NOW(), INTERVAL {$days} DAY) THEN 1 END) as recent_orders")
        ]);

        return $this;
    }

    public function onlyActiveCustomers(): self
    {
        $this->query->having('order_count', '>', 0);

        return $this;
    }

    public function topSpenders(int $limit = 100): self
    {
        $this->query->orderBy('total_spent', 'desc')->limit($limit);

        return $this;
    }

    public function get(): Collection
    {
        return $this->query->get();
    }

    public function paginate(int $perPage = 15): LengthAwarePaginator
    {
        return $this->query->paginate($perPage);
    }
}

// Usage in service
class AnalyticsService
{
    public function getTopCustomers(int $limit = 50): Collection
    {
        return (new UserAnalyticsQueryBuilder())
            ->withOrderStatistics()
            ->withRecentActivity()
            ->onlyActiveCustomers()
            ->topSpenders($limit)
            ->get();
    }

    public function getCustomerAnalytics(): LengthAwarePaginator
    {
        return (new UserAnalyticsQueryBuilder())
            ->withOrderStatistics()
            ->withRecentActivity(60)
            ->paginate(25);
    }
}
```

### Performance Pitfalls with Hidden Global Scopes

**Problem**: Global scopes can hide performance issues and make optimization difficult.

```php
<?php

// PROBLEMATIC: Global scope that causes performance issues
class ExpensiveGlobalScope implements Scope
{
    public function apply(Builder $builder, Model $model): void
    {
        // This scope adds expensive joins and subqueries to EVERY query
        $builder->join('user_preferences', 'users.id', '=', 'user_preferences.user_id')
               ->leftJoin('subscriptions', 'users.id', '=', 'subscriptions.user_id')
               ->where(function ($q) {
                   $q->where('user_preferences.privacy_level', 'public')
                     ->orWhere(function ($subQ) {
                         $subQ->where('subscriptions.status', 'active')
                              ->where('subscriptions.tier', 'premium');
                     });
               });
    }
}

// BETTER: Use targeted scopes when needed
class User extends Model
{
    // Don't use expensive global scope

    // Instead, provide specific scopes for when this data is needed
    public function scopeWithPreferences($query)
    {
        return $query->join('user_preferences', 'users.id', '=', 'user_preferences.user_id');
    }

    public function scopePublicProfile($query)
    {
        return $query->withPreferences()
                    ->where('user_preferences.privacy_level', 'public');
    }

    public function scopePremiumSubscribers($query)
    {
        return $query->whereHas('subscription', function ($q) {
            $q->where('status', 'active')
              ->where('tier', 'premium');
        });
    }
}

// Use specific scopes only when the data is actually needed
class UserController extends Controller
{
    public function index()
    {
        // Simple query when we just need basic user data
        $users = User::active()->paginate(20);

        return view('users.index', compact('users'));
    }

    public function publicProfiles()
    {
        // Only apply expensive scope when actually needed
        $users = User::publicProfile()
                    ->with('profile')
                    ->paginate(20);

        return view('users.public-profiles', compact('users'));
    }

    public function premiumDirectory()
    {
        // Specific scope for specific use case
        $users = User::premiumSubscribers()
                    ->with(['subscription', 'profile'])
                    ->paginate(15);

        return view('users.premium-directory', compact('users'));
    }
}
```

---

## Senior Developer Best Practices

### Scope Organization and Naming

**Keep Scopes Focused and Single-Purpose:**

```php
<?php

// GOOD: Focused, single-purpose scopes
class User extends Model
{
    // Clear, descriptive names
    public function scopeActive($query)
    {
        return $query->where('is_active', true);
    }

    public function scopeVerified($query)
    {
        return $query->whereNotNull('email_verified_at');
    }

    public function scopeSubscribed($query)
    {
        return $query->where('subscription_status', 'active');
    }

    // Parameterized for flexibility
    public function scopeJoinedWithinDays($query, int $days)
    {
        return $query->where('created_at', '>=', now()->subDays($days));
    }

    // Role-specific scopes
    public function scopeAdmins($query)
    {
        return $query->whereHas('roles', function ($q) {
            $q->where('name', 'admin');
        });
    }
}

// BAD: Overly complex, multi-purpose scope
class User extends Model
{
    // This scope does too many things
    public function scopeActiveVerifiedSubscribedUsers($query, $roleFilter = null, $daysSinceJoined = null)
    {
        $query = $query->where('is_active', true)
                      ->whereNotNull('email_verified_at')
                      ->where('subscription_status', 'active');

        if ($roleFilter) {
            $query->whereHas('roles', function ($q) use ($roleFilter) {
                $q->where('name', $roleFilter);
            });
        }

        if ($daysSinceJoined) {
            $query->where('created_at', '>=', now()->subDays($daysSinceJoined));
        }

        return $query;
    }
}
```

### Documentation and Team Communication

**Document Hidden Behavior:**

```php
<?php

/**
 * User model with automatic tenant isolation.
 *
 * IMPORTANT: This model uses a global scope that automatically filters
 * by tenant_id for all queries. Use User::withoutTenant() for cross-tenant queries.
 *
 * Global Scopes Applied:
 * - TenantScope: Filters by current user's tenant_id
 * - SoftDeletes: Excludes soft-deleted records
 *
 * @method static Builder active() Only active users
 * @method static Builder verified() Only email-verified users
 * @method static Builder withoutTenant() Remove tenant filtering
 */
class User extends Model
{
    use SoftDeletes, HasTenant;

    /**
     * Get only active users.
     *
     * Active users have is_active = true and email_verified_at is not null.
     */
    public function scopeActive($query)
    {
        return $query->where('is_active', true)
                    ->whereNotNull('email_verified_at');
    }

    /**
     * Filter users by role.
     *
     * @param  string  $role  The role name to filter by
     */
    public function scopeWithRole($query, string $role)
    {
        return $query->whereHas('roles', function ($q) use ($role) {
            $q->where('name', $role);
        });
    }
}

// Service class documentation
/**
 * UserService handles user-related business operations.
 *
 * Note: All methods respect tenant boundaries unless explicitly documented.
 * Use AdminUserService for cross-tenant operations.
 */
class UserService
{
    /**
     * Get active users for the current tenant.
     *
     * This method automatically applies tenant filtering via global scope.
     */
    public function getActiveUsers(): Collection
    {
        return User::active()->get();
    }

    /**
     * Get users across all tenants (admin function).
     *
     * WARNING: This bypasses tenant isolation. Only use for admin functions.
     */
    public function getAllUsersGlobally(): Collection
    {
        return User::withoutTenant()->active()->get();
    }
}
```

### Testing Global Scopes

**Comprehensive Global Scope Testing:**

```php
<?php

class TenantScopeTest extends TestCase
{
    use RefreshDatabase;

    protected User $tenant1User;
    protected User $tenant2User;
    protected Post $tenant1Post;
    protected Post $tenant2Post;

    protected function setUp(): void
    {
        parent::setUp();

        // Create test data for multiple tenants
        $tenant1 = Tenant::factory()->create(['name' => 'Tenant 1']);
        $tenant2 = Tenant::factory()->create(['name' => 'Tenant 2']);

        $this->tenant1User = User::factory()->create(['tenant_id' => $tenant1->id]);
        $this->tenant2User = User::factory()->create(['tenant_id' => $tenant2->id]);

        $this->tenant1Post = Post::factory()->create(['tenant_id' => $tenant1->id]);
        $this->tenant2Post = Post::factory()->create(['tenant_id' => $tenant2->id]);
    }

    /** @test */
    public function global_scope_filters_by_current_user_tenant()
    {
        $this->actingAs($this->tenant1User);

        $posts = Post::all();

        $this->assertCount(1, $posts);
        $this->assertTrue($posts->contains($this->tenant1Post));
        $this->assertFalse($posts->contains($this->tenant2Post));
    }

    /** @test */
    public function can_bypass_global_scope_when_needed()
    {
        $this->actingAs($this->tenant1User);

        $allPosts = Post::withoutTenant()->get();

        $this->assertCount(2, $allPosts);
        $this->assertTrue($allPosts->contains($this->tenant1Post));
        $this->assertTrue($allPosts->contains($this->tenant2Post));
    }

    /** @test */
    public function route_model_binding_respects_global_scope()
    {
        $this->actingAs($this->tenant1User);

        // Should find tenant 1 post
        $response = $this->get(route('posts.show', $this->tenant1Post));
        $response->assertOk();

        // Should not find tenant 2 post (404 due to scope)
        $response = $this->get(route('posts.show', $this->tenant2Post));
        $response->assertNotFound();
    }

    /** @test */
    public function global_scope_does_not_apply_in_console_context()
    {
        // Simulate console environment
        app()->instance('env', 'testing');
        $this->app['env'] = 'testing';

        // In console, global scope should not filter
        $allPosts = Post::all();
        $this->assertCount(2, $allPosts);
    }

    /** @test */
    public function global_scope_works_with_relationships()
    {
        $this->actingAs($this->tenant1User);

        $user = User::with('posts')->first();

        // Should only load posts from the same tenant
        $this->assertCount(1, $user->posts);
        $this->assertEquals($this->tenant1Post->id, $user->posts->first()->id);
    }

    /** @test */
    public function can_query_specific_tenant_as_admin()
    {
        $admin = User::factory()->create(['tenant_id' => $this->tenant1User->tenant_id]);
        $admin->assignRole('super_admin');

        $this->actingAs($admin);

        // Admin can query specific tenant
        $tenant2Posts = Post::forTenant($this->tenant2User->tenant_id)->get();

        $this->assertCount(1, $tenant2Posts);
        $this->assertTrue($tenant2Posts->contains($this->tenant2Post));
    }
}
```

### Performance Monitoring

**Monitor Scope Performance Impact:**

```php
<?php

class ScopePerformanceMonitoring
{
    public static function monitorScopeUsage(): void
    {
        if (app()->environment('local', 'testing')) {
            DB::listen(function ($query) {
                // Log slow queries that might be caused by scopes
                if ($query->time > 100) { // 100ms threshold
                    \Log::warning('Slow query detected', [
                        'sql' => $query->sql,
                        'bindings' => $query->bindings,
                        'time' => $query->time,
                        'stack_trace' => debug_backtrace(DEBUG_BACKTRACE_IGNORE_ARGS, 10)
                    ]);
                }
            });
        }
    }

    public static function benchmarkScopeVsRawQuery(): array
    {
        $iterations = 1000;

        // Benchmark scope usage
        $scopeStart = microtime(true);
        for ($i = 0; $i < $iterations; $i++) {
            User::active()->count();
        }
        $scopeTime = microtime(true) - $scopeStart;

        // Benchmark raw query
        $rawStart = microtime(true);
        for ($i = 0; $i < $iterations; $i++) {
            User::where('is_active', true)
                ->whereNotNull('email_verified_at')
                ->count();
        }
        $rawTime = microtime(true) - $rawStart;

        return [
            'scope_time' => $scopeTime,
            'raw_time' => $rawTime,
            'overhead_percentage' => (($scopeTime - $rawTime) / $rawTime) * 100,
        ];
    }
}

// Service provider registration
class AppServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        if (config('app.monitor_query_performance')) {
            ScopePerformanceMonitoring::monitorScopeUsage();
        }
    }
}
```

### Code Review Guidelines

**Scope Review Checklist:**

```php
<?php

/**
 * Code Review Guidelines for Query Scopes
 *
 * Local Scopes:
 * □ Is the scope name descriptive and follows `scopeCamelCase` convention?
 * □ Does the scope do one thing well (single responsibility)?
 * □ Are parameters type-hinted and documented?
 * □ Is the scope reusable across different contexts?
 * □ Does it return the query builder for chaining?
 *
 * Global Scopes:
 * □ Is there a compelling reason for this to be global?
 * □ Is the scope behavior clearly documented?
 * □ Are there methods to bypass the scope when needed?
 * □ Does it handle edge cases (console commands, admin users)?
 * □ Is there a test ensuring it doesn't leak data between tenants/users?
 *
 * Performance:
 * □ Does the scope add minimal query overhead?
 * □ Are any joins/subqueries necessary and optimized?
 * □ Is there an index supporting the scope's WHERE clauses?
 *
 * Security:
 * □ Does the scope properly isolate tenant data?
 * □ Are there any injection vulnerabilities in dynamic parameters?
 * □ Is sensitive data properly protected?
 */

// Example code review comment templates:

/*
❌ This scope is doing too much:

public function scopeComplexFilter($query, $params) {
    // 50 lines of complex logic
}

✅ Consider breaking this into multiple focused scopes:

public function scopeActive($query) { return $query->where('is_active', true); }
public function scopeInCategory($query, $category) { return $query->where('category', $category); }
public function scopeRecent($query, $days = 30) { return $query->where('created_at', '>=', now()->subDays($days)); }

Then use: Model::active()->inCategory($category)->recent()->get()
*/

/*
❌ Global scope performance concern:

This global scope adds a JOIN to every query. Consider if this should be a local scope instead, or if the data model should be restructured to avoid the JOIN.
*/

/*
✅ Good scope implementation:

- Single responsibility ✓
- Descriptive name ✓
- Type-hinted parameters ✓
- Chainable ✓
- Well documented ✓
*/
```

---

## Testing Strategies

### Testing Local Scopes

**Comprehensive Local Scope Testing:**

```php
<?php

class UserScopeTest extends TestCase
{
    use RefreshDatabase;

    /** @test */
    public function active_scope_filters_correctly()
    {
        // Arrange: Create test data
        $activeUser = User::factory()->create([
            'is_active' => true,
            'email_verified_at' => now(),
        ]);

        $inactiveUser = User::factory()->create([
            'is_active' => false,
            'email_verified_at' => now(),
        ]);

        $unverifiedUser = User::factory()->create([
            'is_active' => true,
            'email_verified_at' => null,
        ]);

        // Act: Apply scope
        $activeUsers = User::active()->get();

        // Assert: Only active and verified users returned
        $this->assertCount(1, $activeUsers);
        $this->assertTrue($activeUsers->contains($activeUser));
        $this->assertFalse($activeUsers->contains($inactiveUser));
        $this->assertFalse($activeUsers->contains($unverifiedUser));
    }

    /** @test */
    public function parameterized_scope_works_with_different_values()
    {
        // Arrange: Create users with different join dates
        $oldUser = User::factory()->create([
            'created_at' => now()->subDays(45),
        ]);

        $recentUser = User::factory()->create([
            'created_at' => now()->subDays(15),
        ]);

        $newUser = User::factory()->create([
            'created_at' => now()->subDays(5),
        ]);

        // Act & Assert: Test different parameter values
        $usersLast30Days = User::joinedWithinDays(30)->get();
        $this->assertCount(2, $usersLast30Days);
        $this->assertFalse($usersLast30Days->contains($oldUser));

        $usersLast7Days = User::joinedWithinDays(7)->get();
        $this->assertCount(1, $usersLast7Days);
        $this->assertTrue($usersLast7Days->contains($newUser));
    }

    /** @test */
    public function scopes_are_chainable()
    {
        // Arrange
        User::factory()->create([
            'is_active' => true,
            'email_verified_at' => now(),
            'created_at' => now()->subDays(10),
        ]);

        User::factory()->create([
            'is_active' => false,
            'email_verified_at' => now(),
            'created_at' => now()->subDays(5),
        ]);

        // Act: Chain multiple scopes
        $result = User::active()
                     ->joinedWithinDays(30)
                     ->get();

        // Assert
        $this->assertCount(1, $result);
    }

    /** @test */
    public function scope_generates_correct_sql()
    {
        // Act: Get the SQL without executing
        $query = User::active()->toSql();

        // Assert: Check SQL structure
        $this->assertStringContains('is_active', $query);
        $this->assertStringContains('email_verified_at', $query);
        $this->assertStringContains('is not null', strtolower($query));
    }

    /** @test */
    public function scope_works_with_pagination()
    {
        // Arrange: Create many users
        User::factory()->count(25)->create([
            'is_active' => true,
            'email_verified_at' => now(),
        ]);

        User::factory()->count(10)->create([
            'is_active' => false,
        ]);

        // Act: Apply scope with pagination
        $paginatedUsers = User::active()->paginate(10);

        // Assert: Pagination works correctly
        $this->assertEquals(25, $paginatedUsers->total());
        $this->assertEquals(10, $paginatedUsers->perPage());
        $this->assertEquals(3, $paginatedUsers->lastPage());
    }
}
```

### Testing Global Scopes

**Global Scope Security and Functionality Testing:**

```php
<?php

class TenantScopeSecurityTest extends TestCase
{
    use RefreshDatabase;

    protected Tenant $tenant1;
    protected Tenant $tenant2;
    protected User $tenant1User;
    protected User $tenant2User;

    protected function setUp(): void
    {
        parent::setUp();

        $this->tenant1 = Tenant::factory()->create();
        $this->tenant2 = Tenant::factory()->create();

        $this->tenant1User = User::factory()->create(['tenant_id' => $this->tenant1->id]);
        $this->tenant2User = User::factory()->create(['tenant_id' => $this->tenant2->id]);
    }

    /** @test */
    public function users_can_only_see_their_tenant_data()
    {
        // Create orders for both tenants
        $tenant1Order = Order::factory()->create(['tenant_id' => $this->tenant1->id]);
        $tenant2Order = Order::factory()->create(['tenant_id' => $this->tenant2->id]);

        // Act as tenant 1 user
        $this->actingAs($this->tenant1User);

        $orders = Order::all();

        // Assert: Only tenant 1 orders visible
        $this->assertCount(1, $orders);
        $this->assertTrue($orders->contains($tenant1Order));
        $this->assertFalse($orders->contains($tenant2Order));
    }

    /** @test */
    public function global_scope_prevents_unauthorized_access_via_find()
    {
        $this->actingAs($this->tenant1User);

        $tenant2Order = Order::factory()->create(['tenant_id' => $this->tenant2->id]);

        // Attempt to find order from different tenant
        $foundOrder = Order::find($tenant2Order->id);

        // Assert: Should return null due to global scope
        $this->assertNull($foundOrder);
    }

    /** @test */
    public function route_model_binding_respects_tenant_scope()
    {
        $tenant2Order = Order::factory()->create(['tenant_id' => $this->tenant2->id]);

        $this->actingAs($this->tenant1User);

        // Attempt to access route with other tenant's order
        $response = $this->get(route('orders.show', $tenant2Order->id));

        // Assert: Should get 404 due to global scope
        $response->assertNotFound();
    }

    /** @test */
    public function admin_can_bypass_tenant_scope_when_needed()
    {
        $admin = User::factory()->create(['tenant_id' => $this->tenant1->id]);
        $admin->assignRole('super_admin');

        Order::factory()->create(['tenant_id' => $this->tenant1->id]);
        Order::factory()->create(['tenant_id' => $this->tenant2->id]);

        $this->actingAs($admin);

        // Admin should see all orders when scope is bypassed
        $allOrders = Order::withoutTenant()->get();
        $this->assertCount(2, $allOrders);

        // But still see only their tenant's orders by default
        $tenantOrders = Order::all();
        $this->assertCount(1, $tenantOrders);
    }

    /** @test */
    public function global_scope_works_with_relationships()
    {
        $customer1 = Customer::factory()->create(['tenant_id' => $this->tenant1->id]);
        $customer2 = Customer::factory()->create(['tenant_id' => $this->tenant2->id]);

        Order::factory()->create(['customer_id' => $customer1->id, 'tenant_id' => $this->tenant1->id]);
        Order::factory()->create(['customer_id' => $customer2->id, 'tenant_id' => $this->tenant2->id]);

        $this->actingAs($this->tenant1User);

        // Load customers with their orders
        $customers = Customer::with('orders')->get();

        // Assert: Only tenant 1 customer and their orders
        $this->assertCount(1, $customers);
        $this->assertEquals($customer1->id, $customers->first()->id);
        $this->assertCount(1, $customers->first()->orders);
    }

    /** @test */
    public function global_scope_handles_unauthenticated_users()
    {
        Order::factory()->create(['tenant_id' => $this->tenant1->id]);
        Order::factory()->create(['tenant_id' => $this->tenant2->id]);

        // No authenticated user
        Auth::logout();

        // Should return empty collection when no user is authenticated
        $orders = Order::all();
        $this->assertCount(0, $orders);
    }

    /** @test */
    public function scope_applies_to_aggregate_queries()
    {
        Order::factory()->count(3)->create(['tenant_id' => $this->tenant1->id]);
        Order::factory()->count(2)->create(['tenant_id' => $this->tenant2->id]);

        $this->actingAs($this->tenant1User);

        // Aggregate queries should also respect scope
        $this->assertEquals(3, Order::count());
        $this->assertEquals(0, Order::where('tenant_id', $this->tenant2->id)->count());
    }

    /** @test */
    public function scope_doesnt_interfere_with_model_creation()
    {
        $this->actingAs($this->tenant1User);

        // Creating new models should work normally
        $order = Order::create([
            'customer_id' => Customer::factory()->create(['tenant_id' => $this->tenant1->id])->id,
            'total' => 100.00,
            'status' => 'pending'
        ]);

        // tenant_id should be set automatically
        $this->assertEquals($this->tenant1->id, $order->tenant_id);

        // Should be able to find the created order
        $foundOrder = Order::find($order->id);
        $this->assertNotNull($foundOrder);
    }
}
```

### Mocking and Fake Data for Scope Testing

```php
<?php

class ScopeTestingHelpers extends TestCase
{
    use RefreshDatabase;

    /**
     * Create test data factory for scope testing
     */
    protected function createScopeTestData(): object
    {
        return (object) [
            'active_users' => User::factory()->count(5)->create([
                'is_active' => true,
                'email_verified_at' => now(),
            ]),
            'inactive_users' => User::factory()->count(3)->create([
                'is_active' => false,
                'email_verified_at' => now(),
            ]),
            'unverified_users' => User::factory()->count(2)->create([
                'is_active' => true,
                'email_verified_at' => null,
            ]),
            'old_users' => User::factory()->count(4)->create([
                'is_active' => true,
                'email_verified_at' => now(),
                'created_at' => now()->subDays(60),
            ]),
            'recent_users' => User::factory()->count(6)->create([
                'is_active' => true,
                'email_verified_at' => now(),
                'created_at' => now()->subDays(15),
            ]),
        ];
    }

    /** @test */
    public function comprehensive_scope_behavior_test()
    {
        $data = $this->createScopeTestData();

        // Test individual scopes
        $this->assertEquals(15, User::active()->count()); // 5 + 4 + 6
        $this->assertEquals(10, User::joinedWithinDays(30)->count()); // 5 + 3 + 2 (recent data)

        // Test combined scopes
        $this->assertEquals(6, User::active()->joinedWithinDays(30)->count());

        // Test scope negation
        $this->assertEquals(5, User::where('is_active', false)->orWhereNull('email_verified_at')->count());
    }

    /**
     * Test scope behavior with different data sets
     */
    public function test_scope_with_various_data_combinations()
    {
        $scenarios = [
            'empty_database' => [],
            'only_active_users' => ['active_users' => 3],
            'only_inactive_users' => ['inactive_users' => 2],
            'mixed_users' => ['active_users' => 5, 'inactive_users' => 3, 'unverified_users' => 2],
        ];

        foreach ($scenarios as $scenarioName => $data) {
            $this->refreshDatabase();

            foreach ($data as $type => $count) {
                User::factory()->count($count)->create($this->getUserAttributesForType($type));
            }

            $activeCount = User::active()->count();
            $expectedCount = $data['active_users'] ?? 0;

            $this->assertEquals(
                $expectedCount,
                $activeCount,
                "Failed for scenario: {$scenarioName}"
            );
        }
    }

    protected function getUserAttributesForType(string $type): array
    {
        return match($type) {
            'active_users' => ['is_active' => true, 'email_verified_at' => now()],
            'inactive_users' => ['is_active' => false, 'email_verified_at' => now()],
            'unverified_users' => ['is_active' => true, 'email_verified_at' => null],
            default => []
        };
    }
}
```

---

## Summary & Key Takeaways

### When to Use Local Scopes

• **Common query patterns**: Use local scopes for frequently repeated WHERE clauses, ORDER BY, or JOIN conditions that appear across multiple controllers and services

• **Business domain filtering**: Create scopes that reflect your business language - `User::active()`, `Post::published()`, `Order::pending()` - making queries self-documenting

• **Parameterized filtering**: Use local scopes with parameters for flexible, reusable query constraints like `Post::createdWithinDays(30)` or `Product::inPriceRange($min, $max)`

• **Chainable query building**: Local scopes excel when you need to combine multiple conditions: `User::active()->verified()->joinedWithinDays(30)`

### When to Use Global Scopes

• **Security requirements**: Use global scopes for security-critical filtering like tenant isolation in multi-tenant applications or role-based data access

• **Consistent business rules**: Apply global scopes when certain conditions should **always** be enforced (published content, non-deleted records, active status)

• **Data integrity**: Use global scopes to ensure compliance with business rules that should never be accidentally bypassed

• **Framework integration**: Global scopes work seamlessly with route model binding, relationships, and Eloquent's lazy loading

### Critical Pitfalls to Avoid

• **Don't put business logic in scopes**: Keep scopes focused on query constraints, not complex business rules or external API calls

• **Avoid expensive global scopes**: Global scopes run on every query - don't add expensive JOINs or subqueries that aren't always needed

• **Document hidden behavior**: Global scopes are invisible to calling code, so document them clearly to prevent team confusion

• **Test tenant isolation**: Always write comprehensive tests for global scopes to ensure they don't leak data between tenants or users

• **Don't overuse global scopes**: Prefer local scopes for most scenarios - global scopes should be reserved for truly global constraints

### Best Practices for Production

• **Single responsibility**: Each scope should do one thing well - filter by status, apply date ranges, or join specific relationships

• **Descriptive naming**: Use clear, business-domain names that make queries read like English: `Order::pending()->forCustomer($customer)`

• **Type hints and documentation**: Always type-hint parameters and document scope behavior, especially for global scopes

• **Performance awareness**: Monitor query performance and ensure scopes don't inadvertently create N+1 problems or missing index issues

• **Escape hatches**: Provide ways to bypass global scopes when needed (admin functions, data migrations, reports)

Laravel query scopes are powerful tools for creating expressive, maintainable queries while keeping your codebase DRY. Use local scopes liberally for common patterns, reserve global scopes for security and compliance requirements, and always prioritize code clarity and team understanding over clever abstractions.
