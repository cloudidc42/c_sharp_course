# Part 05: เมธอดและฟังก์ชัน (Methods & Functions)
## Steps 81-110: การสร้างและใช้งาน Methods ใน C#

---

## Step 81: การประกาศ Method

```csharp
// ============ Method Syntax ============
// [access modifier] [return type] MethodName([parameters])
// {
//     // body
//     return value;
// }

// Method ที่ไม่คืนค่า (void)
void PrintMessage(string message)
{
    Console.WriteLine($"[MSG] {message}");
}

// Method ที่คืนค่า
int Add(int a, int b)
{
    return a + b;
}

// Method แบบ Expression-bodied (C# 6+)
int Multiply(int a, int b) => a * b;
string Greet(string name) => $"สวัสดี, {name}!";

// เรียกใช้
PrintMessage("Hello World");
int sum = Add(10, 20);
Console.WriteLine($"Sum: {sum}");
Console.WriteLine($"Multiply: {Multiply(5, 6)}");
Console.WriteLine(Greet("Alice"));

// ============ Access Modifiers ============
class Calculator
{
    // public: ใช้ได้จากทุกที่
    public int Add(int a, int b) => a + b;
    
    // private: ใช้ได้เฉพาะใน class นี้
    private double Divide(double a, double b) => a / b;
    
    // protected: ใช้ได้ใน class นี้และ subclass
    protected int Subtract(int a, int b) => a - b;
    
    // internal: ใช้ได้ใน assembly เดียวกัน
    internal double SafeDivide(double a, double b)
    {
        if (b == 0) throw new DivideByZeroException();
        return Divide(a, b); // เรียก private method ได้
    }
    
    // static: ไม่ต้องสร้าง instance
    public static int Square(int n) => n * n;
}

var calc = new Calculator();
Console.WriteLine(calc.Add(10, 5));
Console.WriteLine(calc.SafeDivide(10, 3));
Console.WriteLine(Calculator.Square(7)); // เรียก static

// ============ Return Multiple Values ============
// วิธีที่ 1: Tuple
(int Min, int Max, double Average) GetStats(int[] numbers)
{
    return (numbers.Min(), numbers.Max(), numbers.Average());
}

int[] data = { 5, 3, 8, 1, 9, 2, 7 };
var (min, max, avg) = GetStats(data);
Console.WriteLine($"Min: {min}, Max: {max}, Avg: {avg:F2}");

// วิธีที่ 2: out parameters
bool TryDivide(int a, int b, out double result, out string error)
{
    if (b == 0)
    {
        result = 0;
        error = "ไม่สามารถหารด้วยศูนย์";
        return false;
    }
    result = (double)a / b;
    error = string.Empty;
    return true;
}

if (TryDivide(10, 3, out double divResult, out string divError))
    Console.WriteLine($"Result: {divResult:F4}");
else
    Console.WriteLine($"Error: {divError}");
```

---

## Step 82: Parameters ชนิดต่างๆ

