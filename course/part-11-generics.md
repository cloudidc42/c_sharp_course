# Part 11: Generics
## Steps 291-320: การเขียนโค้ดที่ใช้ซ้ำได้ด้วย Generics

---

## Step 291: Generic Types พื้นฐาน

```csharp
// ============ ทำไมต้อง Generics? ============

// ❌ โดยไม่ใช้ Generics: ต้องเขียน method แยกทุกชนิด
int MaxInt(int a, int b) => a > b ? a : b;
double MaxDouble(double a, double b) => a > b ? a : b;
string MaxString(string a, string b) => string.Compare(a, b) > 0 ? a : b;

// ✅ ใช้ Generics: เขียนครั้งเดียว ใช้ได้ทุกชนิด
T Max<T>(T a, T b) where T : IComparable<T> => a.CompareTo(b) > 0 ? a : b;

Console.WriteLine(Max(10, 20));            // 20 (int)
Console.WriteLine(Max(3.14, 2.72));        // 3.14 (double)
Console.WriteLine(Max("Apple", "Banana")); // Banana (string)
Console.WriteLine(Max(DateTime.Now.AddDays(-1), DateTime.Now)); // Now

// ============ Generic Methods ============
void Swap<T>(ref T a, ref T b) => (a, b) = (b, a);

int x = 5, y = 10;
Swap(ref x, ref y);
Console.WriteLine($"After swap: x={x}, y={y}"); // x=10, y=5

string s1 = "Hello", s2 = "World";
Swap(ref s1, ref s2);
Console.WriteLine($"After swap: s1={s1}, s2={s2}");

// ============ Generic Classes ============
class Pair<T1, T2>
{
    public T1 First { get; }
    public T2 Second { get; }
    
    public Pair(T1 first, T2 second)
    {
        First = first;
        Second = second;
    }
    
    public (T1, T2) ToTuple() => (First, Second);
    
    public Pair<T2, T1> Flip() => new Pair<T2, T1>(Second, First);
    
    public override string ToString() => $"({First}, {Second})";
}

var nameAge = new Pair<string, int>("Alice", 30);
var ageScore = new Pair<int, double>(90, 98.5);
var flipped = nameAge.Flip();

Console.WriteLine(nameAge);
Console.WriteLine(flipped);
Console.WriteLine(ageScore.ToTuple());

// ============ Generic with Type Parameters ============
class TypedCollection<TItem>
{
    private List<TItem> _items = new();
    
    public void Add(TItem item) => _items.Add(item);
    public TItem this[int index] => _items[index];
    public int Count => _items.Count;
    
    public TypedCollection<TResult> Map<TResult>(Func<TItem, TResult> transform)
    {
        var result = new TypedCollection<TResult>();
        foreach (var item in _items)
            result.Add(transform(item));
        return result;
    }
    
    public TypedCollection<TItem> Filter(Func<TItem, bool> predicate)
    {
        var result = new TypedCollection<TItem>();
        foreach (var item in _items.Where(predicate))
            result.Add(item);
        return result;
    }
    
    public TResult Reduce<TResult>(TResult seed, Func<TResult, TItem, TResult> accumulate)
        => _items.Aggregate(seed, accumulate);
}

var numbers = new TypedCollection<int>();
for (int i = 1; i <= 10; i++) numbers.Add(i);

var doubled = numbers.Map(n => n * 2);
var evens = numbers.Filter(n => n % 2 == 0);
var sum = numbers.Reduce(0, (acc, n) => acc + n);

Console.WriteLine($"Count: {numbers.Count}");
Console.WriteLine($"Sum: {sum}");
```

---

## Step 292: Generic Constraints

