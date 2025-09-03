# Senior-Level Laravel Raw SQL Query Builder Walkthrough

## Core Use Cases for Raw SQL in Query Builder

Raw SQL in Laravel's Query Builder serves specific purposes where Eloquent's abstraction becomes limiting:

### 1. Complex Aggregations & Analytics

```php
// Revenue analysis with conditional aggregation
DB::table('orders')
    ->selectRaw('
        DATE_FORMAT(created_at, "%Y-%m") as month,
        SUM(CASE WHEN status = "completed" THEN total ELSE 0 END) as completed_revenue,
        SUM(CASE WHEN status = "refunded" THEN total ELSE 0 END) as refunded_revenue,
        COUNT(DISTINCT customer_id) as unique_customers
    ')
    ->groupByRaw('DATE_FORMAT(created_at, "%Y-%m")')
    ->orderByRaw('month DESC')
    ->get();
```

### 2. Database-Specific Features

```php
// PostgreSQL array operations
DB::table('products')
    ->whereRaw("tags @> ?", [json_encode(['electronics'])])
    ->selectRaw("array_length(tags, 1) as tag_count")
    ->get();

// MySQL full-text search with relevance scoring
DB::table('articles')
    ->selectRaw('*, MATCH(title, content) AGAINST(? IN NATURAL LANGUAGE MODE) as relevance', [$query])
    ->whereRaw('MATCH(title, content) AGAINST(? IN NATURAL LANGUAGE MODE)', [$query])
    ->orderByDesc('relevance')
    ->get();
```

### 3. Performance-Critical Queries

```php
// Bulk operations that would be inefficient with Eloquent
DB::table('user_stats')
    ->whereRaw('last_login < DATE_SUB(NOW(), INTERVAL 30 DAY)')
    ->update([
        'status' => 'inactive',
        'updated_at' => DB::raw('NOW()')
    ]);
```

## Advanced Raw SQL Techniques

### DB::raw() - The Foundation

```php
// Basic usage
DB::table('products')
    ->select('name', DB::raw('price * 0.9 as discounted_price'))
    ->where('category_id', 1)
    ->get();

// Complex calculations
DB::table('orders')
    ->selectRaw('
        customer_id,
        AVG(total) as avg_order_value,
        STDDEV(total) as order_variance,
        DATEDIFF(MAX(created_at), MIN(created_at)) as customer_lifespan_days
    ')
    ->groupBy('customer_id')
    ->havingRaw('COUNT(*) >= ? AND AVG(total) > ?', [5, 100])
    ->get();
```

### selectRaw() - Advanced Selections

```php
// Conditional aggregations with window functions
DB::table('sales')
    ->selectRaw('
        product_id,
        month,
        revenue,
        LAG(revenue, 1) OVER (PARTITION BY product_id ORDER BY month) as prev_month_revenue,
        revenue - LAG(revenue, 1) OVER (PARTITION BY product_id ORDER BY month) as revenue_change,
        RANK() OVER (PARTITION BY month ORDER BY revenue DESC) as monthly_rank
    ')
    ->fromRaw('(
        SELECT
            product_id,
            DATE_FORMAT(created_at, "%Y-%m") as month,
            SUM(amount) as revenue
        FROM sales
        WHERE created_at >= "2024-01-01"
        GROUP BY product_id, month
    ) as monthly_sales')
    ->get();
```

### whereRaw() - Complex Conditions

```php
// Geospatial queries
DB::table('stores')
    ->selectRaw('
        id, name, latitude, longitude,
        ST_Distance_Sphere(
            POINT(longitude, latitude),
            POINT(?, ?)
        ) / 1000 as distance_km
    ', [$userLng, $userLat])
    ->whereRaw('
        ST_Distance_Sphere(
            POINT(longitude, latitude),
            POINT(?, ?)
        ) <= ?
    ', [$userLng, $userLat, $radiusMeters])
    ->orderByRaw('distance_km ASC')
    ->get();

// Advanced date/time conditions
DB::table('events')
    ->whereRaw('
        DAYOFWEEK(start_date) IN (1, 7) -- Weekends only
        AND TIME(start_date) BETWEEN "09:00:00" AND "17:00:00"
        AND YEAR(start_date) = YEAR(CURDATE())
    ')
    ->get();
```

### JSON Functions (MySQL 5.7+/PostgreSQL)

