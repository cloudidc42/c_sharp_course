# Part 12: LINQ — Language Integrated Query (Steps 321-370)

## เป้าหมายการเรียนรู้
- เข้าใจ LINQ และ Deferred Execution
- Query Syntax vs Method Syntax
- Standard Query Operators ทั้งหมด
- LINQ to Objects, LINQ to XML, LINQ to JSON
- Custom LINQ operators ด้วย Extension Methods
- Parallel LINQ (PLINQ)
- Performance considerations

---

## Step 321: LINQ Fundamentals และ Deferred Execution

```csharp
using System.Linq;

// LINQ = Language Integrated Query
// ช่วยให้เราสามารถ query ข้อมูลจาก collection ได้โดยใช้ syntax คล้าย SQL

int[] numbers = { 5, 3, 8, 1, 9, 2, 7, 4, 6 };

// Method Syntax (Fluent)
var evenNumbers = numbers
    .Where(n => n % 2 == 0)
    .OrderBy(n => n)
    .ToList();

// Query Syntax (SQL-like)
var evenQuery = from n in numbers
                where n % 2 == 0
                orderby n
                select n;

Console.WriteLine(string.Join(", ", evenNumbers)); // 2, 4, 6, 8

// DEFERRED EXECUTION — สำคัญมาก!
// LINQ query ไม่ execute จนกว่าจะมีการ enumerate

var query = numbers.Where(n =>
{
    Console.WriteLine($"Checking {n}");
    return n > 5;
});
Console.WriteLine("Query defined (not yet executed)");

// Execute เมื่อ enumerate
Console.WriteLine("Executing...");
foreach (var n in query)
    Console.Write(n + " ");

// Immediate Execution — ทำได้ด้วย ToList(), ToArray(), Count(), First(), etc.
var immediate = numbers.Where(n => n > 5).ToList(); // Execute ทันที

// Multiple Enumeration Problem
var delayed = numbers.Where(n => n > 5); // ไม่ execute
var count = delayed.Count();   // Execute ครั้งที่ 1
var list = delayed.ToList();   // Execute ครั้งที่ 2 (อาจต่างกันถ้า source เปลี่ยน!)

// Best Practice: ToList() ถ้าจะใช้หลายครั้ง
var cached = numbers.Where(n => n > 5).ToList(); // Execute ครั้งเดียว
var count2 = cached.Count;     // ไม่ execute ซ้ำ
var first = cached[0];        // ไม่ execute ซ้ำ

// IEnumerable<T> vs IQueryable<T>
// IEnumerable<T>: LINQ to Objects (in-memory)
// IQueryable<T>: LINQ to SQL/EF (translated to SQL query)
```

---

## Step 322: Filtering — Where, OfType, Distinct, Skip, Take

```csharp
var numbers = Enumerable.Range(1, 20).ToList();
var people = new List<(string Name, int Age, string City)>
{
    ("Alice", 30, "Bangkok"),
    ("Bob", 25, "Chiang Mai"),
    ("Carol", 35, "Bangkok"),
    ("Dave", 28, "Phuket"),
    ("Eve", 22, "Bangkok"),
    ("Frank", 40, "Chiang Mai"),
};

// Where — filter by condition
var adults = people.Where(p => p.Age >= 30);
var bangkokAdults = people.Where(p => p.City == "Bangkok" && p.Age >= 30);

// Multiple Where (equivalent, but one is more readable)
var filtered1 = people.Where(p => p.City == "Bangkok").Where(p => p.Age < 35);
var filtered2 = people.Where(p => p.City == "Bangkok" && p.Age < 35);

// Where with index
var oddIndexed = numbers.Where((n, i) => i % 2 == 0); // Elements at even index

// OfType — filter and cast
object[] mixed = { 1, "hello", 2.5, "world", 3, true };
var strings = mixed.OfType<string>();   // ["hello", "world"]
var ints = mixed.OfType<int>();         // [1, 3]

// Distinct
var cities = people.Select(p => p.City).Distinct(); // Bangkok, Chiang Mai, Phuket

// DistinctBy (C# 6+, .NET 6+)
var uniqueByCity = people.DistinctBy(p => p.City); // First person from each city

// Skip, Take
var page2 = numbers.Skip(5).Take(5); // Items 6-10 (page 2 of page size 5)

// SkipWhile, TakeWhile
var afterFive = numbers.SkipWhile(n => n <= 5);  // [6, 7, 8, ...]
var upToFive = numbers.TakeWhile(n => n <= 5);    // [1, 2, 3, 4, 5]

// SkipLast, TakeLast (.NET 6+)
var allButLast3 = numbers.SkipLast(3);
var last3 = numbers.TakeLast(3);

// Chunk (.NET 6+) — split into fixed-size chunks
foreach (var chunk in numbers.Chunk(5))
    Console.WriteLine(string.Join(", ", chunk));
// 1, 2, 3, 4, 5
// 6, 7, 8, 9, 10
// ...

// Pagination helper
static IEnumerable<T> Page<T>(IEnumerable<T> source, int page, int pageSize)
    => source.Skip((page - 1) * pageSize).Take(pageSize);

var page1Items = Page(people, 1, 3);
```

---

## Step 323: Projection — Select, SelectMany

