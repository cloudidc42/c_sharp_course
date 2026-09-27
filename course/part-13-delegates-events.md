# Part 13: Delegates and Events (Steps 371-410)

## เป้าหมายการเรียนรู้
- เข้าใจ Delegate เป็น type-safe function pointer
- Built-in delegates: Action, Func, Predicate
- Multicast delegates และ invocation list
- Events และ event handling
- Custom EventArgs
- Event aggregator pattern
- Weak event pattern

---

## Step 371: Delegate Fundamentals

```csharp
// Delegate = type-safe function pointer
// ประกาศ delegate type
delegate int MathOperation(int a, int b);
delegate string Transform(string input);
delegate bool Predicate(int number);

class Program
{
    static int Add(int a, int b) => a + b;
    static int Multiply(int a, int b) => a * b;
    static int Subtract(int a, int b) => a - b;
    
    static void Main()
    {
        // สร้าง delegate instance
        MathOperation op = Add;
        Console.WriteLine(op(3, 4));  // 7
        
        // เปลี่ยน method ที่ delegate ชี้ไป
        op = Multiply;
        Console.WriteLine(op(3, 4));  // 12
        
        // ส่ง delegate เป็น parameter
        PrintResult(Add, 10, 5);       // 10 + 5 = 15
        PrintResult(Subtract, 10, 5);  // 10 - 5 = 5
        
        // Return delegate from method
        MathOperation selected = SelectOperation("+");
        Console.WriteLine(selected(3, 4)); // 7
        
        // Lambda as delegate
        MathOperation power = (a, b) => (int)Math.Pow(a, b);
        Console.WriteLine(power(2, 10)); // 1024
        
        // Anonymous method (older syntax)
        MathOperation modulo = delegate(int a, int b) { return a % b; };
        Console.WriteLine(modulo(10, 3)); // 1
        
        // Delegate with instance method
        var calc = new Calculator();
        MathOperation divideOp = calc.Divide;
        Console.WriteLine(divideOp(10, 2)); // 5
        
        // Null check
        MathOperation? nullOp = null;
        Console.WriteLine(nullOp?.Invoke(1, 2) ?? 0); // 0 (safe invoke)
    }
    
    static void PrintResult(MathOperation operation, int a, int b)
        => Console.WriteLine($"{a} op {b} = {operation(a, b)}");
    
    static MathOperation SelectOperation(string op) => op switch
    {
        "+" => Add,
        "*" => Multiply,
        "-" => Subtract,
        _   => throw new ArgumentException($"Unknown: {op}")
    };
}

class Calculator
{
    public int Divide(int a, int b) => a / b;
}
```

---

## Step 372: Multicast Delegates

```csharp
// Multicast delegate = delegate ที่ชี้ไปหลาย methods
delegate void Logger(string message);
delegate int Processor(int value); // return value: last one wins

class Program
{
    static void ConsoleLog(string msg) => Console.WriteLine($"[Console] {msg}");
    static void FileLog(string msg) => Console.WriteLine($"[File] {msg}");
    static void NetworkLog(string msg) => Console.WriteLine($"[Network] {msg}");
    
    static void Main()
    {
        // Combine delegates with +
        Logger multi = ConsoleLog;
        multi += FileLog;
        multi += NetworkLog;
        
        multi("Error occurred!"); 
        // [Console] Error occurred!
        // [File] Error occurred!
        // [Network] Error occurred!
        
        // Remove with -
        multi -= FileLog;
        multi("After removing file log");
        // [Console] After removing file log
        // [Network] After removing file log
        
        // GetInvocationList — inspect delegate chain
        Delegate[] handlers = multi.GetInvocationList();
        Console.WriteLine($"Handlers: {handlers.Length}"); // 2
        
        // Invoke each individually (to handle exceptions)
        foreach (Delegate handler in handlers)
        {
            try
            {
                handler.DynamicInvoke("Message");
            }
            catch (Exception ex)
            {
                Console.WriteLine($"Handler failed: {ex.Message}");
            }
        }
        
        // Return values: last delegate's return value
        Func<int, int> pipeline = x => x + 1;
        pipeline += x => x * 2;
        pipeline += x => x - 3;
        Console.WriteLine(pipeline(5)); // Only last result: (5-3)=2? No, last registered
        // Actually: all execute with original value 5
        // Last: 5 - 3 = 2
        
        // Null checks
        Logger? maybeNull = null;
        maybeNull?.Invoke("Safe call"); // no-op
        
        // Thread-safe delegate invocation pattern
        Logger? current = multi; // copy reference
        current?.Invoke("Thread safe");
    }
}
```

---

## Step 373: Built-in Delegates — Action, Func, Predicate