```php
// Complex JSON operations
DB::table('users')
    ->selectRaw('
        id, name,
        JSON_UNQUOTE(JSON_EXTRACT(preferences, "$.theme")) as theme,
        JSON_LENGTH(JSON_EXTRACT(preferences, "$.notifications")) as notification_count
    ')
    ->whereRaw('JSON_EXTRACT(preferences, "$.email_notifications") = true')
    ->whereRaw('JSON_CONTAINS(preferences, ?, "$.roles")', ['"admin"'])
    ->get();

// PostgreSQL JSONB operations
DB::table('products')
    ->selectRaw("
        id, name,
        specifications->>'cpu' as cpu,
        specifications->'ram'->>'size' as ram_size
    ")
    ->whereRaw("specifications @> ?", ['{"category": "laptop"}'])
    ->whereRaw("(specifications->'price')::int BETWEEN ? AND ?", [500, 2000])
    ->get();
```

### Common Table Expressions (CTEs)

```php
// Recursive CTE for hierarchical data
$hierarchyQuery = DB::select("
    WITH RECURSIVE category_hierarchy AS (
        -- Base case: root categories
        SELECT id, name, parent_id, 0 as level,
               CAST(name AS CHAR(1000)) as path
        FROM categories
        WHERE parent_id IS NULL

        UNION ALL

        -- Recursive case: child categories
        SELECT c.id, c.name, c.parent_id, h.level + 1,
               CONCAT(h.path, ' > ', c.name)
        FROM categories c
        INNER JOIN category_hierarchy h ON c.parent_id = h.id
        WHERE h.level < 10 -- Prevent infinite recursion
    )
    SELECT id, name, level, path
    FROM category_hierarchy
    ORDER BY path
");

// Advanced analytics with CTEs
$revenueAnalysis = DB::select("
    WITH monthly_revenue AS (
        SELECT
            DATE_TRUNC('month', created_at) as month,
            SUM(total) as revenue
        FROM orders
        WHERE created_at >= ? AND status = 'completed'
        GROUP BY DATE_TRUNC('month', created_at)
    ),
    revenue_with_growth AS (
        SELECT
            month,
            revenue,
            LAG(revenue) OVER (ORDER BY month) as prev_month_revenue,
            (revenue - LAG(revenue) OVER (ORDER BY month)) /
            LAG(revenue) OVER (ORDER BY month) * 100 as growth_rate
        FROM monthly_revenue
    )
    SELECT
        month,
        revenue,
        COALESCE(growth_rate, 0) as growth_rate,
        AVG(growth_rate) OVER (
            ORDER BY month
            ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
        ) as rolling_avg_growth
    FROM revenue_with_growth
    ORDER BY month
", [now()->subYear()]);
```

## Real-World Examples

### E-commerce Analytics Dashboard

```php
class EcommerceAnalytics
{
    public function getDashboardMetrics($startDate, $endDate)
    {
        return DB::select("
            WITH order_metrics AS (
                SELECT
                    COUNT(*) as total_orders,
                    COUNT(DISTINCT customer_id) as unique_customers,
                    SUM(total) as gross_revenue,
                    SUM(CASE WHEN status = 'completed' THEN total ELSE 0 END) as net_revenue,
                    AVG(total) as avg_order_value
                FROM orders
                WHERE created_at BETWEEN ? AND ?
            ),
            product_performance AS (
                SELECT
                    p.category_id,
                    c.name as category_name,
                    COUNT(oi.id) as items_sold,
                    SUM(oi.quantity * oi.price) as category_revenue
                FROM products p
                JOIN categories c ON p.category_id = c.id
                JOIN order_items oi ON p.id = oi.product_id
                JOIN orders o ON oi.order_id = o.id
                WHERE o.created_at BETWEEN ? AND ? AND o.status = 'completed'
                GROUP BY p.category_id, c.name
            ),
            customer_segments AS (
                SELECT
                    CASE
                        WHEN customer_lifetime_value > 1000 THEN 'High Value'
                        WHEN customer_lifetime_value > 500 THEN 'Medium Value'
                        ELSE 'Low Value'
                    END as segment,
                    COUNT(*) as customer_count,
                    AVG(customer_lifetime_value) as avg_clv
                FROM (
                    SELECT
                        customer_id,
                        SUM(total) as customer_lifetime_value
                    FROM orders
                    WHERE status = 'completed'
                    GROUP BY customer_id
                ) customer_totals
                GROUP BY segment
            )
            SELECT
                (SELECT json_object(
                    'total_orders', total_orders,
                    'unique_customers', unique_customers,
                    'gross_revenue', gross_revenue,
                    'net_revenue', net_revenue,
                    'avg_order_value', avg_order_value
                ) FROM order_metrics) as order_metrics,

                (SELECT json_arrayagg(
                    json_object(
                        'category_name', category_name,
                        'items_sold', items_sold,
                        'revenue', category_revenue
                    )
                ) FROM product_performance) as category_performance,

                (SELECT json_arrayagg(
                    json_object(
                        'segment', segment,
                        'customer_count', customer_count,
                        'avg_clv', avg_clv
                    )
                ) FROM customer_segments) as customer_segments
        ", [$startDate, $endDate, $startDate, $endDate]);
    }
}
```

