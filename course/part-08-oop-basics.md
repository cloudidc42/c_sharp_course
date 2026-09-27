# Part 08: OOP พื้นฐาน (Object-Oriented Programming)
## Steps 171-210: Classes, Objects, Properties, และ Encapsulation

---

## Step 171: Classes และ Objects

```csharp
// ============ Class Definition ============
// Class เป็น blueprint (แม่พิมพ์) ของ object

class BankAccount
{
    // ============ Fields (state) ============
    private string _owner;
    private decimal _balance;
    private List<string> _transactions = new();
    private static int _totalAccounts = 0; // static: shared ทุก instances
    
    // ============ Constructor ============
    public BankAccount(string owner, decimal initialBalance = 0)
    {
        if (string.IsNullOrWhiteSpace(owner))
            throw new ArgumentException("Owner name cannot be empty");
        if (initialBalance < 0)
            throw new ArgumentException("Initial balance cannot be negative");
        
        _owner = owner;
        _balance = initialBalance;
        AccountNumber = GenerateAccountNumber();
        CreatedAt = DateTime.Now;
        _totalAccounts++;
        
        if (initialBalance > 0)
            _transactions.Add($"Initial deposit: +{initialBalance:N2}");
    }
    
    // ============ Properties ============
    public string AccountNumber { get; private set; }
    public string Owner => _owner;
    public decimal Balance => _balance;
    public DateTime CreatedAt { get; }
    public static int TotalAccounts => _totalAccounts;
    
    // ============ Methods (behavior) ============
    public void Deposit(decimal amount)
    {
        ValidateAmount(amount);
        _balance += amount;
        _transactions.Add($"Deposit: +{amount:N2} | Balance: {_balance:N2}");
    }
    
    public bool Withdraw(decimal amount)
    {
        ValidateAmount(amount);
        
        if (_balance < amount)
        {
            Console.WriteLine($"Insufficient funds. Balance: {_balance:N2}");
            return false;
        }
        
        _balance -= amount;
        _transactions.Add($"Withdrawal: -{amount:N2} | Balance: {_balance:N2}");
        return true;
    }
    
    public bool Transfer(BankAccount target, decimal amount)
    {
        if (!Withdraw(amount)) return false;
        target.Deposit(amount);
        _transactions.Add($"Transfer to {target.AccountNumber}: -{amount:N2}");
        target._transactions.Add($"Transfer from {AccountNumber}: +{amount:N2}");
        return true;
    }
    
    public IEnumerable<string> GetTransactions() => _transactions.AsReadOnly();
    
    // ============ Private Helper ============
    private void ValidateAmount(decimal amount)
    {
        if (amount <= 0)
            throw new ArgumentException("Amount must be positive");
    }
    
    private static string GenerateAccountNumber()
    {
        return $"ACC-{DateTime.Now:yyyyMMdd}-{Random.Shared.Next(1000, 9999)}";
    }
    
    public override string ToString() =>
        $"Account: {AccountNumber} | Owner: {Owner} | Balance: ฿{Balance:N2}";
}

// ============ Creating Objects ============
var alice = new BankAccount("Alice", 10000);
var bob = new BankAccount("Bob", 5000);

Console.WriteLine(alice);
Console.WriteLine(bob);
Console.WriteLine($"Total accounts: {BankAccount.TotalAccounts}");

alice.Deposit(5000);
alice.Withdraw(2000);
alice.Transfer(bob, 3000);

Console.WriteLine("\nAlice's transactions:");
foreach (string tx in alice.GetTransactions())
    Console.WriteLine($"  {tx}");

Console.WriteLine($"\nFinal balances:");
Console.WriteLine(alice);
Console.WriteLine(bob);
```

---

## Step 172: Properties ครบถ้วน