```csharp
// ============ 1. Required Parameters ============
int AddTwo(int a, int b) => a + b;
Console.WriteLine(AddTwo(5, 3)); // 8

// ============ 2. Optional Parameters (Default Values) ============
void PrintInfo(string name, int age = 0, string city = "Unknown")
{
    Console.WriteLine($"{name}, Age: {age}, City: {city}");
}

PrintInfo("Alice");           // Alice, Age: 0, City: Unknown
PrintInfo("Bob", 25);         // Bob, Age: 25, City: Unknown
PrintInfo("Charlie", 30, "Bangkok"); // Charlie, Age: 30, City: Bangkok

// ✅ Optional parameters ต้องอยู่หลัง required parameters
// ❌ void Test(int a = 0, int b) // Error!

// ============ 3. Named Arguments ============
PrintInfo(name: "Dave", city: "Chiang Mai", age: 28);
PrintInfo(city: "Phuket", name: "Eve"); // เปลี่ยนลำดับได้

// ============ 4. params (Variable Arguments) ============
int Sum(params int[] numbers)
{
    int total = 0;
    foreach (int n in numbers)
        total += n;
    return total;
}

Console.WriteLine(Sum());              // 0
Console.WriteLine(Sum(1, 2, 3));       // 6
Console.WriteLine(Sum(1, 2, 3, 4, 5)); // 15

int[] arr = { 10, 20, 30 };
Console.WriteLine(Sum(arr));           // 60 (ส่ง array ได้)

// params IEnumerable (C# 13+)
void PrintAll(params IEnumerable<string> items)
{
    foreach (var item in items) Console.Write($"{item} | ");
    Console.WriteLine();
}
PrintAll("Apple", "Banana", "Cherry");

// ============ 5. ref Parameters ============
// ส่งด้วย reference - method เปลี่ยนค่าได้
void Increment(ref int value)
{
    value++;
}

int x = 5;
Increment(ref x);
Console.WriteLine($"After Increment: {x}"); // 6

void Swap(ref int a, ref int b)
{
    (a, b) = (b, a);
}

int p = 10, q = 20;
Swap(ref p, ref q);
Console.WriteLine($"After Swap: p={p}, q={q}"); // p=20, q=10

// ============ 6. out Parameters ============
// ต้องกำหนดค่าใน method
bool TryParseInt(string s, out int result)
{
    return int.TryParse(s, out result);
}

if (TryParseInt("42", out int parsed))
    Console.WriteLine($"Parsed: {parsed}");

// out var (inline declaration)
if (int.TryParse("100", out var val))
    Console.WriteLine($"Inline out: {val}");

// ============ 7. in Parameters (C# 7.2+) ============
// ส่ง reference แต่ read-only (ประสิทธิภาพสูงสำหรับ struct ใหญ่)
struct LargeStruct
{
    public double X, Y, Z, W;
    public LargeStruct(double x, double y, double z, double w)
    { X = x; Y = y; Z = z; W = w; }
}

double ComputeLength(in LargeStruct s)
{
    // ✅ อ่านได้
    return Math.Sqrt(s.X * s.X + s.Y * s.Y + s.Z * s.Z + s.W * s.W);
    // ❌ s.X = 0; // Error! ไม่สามารถเปลี่ยนค่าได้
}

var large = new LargeStruct(1, 2, 3, 4);
Console.WriteLine($"Length: {ComputeLength(in large):F4}");

// ============ 8. ref readonly (C# 12) ============
ref readonly int GetFirst(int[] arr) => ref arr[0];

int[] numbers = { 10, 20, 30 };
ref readonly int first = ref GetFirst(numbers);
Console.WriteLine($"First (ref readonly): {first}"); // 10
// ❌ first = 99; // Error!
```

---

## Step 83: Method Overloading

```csharp
// ============ Method Overloading ============
// ชื่อ method เดียวกัน แต่ parameters ต่างกัน

class Printer
{
    // Overload 1: int
    public void Print(int value)
        => Console.WriteLine($"[int] {value}");
    
    // Overload 2: double
    public void Print(double value)
        => Console.WriteLine($"[double] {value:F4}");
    
    // Overload 3: string
    public void Print(string value)
        => Console.WriteLine($"[string] '{value}'");
    
    // Overload 4: int[]
    public void Print(int[] values)
        => Console.WriteLine($"[array] [{string.Join(", ", values)}]");
    
    // Overload 5: สองพารามิเตอร์
    public void Print(string label, int value)
        => Console.WriteLine($"[{label}] = {value}");
    
    // Overload 6: bool
    public void Print(bool value)
        => Console.WriteLine($"[bool] {value}");
}

var printer = new Printer();
printer.Print(42);
printer.Print(3.14);
printer.Print("Hello");
printer.Print(new[] { 1, 2, 3 });
printer.Print("Score", 95);
printer.Print(true);

// ============ Overloading Rules ============
// ✅ ต่าง: จำนวน parameters
// ✅ ต่าง: ชนิด parameters
// ✅ ต่าง: ลำดับ parameters (ถ้าต่างชนิด)
// ❌ เหมือน: เฉพาะ return type ต่างกัน (Error!)

// ============ Operator vs Method Overloading ============
class Temperature
{
    public double Celsius { get; }
    
    public Temperature(double celsius) => Celsius = celsius;
    
    // Overload ToString
    public override string ToString() => $"{Celsius}°C";
    
    // Overload +
    public static Temperature operator +(Temperature a, Temperature b)
        => new Temperature(a.Celsius + b.Celsius);
    
    // Overload implicit conversion
    public static implicit operator double(Temperature t) => t.Celsius;
    public static explicit operator Temperature(double d) => new Temperature(d);
}

var t1 = new Temperature(25);
var t2 = new Temperature(10);
var t3 = t1 + t2;
Console.WriteLine($"{t1} + {t2} = {t3}");

double val = t1; // Implicit
Console.WriteLine($"As double: {val}");

Temperature t4 = (Temperature)37.5; // Explicit
Console.WriteLine($"As Temperature: {t4}");

// ============ Generic Method Overloading ============
class TypeChecker
{
    // ทำงานกับ any type
    public static string GetTypeName<T>() => typeof(T).Name;
    
    public static bool IsNull<T>(T value) where T : class
        => value is null;
    
    public static T Default<T>() => default!;
}

Console.WriteLine(TypeChecker.GetTypeName<int>());     // Int32
Console.WriteLine(TypeChecker.GetTypeName<string>());  // String
Console.WriteLine(TypeChecker.GetTypeName<List<int>>()); // List`1
```

---

## Step 84: Recursion

```csharp
// ============ Basic Recursion ============

