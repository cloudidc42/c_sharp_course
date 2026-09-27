# Part 15: Exception Handling (Steps 451-480)

## เป้าหมายการเรียนรู้
- Exception hierarchy และ custom exceptions
- try-catch-finally, exception filters
- Stack unwinding และ inner exceptions
- Global exception handling
- Exception best practices และ anti-patterns
- Result pattern เป็นทางเลือก

---

## Step 451: Exception Hierarchy

```csharp
// Exception hierarchy ใน .NET
// object
//   └── Exception
//         ├── SystemException
//         │     ├── NullReferenceException
//         │     ├── IndexOutOfRangeException
//         │     ├── InvalidOperationException
//         │     ├── ArgumentException
//         │     │     ├── ArgumentNullException
//         │     │     └── ArgumentOutOfRangeException
//         │     ├── OverflowException
//         │     ├── DivideByZeroException
//         │     ├── IOException
//         │     │     ├── FileNotFoundException
//         │     │     └── DirectoryNotFoundException
//         │     ├── NotSupportedException
//         │     ├── NotImplementedException
//         │     ├── StackOverflowException      // cannot catch!
//         │     ├── OutOfMemoryException        // cannot catch!
//         │     └── AccessViolationException   // cannot catch!
//         └── ApplicationException (deprecated, avoid)

// Exception properties
try
{
    throw new InvalidOperationException("Operation failed");
}
catch (Exception ex)
{
    Console.WriteLine($"Message:    {ex.Message}");
    Console.WriteLine($"Type:       {ex.GetType().Name}");
    Console.WriteLine($"Source:     {ex.Source}");
    Console.WriteLine($"TargetSite: {ex.TargetSite}");
    Console.WriteLine($"StackTrace: {ex.StackTrace}");
    Console.WriteLine($"HResult:    {ex.HResult}");
    Console.WriteLine($"InnerEx:    {ex.InnerException?.Message ?? "none"}");
}

// Common exceptions
try { int x = int.Parse("abc"); } catch (FormatException e) { Console.WriteLine(e.Message); }
try { int x = int.Parse("99999999999"); } catch (OverflowException e) { Console.WriteLine(e.Message); }

string? s = null;
try { int l = s!.Length; } catch (NullReferenceException e) { Console.WriteLine(e.Message); }

int[] arr = new int[3];
try { int x = arr[5]; } catch (IndexOutOfRangeException e) { Console.WriteLine(e.Message); }

try { int x = 10 / 0; } catch (DivideByZeroException e) { Console.WriteLine(e.Message); }

// checked — throw OverflowException on overflow
try
{
    checked
    {
        int max = int.MaxValue;
        int overflow = max + 1; // throws OverflowException
    }
}
catch (OverflowException e) { Console.WriteLine(e.Message); }
```

---

## Step 452: try-catch-finally

```csharp
// Basic try-catch
static int Divide(int a, int b)
{
    try
    {
        return a / b;
    }
    catch (DivideByZeroException ex)
    {
        Console.WriteLine($"Cannot divide by zero: {ex.Message}");
        return 0;
    }
}

// Multiple catch blocks (order matters: specific → general)
static void ProcessInput(string input)
{
    try
    {
        int value = int.Parse(input);
        int result = 100 / value;
        Console.WriteLine($"Result: {result}");
    }
    catch (FormatException ex)
    {
        Console.WriteLine($"Invalid format: {ex.Message}");
    }
    catch (OverflowException ex)
    {
        Console.WriteLine($"Number too large: {ex.Message}");
    }
    catch (DivideByZeroException)
    {
        Console.WriteLine("Cannot divide by zero");
    }
    catch (Exception ex) // catch-all (general)
    {
        Console.WriteLine($"Unexpected error: {ex.Message}");
        throw; // re-throw to preserve stack trace
    }
    finally
    {
        Console.WriteLine("Finally block always runs");
        // Use for cleanup: close files, release resources, etc.
    }
}

// Finally guarantees
static string? TryGetData()
{
    StreamReader? reader = null;
    try
    {
        reader = new StreamReader("data.txt");
        return reader.ReadToEnd();
    }
    catch (FileNotFoundException)
    {
        return null;
    }
    finally
    {
        reader?.Close(); // Always runs, even on exception
        Console.WriteLine("StreamReader closed");
    }
}

// Better: using statement (calls Dispose automatically)
static string? TryGetDataBetter()
{
    try
    {
        using var reader = new StreamReader("data.txt");
        return reader.ReadToEnd();
    }
    catch (FileNotFoundException)
    {
        return null;
    }
}

// Nested try-catch
static void NestedExample()
{
    try
    {
        Console.WriteLine("Outer try");
        try
        {
            Console.WriteLine("Inner try");
            throw new InvalidOperationException("Inner exception");
        }
        catch (InvalidOperationException ex)
        {
            Console.WriteLine($"Inner catch: {ex.Message}");
            throw new Exception("Outer exception from inner catch", ex); // wrap
        }
    }
    catch (Exception ex)
    {
        Console.WriteLine($"Outer catch: {ex.Message}");
        Console.WriteLine($"Inner exception: {ex.InnerException?.Message}");
    }
}
```

