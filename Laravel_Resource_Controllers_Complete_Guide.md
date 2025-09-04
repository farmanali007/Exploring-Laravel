# Laravel Resource Controllers: Complete Guide

## 1. Introduction – What & Why

**Resource Controllers** are Laravel's elegant solution to the common CRUD (Create, Read, Update, Delete) pattern. Instead of manually defining routes and controller methods for each operation, Laravel provides a convention that maps HTTP verbs to controller methods automatically.

### Why Use Resource Controllers?

- **Convention over Configuration**: Follows RESTful principles automatically
- **Consistency**: Standardized method names across your application
- **Rapid Development**: Less boilerplate code
- **Team Collaboration**: Everyone knows the expected structure
- **Route Organization**: Clean, predictable URL patterns

### The RESTful Convention

| HTTP Verb | URI                | Controller Method | Purpose                    |
| --------- | ------------------ | ----------------- | -------------------------- |
| GET       | `/posts`           | `index()`         | Display list of resources  |
| GET       | `/posts/create`    | `create()`        | Show form to create new    |
| POST      | `/posts`           | `store()`         | Store new resource         |
| GET       | `/posts/{id}`      | `show()`          | Display specific resource  |
| GET       | `/posts/{id}/edit` | `edit()`          | Show form to edit resource |
| PUT/PATCH | `/posts/{id}`      | `update()`        | Update specific resource   |
| DELETE    | `/posts/{id}`      | `destroy()`       | Delete specific resource   |

---

## 2. Basic Usage

### Creating a Resource Controller

```bash
# Basic resource controller
php artisan make:controller PostController --resource

# With model binding
php artisan make:controller PostController --resource --model=Post

# API resource controller (no create/edit methods)
php artisan make:controller Api/PostController --api --resource
```

### Basic Resource Controller Structure

```php
<?php

namespace App\Http\Controllers;

use App\Http\Requests\StorePostRequest;
use App\Http\Requests\UpdatePostRequest;
use App\Models\Post;

class PostController extends Controller
{
    /**
     * Display a listing of the resource.
     */
    public function index()
    {
        $posts = Post::latest()->paginate(15);
        return view('posts.index', compact('posts'));
    }

    /**
     * Show the form for creating a new resource.
     */
    public function create()
    {
        return view('posts.create');
    }

    /**
     * Store a newly created resource in storage.
     */
    public function store(StorePostRequest $request)
    {
        $post = Post::create($request->validated());

        return redirect()
            ->route('posts.show', $post)
            ->with('success', 'Post created successfully!');
    }

    /**
     * Display the specified resource.
     */
    public function show(Post $post)
    {
        return view('posts.show', compact('post'));
    }

    /**
     * Show the form for editing the specified resource.
     */
    public function edit(Post $post)
    {
        return view('posts.edit', compact('post'));
    }

    /**
     * Update the specified resource in storage.
     */
    public function update(UpdatePostRequest $request, Post $post)
    {
        $post->update($request->validated());

        return redirect()
            ->route('posts.show', $post)
            ->with('success', 'Post updated successfully!');
    }

    /**
     * Remove the specified resource from storage.
     */
    public function destroy(Post $post)
    {
        $post->delete();

        return redirect()
            ->route('posts.index')
            ->with('success', 'Post deleted successfully!');
    }
}
```

### Registering Resource Routes

```php
// routes/web.php
use App\Http\Controllers\PostController;

// Creates all 7 RESTful routes
Route::resource('posts', PostController::class);

// This is equivalent to:
// Route::get('/posts', [PostController::class, 'index'])->name('posts.index');
// Route::get('/posts/create', [PostController::class, 'create'])->name('posts.create');
// Route::post('/posts', [PostController::class, 'store'])->name('posts.store');
// Route::get('/posts/{post}', [PostController::class, 'show'])->name('posts.show');
// Route::get('/posts/{post}/edit', [PostController::class, 'edit'])->name('posts.edit');
// Route::put('/posts/{post}', [PostController::class, 'update'])->name('posts.update');
// Route::delete('/posts/{post}', [PostController::class, 'destroy'])->name('posts.destroy');
```

---

## 3. Real-World Example: Blog Management System

Let's build a comprehensive blog post management system to see resource controllers in action.

### The Post Model

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\SoftDeletes;

class Post extends Model
{
    use SoftDeletes;

    protected $fillable = [
        'title', 'slug', 'content', 'excerpt', 'status', 'published_at', 'user_id'
    ];

    protected $casts = [
        'published_at' => 'datetime',
    ];

