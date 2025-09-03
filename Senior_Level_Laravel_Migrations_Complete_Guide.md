# Laravel Migrations: Senior Developer's Complete Guide

## Architectural Foundation & MVC Integration

### Migration's Role in Domain Architecture

Migrations serve as the **evolutionary backbone** of your domain schema, sitting at the intersection of infrastructure and domain modeling. They're not just database scripts—they're domain evolution artifacts that encode business rule changes into persistent storage structures.

In DDD contexts, migrations translate aggregate boundaries, value object constraints, and domain invariants into relational structures. Each migration represents a bounded context evolution, making schema changes traceable and domain-centric rather than purely technical.

### Model Layer Coupling Patterns

```php
// Anti-pattern: Tight coupling to current model state
Schema::create('orders', function (Blueprint $table) {
    // Referencing Order model during migration creation
    // creates temporal coupling issues
});

// Pattern: Schema-first, model-agnostic migrations
Schema::create('orders', function (Blueprint $table) {
    $table->id();
    $table->morphs('orderable'); // Future-proof polymorphism
    $table->json('metadata')->nullable(); // Extensible attributes
    $table->timestamps();
});
```

The critical insight: **migrations define schema evolution independent of current model implementations**. Your models adapt to schema, not vice versa.

## Advanced Schema Builder Mastery

### Fluent Interface Composition

```php
public function up()
{
    Schema::create('financial_instruments', function (Blueprint $table) {
        $table->id();

        // Composite business keys with strategic indexing
        $table->string('symbol', 10);
        $table->string('exchange', 10);
        $table->date('maturity_date')->nullable();

        // Precision-critical decimal handling
        $table->decimal('nominal_value', 19, 4);
        $table->decimal('current_price', 15, 6);

        // Optimized JSON with generated columns for frequent queries
        $table->json('instrument_metadata');
        $table->string('instrument_type')->virtualAs("json_unquote(json_extract(`instrument_metadata`, '$.type'))");

        // Strategic composite indexing
        $table->unique(['symbol', 'exchange', 'maturity_date'], 'financial_instrument_unique');
        $table->index(['instrument_type', 'current_price'], 'type_price_index');

        $table->timestamps();
    });
}
```

### Advanced Constraint Modeling

```php
// Domain-driven constraint implementation
Schema::table('subscription_plans', function (Blueprint $table) {
    // Business rule: trial periods can't exceed 90 days
    $table->unsignedSmallInteger('trial_days')
          ->default(0)
          ->comment('Trial period in days, max 90 per business rules');

    // Database-level constraint enforcement
    DB::statement('ALTER TABLE subscription_plans ADD CONSTRAINT trial_days_business_rule CHECK (trial_days <= 90)');

    // Enum-like constraints with future extensibility
    $table->string('billing_cycle', 20);
    DB::statement("ALTER TABLE subscription_plans ADD CONSTRAINT valid_billing_cycles CHECK (billing_cycle IN ('monthly', 'quarterly', 'annual', 'lifetime'))");
});
```

## Foreign Key Strategy & Referential Integrity

### Cascade Strategy Planning

```php
public function up()
{
    Schema::create('order_line_items', function (Blueprint $table) {
        $table->id();
        $table->foreignId('order_id');
        $table->foreignId('product_id');

        // Strategic cascade decisions based on business domain
        $table->foreign('order_id')
              ->references('id')
              ->on('orders')
              ->cascadeOnDelete()    // Business rule: line items can't exist without orders
              ->cascadeOnUpdate();

        $table->foreign('product_id')
              ->references('id')
              ->on('products')
              ->restrictOnDelete()   // Business rule: can't delete products with existing orders
              ->cascadeOnUpdate();
    });
}
```

### Soft Delete Referential Patterns