---

## Step 453: Exception Filters (C# 6+)

```csharp
// Exception filters — catch based on condition
// Advantage: stack trace preserved (unlike catch + re-throw)

static void HandleHttpError(int statusCode)
{
    try
    {
        SimulateHttpRequest(statusCode);
    }
    catch (HttpRequestException ex) when (ex.StatusCode == System.Net.HttpStatusCode.NotFound)
    {
        Console.WriteLine("404: Resource not found");
    }
    catch (HttpRequestException ex) when (ex.StatusCode == System.Net.HttpStatusCode.Unauthorized)
    {
        Console.WriteLine("401: Unauthorized — please login");
    }
    catch (HttpRequestException ex) when ((int)ex.StatusCode! >= 500)
    {
        Console.WriteLine($"5xx Server error: {ex.Message}");
    }
    catch (HttpRequestException ex)
    {
        Console.WriteLine($"HTTP error: {ex.Message}");
    }
}

// Using filter for logging (without catching)
static void WithLogging()
{
    try
    {
        RiskyOperation();
    }
    catch (Exception ex) when (LogAndReturnFalse(ex))
    {
        // This block never runs (filter returns false)
        // But LogAndReturnFalse still executes!
    }
    catch (Exception ex)
    {
        Console.WriteLine($"Handling: {ex.Message}");
    }
}

static bool LogAndReturnFalse(Exception ex)
{
    Console.WriteLine($"[LOG] Exception: {ex.GetType().Name}: {ex.Message}");
    return false; // Don't catch, just log
}

// Conditional retry
static async Task<T> RetryWithFilter<T>(Func<Task<T>> operation, int maxRetries = 3)
{
    int attempt = 0;
    while (true)
    {
        try
        {
            return await operation();
        }
        catch (Exception ex) when (++attempt < maxRetries && IsTransient(ex))
        {
            Console.WriteLine($"Transient error (attempt {attempt}): {ex.Message}");
            await Task.Delay(TimeSpan.FromSeconds(Math.Pow(2, attempt)));
        }
    }
}

static bool IsTransient(Exception ex) => ex is TimeoutException or IOException;

static void SimulateHttpRequest(int code)
    => throw new HttpRequestException("Error", null, (System.Net.HttpStatusCode)code);

static void RiskyOperation() => throw new InvalidOperationException("Risky!");
```

---

## Step 454: Custom Exceptions

