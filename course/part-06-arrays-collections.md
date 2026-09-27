# Part 06: Arrays และ Collections
## Steps 111-140: การจัดการข้อมูลด้วย Collections ใน C#

---

## Step 111: Arrays พื้นฐาน

```csharp
// ============ Single-dimensional Array ============
// ประกาศและ initialize
int[] numbers = new int[5];        // [0, 0, 0, 0, 0]
int[] primes = new int[] { 2, 3, 5, 7, 11 };
int[] squares = { 1, 4, 9, 16, 25 }; // Short syntax
string[] fruits = { "apple", "banana", "cherry" };

// Access elements
Console.WriteLine(primes[0]);  // 2 (first element)
Console.WriteLine(primes[^1]); // 11 (last element - C# 8+)
Console.WriteLine(primes[^2]); // 7 (second from last)

// Modify
numbers[0] = 10;
numbers[1] = 20;

// Array properties
Console.WriteLine($"Length: {primes.Length}");   // 5
Console.WriteLine($"Rank: {primes.Rank}");        // 1 (1D array)

// ============ Array Operations ============
int[] data = { 5, 3, 8, 1, 9, 2, 7, 4, 6 };

Array.Sort(data);
Console.WriteLine($"Sorted: {string.Join(", ", data)}");  // 1, 2, 3, 4, 5, 6, 7, 8, 9

Array.Reverse(data);
Console.WriteLine($"Reversed: {string.Join(", ", data)}"); // 9, 8, 7, 6, 5, 4, 3, 2, 1

// Binary search (ต้อง sort ก่อน)
Array.Sort(data);
int index = Array.BinarySearch(data, 5);
Console.WriteLine($"Index of 5: {index}"); // 4

// Find
int found = Array.Find(data, x => x > 7); // หาตัวแรกที่ > 7
int[] foundAll = Array.FindAll(data, x => x > 5); // หาทุกตัวที่ > 5
Console.WriteLine($"First > 7: {found}");
Console.WriteLine($"All > 5: {string.Join(", ", foundAll)}");

// Copy
int[] copy = new int[data.Length];
Array.Copy(data, copy, data.Length);
int[] sliceCopy = data[2..5]; // Copy index 2,3,4

// Fill
int[] filled = new int[5];
Array.Fill(filled, 42);
Console.WriteLine($"Filled: {string.Join(", ", filled)}");

// Clear
Array.Clear(data, 2, 3); // Clear index 2,3,4 to 0
Console.WriteLine($"After Clear: {string.Join(", ", data)}");

// ============ Span<T> สำหรับ slice ที่มีประสิทธิภาพ ============
int[] original = { 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };
Span<int> span = original.AsSpan(2, 5); // index 2, length 5
Console.WriteLine($"Span: {string.Join(", ", span.ToArray())}"); // 3,4,5,6,7
span[0] = 99; // แก้ไข original array ด้วย!
Console.WriteLine($"Original[2]: {original[2]}"); // 99
```

---

## Step 112: Multi-dimensional Arrays

```csharp
// ============ 2D Array (Rectangular) ============
int[,] matrix = new int[3, 4]; // 3 rows, 4 columns

// Initialize
int[,] grid = {
    { 1, 2, 3, 4 },
    { 5, 6, 7, 8 },
    { 9, 10, 11, 12 }
};

Console.WriteLine($"Rows: {grid.GetLength(0)}, Cols: {grid.GetLength(1)}");
Console.WriteLine($"Element [1,2]: {grid[1, 2]}"); // 7

// Traverse 2D array
Console.WriteLine("Matrix:");
for (int row = 0; row < grid.GetLength(0); row++)
{
    for (int col = 0; col < grid.GetLength(1); col++)
        Console.Write($"{grid[row, col],4}");
    Console.WriteLine();
}

// ============ Jagged Array (Array of Arrays) ============
// แต่ละแถวมีความยาวต่างกันได้
int[][] jagged = new int[3][];
jagged[0] = new int[] { 1, 2 };
jagged[1] = new int[] { 3, 4, 5 };
jagged[2] = new int[] { 6, 7, 8, 9 };

// หรือ
int[][] triangle = {
    new[] { 1 },
    new[] { 1, 2 },
    new[] { 1, 2, 3 },
    new[] { 1, 2, 3, 4 }
};

Console.WriteLine("\nJagged (Triangle):");
for (int i = 0; i < triangle.Length; i++)
{
    for (int j = 0; j < triangle[i].Length; j++)
        Console.Write($"{triangle[i][j]} ");
    Console.WriteLine();
}

// ============ Matrix Operations ============
int[,] Multiply(int[,] a, int[,] b)
{
    int rows = a.GetLength(0);
    int cols = b.GetLength(1);
    int inner = a.GetLength(1);
    
    int[,] result = new int[rows, cols];
    for (int i = 0; i < rows; i++)
        for (int j = 0; j < cols; j++)
            for (int k = 0; k < inner; k++)
                result[i, j] += a[i, k] * b[k, j];
    
    return result;
}

int[,] A = { { 1, 2 }, { 3, 4 } };
int[,] B = { { 5, 6 }, { 7, 8 } };
int[,] C = Multiply(A, B);

Console.WriteLine("\nMatrix Multiply:");
Console.WriteLine($"[{C[0,0]}, {C[0,1]}]"); // [19, 22]
Console.WriteLine($"[{C[1,0]}, {C[1,1]}]"); // [43, 50]

// ============ 3D Array ============
int[,,] cube = new int[2, 3, 4]; // 2 layers, 3 rows, 4 cols
Console.WriteLine($"3D Length: {cube.Length}"); // 24
```