```csharp
// .NET มี built-in generic delegates ที่ใช้บ่อย

// Action<T1, T2, ...> — void method, 0-16 parameters
Action print = () => Console.WriteLine("Hello");
Action<string> printStr = s => Console.WriteLine(s);
Action<string, int> printName = (name, age) => Console.WriteLine($"{name}: {age}");
Action<int, int, string> triple = (a, b, c) => Console.WriteLine($"{a}, {b}, {c}");

print();
printStr("World");
printName("Alice", 30);

// Func<T1, T2, ..., TResult> — returns value, 0-16 input params
Func<int> getNumber = () => 42;
Func<int, int> square = x => x * x;
Func<int, int, int> add = (a, b) => a + b;
Func<string, string, bool> contains = (s, sub) => s.Contains(sub);
Func<int, string> format = n => $"Number: {n:N0}";

Console.WriteLine(getNumber());         // 42
Console.WriteLine(square(5));           // 25
Console.WriteLine(add(3, 4));           // 7
Console.WriteLine(format(1234));        // Number: 1,234

// Predicate<T> — takes T, returns bool (equivalent to Func<T, bool>)
Predicate<int> isEven = n => n % 2 == 0;
Predicate<string> isEmpty = s => string.IsNullOrEmpty(s);

Console.WriteLine(isEven(4));           // True
Console.WriteLine(isEmpty(""));         // True

List<int> numbers = new() { 1, 2, 3, 4, 5, 6 };
List<int> evens = numbers.FindAll(isEven); // [2, 4, 6]

// Comparison<T> — for sorting
Comparison<string> byLength = (a, b) => a.Length.CompareTo(b.Length);
List<string> words = new() { "banana", "apple", "cherry", "fig" };
words.Sort(byLength);
Console.WriteLine(string.Join(", ", words)); // fig, apple, banana, cherry

// Converter<TInput, TOutput>
Converter<string, int> toInt = s => int.Parse(s);
string[] strNums = { "1", "2", "3", "4", "5" };
int[] intNums = Array.ConvertAll(strNums, toInt);

// EventHandler<TEventArgs>
EventHandler<EventArgs> handler = (sender, e) => Console.WriteLine("Event fired!");

// Practical: Higher-order functions
static T Memoize<T>(Func<int, T> f)
{
    var cache = new Dictionary<int, T>();
    return n => cache.TryGetValue(n, out T? v) ? v : cache[n] = f(n);
}

Func<int, long> fibonacci = null!;
fibonacci = Memoize<long>(n => n <= 1 ? n : fibonacci(n - 1) + fibonacci(n - 2));
Console.WriteLine(fibonacci(40)); // Fast!
```

---

## Step 374: Events — Basics

```csharp
// Event = multicast delegate ที่มี access modifier
// ป้องกันไม่ให้ subscriber invoke event โดยตรง

class Button
{
    // Event declaration
    public event EventHandler? Click;
    public event EventHandler<MouseEventArgs>? MouseMove;
    
    // ปุ่มที่มี event ใหม่
    public void OnClick()
    {
        // Raise event — thread-safe null check
        Click?.Invoke(this, EventArgs.Empty);
    }
    
    public void OnMouseMove(int x, int y)
    {
        MouseMove?.Invoke(this, new MouseEventArgs(x, y));
    }
}

class MouseEventArgs : EventArgs
{
    public int X { get; init; }
    public int Y { get; init; }
    
    public MouseEventArgs(int x, int y)
    {
        X = x;
        Y = y;
    }
}

class Program
{
    static void Main()
    {
        var button = new Button();
        
        // Subscribe (+=)
        button.Click += OnButtonClick;
        button.Click += (sender, e) => Console.WriteLine("Lambda handler");
        
        // Named method handler
        void LocalHandler(object? sender, EventArgs e) => Console.WriteLine("Local function handler");
        button.Click += LocalHandler;
        
        // Trigger events
        button.OnClick();
        // Output:
        // Button clicked!
        // Lambda handler
        // Local function handler
        
        // Unsubscribe (-=)
        button.Click -= OnButtonClick;
        button.OnClick();
        // Only lambda and local function handlers fire
        
        // Mouse event
        button.MouseMove += (sender, e) => Console.WriteLine($"Mouse at ({e.X}, {e.Y})");
        button.OnMouseMove(100, 200);
        
        // Can't do: button.Click(this, EventArgs.Empty); // Compile error!
        // Can't do: button.Click = null;                 // Compile error!
    }
    
    static void OnButtonClick(object? sender, EventArgs e)
    {
        Console.WriteLine($"Button clicked! Sender: {sender?.GetType().Name}");
    }
}
```

---

## Step 375: Custom EventArgs