```php
// Complex referential integrity with soft deletes
Schema::create('project_assignments', function (Blueprint $table) {
    $table->id();
    $table->foreignId('user_id');
    $table->foreignId('project_id');
    $table->timestamp('assigned_at');
    $table->timestamps();
    $table->softDeletes();

    // Foreign keys that respect soft delete semantics
    $table->foreign('user_id')
          ->references('id')
          ->on('users')
          ->restrictOnDelete(); // Prevent hard delete if assignments exist

    // Unique constraint that ignores soft-deleted records
    $table->unique(['user_id', 'project_id', 'deleted_at'], 'unique_active_assignment');
});

// Supporting index for soft delete performance
Schema::table('project_assignments', function (Blueprint $table) {
    $table->index(['deleted_at', 'project_id'], 'project_active_assignments');
});
```

## Production-Grade Index Strategy

### Query Pattern-Driven Indexing

```php
public function up()
{
    Schema::create('analytics_events', function (Blueprint $table) {
        $table->id();
        $table->string('user_id', 36)->nullable();
        $table->string('session_id', 64);
        $table->string('event_type', 50);
        $table->json('event_data');
        $table->timestamp('occurred_at', 3); // Microsecond precision
        $table->timestamps();

        // Time-series optimization indexes
        $table->index(['occurred_at', 'event_type'], 'time_event_index');
        $table->index(['user_id', 'occurred_at'], 'user_timeline_index');

        // Session analysis optimization
        $table->index(['session_id', 'occurred_at'], 'session_chronology_index');

        // Covering index for common analytical queries
        $table->index(['event_type', 'occurred_at', 'user_id'], 'analytics_covering_index');
    });

    // Partitioning strategy for high-volume time-series data
    DB::statement("
        ALTER TABLE analytics_events
        PARTITION BY RANGE (UNIX_TIMESTAMP(occurred_at)) (
            PARTITION p_current VALUES LESS THAN (UNIX_TIMESTAMP(DATE_ADD(NOW(), INTERVAL 1 MONTH))),
            PARTITION p_future VALUES LESS THAN MAXVALUE
        )
    ");
}
```

### Conditional Index Patterns

```php
// PostgreSQL partial indexes for selective optimization
public function up()
{
    if (DB::getDriverName() === 'pgsql') {
        DB::statement("
            CREATE INDEX CONCURRENTLY idx_orders_pending
            ON orders (created_at, customer_id)
            WHERE status = 'pending'
        ");
    }

    // MySQL equivalent using functional indexes
    if (DB::getDriverName() === 'mysql') {
        DB::statement("
            CREATE INDEX idx_orders_recent_pending
            ON orders (created_at)
            WHERE status = 'pending' AND created_at >= DATE_SUB(NOW(), INTERVAL 30 DAY)
        ");
    }
}
```

## Zero-Downtime Migration Strategies

### Backwards-Compatible Schema Evolution

```php
// Phase 1: Add new column with default, maintain old column
public function up()
{
    Schema::table('users', function (Blueprint $table) {
        // Add new column structure
        $table->json('preferences_v2')->nullable()->after('preferences');

        // Maintain existing column temporarily for backwards compatibility
        // Old 'preferences' column remains functional
    });

    // Populate new column from existing data
    DB::statement("
        UPDATE users
        SET preferences_v2 = JSON_OBJECT('legacy', preferences)
        WHERE preferences IS NOT NULL
    ");
}

// Phase 2 (separate deployment): Remove old column after application update
public function up()
{
    Schema::table('users', function (Blueprint $table) {
        $table->dropColumn('preferences');
        $table->renameColumn('preferences_v2', 'preferences');
    });
}
```

### Online Index Creation Patterns

```php
public function up()
{
    // PostgreSQL concurrent index creation
    if (DB::getDriverName() === 'pgsql') {
        DB::unprepared('CREATE INDEX CONCURRENTLY idx_orders_customer_status ON orders (customer_id, status)');
        return;
    }

    // MySQL online DDL with algorithm specification
    if (DB::getDriverName() === 'mysql') {
        DB::statement('ALTER TABLE orders ADD INDEX idx_orders_customer_status (customer_id, status) ALGORITHM=INPLACE, LOCK=NONE');
        return;
    }

    // Fallback for other databases
    Schema::table('orders', function (Blueprint $table) {
        $table->index(['customer_id', 'status']);
    });
}
```