### Financial Data Processing

```php
class FinancialReporting
{
    public function generateProfitLossStatement($year, $quarter = null)
    {
        $dateFilter = $quarter
            ? "YEAR(transaction_date) = ? AND QUARTER(transaction_date) = ?"
            : "YEAR(transaction_date) = ?";

        $params = $quarter ? [$year, $quarter] : [$year];

        return DB::select("
            WITH account_balances AS (
                SELECT
                    a.code,
                    a.name,
                    a.type,
                    a.category,
                    SUM(
                        CASE
                            WHEN a.type IN ('revenue', 'liability', 'equity')
                            THEN t.credit_amount - t.debit_amount
                            ELSE t.debit_amount - t.credit_amount
                        END
                    ) as balance
                FROM accounts a
                LEFT JOIN transactions t ON a.id = t.account_id
                WHERE {$dateFilter}
                GROUP BY a.id, a.code, a.name, a.type, a.category
            ),
            revenue_summary AS (
                SELECT
                    category,
                    SUM(balance) as total
                FROM account_balances
                WHERE type = 'revenue'
                GROUP BY category
            ),
            expense_summary AS (
                SELECT
                    category,
                    SUM(balance) as total
                FROM account_balances
                WHERE type = 'expense'
                GROUP BY category
            )
            SELECT
                'Revenue' as section,
                category,
                total,
                (total / (SELECT SUM(total) FROM revenue_summary)) * 100 as percentage
            FROM revenue_summary

            UNION ALL

            SELECT
                'Expenses' as section,
                category,
                total,
                (total / (SELECT SUM(total) FROM expense_summary)) * 100 as percentage
            FROM expense_summary

            UNION ALL

            SELECT
                'Net Income' as section,
                'Total' as category,
                (SELECT SUM(total) FROM revenue_summary) -
                (SELECT SUM(total) FROM expense_summary) as total,
                100.0 as percentage

            ORDER BY
                CASE section
                    WHEN 'Revenue' THEN 1
                    WHEN 'Expenses' THEN 2
                    WHEN 'Net Income' THEN 3
                END,
                total DESC
        ", $params);
    }
}
```

## Performance & Optimization Strategies

### Index-Aware Query Design

```php
// Leveraging composite indexes effectively
DB::table('orders')
    ->selectRaw('
        customer_id,
        COUNT(*) as order_count,
        SUM(total) as total_spent,
        MAX(created_at) as last_order_date
    ')
    // Ensure index: (status, created_at, customer_id)
    ->where('status', 'completed')
    ->whereBetween('created_at', [$startDate, $endDate])
    ->groupBy('customer_id')
    ->havingRaw('COUNT(*) >= ?', [3])
    ->orderByDesc('total_spent')
    ->get();

// Using covering indexes
DB::table('products')
    ->selectRaw('id, name, price') // All columns in covering index
    ->where('category_id', 1)      // Index: (category_id, status, id, name, price)
    ->where('status', 'active')
    ->orderBy('id')
    ->get();
```

### Parameter Binding for Safety & Performance

```php
// Proper parameter binding
$userIds = [1, 2, 3, 4, 5];
$placeholders = str_repeat('?,', count($userIds) - 1) . '?';

DB::select("
    SELECT
        u.id, u.name,
        COUNT(o.id) as order_count,
        COALESCE(SUM(o.total), 0) as total_spent
    FROM users u
    LEFT JOIN orders o ON u.id = o.customer_id
        AND o.status = 'completed'
        AND o.created_at >= ?
    WHERE u.id IN ({$placeholders})
    GROUP BY u.id, u.name
    ORDER BY total_spent DESC
", array_merge([now()->subYear()], $userIds));

// Named bindings for complex queries
DB::select("
    SELECT *
    FROM products p
    WHERE p.price BETWEEN :min_price AND :max_price
      AND p.category_id = :category_id
      AND p.created_at >= :start_date
      AND (:brand_id IS NULL OR p.brand_id = :brand_id)
", [
    'min_price' => $filters['min_price'],
    'max_price' => $filters['max_price'],
    'category_id' => $filters['category_id'],
    'start_date' => $filters['start_date'],
    'brand_id' => $filters['brand_id'] ?? null,
]);
```