```csharp
var people = new[]
{
    new { Name = "Alice", Age = 30, Hobbies = new[] {"Reading", "Coding", "Hiking"} },
    new { Name = "Bob",   Age = 25, Hobbies = new[] {"Gaming", "Cooking"} },
    new { Name = "Carol", Age = 35, Hobbies = new[] {"Reading", "Painting"} },
};

// Select — transform each element
var names = people.Select(p => p.Name);                     // ["Alice", "Bob", "Carol"]
var upperNames = people.Select(p => p.Name.ToUpper());      // ["ALICE", "BOB", "CAROL"]

// Select into anonymous type
var summaries = people.Select(p => new { p.Name, IsAdult = p.Age >= 18 });
foreach (var s in summaries)
    Console.WriteLine($"{s.Name}: {s.IsAdult}");

// Select with index
var indexed = people.Select((p, i) => new { Index = i + 1, p.Name });
// { Index = 1, Name = "Alice" }, etc.

// Select into record / class
record PersonDto(string Name, int Age);
var dtos = people.Select(p => new PersonDto(p.Name, p.Age));

// SelectMany — flatten nested collections
var allHobbies = people.SelectMany(p => p.Hobbies);
// Reading, Coding, Hiking, Gaming, Cooking, Reading, Painting

// SelectMany with result selector
var personHobbies = people.SelectMany(
    p => p.Hobbies,
    (person, hobby) => new { person.Name, Hobby = hobby }
);
foreach (var ph in personHobbies)
    Console.WriteLine($"{ph.Name}: {ph.Hobby}");

// Flatten nested arrays
int[][] matrix = { new[] {1, 2, 3}, new[] {4, 5, 6}, new[] {7, 8, 9} };
var flat = matrix.SelectMany(row => row); // 1, 2, 3, 4, 5, 6, 7, 8, 9

// Cartesian product
var colors = new[] { "Red", "Green", "Blue" };
var sizes = new[] { "S", "M", "L" };
var products = colors.SelectMany(c => sizes, (c, s) => $"{c}-{s}");
// Red-S, Red-M, Red-L, Green-S, ...

// Cast — force cast (throws if wrong type)
object[] nums = { 1, 2, 3 };
var intNums = nums.Cast<int>(); // [1, 2, 3]

// Zip — combine two sequences
var names2 = new[] { "Alice", "Bob", "Carol" };
var scores = new[] { 95, 87, 92 };
var combined = names2.Zip(scores, (n, s) => new { Name = n, Score = s });
// {Alice, 95}, {Bob, 87}, {Carol, 92}

// Zip with 3 sequences (.NET 6+)
var ranks = new[] { 1, 2, 3 };
var tripled = names2.Zip(scores).Zip(ranks, (ns, r) => new { ns.First, ns.Second, Rank = r });
```

---

## Step 324: Ordering — OrderBy, ThenBy, Reverse

```csharp
var employees = new[]
{
    new { Name = "Frank", Dept = "IT",  Salary = 75000 },
    new { Name = "Alice", Dept = "HR",  Salary = 65000 },
    new { Name = "Dave",  Dept = "IT",  Salary = 80000 },
    new { Name = "Bob",   Dept = "HR",  Salary = 70000 },
    new { Name = "Eve",   Dept = "IT",  Salary = 75000 },
    new { Name = "Carol", Dept = "Fin", Salary = 90000 },
};

// OrderBy — ascending
var byName = employees.OrderBy(e => e.Name);
var bySalary = employees.OrderBy(e => e.Salary);

// OrderByDescending
var byHighSalary = employees.OrderByDescending(e => e.Salary);

// ThenBy, ThenByDescending — secondary sort
var byDeptThenName = employees
    .OrderBy(e => e.Dept)
    .ThenBy(e => e.Name);

var byDeptThenSalaryDesc = employees
    .OrderBy(e => e.Dept)
    .ThenByDescending(e => e.Salary);

foreach (var e in byDeptThenName)
    Console.WriteLine($"{e.Dept,-5} {e.Name,-8} {e.Salary:C}");

// Reverse — reverses existing order
var reversed = employees.OrderBy(e => e.Name).Reverse();

// Custom comparison
var byNameLength = employees.OrderBy(e => e.Name.Length).ThenBy(e => e.Name);

// Custom IComparer
var customSort = employees.OrderBy(e => e.Name, StringComparer.OrdinalIgnoreCase);

// LINQ Order (C# Preview .NET 7+)
// employees.Order() — uses default comparer
// employees.OrderDescending()

// Query Syntax with ordering
var query = from e in employees
            orderby e.Dept ascending, e.Salary descending
            select e;

// Sort stable in .NET — same-key elements maintain relative order
```

---

## Step 325: Grouping — GroupBy, ToLookup

```csharp
var orders = new[]
{
    new { Id = 1, Customer = "Alice", Amount = 100m, Category = "Electronics" },
    new { Id = 2, Customer = "Bob",   Amount = 250m, Category = "Clothing" },
    new { Id = 3, Customer = "Alice", Amount = 150m, Category = "Electronics" },
    new { Id = 4, Customer = "Carol", Amount = 300m, Category = "Electronics" },
    new { Id = 5, Customer = "Bob",   Amount = 75m,  Category = "Clothing" },
    new { Id = 6, Customer = "Alice", Amount = 200m, Category = "Books" },
};

// GroupBy — group elements by key
var byCustomer = orders.GroupBy(o => o.Customer);
foreach (var group in byCustomer)
{
    Console.WriteLine($"{group.Key}: {group.Count()} orders, Total: {group.Sum(o => o.Amount):C}");
}
// Alice: 3 orders, Total: $450.00
// Bob:   2 orders, Total: $325.00
// Carol: 1 orders, Total: $300.00

// GroupBy with element selector
var customerTotals = orders
    .GroupBy(o => o.Customer, o => o.Amount)
    .Select(g => new { Customer = g.Key, Total = g.Sum() });

// GroupBy with result selector
var grouped = orders.GroupBy(
    o => o.Category,
    (category, items) => new
    {
        Category = category,
        Count = items.Count(),
        Total = items.Sum(i => i.Amount),
        Average = items.Average(i => i.Amount)
    }
).OrderByDescending(g => g.Total);

foreach (var g in grouped)
    Console.WriteLine($"{g.Category,-15} Count: {g.Count}, Total: {g.Total:C}, Avg: {g.Average:C}");

// Nested grouping
var nestedGroups = orders
    .GroupBy(o => o.Customer)
    .Select(customerGroup => new
    {
        Customer = customerGroup.Key,
        Categories = customerGroup
            .GroupBy(o => o.Category)
            .Select(catGroup => new
            {
                Category = catGroup.Key,
                Total = catGroup.Sum(o => o.Amount)
            })
    });

// ToLookup — like GroupBy but creates an ILookup (dictionary-like, efficient for repeated lookup)
ILookup<string, decimal> customerAmounts = orders.ToLookup(o => o.Customer, o => o.Amount);

// O(1) lookup
decimal[] aliceAmounts = customerAmounts["Alice"].ToArray(); // [100, 150, 200]
decimal[] unknownAmounts = customerAmounts["Unknown"].ToArray(); // [] (no exception!)

// Query Syntax GroupBy
var queryGrouped = from o in orders
                   group o by o.Category into g
                   select new { Category = g.Key, Total = g.Sum(o => o.Amount) };
```

---

## Step 326: Aggregation — Count, Sum, Min, Max, Average, Aggregate