### Large Table Modification Strategy

```php
public function up()
{
    // Strategy: Shadow table approach for large table modifications
    $tableName = 'large_user_table';
    $shadowTable = $tableName . '_shadow_' . time();

    // Create shadow table with new structure
    DB::statement("CREATE TABLE {$shadowTable} LIKE {$tableName}");

    // Add new columns to shadow table
    Schema::table($shadowTable, function (Blueprint $table) {
        $table->timestamp('email_verified_at')->nullable();
        $table->string('timezone', 50)->default('UTC');
    });

    // Migrate data in chunks (handle in separate command/job)
    // This migration just sets up the infrastructure

    // Store migration state for monitoring
    DB::table('migration_state')->insert([
        'migration' => '2024_01_15_large_table_migration',
        'shadow_table' => $shadowTable,
        'original_table' => $tableName,
        'status' => 'initialized',
        'created_at' => now(),
    ]);
}

public function down()
{
    // Clean up shadow table
    $state = DB::table('migration_state')
               ->where('migration', '2024_01_15_large_table_migration')
               ->first();

    if ($state && $state->shadow_table) {
        DB::statement("DROP TABLE IF EXISTS {$state->shadow_table}");
    }
}
```

## Advanced Rollback & Recovery Patterns

### Stateful Rollback Handling

```php
public function up()
{
    // Complex transformation that requires state preservation
    Schema::create('user_roles_v2', function (Blueprint $table) {
        $table->id();
        $table->foreignId('user_id');
        $table->string('role_type');
        $table->json('permissions');
        $table->timestamps();
    });

    // Preserve rollback data
    DB::statement("
        INSERT INTO migration_rollback_data (migration, data_type, data_json)
        SELECT
            '2024_01_15_role_restructure' as migration,
            'user_roles_backup' as data_type,
            JSON_ARRAYAGG(JSON_OBJECT('user_id', user_id, 'role', role, 'permissions', permissions)) as data_json
        FROM user_roles
    ");

    // Migrate data with complex transformation
    DB::statement("
        INSERT INTO user_roles_v2 (user_id, role_type, permissions)
        SELECT
            user_id,
            CASE
                WHEN role = 'super_admin' THEN 'administrator'
                WHEN role IN ('editor', 'author') THEN 'content_manager'
                ELSE 'viewer'
            END,
            JSON_OBJECT('legacy_role', role, 'permissions', permissions)
        FROM user_roles
    ");
}

public function down()
{
    // Restore from preserved state
    $rollbackData = DB::table('migration_rollback_data')
                     ->where('migration', '2024_01_15_role_restructure')
                     ->where('data_type', 'user_roles_backup')
                     ->value('data_json');

    if ($rollbackData) {
        // Recreate original structure
        Schema::create('user_roles', function (Blueprint $table) {
            $table->id();
            $table->foreignId('user_id');
            $table->string('role');
            $table->json('permissions');
            $table->timestamps();
        });

        // Restore original data
        foreach (json_decode($rollbackData, true) as $record) {
            DB::table('user_roles')->insert($record);
        }
    }

    Schema::dropIfExists('user_roles_v2');
}
```

### Data Integrity Verification

```php
public function up()
{
    // Pre-migration data verification
    $preCount = DB::table('orders')->count();
    $preSum = DB::table('order_items')->sum('quantity');

    // Perform migration operations
    Schema::table('orders', function (Blueprint $table) {
        $table->decimal('total_items', 10, 2)->default(0);
    });

    // Update calculated field
    DB::statement("
        UPDATE orders o
        SET total_items = (
            SELECT COALESCE(SUM(oi.quantity), 0)
            FROM order_items oi
            WHERE oi.order_id = o.id
        )
    ");

    // Post-migration verification
    $postCount = DB::table('orders')->count();
    $postSum = DB::table('orders')->sum('total_items');

    if ($preCount !== $postCount || $preSum !== $postSum) {
        throw new Exception("Migration data integrity check failed. Pre: {$preCount}/{$preSum}, Post: {$postCount}/{$postSum}");
    }
}
```