```csharp
// ควรสร้าง custom exception เมื่อ:
// 1. Exception มี domain-specific data
// 2. Caller ต้องการ catch exception เฉพาะ type นี้
// 3. Exception แตกต่างจาก built-in exceptions อย่างมีนัยสำคัญ

// Basic custom exception
class OrderNotFoundException : Exception
{
    public int OrderId { get; }
    
    public OrderNotFoundException(int orderId)
        : base($"Order {orderId} not found")
    {
        OrderId = orderId;
    }
    
    public OrderNotFoundException(int orderId, Exception innerException)
        : base($"Order {orderId} not found", innerException)
    {
        OrderId = orderId;
    }
}

// Domain-rich custom exception
class ValidationException : Exception
{
    public string FieldName { get; }
    public object? Value { get; }
    public string[] ValidationErrors { get; }
    
    public ValidationException(string fieldName, object? value, params string[] errors)
        : base($"Validation failed for '{fieldName}': {string.Join("; ", errors)}")
    {
        FieldName = fieldName;
        Value = value;
        ValidationErrors = errors;
    }
}

// Exception hierarchy for a payment system
class PaymentException : Exception
{
    public string? TransactionId { get; }
    public PaymentException(string message, string? transactionId = null) 
        : base(message) { TransactionId = transactionId; }
    public PaymentException(string message, Exception inner, string? transactionId = null)
        : base(message, inner) { TransactionId = transactionId; }
}

class InsufficientFundsException : PaymentException
{
    public decimal Amount { get; }
    public decimal Balance { get; }
    
    public InsufficientFundsException(decimal amount, decimal balance)
        : base($"Insufficient funds: need {amount:C}, have {balance:C}")
    {
        Amount = amount;
        Balance = balance;
    }
}

class CardDeclinedException : PaymentException
{
    public string DeclineCode { get; }
    public string DeclineReason { get; }
    
    public CardDeclinedException(string declineCode, string declineReason)
        : base($"Card declined: {declineReason} (code: {declineCode})")
    {
        DeclineCode = declineCode;
        DeclineReason = declineReason;
    }
}

// Usage
static void ProcessPayment(decimal amount, decimal balance)
{
    try
    {
        if (amount > balance)
            throw new InsufficientFundsException(amount, balance);
        
        if (amount > 10000)
            throw new CardDeclinedException("LIMIT_EXCEEDED", "Daily limit exceeded");
        
        Console.WriteLine($"Payment of {amount:C} successful!");
    }
    catch (InsufficientFundsException ex)
    {
        Console.WriteLine($"Not enough funds: {ex.Message}");
        Console.WriteLine($"  Need: {ex.Amount:C}, Have: {ex.Balance:C}");
    }
    catch (CardDeclinedException ex)
    {
        Console.WriteLine($"Card declined: {ex.DeclineReason} (Code: {ex.DeclineCode})");
    }
    catch (PaymentException ex)
    {
        Console.WriteLine($"Payment failed: {ex.Message}");
    }
}

ProcessPayment(500, 200);   // InsufficientFunds
ProcessPayment(15000, 20000); // CardDeclined
ProcessPayment(100, 500);   // Success
```

---

## Step 455: Re-throwing และ Exception Wrapping

```csharp
// Re-throw patterns

// BAD: loses original stack trace
static void BadRethrow()
{
    try { RiskyWork(); }
    catch (Exception ex)
    {
        LogError(ex);
        throw ex; // Stack trace reset to here!
    }
}

// GOOD: preserve stack trace
static void GoodRethrow()
{
    try { RiskyWork(); }
    catch (Exception ex)
    {
        LogError(ex);
        throw; // Preserve original stack trace
    }
}

// Exception wrapping — wrap low-level exceptions in domain exceptions
static async Task<Order> GetOrderAsync(int orderId)
{
    try
    {
        // Database operation
        return await _repository.GetByIdAsync(orderId);
    }
    catch (SqlException ex) when (ex.Number == 2) // timeout
    {
        throw new OrderException(orderId, "Database timeout", ex);
    }
    catch (SqlException ex)
    {
        throw new OrderException(orderId, "Database error", ex);
    }
    catch (JsonException ex)
    {
        throw new OrderException(orderId, "Invalid data format", ex);
    }
}

// ExceptionDispatchInfo — capture and rethrow with original context
using System.Runtime.ExceptionServices;

static void CaptureAndRethrow()
{
    ExceptionDispatchInfo? captured = null;
    
    try
    {
        RiskyWork();
    }
    catch (Exception ex)
    {
        captured = ExceptionDispatchInfo.Capture(ex);
    }
    
    // Later, re-throw with original stack trace
    if (captured != null)
    {
        Console.WriteLine("Re-throwing later...");
        captured.Throw(); // Preserves original stack trace!
    }
}

// AggregateException handling
static async Task HandleAggregateException()
{
    try
    {
        await Task.WhenAll(
            ThrowAsync("Error 1"),
            ThrowAsync("Error 2"),
            ThrowAsync("Error 3")
        );
    }
    catch (Exception ex) when (ex is AggregateException agg)
    {
        // Flatten nested AggregateExceptions
        var flat = agg.Flatten();
        foreach (var inner in flat.InnerExceptions)
            Console.WriteLine($"  - {inner.Message}");
    }
}

static void RiskyWork() => throw new InvalidOperationException("Risky!");
static void LogError(Exception ex) => Console.WriteLine($"[LOG] {ex.Message}");
static async Task ThrowAsync(string msg)
{
    await Task.Delay(10);
    throw new Exception(msg);
}

class SqlException : Exception
{
    public int Number { get; init; }
    public SqlException(string msg, int number) : base(msg) { Number = number; }
}
record Order(int Id);
static class _repository
{
    public static async Task<Order> GetByIdAsync(int id) => await Task.FromResult(new Order(id));
}
class OrderException : Exception
{
    public int OrderId { get; }
    public OrderException(int orderId, string msg, Exception inner) : base(msg, inner) { OrderId = orderId; }
}
```