---

## Step 113: List<T>

```csharp
// ============ List<T> - Dynamic Array ============
using System.Collections.Generic;

// Create
List<int> numbers = new List<int>();
List<string> names = new List<string> { "Alice", "Bob", "Charlie" };
List<int> init = new List<int>(Enumerable.Range(1, 10)); // from IEnumerable

// Add
numbers.Add(1);
numbers.Add(2);
numbers.Add(3);
numbers.AddRange(new[] { 4, 5, 6 }); // Add multiple

numbers.Insert(0, 0);      // Insert at index
numbers.InsertRange(1, new[] { -1, -2 }); // Insert range

Console.WriteLine($"Count: {numbers.Count}");
Console.WriteLine($"Items: {string.Join(", ", numbers)}");

// Remove
numbers.Remove(0);         // Remove first occurrence of value
numbers.RemoveAt(0);       // Remove at index
numbers.RemoveRange(0, 2); // Remove range
numbers.RemoveAll(n => n < 0); // Remove all matching

// Access
int first = numbers[0];
int last = numbers[^1];

// Contains, IndexOf
bool hasThree = numbers.Contains(3);
int indexOfFour = numbers.IndexOf(4);
Console.WriteLine($"Contains 3: {hasThree}, Index of 4: {indexOfFour}");

// Sort
numbers.Sort();
numbers.Sort((a, b) => b.CompareTo(a)); // Descending

// LINQ on List
var evens = numbers.Where(n => n % 2 == 0).ToList();
var doubled = numbers.Select(n => n * 2).ToList();
var sum = numbers.Sum();

Console.WriteLine($"Evens: {string.Join(", ", evens)}");
Console.WriteLine($"Sum: {sum}");

// Convert
int[] array = numbers.ToArray();
HashSet<int> hashSet = numbers.ToHashSet();

// Capacity vs Count
var list = new List<int>(100); // Pre-allocate 100
Console.WriteLine($"Capacity: {list.Capacity}, Count: {list.Count}");

// ForEach
names.ForEach(name => Console.Write($"{name} "));
Console.WriteLine();

// Exists, Find, FindAll
bool exists = names.Exists(n => n.StartsWith("A"));
string? found = names.Find(n => n.Length > 3);
List<string> longNames = names.FindAll(n => n.Length >= 4);
Console.WriteLine($"Exists A*: {exists}, Found: {found}");
Console.WriteLine($"Long names: {string.Join(", ", longNames)}");
```

---

## Step 114: Dictionary<TKey, TValue>