```csharp
var numbers = new[] { 3, 1, 4, 1, 5, 9, 2, 6, 5, 3, 5 };
var products = new[]
{
    new { Name = "Apple",  Price = 0.99m,  Stock = 100 },
    new { Name = "Banana", Price = 0.50m,  Stock = 200 },
    new { Name = "Cherry", Price = 3.99m,  Stock = 50  },
    new { Name = "Durian", Price = 15.00m, Stock = 10  },
};

// Count
int total = numbers.Count();                             // 11
int distinctCount = numbers.Distinct().Count();          // 6
int countAbove5 = numbers.Count(n => n > 5);            // 3

// LongCount (for very large collections)
long longCount = numbers.LongCount();

// Sum
decimal totalValue = products.Sum(p => p.Price * p.Stock); // 0.99*100 + ...
int sumNumbers = numbers.Sum(); // 44

// Min, Max
int min = numbers.Min(); // 1
int max = numbers.Max(); // 9
decimal maxPrice = products.Max(p => p.Price); // 15.00

// MinBy, MaxBy (.NET 6+) — returns the element, not the min/max value
var cheapest = products.MinBy(p => p.Price);
var mostExpensive = products.MaxBy(p => p.Price);
Console.WriteLine($"Cheapest: {cheapest.Name} at {cheapest.Price:C}");

// Average
double average = numbers.Average();    // 4.0
decimal avgPrice = products.Average(p => p.Price); // 5.12

// Aggregate — general purpose reduction
int factorial = Enumerable.Range(1, 5)
    .Aggregate(1, (acc, n) => acc * n); // 120 (1*1*2*3*4*5)

string sentence = new[] { "Hello", "World", "from", "LINQ" }
    .Aggregate((a, b) => $"{a} {b}"); // "Hello World from LINQ"

// Aggregate with seed and result selector
var stats = numbers.Aggregate(
    seed: (Min: int.MaxValue, Max: int.MinValue, Sum: 0, Count: 0),
    func: (acc, n) => (
        Min: Math.Min(acc.Min, n),
        Max: Math.Max(acc.Max, n),
        Sum: acc.Sum + n,
        Count: acc.Count + 1
    ),
    resultSelector: acc => new
    {
        acc.Min,
        acc.Max,
        acc.Sum,
        acc.Count,
        Average = (double)acc.Sum / acc.Count
    }
);
Console.WriteLine($"Stats: Min={stats.Min}, Max={stats.Max}, Avg={stats.Average:F2}");

// Any, All, Contains
bool hasEven = numbers.Any(n => n % 2 == 0);    // true
bool allPositive = numbers.All(n => n > 0);      // true
bool hasSeven = numbers.Contains(7);             // false

// Sum of multiple properties
var report = products.Aggregate(
    (TotalValue: 0m, TotalStock: 0, Count: 0),
    (acc, p) => (
        acc.TotalValue + p.Price * p.Stock,
        acc.TotalStock + p.Stock,
        acc.Count + 1
    )
);
```

---

## Step 327: Joining — Join, GroupJoin, Cross Join

```csharp
var customers = new[]
{
    new { Id = 1, Name = "Alice" },
    new { Id = 2, Name = "Bob" },
    new { Id = 3, Name = "Carol" },
    new { Id = 4, Name = "Dave" },
};

var orders = new[]
{
    new { OrderId = 101, CustomerId = 1, Amount = 100m },
    new { OrderId = 102, CustomerId = 1, Amount = 200m },
    new { OrderId = 103, CustomerId = 2, Amount = 150m },
    new { OrderId = 104, CustomerId = 3, Amount = 300m },
};

// Join (Inner Join)
var customerOrders = customers.Join(
    orders,
    c => c.Id,
    o => o.CustomerId,
    (c, o) => new { c.Name, o.OrderId, o.Amount }
);
foreach (var co in customerOrders)
    Console.WriteLine($"{co.Name}: Order {co.OrderId}, ${co.Amount}");
// Alice: Order 101, 100
// Alice: Order 102, 200
// Bob: Order 103, 150
// Carol: Order 104, 300
// Dave is NOT in results (inner join)

// GroupJoin (Left Outer Join)
var customerWithOrders = customers.GroupJoin(
    orders,
    c => c.Id,
    o => o.CustomerId,
    (c, orderGroup) => new
    {
        c.Name,
        OrderCount = orderGroup.Count(),
        Total = orderGroup.Sum(o => o.Amount),
        Orders = orderGroup.ToList()
    }
);
foreach (var cwo in customerWithOrders)
    Console.WriteLine($"{cwo.Name}: {cwo.OrderCount} orders, Total: {cwo.Total:C}");
// Dave: 0 orders, Total: $0.00 (Dave IS in results)

// Left Outer Join using SelectMany
var leftJoin = customers
    .GroupJoin(orders, c => c.Id, o => o.CustomerId, (c, g) => new { c, g })
    .SelectMany(x => x.g.DefaultIfEmpty(), (x, o) => new
    {
        x.c.Name,
        OrderId = o?.OrderId,
        Amount = o?.Amount
    });

// Cross Join
var colors = new[] { "Red", "Blue", "Green" };
var sizes = new[] { "S", "M", "L" };
var variants = from c in colors
               from s in sizes
               select new { Color = c, Size = s };
// Red-S, Red-M, Red-L, Blue-S, ...

// Query Syntax Join
var joinQuery = from c in customers
                join o in orders on c.Id equals o.CustomerId
                select new { c.Name, o.OrderId, o.Amount };

// Multi-key Join
var multiKey = customers.Join(
    orders,
    c => new { c.Id, Type = "Regular" },
    o => new { Id = o.CustomerId, Type = "Regular" },
    (c, o) => new { c.Name, o.Amount }
);
```

---

## Step 328: Set Operations — Distinct, Union, Intersect, Except