    // Relationships
    public function author()
    {
        return $this->belongsTo(User::class, 'user_id');
    }

    public function categories()
    {
        return $this->belongsToMany(Category::class);
    }

    // Scopes
    public function scopePublished($query)
    {
        return $query->where('status', 'published')
                    ->where('published_at', '<=', now());
    }
}
```

### Enhanced PostController with Business Logic

```php
<?php

namespace App\Http\Controllers;

use App\Http\Requests\StorePostRequest;
use App\Http\Requests\UpdatePostRequest;
use App\Models\Post;
use App\Models\Category;
use App\Services\PostService;
use Illuminate\Http\Request;

class PostController extends Controller
{
    public function __construct(
        private PostService $postService
    ) {
        $this->middleware('auth')->except(['index', 'show']);
        $this->middleware('can:create,App\Models\Post')->only(['create', 'store']);
        $this->middleware('can:update,post')->only(['edit', 'update']);
        $this->middleware('can:delete,post')->only('destroy');
    }

    public function index(Request $request)
    {
        $query = Post::with(['author', 'categories'])
                    ->published()
                    ->latest('published_at');

        // Handle search
        if ($request->filled('search')) {
            $query->where('title', 'like', '%' . $request->search . '%')
                  ->orWhere('content', 'like', '%' . $request->search . '%');
        }

        // Handle category filter
        if ($request->filled('category')) {
            $query->whereHas('categories', function ($q) use ($request) {
                $q->where('slug', $request->category);
            });
        }

        $posts = $query->paginate(12)->withQueryString();
        $categories = Category::all();

        return view('posts.index', compact('posts', 'categories'));
    }

    public function create()
    {
        $categories = Category::all();
        return view('posts.create', compact('categories'));
    }

    public function store(StorePostRequest $request)
    {
        try {
            $post = $this->postService->createPost($request->validated(), auth()->user());

            return redirect()
                ->route('posts.show', $post)
                ->with('success', 'Post created successfully!');

        } catch (\Exception $e) {
            return back()
                ->withInput()
                ->with('error', 'Failed to create post. Please try again.');
        }
    }

    public function show(Post $post)
    {
        // Load relationships
        $post->load(['author', 'categories']);

        // Increment views (could be moved to service/job)
        $post->increment('views');

        // Get related posts
        $relatedPosts = Post::published()
            ->where('id', '!=', $post->id)
            ->whereHas('categories', function ($query) use ($post) {
                $query->whereIn('categories.id', $post->categories->pluck('id'));
            })
            ->limit(3)
            ->get();

        return view('posts.show', compact('post', 'relatedPosts'));
    }

    public function edit(Post $post)
    {
        $categories = Category::all();
        return view('posts.edit', compact('post', 'categories'));
    }

    public function update(UpdatePostRequest $request, Post $post)
    {
        try {
            $this->postService->updatePost($post, $request->validated());

            return redirect()
                ->route('posts.show', $post)
                ->with('success', 'Post updated successfully!');

        } catch (\Exception $e) {
            return back()
                ->withInput()
                ->with('error', 'Failed to update post. Please try again.');
        }
    }

    public function destroy(Post $post)
    {
        try {
            $this->postService->deletePost($post);

            return redirect()
                ->route('posts.index')
                ->with('success', 'Post deleted successfully!');

        } catch (\Exception $e) {
            return back()
                ->with('error', 'Failed to delete post. Please try again.');
        }
    }
}
```

---

## 4. Advanced Usage

### 4.1 Partial Resource Routes

Sometimes you don't need all 7 resource methods:

```php
// Only specific methods
Route::resource('posts', PostController::class)
    ->only(['index', 'show', 'create', 'store']);

// Exclude specific methods
Route::resource('posts', PostController::class)
    ->except(['create', 'edit']);

// API resources (excludes create and edit by default)
Route::apiResource('posts', PostController::class);

// Multiple resources with same restrictions
Route::resources([
    'posts' => PostController::class,
    'categories' => CategoryController::class,
], ['only' => ['index', 'show']]);
```

### 4.2 Nested Resource Controllers

Handle relationships between resources:

```php
// routes/web.php
Route::resource('posts.comments', CommentController::class);
// Creates routes like: /posts/{post}/comments/{comment}

// With shallow nesting (Laravel 8+)
Route::resource('posts.comments', CommentController::class)->shallow();
// Show, edit, update, destroy don't need parent: /comments/{comment}
```

```php
<?php

namespace App\Http\Controllers;

use App\Models\Post;
use App\Models\Comment;
use App\Http\Requests\StoreCommentRequest;