```csharp
// สร้าง EventArgs แบบต่างๆ สำหรับ domain events

// 1. Simple event args
class OrderCreatedEventArgs : EventArgs
{
    public int OrderId { get; init; }
    public decimal Amount { get; init; }
    public DateTime CreatedAt { get; init; }
    
    public OrderCreatedEventArgs(int orderId, decimal amount)
    {
        OrderId = orderId;
        Amount = amount;
        CreatedAt = DateTime.UtcNow;
    }
}

// 2. Cancellable event args (like in Windows Forms)
class BeforeDeleteEventArgs : EventArgs
{
    public int ItemId { get; init; }
    public bool Cancel { get; set; }   // subscriber สามารถยกเลิกได้
    public string? CancelReason { get; set; }
    
    public BeforeDeleteEventArgs(int itemId)
    {
        ItemId = itemId;
    }
}

// 3. Progress event args
class ProgressEventArgs : EventArgs
{
    public int Current { get; init; }
    public int Total { get; init; }
    public string Message { get; init; }
    public double Percentage => Total > 0 ? (double)Current / Total * 100 : 0;
    
    public ProgressEventArgs(int current, int total, string message)
    {
        Current = current;
        Total = total;
        Message = message;
    }
}

// 4. Error event args
class ErrorEventArgs : EventArgs
{
    public Exception Exception { get; init; }
    public string Context { get; init; }
    public bool Handled { get; set; }
    
    public ErrorEventArgs(Exception exception, string context)
    {
        Exception = exception;
        Context = context;
    }
}

// Usage
class OrderService
{
    public event EventHandler<OrderCreatedEventArgs>? OrderCreated;
    public event EventHandler<BeforeDeleteEventArgs>? BeforeDelete;
    public event EventHandler<ProgressEventArgs>? ExportProgress;
    public event EventHandler<ErrorEventArgs>? ErrorOccurred;
    
    public void CreateOrder(decimal amount)
    {
        int orderId = new Random().Next(1000, 9999);
        Console.WriteLine($"Creating order {orderId}...");
        
        OrderCreated?.Invoke(this, new OrderCreatedEventArgs(orderId, amount));
    }
    
    public bool DeleteOrder(int orderId)
    {
        var args = new BeforeDeleteEventArgs(orderId);
        BeforeDelete?.Invoke(this, args);
        
        if (args.Cancel)
        {
            Console.WriteLine($"Delete cancelled: {args.CancelReason}");
            return false;
        }
        
        Console.WriteLine($"Order {orderId} deleted");
        return true;
    }
    
    public async Task ExportOrdersAsync(int count)
    {
        for (int i = 1; i <= count; i++)
        {
            await Task.Delay(100); // simulate work
            ExportProgress?.Invoke(this, new ProgressEventArgs(i, count, $"Exporting order {i}"));
        }
    }
}

class Program
{
    static async Task Main()
    {
        var service = new OrderService();
        
        service.OrderCreated += (s, e) =>
            Console.WriteLine($"New order: #{e.OrderId}, Amount: {e.Amount:C}, At: {e.CreatedAt}");
        
        service.BeforeDelete += (s, e) =>
        {
            if (e.ItemId < 1000)
            {
                e.Cancel = true;
                e.CancelReason = "Cannot delete low-ID orders";
            }
        };
        
        service.ExportProgress += (s, e) =>
        {
            Console.Write($"\r[{new string('#', (int)(e.Percentage / 5)),-20}] {e.Percentage:F0}% - {e.Message}");
        };
        
        service.CreateOrder(999.99m);
        service.DeleteOrder(500);   // will be cancelled
        service.DeleteOrder(5000);  // will succeed
        
        await service.ExportOrdersAsync(10);
        Console.WriteLine("\nExport complete!");
    }
}
```

---

## Step 376: Event Pattern Best Practices

