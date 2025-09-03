# Laravel Model Factories: Senior Developer's Advanced Guide

## Architectural Foundation & Design Philosophy

### Factory Pattern in Domain Context

Laravel factories implement the **Factory Method pattern** as **domain object builders**, not merely test data generators. They encapsulate complex object construction logic, abstract creation concerns from business logic, and provide **bounded context-aware** entity instantiation.

In Domain-Driven Design, factories serve as:

- **Aggregate Root constructors** that ensure invariant satisfaction
- **Value Object builders** that encapsulate validation logic
- **Domain scenario generators** that represent realistic business cases
- **Bounded Context bridges** that translate between domains

### MVC Integration Patterns

```php
// Anti-pattern: Factories tightly coupled to database structure
class UserFactory extends Factory
{
    public function definition()
    {
        return [
            'name' => fake()->name(),
            'email' => fake()->email(),
            'password' => Hash::make('password'),
        ];
    }
}

// Pattern: Domain-centric factory design
class UserFactory extends Factory
{
    public function definition()
    {
        return [
            'name' => fake()->name(),
            'email' => fake()->unique()->safeEmail(),
            'email_verified_at' => now(),
            'password' => '$2y$10$92IXUNpkjO0rOQ5byMi.Ye4oKoEa3Ro9llC/.og/at2.uheWG/igi', // password
            'remember_token' => Str::random(10),
        ];
    }

    // Domain-specific states represent business scenarios
    public function unverified(): static
    {
        return $this->state(fn () => ['email_verified_at' => null]);
    }

    public function suspended(): static
    {
        return $this->state(fn () => [
            'suspended_at' => now(),
            'suspension_reason' => 'policy_violation'
        ]);
    }

    // Complex domain scenarios
    public function premiumCustomer(): static
    {
        return $this->state(fn () => [
            'account_tier' => 'premium',
            'billing_cycle' => 'annual',
            'subscription_starts_at' => now()->subMonths(3),
        ]);
    }
}
```

## Advanced State Management & Composition

### Fluent State Chaining

```php
class SubscriptionFactory extends Factory
{
    protected $model = Subscription::class;

    public function definition()
    {
        return [
            'plan_id' => Plan::factory(),
            'status' => 'active',
            'starts_at' => now(),
            'ends_at' => now()->addYear(),
            'trial_ends_at' => now()->addDays(14),
        ];
    }

    // Composable state methods with business logic
    public function trial(): static
    {
        return $this->state(fn () => [
            'status' => 'trialing',
            'trial_ends_at' => now()->addDays(rand(1, 14)),
            'starts_at' => now(),
            'ends_at' => null,
        ]);
    }

    public function expired(): static
    {
        return $this->state(function () {
            $endDate = now()->subDays(rand(1, 90));
            return [
                'status' => 'expired',
                'ends_at' => $endDate,
                'trial_ends_at' => $endDate->copy()->subDays(14),
            ];
        });
    }

    public function cancelled(): static
    {
        return $this->state(fn () => [
            'status' => 'cancelled',
            'cancelled_at' => now()->subDays(rand(1, 30)),
            'cancellation_reason' => fake()->randomElement([
                'user_requested', 'payment_failed', 'policy_violation'
            ]),
        ]);
    }

    // Advanced: State combination with validation
    public function expiredWithGracePeriod(): static
    {
        return $this->expired()->state(function () {
            $gracePeriodEnd = now()->addDays(7);
            return [
                'grace_period_ends_at' => $gracePeriodEnd,
                'status' => 'past_due',
            ];
        });
    }
}

// Usage: Expressive domain scenarios
$expiredSubscriptions = Subscription::factory()
    ->count(50)
    ->expired()
    ->create();

$trialWithUpgrade = Subscription::factory()
    ->trial()
    ->has(Payment::factory()->successful(), 'payments')
    ->create();
```

### Dynamic State Resolution

```php
class OrderFactory extends Factory
{
    protected $model = Order::class;

    public function definition()
    {
        return [
            'customer_id' => Customer::factory(),
            'status' => 'pending',
            'total_amount' => 0, // Will be calculated
            'currency' => 'USD',
            'placed_at' => now(),
        ];
    }

    // Dynamic state based on business rules
    public function withRandomProductMix(): static
    {
        return $this->afterCreating(function (Order $order) {
            $products = Product::inRandomOrder()->limit(rand(1, 5))->get();

            foreach ($products as $product) {
                $quantity = rand(1, 3);
                $order->items()->create([
                    'product_id' => $product->id,
                    'quantity' => $quantity,
                    'unit_price' => $product->price,
                    'total_price' => $product->price * $quantity,
                ]);
            }

            // Recalculate order total
            $order->update(['total_amount' => $order->items()->sum('total_price')]);
        });
    }

    // Context-aware state resolution
    public function forRegion(string $region): static
    {
        $regionConfig = config("business.regions.{$region}");

        return $this->state(fn () => [
            'currency' => $regionConfig['currency'],
            'tax_rate' => $regionConfig['tax_rate'],
            'shipping_cost' => $regionConfig['base_shipping'],
        ]);
    }

    // Seasonal business logic
    public function holidayRush(): static
    {
        return $this->state(function () {
            $holidayStart = Carbon::create(now()->year, 11, 20);
            $holidayEnd = Carbon::create(now()->year, 12, 31);

            return [
                'placed_at' => fake()->dateTimeBetween($holidayStart, $holidayEnd),
                'priority_shipping' => true,
                'gift_message' => fake()->optional(0.7)->sentence(),
            ];
        });
    }
}
```