### Chunking Large Result Sets

```php
// Memory-efficient processing of large datasets
DB::table('transactions')
    ->selectRaw('
        account_id,
        DATE(created_at) as transaction_date,
        SUM(amount) as daily_total
    ')
    ->whereRaw('created_at >= DATE_SUB(CURDATE(), INTERVAL 1 YEAR)')
    ->groupByRaw('account_id, DATE(created_at)')
    ->orderBy('account_id')
    ->orderBy('transaction_date')
    ->chunk(1000, function ($transactions) {
        foreach ($transactions as $transaction) {
            // Process each chunk without memory overflow
            $this->processTransactionSummary($transaction);
        }
    });
```

### Query Result Caching

```php
// Strategic caching of expensive raw queries
public function getTopProducts($categoryId, $days = 30)
{
    $cacheKey = "top_products_{$categoryId}_{$days}";

    return Cache::remember($cacheKey, now()->addHours(2), function () use ($categoryId, $days) {
        return DB::select("
            SELECT
                p.id,
                p.name,
                SUM(oi.quantity) as units_sold,
                SUM(oi.quantity * oi.price) as revenue,
                AVG(r.rating) as avg_rating,
                COUNT(DISTINCT o.customer_id) as unique_buyers
            FROM products p
            INNER JOIN order_items oi ON p.id = oi.product_id
            INNER JOIN orders o ON oi.order_id = o.id
            LEFT JOIN reviews r ON p.id = r.product_id
            WHERE p.category_id = ?
              AND o.status = 'completed'
              AND o.created_at >= DATE_SUB(NOW(), INTERVAL ? DAY)
            GROUP BY p.id, p.name
            HAVING units_sold >= 10
            ORDER BY revenue DESC, avg_rating DESC
            LIMIT 20
        ", [$categoryId, $days]);
    });
}
```

## Security Best Practices

### SQL Injection Prevention

```php
// NEVER do this - vulnerable to SQL injection
public function badSearch($query)
{
    return DB::select("SELECT * FROM products WHERE name LIKE '%{$query}%'");
}

// Always use parameter binding
public function safeSearch($query)
{
    return DB::select(
        "SELECT * FROM products WHERE name LIKE ? OR description LIKE ?",
        ["%{$query}%", "%{$query}%"]
    );
}

// For dynamic WHERE clauses, validate inputs
public function dynamicFilter($filters)
{
    $allowedColumns = ['name', 'price', 'category_id', 'status'];
    $allowedOperators = ['=', '!=', '<', '>', '<=', '>=', 'LIKE'];

    $query = DB::table('products');

    foreach ($filters as $filter) {
        if (!in_array($filter['column'], $allowedColumns) ||
            !in_array($filter['operator'], $allowedOperators)) {
            throw new InvalidArgumentException('Invalid filter criteria');
        }

        $query->whereRaw(
            "{$filter['column']} {$filter['operator']} ?",
            [$filter['value']]
        );
    }

    return $query->get();
}
```

### Input Validation & Sanitization