```csharp
var set1 = new[] { 1, 2, 3, 4, 5 };
var set2 = new[] { 3, 4, 5, 6, 7 };

// Union — combine, remove duplicates
var union = set1.Union(set2);               // 1, 2, 3, 4, 5, 6, 7

// Intersect — elements in both
var intersect = set1.Intersect(set2);       // 3, 4, 5

// Except — elements in first but not second
var except1 = set1.Except(set2);            // 1, 2
var except2 = set2.Except(set1);            // 6, 7

// SymmetricExcept (not in LINQ but can compose)
var symmetric = set1.Union(set2).Except(set1.Intersect(set2)); // 1, 2, 6, 7

// Distinct
var withDups = new[] { 1, 2, 2, 3, 3, 3, 4 };
var distinct = withDups.Distinct();          // 1, 2, 3, 4

// Operations with custom comparer
var fruits1 = new[] { "Apple", "Banana", "Cherry" };
var fruits2 = new[] { "apple", "DURIAN", "cherry" };
var unionCI = fruits1.Union(fruits2, StringComparer.OrdinalIgnoreCase);
// Apple, Banana, Cherry, DURIAN

// UnionBy, IntersectBy, ExceptBy, DistinctBy (.NET 6+)
var people1 = new[] { new { Id = 1, Name = "Alice" }, new { Id = 2, Name = "Bob" } };
var people2 = new[] { new { Id = 2, Name = "Bob" }, new { Id = 3, Name = "Carol" } };

var unionById = people1.UnionBy(people2, p => p.Id);
// {1, Alice}, {2, Bob}, {3, Carol}

var intersectById = people1.IntersectBy(people2.Select(p => p.Id), p => p.Id);
// {2, Bob}

var exceptById = people1.ExceptBy(people2.Select(p => p.Id), p => p.Id);
// {1, Alice}

// Concat — combines without removing duplicates
var concat = set1.Concat(set2);             // 1, 2, 3, 4, 5, 3, 4, 5, 6, 7

// Append, Prepend
var appended = set1.Append(6);              // 1, 2, 3, 4, 5, 6
var prepended = set1.Prepend(0);            // 0, 1, 2, 3, 4, 5
```

---

## Step 329: Element Operations — First, Last, Single, ElementAt

```csharp
var numbers = new[] { 3, 1, 4, 1, 5, 9, 2, 6 };

// First, FirstOrDefault
int first = numbers.First();                           // 3
int firstEven = numbers.First(n => n % 2 == 0);       // 4
int firstOrDefault = numbers.FirstOrDefault();         // 3
int notFound = numbers.FirstOrDefault(n => n > 100);  // 0 (default int)
int? notFoundNullable = numbers.Cast<int?>().FirstOrDefault(n => n > 100); // null

// FirstOrDefault with default value (C# 10+)
int withDefault = numbers.FirstOrDefault(n => n > 100, -1); // -1

// Last, LastOrDefault
int last = numbers.Last();                            // 6
int lastOdd = numbers.Last(n => n % 2 != 0);         // 9

// Single, SingleOrDefault — throws if 0 or 2+ match
int single = numbers.Where(n => n == 9).Single();    // 9
// numbers.Single(n => n > 5); // throws InvalidOperationException (multiple)

// ElementAt, ElementAtOrDefault
int third = numbers.ElementAt(2);                    // 4
int byIndex = numbers.ElementAtOrDefault(100);       // 0 (no exception)

// ElementAt with Index (C# 8+)
int fromEnd = numbers.ElementAt(^1);                // 6 (last)

// DefaultIfEmpty
var empty = Enumerable.Empty<int>();
var withDefault2 = empty.DefaultIfEmpty(42).First(); // 42
int[] emptyArr = Array.Empty<int>();
var defaults = emptyArr.DefaultIfEmpty(0);           // [0]

// Sequence equality
bool equal1 = new[] {1,2,3}.SequenceEqual(new[] {1,2,3}); // true
bool equal2 = new[] {1,2,3}.SequenceEqual(new[] {1,3,2}); // false
bool equalCI = new[] {"a","b"}.SequenceEqual(new[] {"A","B"}, StringComparer.OrdinalIgnoreCase); // true

// Min/Max single value
int min = numbers.Min();                // 1
int max = numbers.Max();                // 9
var minObj = numbers.MinBy(n => -n);   // 9 (object with smallest neg)
```

---

## Step 330: Generation Operators

```csharp
// Range — generate sequence of integers
var range = Enumerable.Range(1, 10);            // 1..10
var squares = Enumerable.Range(1, 10).Select(n => n * n); // 1,4,9,16,...

// Repeat — repeat a value
var zeros = Enumerable.Repeat(0, 5);            // 0,0,0,0,0
var hellos = Enumerable.Repeat("Hello", 3);     // Hello,Hello,Hello

// Empty — empty sequence
var empty = Enumerable.Empty<int>();

// Practical uses
// Initialize 2D grid
var grid = Enumerable.Range(0, 5)
    .Select(row => Enumerable.Range(0, 5).Select(col => row * 5 + col).ToArray())
    .ToArray();

// Fibonacci with Aggregate
var fibonacci = Enumerable.Range(0, 10)
    .Aggregate((prev: 0, curr: 1, result: new List<int>()),
        (acc, _) =>
        {
            acc.result.Add(acc.prev);
            return (acc.curr, acc.prev + acc.curr, acc.result);
        },
        acc => acc.result);

// Generate sequence with custom logic using yield
static IEnumerable<int> InfiniteCounter(int start = 0)
{
    int current = start;
    while (true)
        yield return current++;
}
var first10Even = InfiniteCounter(0).Where(n => n % 2 == 0).Take(10);

// Generate using LINQ
static IEnumerable<T> Generate<T>(T seed, Func<T, T> next)
{
    T current = seed;
    while (true)
    {
        yield return current;
        current = next(current);
    }
}

var powers = Generate(1, n => n * 2).Take(10); // 1,2,4,8,16,...
var dateRange = Generate(DateTime.Today, d => d.AddDays(1)).Take(7)
    .Select(d => d.ToString("yyyy-MM-dd"));
```

---

## Step 331: LINQ with Complex Objects