## Advanced Relationship Orchestration

### Complex Nested Relationships

```php
class ProjectFactory extends Factory
{
    protected $model = Project::class;

    public function definition()
    {
        return [
            'name' => fake()->sentence(3),
            'description' => fake()->paragraph(),
            'status' => 'planning',
            'budget' => fake()->numberBetween(10000, 500000),
            'starts_at' => now()->addDays(rand(1, 30)),
            'ends_at' => now()->addDays(rand(60, 365)),
        ];
    }

    // Orchestrate complex domain scenarios
    public function enterpriseProject(): static
    {
        return $this
            ->state(['budget' => fake()->numberBetween(500000, 2000000)])
            ->has(
                User::factory()
                    ->count(rand(8, 15))
                    ->state(['role' => 'developer']),
                'team'
            )
            ->has(
                User::factory()
                    ->count(2)
                    ->state(['role' => 'project_manager']),
                'managers'
            )
            ->has(
                Task::factory()
                    ->count(rand(50, 200))
                    ->sequence(
                        ['priority' => 'high'],
                        ['priority' => 'medium'],
                        ['priority' => 'low'],
                    ),
                'tasks'
            );
    }

    // Cross-aggregate relationship building
    public function withCompleteWorkflow(): static
    {
        return $this->afterCreating(function (Project $project) {
            // Create project phases
            $phases = collect(['planning', 'development', 'testing', 'deployment'])
                ->map(fn ($phase, $index) => Phase::factory()->create([
                    'project_id' => $project->id,
                    'name' => ucfirst($phase),
                    'order' => $index + 1,
                    'starts_at' => $project->starts_at->addDays($index * 30),
                    'ends_at' => $project->starts_at->addDays(($index + 1) * 30),
                ]));

            // Assign tasks to phases
            $project->tasks->chunk(ceil($project->tasks->count() / 4))
                ->zip($phases)
                ->each(function ($chunk) {
                    [$tasks, $phase] = $chunk;
                    $tasks->each(fn ($task) => $task->update(['phase_id' => $phase->id]));
                });

            // Create realistic progress tracking
            $project->tasks->random(rand(10, 30))->each(function ($task) {
                $task->update(['status' => 'completed', 'completed_at' => now()]);
            });
        });
    }
}
```

### Polymorphic Relationship Factories

```php
class CommentableFactory extends Factory
{
    protected $model = Comment::class;

    public function definition()
    {
        return [
            'content' => fake()->paragraph(),
            'author_id' => User::factory(),
            'created_at' => fake()->dateTimeThisYear(),
        ];
    }

    // Polymorphic relationship builders
    public function onPost(): static
    {
        return $this->for(Post::factory(), 'commentable');
    }

    public function onVideo(): static
    {
        return $this->for(Video::factory(), 'commentable');
    }

    public function onProduct(): static
    {
        return $this->for(Product::factory(), 'commentable');
    }

    // Advanced: Context-aware polymorphic creation
    public function forContentType(string $type): static
    {
        $factoryMap = [
            'post' => Post::factory(),
            'video' => Video::factory(),
            'product' => Product::factory(),
        ];

        return $this->for($factoryMap[$type], 'commentable');
    }

    // Nested polymorphic scenarios
    public function threadedDiscussion(): static
    {
        return $this->afterCreating(function (Comment $comment) {
            // Create reply chain
            $parent = $comment;
            for ($i = 0; $i < rand(2, 5); $i++) {
                $reply = Comment::factory()->create([
                    'commentable_type' => $parent->commentable_type,
                    'commentable_id' => $parent->commentable_id,
                    'parent_id' => $parent->id,
                ]);
                $parent = $reply;
            }
        });
    }
}
```

## Sequence Mastery & Advanced Patterns

### Complex Sequence Orchestration