```php
class SecureQueryBuilder
{
    private array $allowedSortColumns = ['name', 'created_at', 'price', 'rating'];
    private array $allowedSortDirections = ['ASC', 'DESC'];

    public function getProducts($sortBy = 'name', $sortDir = 'ASC', $limit = 20)
    {
        // Validate inputs
        if (!in_array($sortBy, $this->allowedSortColumns)) {
            throw new InvalidArgumentException('Invalid sort column');
        }

        if (!in_array(strtoupper($sortDir), $this->allowedSortDirections)) {
            throw new InvalidArgumentException('Invalid sort direction');
        }

        $limit = min(100, max(1, (int) $limit)); // Clamp between 1-100

        return DB::table('products')
            ->selectRaw('id, name, price, rating, created_at')
            ->whereRaw('status = ? AND deleted_at IS NULL', ['active'])
            ->orderByRaw("{$sortBy} {$sortDir}")
            ->limit($limit)
            ->get();
    }

    public function searchProducts($query, $filters = [])
    {
        // Sanitize search query
        $sanitizedQuery = strip_tags(trim($query));

        if (strlen($sanitizedQuery) < 3) {
            throw new InvalidArgumentException('Search query too short');
        }

        $sql = "
            SELECT p.*,
                   MATCH(p.name, p.description) AGAINST(? IN NATURAL LANGUAGE MODE) as relevance
            FROM products p
            WHERE MATCH(p.name, p.description) AGAINST(? IN NATURAL LANGUAGE MODE)
              AND p.status = 'active'
        ";

        $params = [$sanitizedQuery, $sanitizedQuery];

        // Add optional filters with validation
        if (isset($filters['category_id']) && is_numeric($filters['category_id'])) {
            $sql .= " AND p.category_id = ?";
            $params[] = (int) $filters['category_id'];
        }

        if (isset($filters['min_price']) && is_numeric($filters['min_price'])) {
            $sql .= " AND p.price >= ?";
            $params[] = (float) $filters['min_price'];
        }

        $sql .= " ORDER BY relevance DESC, p.created_at DESC LIMIT ?";
        $params[] = 50; // Hard limit

        return DB::select($sql, $params);
    }
}
```

## Maintainability & Code Organization

### Query Builder Macros

```php
// AppServiceProvider.php
use Illuminate\Database\Query\Builder;

public function boot()
{
    Builder::macro('whereDateRange', function ($column, $start, $end) {
        return $this->whereRaw(
            "DATE({$column}) BETWEEN ? AND ?",
            [$start, $end]
        );
    });

    Builder::macro('selectWithRank', function ($column, $partitionBy = null) {
        $partition = $partitionBy ? "PARTITION BY {$partitionBy}" : '';
        return $this->selectRaw(
            "*, RANK() OVER ({$partition} ORDER BY {$column} DESC) as ranking"
        );
    });

    Builder::macro('withDistanceFrom', function ($lat, $lng, $alias = 'distance_km') {
        return $this->selectRaw("
            *,
            ST_Distance_Sphere(
                POINT(longitude, latitude),
                POINT(?, ?)
            ) / 1000 as {$alias}
        ", [$lng, $lat]);
    });
}

// Usage
DB::table('orders')
    ->whereDateRange('created_at', '2024-01-01', '2024-12-31')
    ->selectWithRank('total', 'customer_id')
    ->get();
```

### Reusable Query Classes

```php
abstract class BaseReportQuery
{
    protected $startDate;
    protected $endDate;

    public function __construct($startDate, $endDate)
    {
        $this->startDate = $startDate;
        $this->endDate = $endDate;
    }

    abstract public function execute();

    protected function getBaseDateFilter(): string
    {
        return 'created_at BETWEEN ? AND ?';
    }

    protected function getDateParams(): array
    {
        return [$this->startDate, $this->endDate];
    }
}

class RevenueReportQuery extends BaseReportQuery
{
    private $groupBy;

    public function __construct($startDate, $endDate, $groupBy = 'day')
    {
        parent::__construct($startDate, $endDate);
        $this->groupBy = $groupBy;
    }

    public function execute()
    {
        $dateFormat = $this->getDateFormat();

        return DB::select("
            SELECT
                {$dateFormat} as period,
                COUNT(DISTINCT id) as order_count,
                COUNT(DISTINCT customer_id) as unique_customers,
                SUM(CASE WHEN status = 'completed' THEN total ELSE 0 END) as revenue,
                AVG(CASE WHEN status = 'completed' THEN total ELSE NULL END) as avg_order_value
            FROM orders
            WHERE {$this->getBaseDateFilter()}
            GROUP BY period
            ORDER BY period
        ", $this->getDateParams());
    }

    private function getDateFormat(): string
    {
        return match ($this->groupBy) {
            'hour' => 'DATE_FORMAT(created_at, "%Y-%m-%d %H:00:00")',
            'day' => 'DATE(created_at)',
            'week' => 'DATE_FORMAT(created_at, "%Y-%u")',
            'month' => 'DATE_FORMAT(created_at, "%Y-%m")',
            'year' => 'YEAR(created_at)',
            default => 'DATE(created_at)',
        };
    }
}
```

### Hybrid Eloquent + Raw SQL