```csharp
// Domain model
record Address(string Street, string City, string Country);
record Product(int Id, string Name, decimal Price, string Category, int Stock);
record Customer(int Id, string Name, string Email, Address Address);
record OrderItem(int ProductId, int Quantity);
record Order(int Id, int CustomerId, DateTime Date, List<OrderItem> Items);

// Sample data
var products = new List<Product>
{
    new(1, "Laptop",   1200m, "Electronics", 50),
    new(2, "Phone",     800m, "Electronics", 100),
    new(3, "Desk",      350m, "Furniture",   20),
    new(4, "Chair",     250m, "Furniture",   30),
    new(5, "Notebook",   15m, "Stationery", 500),
    new(6, "Pen",         5m, "Stationery", 1000),
    new(7, "Monitor",   400m, "Electronics", 40),
};

var customers = new List<Customer>
{
    new(1, "Alice", "alice@email.com", new("123 Main St", "Bangkok", "Thailand")),
    new(2, "Bob",   "bob@email.com",   new("456 Oak Ave", "Bangkok", "Thailand")),
    new(3, "Carol", "carol@email.com", new("789 Pine Rd", "Chiang Mai", "Thailand")),
};

var orders = new List<Order>
{
    new(1, 1, DateTime.Today.AddDays(-10), new() { new(1, 1), new(5, 3) }),
    new(2, 1, DateTime.Today.AddDays(-5),  new() { new(2, 2) }),
    new(3, 2, DateTime.Today.AddDays(-3),  new() { new(3, 1), new(4, 1) }),
    new(4, 3, DateTime.Today.AddDays(-1),  new() { new(7, 1), new(6, 10) }),
};

// Complex query: Order summary with customer info
var orderSummaries = orders.Join(
    customers,
    o => o.CustomerId,
    c => c.Id,
    (o, c) => new
    {
        OrderId = o.Id,
        CustomerName = c.Name,
        Date = o.Date,
        Total = o.Items.Sum(item =>
        {
            var product = products.First(p => p.Id == item.ProductId);
            return product.Price * item.Quantity;
        }),
        Items = o.Items.Select(item =>
        {
            var product = products.First(p => p.Id == item.ProductId);
            return new { product.Name, product.Price, item.Quantity, Subtotal = product.Price * item.Quantity };
        }).ToList()
    }
).OrderBy(o => o.Date);

foreach (var order in orderSummaries)
{
    Console.WriteLine($"\nOrder #{order.OrderId} - {order.CustomerName} ({order.Date:d})");
    foreach (var item in order.Items)
        Console.WriteLine($"  {item.Name,-15} {item.Quantity:D2} x {item.Price:C} = {item.Subtotal:C}");
    Console.WriteLine($"  Total: {order.Total:C}");
}

// Category summary
var categorySummary = products
    .GroupBy(p => p.Category)
    .Select(g => new
    {
        Category = g.Key,
        ProductCount = g.Count(),
        TotalValue = g.Sum(p => p.Price * p.Stock),
        AvgPrice = g.Average(p => (double)p.Price),
        MinPrice = g.Min(p => p.Price),
        MaxPrice = g.Max(p => p.Price),
        Products = g.OrderBy(p => p.Price).Select(p => p.Name).ToList()
    })
    .OrderByDescending(c => c.TotalValue);

Console.WriteLine("\n=== Category Summary ===");
foreach (var cat in categorySummary)
    Console.WriteLine($"{cat.Category}: {cat.ProductCount} products, Total Value: {cat.TotalValue:C}, Avg: {cat.AvgPrice:C}");
```

---

## Step 332: LINQ to XML

```csharp
using System.Xml.Linq;

// สร้าง XML ด้วย LINQ to XML
XDocument doc = new XDocument(
    new XElement("Catalog",
        new XElement("Product",
            new XAttribute("Id", 1),
            new XElement("Name", "Laptop"),
            new XElement("Price", 1200),
            new XElement("Category", "Electronics")
        ),
        new XElement("Product",
            new XAttribute("Id", 2),
            new XElement("Name", "Phone"),
            new XElement("Price", 800),
            new XElement("Category", "Electronics")
        ),
        new XElement("Product",
            new XAttribute("Id", 3),
            new XElement("Name", "Desk"),
            new XElement("Price", 350),
            new XElement("Category", "Furniture")
        )
    )
);

// Save and Load
doc.Save("catalog.xml");
XDocument loaded = XDocument.Load("catalog.xml");

// Query XML with LINQ
var electronics = from p in loaded.Root!.Elements("Product")
                  where (string)p.Element("Category")! == "Electronics"
                  orderby (decimal)p.Element("Price")!
                  select new
                  {
                      Id = (int)p.Attribute("Id")!,
                      Name = (string)p.Element("Name")!,
                      Price = (decimal)p.Element("Price")!
                  };

foreach (var p in electronics)
    Console.WriteLine($"[{p.Id}] {p.Name}: {p.Price:C}");

// Method syntax
var expensive = loaded.Root!
    .Elements("Product")
    .Where(p => (decimal)p.Element("Price")! > 500)
    .Select(p => (string)p.Element("Name")!);

// Parse XML string
string xmlString = """
    <People>
        <Person age="30"><Name>Alice</Name></Person>
        <Person age="25"><Name>Bob</Name></Person>
    </People>
    """;

XElement people = XElement.Parse(xmlString);
var names = people.Elements("Person")
    .Where(p => (int)p.Attribute("age")! >= 28)
    .Select(p => (string)p.Element("Name")!);
// ["Alice"]

// Modify XML
XElement catalog = loaded.Root!;
catalog.Add(new XElement("Product",
    new XAttribute("Id", 4),
    new XElement("Name", "Tablet"),
    new XElement("Price", 600),
    new XElement("Category", "Electronics")
));

// Update element
XElement? firstProduct = catalog.Elements("Product").FirstOrDefault();
firstProduct?.Element("Price")?.SetValue(1100);

// Transform XML to objects
record ProductXml(int Id, string Name, decimal Price, string Category);
var productsList = catalog.Elements("Product").Select(p => new ProductXml(
    (int)p.Attribute("Id")!,
    (string)p.Element("Name")!,
    (decimal)p.Element("Price")!,
    (string)p.Element("Category")!
)).ToList();
```

---

## Step 333: Parallel LINQ (PLINQ)