```csharp
// ============ Generic Constraints ============

// 1. where T : struct (value type)
T DefaultValue<T>() where T : struct => default;
Console.WriteLine(DefaultValue<int>());    // 0
Console.WriteLine(DefaultValue<double>()); // 0
Console.WriteLine(DefaultValue<bool>());   // False

// 2. where T : class (reference type)
T? GetDefault<T>() where T : class => null;

// 3. where T : new() (must have parameterless constructor)
T CreateInstance<T>() where T : new() => new T();

class MyClass { public int Value { get; set; } }
var instance = CreateInstance<MyClass>();
Console.WriteLine(instance.GetType().Name);

// 4. where T : BaseClass (inherit from specific class)
class Repository<T> where T : Entity
{
    private List<T> _items = new();
    
    public void Add(T item)
    {
        item.CreatedAt = DateTime.Now;
        _items.Add(item);
    }
    
    public T? FindById(int id) => _items.FirstOrDefault(e => e.Id == id);
}

abstract class Entity
{
    public int Id { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime? UpdatedAt { get; set; }
}

class Product : Entity
{
    public string Name { get; set; } = "";
    public decimal Price { get; set; }
}

var repo = new Repository<Product>();
repo.Add(new Product { Id = 1, Name = "Apple", Price = 25 });
repo.Add(new Product { Id = 2, Name = "Banana", Price = 15 });
var found = repo.FindById(1);
Console.WriteLine($"Found: {found?.Name}");

// 5. where T : IInterface
void ProcessItems<T>(IEnumerable<T> items) where T : IComparable<T>
{
    var sorted = items.OrderBy(x => x).ToList();
    Console.WriteLine($"Min: {sorted.First()}, Max: {sorted.Last()}");
}

ProcessItems(new[] { 5, 2, 8, 1, 9 });
ProcessItems(new[] { "banana", "apple", "cherry" });

// 6. Multiple constraints
T CreateAndLog<T>(string label) 
    where T : Entity, new()
{
    var entity = new T();
    entity.CreatedAt = DateTime.Now;
    Console.WriteLine($"Created {label} at {entity.CreatedAt:HH:mm:ss}");
    return entity;
}

// 7. where T : unmanaged (C# 7.3+)
unsafe void ProcessBuffer<T>(T[] data) where T : unmanaged
{
    Console.WriteLine($"Processing {data.Length} items of size {sizeof(T)} bytes each");
}

// 8. where T : notnull (C# 8+)
Dictionary<TKey, TValue> CreateDictionary<TKey, TValue>() 
    where TKey : notnull
    => new Dictionary<TKey, TValue>();

// 9. Combining constraints
class SmartList<T> where T : IComparable<T>, IFormattable, new()
{
    private List<T> _items = new();
    
    public void Add(T item) => _items.Add(item);
    
    public string FormatAll(string format) =>
        string.Join(", ", _items.Select(i => i.ToString(format, null)));
    
    public T Min() => _items.Min()!;
    public T Max() => _items.Max()!;
}

// ============ Covariance และ Contravariance ============

// Covariance (out): สามารถใช้ชนิดที่เป็น subtype ได้
interface IProducer<out T>
{
    T Produce();
    IEnumerable<T> ProduceMany(int count);
}

// Contravariance (in): สามารถใช้ชนิดที่เป็น supertype ได้
interface IConsumer<in T>
{
    void Consume(T item);
    void ConsumeMany(IEnumerable<T> items);
}

class AnimalProducer : IProducer<Dog>
{
    public Dog Produce() => new Dog("Spot");
    public IEnumerable<Dog> ProduceMany(int count) =>
        Enumerable.Range(1, count).Select(i => new Dog($"Dog{i}"));
}

class Dog
{
    public string Name { get; }
    public Dog(string name) => Name = name;
}

// Covariance: Dog is-a Animal, so IProducer<Dog> can be used as IProducer<object>
IProducer<Dog> dogProducer = new AnimalProducer();
IProducer<object> objectProducer = dogProducer; // OK because of 'out'
```

---

## Step 293: Generic Data Structures

