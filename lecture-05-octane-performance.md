# Lecture 05: Laravel Octane & Performance Optimization

## 🎯 Learning Objectives

By the end of this lecture, you will understand:
- Laravel Octane and its benefits
- Application performance optimization techniques
- Memory management and leak prevention
- Database query optimization
- Server configuration best practices
- Monitoring and profiling tools

> 🗣️ بالمصري:
> احنا هنتعلم:
> - ايه هو Laravel Octane وليه مهم
> - ازاي نخلي التطبيق اسرع
> - ازاي نتحكم في الميموري
> - ازاي نحسن الـ queries بتاعت الداتابيز
> - ازاي نظبط السيرفر
> - ازاي نراقب اداء التطبيق

## 🌟 Key Concepts Overview

### 1. Laravel Octane Basics

> 🗣️ بالمصري:
> Octane بيخلي التطبيق اسرع عن طريق:
> - يحمل الكود مرة واحدة ويفضل في الميموري
> - يشغل اكتر من request في نفس الوقت
> - يستخدم سيرفرات سريعة زي RoadRunner و Swoole

```php
// config/octane.php
return [
    'server' => env('OCTANE_SERVER', 'roadrunner'),
    
    'https' => env('OCTANE_HTTPS', false),
    
    'workers' => env('OCTANE_WORKERS', 2),
    
    'warm' => [
        // Services to pre-load
        \App\Providers\RouteServiceProvider::class,
    ],
    
    'listeners' => [
        'workerStarting' => [
            function () {
                // Initialize on worker start
                cache()->clear();
            },
        ],
    ],
];

// Starting Octane
php artisan octane:start --workers=4 --task-workers=2
```

### 2. Query Optimization

> 🗣️ بالمصري:
> هنتعلم ازاي نخلي الـ queries اسرع:
> - نستخدم الـ indexes صح
> - نقلل عدد الـ queries
> - نجيب البيانات اللي محتاجينها بس
> - نستخدم الـ caching

```php
// Bad Query
$users = User::all(); // Gets all columns
foreach ($users as $user) {
    echo $user->profile->name; // N+1 Problem
}

// Optimized Query
$users = User::select(['id', 'name', 'email'])
    ->with(['profile' => function ($query) {
        $query->select(['user_id', 'name']);
    }])
    ->whereHas('orders')
    ->get();

// Using Database Indexes
Schema::table('users', function (Blueprint $table) {
    $table->index(['email', 'status']);
    $table->index('last_login_at');
});

// Chunk Processing for Large Datasets
User::chunk(1000, function ($users) {
    foreach ($users as $user) {
        ProcessUser::dispatch($user);
    }
});
```

### 3. Memory Management

> 🗣️ بالمصري:
> هنتعلم ازاي نتحكم في الميموري:
> - نمسح الداتا اللي مش محتاجينها
> - نستخدم الـ queues للعمليات الكبيرة
> - نتجنب الـ memory leaks
> - نراقب استهلاك الميموري

```php
class LargeDataProcessor {
    public function process() {
        // Process in chunks to manage memory
        User::chunk(1000, function ($users) {
            foreach ($users as $user) {
                $this->processUser($user);
                
                // Clear model from memory
                unset($user);
            }
            
            // Clear query log
            DB::getQueryLog();
            
            // Garbage collection
            if (function_exists('gc_collect_cycles')) {
                gc_collect_cycles();
            }
        });
    }
}

// Using Generator for Memory Efficiency
public function exportLargeFile() {
    $handle = fopen('large-file.csv', 'w');
    
    User::chunk(1000, function ($users) use ($handle) {
        foreach ($users as $user) {
            fputcsv($handle, [
                $user->id,
                $user->name,
                $user->email
            ]);
        }
    });
    
    fclose($handle);
}
```

### 4. Server Configuration

> 🗣️ بالمصري:
> هنتعلم ازاي نظبط السيرفر:
> - نظبط الـ PHP configuration
> - نستخدم OPcache
> - نظبط Nginx او Apache
> - نعمل load balancing

```ini
; php.ini Optimization
memory_limit = 512M
max_execution_time = 60
opcache.enable=1
opcache.memory_consumption=512
opcache.interned_strings_buffer=64
opcache.max_accelerated_files=32531
opcache.validate_timestamps=0
opcache.save_comments=1
opcache.fast_shutdown=0

; nginx.conf
worker_processes auto;
worker_rlimit_nofile 65535;

events {
    worker_connections 65535;
    multi_accept on;
    use epoll;
}

http {
    keepalive_timeout 65;
    keepalive_requests 100;
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
}
```