// Factorial n!
long Factorial(int n)
{
    if (n <= 1) return 1;     // Base case
    return n * Factorial(n - 1); // Recursive case
}

for (int i = 0; i <= 10; i++)
    Console.WriteLine($"{i}! = {Factorial(i)}");

// ============ Fibonacci (Recursive) ============
long FibRecursive(int n)
{
    if (n <= 1) return n;
    return FibRecursive(n - 1) + FibRecursive(n - 2);
}

// ⚠️ ช้ามากสำหรับ n ใหญ่ เพราะคำนวณซ้ำ

// Fibonacci กับ Memoization (Dynamic Programming)
Dictionary<int, long> memo = new();
long FibMemo(int n)
{
    if (n <= 1) return n;
    if (memo.ContainsKey(n)) return memo[n]; // Cache hit
    
    long result = FibMemo(n - 1) + FibMemo(n - 2);
    memo[n] = result; // Cache result
    return result;
}

Console.WriteLine($"\nFib(40) = {FibMemo(40)}");
Console.WriteLine($"Fib(50) = {FibMemo(50)}");

// ============ Tower of Hanoi ============
int moves = 0;
void Hanoi(int n, string from, string to, string via)
{
    if (n == 1)
    {
        Console.WriteLine($"Move disk 1 from {from} to {to}");
        moves++;
        return;
    }
    Hanoi(n - 1, from, via, to);
    Console.WriteLine($"Move disk {n} from {from} to {to}");
    moves++;
    Hanoi(n - 1, via, to, from);
}

Console.WriteLine("\nTower of Hanoi (3 disks):");
Hanoi(3, "A", "C", "B");
Console.WriteLine($"Total moves: {moves}"); // 2^3 - 1 = 7

// ============ Tree Traversal ============
class TreeNode
{
    public int Value;
    public TreeNode? Left, Right;
    
    public TreeNode(int val) => Value = val;
}

class BinarySearchTree
{
    public TreeNode? Root;
    
    public void Insert(int val)
    {
        Root = Insert(Root, val);
    }
    
    private TreeNode Insert(TreeNode? node, int val)
    {
        if (node is null) return new TreeNode(val);
        if (val < node.Value) node.Left = Insert(node.Left, val);
        else if (val > node.Value) node.Right = Insert(node.Right, val);
        return node;
    }
    
    // In-order traversal (sorted)
    public IEnumerable<int> InOrder() => InOrder(Root);
    
    private IEnumerable<int> InOrder(TreeNode? node)
    {
        if (node is null) yield break;
        foreach (int val in InOrder(node.Left)) yield return val;
        yield return node.Value;
        foreach (int val in InOrder(node.Right)) yield return val;
    }
    
    // Search
    public bool Contains(int val) => Contains(Root, val);
    private bool Contains(TreeNode? node, int val)
    {
        if (node is null) return false;
        if (val == node.Value) return true;
        return val < node.Value ? Contains(node.Left, val) : Contains(node.Right, val);
    }
}