---

## Step 456: Global Exception Handling

```csharp
// Global exception handlers สำหรับ unhandled exceptions

// 1. Console app
AppDomain.CurrentDomain.UnhandledException += (sender, e) =>
{
    var ex = (Exception)e.ExceptionObject;
    Console.WriteLine($"[FATAL] Unhandled exception: {ex.Message}");
    Console.WriteLine(ex.StackTrace);
    // Log to file, send to monitoring, etc.
    // Note: App will terminate after this
};

// 2. Task exceptions
TaskScheduler.UnobservedTaskException += (sender, e) =>
{
    Console.WriteLine($"[WARN] Unobserved task exception: {e.Exception.Message}");
    e.SetObserved(); // Prevent application crash
};

// 3. ASP.NET Core — in Program.cs
// app.UseExceptionHandler(errorApp =>
// {
//     errorApp.Run(async context =>
//     {
//         context.Response.StatusCode = 500;
//         var error = context.Features.Get<IExceptionHandlerFeature>();
//         await context.Response.WriteAsJsonAsync(new { error = error?.Error.Message });
//     });
// });

// Exception middleware pattern
class ExceptionHandlerMiddleware
{
    private readonly Func<Exception, (int StatusCode, string Message)> _mapper;
    
    public ExceptionHandlerMiddleware()
    {
        _mapper = MapException;
    }
    
    public async Task<T> Execute<T>(Func<Task<T>> operation)
    {
        try
        {
            return await operation();
        }
        catch (Exception ex)
        {
            var (statusCode, message) = _mapper(ex);
            Console.WriteLine($"[{statusCode}] {message}");
            throw new HttpException(statusCode, message, ex);
        }
    }
    
    private static (int, string) MapException(Exception ex) => ex switch
    {
        OrderNotFoundException e    => (404, $"Order {e.OrderId} not found"),
        ValidationException e       => (400, $"Validation error: {e.Message}"),
        UnauthorizedAccessException => (401, "Unauthorized"),
        InsufficientFundsException  => (402, "Insufficient funds"),
        _                           => (500, "Internal server error")
    };
}

class HttpException : Exception
{
    public int StatusCode { get; }
    public HttpException(int code, string msg, Exception inner) : base(msg, inner) { StatusCode = code; }
}
```

---

## Step 457: Result Pattern — Exception Alternative