```csharp
using System.Linq;

// PLINQ ใช้ .AsParallel() เพื่อ process ข้อมูลแบบ parallel
int[] numbers = Enumerable.Range(1, 1_000_000).ToArray();

// Sequential
var slowResult = numbers
    .Where(n => n % 2 == 0)
    .Select(n => n * n)
    .Sum();

// Parallel (faster on multi-core)
var fastResult = numbers
    .AsParallel()
    .Where(n => n % 2 == 0)
    .Select(n => n * n)
    .Sum();

// PLINQ with degree of parallelism
var controlled = numbers
    .AsParallel()
    .WithDegreeOfParallelism(4)  // use max 4 threads
    .Where(n => IsPrime(n))
    .Take(100)
    .ToList();

// Ordered parallel (preserve input order)
var ordered = numbers
    .AsParallel()
    .AsOrdered()    // preserve order (slower)
    .Where(n => n % 3 == 0)
    .Take(10)
    .ToList();

// Unordered (faster, may reorder)
var unordered = numbers
    .AsParallel()
    .AsUnordered()   // explicit (default)
    .Where(n => n % 3 == 0)
    .Take(10)
    .ToList();

// Error handling in PLINQ
try
{
    var result = numbers
        .AsParallel()
        .Select(n => n % 5 == 0 ? throw new Exception($"Bad {n}") : n)
        .ToList();
}
catch (AggregateException ae)
{
    foreach (var ex in ae.InnerExceptions)
        Console.WriteLine(ex.Message);
}

// ForAll — parallel foreach (no result collection)
numbers.AsParallel()
    .Where(n => n % 1000 == 0)
    .ForAll(n => Console.WriteLine($"Processing {n} on thread {Environment.CurrentManagedThreadId}"));

// Cancel PLINQ
using var cts = new CancellationTokenSource();
cts.CancelAfter(1000); // cancel after 1 second

try
{
    var result2 = numbers
        .AsParallel()
        .WithCancellation(cts.Token)
        .Select(n => SimulateWork(n))
        .ToList();
}
catch (OperationCanceledException)
{
    Console.WriteLine("PLINQ cancelled");
}

// When to use PLINQ:
// - Large datasets (10,000+ elements)
// - CPU-bound operations
// - Order doesn't matter (or AsOrdered is acceptable)
// - Independent operations (no shared state)

static bool IsPrime(int n)
{
    if (n < 2) return false;
    for (int i = 2; i <= Math.Sqrt(n); i++)
        if (n % i == 0) return false;
    return true;
}

static int SimulateWork(int n) { Thread.Sleep(1); return n; }
```

---

## Step 334: Custom LINQ Operators

```csharp
// Extension methods สำหรับสร้าง custom LINQ operators

public static class LinqExtensions
{
    // Batch / Chunk alternative
    public static IEnumerable<IEnumerable<T>> Batch<T>(this IEnumerable<T> source, int size)
    {
        var batch = new List<T>(size);
        foreach (var item in source)
        {
            batch.Add(item);
            if (batch.Count == size)
            {
                yield return batch;
                batch = new List<T>(size);
            }
        }
        if (batch.Count > 0)
            yield return batch;
    }
    
    // ForEach (side effects)
    public static void ForEach<T>(this IEnumerable<T> source, Action<T> action)
    {
        foreach (var item in source) action(item);
    }
    
    // ForEach with index
    public static void ForEach<T>(this IEnumerable<T> source, Action<T, int> action)
    {
        int i = 0;
        foreach (var item in source) action(item, i++);
    }
    
    // Flatten
    public static IEnumerable<T> Flatten<T>(this IEnumerable<IEnumerable<T>> source)
        => source.SelectMany(x => x);
    
    // Transpose (matrix)
    public static IEnumerable<IEnumerable<T>> Transpose<T>(this IEnumerable<IEnumerable<T>> matrix)
    {
        var arrays = matrix.Select(row => row.ToArray()).ToArray();
        int cols = arrays.Max(row => row.Length);
        return Enumerable.Range(0, cols).Select(col => arrays.Select(row => col < row.Length ? row[col] : default!));
    }
    
    // Window / Sliding window
    public static IEnumerable<IEnumerable<T>> Window<T>(this IEnumerable<T> source, int size)
    {
        var buffer = new Queue<T>(size);
        foreach (var item in source)
        {
            buffer.Enqueue(item);
            if (buffer.Count == size)
            {
                yield return buffer.ToArray();
                buffer.Dequeue();
            }
        }
    }
    
    // Pairwise
    public static IEnumerable<(T First, T Second)> Pairwise<T>(this IEnumerable<T> source)
    {
        T? prev = default;
        bool first = true;
        foreach (var item in source)
        {
            if (!first) yield return (prev!, item);
            prev = item;
            first = false;
        }
    }
    
    // Tap — for debugging pipeline
    public static IEnumerable<T> Tap<T>(this IEnumerable<T> source, Action<T> action)
    {
        foreach (var item in source)
        {
            action(item);
            yield return item;
        }
    }
    
    // Null-safe
    public static IEnumerable<T> EmptyIfNull<T>(this IEnumerable<T>? source)
        => source ?? Enumerable.Empty<T>();
    
    // WhereNotNull
    public static IEnumerable<T> WhereNotNull<T>(this IEnumerable<T?> source) where T : class
        => source.Where(x => x != null)!;
    
    // Index pairs
    public static IEnumerable<(int Index, T Item)> WithIndex<T>(this IEnumerable<T> source)
        => source.Select((item, i) => (i, item));
}

// Usage
var numbers = Enumerable.Range(1, 20);

// Batch
foreach (var batch in numbers.Batch(5))
    Console.WriteLine(string.Join(", ", batch));

// Window / Rolling average
var prices = new double[] { 10, 12, 11, 13, 15, 14, 16, 18 };
var movingAvg = prices.Window(3).Select(w => w.Average());
// (10+12+11)/3, (12+11+13)/3, ...

// Pairwise differences
var diffs = prices.Pairwise().Select(p => p.Second - p.First);
// 2, -1, 2, 2, -1, 2, 2

// Debug pipeline
var debug = numbers
    .Where(n => n % 2 == 0)
    .Tap(n => Console.WriteLine($"After where: {n}"))
    .Select(n => n * n)
    .Tap(n => Console.WriteLine($"After select: {n}"))
    .Take(3)
    .ToList();

// WithIndex
numbers.WithIndex().ForEach((item, i) => Console.WriteLine($"{i}: {item.Item}"));
```

---

## Step 335: LINQ Performance Patterns