```php
class TransactionFactory extends Factory
{
    protected $model = Transaction::class;

    public function definition()
    {
        return [
            'amount' => fake()->randomFloat(2, 10, 1000),
            'type' => 'credit',
            'processed_at' => now(),
            'reference' => fake()->uuid(),
        ];
    }

    // Advanced sequence with state accumulation
    public function accountHistory(): static
    {
        $balance = 0;

        return $this->sequence(function ($sequence) use (&$balance) {
            $amount = fake()->randomFloat(2, 10, 500);
            $isCredit = fake()->boolean(60); // 60% credits, 40% debits

            if ($isCredit) {
                $balance += $amount;
                $type = 'credit';
            } else {
                $balance -= $amount;
                $type = 'debit';
                $amount = -$amount;
            }

            return [
                'amount' => $amount,
                'type' => $type,
                'balance_after' => $balance,
                'processed_at' => now()->subDays($sequence->index * 2),
                'description' => $isCredit ? 'Deposit' : 'Withdrawal',
            ];
        });
    }

    // Time-series data generation
    public function monthlyPattern(): static
    {
        return $this->sequence(function ($sequence) {
            $dayOfMonth = ($sequence->index % 30) + 1;
            $baseDate = now()->startOfMonth();

            // Simulate business patterns
            $multiplier = match (true) {
                $dayOfMonth <= 5 => 1.5,  // Month start higher activity
                $dayOfMonth >= 25 => 1.2, // Month end activity
                in_array($dayOfMonth, [15, 16]) => 2.0, // Mid-month payroll
                default => 1.0,
            };

            return [
                'amount' => fake()->randomFloat(2, 50, 200) * $multiplier,
                'processed_at' => $baseDate->copy()->addDays($dayOfMonth - 1),
                'category' => $this->getCategoryForDay($dayOfMonth),
            ];
        });
    }

    private function getCategoryForDay(int $day): string
    {
        return match (true) {
            $day <= 5 => 'rent_utilities',
            $day <= 15 => 'groceries',
            $day <= 25 => 'entertainment',
            default => 'miscellaneous',
        };
    }
}

// Usage: Generate realistic financial data
$transactions = Transaction::factory()
    ->count(90)
    ->accountHistory()
    ->for($user, 'account')
    ->create();
```

### Conditional Sequence Logic

```php
class EmployeeFactory extends Factory
{
    public function definition()
    {
        return [
            'name' => fake()->name(),
            'department' => fake()->randomElement(['engineering', 'sales', 'marketing']),
            'hire_date' => fake()->dateTimeBetween('-5 years'),
        ];
    }

    // Organizational hierarchy sequence
    public function organizationChart(): static
    {
        $departments = ['engineering', 'sales', 'marketing'];
        $hierarchyLevels = ['junior', 'mid', 'senior', 'lead', 'director'];

        return $this->sequence(function ($sequence) use ($departments, $hierarchyLevels) {
            $deptIndex = $sequence->index % count($departments);
            $department = $departments[$deptIndex];

            // Determine level based on department size simulation
            $levelDistribution = [
                'junior' => 0.4,
                'mid' => 0.3,
                'senior' => 0.2,
                'lead' => 0.08,
                'director' => 0.02,
            ];

            $level = fake()->randomElement(
                array_map(
                    fn($level, $weight) => array_fill(0, (int)($weight * 100), $level),
                    array_keys($levelDistribution),
                    $levelDistribution
                )
            );

            $salaryRange = $this->getSalaryRange($department, $level);

            return [
                'department' => $department,
                'level' => $level,
                'salary' => fake()->numberBetween($salaryRange['min'], $salaryRange['max']),
                'manager_id' => $this->getManagerId($department, $level, $sequence->index),
            ];
        });
    }

    private function getSalaryRange(string $department, string $level): array
    {
        $baseSalaries = [
            'engineering' => ['junior' => [60000, 80000], 'mid' => [80000, 120000], 'senior' => [120000, 160000]],
            'sales' => ['junior' => [45000, 65000], 'mid' => [65000, 95000], 'senior' => [95000, 140000]],
            'marketing' => ['junior' => [50000, 70000], 'mid' => [70000, 100000], 'senior' => [100000, 130000]],
        ];

        return ['min' => $baseSalaries[$department][$level][0], 'max' => $baseSalaries[$department][$level][1]];
    }
}
```

## Performance Optimization Techniques

### Bulk Creation Optimization