```csharp
// Result Pattern: ทางเลือกแทน exceptions สำหรับ expected failures
// Pros: explicit error handling, better performance, no stack unwinding
// Cons: more verbose, requires caller to check

// Basic Result type
readonly struct Result<T>
{
    private readonly T? _value;
    private readonly string? _error;
    
    private Result(T value) { _value = value; _error = null; IsSuccess = true; }
    private Result(string error) { _value = default; _error = error; IsSuccess = false; }
    
    public bool IsSuccess { get; }
    public bool IsFailure => !IsSuccess;
    public T Value => IsSuccess ? _value! : throw new InvalidOperationException($"Result failed: {_error}");
    public string Error => IsFailure ? _error! : throw new InvalidOperationException("Result succeeded");
    
    public static Result<T> Ok(T value) => new(value);
    public static Result<T> Fail(string error) => new(error);
    
    // Functional transformations
    public Result<TNew> Map<TNew>(Func<T, TNew> mapper)
        => IsSuccess ? Result<TNew>.Ok(mapper(_value!)) : Result<TNew>.Fail(_error!);
    
    public Result<TNew> Bind<TNew>(Func<T, Result<TNew>> binder)
        => IsSuccess ? binder(_value!) : Result<TNew>.Fail(_error!);
    
    public T GetOrDefault(T defaultValue) => IsSuccess ? _value! : defaultValue;
    
    public void Match(Action<T> onSuccess, Action<string> onFailure)
    {
        if (IsSuccess) onSuccess(_value!);
        else onFailure(_error!);
    }
    
    public TResult Match<TResult>(Func<T, TResult> onSuccess, Func<string, TResult> onFailure)
        => IsSuccess ? onSuccess(_value!) : onFailure(_error!);
    
    public override string ToString() => IsSuccess ? $"Ok({_value})" : $"Fail({_error})";
}

static class Result
{
    public static Result<T> Ok<T>(T value) => Result<T>.Ok(value);
    public static Result<T> Fail<T>(string error) => Result<T>.Fail(error);
    
    public static Result<IReadOnlyList<T>> Sequence<T>(IEnumerable<Result<T>> results)
    {
        var list = new List<T>();
        foreach (var result in results)
        {
            if (result.IsFailure) return Fail<IReadOnlyList<T>>(result.Error);
            list.Add(result.Value);
        }
        return Ok<IReadOnlyList<T>>(list);
    }
}

// Domain example
record User(int Id, string Email, string Name);

static Result<User> CreateUser(string email, string name)
{
    if (string.IsNullOrWhiteSpace(email))
        return Result.Fail<User>("Email is required");
    
    if (!email.Contains('@'))
        return Result.Fail<User>("Invalid email format");
    
    if (string.IsNullOrWhiteSpace(name))
        return Result.Fail<User>("Name is required");
    
    if (name.Length < 2)
        return Result.Fail<User>("Name must be at least 2 characters");
    
    return Result.Ok(new User(1, email, name));
}

static Result<string> SendWelcomeEmail(User user)
{
    // Simulate email sending
    if (user.Email.EndsWith("@blocked.com"))
        return Result.Fail<string>("Email domain is blocked");
    
    return Result.Ok($"Welcome email sent to {user.Email}");
}

// Railway-oriented programming (chain of operations)
var result = CreateUser("alice@example.com", "Alice")
    .Bind(user => SendWelcomeEmail(user).Map(msg => (User: user, Message: msg)));

result.Match(
    onSuccess: x => Console.WriteLine($"Success! {x.Message}"),
    onFailure: err => Console.WriteLine($"Error: {err}")
);

// Multiple operations
var results = new[] { "alice@test.com", "invalid-email", "bob@test.com" }
    .Select(email => CreateUser(email, "Test User"));

var allSuccessful = Result.Sequence(results);
allSuccessful.Match(
    onSuccess: users => Console.WriteLine($"Created {users.Count} users"),
    onFailure: err => Console.WriteLine($"Batch failed: {err}")
);
```

---

## Step 458: Exception Best Practices

