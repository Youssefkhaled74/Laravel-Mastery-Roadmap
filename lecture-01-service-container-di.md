# Lecture 01: Service Container & Dependency Injection in Laravel

## 🎯 Learning Objectives

By the end of this lecture, you will understand:
- The core concepts of IoC (Inversion of Control)
- How Dependency Injection (DI) works in Laravel
- The power of Laravel's Service Container
- Real-world applications of these patterns
- Best practices for enterprise applications

## 🌟 Key Concepts Overview

### 1. IoC (Inversion of Control)
IoC is a design principle where control over the flow of an application is inverted: instead of your code controlling the flow, a framework (like Laravel) controls it. Think of it as "Don't call us, we'll call you."

### 2. DI (Dependency Injection)
DI is a technique where one object supplies the dependencies of another object. Instead of creating dependencies inside the class, they are injected from outside.

### 3. Service Container
Laravel's Service Container is a powerful tool for managing class dependencies and performing dependency injection.

---

## 📚 Detailed Explanation

### Understanding IoC with Real Examples

#### ❌ Without IoC (Traditional Approach):
```php
class OrderProcessor {
    public function __construct() {
        $this->paymentGateway = new StripePayment();
        $this->emailService = new SendGridMailer();
        $this->logger = new FileLogger();
    }
}
```

Problems with this approach:
- Tightly coupled code
- Hard to test
- Hard to change implementations
- Violates Single Responsibility Principle

#### ✅ With IoC (Laravel Way):
```php
class OrderProcessor {
    public function __construct(
        PaymentGatewayInterface $paymentGateway,
        EmailServiceInterface $emailService,
        LoggerInterface $logger
    ) {
        $this->paymentGateway = $paymentGateway;
        $this->emailService = $emailService;
        $this->logger = $logger;
    }
}
```

Benefits:
- Loosely coupled code
- Easy to test with mocks
- Easy to swap implementations
- Follows SOLID principles

---

## 🛠 Real-World Examples

### Example 1: E-commerce Payment System

```php
// Interface
interface PaymentGatewayInterface {
    public function process(Order $order): PaymentResult;
    public function refund(string $transactionId): bool;
}

// Implementation
class StripePaymentGateway implements PaymentGatewayInterface {
    private $apiKey;
    private $stripeClient;

    public function __construct(StripeClient $stripeClient) {
        $this->stripeClient = $stripeClient;
    }

    public function process(Order $order): PaymentResult {
        try {
            $charge = $this->stripeClient->charges->create([
                'amount' => $order->getTotalAmount(),
                'currency' => 'usd',
                'source' => $order->getPaymentToken(),
                'description' => "Order #{$order->getId()}"
            ]);
            
            return new PaymentResult($charge->id, PaymentStatus::SUCCESS);
        } catch (StripeException $e) {
            return new PaymentResult(null, PaymentStatus::FAILED, $e->getMessage());
        }
    }
}

// Service Provider Registration
class PaymentServiceProvider extends ServiceProvider {
    public function register() {
        $this->app->bind(PaymentGatewayInterface::class, function ($app) {
            return new StripePaymentGateway(
                new StripeClient(config('services.stripe.secret'))
            );
        });
    }
}
```

### Example 2: Notification System

```php
// Interface
interface NotificationService {
    public function send(User $user, Notification $notification): void;
}

// Multiple Implementations
class EmailNotification implements NotificationService {
    private $mailer;
    
    public function __construct(Mailer $mailer) {
        $this->mailer = $mailer;
    }
    
    public function send(User $user, Notification $notification): void {
        $this->mailer->to($user->email)
            ->send(new NotificationMail($notification));
    }
}

class SmsNotification implements NotificationService {
    private $twilioClient;
    
    public function __construct(TwilioClient $twilioClient) {
        $this->twilioClient = $twilioClient;
    }
    
    public function send(User $user, Notification $notification): void {
        $this->twilioClient->messages->create(
            $user->phone_number,
            ['body' => $notification->getMessage()]
        );
    }
}
```

---

## 🎓 Interview Questions & Answers

### Q1: What's the difference between bind() and singleton()?
**Answer:** 
- `bind()` creates a new instance every time the dependency is requested
- `singleton()` creates one instance and reuses it throughout the application lifecycle

Example:
```php
// New instance every time
$this->app->bind(PaymentGateway::class, function ($app) {
    return new StripePaymentGateway(config('services.stripe.key'));
});

// Same instance every time
$this->app->singleton(Cache::class, function ($app) {
    return new RedisCache(config('cache.redis'));
});
```

### Q2: What is the Service Container used for?
**Answer:** The Service Container is Laravel's dependency injection container that manages:
- Class dependencies
- Dependency resolution
- Service registration
- Interface to implementation binding
- Singleton management

### Q3: Real example where DI helped in testing
**Answer:**
```php
// Production Code
class OrderService {
    public function __construct(PaymentGatewayInterface $payment) {
        $this->payment = $payment;
    }
}

// Test Code
class OrderServiceTest extends TestCase {
    public function test_order_processing() {
        $mockPayment = Mockery::mock(PaymentGatewayInterface::class);
        $mockPayment->shouldReceive('process')
            ->once()
            ->andReturn(new PaymentResult('fake-id', PaymentStatus::SUCCESS));
            
        $orderService = new OrderService($mockPayment);
        $result = $orderService->processOrder($order);
        
        $this->assertTrue($result->isSuccessful());
    }
}
```

### Q4: What happens if you don't bind an interface?
**Answer:** Laravel will attempt to resolve the concrete implementation directly. If it can't:
- For interfaces: Laravel throws a BindingResolutionException
- For concrete classes: Laravel attempts to instantiate them directly

### Q5: Constructor vs Setter Injection
**Answer:**
```php
// Constructor Injection
class UserService {
    private $repository;
    
    public function __construct(UserRepository $repository) {
        $this->repository = $repository; // Required dependency
    }
}

// Setter Injection
class UserService {
    private $logger;
    
    public function setLogger(LoggerInterface $logger) {
        $this->logger = $logger; // Optional dependency
    }
}
```

Key differences:
- Constructor injection ensures required dependencies are available
- Setter injection is better for optional dependencies
- Constructor injection makes dependencies obvious
- Setter injection allows more flexibility but can lead to incomplete objects

---

## 🏆 Best Practices

1. **Use Interface Segregation:**
```php
interface PaymentGatewayInterface {
    public function charge(float $amount): PaymentResult;
}

interface RefundablePaymentGateway extends PaymentGatewayInterface {
    public function refund(string $transactionId): RefundResult;
}
```

2. **Contextual Binding:**
```php
$this->app->when(PhotoController::class)
          ->needs(FileSystemInterface::class)
          ->give(S3FileSystem::class);
```

3. **Use Factory Pattern when needed:**
```php
class PaymentGatewayFactory {
    public function create(string $type): PaymentGatewayInterface {
        return match($type) {
            'stripe' => new StripePaymentGateway(),
            'paypal' => new PayPalPaymentGateway(),
            default => throw new InvalidArgumentException("Unknown gateway type")
        };
    }
}
```

## 📚 Additional Resources

- [Laravel Official Documentation](https://laravel.com/docs/container)
- [SOLID Principles in PHP](https://laracasts.com/series/solid-principles-in-php)
- [Laravel Service Container Deep Dive](https://laravel.com/docs/providers)
- [Testing Laravel Applications](https://laravel.com/docs/testing) 