class CommentController extends Controller
{
    public function index(Post $post)
    {
        $comments = $post->comments()
            ->with('author')
            ->latest()
            ->paginate(20);

        return view('comments.index', compact('post', 'comments'));
    }

    public function store(StoreCommentRequest $request, Post $post)
    {
        $comment = $post->comments()->create([
            'content' => $request->content,
            'user_id' => auth()->id(),
        ]);

        return back()->with('success', 'Comment added successfully!');
    }

    // For shallow routes, these methods don't receive $post
    public function show(Comment $comment)
    {
        return view('comments.show', compact('comment'));
    }

    public function update(UpdateCommentRequest $request, Comment $comment)
    {
        $this->authorize('update', $comment);

        $comment->update($request->validated());

        return back()->with('success', 'Comment updated!');
    }
}
```

### 4.3 API Resource Controllers

API controllers focus on data exchange without web forms:

```php
<?php

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Http\Requests\StorePostRequest;
use App\Http\Resources\PostResource;
use App\Models\Post;

class PostController extends Controller
{
    public function index()
    {
        $posts = Post::with('author')->published()->paginate();

        return PostResource::collection($posts);
    }

    public function store(StorePostRequest $request)
    {
        $post = Post::create($request->validated());

        return new PostResource($post->load('author'));
    }

    public function show(Post $post)
    {
        return new PostResource($post->load(['author', 'categories']));
    }

    public function update(UpdatePostRequest $request, Post $post)
    {
        $post->update($request->validated());

        return new PostResource($post->load('author'));
    }

    public function destroy(Post $post)
    {
        $post->delete();

        return response()->json([
            'message' => 'Post deleted successfully'
        ]);
    }
}
```

### 4.4 Advanced Route Model Binding

Customize how models are resolved:

```php
// In your controller
public function show(Post $post)
{
    // $post is automatically resolved by ID
}

// Custom route key (use slug instead of ID)
// In Post model:
public function getRouteKeyName()
{
    return 'slug';
}

// Or in routes:
Route::resource('posts', PostController::class)
    ->parameters(['posts' => 'post:slug']);

// Advanced binding with scopes
Route::bind('post', function ($value) {
    return Post::published()->where('slug', $value)->firstOrFail();
});
```

### 4.5 Form Requests for Validation

Separate validation logic from controllers:

```php
<?php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;

class StorePostRequest extends FormRequest
{
    public function authorize()
    {
        return auth()->check();
    }

    public function rules()
    {
        return [
            'title' => ['required', 'string', 'max:255'],
            'slug' => ['required', 'string', 'unique:posts,slug'],
            'content' => ['required', 'string', 'min:100'],
            'excerpt' => ['nullable', 'string', 'max:500'],
            'status' => ['required', Rule::in(['draft', 'published'])],
            'published_at' => ['nullable', 'date', 'after:now'],
            'categories' => ['array', 'exists:categories,id'],
        ];
    }

    public function messages()
    {
        return [
            'content.min' => 'Post content must be at least 100 characters long.',
        ];
    }

    protected function prepareForValidation()
    {
        if (!$this->slug && $this->title) {
            $this->merge([
                'slug' => Str::slug($this->title)
            ]);
        }
    }
}
```

### 4.6 Middleware on Resource Methods

Apply middleware to specific resource actions:

```php
// In controller constructor
public function __construct()
{
    // Apply to all methods
    $this->middleware('auth');

    // Apply to specific methods
    $this->middleware('throttle:10,1')->only('store');
    $this->middleware('can:admin')->except(['index', 'show']);
}

// Or in routes
Route::resource('posts', PostController::class)
    ->middleware(['auth', 'verified'])
    ->only(['create', 'store', 'edit', 'update', 'destroy']);
```

---

## 5. Best Practices & Senior-Level Tips

### 5.1 Keep Controllers Thin

❌ **Bad Practice: Fat Controller**

```php
public function store(Request $request)
{
    // Validation in controller
    $request->validate([
        'title' => 'required|max:255',
        'content' => 'required',
    ]);

    // Business logic in controller
    $slug = Str::slug($request->title);
    $counter = 1;
    while (Post::where('slug', $slug)->exists()) {
        $slug = Str::slug($request->title) . '-' . $counter++;
    }

    // Direct model creation
    $post = Post::create([
        'title' => $request->title,
        'slug' => $slug,
        'content' => $request->content,
        'user_id' => auth()->id(),
    ]);

    // File handling in controller
    if ($request->hasFile('image')) {
        $path = $request->file('image')->store('posts', 'public');
        $post->update(['image' => $path]);
    }

    // Email sending in controller
    Mail::to($post->author)->send(new PostCreated($post));

    return redirect()->route('posts.show', $post);
}
```

✅ **Good Practice: Thin Controller**

```php
public function store(StorePostRequest $request)
{
    $post = $this->postService->createPost(
        $request->validated(),
        auth()->user()
    );

    return redirect()
        ->route('posts.show', $post)
        ->with('success', 'Post created successfully!');
}
```

### 5.2 Use Service Classes for Complex Logic

```php
<?php