```csharp
// BEST PRACTICES:

// 1. Use specific exception types
// Bad:
throw new Exception("User not found");

// Good:
throw new UserNotFoundException(userId);

// 2. Include meaningful messages
// Bad:
throw new ArgumentException("Invalid argument");

// Good:
throw new ArgumentException($"UserId must be positive, but was {userId}", nameof(userId));

// 3. Don't catch what you can't handle
// Bad: (swallows exception)
try { DoRiskyThing(); }
catch (Exception ex) { /* swallow */ }

// Good: (let it propagate or log + re-throw)
try { DoRiskyThing(); }
catch (Exception ex)
{
    logger.LogError(ex, "Failed to do risky thing");
    throw;
}

// 4. Validate before throwing, use Guard methods
static class Guard
{
    public static T NotNull<T>(T? value, string paramName) where T : class
    {
        if (value == null) throw new ArgumentNullException(paramName);
        return value;
    }
    
    public static string NotNullOrEmpty(string? value, string paramName)
    {
        if (string.IsNullOrEmpty(value)) throw new ArgumentException("Cannot be null or empty", paramName);
        return value;
    }
    
    public static int Positive(int value, string paramName)
    {
        if (value <= 0) throw new ArgumentOutOfRangeException(paramName, value, "Must be positive");
        return value;
    }
    
    public static T NotNull<T>(T? value, string paramName) where T : struct
    {
        if (!value.HasValue) throw new ArgumentNullException(paramName);
        return value.Value;
    }
}

// .NET 6+ built-in ArgumentNullException.ThrowIfNull
static void ModernValidation(string? name, int? age)
{
    ArgumentNullException.ThrowIfNull(name);
    ArgumentException.ThrowIfNullOrEmpty(name);
    ArgumentException.ThrowIfNullOrWhiteSpace(name);
    ArgumentOutOfRangeException.ThrowIfNegative(age ?? 0);
    ArgumentOutOfRangeException.ThrowIfGreaterThan(age ?? 0, 150);
}

// 5. Don't use exceptions for control flow
// Bad: using exception for flow control (slow, wrong semantic)
static bool IsValidEmail(string email)
{
    try
    {
        var addr = new System.Net.Mail.MailAddress(email);
        return addr.Address == email;
    }
    catch (FormatException)
    {
        return false; // exception as flow control
    }
}

// Good: validate without exceptions
static bool IsValidEmailBetter(string email)
    => !string.IsNullOrEmpty(email) && email.Contains('@') && email.Contains('.');

// 6. Clean up resources in finally or use 'using'
// Bad:
FileStream? fs = null;
try
{
    fs = File.OpenRead("file.txt");
    // use fs
}
catch { throw; }
finally { fs?.Dispose(); }

// Good:
try
{
    using var fs2 = File.OpenRead("file.txt");
    // use fs2
}
catch (FileNotFoundException ex)
{
    Console.WriteLine($"File not found: {ex.FileName}");
}

// 7. Don't throw from static constructors
// Bad:
class BadClass
{
    static BadClass()
    {
        // throw new Exception("Oops"); // Becomes TypeInitializationException
        // All future access to BadClass will throw!
    }
}

// 8. Document exceptions with XML docs
/// <summary>Gets a user by ID.</summary>
/// <param name="userId">The user ID. Must be positive.</param>
/// <returns>The user.</returns>
/// <exception cref="ArgumentOutOfRangeException">When userId is not positive.</exception>
/// <exception cref="UserNotFoundException">When the user does not exist.</exception>
static User GetUser(int userId)
{
    if (userId <= 0) throw new ArgumentOutOfRangeException(nameof(userId), "Must be positive");
    // ...
    return new User(userId, "test@test.com", "Test");
}

class UserNotFoundException : Exception
{
    public int UserId { get; }
    public UserNotFoundException(int userId) : base($"User {userId} not found") { UserId = userId; }
}
static void DoRiskyThing() { }
static ILogger logger = new ConsoleLogger();
interface ILogger { void LogError(Exception ex, string msg); }
class ConsoleLogger : ILogger
{
    public void LogError(Exception ex, string msg) => Console.WriteLine($"{msg}: {ex.Message}");
}
```

---

## Step 459: โปรแกรมตัวอย่าง — Robust File Processor