```csharp
// ============ Property Types ============

class Person
{
    private string _name = "";
    private int _age;
    
    // 1. Auto-implemented Property
    public string Email { get; set; } = "";
    public string Phone { get; set; } = "";
    
    // 2. Auto-property with init (C# 9+) - can only set in constructor/initializer
    public string Id { get; init; } = Guid.NewGuid().ToString();
    
    // 3. Full Property with validation
    public string Name
    {
        get => _name;
        set
        {
            if (string.IsNullOrWhiteSpace(value))
                throw new ArgumentException("Name cannot be empty");
            _name = value.Trim();
        }
    }
    
    // 4. Property with validation
    public int Age
    {
        get => _age;
        set
        {
            if (value < 0 || value > 150)
                throw new ArgumentOutOfRangeException(nameof(Age), "Age must be 0-150");
            _age = value;
        }
    }
    
    // 5. Computed Property (read-only)
    public string FullInfo => $"{Name} (age {Age})";
    public bool IsAdult => Age >= 18;
    public int BirthYear => DateTime.Now.Year - Age;
    
    // 6. Property with private setter
    public DateTime RegisteredAt { get; private set; } = DateTime.Now;
    
    // 7. Static Property
    public static int TotalPersons { get; private set; } = 0;
    
    public Person(string name, int age)
    {
        Name = name;
        Age = age;
        TotalPersons++;
    }
}

// Object Initializer Syntax
var p1 = new Person("Alice", 30) { Email = "alice@example.com", Phone = "081-234-5678" };
Console.WriteLine(p1.FullInfo);
Console.WriteLine($"Is Adult: {p1.IsAdult}");
Console.WriteLine($"Birth Year: {p1.BirthYear}");

// ============ Required Properties (C# 11) ============
class Configuration
{
    public required string Host { get; set; }  // ต้องกำหนดค่าตอนสร้าง
    public required int Port { get; set; }
    public string? Database { get; set; }
    public bool UseSsl { get; set; } = true;
}

var config = new Configuration
{
    Host = "localhost",
    Port = 5432,
    Database = "mydb"
};
Console.WriteLine($"Config: {config.Host}:{config.Port}/{config.Database}");

// ============ Record Properties ============
record Employee(int Id, string Name, string Department, decimal Salary)
{
    // เพิ่ม property ใน record
    public string DisplayName => $"[{Id}] {Name} ({Department})";
    
    // Validation ใน constructor
    public Employee : this(Id, Name, Department, Salary)
    {
        if (Salary < 0) throw new ArgumentException("Salary cannot be negative");
    }
}

var emp = new Employee(1, "Alice", "Engineering", 75000);
Console.WriteLine(emp.DisplayName);
Console.WriteLine($"Salary: {emp.Salary:N0}");

// With expression
var promoted = emp with { Department = "Senior Engineering", Salary = 90000 };
Console.WriteLine($"Promoted: {promoted.DisplayName}");

// ============ Property Indexers ============
class Matrix
{
    private double[,] _data;
    public int Rows { get; }
    public int Cols { get; }
    
    public Matrix(int rows, int cols)
    {
        Rows = rows;
        Cols = cols;
        _data = new double[rows, cols];
    }
    
    // Indexer
    public double this[int row, int col]
    {
        get
        {
            ValidateBounds(row, col);
            return _data[row, col];
        }
        set
        {
            ValidateBounds(row, col);
            _data[row, col] = value;
        }
    }
    
    private void ValidateBounds(int row, int col)
    {
        if (row < 0 || row >= Rows || col < 0 || col >= Cols)
            throw new IndexOutOfRangeException($"Index ({row},{col}) out of bounds");
    }
}

var m = new Matrix(3, 3);
m[0, 0] = 1; m[1, 1] = 5; m[2, 2] = 9;
Console.WriteLine($"Matrix[1,1] = {m[1, 1]}");
```

---

## Step 173: Constructors ครบถ้วน