```php
class OptimizedDataFactory
{
    // Performance hack: Bypass Eloquent for massive data creation
    public static function createBulkTransactions(int $count, array $attributes = []): void
    {
        $batchSize = 1000;
        $batches = ceil($count / $batchSize);

        for ($batch = 0; $batch < $batches; $batch++) {
            $records = [];
            $currentBatchSize = min($batchSize, $count - ($batch * $batchSize));

            for ($i = 0; $i < $currentBatchSize; $i++) {
                $records[] = array_merge([
                    'id' => fake()->uuid(),
                    'amount' => fake()->randomFloat(2, 10, 1000),
                    'type' => fake()->randomElement(['credit', 'debit']),
                    'created_at' => now(),
                    'updated_at' => now(),
                ], $attributes);
            }

            // Raw insert for performance
            DB::table('transactions')->insert($records);
        }
    }

    // Memory-efficient relationship creation
    public static function attachBulkRelationships(string $model, string $relation, int $count): void
    {
        $modelInstance = new $model();
        $parentIds = $modelInstance::pluck('id')->toArray();
        $relationModel = $modelInstance->{$relation}()->getModel();

        collect($parentIds)->chunk(100)->each(function ($chunk) use ($relationModel, $count) {
            $pivotData = [];

            foreach ($chunk as $parentId) {
                $relatedIds = $relationModel::inRandomOrder()->limit($count)->pluck('id');

                foreach ($relatedIds as $relatedId) {
                    $pivotData[] = [
                        'parent_id' => $parentId,
                        'related_id' => $relatedId,
                        'created_at' => now(),
                    ];
                }
            }

            DB::table('pivot_table_name')->insert($pivotData);
        });
    }
}

// Usage in performance-critical scenarios
class LargeDatasetSeeder extends Seeder
{
    public function run()
    {
        DB::disableQueryLog();

        // Create 100k records efficiently
        OptimizedDataFactory::createBulkTransactions(100000, [
            'account_id' => Account::factory()->create()->id,
        ]);

        // Batch relationship attachment
        OptimizedDataFactory::attachBulkRelationships(User::class, 'roles', 3);

        DB::enableQueryLog();
    }
}
```

### Memory Management Strategies

```php
class MemoryEfficientFactory extends Factory
{
    protected $model = LargeDataModel::class;

    // Lazy relationship loading to prevent memory bloat
    public function withLazyRelationships(): static
    {
        return $this->afterCreating(function (LargeDataModel $model) {
            // Use lazy collections for memory efficiency
            LazyCollection::times(1000, function () use ($model) {
                return RelatedModel::factory()->make(['parent_id' => $model->id]);
            })->chunk(100)->each(function ($chunk) {
                RelatedModel::insert($chunk->toArray());
            });
        });
    }

    // Generator-based factory for infinite sequences
    public function infiniteSequence(): Generator
    {
        while (true) {
            yield $this->make();
        }
    }

    // Memory-conscious bulk operations
    public static function createInChunks(int $total, int $chunkSize = 100): void
    {
        for ($i = 0; $i < $total; $i += $chunkSize) {
            $count = min($chunkSize, $total - $i);

            static::new()->count($count)->create();

            // Force garbage collection after each chunk
            if (memory_get_usage() > 50 * 1024 * 1024) { // 50MB threshold
                gc_collect_cycles();
            }
        }
    }
}
```

## Domain-Driven Factory Architecture

### Bounded Context Factory Organization

```php
// Context: Order Management
namespace App\OrderManagement\Factories;

class OrderFactory extends Factory
{
    protected $model = Order::class;

    public function definition()
    {
        return [
            'customer_id' => Customer::factory(),
            'status' => OrderStatus::PENDING,
            'total_amount' => 0,
            'placed_at' => now(),
        ];
    }

    // Domain scenarios as first-class factory methods
    public function b2bOrder(): static
    {
        return $this
            ->for(Customer::factory()->business(), 'customer')
            ->state([
                'payment_terms' => '30_days',
                'requires_approval' => true,
                'volume_discount_applied' => true,
            ])
            ->has(OrderLineItem::factory()->count(rand(10, 50)), 'lineItems');
    }

    public function rushOrder(): static
    {
        return $this->state([
            'priority' => 'urgent',
            'expedited_shipping' => true,
            'processing_deadline' => now()->addHours(4),
        ]);
    }

    // Complex business scenarios
    public function internationalOrder(): static
    {
        return $this->afterCreating(function (Order $order) {
            // Create customs documentation
            CustomsDeclaration::factory()
                ->for($order)
                ->withRandomItems()
                ->create();

            // Add international shipping requirements
            $order->update([
                'requires_customs_declaration' => true,
                'estimated_delivery' => now()->addDays(rand(7, 21)),
                'shipping_insurance_required' => true,
            ]);
        });
    }
}

// Context: Billing & Payments
namespace App\Billing\Factories;

class InvoiceFactory extends Factory
{
    protected $model = Invoice::class;

    // Domain-specific factory scenarios
    public function recurringSubscription(): static
    {
        return $this->state([
            'type' => 'subscription',
            'billing_cycle' => fake()->randomElement(['monthly', 'quarterly', 'annual']),
            'auto_pay_enabled' => true,
        ])->afterCreating(function (Invoice $invoice) {
            // Create payment schedule
            PaymentSchedule::factory()
                ->for($invoice)
                ->withFuturePayments()
                ->create();
        });
    }

    public function overdueWithDunning(): static
    {
        return $this->state([
            'status' => 'overdue',
            'due_date' => now()->subDays(rand(30, 90)),
        ])->has(
            DunningNotice::factory()->count(rand(2, 4)),
            'dunningNotices'
        );
    }
}
```

### Aggregate Root Factory Patterns