```csharp
using System;
using System.Collections.Generic;
using System.IO;
using System.Linq;
using System.Threading.Tasks;

// ============================================================
// Robust File Processor with comprehensive error handling
// ============================================================

// Custom exceptions
class FileProcessingException : Exception
{
    public string FilePath { get; }
    public int LineNumber { get; }
    
    public FileProcessingException(string filePath, int lineNumber, string message, Exception? inner = null)
        : base($"Error processing '{filePath}' at line {lineNumber}: {message}", inner)
    {
        FilePath = filePath;
        LineNumber = lineNumber;
    }
}

class CsvParseException : FileProcessingException
{
    public string InvalidLine { get; }
    
    public CsvParseException(string filePath, int lineNumber, string invalidLine, string reason)
        : base(filePath, lineNumber, reason)
    {
        InvalidLine = invalidLine;
    }
}

// Result types
record ProcessingResult<T>
{
    public T? Value { get; init; }
    public string? Error { get; init; }
    public bool IsSuccess => Error == null;
    
    public static ProcessingResult<T> Ok(T value) => new() { Value = value };
    public static ProcessingResult<T> Fail(string error) => new() { Error = error };
}

record ParsedRecord(string Name, string Email, int Age, decimal Salary);

class CsvProcessor
{
    private int _processedCount;
    private int _errorCount;
    private readonly List<(int Line, string Error)> _errors = new();
    
    public async Task<ProcessingResult<IReadOnlyList<ParsedRecord>>> ProcessFileAsync(
        string filePath)
    {
        // Validate inputs
        ArgumentException.ThrowIfNullOrEmpty(filePath);
        
        if (!File.Exists(filePath))
            return ProcessingResult<IReadOnlyList<ParsedRecord>>.Fail(
                $"File not found: {filePath}");
        
        if (new FileInfo(filePath).Length > 100 * 1024 * 1024) // 100MB limit
            return ProcessingResult<IReadOnlyList<ParsedRecord>>.Fail(
                "File is too large (max 100MB)");
        
        var records = new List<ParsedRecord>();
        
        try
        {
            await using var stream = new FileStream(
                filePath,
                FileMode.Open,
                FileAccess.Read,
                FileShare.Read,
                bufferSize: 4096,
                useAsync: true);
            
            using var reader = new StreamReader(stream);
            
            // Skip header
            string? header = await reader.ReadLineAsync();
            if (header == null)
                return ProcessingResult<IReadOnlyList<ParsedRecord>>.Fail("Empty file");
            
            int lineNumber = 1;
            string? line;
            
            while ((line = await reader.ReadLineAsync()) != null)
            {
                lineNumber++;
                
                if (string.IsNullOrWhiteSpace(line)) continue; // skip blank lines
                
                var result = ParseLine(filePath, lineNumber, line);
                if (result.IsSuccess)
                {
                    records.Add(result.Value!);
                    _processedCount++;
                }
                else
                {
                    _errors.Add((lineNumber, result.Error!));
                    _errorCount++;
                }
            }
        }
        catch (UnauthorizedAccessException ex)
        {
            return ProcessingResult<IReadOnlyList<ParsedRecord>>.Fail(
                $"No permission to read file: {ex.Message}");
        }
        catch (IOException ex)
        {
            return ProcessingResult<IReadOnlyList<ParsedRecord>>.Fail(
                $"IO error reading file: {ex.Message}");
        }
        
        return ProcessingResult<IReadOnlyList<ParsedRecord>>.Ok(records.AsReadOnly());
    }
    
    private ProcessingResult<ParsedRecord> ParseLine(string filePath, int lineNumber, string line)
    {
        try
        {
            string[] parts = line.Split(',');
            
            if (parts.Length != 4)
                return ProcessingResult<ParsedRecord>.Fail(
                    $"Expected 4 fields, got {parts.Length}: '{line}'");
            
            string name = parts[0].Trim();
            string email = parts[1].Trim();
            
            if (!email.Contains('@'))
                return ProcessingResult<ParsedRecord>.Fail(
                    $"Invalid email: '{email}'");
            
            if (!int.TryParse(parts[2].Trim(), out int age) || age < 0 || age > 150)
                return ProcessingResult<ParsedRecord>.Fail(
                    $"Invalid age: '{parts[2]}'");
            
            if (!decimal.TryParse(parts[3].Trim(), out decimal salary) || salary < 0)
                return ProcessingResult<ParsedRecord>.Fail(
                    $"Invalid salary: '{parts[3]}'");
            
            return ProcessingResult<ParsedRecord>.Ok(new ParsedRecord(name, email, age, salary));
        }
        catch (Exception ex)
        {
            return ProcessingResult<ParsedRecord>.Fail($"Parse error: {ex.Message}");
        }
    }
    
    public void PrintSummary()
    {
        Console.WriteLine($"\n=== Processing Summary ===");
        Console.WriteLine($"Processed: {_processedCount}");
        Console.WriteLine($"Errors:    {_errorCount}");
        
        if (_errors.Count > 0)
        {
            Console.WriteLine("\nErrors:");
            foreach (var (line, error) in _errors.Take(10))
                Console.WriteLine($"  Line {line}: {error}");
            if (_errors.Count > 10)
                Console.WriteLine($"  ... and {_errors.Count - 10} more");
        }
    }
}

// Demo
const string csvContent = """
    Name,Email,Age,Salary
    Alice Smith,alice@email.com,30,75000
    Bob Jones,bob@email.com,25,55000
    invalid-line
    Carol White,carol-email,35,65000
    Dave Brown,dave@email.com,-5,70000
    Eve Davis,eve@email.com,28,80000
    """;

// Create test file
string testFile = Path.GetTempFileName();
await File.WriteAllTextAsync(testFile, csvContent);

var processor = new CsvProcessor();

try
{
    var result = await processor.ProcessFileAsync(testFile);
    
    if (result.IsSuccess)
    {
        Console.WriteLine($"Successfully loaded {result.Value!.Count} records:");
        foreach (var record in result.Value)
            Console.WriteLine($"  {record.Name,-20} {record.Email,-25} Age:{record.Age} Salary:{record.Salary:C}");
    }
    else
    {
        Console.WriteLine($"Processing failed: {result.Error}");
    }
    
    processor.PrintSummary();
}
finally
{
    File.Delete(testFile); // cleanup
}

// Test with non-existent file
var badResult = await processor.ProcessFileAsync("nonexistent.csv");
Console.WriteLine($"\nNon-existent file result: {badResult.Error}");
```