```csharp
// ============ Dictionary ============

// Create
var dict = new Dictionary<string, int>
{
    { "one", 1 },
    { "two", 2 },
    { "three", 3 }
};

// Add
dict.Add("four", 4);
dict["five"] = 5;        // Add or update

// Access
int val = dict["one"];   // KeyNotFoundException ถ้าไม่มี
bool gotTwo = dict.TryGetValue("two", out int twoVal);
Console.WriteLine($"one={val}, two={twoVal}");

// Check
bool hasKey = dict.ContainsKey("three");
bool hasVal = dict.ContainsValue(4);
Console.WriteLine($"Has 'three': {hasKey}, Has 4: {hasVal}");

// Remove
dict.Remove("five");

// Iterate
foreach (var kvp in dict)
    Console.WriteLine($"  {kvp.Key} = {kvp.Value}");

// Keys and Values
Console.WriteLine($"Keys: {string.Join(", ", dict.Keys)}");
Console.WriteLine($"Values: {string.Join(", ", dict.Values)}");

// ============ Dictionary Operations ============
var wordCount = new Dictionary<string, int>();
string text = "the quick brown fox jumps over the lazy dog the fox";

foreach (string word in text.Split(' '))
{
    wordCount[word] = wordCount.GetValueOrDefault(word) + 1;
    // หรือ
    // if (!wordCount.ContainsKey(word)) wordCount[word] = 0;
    // wordCount[word]++;
}

Console.WriteLine("\nWord frequency:");
foreach (var (word, count) in wordCount.OrderByDescending(kv => kv.Value))
    Console.WriteLine($"  '{word}': {count}");

// ============ Nested Dictionary ============
var phoneBook = new Dictionary<string, Dictionary<string, string>>
{
    {
        "Alice",
        new Dictionary<string, string>
        {
            { "mobile", "081-234-5678" },
            { "work", "02-123-4567" }
        }
    },
    {
        "Bob",
        new Dictionary<string, string>
        {
            { "mobile", "089-876-5432" }
        }
    }
};

Console.WriteLine($"\nAlice mobile: {phoneBook["Alice"]["mobile"]}");

// ============ SortedDictionary ============
var sorted = new SortedDictionary<string, int>
{
    { "banana", 2 }, { "apple", 5 }, { "cherry", 1 }
};

Console.WriteLine("\nSorted Dictionary (by key):");
foreach (var kv in sorted)
    Console.WriteLine($"  {kv.Key}: {kv.Value}");

// ============ ReadOnlyDictionary ============
using System.Collections.ObjectModel;
var original = new Dictionary<string, int> { { "a", 1 }, { "b", 2 } };
var readOnly = new ReadOnlyDictionary<string, int>(original);
// ❌ readOnly.Add("c", 3); // Error!
Console.WriteLine($"ReadOnly 'a': {readOnly["a"]}");

// ============ GetOrAdd pattern ============
var cache = new Dictionary<string, List<int>>();

void AddToGroup(string key, int value)
{
    if (!cache.TryGetValue(key, out var list))
    {
        list = new List<int>();
        cache[key] = list;
    }
    list.Add(value);
}

AddToGroup("even", 2);
AddToGroup("even", 4);
AddToGroup("odd", 1);
AddToGroup("odd", 3);

foreach (var (group, items) in cache)
    Console.WriteLine($"{group}: {string.Join(", ", items)}");
```

---

## Step 115: HashSet<T> และ SortedSet<T>

```csharp
// ============ HashSet<T> ============
// Unordered, unique elements, O(1) lookup

var set1 = new HashSet<int> { 1, 2, 3, 4, 5 };
var set2 = new HashSet<int> { 3, 4, 5, 6, 7 };

// Add, Remove, Contains
set1.Add(6);
set1.Add(2); // ไม่เพิ่มซ้ำ
set1.Remove(1);
Console.WriteLine($"Contains 3: {set1.Contains(3)}");
Console.WriteLine($"Count: {set1.Count}");

// Set Operations
var union = new HashSet<int>(set1);
union.UnionWith(set2);           // set1 ∪ set2
Console.WriteLine($"Union: {string.Join(", ", union.OrderBy(x => x))}");

var intersection = new HashSet<int>(set1);
intersection.IntersectWith(set2);  // set1 ∩ set2
Console.WriteLine($"Intersection: {string.Join(", ", intersection.OrderBy(x => x))}");

var difference = new HashSet<int>(set1);
difference.ExceptWith(set2);     // set1 - set2
Console.WriteLine($"Difference (set1-set2): {string.Join(", ", difference.OrderBy(x => x))}");

var symDiff = new HashSet<int>(set1);
symDiff.SymmetricExceptWith(set2); // (set1 ∪ set2) - (set1 ∩ set2)
Console.WriteLine($"Symmetric Diff: {string.Join(", ", symDiff.OrderBy(x => x))}");

// Subset/Superset
var small = new HashSet<int> { 3, 4 };
Console.WriteLine($"small.IsSubsetOf(set1): {small.IsSubsetOf(set1)}");
Console.WriteLine($"set1.IsSupersetOf(small): {set1.IsSupersetOf(small)}");

// Remove duplicates from list
var withDups = new List<int> { 1, 2, 2, 3, 3, 3, 4 };
var unique = new HashSet<int>(withDups);
Console.WriteLine($"Unique: {string.Join(", ", unique)}");

// ============ SortedSet<T> ============
var sortedSet = new SortedSet<int> { 5, 2, 8, 1, 9, 3 };
Console.WriteLine($"\nSortedSet: {string.Join(", ", sortedSet)}"); // Sorted!
Console.WriteLine($"Min: {sortedSet.Min}, Max: {sortedSet.Max}");

// GetViewBetween
var between = sortedSet.GetViewBetween(3, 7);
Console.WriteLine($"Between 3-7: {string.Join(", ", between)}");
```