namespace App\Services;

use App\Models\Post;
use App\Models\User;
use App\Jobs\ProcessPostImage;
use App\Events\PostCreated;
use Illuminate\Support\Str;

class PostService
{
    public function createPost(array $data, User $author): Post
    {
        $data['user_id'] = $author->id;
        $data['slug'] = $this->generateUniqueSlug($data['title']);

        $post = Post::create($data);

        if (isset($data['categories'])) {
            $post->categories()->attach($data['categories']);
        }

        if (isset($data['image'])) {
            ProcessPostImage::dispatch($post, $data['image']);
        }

        event(new PostCreated($post));

        return $post->load(['author', 'categories']);
    }

    private function generateUniqueSlug(string $title): string
    {
        $slug = Str::slug($title);
        $counter = 1;

        while (Post::where('slug', $slug)->exists()) {
            $slug = Str::slug($title) . '-' . $counter++;
        }

        return $slug;
    }
}
```

### 5.3 Resource-Specific Middleware

Create custom middleware for resource-specific logic:

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;

class EnsurePostIsPublished
{
    public function handle(Request $request, Closure $next)
    {
        $post = $request->route('post');

        if ($post && $post->status !== 'published' && !auth()->user()?->can('view-unpublished', $post)) {
            abort(404);
        }

        return $next($request);
    }
}

// Apply in controller
public function __construct()
{
    $this->middleware(EnsurePostIsPublished::class)->only('show');
}
```

### 5.4 Optimized Queries

Prevent N+1 problems and optimize database queries:

```php
public function index()
{
    $posts = Post::query()
        ->with(['author:id,name', 'categories:id,name,slug']) // Only load needed columns
        ->withCount(['comments', 'likes']) // Get counts efficiently
        ->published()
        ->latest('published_at')
        ->paginate(15);

    return view('posts.index', compact('posts'));
}
```

### 5.5 Response Consistency

Use consistent response patterns:

```php
trait ApiResponses
{
    protected function successResponse($data, $message = null, $code = 200)
    {
        return response()->json([
            'success' => true,
            'data' => $data,
            'message' => $message,
        ], $code);
    }

    protected function errorResponse($message, $code = 400)
    {
        return response()->json([
            'success' => false,
            'message' => $message,
        ], $code);
    }
}

// In API controller
public function store(StorePostRequest $request)
{
    $post = $this->postService->createPost($request->validated(), auth()->user());

    return $this->successResponse(
        new PostResource($post),
        'Post created successfully',
        201
    );
}
```

---

## 6. Comparison: Resource vs Alternatives

### Resource Controllers vs Single Action Controllers

**Resource Controllers:**

- ✅ RESTful convention
- ✅ Less boilerplate
- ✅ Consistent naming
- ❌ Can become large
- ❌ Less granular control

**Single Action Controllers:**

- ✅ Single responsibility
- ✅ Easier testing
- ✅ Clearer purpose
- ❌ More files to manage
- ❌ More route definitions

```php
// Single action controller example
<?php

namespace App\Http\Controllers\Post;

use App\Http\Controllers\Controller;

class CreatePostController extends Controller
{
    public function __invoke(StorePostRequest $request)
    {
        // Handle post creation
    }
}

// Route
Route::post('posts', CreatePostController::class);
```

### When to Use What

**Use Resource Controllers When:**

- Standard CRUD operations
- Following RESTful principles
- Building traditional web applications
- Team prefers convention

**Use Single Action Controllers When:**

- Complex, specialized operations
- Microservices architecture
- Domain-driven design approach
- Operations don't fit CRUD pattern

**Use Custom Routes When:**

- Non-standard operations
- Legacy system integration
- Unique business requirements

---

## 7. Laravel 10/11 Updates

### Enhanced Route Model Binding (Laravel 10+)

```php
// Enum route binding
Route::get('/posts/{status}', function (PostStatus $status) {
    // $status is automatically resolved from enum
});

// Multiple parameter binding
Route::get('/posts/{post}/comments/{comment}', function (Post $post, Comment $comment) {
    // Both models automatically scoped
});
```

