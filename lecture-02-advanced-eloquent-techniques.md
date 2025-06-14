# Lecture 02: Advanced Eloquent Techniques in Laravel

## 🎯 Learning Objectives

By the end of this lecture, you will understand:
- Advanced Eloquent relationships and their practical usage
- Query optimization and performance techniques
- Model events and observers
- Global scopes and local scopes
- Custom query builders
- Polymorphic relationships

> 🗣️ بالمصري:
> احنا هنتعلم حاجات متقدمة في Eloquent:
> - ازاي نربط الجداول ببعض بطرق احترافية
> - ازاي نخلي الكويريز بتاعتنا اسرع واحسن
> - ازاي نعمل اكشنز اوتوماتيك لما نعمل حاجة في الداتابيز
> - ازاي نعمل فلاتر جاهزة للكويريز بتاعتنا
> - ازاي نعمل علاقات معقدة بين الجداول

## 🌟 Key Concepts Overview

### 1. Advanced Relationships
Eloquent provides powerful relationship types that can handle complex database structures.

> 🗣️ بالمصري:
> هنتعلم ازاي نربط الجداول ببعض بطرق محترفة، مش بس one-to-one و one-to-many
> زي مثلاً لما يكون عندك منتج ممكن يكون ليه اكتر من كاتيجوري، والكاتيجوري ممكن يكون فيها اكتر من منتج

```php
// Many-to-Many Relationship Example
class Product extends Model {
    public function categories() {
        return $this->belongsToMany(Category::class)
            ->withTimestamps()
            ->withPivot('featured', 'order');
    }
}

// Has One Through Example
class Mechanic extends Model {
    public function carOwner() {
        return $this->hasOneThrough(
            Owner::class,
            Car::class,
            'mechanic_id', // Foreign key on cars table
            'car_id',      // Foreign key on owners table
            'id',          // Local key on mechanics table
            'id'          // Local key on cars table
        );
    }
}
```

### 2. Query Optimization Techniques

> 🗣️ بالمصري:
> هنتعلم ازاي نخلي الكويريز بتاعتنا اسرع واحسن:
> - ازاي نجيب البيانات اللي احنا محتاجينها بس
> - ازاي نمنع مشكلة N+1 
> - ازاي نعمل caching للكويريز

```php
// Eager Loading to Prevent N+1 Problem
$users = User::with(['posts' => function ($query) {
    $query->latest()->limit(3);
}, 'profile'])->get();

// Lazy Eager Loading
$book->author()->load('profile', 'publications');

// Counting Related Models
$posts = Post::withCount(['comments', 'likes'])
    ->having('comments_count', '>', 3)
    ->get();
```

### 3. Model Events & Observers

> 🗣️ بالمصري:
> هنتعلم ازاي نخلي حاجات تحصل اوتوماتيك:
> - لما نعمل save لحاجة في الداتابيز
> - لما نمسح حاجة
> - لما نعدل حاجة
> زي مثلاً لما حد يعمل اوردر، نبعتله ايميل اوتوماتيك

```php
// Model Events
class Order extends Model {
    protected static function booted() {
        static::created(function ($order) {
            Mail::to($order->user)->send(new OrderConfirmation($order));
        });

        static::updating(function ($order) {
            if ($order->isDirty('status')) {
                event(new OrderStatusChanged($order));
            }
        });
    }
}

// Observer Class
class OrderObserver {
    public function created(Order $order) {
        Cache::tags('orders')->flush();
    }

    public function updated(Order $order) {
        if ($order->status === 'completed') {
            $this->updateInventory($order);
        }
    }
}
```

### 4. Global & Local Scopes

> 🗣️ بالمصري:
> هنتعلم ازاي نعمل فلاتر جاهزة للكويريز:
> - فلاتر بتتطبق على كل الكويريز (Global Scopes)
> - فلاتر بنستخدمها لما نحتاجها بس (Local Scopes)

```php
// Global Scope
class ActiveScope implements Scope {
    public function apply(Builder $builder, Model $model) {
        $builder->where('is_active', true);
    }
}

class Product extends Model {
    protected static function booted() {
        static::addGlobalScope(new ActiveScope);
    }
}

// Local Scope
class User extends Model {
    public function scopePopular($query) {
        return $query->where('followers_count', '>', 1000);
    }

    public function scopeActive($query) {
        return $query->where('last_login_at', '>', now()->subDays(30));
    }
}

// Using Scopes
$popularActiveUsers = User::popular()->active()->get();
```

### 5. Custom Query Builders

> 🗣️ بالمصري:
> هنتعلم ازاي نعمل كويريز خاصة بينا:
> - نضيف methods جديدة للكويريز
> - نعمل فلاتر معقدة
> - نخلي الكود اسهل في الاستخدام

```php
// Custom Query Builder
class PostQueryBuilder extends Builder {
    public function published() {
        return $this->where('status', 'published');
    }

    public function trending() {
        return $this->where('views', '>', 1000)
            ->where('created_at', '>', now()->subDays(7));
    }
}

class Post extends Model {
    public function newEloquentBuilder($query) {
        return new PostQueryBuilder($query);
    }
}

// Usage
$trendingPosts = Post::trending()->get();
```

### 6. Polymorphic Relationships

> 🗣️ بالمصري:
> هنتعلم ازاي نعمل علاقات معقدة:
> - علاقة واحدة تشتغل مع اكتر من موديل
> - زي مثلاً الكومنتات، ممكن تكون على بوست او على صورة او على فيديو