var bst = new BinarySearchTree();
foreach (int val in new[] { 5, 3, 7, 1, 4, 6, 8, 2 })
    bst.Insert(val);

Console.WriteLine("\nBST In-order (sorted):");
Console.WriteLine(string.Join(", ", bst.InOrder()));

Console.WriteLine($"Contains 4: {bst.Contains(4)}");
Console.WriteLine($"Contains 9: {bst.Contains(9)}");

// ============ Tail Recursion Optimization ============
// C# ไม่ optimize tail recursion โดยอัตโนมัติ
// ใช้ iteration แทนถ้าเป็นไปได้

long FactorialIterative(int n)
{
    long result = 1;
    for (int i = 2; i <= n; i++)
        result *= i;
    return result;
}

Console.WriteLine($"\n20! = {FactorialIterative(20)}");
```

---

## Step 85: Local Functions

```csharp
// ============ Local Functions (C# 7+) ============
// ประกาศ function ภายใน method อื่น

void ProcessData(int[] data)
{
    // Local function - ใช้ได้เฉพาะใน ProcessData
    int Square(int n) => n * n;
    bool IsEven(int n) => n % 2 == 0;
    
    Console.WriteLine("Processing:");
    foreach (int num in data)
    {
        string label = IsEven(num) ? "Even" : "Odd";
        Console.WriteLine($"  {num} ({label}) -> squared: {Square(num)}");
    }
}

ProcessData(new[] { 1, 2, 3, 4, 5 });

// ============ Local Function กับ Closure ============
int multiplier = 3;

// Local function สามารถ capture ตัวแปร outer ได้
IEnumerable<int> MultiplyAll(IEnumerable<int> numbers)
{
    return numbers.Select(Apply);
    
    int Apply(int n) => n * multiplier; // Captures multiplier
}

Console.WriteLine("\nMultiply by 3:");
foreach (int n in MultiplyAll(new[] { 1, 2, 3, 4, 5 }))
    Console.Write($"{n} ");
Console.WriteLine();

// ============ static Local Function (C# 8+) ============
// ไม่สามารถ capture ตัวแปร outer ได้ (ประสิทธิภาพดีกว่า)

int x = 10;

void Demo()
{
    // ❌ static int Add1(int n) => n + x; // Error! Cannot capture x
    static int Add1(int n) => n + 1; // OK - ไม่ capture อะไร
    static int Double(int n) => n * 2; // OK
    
    Console.WriteLine(Add1(5));
    Console.WriteLine(Double(7));
}
Demo();

// ============ Recursive Local Function ============
int[] SortArray(int[] arr)
{
    // Quick Sort as local function
    void QuickSort(int[] array, int low, int high)
    {
        if (low < high)
        {
            int pi = Partition(array, low, high);
            QuickSort(array, low, pi - 1);
            QuickSort(array, pi + 1, high);
        }
    }
    
    int Partition(int[] array, int low, int high)
    {
        int pivot = array[high];
        int i = low - 1;
        
        for (int j = low; j < high; j++)
        {
            if (array[j] <= pivot)
            {
                i++;
                (array[i], array[j]) = (array[j], array[i]);
            }
        }
        (array[i + 1], array[high]) = (array[high], array[i + 1]);
        return i + 1;
    }
    
    int[] result = (int[])arr.Clone();
    QuickSort(result, 0, result.Length - 1);
    return result;
}

int[] unsorted = { 64, 34, 25, 12, 22, 11, 90 };
int[] sorted = SortArray(unsorted);
Console.WriteLine($"\nSorted: {string.Join(", ", sorted)}");
```

---

## Step 86-90: Extension Methods

```csharp
// ============ Extension Methods ============
// เพิ่ม method ให้ existing type โดยไม่ต้องแก้ไข source code

// Syntax: static method ใน static class, first param มี 'this'
public static class StringExtensions
{
    // Extension method สำหรับ string
    public static bool IsNullOrEmpty(this string? str)
        => string.IsNullOrEmpty(str);
    
    public static string Truncate(this string str, int maxLength, string suffix = "...")
    {
        if (str.Length <= maxLength) return str;
        return str[..(maxLength - suffix.Length)] + suffix;
    }
    