---

## Step 116: Queue<T> และ Stack<T>

```csharp
// ============ Queue<T> - FIFO (First In, First Out) ============
var queue = new Queue<string>();

// Enqueue (เพิ่มท้าย)
queue.Enqueue("Task 1");
queue.Enqueue("Task 2");
queue.Enqueue("Task 3");
queue.Enqueue("Task 4");

Console.WriteLine($"Queue count: {queue.Count}");
Console.WriteLine($"Peek (next): {queue.Peek()}"); // ดูโดยไม่เอาออก

// Dequeue (เอาออกจากต้น)
while (queue.Count > 0)
{
    string task = queue.Dequeue();
    Console.WriteLine($"Processing: {task}");
}

// TryDequeue, TryPeek (ไม่ throw exception)
bool gotItem = queue.TryDequeue(out string? item);
Console.WriteLine($"TryDequeue: {gotItem}");

// ============ Queue Use Case: BFS ============
void BFS(int[][] adjacency, int start)
{
    bool[] visited = new bool[adjacency.Length];
    var queue2 = new Queue<int>();
    
    queue2.Enqueue(start);
    visited[start] = true;
    
    Console.Write($"BFS from {start}: ");
    
    while (queue2.Count > 0)
    {
        int node = queue2.Dequeue();
        Console.Write($"{node} ");
        
        foreach (int neighbor in adjacency[node])
        {
            if (!visited[neighbor])
            {
                visited[neighbor] = true;
                queue2.Enqueue(neighbor);
            }
        }
    }
    Console.WriteLine();
}

int[][] graph = {
    new[] { 1, 2 },    // 0 -> 1, 2
    new[] { 0, 3, 4 }, // 1 -> 0, 3, 4
    new[] { 0, 5 },    // 2 -> 0, 5
    new[] { 1 },       // 3 -> 1
    new[] { 1 },       // 4 -> 1
    new[] { 2 }        // 5 -> 2
};
BFS(graph, 0);

// ============ Stack<T> - LIFO (Last In, First Out) ============
var stack = new Stack<int>();

// Push
stack.Push(1);
stack.Push(2);
stack.Push(3);
stack.Push(4);
stack.Push(5);

Console.WriteLine($"\nStack count: {stack.Count}");
Console.WriteLine($"Peek (top): {stack.Peek()}"); // 5

// Pop
while (stack.Count > 0)
{
    Console.Write($"{stack.Pop()} ");
}
Console.WriteLine();

// ============ Stack Use Case: Balanced Parentheses ============
bool IsBalanced(string expression)
{
    var stack2 = new Stack<char>();
    
    foreach (char c in expression)
    {
        switch (c)
        {
            case '(' or '[' or '{':
                stack2.Push(c);
                break;
            case ')':
                if (stack2.Count == 0 || stack2.Pop() != '(') return false;
                break;
            case ']':
                if (stack2.Count == 0 || stack2.Pop() != '[') return false;
                break;
            case '}':
                if (stack2.Count == 0 || stack2.Pop() != '{') return false;
                break;
        }
    }
    
    return stack2.Count == 0;
}

Console.WriteLine($"\nBalanced '({{}})': {IsBalanced("({})")}");     // True
Console.WriteLine($"Balanced '([)]': {IsBalanced("([)]")}");         // False
Console.WriteLine($"Balanced '{{[()]}}': {IsBalanced("{[()]}")}");   // True

// ============ PriorityQueue<T, TPriority> (.NET 6+) ============
var pq = new PriorityQueue<string, int>();

pq.Enqueue("Low priority task", 10);
pq.Enqueue("Critical task", 1);
pq.Enqueue("Medium task", 5);
pq.Enqueue("High priority", 2);

Console.WriteLine("\nPriority Queue (lower = higher priority):");
while (pq.Count > 0)
{
    pq.TryDequeue(out string? task, out int priority);
    Console.WriteLine($"  Priority {priority}: {task}");
}
```