```csharp
// ============ Generic Stack ============
class Stack<T>
{
    private T[] _items;
    private int _top = -1;
    private int _capacity;
    
    public Stack(int capacity = 16)
    {
        _capacity = capacity;
        _items = new T[capacity];
    }
    
    public int Count => _top + 1;
    public bool IsEmpty => _top == -1;
    public bool IsFull => _top == _capacity - 1;
    
    public void Push(T item)
    {
        if (IsFull) Resize();
        _items[++_top] = item;
    }
    
    public T Pop()
    {
        if (IsEmpty) throw new InvalidOperationException("Stack is empty");
        var item = _items[_top];
        _items[_top--] = default!; // Clear reference
        return item;
    }
    
    public T Peek()
    {
        if (IsEmpty) throw new InvalidOperationException("Stack is empty");
        return _items[_top];
    }
    
    public bool TryPop(out T item)
    {
        if (IsEmpty) { item = default!; return false; }
        item = Pop();
        return true;
    }
    
    private void Resize()
    {
        _capacity *= 2;
        Array.Resize(ref _items, _capacity);
    }
    
    public override string ToString() =>
        $"Stack[{string.Join(", ", _items[..(_top + 1)].Reverse())}]";
}

var intStack = new Stack<int>();
intStack.Push(1); intStack.Push(2); intStack.Push(3);
Console.WriteLine(intStack);
Console.WriteLine($"Pop: {intStack.Pop()}");
Console.WriteLine($"Peek: {intStack.Peek()}");

// ============ Generic Queue ============
class Queue<T>
{
    private LinkedList<T> _list = new();
    
    public int Count => _list.Count;
    public bool IsEmpty => _list.Count == 0;
    
    public void Enqueue(T item) => _list.AddLast(item);
    
    public T Dequeue()
    {
        if (IsEmpty) throw new InvalidOperationException("Queue is empty");
        var item = _list.First!.Value;
        _list.RemoveFirst();
        return item;
    }
    
    public T Peek()
    {
        if (IsEmpty) throw new InvalidOperationException("Queue is empty");
        return _list.First!.Value;
    }
}

// ============ Generic Binary Tree ============
class BinaryTree<T> where T : IComparable<T>
{
    private class Node
    {
        public T Value;
        public Node? Left, Right;
        public Node(T value) => Value = value;
    }
    
    private Node? _root;
    
    public void Insert(T value)
    {
        _root = Insert(_root, value);
    }
    
    private Node Insert(Node? node, T value)
    {
        if (node == null) return new Node(value);
        int cmp = value.CompareTo(node.Value);
        if (cmp < 0) node.Left = Insert(node.Left, value);
        else if (cmp > 0) node.Right = Insert(node.Right, value);
        return node;
    }
    
    public bool Contains(T value) => Contains(_root, value);
    
    private bool Contains(Node? node, T value)
    {
        if (node == null) return false;
        int cmp = value.CompareTo(node.Value);
        return cmp == 0 || (cmp < 0 ? Contains(node.Left, value) : Contains(node.Right, value));
    }
    
    public IEnumerable<T> InOrder() => InOrder(_root);
    private IEnumerable<T> InOrder(Node? node)
    {
        if (node == null) yield break;
        foreach (T v in InOrder(node.Left)) yield return v;
        yield return node.Value;
        foreach (T v in InOrder(node.Right)) yield return v;
    }
    
    public int Height() => Height(_root);
    private int Height(Node? node) =>
        node == null ? 0 : 1 + Math.Max(Height(node.Left), Height(node.Right));
}

var tree = new BinaryTree<int>();
foreach (int n in new[] { 5, 3, 7, 1, 4, 6, 8, 2 })
    tree.Insert(n);

Console.WriteLine($"\nBST InOrder: {string.Join(", ", tree.InOrder())}");
Console.WriteLine($"Height: {tree.Height()}");
Console.WriteLine($"Contains 4: {tree.Contains(4)}");
Console.WriteLine($"Contains 9: {tree.Contains(9)}");

var strTree = new BinaryTree<string>();
foreach (string s in new[] { "banana", "apple", "cherry", "date" })
    strTree.Insert(s);
Console.WriteLine($"String BST: {string.Join(", ", strTree.InOrder())}");
```