```csharp
// ============ Constructor Types ============

class Vehicle
{
    public string Make { get; }
    public string Model { get; }
    public int Year { get; }
    public string Color { get; private set; }
    public decimal Price { get; private set; }
    
    // ============ 1. Default Constructor ============
    public Vehicle()
    {
        Make = "Unknown";
        Model = "Unknown";
        Year = DateTime.Now.Year;
        Color = "White";
        Price = 0;
    }
    
    // ============ 2. Parameterized Constructor ============
    public Vehicle(string make, string model, int year)
    {
        Make = make;
        Model = model;
        Year = year;
        Color = "White";
        Price = 0;
    }
    
    // ============ 3. Constructor Chaining (this) ============
    public Vehicle(string make, string model, int year, string color)
        : this(make, model, year)  // เรียก constructor ก่อนหน้า
    {
        Color = color;
    }
    
    public Vehicle(string make, string model, int year, string color, decimal price)
        : this(make, model, year, color)
    {
        Price = price;
    }
    
    // ============ 4. Copy Constructor ============
    public Vehicle(Vehicle other)
    {
        Make = other.Make;
        Model = other.Model;
        Year = other.Year;
        Color = other.Color;
        Price = other.Price;
    }
    
    // ============ 5. Static Constructor (runs once, before first use) ============
    static Vehicle()
    {
        Console.WriteLine("Vehicle class initialized (static constructor)");
        // ใช้สำหรับ initialize static members
    }
    
    public override string ToString() =>
        $"{Year} {Make} {Model} ({Color}) - ฿{Price:N0}";
}

var v1 = new Vehicle();
var v2 = new Vehicle("Toyota", "Camry", 2024);
var v3 = new Vehicle("Honda", "Civic", 2024, "Blue");
var v4 = new Vehicle("BMW", "M3", 2024, "Black", 3500000);
var v5 = new Vehicle(v4); // Copy

Console.WriteLine(v1);
Console.WriteLine(v2);
Console.WriteLine(v3);
Console.WriteLine(v4);
Console.WriteLine(v5);

// ============ Primary Constructor (C# 12) ============
class DatabaseConfig(string host, int port, string database)
{
    // primary constructor parameters สามารถใช้ในทั้ง class
    public string ConnectionString =>
        $"Server={host};Port={port};Database={database}";
    
    public string Host => host;
    public int Port => port;
    public string Database => database;
    
    public override string ToString() => ConnectionString;
}

var dbConfig = new DatabaseConfig("localhost", 5432, "myapp");
Console.WriteLine(dbConfig);

// ============ Deconstruct ============
class Point3D
{
    public double X { get; }
    public double Y { get; }
    public double Z { get; }
    
    public Point3D(double x, double y, double z) => (X, Y, Z) = (x, y, z);
    
    // Deconstruct method
    public void Deconstruct(out double x, out double y, out double z)
        => (x, y, z) = (X, Y, Z);
    
    public override string ToString() => $"({X}, {Y}, {Z})";
}

var point = new Point3D(1.5, 2.5, 3.5);
var (x, y, z) = point; // Deconstruction
Console.WriteLine($"x={x}, y={y}, z={z}");
```

---

## Step 174: Static Members

```csharp
// ============ Static Class ============
public static class MathHelper
{
    // Static constants
    public const double GoldenRatio = 1.6180339887;
    public const double EulerNumber = 2.71828182845;
    
    // Static properties
    public static double Pi => Math.PI;
    
    // Static methods
    public static bool IsPrime(int n)
    {
        if (n < 2) return false;
        for (int i = 2; i * i <= n; i++)
            if (n % i == 0) return false;
        return true;
    }
    
    public static IEnumerable<int> GetPrimes(int max)
        => Enumerable.Range(2, max - 1).Where(IsPrime);
    
    public static double CircleArea(double radius) => Pi * radius * radius;
    
    public static long Fibonacci(int n)
    {
        if (n <= 1) return n;
        long a = 0, b = 1;
        for (int i = 2; i <= n; i++)
            (a, b) = (b, a + b);
        return b;
    }
    
    public static int GCD(int a, int b) => b == 0 ? a : GCD(b, a % b);
    public static int LCM(int a, int b) => a / GCD(a, b) * b;
    
    public static double Clamp(double value, double min, double max)
        => Math.Max(min, Math.Min(max, value));
}

Console.WriteLine($"Pi: {MathHelper.Pi:F10}");
Console.WriteLine($"Golden Ratio: {MathHelper.GoldenRatio}");
Console.WriteLine($"Is 17 prime: {MathHelper.IsPrime(17)}");
Console.WriteLine($"Primes < 50: {string.Join(", ", MathHelper.GetPrimes(50))}");
Console.WriteLine($"Fib(10): {MathHelper.Fibonacci(10)}");
Console.WriteLine($"GCD(48,18): {MathHelper.GCD(48, 18)}");
Console.WriteLine($"LCM(12,18): {MathHelper.LCM(12, 18)}");

// ============ Singleton Pattern ============
class AppConfig
{
    private static AppConfig? _instance;
    private static readonly object _lock = new();
    
    private Dictionary<string, string> _settings = new();
    
    // Private constructor - ป้องกันการสร้างจากภายนอก
    private AppConfig()
    {
        // Load default settings
        _settings["theme"] = "dark";
        _settings["language"] = "th";
        _settings["version"] = "1.0.0";
    }
    
    // Thread-safe singleton
    public static AppConfig Instance
    {
        get
        {
            if (_instance == null)
            {
                lock (_lock)
                {
                    _instance ??= new AppConfig();
                }
            }
            return _instance;
        }
    }
    
    public string this[string key]
    {
        get => _settings.TryGetValue(key, out string? val) ? val : "";
        set => _settings[key] = value;
    }
    
    public void Set(string key, string value) => _settings[key] = value;
    public string Get(string key, string defaultValue = "") =>
        _settings.TryGetValue(key, out string? val) ? val : defaultValue;
}

var config1 = AppConfig.Instance;
var config2 = AppConfig.Instance;
Console.WriteLine($"Same instance: {ReferenceEquals(config1, config2)}"); // True

config1["theme"] = "light";
Console.WriteLine($"Theme from config2: {config2["theme"]}"); // light (same object)
```