---

## Step 117: LinkedList<T>

```csharp
// ============ LinkedList<T> - Doubly Linked List ============
var linked = new LinkedList<int>();

// AddLast, AddFirst
linked.AddLast(3);
linked.AddLast(4);
linked.AddFirst(2);
linked.AddFirst(1);

Console.WriteLine($"List: {string.Join(" <-> ", linked)}");
// 1 <-> 2 <-> 3 <-> 4

// AddBefore, AddAfter
LinkedListNode<int>? node3 = linked.Find(3);
if (node3 != null)
{
    linked.AddBefore(node3, 25); // 1 2 25 3 4
    linked.AddAfter(node3, 35);  // 1 2 25 3 35 4
}

Console.WriteLine($"After insert: {string.Join(" <-> ", linked)}");

// Remove
linked.Remove(25);
linked.RemoveFirst();
linked.RemoveLast();

Console.WriteLine($"After remove: {string.Join(" <-> ", linked)}");

// Traverse
Console.Write("Forward: ");
for (var node = linked.First; node != null; node = node.Next)
    Console.Write($"{node.Value} ");
Console.WriteLine();

Console.Write("Backward: ");
for (var node = linked.Last; node != null; node = node.Previous)
    Console.Write($"{node.Value} ");
Console.WriteLine();

// Properties
Console.WriteLine($"First: {linked.First?.Value}");
Console.WriteLine($"Last: {linked.Last?.Value}");
Console.WriteLine($"Count: {linked.Count}");

// ============ Immutable Collections (.NET) ============
using System.Collections.Immutable;

ImmutableList<int> immList = ImmutableList.Create(1, 2, 3);
ImmutableList<int> immList2 = immList.Add(4);  // Returns new list!
ImmutableList<int> immList3 = immList2.Remove(2);

Console.WriteLine($"\nOriginal: {string.Join(", ", immList)}");  // 1, 2, 3
Console.WriteLine($"Added 4: {string.Join(", ", immList2)}");    // 1, 2, 3, 4
Console.WriteLine($"Removed 2: {string.Join(", ", immList3)}");  // 1, 3, 4

ImmutableDictionary<string, int> immDict = ImmutableDictionary.Create<string, int>()
    .Add("a", 1)
    .Add("b", 2)
    .SetItem("a", 10); // Update

Console.WriteLine($"Immutable dict 'a': {immDict["a"]}");
```

---

## Step 118-125: LINQ Operations on Collections