```php
class AdvancedUserRepository
{
    public function getUsersWithComplexMetrics($filters = [])
    {
        // Start with Eloquent for basic filtering
        $query = User::query()
            ->when($filters['active_only'] ?? false, fn($q) => $q->where('status', 'active'))
            ->when($filters['verified_only'] ?? false, fn($q) => $q->whereNotNull('email_verified_at'));

        // Add complex raw selections
        $query->selectRaw('
            users.*,
            (
                SELECT COUNT(*) FROM orders
                WHERE customer_id = users.id
                AND status = "completed"
                AND created_at >= DATE_SUB(NOW(), INTERVAL 90 DAY)
            ) as recent_orders,
            (
                SELECT COALESCE(SUM(total), 0) FROM orders
                WHERE customer_id = users.id
                AND status = "completed"
            ) as lifetime_value,
            (
                SELECT AVG(rating) FROM reviews
                WHERE user_id = users.id
            ) as avg_review_rating
        ');

        // Add complex WHERE clauses
        if ($filters['min_lifetime_value'] ?? null) {
            $query->havingRaw('lifetime_value >= ?', [$filters['min_lifetime_value']]);
        }

        if ($filters['has_recent_activity'] ?? false) {
            $query->havingRaw('recent_orders > 0');
        }

        return $query->orderByRaw('lifetime_value DESC, recent_orders DESC')
            ->paginate($filters['per_page'] ?? 20);
    }

    public function getTopCustomersByCategory($categoryId, $limit = 10)
    {
        // Complex query that would be difficult with pure Eloquent
        $results = DB::select("
            WITH customer_category_stats AS (
                SELECT
                    u.id, u.name, u.email,
                    COUNT(DISTINCT o.id) as orders_in_category,
                    SUM(oi.quantity * oi.price) as spent_in_category,
                    AVG(oi.quantity * oi.price) as avg_order_value,
                    MAX(o.created_at) as last_order_date
                FROM users u
                INNER JOIN orders o ON u.id = o.customer_id
                INNER JOIN order_items oi ON o.id = oi.order_id
                INNER JOIN products p ON oi.product_id = p.id
                WHERE p.category_id = ?
                  AND o.status = 'completed'
                  AND o.created_at >= DATE_SUB(NOW(), INTERVAL 1 YEAR)
                GROUP BY u.id, u.name, u.email
            )
            SELECT *,
                   RANK() OVER (ORDER BY spent_in_category DESC) as spending_rank
            FROM customer_category_stats
            WHERE orders_in_category >= 2
            ORDER BY spent_in_category DESC
            LIMIT ?
        ", [$categoryId, $limit]);

        // Hydrate as User models for consistent API
        return User::hydrate($results);
    }
}
```

## Common Anti-Patterns & Pitfalls

### Anti-Pattern: Over-Using Raw SQL

```php
// BAD: Using raw SQL for simple operations
DB::selectRaw('SELECT * FROM users WHERE id = ?', [1]);

// GOOD: Use Eloquent for simple operations
User::find(1);

// BAD: Raw SQL for basic relationships
DB::select('
    SELECT u.*, p.title
    FROM users u
    LEFT JOIN posts p ON u.id = p.user_id
    WHERE u.id = ?
', [1]);

// GOOD: Use Eloquent relationships
User::with('posts')->find(1);
```

### Anti-Pattern: N+1 in Raw Queries

```php
// BAD: Causing N+1 queries
$orders = DB::select('SELECT * FROM orders WHERE status = ?', ['completed']);
foreach ($orders as $order) {
    // This creates N+1 queries!
    $customer = DB::selectOne('SELECT * FROM users WHERE id = ?', [$order->customer_id]);
}

// GOOD: Use JOINs or subqueries
$orders = DB::select('
    SELECT
        o.*,
        u.name as customer_name,
        u.email as customer_email
    FROM orders o
    INNER JOIN users u ON o.customer_id = u.id
    WHERE o.status = ?
', ['completed']);
```

### Anti-Pattern: Ignoring SQL Injection Vectors

```php
// BAD: Dynamic ORDER BY without validation
public function getProducts($sortBy, $direction)
{
    return DB::select("
        SELECT * FROM products
        ORDER BY {$sortBy} {$direction}
    ");
}

// GOOD: Validate against whitelist
public function getProducts($sortBy, $direction)
{
    $allowedSort = ['name', 'price', 'created_at'];
    $allowedDir = ['ASC', 'DESC'];

    if (!in_array($sortBy, $allowedSort) || !in_array($direction, $allowedDir)) {
        throw new InvalidArgumentException('Invalid sort parameters');
    }

    return DB::select("
        SELECT * FROM products
        ORDER BY {$sortBy} {$direction}
    ");
}
```