## Team Collaboration & Conflict Resolution

### Branch-Aware Migration Naming

```php
// Convention: feature_[feature-name]_[timestamp]_[description]
// 2024_01_15_123456_feature_payment_gateway_add_stripe_webhook_table.php

public function up()
{
    // Include branch context in comments for conflict resolution
    Schema::create('stripe_webhook_events', function (Blueprint $table) {
        $table->comment('Created for feature/payment-gateway branch - handles Stripe webhooks');
        $table->id();
        $table->string('stripe_event_id')->unique();
        $table->string('event_type', 50);
        $table->json('payload');
        $table->boolean('processed')->default(false);
        $table->timestamps();

        // Indexes based on expected query patterns
        $table->index(['processed', 'created_at']);
        $table->index('event_type');
    });
}
```

### Conflict-Resistant Schema Design

```php
public function up()
{
    // Extensible design that minimizes merge conflicts
    Schema::create('feature_flags', function (Blueprint $table) {
        $table->id();
        $table->string('key')->unique();
        $table->json('configuration'); // Flexible structure
        $table->boolean('is_active')->default(true);
        $table->string('environment')->default('production');
        $table->timestamps();

        // Compound index for efficient lookups
        $table->index(['environment', 'is_active', 'key']);
    });

    // Seed with namespace structure to avoid key conflicts
    DB::table('feature_flags')->insert([
        ['key' => 'payments.stripe_webhooks', 'configuration' => json_encode(['enabled' => false])],
        ['key' => 'ui.new_dashboard', 'configuration' => json_encode(['rollout_percentage' => 0])],
    ]);
}
```

## Performance & Production Considerations

### Migration Monitoring & Observability

```php
public function up()
{
    $startTime = microtime(true);
    $migrationName = '2024_01_15_performance_critical_migration';

    try {
        // Log migration start
        Log::info("Starting migration: {$migrationName}", [
            'estimated_duration' => '5-10 minutes',
            'affected_tables' => ['orders', 'order_items', 'customers'],
            'lock_level' => 'table',
        ]);

        // Perform migration with progress tracking
        $this->performMigrationSteps($migrationName);

        $duration = microtime(true) - $startTime;

        // Log successful completion
        Log::info("Completed migration: {$migrationName}", [
            'duration_seconds' => round($duration, 2),
            'peak_memory_mb' => round(memory_get_peak_usage() / 1024 / 1024, 2),
        ]);

    } catch (Exception $e) {
        $duration = microtime(true) - $startTime;

        Log::error("Failed migration: {$migrationName}", [
            'duration_seconds' => round($duration, 2),
            'error' => $e->getMessage(),
            'line' => $e->getLine(),
        ]);

        throw $e;
    }
}

private function performMigrationSteps(string $migrationName)
{
    // Step 1: Large table modification
    Log::info("{$migrationName}: Starting index creation");
    Schema::table('orders', function (Blueprint $table) {
        $table->index(['customer_id', 'status', 'created_at'], 'customer_order_timeline');
    });

    // Step 2: Data migration
    Log::info("{$migrationName}: Starting data migration");
    $batchSize = 10000;
    $processed = 0;

    do {
        $count = DB::table('orders')
                   ->where('migrated_flag', false)
                   ->limit($batchSize)
                   ->update([
                       'normalized_status' => DB::raw("LOWER(TRIM(status))"),
                       'migrated_flag' => true
                   ]);

        $processed += $count;

        if ($count > 0) {
            Log::info("{$migrationName}: Processed {$processed} records");
        }

    } while ($count > 0);
}
```

### Resource Management & Constraints

```php
public function up()
{
    // Memory-efficient large dataset processing
    $this->processInChunks('large_table', 5000, function ($records) {
        foreach ($records as $record) {
            // Transform and update individual records
            DB::table('large_table_normalized')->insert([
                'original_id' => $record->id,
                'processed_data' => $this->transformData($record),
                'created_at' => now(),
            ]);
        }
    });
}

private function processInChunks(string $table, int $chunkSize, callable $callback)
{
    DB::table($table)->chunkById($chunkSize, function ($records) use ($callback) {
        $callback($records);

        // Memory management
        if (memory_get_usage() > 100 * 1024 * 1024) { // 100MB threshold
            gc_collect_cycles();
        }
    });
}
```