---

## Step 294-300: Generic Algorithms

```csharp
// ============ Generic Sort Algorithms ============
static class SortAlgorithms
{
    // Bubble Sort
    public static void BubbleSort<T>(T[] arr) where T : IComparable<T>
    {
        for (int i = 0; i < arr.Length - 1; i++)
            for (int j = 0; j < arr.Length - i - 1; j++)
                if (arr[j].CompareTo(arr[j + 1]) > 0)
                    (arr[j], arr[j + 1]) = (arr[j + 1], arr[j]);
    }
    
    // Quick Sort
    public static void QuickSort<T>(T[] arr, int low = 0, int high = -1) where T : IComparable<T>
    {
        if (high == -1) high = arr.Length - 1;
        if (low < high)
        {
            int pivot = Partition(arr, low, high);
            QuickSort(arr, low, pivot - 1);
            QuickSort(arr, pivot + 1, high);
        }
    }
    
    private static int Partition<T>(T[] arr, int low, int high) where T : IComparable<T>
    {
        T pivot = arr[high];
        int i = low - 1;
        for (int j = low; j < high; j++)
            if (arr[j].CompareTo(pivot) <= 0)
                (arr[++i], arr[j]) = (arr[j], arr[i]);
        (arr[i + 1], arr[high]) = (arr[high], arr[i + 1]);
        return i + 1;
    }
    
    // Merge Sort
    public static T[] MergeSort<T>(T[] arr) where T : IComparable<T>
    {
        if (arr.Length <= 1) return arr;
        
        int mid = arr.Length / 2;
        T[] left = MergeSort(arr[..mid]);
        T[] right = MergeSort(arr[mid..]);
        
        return Merge(left, right);
    }
    
    private static T[] Merge<T>(T[] left, T[] right) where T : IComparable<T>
    {
        T[] result = new T[left.Length + right.Length];
        int i = 0, j = 0, k = 0;
        
        while (i < left.Length && j < right.Length)
        {
            if (left[i].CompareTo(right[j]) <= 0)
                result[k++] = left[i++];
            else
                result[k++] = right[j++];
        }
        
        while (i < left.Length) result[k++] = left[i++];
        while (j < right.Length) result[k++] = right[j++];
        
        return result;
    }
    
    // Binary Search
    public static int BinarySearch<T>(T[] arr, T target) where T : IComparable<T>
    {
        int low = 0, high = arr.Length - 1;
        
        while (low <= high)
        {
            int mid = (low + high) / 2;
            int cmp = arr[mid].CompareTo(target);
            
            if (cmp == 0) return mid;
            if (cmp < 0) low = mid + 1;
            else high = mid - 1;
        }
        
        return -1;
    }
    
    // Generic with custom comparer
    public static void Sort<T>(T[] arr, Comparison<T> comparison)
    {
        Array.Sort(arr, comparison);
    }
}

// Test
int[] nums = { 64, 34, 25, 12, 22, 11, 90 };
int[] sorted = SortAlgorithms.MergeSort(nums);
Console.WriteLine($"MergeSort: {string.Join(", ", sorted)}");

string[] words = { "banana", "apple", "cherry", "date" };
SortAlgorithms.QuickSort(words);
Console.WriteLine($"QuickSort: {string.Join(", ", words)}");

int idx = SortAlgorithms.BinarySearch(sorted, 34);
Console.WriteLine($"BinarySearch(34): index {idx}");

// Custom comparison
var products = new[] { ("Apple", 25m), ("Banana", 15m), ("Cherry", 50m) };
SortAlgorithms.Sort(products, (a, b) => a.Item2.CompareTo(b.Item2));
foreach (var (name, price) in products)
    Console.WriteLine($"  {name}: {price}");

// ============ Generic Functional Operations ============
static class Functional
{
    // Map
    public static TResult[] Map<T, TResult>(T[] source, Func<T, TResult> transform)
    {
        var result = new TResult[source.Length];
        for (int i = 0; i < source.Length; i++)
            result[i] = transform(source[i]);
        return result;
    }
    
    // Filter
    public static T[] Filter<T>(T[] source, Func<T, bool> predicate)
        => source.Where(predicate).ToArray();
    
    // Reduce/Fold
    public static TResult Reduce<T, TResult>(T[] source, TResult seed, Func<TResult, T, TResult> accumulate)
        => source.Aggregate(seed, accumulate);
    
    // Zip
    public static TResult[] Zip<T1, T2, TResult>(T1[] a, T2[] b, Func<T1, T2, TResult> combine)
    {
        int len = Math.Min(a.Length, b.Length);
        var result = new TResult[len];
        for (int i = 0; i < len; i++)
            result[i] = combine(a[i], b[i]);
        return result;
    }
    
    // GroupBy
    public static Dictionary<TKey, List<T>> GroupBy<T, TKey>(T[] source, Func<T, TKey> keySelector)
        where TKey : notnull
    {
        var result = new Dictionary<TKey, List<T>>();
        foreach (var item in source)
        {
            var key = keySelector(item);
            if (!result.TryGetValue(key, out var list))
                result[key] = list = new List<T>();
            list.Add(item);
        }
        return result;
    }
    
    // Flatten
    public static T[] Flatten<T>(T[][] source)
        => source.SelectMany(x => x).ToArray();
}

int[] arr = { 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };

var doubled2 = Functional.Map(arr, x => x * 2);
var evens2 = Functional.Filter(arr, x => x % 2 == 0);
var total = Functional.Reduce(arr, 0, (acc, n) => acc + n);
var grouped = Functional.GroupBy(arr, n => n % 2 == 0 ? "even" : "odd");

Console.WriteLine($"\nDoubled: {string.Join(", ", doubled2)}");
Console.WriteLine($"Evens: {string.Join(", ", evens2)}");
Console.WriteLine($"Total: {total}");
foreach (var (key, items) in grouped)
    Console.WriteLine($"{key}: {string.Join(", ", items)}");

int[] prices = { 100, 200, 300 };
int[] qty = { 3, 2, 5 };
var revenue = Functional.Zip(prices, qty, (p, q) => p * q);
Console.WriteLine($"Revenue: {string.Join(", ", revenue)}");
```