### Improved API Resources (Laravel 11+)

```php
// Conditional resource loading
class PostResource extends JsonResource
{
    public function toArray($request)
    {
        return [
            'id' => $this->id,
            'title' => $this->title,
            'author' => $this->whenLoaded('author'),
            'categories' => CategoryResource::collection($this->whenLoaded('categories')),
            'admin_notes' => $this->when($request->user()?->isAdmin(), $this->admin_notes),
        ];
    }
}
```

### Better Validation (Laravel 11+)

```php
// Nested validation rules
public function rules()
{
    return [
        'post' => ['required', 'array'],
        'post.title' => ['required', 'string', 'max:255'],
        'post.categories' => ['array', 'exists:categories,id'],
        'post.tags.*' => ['string', 'max:50'],
    ];
}
```

---

## 8. Use Cases & Real-World Applications

### ✅ Perfect Use Cases

1. **E-commerce Product Management**

   ```php
   Route::resource('products', ProductController::class);
   // Standard CRUD for products, categories, orders
   ```

2. **User Management Systems**

   ```php
   Route::resource('users', UserController::class)->except('destroy');
   // User profiles with standard operations
   ```

3. **Content Management**

   ```php
   Route::resource('articles', ArticleController::class);
   Route::resource('articles.comments', CommentController::class)->shallow();
   ```

4. **Task/Project Management**
   ```php
   Route::resource('projects', ProjectController::class);
   Route::resource('projects.tasks', TaskController::class);
   ```

### ❌ Not Suitable When

1. **Complex Workflows**

   ```php
   // Don't force these into resource controllers
   Route::post('orders/{order}/approve', ApproveOrderController::class);
   Route::post('users/{user}/suspend', SuspendUserController::class);
   Route::post('posts/{post}/publish', PublishPostController::class);
   ```

2. **Search/Filtering Endpoints**

   ```php
   // Better as dedicated controllers
   Route::get('search/posts', SearchPostsController::class);
   Route::get('reports/sales', SalesReportController::class);
   ```

3. **Third-party Integrations**
   ```php
   // Specific integration logic
   Route::post('webhooks/stripe', StripeWebhookController::class);
   Route::post('api/import-users', ImportUsersController::class);
   ```

### Real-World Example: E-commerce System

```php
// Good resource controller usage
Route::resource('products', ProductController::class);
Route::resource('categories', CategoryController::class);
Route::resource('orders', OrderController::class)->only(['index', 'show']);

// Don't force these into resource controllers
Route::post('cart/add/{product}', AddToCartController::class);
Route::post('orders/{order}/cancel', CancelOrderController::class);
Route::post('products/{product}/review', CreateReviewController::class);
Route::get('search', ProductSearchController::class);
```

---

## 📝 Resource Controller Checklist

### ✅ Do's

- Use resource controllers for standard CRUD operations
- Keep controllers thin, move business logic to services
- Use Form Requests for validation
- Implement proper authorization with policies
- Use route model binding for cleaner parameter handling
- Apply appropriate middleware for authentication/authorization
- Use API resources for consistent JSON responses
- Optimize database queries with eager loading
- Handle exceptions gracefully
- Follow RESTful naming conventions

### ❌ Don'ts

- Don't put complex business logic in controllers
- Don't ignore validation - always use Form Requests
- Don't forget authorization checks
- Don't return raw models in API responses
- Don't create resource controllers for non-CRUD operations
- Don't skip error handling
- Don't ignore N+1 query problems
- Don't mix web and API logic in the same controller
- Don't create overly large resource controllers
- Don't forget to use transactions for multi-step operations

### 🔧 Performance Tips

- Use `only()` and `except()` to limit unnecessary routes
- Implement caching where appropriate
- Use database indexes for commonly queried fields
- Consider using queued jobs for heavy operations
- Implement proper pagination
- Use resource collections for better memory management
- Cache expensive queries in index methods

### 🎯 Senior-Level Considerations

- Design controllers with single responsibility in mind
- Consider using DTOs (Data Transfer Objects) for complex data
- Implement proper logging and monitoring
- Use feature flags for gradual rollouts
- Consider API versioning strategies
- Implement proper rate limiting
- Use database transactions appropriately
- Consider implementing soft deletes where needed
- Plan for scalability from the beginning

---

_Remember: Resource controllers are a tool, not a rule. Use them when they fit naturally, and don't force complex operations into the resource pattern. The goal is clean, maintainable code that serves your application's needs._