```php
class CustomerAggregateFactory extends Factory
{
    protected $model = Customer::class;

    public function definition()
    {
        return [
            'name' => fake()->company(),
            'type' => 'individual',
            'status' => 'active',
            'registration_date' => fake()->dateTimeBetween('-2 years'),
        ];
    }

    // Build complete aggregate with all entities
    public function completeAggregate(): static
    {
        return $this->afterCreating(function (Customer $customer) {
            // Primary contact
            $customer->contacts()->create([
                'name' => fake()->name(),
                'email' => fake()->safeEmail(),
                'phone' => fake()->phoneNumber(),
                'is_primary' => true,
            ]);

            // Billing address
            $customer->addresses()->create([
                'type' => 'billing',
                'street' => fake()->streetAddress(),
                'city' => fake()->city(),
                'state' => fake()->state(),
                'postal_code' => fake()->postcode(),
                'country' => fake()->countryCode(),
            ]);

            // Payment methods
            $customer->paymentMethods()->create([
                'type' => 'credit_card',
                'last_four' => fake()->numerify('####'),
                'expiry_month' => fake()->numberBetween(1, 12),
                'expiry_year' => fake()->numberBetween(2024, 2030),
                'is_default' => true,
            ]);

            // Customer preferences
            $customer->preferences()->create([
                'communication_channel' => 'email',
                'invoice_format' => 'pdf',
                'auto_pay_enabled' => fake()->boolean(70),
            ]);
        });
    }

    // Domain-specific aggregate scenarios
    public function enterpriseCustomer(): static
    {
        return $this
            ->state(['type' => 'enterprise'])
            ->completeAggregate()
            ->afterCreating(function (Customer $customer) {
                // Enterprise-specific entities
                $customer->contracts()->create([
                    'type' => 'master_service_agreement',
                    'start_date' => now(),
                    'end_date' => now()->addYears(3),
                    'value' => fake()->numberBetween(100000, 1000000),
                ]);

                // Multiple contacts for enterprise
                $customer->contacts()->createMany([
                    ['name' => fake()->name(), 'role' => 'technical_lead'],
                    ['name' => fake()->name(), 'role' => 'procurement'],
                    ['name' => fake()->name(), 'role' => 'finance'],
                ]);
            });
    }
}
```

## Testing Integration Strategies

### Test-Specific Factory Configurations

```php
// Base test case with factory utilities
abstract class TestCase extends BaseTestCase
{
    use CreatesApplication, RefreshDatabase;

    // Factory presets for common test scenarios
    protected function createUserWithProfile(array $attributes = []): User
    {
        return User::factory()
            ->has(Profile::factory(), 'profile')
            ->has(Subscription::factory()->active(), 'subscription')
            ->create($attributes);
    }

    // Context-specific factory helpers
    protected function createCompleteOrder(Customer $customer = null): Order
    {
        return Order::factory()
            ->for($customer ?? Customer::factory(), 'customer')
            ->has(OrderLineItem::factory()->count(rand(1, 5)), 'lineItems')
            ->has(Payment::factory()->successful(), 'payment')
            ->withShipping()
            ->create();
    }

    // Data-driven test scenarios
    protected function createTestScenario(string $scenario): Collection
    {
        return match($scenario) {
            'e-commerce_flow' => collect([
                'customer' => Customer::factory()->create(),
                'products' => Product::factory()->count(10)->create(),
                'cart' => ShoppingCart::factory()->withItems()->create(),
                'order' => Order::factory()->pending()->create(),
            ]),

            'subscription_lifecycle' => collect([
                'plan' => Plan::factory()->monthly()->create(),
                'customer' => Customer::factory()->withPaymentMethod()->create(),
                'subscription' => Subscription::factory()->trial()->create(),
                'invoices' => Invoice::factory()->count(12)->create(),
            ]),

            default => throw new InvalidArgumentException("Unknown scenario: {$scenario}")
        };
    }
}

// Feature test with domain scenarios
class OrderProcessingTest extends TestCase
{
    /** @test */
    public function it_processes_bulk_orders_efficiently()
    {
        // Create realistic bulk order scenario
        $customer = Customer::factory()->enterprise()->create();

        $orders = Order::factory()
            ->count(100)
            ->for($customer)
            ->sequence(
                fn($sequence) => [
                    'priority' => $sequence->index % 10 === 0 ? 'urgent' : 'normal',
                    'total_amount' => fake()->numberBetween(1000, 50000),
                ]
            )
            ->create();

        $processor = new BulkOrderProcessor();

        $result = $processor->processOrders($orders);

        $this->assertEquals(100, $result->processedCount);
        $this->assertEquals(10, $result->urgentCount);
    }

    /** @test */
    public function it_handles_complex_pricing_scenarios()
    {
        // Multi-tier pricing scenario
        $scenario = $this->createTestScenario('e-commerce_flow');

        $order = Order::factory()
            ->for($scenario['customer'])
            ->afterCreating(function (Order $order) use ($scenario) {
                // Add various product types with different pricing rules
                $scenario['products']->each(function ($product, $index) use ($order) {
                    $order->lineItems()->create([
                        'product_id' => $product->id,
                        'quantity' => rand(1, 5),
                        'unit_price' => $product->calculatePriceFor($order->customer),
                        'discount_applied' => $index % 3 === 0,
                    ]);
                });
            })
            ->create();

        $calculator = new OrderTotalCalculator();

        $total = $calculator->calculate($order);

        $this->assertGreaterThan(0, $total->subtotal);
        $this->assertEquals($total->total, $total->subtotal + $total->tax - $total->discount);
    }
}
```