    public static string ToTitleCase(this string str)
    {
        if (string.IsNullOrEmpty(str)) return str;
        return System.Globalization.CultureInfo.CurrentCulture.TextInfo.ToTitleCase(str.ToLower());
    }
    
    public static bool IsNumeric(this string str)
        => double.TryParse(str, out _);
    
    public static int WordCount(this string str)
        => string.IsNullOrWhiteSpace(str) ? 0 : str.Trim().Split(' ', StringSplitOptions.RemoveEmptyEntries).Length;
    
    public static string Repeat(this string str, int count)
        => string.Concat(Enumerable.Repeat(str, count));
    
    public static string Reverse(this string str)
        => new string(str.Reverse().ToArray());
    
    public static bool IsPalindrome(this string str)
    {
        string clean = new string(str.Where(char.IsLetterOrDigit).ToArray()).ToLower();
        return clean == clean.Reverse();
    }
}

public static class IntExtensions
{
    public static bool IsEven(this int n) => n % 2 == 0;
    public static bool IsOdd(this int n) => n % 2 != 0;
    public static bool IsPrime(this int n)
    {
        if (n < 2) return false;
        for (int i = 2; i * i <= n; i++)
            if (n % i == 0) return false;
        return true;
    }
    public static IEnumerable<int> To(this int start, int end)
    {
        for (int i = start; i <= end; i++) yield return i;
    }
    public static string Times(this int count, string text)
        => string.Concat(Enumerable.Repeat(text, count));
}

public static class EnumerableExtensions
{
    public static T? Median<T>(this IEnumerable<T> source) where T : IComparable<T>
    {
        var sorted = source.OrderBy(x => x).ToList();
        int count = sorted.Count;
        if (count == 0) return default;
        return sorted[count / 2];
    }
    
    public static IEnumerable<IEnumerable<T>> Chunk<T>(this IEnumerable<T> source, int size)
    {
        var list = source.ToList();
        for (int i = 0; i < list.Count; i += size)
            yield return list.Skip(i).Take(size);
    }
    
    public static Dictionary<TKey, int> CountBy<T, TKey>(
        this IEnumerable<T> source, 
        Func<T, TKey> keySelector) where TKey : notnull
    {
        return source.GroupBy(keySelector).ToDictionary(g => g.Key, g => g.Count());
    }
}

// ใช้งาน Extension Methods
string text = "hello world, this is c sharp";
Console.WriteLine(text.ToTitleCase());          // Hello World, This Is C Sharp
Console.WriteLine(text.WordCount());            // 6
Console.WriteLine(text.Truncate(15));           // hello world,...
Console.WriteLine("=".Repeat(30));
Console.WriteLine("racecar".IsPalindrome());    // True
Console.WriteLine("hello".IsPalindrome());      // False
Console.WriteLine("A man a plan a canal Panama".IsPalindrome()); // True

int n = 7;
Console.WriteLine($"{n} is prime: {n.IsPrime()}");  // True
Console.WriteLine($"{n} is even: {n.IsEven()}");    // False
Console.Write("Range 1..5: ");
foreach (int i in 1.To(5)) Console.Write($"{i} ");
Console.WriteLine();
Console.WriteLine(3.Times("Ha"));  // HaHaHa

int[] numbers = { 5, 2, 8, 1, 9, 3, 7, 4, 6 };
Console.WriteLine($"Median: {numbers.Median()}");

string[] words = { "apple", "banana", "cherry", "avocado", "blueberry", "apricot" };
var grouped = words.CountBy(w => w[0]);
foreach (var (letter, count) in grouped.OrderBy(kv => kv.Key))
    Console.WriteLine($"  '{letter}': {count} words");

Console.WriteLine("\nChunks of 3:");
foreach (var chunk in numbers.Chunk(3))
    Console.WriteLine($"  [{string.Join(", ", chunk)}]");
```

---

## Step 91-100: Lambda Expressions และ Functional Programming Basics

```csharp
// ============ Lambda Expressions ============
// (parameters) => expression  หรือ  (parameters) => { statements }

// Basic lambda
Func<int, int> square = x => x * x;
Func<int, int, int> add = (a, b) => a + b;
Func<string, bool> isLong = s => s.Length > 5;
Action<string> print = s => Console.WriteLine(s);
Action<int, int> printSum = (a, b) => Console.WriteLine($"{a} + {b} = {a + b}");