```csharp
// Pattern: Thread-safe event raising
class SafeEventPublisher
{
    // Thread-safe null check with local copy
    private EventHandler? _dataReceived;
    public event EventHandler? DataReceived
    {
        add    { _dataReceived += value; }
        remove { _dataReceived -= value; }
    }
    
    protected virtual void OnDataReceived(EventArgs e)
    {
        // Local copy prevents race condition where handler is removed between check and invoke
        EventHandler? handler = _dataReceived;
        handler?.Invoke(this, e);
    }
    
    // Pattern: virtual protected OnXxx for inheritance
    public void ReceiveData()
    {
        // ... process data ...
        OnDataReceived(EventArgs.Empty);
    }
}

// Pattern: Observable with IDisposable subscription
class EventSubscription : IDisposable
{
    private readonly Action _unsubscribe;
    private bool _disposed;
    
    public EventSubscription(Action unsubscribe)
    {
        _unsubscribe = unsubscribe;
    }
    
    public void Dispose()
    {
        if (!_disposed)
        {
            _unsubscribe();
            _disposed = true;
        }
    }
}

class TemperatureSensor
{
    public event EventHandler<double>? TemperatureChanged;
    
    public IDisposable Subscribe(Action<double> handler)
    {
        EventHandler<double> eventHandler = (_, temp) => handler(temp);
        TemperatureChanged += eventHandler;
        return new EventSubscription(() => TemperatureChanged -= eventHandler);
    }
    
    public void UpdateTemperature(double celsius)
    {
        TemperatureChanged?.Invoke(this, celsius);
    }
}

// Pattern: Weak event (prevent memory leaks)
class WeakEventManager<TEventArgs> where TEventArgs : EventArgs
{
    private readonly List<WeakReference<Action<object?, TEventArgs>>> _handlers = new();
    
    public void AddHandler(Action<object?, TEventArgs> handler)
    {
        _handlers.Add(new WeakReference<Action<object?, TEventArgs>>(handler));
    }
    
    public void RemoveHandler(Action<object?, TEventArgs> handler)
    {
        _handlers.RemoveAll(wr => !wr.TryGetTarget(out var h) || h == handler);
    }
    
    public void RaiseEvent(object? sender, TEventArgs args)
    {
        var dead = new List<WeakReference<Action<object?, TEventArgs>>>();
        foreach (var wr in _handlers.ToArray())
        {
            if (wr.TryGetTarget(out var handler))
                handler(sender, args);
            else
                dead.Add(wr);
        }
        foreach (var d in dead)
            _handlers.Remove(d);
    }
}

// Usage
var sensor = new TemperatureSensor();

// Auto-unsubscribe with using
using (var sub = sensor.Subscribe(temp =>
    Console.WriteLine($"Temperature: {temp}°C")))
{
    sensor.UpdateTemperature(25.5);
    sensor.UpdateTemperature(26.3);
} // Unsubscribed here automatically

sensor.UpdateTemperature(27.0); // No output (unsubscribed)
```

---

## Step 377: Event Aggregator Pattern

```csharp
using System.Collections.Concurrent;

// Event Aggregator = central hub สำหรับ publish/subscribe
// Decouples publishers from subscribers

// 1. Define event types
interface IEvent { }

record UserLoggedIn(int UserId, string Username, DateTime At) : IEvent;
record OrderPlaced(int OrderId, int UserId, decimal Amount) : IEvent;
record PaymentReceived(int OrderId, decimal Amount, string Method) : IEvent;
record EmailRequested(string To, string Subject, string Body) : IEvent;

// 2. Event Aggregator interface and implementation
interface IEventAggregator
{
    void Subscribe<T>(Action<T> handler) where T : IEvent;
    void Unsubscribe<T>(Action<T> handler) where T : IEvent;
    void Publish<T>(T @event) where T : IEvent;
}

class EventAggregator : IEventAggregator
{
    private readonly ConcurrentDictionary<Type, List<Delegate>> _handlers = new();
    private readonly object _lock = new();
    
    public void Subscribe<T>(Action<T> handler) where T : IEvent
    {
        _handlers.AddOrUpdate(
            typeof(T),
            _ => new List<Delegate> { handler },
            (_, list) => { lock (_lock) { list.Add(handler); return list; } }
        );
    }
    
    public void Unsubscribe<T>(Action<T> handler) where T : IEvent
    {
        if (_handlers.TryGetValue(typeof(T), out var list))
            lock (_lock) { list.Remove(handler); }
    }
    
    public void Publish<T>(T @event) where T : IEvent
    {
        if (_handlers.TryGetValue(typeof(T), out var list))
        {
            List<Delegate> snapshot;
            lock (_lock) { snapshot = new List<Delegate>(list); }
            foreach (var handler in snapshot)
                (handler as Action<T>)?.Invoke(@event);
        }
    }
}

// 3. Subscribers (handlers)
class AuditLogger
{
    public AuditLogger(IEventAggregator aggregator)
    {
        aggregator.Subscribe<UserLoggedIn>(OnUserLoggedIn);
        aggregator.Subscribe<OrderPlaced>(OnOrderPlaced);
        aggregator.Subscribe<PaymentReceived>(OnPaymentReceived);
    }
    
    void OnUserLoggedIn(UserLoggedIn e) =>
        Console.WriteLine($"[AUDIT] User {e.Username} logged in at {e.At}");
    
    void OnOrderPlaced(OrderPlaced e) =>
        Console.WriteLine($"[AUDIT] Order #{e.OrderId} placed by user {e.UserId} for {e.Amount:C}");
    
    void OnPaymentReceived(PaymentReceived e) =>
        Console.WriteLine($"[AUDIT] Payment {e.Amount:C} via {e.Method} for order #{e.OrderId}");
}

class EmailService
{
    public EmailService(IEventAggregator aggregator)
    {
        aggregator.Subscribe<UserLoggedIn>(OnUserLoggedIn);
        aggregator.Subscribe<OrderPlaced>(OnOrderPlaced);
        aggregator.Subscribe<EmailRequested>(OnEmailRequested);
    }
    
    void OnUserLoggedIn(UserLoggedIn e) =>
        Console.WriteLine($"[EMAIL] Welcome back, {e.Username}!");
    
    void OnOrderPlaced(OrderPlaced e) =>
        Console.WriteLine($"[EMAIL] Order confirmation sent for #{e.OrderId}");
    
    void OnEmailRequested(EmailRequested e) =>
        Console.WriteLine($"[EMAIL] Sending to {e.To}: {e.Subject}");
}

class InventoryService
{
    public InventoryService(IEventAggregator aggregator)
    {
        aggregator.Subscribe<OrderPlaced>(OnOrderPlaced);
    }
    
    void OnOrderPlaced(OrderPlaced e) =>
        Console.WriteLine($"[INVENTORY] Reserving items for order #{e.OrderId}");
}

// 4. Publisher (business logic)
class OrderProcessor
{
    private readonly IEventAggregator _aggregator;
    private int _nextOrderId = 1;
    
    public OrderProcessor(IEventAggregator aggregator)
        => _aggregator = aggregator;
    
    public int PlaceOrder(int userId, decimal amount)
    {
        int orderId = _nextOrderId++;
        Console.WriteLine($"\n--- Processing Order #{orderId} ---");
        
        _aggregator.Publish(new OrderPlaced(orderId, userId, amount));
        return orderId;
    }
    
    public void ProcessPayment(int orderId, decimal amount, string method)
    {
        Console.WriteLine($"\n--- Processing Payment for Order #{orderId} ---");
        _aggregator.Publish(new PaymentReceived(orderId, amount, method));
    }
}

// Demo
var aggregator = new EventAggregator();

// Register subscribers
var audit = new AuditLogger(aggregator);
var email = new EmailService(aggregator);
var inventory = new InventoryService(aggregator);
var processor = new OrderProcessor(aggregator);

// Simulate activity
aggregator.Publish(new UserLoggedIn(1, "Alice", DateTime.Now));
int orderId = processor.PlaceOrder(userId: 1, amount: 299.99m);
processor.ProcessPayment(orderId, amount: 299.99m, method: "Credit Card");
```