```csharp
// 1. Avoid multiple enumeration
IEnumerable<int> query = GetData(); // returns IEnumerable

// Bad: enumerates twice
if (query.Any())
    Console.WriteLine(query.First());

// Good: materialize once
var list = query.ToList();
if (list.Count > 0)
    Console.WriteLine(list[0]);

// 2. Use Count property instead of Count() for collections
List<int> numbers = GetNumbers();
// Bad: iterates all elements
if (numbers.Count() > 0) { }
// Good: O(1) property
if (numbers.Count > 0) { }

// 3. Use Any() instead of Count() == 0
// Bad: counts all
if (numbers.Count() == 0) { }
// Good: stops at first element
if (!numbers.Any()) { }

// 4. Short-circuit with First()/Any() instead of Where().First()
// Bad: filters all, then takes first
var first = numbers.Where(n => n > 5).First();
// Good: stops at first match
var firstBetter = numbers.First(n => n > 5);

// 5. Use indexed access for lists
// Bad: iterates to find element
var third = numbers.ElementAt(2);
// Good: direct index access
var thirdBetter = numbers[2];

// 6. Prefer specialized methods over general LINQ
// For sorted collections, use BinarySearch instead of FirstOrDefault
List<int> sorted = Enumerable.Range(1, 1000).ToList();
// Bad:
int found1 = sorted.FirstOrDefault(n => n == 500);
// Better (O(log n) vs O(n)):
int index = sorted.BinarySearch(500);

// 7. ToHashSet() for O(1) lookups
List<int> ids = GetIds();
HashSet<int> idSet = ids.ToHashSet(); // O(1) Contains

// Bad: O(n) Contains on list
var filtered = data.Where(d => ids.Contains(d.Id));
// Good: O(1) Contains with HashSet
var filteredFast = data.Where(d => idSet.Contains(d.Id));

// 8. Use Dictionary for fast groupBy lookups
// Bad: GroupBy creates IGrouping which requires enumeration
var grouped = data.GroupBy(d => d.Category).ToDictionary(g => g.Key, g => g.ToList());

// Use ToLookup when querying repeatedly
ILookup<string, Data> lookup = data.ToLookup(d => d.Category);
var electronics = lookup["Electronics"]; // O(1)
var furniture = lookup["Furniture"];     // O(1)

// 9. Where before Select (when possible)
// Bad: transform all, then filter
var result1 = data.Select(d => d.Transform()).Where(d => d.IsValid);
// Good: filter first, then transform fewer
var result2 = data.Where(d => d.IsValidRaw).Select(d => d.Transform());

// 10. Avoid closure captures of large objects in LINQ
// Bad: captures entire large object
var largeThing = GetLargeObject();
var result3 = data.Where(d => d.Id == largeThing.Id); // largeThing kept alive

// Good: capture only what's needed
var id = largeThing.Id;
largeThing = null; // can be GC'd
var result4 = data.Where(d => d.Id == id);

record Data(int Id, string Category, bool IsValidRaw) 
{
    public Data Transform() => this;
    public bool IsValid => IsValidRaw;
}
static IEnumerable<int> GetData() => Enumerable.Range(1, 10);
static List<int> GetNumbers() => new() { 1, 2, 3 };
static List<int> GetIds() => new() { 1, 2, 3 };
static object GetLargeObject() => new();
```

---

## Step 336: โปรแกรมตัวอย่าง — Sales Analytics System

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

// ============================================================
// Sales Analytics System — ระบบวิเคราะห์ยอดขาย
// ============================================================

record Product(int Id, string Name, string Category, decimal Price);
record Sale(int Id, int ProductId, int Quantity, DateTime Date, string Region);

class SalesAnalytics
{
    private readonly List<Product> _products;
    private readonly List<Sale> _sales;
    
    public SalesAnalytics()
    {
        _products = new List<Product>
        {
            new(1, "Laptop Pro",    "Electronics", 1299m),
            new(2, "Phone X",       "Electronics",  799m),
            new(3, "Tablet Air",    "Electronics",  599m),
            new(4, "Office Chair",  "Furniture",    399m),
            new(5, "Standing Desk", "Furniture",    699m),
            new(6, "Notebook A5",   "Stationery",    15m),
            new(7, "Pen Set",       "Stationery",    25m),
            new(8, "Headphones",    "Electronics",  249m),
        };
        
        var rng = new Random(42);
        var regions = new[] { "North", "South", "East", "West", "Central" };
        _sales = Enumerable.Range(1, 500).Select(i => new Sale(
            Id:        i,
            ProductId: rng.Next(1, 9),
            Quantity:  rng.Next(1, 6),
            Date:      DateTime.Today.AddDays(-rng.Next(365)),
            Region:    regions[rng.Next(regions.Length)]
        )).ToList();
    }
    
    // รายได้รวม
    public decimal TotalRevenue() => _sales.Join(
        _products, s => s.ProductId, p => p.Id,
        (s, p) => s.Quantity * p.Price
    ).Sum();
    
    // Top N สินค้าขายดี
    public IEnumerable<(string Name, int Units, decimal Revenue)> TopProducts(int n = 5)
        => _sales
            .GroupBy(s => s.ProductId)
            .Select(g =>
            {
                var product = _products.First(p => p.Id == g.Key);
                return (
                    Name:    product.Name,
                    Units:   g.Sum(s => s.Quantity),
                    Revenue: g.Sum(s => s.Quantity * product.Price)
                );
            })
            .OrderByDescending(x => x.Revenue)
            .Take(n);
    
    // ยอดขายตาม Category
    public IEnumerable<(string Category, decimal Revenue, double Percentage)> SalesByCategory()
    {
        var total = TotalRevenue();
        return _sales
            .Join(_products, s => s.ProductId, p => p.Id,
                (s, p) => new { p.Category, Revenue = s.Quantity * p.Price })
            .GroupBy(x => x.Category)
            .Select(g => (
                Category: g.Key,
                Revenue: g.Sum(x => x.Revenue),
                Percentage: (double)g.Sum(x => x.Revenue) / (double)total * 100
            ))
            .OrderByDescending(x => x.Revenue);
    }
    
    // ยอดขายรายเดือน
    public IEnumerable<(string Month, decimal Revenue)> MonthlySales()
        => _sales
            .Join(_products, s => s.ProductId, p => p.Id,
                (s, p) => new { s.Date.Year, s.Date.Month, Revenue = s.Quantity * p.Price })
            .GroupBy(x => new { x.Year, x.Month })
            .Select(g => (
                Month: new DateTime(g.Key.Year, g.Key.Month, 1).ToString("MMM yyyy"),
                Revenue: g.Sum(x => x.Revenue)
            ))
            .OrderBy(x => x.Month);
    
    // ยอดขายตาม Region
    public IEnumerable<(string Region, decimal Revenue, int Orders)> SalesByRegion()
        => _sales
            .Join(_products, s => s.ProductId, p => p.Id,
                (s, p) => new { s.Region, Revenue = s.Quantity * p.Price })
            .GroupBy(x => x.Region)
            .Select(g => (
                Region:  g.Key,
                Revenue: g.Sum(x => x.Revenue),
                Orders:  g.Count()
            ))
            .OrderByDescending(x => x.Revenue);
    