---

## สรุป Part 15

### Exception Types
- ✅ Exception hierarchy — System, Application exceptions
- ✅ Exception properties: Message, StackTrace, InnerException
- ✅ Custom exceptions with domain data

### try-catch-finally
- ✅ Multiple catch blocks (specific → general order)
- ✅ `finally` always runs (cleanup)
- ✅ `using` statement = try-finally with Dispose

### Exception Filters (C# 6+)
- ✅ `catch (Ex e) when (condition)` — conditional catch
- ✅ Stack trace preserved (unlike re-throw)
- ✅ Log-only pattern with `when (LogAndReturnFalse(ex))`

### Re-throwing
- ✅ `throw` — preserve stack trace
- ✅ `throw ex` — resets stack trace (avoid!)
- ✅ `ExceptionDispatchInfo` — capture and defer re-throw

### Custom Exceptions
- ✅ Inherit from `Exception` or specific exception
- ✅ Include domain-specific data as properties
- ✅ Provide constructors with inner exception

### Global Handling
- ✅ `AppDomain.CurrentDomain.UnhandledException`
- ✅ `TaskScheduler.UnobservedTaskException`

### Result Pattern
- ✅ `Result<T>` instead of exceptions for expected failures
- ✅ `Map`, `Bind`, `Match` for functional composition
- ✅ Railway-oriented programming

### Best Practices
- ✅ Use specific exception types
- ✅ Don't swallow exceptions
- ✅ Use `throw` not `throw ex`
- ✅ Validate with Guard methods
- ✅ Don't use exceptions for control flow

---

## แบบฝึกหัด

1. สร้าง `DomainException` hierarchy สำหรับ e-commerce system (OrderException, ProductException, UserException)
2. เขียน `PollyLike` retry policy ที่รองรับ circuit breaker pattern
3. Implement `ValidationResult` ที่สะสม multiple validation errors
4. สร้าง global exception handler middleware สำหรับ console app ที่ log exceptions ไปยัง file
5. เปลี่ยน existing method ที่ใช้ exceptions เป็น Result pattern แล้ววัด performance