Console.WriteLine(square(5));          // 25
Console.WriteLine(add(3, 4));          // 7
Console.WriteLine(isLong("Hello"));    // False
print("Lambda works!");
printSum(10, 20);

// Multi-statement lambda
Func<int[], int[]> filterAndDouble = numbers =>
{
    var filtered = numbers.Where(n => n > 0).ToArray();
    return filtered.Select(n => n * 2).ToArray();
};

int[] input = { -1, 2, -3, 4, 5, -6 };
int[] result = filterAndDouble(input);
Console.WriteLine($"Filter & Double: {string.Join(", ", result)}");

// ============ Lambda กับ LINQ ============
var people = new[]
{
    new { Name = "Alice", Age = 30, Score = 95 },
    new { Name = "Bob", Age = 25, Score = 78 },
    new { Name = "Charlie", Age = 35, Score = 88 },
    new { Name = "Dave", Age = 28, Score = 92 },
    new { Name = "Eve", Age = 32, Score = 65 },
};

// Where (filter)
var youngsters = people.Where(p => p.Age < 30);
Console.WriteLine("\nUnder 30:");
foreach (var p in youngsters) Console.WriteLine($"  {p.Name} ({p.Age})");

// OrderBy
var byScore = people.OrderByDescending(p => p.Score);
Console.WriteLine("\nBy Score:");
foreach (var p in byScore) Console.WriteLine($"  {p.Name}: {p.Score}");

// Select (transform)
var names = people.Select(p => p.Name.ToUpper());
Console.WriteLine($"\nNames: {string.Join(", ", names)}");

// Aggregate
var totalScore = people.Sum(p => p.Score);
var avgAge = people.Average(p => p.Age);
Console.WriteLine($"Total score: {totalScore}, Avg age: {avgAge:F1}");

// ============ Closures ============
// Lambda ที่ capture ตัวแปร outer

int multiplier = 3;
Func<int, int> tripler = x => x * multiplier; // Captures multiplier

Console.WriteLine(tripler(5));  // 15
multiplier = 4;
Console.WriteLine(tripler(5));  // 20 (ใช้ค่าล่าสุดของ multiplier!)

// ✅ ระวัง: closure ใน loop
var actions = new List<Action>();
for (int i = 0; i < 5; i++)
{
    int captured = i; // ✅ แก้ปัญหาด้วยการ copy ค่า
    actions.Add(() => Console.Write($"{captured} "));
}
foreach (var action in actions) action();
Console.WriteLine();

// ❌ Bug: ถ้าไม่ copy i ทุก action จะใช้ค่า i สุดท้าย (5)
var buggyActions = new List<Action>();
for (int i = 0; i < 5; i++)
{
    buggyActions.Add(() => Console.Write($"{i} ")); // Captures i by reference
}
foreach (var action in buggyActions) action(); // 5 5 5 5 5
Console.WriteLine();

// ============ Currying ============
// แปลง function หลายพารามิเตอร์เป็น chain of single-param functions

Func<int, Func<int, int>> Curry(Func<int, int, int> f)
    => a => b => f(a, b);

var curriedAdd = Curry((a, b) => a + b);
var add5 = curriedAdd(5);     // function ที่บวก 5
var add10 = curriedAdd(10);   // function ที่บวก 10

Console.WriteLine($"\nCurried: add5(3)={add5(3)}, add10(3)={add10(3)}");

// ============ Function Composition ============
Func<T2, T3> Compose<T1, T2, T3>(Func<T1, T2> f, Func<T2, T3> g)
    => x => g(f(x));

Func<int, int> doubler = x => x * 2;
Func<int, string> toStr = x => $"Value: {x}";
Func<string, string> exclaim = s => s + "!";

var doubleToString = Compose(doubler, toStr);
var doubleToStringExclaim = Compose(doubleToString, exclaim);

Console.WriteLine(doubleToString(7));          // Value: 14
Console.WriteLine(doubleToStringExclaim(7));   // Value: 14!

// ============ Higher-Order Functions ============
// Functions ที่รับ function เป็น parameter หรือคืน function