### Mock Integration Patterns

```php
class PaymentServiceTest extends TestCase
{
    /** @test */
    public function it_retries_failed_payments_with_exponential_backoff()
    {
        // Create payment scenario with realistic failure patterns
        $payment = Payment::factory()
            ->for(Order::factory()->create())
            ->failed()
            ->create([
                'failure_reason' => 'temporary_network_error',
                'retry_count' => 0,
                'next_retry_at' => now()->addMinutes(5),
            ]);

        // Mock external payment gateway with realistic responses
        $gateway = $this->mock(PaymentGateway::class);

        $gateway->shouldReceive('processPayment')
            ->times(3)
            ->andReturnUsing(function () use (&$attemptCount) {
                $attemptCount = ($attemptCount ?? 0) + 1;

                // Simulate gradual success pattern
                return match($attemptCount) {
                    1 => new PaymentResult(false, 'network_timeout'),
                    2 => new PaymentResult(false, 'service_unavailable'),
                    3 => new PaymentResult(true, 'success'),
                };
            });

        $service = new PaymentRetryService($gateway);

        $result = $service->retryPayment($payment);

        $this->assertTrue($result->successful);
        $this->assertEquals(3, $payment->fresh()->retry_count);
    }
}
```

## CI/CD Integration & Automation

### Environment-Specific Seeding

```php
class EnvironmentAwareSeeder extends Seeder
{
    public function run()
    {
        match(app()->environment()) {
            'production' => $this->seedProduction(),
            'staging' => $this->seedStaging(),
            'development' => $this->seedDevelopment(),
            'testing' => $this->seedTesting(),
            default => $this->seedLocal(),
        };
    }

    private function seedProduction()
    {
        // Minimal essential data only
        User::factory()
            ->count(1)
            ->admin()
            ->create(['email' => config('app.admin_email')]);

        // System configurations
        Setting::factory()->systemDefaults()->create();
    }

    private function seedStaging()
    {
        // Production-like data volumes but with fake data
        User::factory()->count(1000)->create();
        Order::factory()->count(10000)->withCompleteFlow()->create();

        // Test payment methods with known credentials
        PaymentMethod::factory()
            ->testCredentials()
            ->count(5)
            ->create();
    }

    private function seedDevelopment()
    {
        // Rich development data with edge cases
        $this->seedCommonScenarios();
        $this->seedEdgeCases();
        $this->seedPerformanceTestData();
    }

    private function seedCommonScenarios()
    {
        // Realistic business scenarios developers encounter
        Customer::factory()
            ->count(50)
            ->sequence(
                ['type' => 'individual'],
                ['type' => 'business'],
                ['type' => 'enterprise']
            )
            ->has(Order::factory()->count(rand(1, 20)), 'orders')
            ->create();
    }

    private function seedEdgeCases()
    {
        // Boundary conditions and error scenarios
        Order::factory()->cancelled()->count(10)->create();
        Order::factory()->refunded()->count(5)->create();
        Payment::factory()->failed()->count(15)->create();

        // Unicode and internationalization edge cases
        Customer::factory()
            ->count(10)
            ->state([
                'name' => fake()->randomElement([
                    '测试客户', 'العميل التجريبي', 'テストカスタマー',
                    'Müller & Söhne', 'José María González'
                ])
            ])
            ->create();
    }
}
```

### Automated Factory Validation

```php
class FactoryValidationCommand extends Command
{
    protected $signature = 'factories:validate {--model=*}';
    protected $description = 'Validate factory definitions against model constraints';

    public function handle()
    {
        $models = $this->option('model') ?: $this->discoverModels();

        foreach ($models as $model) {
            $this->validateFactory($model);
        }
    }

    private function validateFactory(string $modelClass)
    {
        $this->info("Validating factory for {$modelClass}");

        try {
            // Test basic factory creation
            $instance = $modelClass::factory()->make();
            $this->checkRequiredFields($instance, $modelClass);

            // Test database constraints
            $created = $modelClass::factory()->create();
            $this->checkDatabaseConstraints($created);

            // Test factory states
            $this->validateFactoryStates($modelClass);

            $this->info("✅ {$modelClass} factory validation passed");

        } catch (Exception $e) {
            $this->error("❌ {$modelClass} factory validation failed: " . $e->getMessage());
        }
    }

    private function checkRequiredFields($instance, string $modelClass)
    {
        $model = new $modelClass();
        $fillable = $model->getFillable();
        $rules = method_exists($model, 'rules') ? $model->rules() : [];

        foreach ($rules as $field => $rule) {
            if (str_contains($rule, 'required') && is_null($instance->$field)) {
                throw new Exception("Required field {$field} is null in factory");
            }
        }
    }

    private function validateFactoryStates(string $modelClass)
    {
        $factory = $modelClass::factory();
        $factoryClass = get_class($factory);

        // Use reflection to find state methods
        $reflection = new ReflectionClass($factoryClass);
        $methods = $reflection->getMethods(ReflectionMethod::IS_PUBLIC);

        foreach ($methods as $method) {
            if ($method->getReturnType()?->getName() === 'static' ||
                str_contains($method->getReturnType()?->getName() ?? '', 'Factory')) {

                $stateName = $method->getName();

                if (!in_array($stateName, ['definition', '__construct'])) {
                    $this->info("  Testing state: {$stateName}");
                    $modelClass::factory()->$stateName()->create();
                }
            }
        }
    }
}
```