---

## Step 378: Delegates and Functional Patterns

```csharp
// Pattern: Strategy with Delegate
class Sorter<T>
{
    private Func<T, T, int>? _comparer;
    private Func<IEnumerable<T>, IEnumerable<T>>? _filter;
    
    public Sorter<T> WithComparer(Func<T, T, int> comparer)
    {
        _comparer = comparer;
        return this;
    }
    
    public Sorter<T> WithFilter(Func<IEnumerable<T>, IEnumerable<T>> filter)
    {
        _filter = filter;
        return this;
    }
    
    public IEnumerable<T> Sort(IEnumerable<T> items)
    {
        var filtered = _filter != null ? _filter(items) : items;
        if (_comparer == null) return filtered;
        return filtered.OrderBy(x => x, Comparer<T>.Create((a, b) => _comparer(a, b)));
    }
}

// Pattern: Retry with delegate
static async Task<T> RetryAsync<T>(
    Func<Task<T>> operation,
    int maxRetries = 3,
    TimeSpan? delay = null,
    Predicate<Exception>? shouldRetry = null)
{
    delay ??= TimeSpan.FromSeconds(1);
    shouldRetry ??= ex => true;
    
    Exception lastException = null!;
    for (int i = 0; i < maxRetries; i++)
    {
        try { return await operation(); }
        catch (Exception ex) when (shouldRetry(ex))
        {
            lastException = ex;
            Console.WriteLine($"Retry {i + 1}/{maxRetries}: {ex.Message}");
            if (i < maxRetries - 1)
                await Task.Delay(delay.Value * (i + 1));
        }
    }
    throw lastException;
}

// Pattern: Pipeline / Chain of responsibility
class Pipeline<T>
{
    private readonly List<Func<T, T>> _steps = new();
    
    public Pipeline<T> AddStep(Func<T, T> step)
    {
        _steps.Add(step);
        return this;
    }
    
    public Pipeline<T> AddConditionalStep(Func<T, bool> condition, Func<T, T> step)
    {
        _steps.Add(input => condition(input) ? step(input) : input);
        return this;
    }
    
    public T Execute(T input)
        => _steps.Aggregate(input, (current, step) => step(current));
}

// Pattern: Command with delegate
class CommandExecutor
{
    private readonly Stack<(Action Execute, Action Undo)> _history = new();
    
    public void Execute(Action execute, Action undo)
    {
        execute();
        _history.Push((execute, undo));
    }
    
    public void Undo()
    {
        if (_history.TryPop(out var command))
            command.Undo();
    }
    
    public void Redo()
    {
        if (_history.TryPop(out var command))
        {
            command.Execute();
            _history.Push(command);
        }
    }
}

// Usage examples
var sorter = new Sorter<string>()
    .WithFilter(items => items.Where(s => s.Length > 3))
    .WithComparer((a, b) => a.Length.CompareTo(b.Length));

var sorted = sorter.Sort(new[] { "hi", "hello", "hey", "world", "c#" });
Console.WriteLine(string.Join(", ", sorted)); // hello, world

// Text pipeline
var textPipeline = new Pipeline<string>()
    .AddStep(s => s.Trim())
    .AddStep(s => s.ToLower())
    .AddStep(s => System.Text.RegularExpressions.Regex.Replace(s, @"\s+", " "))
    .AddConditionalStep(s => s.Length > 100, s => s[..100] + "...");

string processed = textPipeline.Execute("  HELLO   WORLD  ");
Console.WriteLine(processed); // "hello world"

// Command executor
var list = new List<string>();
var executor = new CommandExecutor();

executor.Execute(
    () => list.Add("Hello"),
    () => list.Remove("Hello")
);
executor.Execute(
    () => list.Add("World"),
    () => list.Remove("World")
);
Console.WriteLine(string.Join(", ", list)); // Hello, World
executor.Undo();
Console.WriteLine(string.Join(", ", list)); // Hello
```