### Anti-Pattern: Inefficient Subqueries

```php
// BAD: Correlated subquery in SELECT
DB::select('
    SELECT *,
        (SELECT COUNT(*) FROM orders WHERE customer_id = users.id) as order_count,
        (SELECT SUM(total) FROM orders WHERE customer_id = users.id) as total_spent
    FROM users
');

// GOOD: Use JOINs with aggregation
DB::select('
    SELECT u.*,
        COALESCE(o.order_count, 0) as order_count,
        COALESCE(o.total_spent, 0) as total_spent
    FROM users u
    LEFT JOIN (
        SELECT customer_id,
               COUNT(*) as order_count,
               SUM(total) as total_spent
        FROM orders
        GROUP BY customer_id
    ) o ON u.id = o.customer_id
');
```

## Senior Developer Tricks & Hacks

### Query Performance Debugging

```php
trait QueryPerformanceAnalysis
{
    public function analyzeQuery($sql, $bindings = [])
    {
        // Enable query logging temporarily
        DB::enableQueryLog();

        $startTime = microtime(true);
        $result = DB::select($sql, $bindings);
        $executionTime = (microtime(true) - $startTime) * 1000;

        // Get explain plan
        $explainSql = "EXPLAIN FORMAT=JSON " . $sql;
        $explainResult = DB::select($explainSql, $bindings);

        $analysis = [
            'execution_time_ms' => round($executionTime, 2),
            'rows_returned' => count($result),
            'explain_plan' => json_decode($explainResult[0]->EXPLAIN ?? '{}', true),
            'query_log' => DB::getQueryLog(),
        ];

        DB::disableQueryLog();

        return $analysis;
    }

    public function optimizationSuggestions($analysis)
    {
        $suggestions = [];

        if ($analysis['execution_time_ms'] > 1000) {
            $suggestions[] = 'Query taking >1s - consider adding indexes or breaking into smaller queries';
        }

        if ($analysis['rows_returned'] > 10000) {
            $suggestions[] = 'Large result set - consider pagination or chunking';
        }

        // Analyze explain plan for missing indexes
        $explainText = json_encode($analysis['explain_plan']);
        if (str_contains($explainText, '"key": null') || str_contains($explainText, 'Using filesort')) {
            $suggestions[] = 'Missing indexes detected - review WHERE and ORDER BY clauses';
        }

        return $suggestions;
    }
}
```

### Dynamic Query Builder Pattern

```php
class DynamicReportBuilder
{
    private array $selects = [];
    private array $wheres = [];
    private array $joins = [];
    private array $groups = [];
    private array $orders = [];
    private array $bindings = [];

    public function addMetric(string $alias, string $expression, array $bindings = [])
    {
        $this->selects[] = "{$expression} as {$alias}";
        $this->bindings = array_merge($this->bindings, $bindings);
        return $this;
    }

    public function addDimension(string $alias, string $expression)
    {
        $this->selects[] = "{$expression} as {$alias}";
        $this->groups[] = $expression;
        return $this;
    }

    public function addFilter(string $condition, array $bindings = [])
    {
        $this->wheres[] = $condition;
        $this->bindings = array_merge($this->bindings, $bindings);
        return $this;
    }

    public function addJoin(string $join)
    {
        $this->joins[] = $join;
        return $this;
    }

    public function build(): string
    {
        $sql = 'SELECT ' . implode(', ', $this->selects);
        $sql .= ' FROM orders o';

        if (!empty($this->joins)) {
            $sql .= ' ' . implode(' ', $this->joins);
        }

        if (!empty($this->wheres)) {
            $sql .= ' WHERE ' . implode(' AND ', $this->wheres);
        }

        if (!empty($this->groups)) {
            $sql .= ' GROUP BY ' . implode(', ', $this->groups);
        }

        if (!empty($this->orders)) {
            $sql .= ' ORDER BY ' . implode(', ', $this->orders);
        }

        return $sql;
    }

    public function execute()
    {
        return DB::select($this->build(), $this->bindings);
    }
}

// Usage
$report = (new DynamicReportBuilder())
    ->addDimension('month', 'DATE_FORMAT(o.created_at, "%Y-%m")')
    ->addMetric('total_revenue', 'SUM(CASE WHEN o.status = ? THEN o.total ELSE 0 END)', ['completed'])
    ->addMetric('order_count', 'COUNT(*)')
    ->addMetric('avg_order_value', 'AVG(CASE WHEN o.status = ? THEN o.total ELSE NULL END)', ['completed'])
    ->addFilter('o.created_at >= ?', [now()->subYear()])
    ->addJoin('LEFT JOIN users u ON o.customer_id = u.id')
    ->execute();
```