## Common Pitfalls & Anti-Patterns

### Temporal Coupling Anti-Patterns

```php
// ❌ ANTI-PATTERN: Model dependency in migration
public function up()
{
    Schema::create('posts', function (Blueprint $table) {
        $table->id();
        $table->string('title');
        $table->text('content');
        $table->timestamps();
    });

    // WRONG: Using Eloquent model in migration
    Post::create([
        'title' => 'Welcome Post',
        'content' => 'Welcome to our platform!',
    ]);
}

// ✅ CORRECT PATTERN: Raw database operations
public function up()
{
    Schema::create('posts', function (Blueprint $table) {
        $table->id();
        $table->string('title');
        $table->text('content');
        $table->timestamps();
    });

    // Correct: Direct DB operations
    DB::table('posts')->insert([
        'title' => 'Welcome Post',
        'content' => 'Welcome to our platform!',
        'created_at' => now(),
        'updated_at' => now(),
    ]);
}
```

### Schema Evolution Pitfalls

```php
// ❌ ANTI-PATTERN: Non-reversible destructive changes
public function up()
{
    Schema::table('users', function (Blueprint $table) {
        $table->dropColumn('old_field'); // Data loss without backup
    });
}

// ✅ CORRECT PATTERN: Safe schema evolution
public function up()
{
    // Backup data before destructive changes
    DB::statement("
        CREATE TABLE users_backup_old_field AS
        SELECT id, old_field
        FROM users
        WHERE old_field IS NOT NULL
    ");

    Schema::table('users', function (Blueprint $table) {
        $table->dropColumn('old_field');
    });
}

public function down()
{
    Schema::table('users', function (Blueprint $table) {
        $table->string('old_field')->nullable();
    });

    // Restore backed up data
    DB::statement("
        UPDATE users u
        JOIN users_backup_old_field b ON u.id = b.id
        SET u.old_field = b.old_field
    ");

    DB::statement("DROP TABLE users_backup_old_field");
}
```

## Application Lifecycle Integration

### CI/CD Pipeline Integration

Migrations integrate into your deployment pipeline as **schema contracts**. They enforce database state consistency across environments and enable confident deployments through:

- **Pre-deployment validation**: Schema compatibility checks
- **Deployment gates**: Migration success as deployment criteria
- **Rollback preparation**: Automatic down migration testing
- **Environment parity**: Identical schema evolution across dev/staging/prod

### Domain Event Integration

```php
public function up()
{
    Schema::create('domain_events', function (Blueprint $table) {
        $table->id();
        $table->string('aggregate_type');
        $table->string('aggregate_id');
        $table->string('event_type');
        $table->json('event_data');
        $table->bigInteger('version');
        $table->timestamp('occurred_at', 3);
        $table->timestamps();

        // Event sourcing optimization indexes
        $table->index(['aggregate_type', 'aggregate_id', 'version']);
        $table->index(['occurred_at', 'event_type']);

        // Ensure event ordering integrity
        $table->unique(['aggregate_type', 'aggregate_id', 'version']);
    });
}
```

This architectural approach positions migrations as **domain evolution artifacts** rather than mere database scripts, ensuring they serve both immediate schema needs and long-term architectural integrity.

## Strategic Migration Philosophy

Senior developers understand that migrations are **architectural documentation in code**. Each migration tells a story of domain evolution, business requirement changes, and technical decisions. They serve as:

- **Audit trails** for business logic changes
- **Communication tools** between team members
- **Rollback safety nets** for production deployments
- **Performance optimization** entry points
- **Compliance documentation** for regulated industries

Master these patterns, and migrations become powerful tools for maintaining system integrity while enabling rapid, confident evolution of your Laravel applications.