---

## Step 379: Advanced Event Patterns

```csharp
using System.Reactive; // Optional: Reactive Extensions

// Pattern: Observable using events (simple Rx-like)
class Subject<T>
{
    private readonly List<IObserver<T>> _observers = new();
    
    public IDisposable Subscribe(IObserver<T> observer)
    {
        _observers.Add(observer);
        return new Unsubscriber<T>(_observers, observer);
    }
    
    public IDisposable Subscribe(
        Action<T> onNext,
        Action<Exception>? onError = null,
        Action? onCompleted = null)
    {
        var observer = new AnonymousObserver<T>(onNext, onError, onCompleted);
        return Subscribe(observer);
    }
    
    public void OnNext(T value)
    {
        foreach (var o in _observers.ToArray())
            o.OnNext(value);
    }
    
    public void OnError(Exception error)
    {
        foreach (var o in _observers.ToArray())
            o.OnError(error);
    }
    
    public void OnCompleted()
    {
        foreach (var o in _observers.ToArray())
            o.OnCompleted();
        _observers.Clear();
    }
}

class Unsubscriber<T> : IDisposable
{
    private readonly List<IObserver<T>> _observers;
    private readonly IObserver<T> _observer;
    
    public Unsubscriber(List<IObserver<T>> observers, IObserver<T> observer)
    {
        _observers = observers;
        _observer = observer;
    }
    
    public void Dispose() => _observers.Remove(_observer);
}

class AnonymousObserver<T> : IObserver<T>
{
    private readonly Action<T> _onNext;
    private readonly Action<Exception>? _onError;
    private readonly Action? _onCompleted;
    
    public AnonymousObserver(Action<T> onNext, Action<Exception>? onError, Action? onCompleted)
    {
        _onNext = onNext;
        _onError = onError;
        _onCompleted = onCompleted;
    }
    
    public void OnNext(T value) => _onNext(value);
    public void OnError(Exception error) => _onError?.Invoke(error);
    public void OnCompleted() => _onCompleted?.Invoke();
}

// Pattern: Event sourcing
class EventStore
{
    private readonly List<object> _events = new();
    public IReadOnlyList<object> Events => _events.AsReadOnly();
    
    public event Action<object>? EventAppended;
    
    public void Append(object @event)
    {
        _events.Add(@event);
        EventAppended?.Invoke(@event);
    }
}

// Pattern: Reactive property (property change notification)
class ReactiveProperty<T>
{
    private T _value;
    public event Action<T, T>? Changed; // (oldValue, newValue)
    
    public ReactiveProperty(T initial) => _value = initial;
    
    public T Value
    {
        get => _value;
        set
        {
            if (!EqualityComparer<T>.Default.Equals(_value, value))
            {
                T old = _value;
                _value = value;
                Changed?.Invoke(old, value);
            }
        }
    }
    
    public IDisposable OnChange(Action<T, T> handler)
    {
        Changed += handler;
        return new EventSubscription2(() => Changed -= handler);
    }
}

class EventSubscription2 : IDisposable
{
    private readonly Action _dispose;
    public EventSubscription2(Action dispose) => _dispose = dispose;
    public void Dispose() => _dispose();
}

// Demo
var subject = new Subject<int>();

using var sub1 = subject.Subscribe(
    onNext: n => Console.WriteLine($"Observer1: {n}"),
    onCompleted: () => Console.WriteLine("Observer1 completed")
);

using var sub2 = subject.Subscribe(
    onNext: n => Console.WriteLine($"Observer2: {n * 2}"),
    onCompleted: () => Console.WriteLine("Observer2 completed")
);

subject.OnNext(1); // Observer1: 1, Observer2: 2
subject.OnNext(2); // Observer1: 2, Observer2: 4
sub1.Dispose();    // Unsubscribe observer1
subject.OnNext(3); // Only Observer2: 6
subject.OnCompleted();

// Reactive property
var name = new ReactiveProperty<string>("");
using var sub = name.OnChange((old, @new) =>
    Console.WriteLine($"Name changed: '{old}' → '{@new}'"));

name.Value = "Alice"; // Name changed: '' → 'Alice'
name.Value = "Alice"; // No event (same value)
name.Value = "Bob";   // Name changed: 'Alice' → 'Bob'
```