## Advanced Tips, Tricks & Hacks

### Factory Composition Patterns

```php
// Trait-based factory mixins for reusable behaviors
trait HasTimestamps
{
    public function withCustomTimestamps(Carbon $createdAt, Carbon $updatedAt = null): static
    {
        return $this->state([
            'created_at' => $createdAt,
            'updated_at' => $updatedAt ?? $createdAt,
        ]);
    }

    public function recent(): static
    {
        return $this->withCustomTimestamps(now()->subHours(rand(1, 24)));
    }

    public function ancient(): static
    {
        return $this->withCustomTimestamps(now()->subYears(rand(2, 10)));
    }
}

trait HasRandomizedContent
{
    public function withRealisticContent(): static
    {
        return $this->state(function () {
            $contentTypes = ['technical', 'marketing', 'casual', 'formal'];
            $type = fake()->randomElement($contentTypes);

            return [
                'content' => $this->generateContentByType($type),
                'content_type' => $type,
                'word_count' => str_word_count($this->generateContentByType($type)),
            ];
        });
    }
}

// Factory with trait composition
class ArticleFactory extends Factory
{
    use HasTimestamps, HasRandomizedContent;

    protected $model = Article::class;

    public function definition()
    {
        return [
            'title' => fake()->sentence(),
            'content' => fake()->paragraphs(5, true),
            'status' => 'draft',
        ];
    }

    // Combine trait behaviors
    public function publishedArticle(): static
    {
        return $this
            ->withRealisticContent()
            ->recent()
            ->state(['status' => 'published']);
    }
}
```

### Dynamic Factory Registration

```php
// Service provider for dynamic factory discovery
class FactoryServiceProvider extends ServiceProvider
{
    public function boot()
    {
        $this->registerDynamicFactories();
    }

    private function registerDynamicFactories()
    {
        // Auto-discover and register factories
        $factoryPath = app_path('Factories');

        if (!is_dir($factoryPath)) {
            return;
        }

        $factoryFiles = File::allFiles($factoryPath);

        foreach ($factoryFiles as $file) {
            $className = $this->getClassNameFromFile($file);

            if (class_exists($className) && is_subclass_of($className, Factory::class)) {
                $this->registerFactoryBinding($className);
            }
        }
    }

    private function registerFactoryBinding(string $factoryClass)
    {
        $factory = new $factoryClass();
        $modelClass = $factory->modelName();

        // Register factory binding
        $modelClass::resolveFactory($factoryClass);

        // Register in container for dependency injection
        $this->app->singleton($factoryClass, fn() => $factory);
    }
}

// Usage: Factories auto-discovered and available everywhere
class SomeService
{
    public function __construct(private UserFactory $userFactory) {}

    public function createTestUser(): User
    {
        return $this->userFactory->withProfile()->create();
    }
}
```

### Factory Caching & Performance

```php
// Factory result caching for expensive operations
class CachedFactory extends Factory
{
    private static array $instanceCache = [];

    public function cached(string $key = null): static
    {
        $key = $key ?? md5(serialize($this->definition()));

        if (!isset(self::$instanceCache[$key])) {
            self::$instanceCache[$key] = $this->create();
        }

        return self::$instanceCache[$key];
    }

    // Singleton-like factory instances for expensive setups
    public function singleton(): static
    {
        static $instance;

        if (!$instance) {
            $instance = $this->create();
        }

        return $instance;
    }
}

// Pre-computed factory results for test performance
class TestDataCache
{
    private static array $precomputedData = [];

    public static function precompute()
    {
        self::$precomputedData = [
            'admin_user' => User::factory()->admin()->create(),
            'test_products' => Product::factory()->count(100)->create(),
            'sample_orders' => Order::factory()->count(50)->withItems()->create(),
        ];
    }

    public static function get(string $key)
    {
        return self::$precomputedData[$key] ?? null;
    }
}
```

### Factory Debugging & Introspection