---

## Step 301-320: โปรแกรม Generic Data Pipeline

```csharp
// ============ Generic Data Pipeline ============

// Pipeline Step Interface
interface IPipelineStep<TInput, TOutput>
{
    TOutput Process(TInput input);
    string Name { get; }
}

// Pipeline Builder
class Pipeline<T>
{
    private readonly List<Func<T, T>> _steps = new();
    private readonly List<string> _stepNames = new();
    
    public Pipeline<T> AddStep(string name, Func<T, T> transform)
    {
        _steps.Add(transform);
        _stepNames.Add(name);
        return this;
    }
    
    public T Execute(T input)
    {
        T current = input;
        for (int i = 0; i < _steps.Count; i++)
        {
            Console.WriteLine($"  Step {i+1}: {_stepNames[i]}");
            current = _steps[i](current);
        }
        return current;
    }
    
    public IEnumerable<T> ExecuteMany(IEnumerable<T> inputs)
        => inputs.Select(Execute);
}

// Generic Result type
class Result<T>
{
    public bool IsSuccess { get; }
    public T? Value { get; }
    public string? Error { get; }
    
    private Result(bool success, T? value, string? error)
    {
        IsSuccess = success;
        Value = value;
        Error = error;
    }
    
    public static Result<T> Success(T value) => new(true, value, null);
    public static Result<T> Failure(string error) => new(false, default, error);
    
    public Result<TResult> Map<TResult>(Func<T, TResult> transform)
    {
        if (!IsSuccess) return Result<TResult>.Failure(Error!);
        try { return Result<TResult>.Success(transform(Value!)); }
        catch (Exception ex) { return Result<TResult>.Failure(ex.Message); }
    }
    
    public Result<TResult> Bind<TResult>(Func<T, Result<TResult>> transform)
    {
        if (!IsSuccess) return Result<TResult>.Failure(Error!);
        return transform(Value!);
    }
    
    public T GetOrDefault(T defaultValue) => IsSuccess ? Value! : defaultValue;
    public T GetOrThrow() => IsSuccess ? Value! : throw new InvalidOperationException(Error);
    
    public override string ToString() =>
        IsSuccess ? $"Success({Value})" : $"Failure({Error})";
}

// Generic Option type (Maybe monad)
class Option<T>
{
    private readonly T? _value;
    public bool HasValue { get; }
    
    private Option(T? value, bool hasValue)
    {
        _value = value;
        HasValue = hasValue;
    }
    
    public static Option<T> Some(T value) => new(value, true);
    public static Option<T> None() => new(default, false);
    
    public Option<TResult> Map<TResult>(Func<T, TResult> transform)
        => HasValue ? Option<TResult>.Some(transform(_value!)) : Option<TResult>.None();
    
    public Option<TResult> Bind<TResult>(Func<T, Option<TResult>> transform)
        => HasValue ? transform(_value!) : Option<TResult>.None();
    
    public T GetOrDefault(T defaultValue) => HasValue ? _value! : defaultValue;
    
    public void IfPresent(Action<T> action) { if (HasValue) action(_value!); }
    
    public override string ToString() => HasValue ? $"Some({_value})" : "None";
}

// ============ Using the Generic Types ============

// Result monad
Result<int> ParseInt(string s)
{
    return int.TryParse(s, out int val)
        ? Result<int>.Success(val)
        : Result<int>.Failure($"Cannot parse '{s}' as int");
}

Result<double> Sqrt(int n)
{
    return n >= 0
        ? Result<double>.Success(Math.Sqrt(n))
        : Result<double>.Failure($"Cannot take sqrt of negative number {n}");
}

// Chain operations
var result = ParseInt("16")
    .Bind(Sqrt)
    .Map(x => $"sqrt = {x:F2}");

Console.WriteLine($"Result: {result}");

var failResult = ParseInt("abc")
    .Bind(Sqrt)
    .Map(x => $"sqrt = {x:F2}");

Console.WriteLine($"Fail: {failResult}");

// Pipeline
var textPipeline = new Pipeline<string>()
    .AddStep("Trim", s => s.Trim())
    .AddStep("ToLower", s => s.ToLower())
    .AddStep("Remove punctuation", s => new string(s.Where(c => char.IsLetterOrDigit(c) || c == ' ').ToArray()))
    .AddStep("Normalize spaces", s => string.Join(" ", s.Split(' ', StringSplitOptions.RemoveEmptyEntries)));

Console.WriteLine("\nText Pipeline:");
string input = "  Hello,   World!   ";
Console.WriteLine($"Input: '{input}'");
string output = textPipeline.Execute(input);
Console.WriteLine($"Output: '{output}'");

// Number pipeline
var numPipeline = new Pipeline<int[]>()
    .AddStep("Filter positive", arr => arr.Where(n => n > 0).ToArray())
    .AddStep("Square", arr => arr.Select(n => n * n).ToArray())
    .AddStep("Sort desc", arr => arr.OrderByDescending(n => n).ToArray())
    .AddStep("Take top 5", arr => arr.Take(5).ToArray());

int[] numbers = { -3, 5, -1, 8, 2, -7, 9, 4, 6, 1, -5, 7, 3 };
Console.WriteLine($"\nNumber Pipeline:");
Console.WriteLine($"Input: [{string.Join(", ", numbers)}]");
int[] result2 = numPipeline.Execute(numbers);
Console.WriteLine($"Output: [{string.Join(", ", result2)}]");

// Option monad
var users = new Dictionary<int, string>
{
    { 1, "Alice" },
    { 2, "Bob" },
    { 3, "Charlie" }
};

Option<string> FindUser(int id)
    => users.TryGetValue(id, out string? name)
        ? Option<string>.Some(name)
        : Option<string>.None();

var userGreeting = FindUser(1)
    .Map(name => $"Hello, {name}!")
    .GetOrDefault("User not found");
Console.WriteLine($"\nUser 1: {userGreeting}");

var notFound = FindUser(99)
    .Map(name => $"Hello, {name}!")
    .GetOrDefault("User not found");
Console.WriteLine($"User 99: {notFound}");

FindUser(2).IfPresent(name => Console.WriteLine($"Found: {name}"));

// ============ Generic Caching ============
class Cache<TKey, TValue> where TKey : notnull
{
    private readonly Dictionary<TKey, (TValue Value, DateTime Expiry)> _cache = new();
    private readonly TimeSpan _defaultExpiry;
    
    public Cache(TimeSpan? defaultExpiry = null)
    {
        _defaultExpiry = defaultExpiry ?? TimeSpan.FromMinutes(5);
    }
    
    public void Set(TKey key, TValue value, TimeSpan? expiry = null)
    {
        DateTime expiryTime = DateTime.Now + (expiry ?? _defaultExpiry);
        _cache[key] = (value, expiryTime);
    }
    
    public Option<TValue> Get(TKey key)
    {
        if (!_cache.TryGetValue(key, out var entry)) return Option<TValue>.None();
        if (DateTime.Now > entry.Expiry)
        {
            _cache.Remove(key);
            return Option<TValue>.None();
        }
        return Option<TValue>.Some(entry.Value);
    }
    
    public TValue GetOrSet(TKey key, Func<TValue> factory, TimeSpan? expiry = null)
    {
        var cached = Get(key);
        if (cached.HasValue) return cached.GetOrDefault(default!);
        
        TValue value = factory();
        Set(key, value, expiry);
        return value;
    }
    
    public void Invalidate(TKey key) => _cache.Remove(key);
    
    public void Clear() => _cache.Clear();
    
    public int Count => _cache.Count(e => DateTime.Now <= e.Value.Expiry);
}

var cache = new Cache<string, string>(TimeSpan.FromSeconds(30));
cache.Set("greeting", "Hello, World!");
cache.Set("farewell", "Goodbye!", TimeSpan.FromSeconds(1));

var greeting = cache.Get("greeting");
Console.WriteLine($"\nCache 'greeting': {greeting}");

var data = cache.GetOrSet("computed", () =>
{
    Console.WriteLine("Computing expensive value...");
    return "Computed Result";
});
Console.WriteLine($"Computed: {data}");

// Get again - should be cached
data = cache.GetOrSet("computed", () =>
{
    Console.WriteLine("This should NOT appear (cached)");
    return "New Result";
});
Console.WriteLine($"From cache: {data}");
```

---

## สรุป Part 11

```
✅ Step 291: Generic Types พื้นฐาน (Methods, Classes)
✅ Step 292: Generic Constraints (7+ ชนิด)
✅ Step 293: Generic Data Structures (Stack, Queue, BST)
✅ Steps 294-300: Generic Algorithms (Sort, Search, Functional)
✅ Steps 301-320: Generic Data Pipeline (Result, Option, Cache)
```

## แบบฝึกหัด Part 11

1. สร้าง Generic `Graph<T>` พร้อม BFS และ DFS
2. Implement `Either<TLeft, TRight>` monad
3. สร้าง Generic `LRUCache<TKey, TValue>`
4. เขียน Generic `Trie<T>` สำหรับ string lookups
5. สร้าง `TypedEventBus<TEvent>` ที่ type-safe

---

**ถัดไป: [Part 12 - LINQ →](part-12-linq.md)**