// Map (Select)
IEnumerable<TResult> Map<T, TResult>(IEnumerable<T> source, Func<T, TResult> transform)
    => source.Select(transform);

// Filter (Where)
IEnumerable<T> Filter<T>(IEnumerable<T> source, Func<T, bool> predicate)
    => source.Where(predicate);

// Reduce (Aggregate)
TResult Reduce<T, TResult>(IEnumerable<T> source, TResult seed, Func<TResult, T, TResult> accumulate)
    => source.Aggregate(seed, accumulate);

int[] nums = { 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };

var doubled = Map(nums, x => x * 2);
var evens = Filter(nums, x => x % 2 == 0);
var sumResult = Reduce(nums, 0, (acc, n) => acc + n);
var product = Reduce(nums, 1, (acc, n) => acc * n);

Console.WriteLine($"\nDoubled: {string.Join(", ", doubled)}");
Console.WriteLine($"Evens: {string.Join(", ", evens)}");
Console.WriteLine($"Sum: {sumResult}");
Console.WriteLine($"Product: {product}");
```

---

## Step 101-110: โปรแกรม Text Processor สมบูรณ์

```csharp
// ============ Text Processor ============
// ใช้ methods, extension methods, และ lambda

using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Text.RegularExpressions;

class TextProcessor
{
    private string _text;
    
    public TextProcessor(string text) => _text = text;
    
    // ============ Analysis ============
    public TextStats Analyze() => new TextStats
    {
        CharCount = _text.Length,
        WordCount = CountWords(),
        SentenceCount = CountSentences(),
        ParagraphCount = CountParagraphs(),
        UniqueWords = GetUniqueWords().Count(),
        AvgWordLength = CalculateAvgWordLength(),
        LongestWord = FindLongestWord(),
        MostFrequentWord = FindMostFrequentWord()
    };
    
    public int CountWords() =>
        string.IsNullOrWhiteSpace(_text) ? 0 :
        _text.Trim()
             .Split(new[] { ' ', '\t', '\n', '\r' }, StringSplitOptions.RemoveEmptyEntries)
             .Length;
    
    public int CountSentences() =>
        _text.Count(c => c is '.' or '!' or '?');
    
    public int CountParagraphs() =>
        _text.Split(new[] { "\n\n", "\r\n\r\n" }, StringSplitOptions.RemoveEmptyEntries).Length;
    
    public IEnumerable<string> GetUniqueWords() =>
        GetWords().Distinct(StringComparer.OrdinalIgnoreCase);
    
    public IEnumerable<string> GetWords() =>
        _text.Split(new[] { ' ', '\t', '\n', '\r', '.', ',', '!', '?', ';', ':' },
                    StringSplitOptions.RemoveEmptyEntries)
             .Select(w => w.Trim('\'', '"', '(', ')'));
    
    public double CalculateAvgWordLength()
    {
        var words = GetWords().ToList();
        return words.Count == 0 ? 0 : words.Average(w => w.Length);
    }
    
    public string FindLongestWord() =>
        GetWords().MaxBy(w => w.Length) ?? "";
    
    public string FindMostFrequentWord() =>
        GetWords()
            .GroupBy(w => w.ToLower())
            .OrderByDescending(g => g.Count())
            .FirstOrDefault()?.Key ?? "";
    
    // ============ Transform ============
    public TextProcessor ToUpper() => new TextProcessor(_text.ToUpper());
    public TextProcessor ToLower() => new TextProcessor(_text.ToLower());
    
    public TextProcessor Replace(string oldValue, string newValue)
        => new TextProcessor(_text.Replace(oldValue, newValue));
    
    public TextProcessor RemoveExtraSpaces()
        => new TextProcessor(Regex.Replace(_text, @"\s+", " ").Trim());
    
    public TextProcessor WrapLines(int maxWidth)
    {
        var words = _text.Split(' ');
        var sb = new StringBuilder();
        int lineLength = 0;
        
        foreach (string word in words)
        {
            if (lineLength + word.Length + 1 > maxWidth)
            {
                sb.AppendLine();
                lineLength = 0;
            }
            else if (lineLength > 0)
            {
                sb.Append(' ');
                lineLength++;
            }
            sb.Append(word);
            lineLength += word.Length;
        }
        
        return new TextProcessor(sb.ToString());
    }
    