---

## Step 175: Encapsulation และ Access Modifiers

```csharp
// ============ Access Modifiers ============
// public     - ทุกที่
// private    - เฉพาะ class นี้ (default สำหรับ members)
// protected  - class นี้ + subclasses
// internal   - ใน assembly เดียวกัน
// protected internal - ใน assembly หรือ subclasses
// private protected  - class นี้ + subclasses ใน assembly เดียวกัน

class Employee
{
    // private - ซ่อนจากภายนอก
    private string _ssn; // Social Security Number - sensitive
    private decimal _baseSalary;
    private List<string> _performanceNotes = new();
    
    // public - เข้าถึงได้จากภายนอก
    public int Id { get; }
    public string Name { get; private set; }
    public string Department { get; set; }
    
    // protected - สำหรับ inheritance
    protected string EmployeeCode { get; private set; }
    
    // internal - ใน assembly นี้เท่านั้น
    internal DateTime HireDate { get; private set; }
    
    public Employee(int id, string name, string department, string ssn, decimal salary)
    {
        Id = id;
        Name = name;
        Department = department;
        _ssn = ssn;  // เก็บ sensitive data แบบ private
        _baseSalary = salary;
        EmployeeCode = GenerateCode();
        HireDate = DateTime.Now;
    }
    
    // Public interface - ให้ access แบบ controlled
    public decimal GetSalary() => _baseSalary; // อ่านได้
    
    public bool RequestSalaryIncrease(decimal percentage)
    {
        if (percentage < 0 || percentage > 50)
            return false;
        _baseSalary *= (1 + percentage / 100);
        return true;
    }
    
    // ซ่อน SSN แต่ให้ตรวจสอบได้
    public bool VerifySSN(string ssn) => _ssn == ssn;
    public string GetMaskedSSN() => "***-**-" + _ssn[^4..];
    
    public void AddPerformanceNote(string note) =>
        _performanceNotes.Add($"[{DateTime.Now:yyyy-MM-dd}] {note}");
    
    public IReadOnlyList<string> GetPerformanceNotes() =>
        _performanceNotes.AsReadOnly();
    
    private string GenerateCode() =>
        $"EMP-{Department[..3].ToUpper()}-{Id:0000}";
    
    public override string ToString() =>
        $"[{EmployeeCode}] {Name} | {Department} | ฿{_baseSalary:N0}";
}

var emp = new Employee(1, "Alice Smith", "Engineering", "123-45-6789", 80000);
Console.WriteLine(emp);
Console.WriteLine($"SSN (masked): {emp.GetMaskedSSN()}");
Console.WriteLine($"SSN verify: {emp.VerifySSN("123-45-6789")}");

emp.RequestSalaryIncrease(10);
emp.AddPerformanceNote("Excellent performance Q1");
emp.AddPerformanceNote("Promoted to Senior Engineer");

Console.WriteLine($"After raise: {emp}");
Console.WriteLine("Notes:");
foreach (string note in emp.GetPerformanceNotes())
    Console.WriteLine($"  {note}");

// ============ Immutable Class Pattern ============
sealed class Money
{
    public decimal Amount { get; }
    public string Currency { get; }
    
    public Money(decimal amount, string currency)
    {
        if (amount < 0) throw new ArgumentException("Amount cannot be negative");
        if (string.IsNullOrEmpty(currency)) throw new ArgumentException("Currency required");
        Amount = Math.Round(amount, 2);
        Currency = currency.ToUpper();
    }
    
    // Operations return new Money (immutable)
    public Money Add(Money other)
    {
        if (Currency != other.Currency) throw new InvalidOperationException("Currency mismatch");
        return new Money(Amount + other.Amount, Currency);
    }
    
    public Money Subtract(Money other)
    {
        if (Currency != other.Currency) throw new InvalidOperationException("Currency mismatch");
        return new Money(Amount - other.Amount, Currency);
    }
    
    public Money Multiply(decimal factor) => new Money(Amount * factor, Currency);
    
    public static Money operator +(Money a, Money b) => a.Add(b);
    public static Money operator -(Money a, Money b) => a.Subtract(b);
    public static Money operator *(Money m, decimal f) => m.Multiply(f);
    public static bool operator ==(Money a, Money b) =>
        a.Amount == b.Amount && a.Currency == b.Currency;
    public static bool operator !=(Money a, Money b) => !(a == b);
    
    public override string ToString() => $"{Amount:N2} {Currency}";
    public override bool Equals(object? obj) => obj is Money m && this == m;
    public override int GetHashCode() => HashCode.Combine(Amount, Currency);
}

var price = new Money(1000, "THB");
var tax = new Money(70, "THB");
var total = price + tax;
var discounted = total * 0.9m;

Console.WriteLine($"Price: {price}");
Console.WriteLine($"Tax: {tax}");
Console.WriteLine($"Total: {total}");
Console.WriteLine($"Discounted: {discounted}");
```