---

## Step 380: โปรแกรมตัวอย่าง — Event-Driven Order System

```csharp
using System;
using System.Collections.Generic;
using System.Threading.Tasks;

// ============================================================
// Event-Driven Order Processing System
// ============================================================

// Events
record OrderEvent(int OrderId, DateTime OccurredAt) { }
record OrderSubmitted(int OrderId, int UserId, decimal Amount, List<string> Items)
    : OrderEvent(OrderId, DateTime.UtcNow);
record PaymentProcessed(int OrderId, bool Success, string? FailureReason = null)
    : OrderEvent(OrderId, DateTime.UtcNow);
record OrderFulfilled(int OrderId, string TrackingNumber)
    : OrderEvent(OrderId, DateTime.UtcNow);
record OrderCancelled(int OrderId, string Reason)
    : OrderEvent(OrderId, DateTime.UtcNow);

// Event bus
class EventBus
{
    private readonly Dictionary<Type, List<Delegate>> _handlers = new();
    
    public void Subscribe<T>(Action<T> handler)
    {
        if (!_handlers.ContainsKey(typeof(T)))
            _handlers[typeof(T)] = new();
        _handlers[typeof(T)].Add(handler);
    }
    
    public async Task PublishAsync<T>(T @event)
    {
        Console.WriteLine($"\n[EventBus] Publishing {typeof(T).Name}");
        if (_handlers.TryGetValue(typeof(T), out var handlers))
        {
            var tasks = handlers
                .OfType<Action<T>>()
                .Select(h => Task.Run(() => h(@event)));
            await Task.WhenAll(tasks);
        }
    }
}

// Order aggregate
class Order
{
    public int Id { get; private set; }
    public int UserId { get; private set; }
    public decimal Amount { get; private set; }
    public List<string> Items { get; private set; } = new();
    public string Status { get; private set; } = "Draft";
    
    public static Order Create(int id, int userId, decimal amount, List<string> items)
    {
        return new Order { Id = id, UserId = userId, Amount = amount, Items = items };
    }
    
    public void MarkAsSubmitted() => Status = "Submitted";
    public void MarkAsPaid() => Status = "Paid";
    public void MarkAsFulfilled() => Status = "Fulfilled";
    public void MarkAsCancelled() => Status = "Cancelled";
}

// Order service
class OrderService
{
    private readonly EventBus _eventBus;
    private readonly Dictionary<int, Order> _orders = new();
    private int _nextId = 1;
    
    // Custom events for UI/monitoring
    public event EventHandler<Order>? OrderStateChanged;
    
    public OrderService(EventBus eventBus)
    {
        _eventBus = eventBus;
    }
    
    public async Task<Order> SubmitOrderAsync(int userId, decimal amount, List<string> items)
    {
        var order = Order.Create(_nextId++, userId, amount, items);
        _orders[order.Id] = order;
        order.MarkAsSubmitted();
        
        OrderStateChanged?.Invoke(this, order);
        await _eventBus.PublishAsync(new OrderSubmitted(order.Id, userId, amount, items));
        
        return order;
    }
    
    public void HandlePaymentResult(PaymentProcessed e)
    {
        if (_orders.TryGetValue(e.OrderId, out var order))
        {
            if (e.Success)
            {
                order.MarkAsPaid();
                OrderStateChanged?.Invoke(this, order);
            }
            else
            {
                order.MarkAsCancelled();
                OrderStateChanged?.Invoke(this, order);
            }
        }
    }
}

// Payment service
class PaymentService
{
    private readonly EventBus _eventBus;
    private readonly Random _rng = new();
    
    public PaymentService(EventBus eventBus)
    {
        _eventBus = eventBus;
        _eventBus.Subscribe<OrderSubmitted>(ProcessPaymentAsync);
    }
    
    private async void ProcessPaymentAsync(OrderSubmitted e)
    {
        Console.WriteLine($"[Payment] Processing payment for order #{e.OrderId}: {e.Amount:C}");
        await Task.Delay(500); // simulate payment processing
        
        bool success = _rng.Next(10) > 1; // 90% success rate
        await _eventBus.PublishAsync(new PaymentProcessed(
            e.OrderId, 
            success,
            success ? null : "Insufficient funds"
        ));
    }
}

// Fulfillment service
class FulfillmentService
{
    private readonly EventBus _eventBus;
    
    public FulfillmentService(EventBus eventBus)
    {
        _eventBus = eventBus;
        _eventBus.Subscribe<PaymentProcessed>(OnPaymentProcessed);
    }
    
    private async void OnPaymentProcessed(PaymentProcessed e)
    {
        if (!e.Success) return;
        
        Console.WriteLine($"[Fulfillment] Preparing shipment for order #{e.OrderId}");
        await Task.Delay(300);
        
        string tracking = $"TRK{e.OrderId:D6}";
        await _eventBus.PublishAsync(new OrderFulfilled(e.OrderId, tracking));
    }
}

// Email notification service
class EmailNotificationService
{
    public EmailNotificationService(EventBus eventBus)
    {
        eventBus.Subscribe<OrderSubmitted>(e => 
            Console.WriteLine($"[Email] Order confirmation sent for #{e.OrderId}"));
        eventBus.Subscribe<PaymentProcessed>(e =>
        {
            if (e.Success)
                Console.WriteLine($"[Email] Payment receipt sent for #{e.OrderId}");
            else
                Console.WriteLine($"[Email] Payment failure notice sent for #{e.OrderId}: {e.FailureReason}");
        });
        eventBus.Subscribe<OrderFulfilled>(e =>
            Console.WriteLine($"[Email] Shipping notification sent. Tracking: {e.TrackingNumber}"));
    }
}

// Monitoring dashboard
class OrderDashboard
{
    private int _totalOrders;
    private int _paidOrders;
    private int _cancelledOrders;
    
    public OrderDashboard(OrderService orderService, EventBus eventBus)
    {
        orderService.OrderStateChanged += (_, order) =>
        {
            switch (order.Status)
            {
                case "Submitted": _totalOrders++; break;
                case "Paid": _paidOrders++; break;
                case "Cancelled": _cancelledOrders++; break;
            }
            PrintStatus();
        };
    }
    
    void PrintStatus()
    {
        Console.WriteLine($"\n📊 Dashboard: Total={_totalOrders}, Paid={_paidOrders}, Cancelled={_cancelledOrders}");
    }
}

// Main
var eventBus = new EventBus();
var orderService = new OrderService(eventBus);
var paymentService = new PaymentService(eventBus);
var fulfillmentService = new FulfillmentService(eventBus);
var emailService = new EmailNotificationService(eventBus);
var dashboard = new OrderDashboard(orderService, eventBus);

Console.WriteLine("=== Event-Driven Order System ===\n");

// Process 3 orders
for (int i = 0; i < 3; i++)
{
    await orderService.SubmitOrderAsync(
        userId: i + 1,
        amount: (i + 1) * 199.99m,
        items: new() { $"Product A{i}", $"Product B{i}" }
    );
    await Task.Delay(1500); // wait for events to process
}

Console.WriteLine("\n=== Done ===");
```