```csharp
// ============ LINQ on Collections ============
var products = new List<Product>
{
    new(1, "iPhone", "Electronics", 35000, 50),
    new(2, "MacBook", "Electronics", 65000, 20),
    new(3, "T-Shirt", "Clothing", 599, 200),
    new(4, "Jeans", "Clothing", 1299, 150),
    new(5, "Coffee Maker", "Home", 2500, 30),
    new(6, "Headphones", "Electronics", 3500, 100),
    new(7, "Sneakers", "Clothing", 2999, 80),
    new(8, "Blender", "Home", 1800, 45),
};

record Product(int Id, string Name, string Category, decimal Price, int Stock);

// ============ Filtering ============
var expensive = products.Where(p => p.Price > 5000);
Console.WriteLine("Expensive (>5000):");
foreach (var p in expensive)
    Console.WriteLine($"  {p.Name}: ฿{p.Price:N0}");

// ============ Ordering ============
var byPrice = products.OrderBy(p => p.Price);
var byPriceDesc = products.OrderByDescending(p => p.Price);
var multi = products.OrderBy(p => p.Category).ThenByDescending(p => p.Price);

// ============ Projection ============
var names = products.Select(p => p.Name).ToList();
var summaries = products.Select(p => new { p.Name, p.Price, Value = p.Price * p.Stock });

Console.WriteLine("\nInventory values:");
foreach (var s in summaries.OrderByDescending(s => s.Value))
    Console.WriteLine($"  {s.Name,-15} ฿{s.Price:N0} x {s.Value / s.Price:N0} = ฿{s.Value:N0}");

// ============ Aggregation ============
decimal totalValue = products.Sum(p => p.Price * p.Stock);
decimal avgPrice = products.Average(p => p.Price);
decimal maxPrice = products.Max(p => p.Price);
decimal minPrice = products.Min(p => p.Price);
int totalStock = products.Sum(p => p.Stock);

Console.WriteLine($"\nStats:");
Console.WriteLine($"  Total inventory value: ฿{totalValue:N0}");
Console.WriteLine($"  Average price: ฿{avgPrice:N0}");
Console.WriteLine($"  Price range: ฿{minPrice:N0} - ฿{maxPrice:N0}");
Console.WriteLine($"  Total stock: {totalStock:N0}");

// ============ Grouping ============
var byCategory = products.GroupBy(p => p.Category);
Console.WriteLine("\nBy Category:");
foreach (var group in byCategory)
{
    decimal catTotal = group.Sum(p => p.Price * p.Stock);
    Console.WriteLine($"  {group.Key}:");
    Console.WriteLine($"    Items: {group.Count()}");
    Console.WriteLine($"    Value: ฿{catTotal:N0}");
    foreach (var p in group.OrderBy(p => p.Price))
        Console.WriteLine($"      - {p.Name}: ฿{p.Price:N0}");
}

// ============ Join ============
var orders = new List<Order>
{
    new(101, 1, 2, DateTime.Now.AddDays(-1)),
    new(102, 3, 1, DateTime.Now.AddDays(-2)),
    new(103, 1, 3, DateTime.Now.AddDays(-3)),
    new(104, 6, 1, DateTime.Now),
};

record Order(int OrderId, int ProductId, int Quantity, DateTime Date);

var orderDetails = orders.Join(
    products,
    order => order.ProductId,
    product => product.Id,
    (order, product) => new
    {
        order.OrderId,
        product.Name,
        order.Quantity,
        Subtotal = product.Price * order.Quantity,
        order.Date
    }
);

Console.WriteLine("\nOrder Details:");
foreach (var od in orderDetails.OrderByDescending(od => od.Date))
    Console.WriteLine($"  #{od.OrderId} {od.Name} x{od.Quantity} = ฿{od.Subtotal:N0}");

// ============ Distinct, Except, Intersect, Union ============
var list1 = new[] { 1, 2, 3, 4, 5 };
var list2 = new[] { 3, 4, 5, 6, 7 };

Console.WriteLine($"\nDistinct (with duplicates): {string.Join(", ", new[] {1,1,2,3,3}.Distinct())}");
Console.WriteLine($"Except: {string.Join(", ", list1.Except(list2))}");  // [1,2]
Console.WriteLine($"Intersect: {string.Join(", ", list1.Intersect(list2))}");  // [3,4,5]
Console.WriteLine($"Union: {string.Join(", ", list1.Union(list2))}");  // [1,2,3,4,5,6,7]

// ============ SelectMany (Flatten) ============
var categories = new[]
{
    new { Name = "Electronics", Items = new[] { "iPhone", "MacBook", "iPad" } },
    new { Name = "Clothing", Items = new[] { "T-Shirt", "Jeans" } },
};

var allItems = categories.SelectMany(c => c.Items);
Console.WriteLine($"\nAll items: {string.Join(", ", allItems)}");

// SelectMany with index
var itemsWithCategory = categories.SelectMany(
    c => c.Items,
    (c, item) => $"{c.Name}/{item}"
);
Console.WriteLine($"With category: {string.Join(", ", itemsWithCategory)}");
```

---

## Step 126-130: ConcurrentCollections และ Thread-safe

```csharp
// ============ Concurrent Collections ============
using System.Collections.Concurrent;

// ConcurrentDictionary - thread-safe dictionary
var concurrent = new ConcurrentDictionary<string, int>();

// Thread-safe operations
concurrent.TryAdd("key1", 1);
concurrent.AddOrUpdate("key2", 1, (key, oldVal) => oldVal + 1);
int val = concurrent.GetOrAdd("key3", 42);

// ConcurrentQueue
var concurrentQueue = new ConcurrentQueue<string>();
concurrentQueue.Enqueue("item1");
concurrentQueue.Enqueue("item2");

if (concurrentQueue.TryDequeue(out string? dequeuedItem))
    Console.WriteLine($"Dequeued: {dequeuedItem}");

// ConcurrentBag - unordered, allows duplicates (thread-safe)
var bag = new ConcurrentBag<int>();
Parallel.For(0, 100, i => bag.Add(i));
Console.WriteLine($"Bag count: {bag.Count}");

// BlockingCollection - Producer/Consumer pattern
var blockingColl = new BlockingCollection<int>(boundedCapacity: 10);

// Producer
var producer = Task.Run(() =>
{
    for (int i = 0; i < 20; i++)
    {
        blockingColl.Add(i);
        Console.WriteLine($"Produced: {i}");
        System.Threading.Thread.Sleep(10);
    }
    blockingColl.CompleteAdding();
});

// Consumer
var consumer = Task.Run(() =>
{
    foreach (int item in blockingColl.GetConsumingEnumerable())
    {
        Console.WriteLine($"  Consumed: {item}");
    }
});

Task.WaitAll(producer, consumer);
```