---

## Step 176-185: Advanced OOP Patterns

```csharp
// ============ Builder Pattern ============
class QueryBuilder
{
    private string _table = "";
    private List<string> _conditions = new();
    private List<string> _columns = new() { "*" };
    private string? _orderBy;
    private bool _orderAsc = true;
    private int? _limit;
    private int? _offset;
    
    public QueryBuilder From(string table)
    {
        _table = table;
        return this;
    }
    
    public QueryBuilder Select(params string[] columns)
    {
        _columns = columns.ToList();
        return this;
    }
    
    public QueryBuilder Where(string condition)
    {
        _conditions.Add(condition);
        return this;
    }
    
    public QueryBuilder OrderBy(string column, bool ascending = true)
    {
        _orderBy = column;
        _orderAsc = ascending;
        return this;
    }
    
    public QueryBuilder Limit(int limit)
    {
        _limit = limit;
        return this;
    }
    
    public QueryBuilder Offset(int offset)
    {
        _offset = offset;
        return this;
    }
    
    public string Build()
    {
        var sb = new System.Text.StringBuilder();
        sb.Append($"SELECT {string.Join(", ", _columns)}");
        sb.Append($" FROM {_table}");
        
        if (_conditions.Count > 0)
            sb.Append($" WHERE {string.Join(" AND ", _conditions)}");
        
        if (_orderBy != null)
            sb.Append($" ORDER BY {_orderBy} {(_orderAsc ? "ASC" : "DESC")}");
        
        if (_limit.HasValue)
            sb.Append($" LIMIT {_limit}");
        
        if (_offset.HasValue)
            sb.Append($" OFFSET {_offset}");
        
        return sb.ToString();
    }
}

string query = new QueryBuilder()
    .From("products")
    .Select("id", "name", "price", "stock")
    .Where("price > 1000")
    .Where("stock > 0")
    .OrderBy("price", ascending: false)
    .Limit(10)
    .Offset(0)
    .Build();

Console.WriteLine($"Query: {query}");

// ============ Fluent Interface ============
class EmailBuilder
{
    private string _from = "";
    private List<string> _to = new();
    private string _subject = "";
    private string _body = "";
    private List<string> _attachments = new();
    private bool _isHtml = false;
    
    public EmailBuilder From(string email) { _from = email; return this; }
    public EmailBuilder To(params string[] emails) { _to.AddRange(emails); return this; }
    public EmailBuilder Subject(string subject) { _subject = subject; return this; }
    public EmailBuilder Body(string body, bool isHtml = false)
    {
        _body = body;
        _isHtml = isHtml;
        return this;
    }
    public EmailBuilder Attach(string file) { _attachments.Add(file); return this; }
    
    public void Send()
    {
        Console.WriteLine($"\n📧 Sending Email:");
        Console.WriteLine($"  From: {_from}");
        Console.WriteLine($"  To: {string.Join(", ", _to)}");
        Console.WriteLine($"  Subject: {_subject}");
        Console.WriteLine($"  Body ({(_isHtml ? "HTML" : "Plain")}): {_body[..Math.Min(50, _body.Length)]}...");
        if (_attachments.Count > 0)
            Console.WriteLine($"  Attachments: {string.Join(", ", _attachments)}");
        Console.WriteLine("  ✅ Sent!");
    }
}

new EmailBuilder()
    .From("admin@example.com")
    .To("alice@example.com", "bob@example.com")
    .Subject("Monthly Report")
    .Body("<h1>Hello</h1><p>Please find the attached report.</p>", isHtml: true)
    .Attach("report.pdf")
    .Attach("data.xlsx")
    .Send();
```