    // ============ Search ============
    public IEnumerable<(int Line, int Column, string Context)> Search(string query)
    {
        string[] lines = _text.Split('\n');
        
        for (int i = 0; i < lines.Length; i++)
        {
            int col = 0;
            while ((col = lines[i].IndexOf(query, col, StringComparison.OrdinalIgnoreCase)) >= 0)
            {
                int start = Math.Max(0, col - 20);
                int end = Math.Min(lines[i].Length, col + query.Length + 20);
                string context = "..." + lines[i][start..end] + "...";
                yield return (i + 1, col + 1, context);
                col++;
            }
        }
    }
    
    // ============ Summary ============
    public string CreateSummary(int maxSentences = 3)
    {
        var sentences = _text
            .Split(new[] { ". ", "! ", "? " }, StringSplitOptions.RemoveEmptyEntries)
            .Take(maxSentences);
        return string.Join(". ", sentences) + ".";
    }
    
    public override string ToString() => _text;
}

record TextStats
{
    public int CharCount { get; init; }
    public int WordCount { get; init; }
    public int SentenceCount { get; init; }
    public int ParagraphCount { get; init; }
    public int UniqueWords { get; init; }
    public double AvgWordLength { get; init; }
    public string LongestWord { get; init; } = "";
    public string MostFrequentWord { get; init; } = "";
    
    public override string ToString() => $"""
        📊 Text Statistics:
        ─────────────────────
        Characters:    {CharCount:N0}
        Words:         {WordCount:N0}
        Sentences:     {SentenceCount:N0}
        Paragraphs:    {ParagraphCount:N0}
        Unique words:  {UniqueWords:N0}
        Avg word len:  {AvgWordLength:F1}
        Longest word:  {LongestWord}
        Most frequent: {MostFrequentWord}
        """;
}

// ============ Main Program ============
string sampleText = """
    C# is a modern, object-oriented programming language developed by Microsoft. 
    It is one of the primary languages used in the .NET ecosystem. C# combines the 
    power of C++ with the simplicity of Visual Basic. The language supports 
    object-oriented programming, functional programming, and generic programming.
    
    C# was designed to be a simple, modern, general-purpose language. It provides 
    strong type checking, lexical scoping, and a common type system. C# programs 
    run on the .NET runtime, which provides memory management, garbage collection, 
    and security features. C# is widely used for web development, desktop applications, 
    game development, and cloud services.
    """;

var processor = new TextProcessor(sampleText);
var stats = processor.Analyze();
Console.WriteLine(stats);

Console.WriteLine("\n🔍 Search for 'programming':");
foreach (var (line, col, context) in processor.Search("programming"))
    Console.WriteLine($"  Line {line}, Col {col}: {context}");

Console.WriteLine("\n📝 Summary:");
Console.WriteLine(processor.CreateSummary(2));

Console.WriteLine("\n🔄 Transformed (wrapped at 50 chars):");
Console.WriteLine(processor.RemoveExtraSpaces().WrapLines(50).ToString()[..200] + "...");
```

---

## สรุป Part 05

```
✅ Step 81: การประกาศ Method และ return values
✅ Step 82: Parameters ทุกชนิด (required, optional, params, ref, out, in)
✅ Step 83: Method Overloading
✅ Step 84: Recursion และ Memoization
✅ Step 85: Local Functions
✅ Steps 86-90: Extension Methods
✅ Steps 91-100: Lambda, Closure, Currying, HOF
✅ Steps 101-110: Text Processor Program
```

## แบบฝึกหัด Part 05

1. เขียน extension method สำหรับ `DateTime` ที่แสดงวันในภาษาไทย
2. สร้าง recursive method หา GCD (Greatest Common Divisor)
3. เขียน higher-order function `Memoize<T, TResult>`
4. สร้าง extension methods สำหรับ `IEnumerable<T>` เช่น `ForEach`, `Flatten`
5. เขียนโปรแกรม Word Frequency Analyzer ที่รับ text file แล้วนับความถี่คำ

---

**ถัดไป: [Part 06 - Arrays และ Collections →](part-06-arrays-collections.md)**