### Advanced Caching Strategies

```php
class SmartQueryCache
{
    public function getCachedOrExecute(string $cacheKey, string $sql, array $bindings, int $ttl = 3600)
    {
        // Create cache key including query hash
        $fullKey = $cacheKey . ':' . hash('md5', $sql . serialize($bindings));

        return Cache::remember($fullKey, $ttl, function () use ($sql, $bindings) {
            return DB::select($sql, $bindings);
        });
    }

    public function invalidatePattern(string $pattern)
    {
        // Redis-specific cache invalidation
        if (Cache::getStore() instanceof \Illuminate\Cache\RedisStore) {
            $redis = Cache::getStore()->connection();
            $keys = $redis->keys($pattern);
            if (!empty($keys)) {
                $redis->del($keys);
            }
        }
    }

    public function warmCache(array $queries)
    {
        foreach ($queries as $config) {
            $this->getCachedOrExecute(
                $config['key'],
                $config['sql'],
                $config['bindings'] ?? [],
                $config['ttl'] ?? 3600
            );
        }
    }
}
```

### Query Result Transformation

```php
class QueryResultTransformer
{
    public static function toTree(array $results, string $idField = 'id', string $parentField = 'parent_id'): array
    {
        $tree = [];
        $lookup = [];

        // Create lookup array
        foreach ($results as $item) {
            $lookup[$item->{$idField}] = (object) array_merge(
                (array) $item,
                ['children' => []]
            );
        }

        // Build tree structure
        foreach ($lookup as $item) {
            if ($item->{$parentField} === null) {
                $tree[] = $item;
            } else {
                $lookup[$item->{$parentField}]->children[] = $item;
            }
        }

        return $tree;
    }

    public static function pivot(array $results, string $groupBy, string $pivotColumn, string $valueColumn): array
    {
        $pivoted = [];

        foreach ($results as $row) {
            $groupKey = $row->{$groupBy};
            $pivotKey = $row->{$pivotColumn};
            $value = $row->{$valueColumn};

            if (!isset($pivoted[$groupKey])) {
                $pivoted[$groupKey] = [$groupBy => $groupKey];
            }

            $pivoted[$groupKey][$pivotKey] = $value;
        }

        return array_values($pivoted);
    }

    public static function addCalculatedFields(array $results, array $calculations): array
    {
        return array_map(function ($row) use ($calculations) {
            $item = (array) $row;

            foreach ($calculations as $field => $callback) {
                $item[$field] = $callback($row);
            }

            return (object) $item;
        }, $results);
    }
}

// Usage examples
$hierarchical = QueryResultTransformer::toTree($categories);
$pivoted = QueryResultTransformer::pivot($salesData, 'month', 'product', 'revenue');
$enhanced = QueryResultTransformer::addCalculatedFields($orders, [
    'profit_margin' => fn($order) => ($order->revenue - $order->cost) / $order->revenue * 100,
    'days_since_order' => fn($order) => now()->diffInDays($order->created_at),
]);
```

---

**Pro Tips for Senior Laravel Developers:**

1. **Always profile before optimizing** - Use `DB::enableQueryLog()` and Laravel Debugbar
2. **Favor readability over cleverness** - Complex raw SQL should be well-documented
3. **Use database views for complex reporting** - Move complex logic to the database layer
4. **Consider stored procedures for heavy computations** - Especially for financial calculations
5. **Monitor query performance in production** - Set up alerts for slow queries
6. **Use query result caching strategically** - Cache expensive aggregations, not transactional data
7. **Always validate dynamic SQL inputs** - Never trust user input in raw queries
8. **Leverage database-specific features when beneficial** - Don't stick to lowest common denominator
9. **Use transactions wisely** - Especially with complex multi-table raw operations
10. **Keep raw SQL organized** - Use query builder classes, not inline strings everywhere