---

## Step 131-140: โปรแกรม Inventory System

```csharp
// ============ Inventory Management System ============
using System;
using System.Collections.Generic;
using System.Linq;

// ============ Models ============
class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public string Category { get; set; } = "";
    public decimal Price { get; set; }
    public int Stock { get; set; }
    public DateTime LastUpdated { get; set; } = DateTime.Now;
    
    public override string ToString() =>
        $"[{Id}] {Name,-20} {Category,-15} ฿{Price,8:N0} Stock: {Stock,5}";
}

class InventorySystem
{
    private List<Product> _products = new();
    private int _nextId = 1;
    private Stack<string> _history = new();
    private Dictionary<string, List<Product>> _categoryIndex = new();
    
    // ============ CRUD ============
    public Product AddProduct(string name, string category, decimal price, int stock)
    {
        var product = new Product
        {
            Id = _nextId++,
            Name = name,
            Category = category,
            Price = price,
            Stock = stock
        };
        
        _products.Add(product);
        IndexProduct(product);
        LogAction($"Added product: {product.Name}");
        return product;
    }
    
    public bool UpdateStock(int id, int quantity, bool isAdd = true)
    {
        var product = _products.FirstOrDefault(p => p.Id == id);
        if (product == null) return false;
        
        if (isAdd)
            product.Stock += quantity;
        else
        {
            if (product.Stock < quantity)
            {
                Console.WriteLine($"Error: Insufficient stock for {product.Name}");
                return false;
            }
            product.Stock -= quantity;
        }
        
        product.LastUpdated = DateTime.Now;
        LogAction($"{(isAdd ? "Added" : "Removed")} {quantity} of {product.Name}");
        return true;
    }
    
    public bool UpdatePrice(int id, decimal newPrice)
    {
        var product = _products.FirstOrDefault(p => p.Id == id);
        if (product == null) return false;
        
        decimal old = product.Price;
        product.Price = newPrice;
        product.LastUpdated = DateTime.Now;
        LogAction($"Price changed for {product.Name}: {old:N0} -> {newPrice:N0}");
        return true;
    }
    
    public bool Remove(int id)
    {
        var product = _products.FirstOrDefault(p => p.Id == id);
        if (product == null) return false;
        
        _products.Remove(product);
        RemoveFromIndex(product);
        LogAction($"Removed product: {product.Name}");
        return true;
    }
    
    // ============ Search & Filter ============
    public IEnumerable<Product> GetAll() => _products;
    
    public Product? GetById(int id) => _products.FirstOrDefault(p => p.Id == id);
    
    public IEnumerable<Product> Search(string query)
    {
        query = query.ToLower();
        return _products.Where(p =>
            p.Name.ToLower().Contains(query) ||
            p.Category.ToLower().Contains(query));
    }
    
    public IEnumerable<Product> GetByCategory(string category)
    {
        return _categoryIndex.TryGetValue(category, out var products)
            ? products
            : Enumerable.Empty<Product>();
    }
    
    public IEnumerable<Product> GetLowStock(int threshold = 10)
        => _products.Where(p => p.Stock <= threshold).OrderBy(p => p.Stock);
    
    public IEnumerable<Product> GetPriceRange(decimal min, decimal max)
        => _products.Where(p => p.Price >= min && p.Price <= max);
    
    // ============ Statistics ============
    public Dictionary<string, object> GetStats()
    {
        return new Dictionary<string, object>
        {
            ["TotalProducts"] = _products.Count,
            ["TotalValue"] = _products.Sum(p => p.Price * p.Stock),
            ["AvgPrice"] = _products.Any() ? _products.Average(p => p.Price) : 0,
            ["TotalStock"] = _products.Sum(p => p.Stock),
            ["Categories"] = _products.Select(p => p.Category).Distinct().Count(),
            ["LowStockCount"] = _products.Count(p => p.Stock <= 10),
            ["OutOfStock"] = _products.Count(p => p.Stock == 0),
        };
    }
    
    public IEnumerable<(string Category, decimal TotalValue, int ProductCount)> GetCategoryStats()
    {
        return _products
            .GroupBy(p => p.Category)
            .Select(g => (
                Category: g.Key,
                TotalValue: g.Sum(p => p.Price * p.Stock),
                ProductCount: g.Count()
            ))
            .OrderByDescending(x => x.TotalValue);
    }
    
    // ============ History ============
    private void LogAction(string action)
    {
        _history.Push($"[{DateTime.Now:HH:mm:ss}] {action}");
    }
    
    public IEnumerable<string> GetHistory(int count = 10)
        => _history.Take(count);
    
    // ============ Index ============
    private void IndexProduct(Product p)
    {
        if (!_categoryIndex.TryGetValue(p.Category, out var list))
        {
            list = new List<Product>();
            _categoryIndex[p.Category] = list;
        }
        list.Add(p);
    }
    
    private void RemoveFromIndex(Product p)
    {
        if (_categoryIndex.TryGetValue(p.Category, out var list))
            list.Remove(p);
    }
    
    // ============ Display ============
    public void DisplayAll()
    {
        Console.WriteLine($"\n{'ID',-5} {'Name',-20} {'Category',-15} {'Price',10} {'Stock',8}");
        Console.WriteLine(new string('─', 62));
        
        foreach (var p in _products.OrderBy(p => p.Category).ThenBy(p => p.Name))
            Console.WriteLine($"{p.Id,-5} {p.Name,-20} {p.Category,-15} ฿{p.Price,8:N0} {p.Stock,8}");
        
        Console.WriteLine(new string('─', 62));
        Console.WriteLine($"Total: {_products.Count} items");
    }
}

// ============ Main Program ============
var inventory = new InventorySystem();

// Add products
inventory.AddProduct("iPhone 15", "Electronics", 35900, 50);
inventory.AddProduct("MacBook Pro", "Electronics", 69900, 15);
inventory.AddProduct("iPad Air", "Electronics", 25900, 30);
inventory.AddProduct("Classic T-Shirt", "Clothing", 499, 200);
inventory.AddProduct("Slim Jeans", "Clothing", 1299, 100);
inventory.AddProduct("Coffee Maker", "Home", 2500, 25);
inventory.AddProduct("Air Fryer", "Home", 3200, 8);
inventory.AddProduct("Headphones", "Electronics", 5500, 5);

inventory.DisplayAll();

// Statistics
var stats = inventory.GetStats();
Console.WriteLine("\n📊 Statistics:");
foreach (var (key, value) in stats)
    Console.WriteLine($"  {key,-20}: {value:N0}");

// Category breakdown
Console.WriteLine("\n📂 By Category:");
foreach (var (cat, value, count) in inventory.GetCategoryStats())
    Console.WriteLine($"  {cat,-15}: {count} items, ฿{value:N0}");

// Low stock
Console.WriteLine("\n⚠️  Low Stock:");
foreach (var p in inventory.GetLowStock())
    Console.WriteLine($"  {p}");

// Search
Console.WriteLine("\n🔍 Search 'pro':");
foreach (var p in inventory.Search("pro"))
    Console.WriteLine($"  {p}");

// History
Console.WriteLine("\n📋 Recent Actions:");
foreach (var h in inventory.GetHistory(5))
    Console.WriteLine($"  {h}");
```

---

## สรุป Part 06

```
✅ Step 111: Arrays พื้นฐานและ operations
✅ Step 112: Multi-dimensional และ Jagged Arrays
✅ Step 113: List<T> - Dynamic arrays
✅ Step 114: Dictionary<TKey, TValue>
✅ Step 115: HashSet<T> และ SortedSet<T>
✅ Step 116: Queue<T>, Stack<T>, PriorityQueue<T>
✅ Step 117: LinkedList<T> และ Immutable Collections
✅ Steps 118-125: LINQ operations ครบถ้วน
✅ Steps 126-130: Concurrent Collections
✅ Steps 131-140: Inventory Management System
```

## แบบฝึกหัด Part 06

1. สร้าง generic `Cache<TKey, TValue>` ด้วย Dictionary + Expiry
2. Implement `Graph<T>` ด้วย Dictionary<T, List<T>>
3. เขียน program ที่อ่านคำจาก text และเก็บใน Trie structure
4. สร้าง `LRU Cache` ด้วย Dictionary + LinkedList
5. เขียน program วิเคราะห์ student grades ด้วย LINQ ทุกรูปแบบ

---

**ถัดไป: [Part 07 - การจัดการ String →](part-07-strings.md)**