---

## Step 186-210: โปรแกรม Library Management System

```csharp
// ============ Library Management System ============

// ============ Models ============
record Author(int Id, string Name, string Nationality);

class Book
{
    public int Id { get; }
    public string ISBN { get; }
    public string Title { get; private set; }
    public Author Author { get; private set; }
    public string Genre { get; private set; }
    public int TotalCopies { get; private set; }
    public int AvailableCopies { get; private set; }
    public decimal Price { get; private set; }
    public DateTime AddedDate { get; } = DateTime.Now;
    private static int _nextId = 1;
    
    public Book(string isbn, string title, Author author, string genre, int copies, decimal price)
    {
        Id = _nextId++;
        ISBN = isbn;
        Title = title;
        Author = author;
        Genre = genre;
        TotalCopies = copies;
        AvailableCopies = copies;
        Price = price;
    }
    
    public bool Checkout()
    {
        if (AvailableCopies <= 0) return false;
        AvailableCopies--;
        return true;
    }
    
    public void Return() => AvailableCopies = Math.Min(TotalCopies, AvailableCopies + 1);
    
    public bool IsAvailable => AvailableCopies > 0;
    
    public override string ToString() =>
        $"[{Id}] \"{Title}\" by {Author.Name} ({Genre}) - {AvailableCopies}/{TotalCopies}";
}

class Member
{
    public int Id { get; }
    public string Name { get; set; }
    public string Email { get; set; }
    public MemberType Type { get; set; }
    public DateTime JoinDate { get; } = DateTime.Now;
    private List<Loan> _activeLoans = new();
    private List<Loan> _loanHistory = new();
    private static int _nextId = 1;
    
    public Member(string name, string email, MemberType type = MemberType.Regular)
    {
        Id = _nextId++;
        Name = name;
        Email = email;
        Type = type;
    }
    
    public int MaxLoans => Type switch
    {
        MemberType.Student => 3,
        MemberType.Regular => 5,
        MemberType.Premium => 10,
        MemberType.Staff => 20,
        _ => 3
    };
    
    public bool CanBorrow => _activeLoans.Count < MaxLoans;
    public int ActiveLoanCount => _activeLoans.Count;
    
    public void AddLoan(Loan loan) => _activeLoans.Add(loan);
    public void ReturnBook(Loan loan)
    {
        _activeLoans.Remove(loan);
        _loanHistory.Add(loan);
    }
    
    public IReadOnlyList<Loan> GetActiveLoans() => _activeLoans.AsReadOnly();
    public IReadOnlyList<Loan> GetHistory() => _loanHistory.AsReadOnly();
    
    public override string ToString() =>
        $"[{Id}] {Name} ({Type}) | Active: {ActiveLoanCount}/{MaxLoans}";
}

enum MemberType { Student, Regular, Premium, Staff }

class Loan
{
    public int Id { get; }
    public Book Book { get; }
    public Member Member { get; }
    public DateTime CheckoutDate { get; } = DateTime.Now;
    public DateTime DueDate { get; }
    public DateTime? ReturnDate { get; private set; }
    private static int _nextId = 1;
    
    public Loan(Book book, Member member, int durationDays = 14)
    {
        Id = _nextId++;
        Book = book;
        Member = member;
        DueDate = CheckoutDate.AddDays(durationDays);
    }
    
    public bool IsOverdue => !IsReturned && DateTime.Now > DueDate;
    public bool IsReturned => ReturnDate.HasValue;
    
    public decimal CalculateFine()
    {
        if (!IsOverdue) return 0;
        var overdueDays = (int)(DateTime.Now - DueDate).TotalDays;
        return overdueDays * 5; // 5 baht per day
    }
    
    public void Return() => ReturnDate = DateTime.Now;
    
    public override string ToString() =>
        $"Loan#{Id}: \"{Book.Title}\" by {Member.Name} | Due: {DueDate:dd/MM/yyyy} | {(IsReturned ? "Returned" : IsOverdue ? "OVERDUE" : "Active")}";
}

class Library
{
    private List<Book> _books = new();
    private List<Member> _members = new();
    private List<Loan> _loans = new();
    
    public string Name { get; }
    
    public Library(string name) => Name = name;
    
    // ============ Book Management ============
    public void AddBook(Book book) => _books.Add(book);
    
    public Book? FindBook(string query)
    {
        query = query.ToLower();
        return _books.FirstOrDefault(b =>
            b.Title.ToLower().Contains(query) ||
            b.ISBN.Contains(query) ||
            b.Author.Name.ToLower().Contains(query));
    }
    
    public IEnumerable<Book> SearchBooks(string query)
    {
        query = query.ToLower();
        return _books.Where(b =>
            b.Title.ToLower().Contains(query) ||
            b.Author.Name.ToLower().Contains(query) ||
            b.Genre.ToLower().Contains(query));
    }
    
    // ============ Member Management ============
    public void AddMember(Member member) => _members.Add(member);
    public Member? FindMember(int id) => _members.FirstOrDefault(m => m.Id == id);
    
    // ============ Loan Management ============
    public (bool Success, string Message, Loan? Loan) CheckoutBook(int memberId, string bookQuery)
    {
        var member = FindMember(memberId);
        if (member == null) return (false, "Member not found", null);
        
        if (!member.CanBorrow)
            return (false, $"Member has reached max loans ({member.MaxLoans})", null);
        
        var book = FindBook(bookQuery);
        if (book == null) return (false, "Book not found", null);
        
        if (!book.IsAvailable)
            return (false, $"'{book.Title}' is not available", null);
        
        if (!book.Checkout()) return (false, "Failed to checkout", null);
        
        var loan = new Loan(book, member, member.Type == MemberType.Premium ? 21 : 14);
        member.AddLoan(loan);
        _loans.Add(loan);
        
        return (true, $"Checked out successfully. Due: {loan.DueDate:dd/MM/yyyy}", loan);
    }
    
    public (bool Success, string Message, decimal Fine) ReturnBook(int loanId)
    {
        var loan = _loans.FirstOrDefault(l => l.Id == loanId);
        if (loan == null) return (false, "Loan not found", 0);
        if (loan.IsReturned) return (false, "Book already returned", 0);
        
        decimal fine = loan.CalculateFine();
        loan.Return();
        loan.Book.Return();
        loan.Member.ReturnBook(loan);
        
        string message = fine > 0
            ? $"Returned with fine: ฿{fine}"
            : "Returned successfully";
        
        return (true, message, fine);
    }
    
    // ============ Reports ============
    public void PrintReport()
    {
        Console.WriteLine($"\n{'=':═^60}");
        Console.WriteLine($"  📚 {Name} - Report");
        Console.WriteLine($"{'=':═^60}");
        
        Console.WriteLine($"\n📖 Books: {_books.Count}");
        Console.WriteLine($"  Available: {_books.Count(b => b.IsAvailable)}");
        Console.WriteLine($"  All borrowed: {_books.Count(b => !b.IsAvailable)}");
        
        Console.WriteLine($"\n👥 Members: {_members.Count}");
        foreach (MemberType type in Enum.GetValues<MemberType>())
            Console.WriteLine($"  {type}: {_members.Count(m => m.Type == type)}");
        
        Console.WriteLine($"\n📋 Active Loans: {_loans.Count(l => !l.IsReturned)}");
        Console.WriteLine($"  Overdue: {_loans.Count(l => l.IsOverdue && !l.IsReturned)}");
        
        var genres = _books.GroupBy(b => b.Genre)
                           .OrderByDescending(g => g.Count());
        Console.WriteLine($"\n📂 Books by Genre:");
        foreach (var group in genres)
            Console.WriteLine($"  {group.Key}: {group.Count()} books");
    }
}

// ============ Demo ============
var library = new Library("Bangkok City Library");

// Authors
var author1 = new Author(1, "Andrew Hunt", "American");
var author2 = new Author(2, "Robert C. Martin", "American");
var author3 = new Author(3, "Martin Fowler", "British");
var author4 = new Author(4, "สุชาติ ทิพย์รัตน์", "Thai");

// Add books
library.AddBook(new Book("978-0-201-63361", "The Pragmatic Programmer", author1, "Programming", 5, 1500));
library.AddBook(new Book("978-0-13-235088", "Clean Code", author2, "Programming", 3, 1200));
library.AddBook(new Book("978-0-13-468599", "Refactoring", author3, "Programming", 4, 1800));
library.AddBook(new Book("978-0-13-110362", "The C Programming Language", author1, "Programming", 2, 900));
library.AddBook(new Book("978-6-16-534520", "การเขียนโปรแกรม C#", author4, "Programming", 8, 599));

// Add members
library.AddMember(new Member("Alice Smith", "alice@email.com", MemberType.Premium));
library.AddMember(new Member("Bob Jones", "bob@email.com", MemberType.Regular));
library.AddMember(new Member("Charlie Brown", "charlie@email.com", MemberType.Student));

// Checkout
var (success1, msg1, loan1) = library.CheckoutBook(1, "Clean Code");
Console.WriteLine($"Checkout: {success1} - {msg1}");

var (success2, msg2, loan2) = library.CheckoutBook(2, "Pragmatic");
Console.WriteLine($"Checkout: {success2} - {msg2}");

// Return
if (loan1 != null)
{
    var (ret, retMsg, fine) = library.ReturnBook(loan1.Id);
    Console.WriteLine($"Return: {ret} - {retMsg}");
}

library.PrintReport();

// Show member details
Console.WriteLine("\n👥 Member Details:");
var alice = library.FindMember(1);
if (alice != null)
{
    Console.WriteLine(alice);
    foreach (var loan in alice.GetActiveLoans())
        Console.WriteLine($"  Active: {loan}");
}
```

---

## สรุป Part 08

```
✅ Step 171: Classes, Objects, Fields, Methods
✅ Step 172: Properties ทุกชนิด (auto, full, computed, indexer)
✅ Step 173: Constructors ทุกชนิด (default, parameterized, chaining, copy)
✅ Step 174: Static Members และ Singleton Pattern
✅ Step 175: Encapsulation และ Access Modifiers
✅ Steps 176-185: Builder Pattern และ Fluent Interface
✅ Steps 186-210: Library Management System สมบูรณ์
```

## แบบฝึกหัด Part 08

1. สร้าง `ShoppingCart` class ที่จัดการ items, discount, และ tax
2. Implement Singleton `Logger` class ที่บันทึก log ไปไฟล์
3. สร้าง `Builder` สำหรับ HTTP Request
4. เขียน `Matrix` class ที่มี indexer และ operator overloading
5. สร้าง `BankingSystem` ครบ: accounts, transfers, history

---

**ถัดไป: [Part 09 - Inheritance และ Polymorphism →](part-09-inheritance-polymorphism.md)**