    // แนวโน้มยอดขาย (7-day moving average)
    public IEnumerable<(DateTime Date, decimal Daily, decimal MovingAvg)> SalesTrend()
    {
        var dailySales = _sales
            .Join(_products, s => s.ProductId, p => p.Id,
                (s, p) => new { s.Date.Date, Revenue = s.Quantity * p.Price })
            .GroupBy(x => x.Date)
            .Select(g => (Date: g.Key, Revenue: g.Sum(x => x.Revenue)))
            .OrderBy(x => x.Date)
            .ToList();
        
        return dailySales
            .Window(7)
            .Select((window, i) =>
            {
                var windowList = window.ToList();
                var last = windowList.Last();
                return (last.Date, Daily: last.Revenue, MovingAvg: windowList.Average(x => x.Revenue));
            });
    }
    
    // สินค้าที่ไม่มียอดขาย
    public IEnumerable<Product> UnselledProducts()
        => _products.ExceptBy(_sales.Select(s => s.ProductId).Distinct(), p => p.Id);
    
    // ยอดขายสูงสุดในแต่ละ region
    public IEnumerable<(string Region, string TopProduct, decimal Revenue)> RegionTopProduct()
        => _sales
            .Join(_products, s => s.ProductId, p => p.Id,
                (s, p) => new { s.Region, p.Name, Revenue = s.Quantity * p.Price })
            .GroupBy(x => x.Region)
            .Select(g => (
                Region:     g.Key,
                TopProduct: g.GroupBy(x => x.Name).MaxBy(x => x.Sum(y => y.Revenue))!.Key,
                Revenue:    g.Sum(x => x.Revenue)
            ));
    
    public void PrintReport()
    {
        Console.WriteLine("╔══════════════════════════════════════════════════════╗");
        Console.WriteLine("║          SALES ANALYTICS DASHBOARD                  ║");
        Console.WriteLine("╚══════════════════════════════════════════════════════╝");
        
        Console.WriteLine($"\n💰 Total Revenue: {TotalRevenue():C}");
        Console.WriteLine($"📦 Total Orders: {_sales.Count:N0}");
        Console.WriteLine($"📊 Products: {_products.Count}");
        
        Console.WriteLine("\n🏆 Top 5 Products:");
        Console.WriteLine($"{"Product",-20} {"Units",8} {"Revenue",12}");
        Console.WriteLine(new string('-', 42));
        foreach (var (name, units, revenue) in TopProducts())
            Console.WriteLine($"{name,-20} {units,8:N0} {revenue,12:C}");
        
        Console.WriteLine("\n📂 Sales by Category:");
        Console.WriteLine($"{"Category",-15} {"Revenue",12} {"Share",8}");
        Console.WriteLine(new string('-', 37));
        foreach (var (cat, rev, pct) in SalesByCategory())
            Console.WriteLine($"{cat,-15} {rev,12:C} {pct,8:F1}%");
        
        Console.WriteLine("\n🌍 Sales by Region:");
        Console.WriteLine($"{"Region",-10} {"Revenue",12} {"Orders",8}");
        Console.WriteLine(new string('-', 32));
        foreach (var (region, revenue, orders) in SalesByRegion())
            Console.WriteLine($"{region,-10} {revenue,12:C} {orders,8:N0}");
        
        var unsold = UnselledProducts().ToList();
        if (unsold.Any())
        {
            Console.WriteLine("\n⚠️  Unsold Products:");
            foreach (var p in unsold)
                Console.WriteLine($"  - {p.Name} ({p.Category})");
        }
    }
}

// Window extension from previous step
static class LINQExtensions
{
    public static IEnumerable<IEnumerable<T>> Window<T>(this IEnumerable<T> source, int size)
    {
        var buffer = new Queue<T>(size);
        foreach (var item in source)
        {
            buffer.Enqueue(item);
            if (buffer.Count == size)
            {
                yield return buffer.ToArray();
                buffer.Dequeue();
            }
        }
    }
}

// Run demo
var analytics = new SalesAnalytics();
analytics.PrintReport();
```

---

## สรุป Part 12

### LINQ Fundamentals
- ✅ **Deferred Execution** — query ไม่ execute จนกว่า enumerate
- ✅ Immediate execution ด้วย `ToList()`, `ToArray()`, `Count()`, `First()`
- ✅ Method Syntax vs Query Syntax

### Filtering
- ✅ `Where`, `OfType`, `Distinct`, `DistinctBy`
- ✅ `Skip`, `Take`, `SkipWhile`, `TakeWhile`, `SkipLast`, `TakeLast`
- ✅ `Chunk` (.NET 6+)

### Projection
- ✅ `Select`, `SelectMany`, `Zip`, `Cast`
- ✅ Anonymous types, records ใน projections

### Ordering
- ✅ `OrderBy`, `OrderByDescending`, `ThenBy`, `ThenByDescending`
- ✅ `Reverse`, custom `IComparer`

### Grouping
- ✅ `GroupBy` with key, element, result selectors
- ✅ `ToLookup` for fast repeated lookups

### Aggregation
- ✅ `Count`, `Sum`, `Min`, `Max`, `Average`
- ✅ `MinBy`, `MaxBy` (.NET 6+)
- ✅ `Aggregate` (general purpose reduce)
- ✅ `Any`, `All`, `Contains`

### Joining
- ✅ `Join` (inner), `GroupJoin` (left outer)
- ✅ Cross join, multi-key join

### Set Operations
- ✅ `Union`, `Intersect`, `Except`, `Distinct`
- ✅ `UnionBy`, `IntersectBy`, `ExceptBy`, `DistinctBy` (.NET 6+)

### PLINQ
- ✅ `AsParallel()`, `WithDegreeOfParallelism`, `AsOrdered`
- ✅ Error handling, cancellation

---

## แบบฝึกหัด

1. สร้าง LINQ query ที่หาผลิตภัณฑ์ที่ขายได้มากที่สุดในแต่ละเดือน
2. เขียน custom operator `RunningTotal<T>` ที่คำนวณยอดสะสม
3. ใช้ PLINQ เพื่อค้นหา prime numbers ใน range 1-10,000,000 และวัด performance
4. สร้าง `HierarchicalGroupBy` ที่จัดกลุ่ม 2 ระดับ เช่น Region → Category → Products
5. เขียน LINQ query ที่ใช้ `SelectMany` เพื่อ flatten JSON-like nested structure