## 🛠 Real-World Examples

### Example 1: E-commerce Product Catalog

> 🗣️ بالمصري:
> مثال عملي: كاتالوج منتجات:
> - فيه عدد كبير من المنتجات
> - محتاج يكون سريع في البحث
> - فيه فلترة وسورت
> - بيستخدم caching

```php
class ProductController {
    public function index(Request $request) {
        $cacheKey = "products:{$request->category}:{$request->sort}:{$request->page}";
        
        return cache()->remember($cacheKey, now()->addHours(1), function () use ($request) {
            return Product::select(['id', 'name', 'price', 'thumbnail'])
                ->with(['category:id,name', 'tags:id,name'])
                ->when($request->category, function ($query, $category) {
                    $query->whereHas('category', function ($q) use ($category) {
                        $q->where('slug', $category);
                    });
                })
                ->when($request->sort, function ($query, $sort) {
                    match ($sort) {
                        'price_asc' => $query->orderBy('price'),
                        'price_desc' => $query->orderBy('price', 'desc'),
                        'newest' => $query->latest(),
                        default => $query->orderBy('name')
                    };
                })
                ->paginate(24)
                ->withQueryString();
        });
    }
}
```

### Example 2: Analytics Dashboard

> 🗣️ بالمصري:
> مثال تاني: داشبورد للاحصائيات:
> - بيجمع داتا كتير
> - محتاج يكون real-time
> - فيه عمليات حسابية معقدة
> - بيستخدم caching و queues

```php
class DashboardService {
    public function getStatistics() {
        return cache()->remember('dashboard:stats', now()->addMinutes(5), function () {
            return [
                'sales' => $this->getSalesStats(),
                'users' => $this->getUserStats(),
                'products' => $this->getProductStats(),
            ];
        });
    }
    
    protected function getSalesStats() {
        return DB::table('orders')
            ->select(
                DB::raw('DATE(created_at) as date'),
                DB::raw('COUNT(*) as count'),
                DB::raw('SUM(total) as total')
            )
            ->where('created_at', '>=', now()->subDays(30))
            ->groupBy('date')
            ->orderBy('date')
            ->get()
            ->keyBy('date');
    }
    
    protected function processLargeDataset() {
        $result = collect();
        
        User::with('orders')
            ->lazyById(1000)
            ->each(function ($user) use ($result) {
                $result->push($this->calculateUserMetrics($user));
            });
            
        return $result;
    }
}
```

## 🎓 Interview Questions & Answers

> 🗣️ بالمصري:
> اسئلة مهمة هتتسأل عليها في الشغل:

### Q1: What are the benefits of using Laravel Octane?
**Answer:**
> 🗣️ بالمصري:
> - بيخلي التطبيق اسرع 2x-3x
> - بيحمل الكود مرة واحدة بس
> - بيشغل اكتر من request في نفس الوقت
> - بيقلل استهلاك الميموري

### Q2: How do you handle N+1 query problems?
**Answer:**
1. Use eager loading with `with()`
2. Use lazy eager loading when needed
3. Use select() to limit columns
4. Use database indexes properly

### Q3: What are the best practices for caching in Laravel?
**Answer:**
```php
// Using Cache Tags
Cache::tags(['users', 'profile'])->put('user:1', $user, 3600);

// Using Remember
$value = Cache::remember('key', 3600, function () {
    return DB::table(...)->get();
});

// Cache Invalidation
Cache::tags(['users'])->flush();
```

## 🏆 Best Practices

> 🗣️ بالمصري:
> نصايح مهمة للشغل:

1. **Always Monitor Performance**
> راقب اداء التطبيق باستمرار

2. **Use Proper Indexes**
> استخدم الـ indexes المناسبة في الداتابيز

3. **Cache Wisely**
> استخدم الـ cache بذكاء وحدد وقت صلاحية مناسب

4. **Optimize Assets**
> اضغط الصور والـ CSS والـ JavaScript

## 📚 Additional Resources

- [Laravel Octane Documentation](https://laravel.com/docs/octane)
- [Database Indexing Strategies](https://laravel.com/docs/queries)
- [Laravel Performance Tips](https://laravel.com/docs/deployment)
- [Server Configuration Guide](https://laravel.com/docs/deployment#server-requirements)

> 🗣️ بالمصري:
> لو عايز تتعلم اكتر:
> - اقرا الدوكيومنتيشن بتاع Octane
> - اتعلم ازاي تعمل profiling للتطبيق
> - اتعلم ازاي تظبط السيرفر
</rewritten_file> 