---

## สรุป Part 13

### Delegate Fundamentals
- ✅ `delegate` keyword — type-safe function pointer
- ✅ Instance vs static method delegates
- ✅ Multicast delegates (`+=`, `-=`, `GetInvocationList`)
- ✅ Lambda and anonymous method as delegate

### Built-in Delegates
- ✅ `Action<T1,...>` — void return, 0-16 params
- ✅ `Func<T1,...,TResult>` — returns value
- ✅ `Predicate<T>` — `Func<T, bool>` alias
- ✅ `Comparison<T>`, `Converter<TInput, TOutput>`, `EventHandler<T>`

### Events
- ✅ `event` keyword — restricts direct invocation
- ✅ `EventHandler`, `EventHandler<TEventArgs>`
- ✅ Custom `EventArgs` with `Cancel`, progress, error
- ✅ Thread-safe event raising with local copy
- ✅ `+=`, `-=` subscription/unsubscription

### Patterns
- ✅ Strategy pattern with delegates
- ✅ Pipeline/Chain of Responsibility
- ✅ Command pattern (Execute/Undo)
- ✅ Event Aggregator (decoupled pub/sub)
- ✅ Observable/Subject pattern
- ✅ Reactive property
- ✅ Weak event (prevent memory leaks)

---

## แบบฝึกหัด

1. สร้าง `TypedEventBus` ที่ใช้ Generics และ compile-time type safety
2. เขียน `AsyncEventAggregator` ที่ support async handlers
3. สร้าง `PropertyChangeTracker<T>` ที่ track การเปลี่ยนแปลง properties ทั้งหมดใน object
4. Implement `UndoManager` โดยใช้ Command pattern ที่รองรับ undo history 50 steps
5. สร้าง `SignalR-like` event system ที่ broadcast events ไปยัง multiple connected clients
