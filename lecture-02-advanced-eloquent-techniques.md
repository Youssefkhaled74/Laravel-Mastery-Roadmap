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
Eloquent provides powerful relationship types that can handle complex database structures. Understanding these relationships is crucial for building scalable and maintainable applications.

#### Many-to-Many Relationship (with Pivot Data)
A many-to-many relationship is used when a record in one table can relate to multiple records in another table, and vice versa. For example, products and categories.

> 🗣️ بالمصري:
> يعني المنتج ممكن يكون في كذا كاتيجوري، والكاتيجوري فيها كذا منتج.

```php
// Product Model
class Product extends Model {
    public function categories() {
        return $this->belongsToMany(Category::class)
            ->withTimestamps()
            ->withPivot('featured', 'order');
    }
}

// Category Model
class Category extends Model {
    public function products() {
        return $this->belongsToMany(Product::class)
            ->withTimestamps();
    }
}
```

**Explanation:**
- `belongsToMany` defines the many-to-many relationship.
- `withTimestamps()` automatically manages `created_at` and `updated_at` on the pivot table.
- `withPivot('featured', 'order')` allows you to access extra columns on the pivot table.

> 🗣️ بالمصري:
> الكود ده بيخلي كل منتج يقدر يكون ليه كذا كاتيجوري، وكمان نقدر نضيف بيانات زيادة زي لو المنتج مميز في الكاتيجوري دي.

**Usage Example:**
```php
$product = Product::find(1);
foreach ($product->categories as $category) {
    echo $category->pivot->featured ? 'Featured' : 'Normal';
}
```

#### Has One Through Relationship
This relationship is useful when you want to access a model through another model. For example, a Mechanic has one CarOwner through a Car.

```php
class Mechanic extends Model {
    public function carOwner() {
        return $this->hasOneThrough(
            Owner::class,
            Car::class,
            'mechanic_id', // Foreign key on cars table
            'car_id',      // Foreign key on owners table
            'id',          // Local key on mechanics table
            'id'           // Local key on cars table
        );
    }
}
```

**Explanation:**
- `hasOneThrough` allows you to access the Owner directly from the Mechanic, even though there is no direct relationship.

> 🗣️ بالمصري:
> يعني الميكانيكي يقدر يوصل لصاحب العربية من غير ما يجيب العربية الاول.

#### Polymorphic Relationships (Expanded Example)
Polymorphic relationships allow a model to belong to more than one other model on a single association. For example, comments can belong to posts, videos, or images.

```php
// Comment Model
class Comment extends Model {
    public function commentable() {
        return $this->morphTo();
    }
}

// Post Model
class Post extends Model {
    public function comments() {
        return $this->morphMany(Comment::class, 'commentable');
    }
}

// Video Model
class Video extends Model {
    public function comments() {
        return $this->morphMany(Comment::class, 'commentable');
    }
}
```

**Usage Example:**
```php
$post = Post::find(1);
foreach ($post->comments as $comment) {
    echo $comment->content;
}

$video = Video::find(1);
foreach ($video->comments as $comment) {
    echo $comment->content;
}
```

**Explanation:**
- `morphTo` and `morphMany` allow comments to be attached to any model.
- The `commentable_type` and `commentable_id` columns in the comments table determine the parent model.

> 🗣️ بالمصري:
> الكومنت ممكن يكون على بوست او فيديو او صورة بنفس الجدول.

### 2. Query Optimization Techniques
Optimizing your queries is essential for application performance, especially as your data grows. Laravel Eloquent provides several tools to help you write efficient queries.

#### Eager Loading (Preventing N+1 Problem)
Eager loading retrieves all related models in a single query, preventing the N+1 query problem.

```php
// Eager Loading Posts and Profile for Users
$users = User::with(['posts' => function ($query) {
    $query->latest()->limit(3);
}, 'profile'])->get();
```

**Step-by-step:**
1. `with(['posts' => ...])` tells Eloquent to load the posts for each user in the same query.
2. The closure allows you to customize the posts query (e.g., get only the latest 3).
3. `profile` is also loaded for each user.

> 🗣️ بالمصري:
> هنا بنجيب كل البوستات والبروفايل لكل يوزر مرة واحدة بدل ما نعمل كذا كويري.

#### Lazy Eager Loading
Sometimes you need to load relationships after the initial query.

```php
$books = Book::all();
$books->load('author.profile', 'author.publications');
```

**Explanation:**
- `load()` loads the specified relationships for an existing collection.

#### Counting Related Models
You can count related models efficiently using `withCount`.

```php
$posts = Post::withCount(['comments', 'likes'])
    ->having('comments_count', '>', 3)
    ->get();
```

**Explanation:**
- `withCount` adds a `comments_count` and `likes_count` attribute to each post.
- You can filter posts based on these counts.

#### Selecting Only Needed Columns
Fetching only the columns you need reduces memory usage and speeds up queries.

```php
$users = User::select('id', 'name', 'email')->get();
```

#### Chunking Large Datasets
For processing large datasets, use chunking to avoid memory issues.

```php
User::chunk(1000, function ($users) {
    foreach ($users as $user) {
        // Process user
    }
});
```

> 🗣️ بالمصري:
> chunk بيقسم الداتا لمجموعات صغيرة عشان ميموري التطبيق متتملاش.

#### Caching Query Results
Caching frequently accessed queries can greatly improve performance.

```php
$products = Cache::remember('featured_products', 3600, function () {
    return Product::where('is_featured', true)->get();
});
```

**Explanation:**
- `Cache::remember` stores the result for 1 hour (3600 seconds).
- The next time, it returns the cached result instead of querying the database.

> 🗣️ بالمصري:
> الكاش بيخلي الكويري اسرع بكتير لو بتتكرر كتير.

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