```php
// Polymorphic Relationship
class Comment extends Model {
    public function commentable() {
        return $this->morphTo();
    }
}

class Post extends Model {
    public function comments() {
        return $this->morphMany(Comment::class, 'commentable');
    }
}

class Video extends Model {
    public function comments() {
        return $this->morphMany(Comment::class, 'commentable');
    }
}
```

## 🛠 Real-World Examples

### Example 1: E-commerce Product System

> 🗣️ بالمصري:
> مثال عملي: نظام منتجات لموقع بيع:
> - المنتج ليه كذا كاتيجوري
> - كل منتج ليه صور وتقييمات
> - المنتجات ليها فلاتر كتير

```php
class Product extends Model {
    use HasFactory;

    protected $with = ['mainImage']; // Always eager load main image

    public function categories() {
        return $this->belongsToMany(Category::class);
    }

    public function images() {
        return $this->morphMany(Image::class, 'imageable');
    }

    public function mainImage() {
        return $this->morphOne(Image::class, 'imageable')
            ->where('is_main', true);
    }

    public function reviews() {
        return $this->hasMany(Review::class);
    }

    public function scopeInStock($query) {
        return $query->where('stock_quantity', '>', 0);
    }

    public function scopePriceRange($query, $min, $max) {
        return $query->whereBetween('price', [$min, $max]);
    }
}

// Usage Example
$featuredProducts = Product::with(['categories', 'reviews'])
    ->inStock()
    ->priceRange(100, 500)
    ->whereHas('reviews', function ($query) {
        $query->where('rating', '>=', 4);
    })
    ->paginate(20);
```

### Example 2: Blog System with Tags and Categories

> 🗣️ بالمصري:
> مثال تاني: نظام بلوج كامل:
> - البوستات ليها تاجز وكاتيجوريز
> - كل بوست ليه كومنتات
> - في نظام سيرش متقدم

```php
class Post extends Model {
    protected static function booted() {
        static::addGlobalScope('published', function ($builder) {
            $builder->where('status', 'published');
        });
    }

    public function tags() {
        return $this->belongsToMany(Tag::class);
    }

    public function category() {
        return $this->belongsTo(Category::class);
    }

    public function comments() {
        return $this->morphMany(Comment::class, 'commentable');
    }

    public function scopeSearch($query, $term) {
        return $query->where(function ($q) use ($term) {
            $q->where('title', 'like', "%{$term}%")
                ->orWhere('content', 'like', "%{$term}%")
                ->orWhereHas('tags', function ($q) use ($term) {
                    $q->where('name', 'like', "%{$term}%");
                });
        });
    }
}

// Advanced Usage
$posts = Post::with(['tags', 'category', 'comments.author'])
    ->withCount('comments')
    ->search($searchTerm)
    ->whereHas('category', function ($query) {
        $query->where('slug', 'technology');
    })
    ->latest()
    ->paginate(15);
```

## 🎓 Interview Questions & Answers

> 🗣️ بالمصري:
> اسئلة مهمة هتتسأل عليها في الشغل:

### Q1: What's the difference between eager loading and lazy loading?
**Answer:**
> 🗣️ بالمصري:
> - Eager loading يعني تجيب كل البيانات مرة واحدة (احسن للperformance)
> - Lazy loading يعني تجيب البيانات لما تحتاجها بس (اسهل في الكتابة)

```php
// Eager Loading (Better Performance)
$users = User::with('posts')->get();

// Lazy Loading (N+1 Problem)
$users = User::all();
foreach ($users as $user) {
    $user->posts; // New query for each user
}
```

### Q2: How do you handle soft deletes in Eloquent?
**Answer:**
```php
class User extends Model {
    use SoftDeletes;
    
    // Restore with relations
    public function restore() {
        $this->posts()->restore();
        parent::restore();
    }
}

// Usage
User::withTrashed()->find(1)->restore();
User::onlyTrashed()->restore();
```

### Q3: How can you optimize Eloquent queries for better performance?
**Answer:**
1. Use eager loading to prevent N+1 problems
2. Select only needed columns
3. Use chunking for large datasets
4. Cache frequently accessed queries
5. Use database indexes properly

```php
// Optimized Query Example
$users = User::select('id', 'name', 'email')
    ->with(['posts' => function ($query) {
        $query->select('id', 'user_id', 'title')
            ->latest();
    }])
    ->whereHas('posts', function ($query) {
        $query->where('is_published', true);
    })
    ->remember(60) // Cache for 60 minutes
    ->paginate(20);
```

## 🏆 Best Practices

> 🗣️ بالمصري:
> نصايح مهمة للشغل:

1. **Always Use Eager Loading When Possible**
> يعني دايماً استخدم with() عشان تمنع مشكلة N+1

2. **Keep Models Clean and Use Traits**
> قسم الكود بتاعك على ترايتس عشان يبقى منظم ونضيف

3. **Use Model Events Wisely**
> متستخدمش الايفنتس كتير عشان ميبقاش في side effects مش متوقعة

4. **Cache Query Results When Appropriate**
> استخدم الكاش للكويريز اللي بتتكرر كتير

## 📚 Additional Resources

- [Laravel Eloquent Documentation](https://laravel.com/docs/eloquent)
- [Laravel Query Builder](https://laravel.com/docs/queries)
- [Eloquent Performance Tips](https://laravel.com/docs/eloquent-relationships)
- [Laravel Model Events](https://laravel.com/docs/eloquent#events)

> 🗣️ بالمصري:
> لو عايز تتعلم اكتر، الروابط دي هتفيدك
> ابدأ بالدوكيومنتيشن الاول وبعدين شوف الباقي 