```php
// Debugging factory with execution tracing
class DebuggableFactory extends Factory
{
    private array $executionTrace = [];

    public function create($attributes = [], ?Model $parent = null)
    {
        $this->trace('create', ['attributes' => $attributes]);

        try {
            $result = parent::create($attributes, $parent);
            $this->trace('created', ['id' => $result->id, 'class' => get_class($result)]);
            return $result;
        } catch (Exception $e) {
            $this->trace('error', ['message' => $e->getMessage()]);
            throw $e;
        }
    }

    private function trace(string $event, array $data = [])
    {
        $this->executionTrace[] = [
            'timestamp' => microtime(true),
            'event' => $event,
            'data' => $data,
            'memory_usage' => memory_get_usage(),
        ];
    }

    public function getExecutionTrace(): array
    {
        return $this->executionTrace;
    }

    // Factory introspection
    public function getStateAnalysis(): array
    {
        return [
            'current_states' => $this->states,
            'applied_callbacks' => count($this->afterMaking) + count($this->afterCreating),
            'relationship_count' => count($this->has),
            'estimated_queries' => $this->estimateQueryCount(),
        ];
    }
}

// Factory performance profiler
class FactoryProfiler
{
    private static array $profiles = [];

    public static function profile(Factory $factory, callable $operation): mixed
    {
        $start = microtime(true);
        $startMemory = memory_get_usage();

        $result = $operation($factory);

        $duration = microtime(true) - $start;
        $memoryUsed = memory_get_usage() - $startMemory;

        self::$profiles[] = [
            'factory' => get_class($factory),
            'duration' => $duration,
            'memory_used' => $memoryUsed,
            'peak_memory' => memory_get_peak_usage(),
            'queries' => DB::getQueryLog(),
        ];

        return $result;
    }

    public static function getProfile(): array
    {
        return self::$profiles;
    }
}
```

## Anti-Patterns & Common Pitfalls

### Temporal Coupling Dangers

```php
// ❌ ANTI-PATTERN: Factory coupled to current application state
class BadUserFactory extends Factory
{
    public function definition()
    {
        // WRONG: Dependent on existing data
        $existingRole = Role::where('name', 'user')->first();

        return [
            'name' => fake()->name(),
            'email' => fake()->email(),
            'role_id' => $existingRole?->id, // Fails if role doesn't exist
        ];
    }
}

// ✅ CORRECT: Self-contained factory
class GoodUserFactory extends Factory
{
    public function definition()
    {
        return [
            'name' => fake()->name(),
            'email' => fake()->unique()->safeEmail(),
            'role_id' => Role::factory(), // Creates role if needed
        ];
    }
}
```

### Performance Anti-Patterns

```php
// ❌ ANTI-PATTERN: N+1 factory queries
$users = User::factory()
    ->count(100)
    ->afterCreating(function ($user) {
        // WRONG: Creates 100 individual queries
        $user->profile()->create(['bio' => fake()->paragraph()]);
    })
    ->create();

// ✅ CORRECT: Relationship factory usage
$users = User::factory()
    ->count(100)
    ->has(Profile::factory(), 'profile') // Single batch operation
    ->create();
```

### State Management Pitfalls

```php
// ❌ ANTI-PATTERN: Stateful factories with side effects
class BadOrderFactory extends Factory
{
    private int $orderCounter = 0; // WRONG: Instance state

    public function definition()
    {
        $this->orderCounter++; // Modifies instance state

        return [
            'order_number' => "ORD-{$this->orderCounter}",
            'total' => fake()->randomFloat(2, 10, 1000),
        ];
    }
}

// ✅ CORRECT: Pure factory functions
class GoodOrderFactory extends Factory
{
    public function definition()
    {
        return [
            'order_number' => 'ORD-' . fake()->unique()->numerify('######'),
            'total' => fake()->randomFloat(2, 10, 1000),
        ];
    }
}
```

## Strategic Factory Philosophy

Senior developers understand that factories are **domain modeling tools**, not just test utilities. They encode business knowledge, capture domain complexity, and serve as **living documentation** of your system's entities and relationships.

**Key Strategic Insights:**

- **Factories as Domain Experts**: Each factory encapsulates deep knowledge about valid entity states and business rules
- **Composition over Inheritance**: Build complex scenarios through factory composition rather than deep inheritance hierarchies
- **Performance as Architecture**: Factory efficiency directly impacts development velocity and CI/CD pipeline speed
- **Maintainability through Expressiveness**: Well-designed factories make tests self-documenting and domain scenarios explicit

**Integration with Application Architecture:**

Factories bridge the gap between your **domain models** and **testing infrastructure**, serving as:

- **Aggregate builders** that respect domain boundaries
- **Scenario orchestrators** that encode business workflows
- **Performance optimization points** for development and CI environments
- **Documentation artifacts** that demonstrate proper entity usage

Master these patterns, and your factories become powerful architectural tools that accelerate development while maintaining domain integrity and system reliability.
