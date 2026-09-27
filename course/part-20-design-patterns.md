# Part 20: Design Patterns (รูปแบบการออกแบบ)

## ขั้นตอนที่ 601: Design Patterns คืออะไร?

Design Patterns คือ solution ที่ผ่านการพิสูจน์แล้วสำหรับปัญหาที่พบบ่อยในการออกแบบซอฟต์แวร์ แบ่งเป็น 3 หมวด:

```
┌─────────────────────────────────────────────────────────┐
│                   Design Patterns                       │
├─────────────────┬──────────────────┬────────────────────┤
│  Creational     │   Structural     │   Behavioral       │
│  (สร้าง Object) │  (จัดโครงสร้าง)  │  (พฤติกรรม Object) │
├─────────────────┼──────────────────┼────────────────────┤
│ • Singleton     │ • Adapter        │ • Observer         │
│ • Factory       │ • Bridge         │ • Strategy         │
│ • Abstract Fac  │ • Composite      │ • Command          │
│ • Builder       │ • Decorator      │ • Iterator         │
│ • Prototype     │ • Facade         │ • Chain of Resp.   │
│                 │ • Flyweight      │ • Template Method  │
│                 │ • Proxy          │ • State            │
│                 │                  │ • Mediator         │
│                 │                  │ • Visitor          │
│                 │                  │ • Memento          │
└─────────────────┴──────────────────┴────────────────────┘
```

## ขั้นตอนที่ 602: Singleton Pattern

```csharp
// Singleton: รับประกันว่ามี instance เดียวในโปรแกรม

// แบบที่ 1: Thread-safe Lazy<T>
public class DatabaseConnection
{
    private static readonly Lazy<DatabaseConnection> _instance =
        new(() => new DatabaseConnection());
    
    private DatabaseConnection()
    {
        Console.WriteLine("DatabaseConnection created");
        // Initialize connection...
    }
    
    public static DatabaseConnection Instance => _instance.Value;
    
    private bool _isConnected = false;
    
    public void Connect(string connectionString)
    {
        if (_isConnected) return;
        Console.WriteLine($"Connected to: {connectionString}");
        _isConnected = true;
    }
    
    public void Execute(string sql)
    {
        if (!_isConnected) throw new InvalidOperationException("Not connected");
        Console.WriteLine($"Executing: {sql}");
    }
}

// การใช้งาน
var db1 = DatabaseConnection.Instance;
var db2 = DatabaseConnection.Instance;
Console.WriteLine(ReferenceEquals(db1, db2)); // True - same instance

db1.Connect("Server=localhost;Database=MyDB");
db2.Execute("SELECT * FROM Users"); // ใช้ connection เดียวกัน

// แบบที่ 2: Monostate (Singleton ด้วย static state)
public class AppSettings
{
    private static string _theme = "Light";
    private static string _language = "en";
    private static readonly Dictionary<string, string> _settings = [];
    
    // ทุก instance ใช้ state เดียวกัน
    public string Theme 
    { 
        get => _theme; 
        set => _theme = value; 
    }
    
    public string Language 
    { 
        get => _language; 
        set => _language = value; 
    }
    
    public string this[string key]
    {
        get => _settings.GetValueOrDefault(key, "");
        set => _settings[key] = value;
    }
}

// แบบที่ 3: DI-friendly Singleton (ดีกว่าสำหรับ testability)
// builder.Services.AddSingleton<IDatabaseConnection, DatabaseConnection>();
```

## ขั้นตอนที่ 603: Factory Method Pattern

```csharp
// Factory Method: ให้ subclass กำหนดว่าจะสร้าง object อะไร

// Products
public interface INotification
{
    Task SendAsync(string recipient, string subject, string body);
    string Type { get; }
}

public class EmailNotification : INotification
{
    private readonly string _smtpServer;
    
    public EmailNotification(string smtpServer)
    {
        _smtpServer = smtpServer;
    }
    
    public string Type => "Email";
    
    public async Task SendAsync(string recipient, string subject, string body)
    {
        Console.WriteLine($"[Email] To: {recipient}, Subject: {subject}");
        Console.WriteLine($"SMTP Server: {_smtpServer}");
        await Task.Delay(100); // Simulate sending
    }
}

public class SmsNotification : INotification
{
    private readonly string _apiKey;
    
    public SmsNotification(string apiKey)
    {
        _apiKey = apiKey;
    }
    
    public string Type => "SMS";
    
    public async Task SendAsync(string recipient, string subject, string body)
    {
        Console.WriteLine($"[SMS] To: {recipient}, Message: {body[..Math.Min(160, body.Length)]}");
        await Task.Delay(50);
    }
}

public class PushNotification : INotification
{
    public string Type => "Push";
    
    public async Task SendAsync(string recipient, string subject, string body)
    {
        Console.WriteLine($"[Push] Device: {recipient}, Title: {subject}");
        await Task.Delay(30);
    }
}

// Creator (Abstract Factory Method)
public abstract class NotificationSender
{
    // Factory Method - ให้ subclass implement
    protected abstract INotification CreateNotification();
    
    // Template method ที่ใช้ factory method
    public async Task NotifyAsync(string recipient, string message)
    {
        var notification = CreateNotification();
        
        Console.WriteLine($"Sending {notification.Type} notification...");
        await notification.SendAsync(recipient, "Notification", message);
        
        await LogAsync(notification.Type, recipient);
    }
    
    protected virtual Task LogAsync(string type, string recipient)
    {
        Console.WriteLine($"Logged: {type} sent to {recipient}");
        return Task.CompletedTask;
    }
}

public class EmailSender : NotificationSender
{
    private readonly string _smtpServer;
    
    public EmailSender(string smtpServer)
    {
        _smtpServer = smtpServer;
    }
    
    protected override INotification CreateNotification()
        => new EmailNotification(_smtpServer);
}

public class SmsSender : NotificationSender
{
    private readonly string _apiKey;
    
    public SmsSender(string apiKey)
    {
        _apiKey = apiKey;
    }
    
    protected override INotification CreateNotification()
        => new SmsNotification(_apiKey);
}

// Simple Factory (ไม่ใช่ pattern จริงๆ แต่พบบ่อยมาก)
public static class NotificationFactory
{
    public static INotification Create(string type, IConfiguration config)
    {
        return type.ToLower() switch
        {
            "email" => new EmailNotification(config["Smtp:Server"] ?? "localhost"),
            "sms" => new SmsNotification(config["Sms:ApiKey"] ?? ""),
            "push" => new PushNotification(),
            _ => throw new NotSupportedException($"Notification type '{type}' is not supported")
        };
    }
}

// การใช้งาน
var emailSender = new EmailSender("smtp.gmail.com");
await emailSender.NotifyAsync("user@example.com", "Your order is ready!");

var notification = NotificationFactory.Create("sms", configuration);
await notification.SendAsync("+66812345678", "OTP", "Your OTP is: 123456");
```

## ขั้นตอนที่ 604: Abstract Factory Pattern

```csharp
// Abstract Factory: สร้าง family ของ objects ที่เกี่ยวข้องกัน

// Abstract Products
public interface IButton
{
    void Render();
    void OnClick(Action handler);
}

public interface ITextBox
{
    void Render();
    string GetValue();
}

public interface IDropdown
{
    void Render();
    void AddItem(string item);
    string? SelectedItem { get; }
}

// Windows Products
public class WindowsButton : IButton
{
    private Action? _clickHandler;
    
    public void Render() => Console.WriteLine("[Windows] <button style='win11'>");
    public void OnClick(Action handler) => _clickHandler = handler;
    public void Click() => _clickHandler?.Invoke();
}

public class WindowsTextBox : ITextBox
{
    private string _value = "";
    
    public void Render() => Console.WriteLine("[Windows] <input type='text' class='win11-textbox'>");
    public string GetValue() => _value;
    public void SetValue(string v) => _value = v;
}

public class WindowsDropdown : IDropdown
{
    private readonly List<string> _items = [];
    
    public void Render() => Console.WriteLine($"[Windows] <select> ({_items.Count} items)");
    public void AddItem(string item) => _items.Add(item);
    public string? SelectedItem => _items.FirstOrDefault();
}

// macOS Products
public class MacButton : IButton
{
    private Action? _clickHandler;
    
    public void Render() => Console.WriteLine("[macOS] <button class='mac-button rounded'>");
    public void OnClick(Action handler) => _clickHandler = handler;
}

public class MacTextBox : ITextBox
{
    private string _value = "";
    
    public void Render() => Console.WriteLine("[macOS] <input type='text' class='mac-input'>");
    public string GetValue() => _value;
}

public class MacDropdown : IDropdown
{
    private readonly List<string> _items = [];
    
    public void Render() => Console.WriteLine($"[macOS] <select class='mac-select'> ({_items.Count} items)");
    public void AddItem(string item) => _items.Add(item);
    public string? SelectedItem => _items.FirstOrDefault();
}

// Abstract Factory
public interface IUIFactory
{
    IButton CreateButton();
    ITextBox CreateTextBox();
    IDropdown CreateDropdown();
}

// Concrete Factories
public class WindowsUIFactory : IUIFactory
{
    public IButton CreateButton() => new WindowsButton();
    public ITextBox CreateTextBox() => new WindowsTextBox();
    public IDropdown CreateDropdown() => new WindowsDropdown();
}

public class MacUIFactory : IUIFactory
{
    public IButton CreateButton() => new MacButton();
    public ITextBox CreateTextBox() => new MacTextBox();
    public IDropdown CreateDropdown() => new MacDropdown();
}

// Client ไม่รู้ว่าใช้ platform ไหน
public class LoginForm
{
    private readonly IButton _loginButton;
    private readonly ITextBox _usernameBox;
    private readonly ITextBox _passwordBox;
    
    public LoginForm(IUIFactory factory)
    {
        _loginButton = factory.CreateButton();
        _usernameBox = factory.CreateTextBox();
        _passwordBox = factory.CreateTextBox();
        
        _loginButton.OnClick(() => 
        {
            Console.WriteLine($"Logging in as: {_usernameBox.GetValue()}");
        });
    }
    
    public void Render()
    {
        Console.WriteLine("=== Login Form ===");
        _usernameBox.Render();
        _passwordBox.Render();
        _loginButton.Render();
    }
}

// การใช้งาน
IUIFactory factory = Environment.OSVersion.Platform == PlatformID.Win32NT
    ? new WindowsUIFactory()
    : new MacUIFactory();

var form = new LoginForm(factory);
form.Render();
```

## ขั้นตอนที่ 605: Builder Pattern

```csharp
// Builder: สร้าง object ที่ซับซ้อนทีละขั้นตอน

public class QueryBuilder
{
    private string? _table;
    private readonly List<string> _columns = [];
    private readonly List<string> _conditions = [];
    private readonly List<string> _joins = [];
    private readonly List<string> _orderBy = [];
    private int? _limit;
    private int? _offset;
    private bool _distinct = false;
    private readonly Dictionary<string, object> _parameters = [];
    
    public QueryBuilder From(string table)
    {
        _table = table;
        return this;
    }
    
    public QueryBuilder Select(params string[] columns)
    {
        _columns.AddRange(columns);
        return this;
    }
    
    public QueryBuilder Distinct()
    {
        _distinct = true;
        return this;
    }
    
    public QueryBuilder Where(string condition, object? parameter = null, string? paramName = null)
    {
        _conditions.Add(condition);
        if (parameter is not null && paramName is not null)
            _parameters[paramName] = parameter;
        return this;
    }
    
    public QueryBuilder Join(string table, string on)
    {
        _joins.Add($"JOIN {table} ON {on}");
        return this;
    }
    
    public QueryBuilder LeftJoin(string table, string on)
    {
        _joins.Add($"LEFT JOIN {table} ON {on}");
        return this;
    }
    
    public QueryBuilder OrderBy(string column, bool ascending = true)
    {
        _orderBy.Add($"{column} {(ascending ? "ASC" : "DESC")}");
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
    
    public (string Sql, Dictionary<string, object> Parameters) Build()
    {
        if (string.IsNullOrEmpty(_table))
            throw new InvalidOperationException("Table must be specified");
        
        var sb = new System.Text.StringBuilder();
        
        sb.Append("SELECT ");
        if (_distinct) sb.Append("DISTINCT ");
        sb.Append(_columns.Count > 0 ? string.Join(", ", _columns) : "*");
        sb.Append($" FROM {_table}");
        
        foreach (var join in _joins)
            sb.Append($" {join}");
        
        if (_conditions.Count > 0)
            sb.Append($" WHERE {string.Join(" AND ", _conditions)}");
        
        if (_orderBy.Count > 0)
            sb.Append($" ORDER BY {string.Join(", ", _orderBy)}");
        
        if (_limit.HasValue)
            sb.Append($" LIMIT {_limit}");
        
        if (_offset.HasValue)
            sb.Append($" OFFSET {_offset}");
        
        return (sb.ToString(), _parameters);
    }
    
    // Reset for reuse
    public QueryBuilder Reset()
    {
        _table = null;
        _columns.Clear();
        _conditions.Clear();
        _joins.Clear();
        _orderBy.Clear();
        _limit = null;
        _offset = null;
        _distinct = false;
        _parameters.Clear();
        return this;
    }
}

// Fluent Builder สำหรับ Domain Object
public class EmailBuilder
{
    private string? _from;
    private readonly List<string> _to = [];
    private readonly List<string> _cc = [];
    private string? _subject;
    private string? _body;
    private bool _isHtml = false;
    private readonly List<(string Name, Stream Content, string MimeType)> _attachments = [];
    
    public EmailBuilder From(string email)
    {
        _from = email;
        return this;
    }
    
    public EmailBuilder To(params string[] emails)
    {
        _to.AddRange(emails);
        return this;
    }
    
    public EmailBuilder Cc(params string[] emails)
    {
        _cc.AddRange(emails);
        return this;
    }
    
    public EmailBuilder Subject(string subject)
    {
        _subject = subject;
        return this;
    }
    
    public EmailBuilder Body(string body, bool isHtml = false)
    {
        _body = body;
        _isHtml = isHtml;
        return this;
    }
    
    public EmailBuilder HtmlBody(string htmlBody)
        => Body(htmlBody, true);
    
    public EmailBuilder Attach(string fileName, Stream content, string mimeType = "application/octet-stream")
    {
        _attachments.Add((fileName, content, mimeType));
        return this;
    }
    
    public Email Build()
    {
        if (string.IsNullOrEmpty(_from)) throw new InvalidOperationException("From is required");
        if (_to.Count == 0) throw new InvalidOperationException("At least one recipient required");
        if (string.IsNullOrEmpty(_subject)) throw new InvalidOperationException("Subject is required");
        
        return new Email(_from, [.. _to], [.. _cc], _subject, _body ?? "", _isHtml, [.. _attachments]);
    }
}

public record Email(
    string From,
    string[] To,
    string[] Cc,
    string Subject,
    string Body,
    bool IsHtml,
    (string Name, Stream Content, string MimeType)[] Attachments);

// การใช้งาน Builder
var (sql, parameters) = new QueryBuilder()
    .From("Users u")
    .Select("u.Id", "u.Name", "u.Email", "r.Name as Role")
    .Join("UserRoles ur", "ur.UserId = u.Id")
    .Join("Roles r", "r.Id = ur.RoleId")
    .Where("u.IsActive = 1")
    .Where("u.CreatedAt > @date", DateTime.Now.AddDays(-30), "date")
    .OrderBy("u.Name")
    .Limit(20)
    .Offset(0)
    .Build();

Console.WriteLine(sql);

var email = new EmailBuilder()
    .From("noreply@myapp.com")
    .To("user@example.com", "user2@example.com")
    .Cc("manager@example.com")
    .Subject("Monthly Report")
    .HtmlBody("<h1>Report</h1><p>Please find the report attached.</p>")
    .Attach("report.pdf", File.OpenRead("report.pdf"), "application/pdf")
    .Build();
```

## ขั้นตอนที่ 606: Adapter Pattern

```csharp
// Adapter: แปลง interface หนึ่งให้เป็นอีก interface ที่ client ต้องการ

// Target Interface (ที่ client ต้องการใช้)
public interface IPaymentProcessor
{
    Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request);
    Task<RefundResult> RefundAsync(string transactionId, decimal amount);
}

public record PaymentRequest(string CardNumber, decimal Amount, string Currency, string Description);
public record PaymentResult(bool Success, string TransactionId, string? ErrorMessage = null);
public record RefundResult(bool Success, string RefundId, string? ErrorMessage = null);

// Adaptee: ระบบ Stripe ที่มี API แตกต่างออกไป
public class StripePaymentGateway
{
    public async Task<StripeCharge> ChargeAsync(
        string cardToken, 
        long amountInCents, 
        string currency,
        Dictionary<string, string>? metadata = null)
    {
        Console.WriteLine($"[Stripe] Charging {amountInCents} cents in {currency}");
        await Task.Delay(100);
        return new StripeCharge(
            "ch_" + Guid.NewGuid().ToString("N")[..10],
            true, "succeeded");
    }
    
    public async Task<StripeRefund> RefundChargeAsync(string chargeId, long? amountInCents = null)
    {
        Console.WriteLine($"[Stripe] Refunding charge {chargeId}");
        await Task.Delay(80);
        return new StripeRefund("re_" + Guid.NewGuid().ToString("N")[..10], true);
    }
}

public record StripeCharge(string Id, bool Paid, string Status);
public record StripeRefund(string Id, bool Success);

// Adapter: ทำให้ Stripe ทำงานกับ IPaymentProcessor
public class StripePaymentAdapter : IPaymentProcessor
{
    private readonly StripePaymentGateway _stripe;
    
    public StripePaymentAdapter(StripePaymentGateway stripe)
    {
        _stripe = stripe;
    }
    
    public async Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request)
    {
        try
        {
            // แปลง decimal amount เป็น cents
            var amountInCents = (long)(request.Amount * 100);
            
            var charge = await _stripe.ChargeAsync(
                cardToken: request.CardNumber,
                amountInCents: amountInCents,
                currency: request.Currency.ToLower(),
                metadata: new() { ["description"] = request.Description });
            
            return charge.Paid
                ? new PaymentResult(true, charge.Id)
                : new PaymentResult(false, "", "Payment declined");
        }
        catch (Exception ex)
        {
            return new PaymentResult(false, "", ex.Message);
        }
    }
    
    public async Task<RefundResult> RefundAsync(string transactionId, decimal amount)
    {
        try
        {
            var amountInCents = (long)(amount * 100);
            var refund = await _stripe.RefundChargeAsync(transactionId, amountInCents);
            
            return refund.Success
                ? new RefundResult(true, refund.Id)
                : new RefundResult(false, "", "Refund failed");
        }
        catch (Exception ex)
        {
            return new RefundResult(false, "", ex.Message);
        }
    }
}

// Object Adapter สำหรับ PayPal
public class PayPalPaymentAdapter : IPaymentProcessor
{
    private readonly PayPalSdk _paypal;
    
    public PayPalPaymentAdapter(PayPalSdk paypal)
    {
        _paypal = paypal;
    }
    
    public async Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request)
    {
        var result = await _paypal.ExecutePaymentAsync(new PayPalPayment
        {
            Amount = new() { Total = request.Amount.ToString("F2"), Currency = request.Currency },
            Description = request.Description
        });
        
        return result.State == "approved"
            ? new PaymentResult(true, result.Id)
            : new PaymentResult(false, "", result.FailureReason);
    }
    
    public async Task<RefundResult> RefundAsync(string transactionId, decimal amount)
    {
        var result = await _paypal.CreateRefundAsync(transactionId, amount);
        return new RefundResult(result.Success, result.RefundId ?? "", result.Error);
    }
}

// Client ใช้ Adapter ผ่าน interface เดียว
public class OrderService
{
    private readonly IPaymentProcessor _payment;
    
    public OrderService(IPaymentProcessor payment)
    {
        _payment = payment;
    }
    
    public async Task<bool> ProcessOrderAsync(Order order)
    {
        var result = await _payment.ProcessPaymentAsync(new PaymentRequest(
            order.CardNumber,
            order.TotalAmount,
            "THB",
            $"Order #{order.Id}"));
        
        if (!result.Success)
        {
            Console.WriteLine($"Payment failed: {result.ErrorMessage}");
            return false;
        }
        
        order.PaymentTransactionId = result.TransactionId;
        Console.WriteLine($"Payment succeeded: {result.TransactionId}");
        return true;
    }
}
```

## ขั้นตอนที่ 607: Decorator Pattern

```csharp
// Decorator: เพิ่ม behavior ให้ object โดยไม่แก้ไข class เดิม

// Component Interface
public interface IDataSource
{
    void WriteData(byte[] data);
    byte[] ReadData();
}

// Concrete Component
public class FileDataSource : IDataSource
{
    private readonly string _filePath;
    
    public FileDataSource(string filePath)
    {
        _filePath = filePath;
    }
    
    public void WriteData(byte[] data)
    {
        File.WriteAllBytes(_filePath, data);
        Console.WriteLine($"Written {data.Length} bytes to {_filePath}");
    }
    
    public byte[] ReadData()
    {
        var data = File.ReadAllBytes(_filePath);
        Console.WriteLine($"Read {data.Length} bytes from {_filePath}");
        return data;
    }
}

// Base Decorator
public abstract class DataSourceDecorator : IDataSource
{
    protected readonly IDataSource _inner;
    
    protected DataSourceDecorator(IDataSource inner)
    {
        _inner = inner;
    }
    
    public virtual void WriteData(byte[] data) => _inner.WriteData(data);
    public virtual byte[] ReadData() => _inner.ReadData();
}

// Concrete Decorators
public class EncryptionDecorator : DataSourceDecorator
{
    private readonly byte[] _key;
    
    public EncryptionDecorator(IDataSource inner, byte[]? key = null) : base(inner)
    {
        _key = key ?? new byte[16] { 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16 };
    }
    
    public override void WriteData(byte[] data)
    {
        var encrypted = Encrypt(data);
        Console.WriteLine($"Encrypted {data.Length} bytes -> {encrypted.Length} bytes");
        base.WriteData(encrypted);
    }
    
    public override byte[] ReadData()
    {
        var encrypted = base.ReadData();
        var decrypted = Decrypt(encrypted);
        Console.WriteLine($"Decrypted {encrypted.Length} bytes -> {decrypted.Length} bytes");
        return decrypted;
    }
    
    private byte[] Encrypt(byte[] data)
    {
        // Simplified XOR encryption (ใน production ใช้ AES)
        return data.Select((b, i) => (byte)(b ^ _key[i % _key.Length])).ToArray();
    }
    
    private byte[] Decrypt(byte[] data) => Encrypt(data); // XOR is symmetric
}

public class CompressionDecorator : DataSourceDecorator
{
    public CompressionDecorator(IDataSource inner) : base(inner) { }
    
    public override void WriteData(byte[] data)
    {
        using var compressed = new System.IO.MemoryStream();
        using (var gzip = new System.IO.Compression.GZipStream(compressed, 
            System.IO.Compression.CompressionLevel.Optimal))
        {
            gzip.Write(data);
        }
        
        var compressedData = compressed.ToArray();
        Console.WriteLine($"Compressed {data.Length} -> {compressedData.Length} bytes " +
            $"({(1 - (double)compressedData.Length / data.Length) * 100:F1}% reduction)");
        base.WriteData(compressedData);
    }
    
    public override byte[] ReadData()
    {
        var compressed = base.ReadData();
        using var input = new System.IO.MemoryStream(compressed);
        using var gzip = new System.IO.Compression.GZipStream(input, 
            System.IO.Compression.CompressionMode.Decompress);
        using var output = new System.IO.MemoryStream();
        gzip.CopyTo(output);
        
        var data = output.ToArray();
        Console.WriteLine($"Decompressed {compressed.Length} -> {data.Length} bytes");
        return data;
    }
}

public class CachingDecorator : DataSourceDecorator
{
    private byte[]? _cache;
    private DateTime _cacheTime;
    private readonly TimeSpan _cacheExpiry;
    
    public CachingDecorator(IDataSource inner, TimeSpan? cacheExpiry = null) : base(inner)
    {
        _cacheExpiry = cacheExpiry ?? TimeSpan.FromMinutes(5);
    }
    
    public override void WriteData(byte[] data)
    {
        _cache = null; // Invalidate cache
        base.WriteData(data);
    }
    
    public override byte[] ReadData()
    {
        if (_cache is not null && DateTime.Now - _cacheTime < _cacheExpiry)
        {
            Console.WriteLine("Cache hit!");
            return _cache;
        }
        
        _cache = base.ReadData();
        _cacheTime = DateTime.Now;
        Console.WriteLine("Cache miss - loaded from source");
        return _cache;
    }
}

// การใช้งาน - Stack decorators
var dataSource = new CachingDecorator(
    new EncryptionDecorator(
        new CompressionDecorator(
            new FileDataSource("data.dat"))));

// Write: Cache invalidate -> Encrypt -> Compress -> File
var text = System.Text.Encoding.UTF8.GetBytes("Hello World! " * 1000);
dataSource.WriteData(text);

// Read: File -> Decompress -> Decrypt -> Cache
var data1 = dataSource.ReadData();
var data2 = dataSource.ReadData(); // จาก cache
```

## ขั้นตอนที่ 608: Observer Pattern

```csharp
// Observer: เมื่อ Subject เปลี่ยนแปลง ทุก Observer จะได้รับแจ้งโดยอัตโนมัติ

// Event-based Observer (C# native)
public class StockMarket
{
    public event EventHandler<StockPriceChangedEventArgs>? PriceChanged;
    public event EventHandler<TradingHaltEventArgs>? TradingHalted;
    
    private readonly Dictionary<string, decimal> _prices = [];
    
    public void UpdatePrice(string symbol, decimal newPrice)
    {
        if (_prices.TryGetValue(symbol, out var oldPrice))
        {
            var change = ((newPrice - oldPrice) / oldPrice) * 100;
            _prices[symbol] = newPrice;
            PriceChanged?.Invoke(this, new StockPriceChangedEventArgs(symbol, oldPrice, newPrice, change));
            
            // ตรวจ circuit breaker
            if (Math.Abs(change) >= 10)
            {
                TradingHalted?.Invoke(this, new TradingHaltEventArgs(symbol, change, TimeSpan.FromMinutes(15)));
            }
        }
        else
        {
            _prices[symbol] = newPrice;
        }
    }
    
    public decimal GetPrice(string symbol)
        => _prices.GetValueOrDefault(symbol, 0);
}

public class StockPriceChangedEventArgs(string symbol, decimal oldPrice, decimal newPrice, decimal changePercent)
    : EventArgs
{
    public string Symbol { get; } = symbol;
    public decimal OldPrice { get; } = oldPrice;
    public decimal NewPrice { get; } = newPrice;
    public decimal ChangePercent { get; } = changePercent;
    public bool IsUp => NewPrice > OldPrice;
}

public class TradingHaltEventArgs(string symbol, decimal changePercent, TimeSpan duration) 
    : EventArgs
{
    public string Symbol { get; } = symbol;
    public decimal ChangePercent { get; } = changePercent;
    public TimeSpan Duration { get; } = duration;
}

// Observers
public class PortfolioTracker
{
    private readonly Dictionary<string, int> _holdings;
    private decimal _totalValue = 0;
    
    public PortfolioTracker(Dictionary<string, int> holdings)
    {
        _holdings = holdings;
    }
    
    public void OnPriceChanged(object? sender, StockPriceChangedEventArgs e)
    {
        if (_holdings.TryGetValue(e.Symbol, out var shares))
        {
            var gain = (e.NewPrice - e.OldPrice) * shares;
            Console.WriteLine($"[Portfolio] {e.Symbol}: {e.OldPrice:F2} -> {e.NewPrice:F2} " +
                $"({e.ChangePercent:+0.00;-0.00}%) | P&L: {gain:+0.00;-0.00}");
        }
    }
}

public class TradingAlertSystem
{
    private readonly decimal _alertThreshold;
    
    public TradingAlertSystem(decimal alertThreshold = 5m)
    {
        _alertThreshold = alertThreshold;
    }
    
    public void OnPriceChanged(object? sender, StockPriceChangedEventArgs e)
    {
        if (Math.Abs(e.ChangePercent) >= _alertThreshold)
        {
            var direction = e.IsUp ? "⬆️ UP" : "⬇️ DOWN";
            Console.WriteLine($"[ALERT] {e.Symbol} {direction} {Math.Abs(e.ChangePercent):F2}%!");
        }
    }
    
    public void OnTradingHalted(object? sender, TradingHaltEventArgs e)
    {
        Console.WriteLine($"[HALT] {e.Symbol} trading halted for {e.Duration.TotalMinutes} minutes " +
            $"(change: {e.ChangePercent:+0.00;-0.00}%)");
    }
}

// IObserver<T> interface (BCL)
public class MarketDataLogger : IObserver<(string Symbol, decimal Price)>
{
    private IDisposable? _subscription;
    
    public void Subscribe(IObservable<(string Symbol, decimal Price)> provider)
    {
        _subscription = provider.Subscribe(this);
    }
    
    public void OnNext((string Symbol, decimal Price) value)
        => Console.WriteLine($"[Log] {DateTime.Now:HH:mm:ss.fff} {value.Symbol}: {value.Price:F2}");
    
    public void OnError(Exception error)
        => Console.WriteLine($"[Log] Error: {error.Message}");
    
    public void OnCompleted()
    {
        Console.WriteLine("[Log] Market closed");
        _subscription?.Dispose();
    }
}

// การใช้งาน
var market = new StockMarket();

var portfolio = new PortfolioTracker(new() { ["AAPL"] = 100, ["MSFT"] = 50 });
var alertSystem = new TradingAlertSystem(alertThreshold: 3m);

market.PriceChanged += portfolio.OnPriceChanged;
market.PriceChanged += alertSystem.OnPriceChanged;
market.TradingHalted += alertSystem.OnTradingHalted;

market.UpdatePrice("AAPL", 150.00m);
market.UpdatePrice("AAPL", 153.75m); // +2.5% - ไม่ alert
market.UpdatePrice("AAPL", 145.00m); // -5.7% - alert!
market.UpdatePrice("AAPL", 130.00m); // -10.3% - halt!
```

## ขั้นตอนที่ 609: Strategy Pattern

```csharp
// Strategy: กำหนด family of algorithms และทำให้สลับกันได้

// Context
public class ShippingCalculator
{
    private IShippingStrategy _strategy;
    
    public ShippingCalculator(IShippingStrategy initialStrategy)
    {
        _strategy = initialStrategy;
    }
    
    public void SetStrategy(IShippingStrategy strategy)
    {
        _strategy = strategy;
    }
    
    public decimal Calculate(Order order)
    {
        Console.WriteLine($"Calculating shipping using {_strategy.Name}...");
        return _strategy.Calculate(order);
    }
}

// Strategy Interface
public interface IShippingStrategy
{
    string Name { get; }
    decimal Calculate(Order order);
}

// Concrete Strategies
public class StandardShipping : IShippingStrategy
{
    public string Name => "Standard Shipping (3-5 days)";
    
    public decimal Calculate(Order order)
    {
        // ฟรีถ้าสั่งมากกว่า 1,000 บาท
        if (order.TotalAmount >= 1000) return 0;
        return order.TotalWeight <= 1 ? 50 : 50 + (order.TotalWeight - 1) * 20;
    }
}

public class ExpressShipping : IShippingStrategy
{
    public string Name => "Express Shipping (1-2 days)";
    
    public decimal Calculate(Order order)
    {
        var baseRate = 150m;
        var weightCharge = order.TotalWeight * 30;
        var distanceMultiplier = order.DestinationRegion switch
        {
            "Bangkok" => 1.0m,
            "Central" => 1.2m,
            "North" or "South" or "East" or "Northeast" => 1.5m,
            _ => 2.0m
        };
        return (baseRate + weightCharge) * distanceMultiplier;
    }
}

public class SameDayShipping : IShippingStrategy
{
    private readonly TimeSpan _cutoffTime;
    
    public SameDayShipping(TimeSpan? cutoffTime = null)
    {
        _cutoffTime = cutoffTime ?? new TimeSpan(14, 0, 0); // 2 PM
    }
    
    public string Name => "Same Day Shipping";
    
    public decimal Calculate(Order order)
    {
        if (TimeOnly.FromDateTime(DateTime.Now) > TimeOnly.FromTimeSpan(_cutoffTime))
            throw new InvalidOperationException(
                $"Same-day shipping cutoff is {_cutoffTime:hh\\:mm}");
        
        if (order.DestinationRegion != "Bangkok")
            throw new InvalidOperationException("Same-day shipping only available in Bangkok");
        
        return 300 + order.TotalWeight * 50;
    }
}

public class PickupStrategy : IShippingStrategy
{
    public string Name => "Store Pickup";
    public decimal Calculate(Order order) => 0; // Free
}

// Smart Strategy Selection
public class ShippingStrategySelector
{
    private readonly List<IShippingStrategy> _strategies;
    
    public ShippingStrategySelector()
    {
        _strategies =
        [
            new PickupStrategy(),
            new StandardShipping(),
            new ExpressShipping(),
            new SameDayShipping()
        ];
    }
    
    public IEnumerable<(IShippingStrategy Strategy, decimal Cost)> GetAllOptions(Order order)
    {
        foreach (var strategy in _strategies)
        {
            decimal cost;
            try
            {
                cost = strategy.Calculate(order);
            }
            catch
            {
                continue; // Strategy not available for this order
            }
            yield return (strategy, cost);
        }
    }
    
    public IShippingStrategy GetCheapest(Order order)
        => GetAllOptions(order).MinBy(x => x.Cost).Strategy;
}

public record Order(
    string CustomerId,
    decimal TotalAmount,
    double TotalWeight,
    string DestinationRegion);

// การใช้งาน
var order = new Order("C001", 500, 2.5, "Bangkok");
var selector = new ShippingStrategySelector();

Console.WriteLine("Available shipping options:");
foreach (var (strategy, cost) in selector.GetAllOptions(order))
{
    Console.WriteLine($"  {strategy.Name}: {cost:F2} THB");
}

var calculator = new ShippingCalculator(new StandardShipping());
Console.WriteLine($"\nStandard: {calculator.Calculate(order):F2} THB");

calculator.SetStrategy(new ExpressShipping());
Console.WriteLine($"Express: {calculator.Calculate(order):F2} THB");
```

## ขั้นตอนที่ 610: Command Pattern

```csharp
// Command: encapsulate request เป็น object เพื่อให้ undo/redo ได้

// Command Interface
public interface ICommand
{
    void Execute();
    void Undo();
    string Description { get; }
}

// Receiver
public class TextDocument
{
    private readonly System.Text.StringBuilder _text = new();
    private int _cursorPosition = 0;
    
    public string Text => _text.ToString();
    public int CursorPosition => _cursorPosition;
    
    public void Insert(int position, string text)
    {
        _text.Insert(position, text);
        _cursorPosition = position + text.Length;
    }
    
    public string Delete(int position, int length)
    {
        var deleted = _text.ToString(position, length);
        _text.Remove(position, length);
        _cursorPosition = position;
        return deleted;
    }
    
    public void SetCursor(int position)
    {
        _cursorPosition = Math.Clamp(position, 0, _text.Length);
    }
    
    public override string ToString() => _text.ToString();
}

// Concrete Commands
public class InsertTextCommand : ICommand
{
    private readonly TextDocument _doc;
    private readonly int _position;
    private readonly string _text;
    
    public InsertTextCommand(TextDocument doc, int position, string text)
    {
        _doc = doc;
        _position = position;
        _text = text;
    }
    
    public string Description => $"Insert '{_text}' at {_position}";
    
    public void Execute() => _doc.Insert(_position, _text);
    
    public void Undo() => _doc.Delete(_position, _text.Length);
}

public class DeleteTextCommand : ICommand
{
    private readonly TextDocument _doc;
    private readonly int _position;
    private readonly int _length;
    private string _deletedText = "";
    
    public DeleteTextCommand(TextDocument doc, int position, int length)
    {
        _doc = doc;
        _position = position;
        _length = length;
    }
    
    public string Description => $"Delete {_length} chars at {_position}";
    
    public void Execute()
    {
        _deletedText = _doc.Delete(_position, _length);
    }
    
    public void Undo() => _doc.Insert(_position, _deletedText);
}

public class ReplaceTextCommand : ICommand
{
    private readonly ICommand _delete;
    private readonly ICommand _insert;
    
    public ReplaceTextCommand(TextDocument doc, int position, int deleteLength, string newText)
    {
        _delete = new DeleteTextCommand(doc, position, deleteLength);
        _insert = new InsertTextCommand(doc, position, newText);
    }
    
    public string Description => "Replace text";
    
    public void Execute()
    {
        _delete.Execute();
        _insert.Execute();
    }
    
    public void Undo()
    {
        _insert.Undo();
        _delete.Undo();
    }
}

// Invoker: Command History Manager
public class CommandHistory
{
    private readonly Stack<ICommand> _history = new();
    private readonly Stack<ICommand> _redoStack = new();
    private readonly int _maxHistory;
    
    public CommandHistory(int maxHistory = 100)
    {
        _maxHistory = maxHistory;
    }
    
    public bool CanUndo => _history.Count > 0;
    public bool CanRedo => _redoStack.Count > 0;
    public int HistoryCount => _history.Count;
    
    public void Execute(ICommand command)
    {
        command.Execute();
        _history.Push(command);
        _redoStack.Clear(); // ล้าง redo stack เมื่อมี command ใหม่
        
        // จำกัดจำนวน history
        while (_history.Count > _maxHistory)
        {
            // ไม่สามารถ remove จาก bottom ของ Stack โดยตรง
            // ในทางปฏิบัติอาจใช้ LinkedList แทน
        }
        
        Console.WriteLine($"Execute: {command.Description}");
    }
    
    public void Undo()
    {
        if (!CanUndo) throw new InvalidOperationException("Nothing to undo");
        
        var command = _history.Pop();
        command.Undo();
        _redoStack.Push(command);
        Console.WriteLine($"Undo: {command.Description}");
    }
    
    public void Redo()
    {
        if (!CanRedo) throw new InvalidOperationException("Nothing to redo");
        
        var command = _redoStack.Pop();
        command.Execute();
        _history.Push(command);
        Console.WriteLine($"Redo: {command.Description}");
    }
    
    public IEnumerable<string> GetHistory()
        => _history.Select(c => c.Description);
}

// การใช้งาน
var doc = new TextDocument();
var history = new CommandHistory();

history.Execute(new InsertTextCommand(doc, 0, "Hello"));
history.Execute(new InsertTextCommand(doc, 5, " World"));
history.Execute(new InsertTextCommand(doc, 11, "!"));
Console.WriteLine($"Text: {doc.Text}");  // Hello World!

history.Execute(new ReplaceTextCommand(doc, 6, 5, "C#"));
Console.WriteLine($"Text: {doc.Text}");  // Hello C#!

history.Undo();
Console.WriteLine($"After undo: {doc.Text}");  // Hello World!

history.Redo();
Console.WriteLine($"After redo: {doc.Text}");  // Hello C#!

Console.WriteLine("\nHistory:");
foreach (var item in history.GetHistory())
    Console.WriteLine($"  {item}");
```

## ขั้นตอนที่ 611: Chain of Responsibility Pattern

```csharp
// Chain of Responsibility: ส่ง request ผ่าน chain ของ handlers

public record HttpRequest(string Method, string Path, Dictionary<string, string> Headers, string? Body);
public record HttpResponse(int StatusCode, string Body);

// Handler Interface
public abstract class HttpMiddleware
{
    protected HttpMiddleware? _next;
    
    public HttpMiddleware Use(HttpMiddleware next)
    {
        _next = next;
        return next;
    }
    
    public abstract Task<HttpResponse> HandleAsync(HttpRequest request);
    
    protected Task<HttpResponse> PassToNextAsync(HttpRequest request)
    {
        if (_next is null)
            return Task.FromResult(new HttpResponse(404, "Not Found"));
        return _next.HandleAsync(request);
    }
}

// Concrete Handlers (Middleware)
public class AuthenticationMiddleware : HttpMiddleware
{
    private readonly HashSet<string> _validTokens = ["token123", "admin-token"];
    
    public override async Task<HttpResponse> HandleAsync(HttpRequest request)
    {
        if (request.Path.StartsWith("/public/"))
            return await PassToNextAsync(request);
        
        if (!request.Headers.TryGetValue("Authorization", out var authHeader) ||
            !authHeader.StartsWith("Bearer "))
        {
            Console.WriteLine("[Auth] Missing or invalid Authorization header");
            return new HttpResponse(401, "Unauthorized");
        }
        
        var token = authHeader["Bearer ".Length..];
        if (!_validTokens.Contains(token))
        {
            Console.WriteLine($"[Auth] Invalid token: {token}");
            return new HttpResponse(401, "Invalid token");
        }
        
        Console.WriteLine($"[Auth] Authenticated with token: {token[..4]}...");
        return await PassToNextAsync(request);
    }
}

public class RateLimitingMiddleware : HttpMiddleware
{
    private readonly Dictionary<string, (int Count, DateTime Reset)> _requestCounts = [];
    private readonly int _maxRequests;
    private readonly TimeSpan _window;
    
    public RateLimitingMiddleware(int maxRequests = 10, TimeSpan? window = null)
    {
        _maxRequests = maxRequests;
        _window = window ?? TimeSpan.FromMinutes(1);
    }
    
    public override async Task<HttpResponse> HandleAsync(HttpRequest request)
    {
        var clientId = request.Headers.GetValueOrDefault("X-Client-Id", "anonymous");
        
        if (_requestCounts.TryGetValue(clientId, out var entry))
        {
            if (DateTime.Now > entry.Reset)
                _requestCounts[clientId] = (1, DateTime.Now + _window);
            else if (entry.Count >= _maxRequests)
            {
                Console.WriteLine($"[RateLimit] Client {clientId} exceeded limit");
                return new HttpResponse(429, "Too Many Requests");
            }
            else
                _requestCounts[clientId] = (entry.Count + 1, entry.Reset);
        }
        else
        {
            _requestCounts[clientId] = (1, DateTime.Now + _window);
        }
        
        Console.WriteLine($"[RateLimit] Client {clientId}: {_requestCounts[clientId].Count}/{_maxRequests}");
        return await PassToNextAsync(request);
    }
}

public class LoggingMiddleware : HttpMiddleware
{
    public override async Task<HttpResponse> HandleAsync(HttpRequest request)
    {
        var sw = System.Diagnostics.Stopwatch.StartNew();
        Console.WriteLine($"[Log] --> {request.Method} {request.Path}");
        
        var response = await PassToNextAsync(request);
        
        sw.Stop();
        Console.WriteLine($"[Log] <-- {response.StatusCode} ({sw.ElapsedMilliseconds}ms)");
        return response;
    }
}

public class RouterMiddleware : HttpMiddleware
{
    private readonly Dictionary<string, Func<HttpRequest, Task<HttpResponse>>> _routes = [];
    
    public RouterMiddleware Get(string path, Func<HttpRequest, Task<HttpResponse>> handler)
    {
        _routes[$"GET:{path}"] = handler;
        return this;
    }
    
    public RouterMiddleware Post(string path, Func<HttpRequest, Task<HttpResponse>> handler)
    {
        _routes[$"POST:{path}"] = handler;
        return this;
    }
    
    public override async Task<HttpResponse> HandleAsync(HttpRequest request)
    {
        var key = $"{request.Method}:{request.Path}";
        if (_routes.TryGetValue(key, out var handler))
            return await handler(request);
        
        return new HttpResponse(404, $"Route not found: {request.Method} {request.Path}");
    }
}

// Setup pipeline
var router = new RouterMiddleware()
    .Get("/api/users", async req => 
        await Task.FromResult(new HttpResponse(200, """[{"id":1,"name":"Alice"}]""")))
    .Post("/api/users", async req =>
        await Task.FromResult(new HttpResponse(201, """{"id":2,"name":"Bob"}""")))
    .Get("/public/health", async _ => 
        await Task.FromResult(new HttpResponse(200, "OK")));

var logging = new LoggingMiddleware();
var auth = new AuthenticationMiddleware();
var rateLimit = new RateLimitingMiddleware(maxRequests: 5);

// Build pipeline: Logging -> RateLimit -> Auth -> Router
logging.Use(rateLimit).Use(auth).Use(router);

// Test requests
var request1 = new HttpRequest("GET", "/api/users", 
    new() { ["Authorization"] = "Bearer token123", ["X-Client-Id"] = "client1" }, null);
var response1 = await logging.HandleAsync(request1);
Console.WriteLine($"Response: {response1.StatusCode}\n");

var request2 = new HttpRequest("GET", "/public/health", [], null);
var response2 = await logging.HandleAsync(request2);
Console.WriteLine($"Response: {response2.StatusCode}\n");
```

## ขั้นตอนที่ 612: Facade Pattern

```csharp
// Facade: Interface ที่เรียบง่ายสำหรับ subsystem ที่ซับซ้อน

// Complex Subsystems
public class VideoEncoder
{
    public async Task<string> EncodeAsync(string sourcePath, string codec, string quality)
    {
        Console.WriteLine($"[VideoEncoder] Encoding {sourcePath} with {codec} @ {quality}");
        await Task.Delay(200);
        return $"{Path.GetFileNameWithoutExtension(sourcePath)}_encoded.mp4";
    }
}

public class AudioNormalizer
{
    public async Task<string> NormalizeAsync(string sourcePath, float targetDb)
    {
        Console.WriteLine($"[AudioNormalizer] Normalizing to {targetDb}dB");
        await Task.Delay(100);
        return $"{Path.GetFileNameWithoutExtension(sourcePath)}_normalized.aac";
    }
}

public class ThumbnailGenerator
{
    public async Task<string> GenerateAsync(string videoPath, TimeSpan position)
    {
        Console.WriteLine($"[ThumbnailGenerator] Generating thumbnail at {position}");
        await Task.Delay(50);
        return $"{Path.GetFileNameWithoutExtension(videoPath)}_thumb.jpg";
    }
}

public class MetadataExtractor
{
    public async Task<VideoMetadata> ExtractAsync(string filePath)
    {
        Console.WriteLine($"[MetadataExtractor] Extracting metadata from {filePath}");
        await Task.Delay(30);
        return new VideoMetadata("1920x1080", TimeSpan.FromMinutes(5), 30, 128);
    }
}

public class CdnUploader
{
    public async Task<string> UploadAsync(string filePath, string destination)
    {
        Console.WriteLine($"[CdnUploader] Uploading {filePath} to CDN");
        await Task.Delay(300);
        return $"https://cdn.example.com/{destination}/{Path.GetFileName(filePath)}";
    }
}

public record VideoMetadata(string Resolution, TimeSpan Duration, int FrameRate, int BitrateKbps);

// Facade
public class VideoProcessingFacade
{
    private readonly VideoEncoder _encoder = new();
    private readonly AudioNormalizer _normalizer = new();
    private readonly ThumbnailGenerator _thumbnailGen = new();
    private readonly MetadataExtractor _metaExtractor = new();
    private readonly CdnUploader _uploader = new();
    
    public async Task<ProcessedVideoResult> ProcessAndPublishAsync(
        string sourcePath,
        VideoProcessingOptions? options = null)
    {
        options ??= new VideoProcessingOptions();
        
        Console.WriteLine($"\n=== Processing: {sourcePath} ===");
        
        // Step 1: Extract metadata
        var metadata = await _metaExtractor.ExtractAsync(sourcePath);
        Console.WriteLine($"Duration: {metadata.Duration}, Resolution: {metadata.Resolution}");
        
        // Step 2: Parallel processing
        var encodeTask = _encoder.EncodeAsync(sourcePath, options.Codec, options.Quality);
        var normalizeTask = _normalizer.NormalizeAsync(sourcePath, options.TargetAudioDb);
        var thumbTask = _thumbnailGen.GenerateAsync(sourcePath, metadata.Duration / 3);
        
        await Task.WhenAll(encodeTask, normalizeTask, thumbTask);
        
        var encodedVideo = await encodeTask;
        var normalizedAudio = await normalizeTask;
        var thumbnail = await thumbTask;
        
        // Step 3: Upload to CDN
        var videoFolder = $"videos/{DateTime.Now:yyyy/MM/dd}";
        var (videoUrl, thumbUrl) = await (
            _uploader.UploadAsync(encodedVideo, videoFolder),
            _uploader.UploadAsync(thumbnail, $"{videoFolder}/thumbnails"));
        
        Console.WriteLine("=== Processing Complete ===\n");
        
        return new ProcessedVideoResult(
            VideoUrl: videoUrl,
            ThumbnailUrl: thumbUrl,
            Duration: metadata.Duration,
            Resolution: metadata.Resolution);
    }
}

public class VideoProcessingOptions
{
    public string Codec { get; init; } = "H.264";
    public string Quality { get; init; } = "720p";
    public float TargetAudioDb { get; init; } = -14.0f;
}

public record ProcessedVideoResult(
    string VideoUrl, 
    string ThumbnailUrl, 
    TimeSpan Duration, 
    string Resolution);

// การใช้งาน - Simple interface
var processor = new VideoProcessingFacade();
var result = await processor.ProcessAndPublishAsync(
    "raw_video.mov",
    new VideoProcessingOptions { Quality = "1080p", Codec = "H.265" });

Console.WriteLine($"Video: {result.VideoUrl}");
Console.WriteLine($"Thumbnail: {result.ThumbnailUrl}");
```

## ขั้นตอนที่ 613: Template Method Pattern

```csharp
// Template Method: กำหนด skeleton ของ algorithm ใน base class
// ให้ subclass override ขั้นตอนเฉพาะ

public abstract class ReportGenerator
{
    // Template Method - กำหนด algorithm skeleton
    public async Task<string> GenerateAsync(ReportConfig config)
    {
        Console.WriteLine($"Generating {GetReportName()} report...");
        
        var data = await FetchDataAsync(config);
        ValidateData(data);
        var processed = ProcessData(data, config);
        var formatted = FormatReport(processed, config);
        
        if (config.SendEmail)
            await SendEmailAsync(formatted, config.Recipients);
        
        if (config.SaveToFile)
            await SaveToFileAsync(formatted, config.FilePath);
        
        Console.WriteLine($"{GetReportName()} report complete.");
        return formatted;
    }
    
    // Abstract steps - ต้อง implement
    protected abstract string GetReportName();
    protected abstract Task<List<DataRow>> FetchDataAsync(ReportConfig config);
    protected abstract string FormatReport(List<ProcessedRow> data, ReportConfig config);
    
    // Virtual steps - optional override
    protected virtual void ValidateData(List<DataRow> data)
    {
        if (data.Count == 0)
            throw new InvalidOperationException("No data to report");
        Console.WriteLine($"Validating {data.Count} rows...");
    }
    
    protected virtual List<ProcessedRow> ProcessData(List<DataRow> data, ReportConfig config)
    {
        return data.Select(row => new ProcessedRow(row.Label, row.Value, row.Date)).ToList();
    }
    
    protected virtual async Task SendEmailAsync(string content, string[] recipients)
    {
        Console.WriteLine($"Sending to: {string.Join(", ", recipients)}");
        await Task.Delay(100);
    }
    
    protected virtual async Task SaveToFileAsync(string content, string? path)
    {
        if (path is null) return;
        await File.WriteAllTextAsync(path, content);
        Console.WriteLine($"Saved to: {path}");
    }
}

public record DataRow(string Label, decimal Value, DateTime Date);
public record ProcessedRow(string Label, decimal Value, DateTime Date);
public record ReportConfig(
    string Period,
    bool SendEmail = false,
    string[] Recipients = null!,
    bool SaveToFile = false,
    string? FilePath = null);

// Concrete Implementations
public class SalesReportGenerator : ReportGenerator
{
    private readonly ISalesRepository _salesRepo;
    
    public SalesReportGenerator(ISalesRepository salesRepo)
    {
        _salesRepo = salesRepo;
    }
    
    protected override string GetReportName() => "Sales";
    
    protected override async Task<List<DataRow>> FetchDataAsync(ReportConfig config)
    {
        Console.WriteLine($"Fetching sales data for {config.Period}...");
        return await _salesRepo.GetSalesByPeriodAsync(config.Period);
    }
    
    protected override List<ProcessedRow> ProcessData(List<DataRow> data, ReportConfig config)
    {
        // เพิ่ม cumulative sum
        var processed = base.ProcessData(data, config);
        return processed.OrderBy(r => r.Date).ToList();
    }
    
    protected override string FormatReport(List<ProcessedRow> data, ReportConfig config)
    {
        var sb = new System.Text.StringBuilder();
        sb.AppendLine($"=== SALES REPORT - {config.Period} ===");
        sb.AppendLine($"Generated: {DateTime.Now:yyyy-MM-dd HH:mm}");
        sb.AppendLine();
        
        foreach (var row in data)
            sb.AppendLine($"{row.Date:dd/MM}: {row.Label,-20} {row.Value,10:N2} THB");
        
        sb.AppendLine($"\nTotal: {data.Sum(r => r.Value):N2} THB");
        return sb.ToString();
    }
}

public class InventoryReportGenerator : ReportGenerator
{
    protected override string GetReportName() => "Inventory";
    
    protected override Task<List<DataRow>> FetchDataAsync(ReportConfig config)
    {
        // Simulate DB query
        var data = new List<DataRow>
        {
            new("Product A", 150, DateTime.Today),
            new("Product B", 0, DateTime.Today),
            new("Product C", 42, DateTime.Today)
        };
        return Task.FromResult(data);
    }
    
    protected override void ValidateData(List<DataRow> data)
    {
        base.ValidateData(data);
        
        // Additional validation: แจ้งเตือนสินค้าหมด
        var outOfStock = data.Where(d => d.Value == 0).ToList();
        if (outOfStock.Any())
        {
            Console.WriteLine($"WARNING: {outOfStock.Count} items out of stock!");
            foreach (var item in outOfStock)
                Console.WriteLine($"  - {item.Label}");
        }
    }
    
    protected override string FormatReport(List<ProcessedRow> data, ReportConfig config)
    {
        var sb = new System.Text.StringBuilder();
        sb.AppendLine("=== INVENTORY REPORT ===");
        
        foreach (var row in data)
        {
            var status = row.Value == 0 ? "OUT OF STOCK" : row.Value < 10 ? "LOW" : "OK";
            sb.AppendLine($"{row.Label,-20} Qty: {row.Value,6} [{status}]");
        }
        
        return sb.ToString();
    }
}
```

## ขั้นตอนที่ 614: State Pattern

```csharp
// State: เปลี่ยน behavior ของ object ตาม internal state

public class VendingMachine
{
    private IVendingState _state;
    
    public int SelectedItem { get; set; } = -1;
    public decimal InsertedAmount { get; private set; } = 0;
    public Dictionary<int, (string Name, decimal Price, int Stock)> Inventory { get; } = new()
    {
        [1] = ("Cola", 20, 5),
        [2] = ("Water", 15, 3),
        [3] = ("Chips", 35, 0)
    };
    
    public VendingMachine()
    {
        _state = new IdleState(this);
    }
    
    internal void SetState(IVendingState state)
    {
        Console.WriteLine($"State: {_state.GetType().Name} -> {state.GetType().Name}");
        _state = state;
    }
    
    internal void AddMoney(decimal amount)
    {
        InsertedAmount += amount;
        Console.WriteLine($"Inserted: {amount:F2} THB (Total: {InsertedAmount:F2} THB)");
    }
    
    internal decimal ReturnMoney()
    {
        var amount = InsertedAmount;
        InsertedAmount = 0;
        if (amount > 0)
            Console.WriteLine($"Returning: {amount:F2} THB");
        return amount;
    }
    
    internal void DispenseItem(int itemId)
    {
        var (name, price, stock) = Inventory[itemId];
        InsertedAmount -= price;
        Inventory[itemId] = (name, price, stock - 1);
        Console.WriteLine($"Dispensed: {name} (Remaining: {stock - 1})");
        
        if (InsertedAmount > 0)
        {
            Console.WriteLine($"Change: {InsertedAmount:F2} THB");
            InsertedAmount = 0;
        }
    }
    
    // Delegate to state
    public void InsertCoin(decimal amount) => _state.InsertCoin(amount);
    public void SelectItem(int itemId) => _state.SelectItem(itemId);
    public void DispenseItem() => _state.DispenseItem();
    public void Cancel() => _state.Cancel();
}

public interface IVendingState
{
    void InsertCoin(decimal amount);
    void SelectItem(int itemId);
    void DispenseItem();
    void Cancel();
}

public class IdleState : IVendingState
{
    private readonly VendingMachine _machine;
    
    public IdleState(VendingMachine machine) => _machine = machine;
    
    public void InsertCoin(decimal amount)
    {
        _machine.AddMoney(amount);
        _machine.SetState(new HasMoneyState(_machine));
    }
    
    public void SelectItem(int itemId)
        => Console.WriteLine("Please insert money first");
    
    public void DispenseItem()
        => Console.WriteLine("No item selected");
    
    public void Cancel()
        => Console.WriteLine("Nothing to cancel");
}

public class HasMoneyState : IVendingState
{
    private readonly VendingMachine _machine;
    
    public HasMoneyState(VendingMachine machine) => _machine = machine;
    
    public void InsertCoin(decimal amount) => _machine.AddMoney(amount);
    
    public void SelectItem(int itemId)
    {
        if (!_machine.Inventory.TryGetValue(itemId, out var item))
        {
            Console.WriteLine($"Item {itemId} not found");
            return;
        }
        
        var (name, price, stock) = item;
        
        if (stock == 0)
        {
            Console.WriteLine($"{name} is out of stock");
            return;
        }
        
        if (_machine.InsertedAmount < price)
        {
            Console.WriteLine($"Insufficient amount. Need {price - _machine.InsertedAmount:F2} more");
            return;
        }
        
        _machine.SelectedItem = itemId;
        _machine.SetState(new ItemSelectedState(_machine));
        Console.WriteLine($"Selected: {name} ({price:F2} THB)");
    }
    
    public void DispenseItem()
        => Console.WriteLine("Please select an item first");
    
    public void Cancel()
    {
        _machine.ReturnMoney();
        _machine.SetState(new IdleState(_machine));
    }
}

public class ItemSelectedState : IVendingState
{
    private readonly VendingMachine _machine;
    
    public ItemSelectedState(VendingMachine machine) => _machine = machine;
    
    public void InsertCoin(decimal amount) => _machine.AddMoney(amount);
    
    public void SelectItem(int itemId)
    {
        _machine.SelectedItem = itemId;
        Console.WriteLine($"Item changed to {itemId}");
    }
    
    public void DispenseItem()
    {
        _machine.DispenseItem(_machine.SelectedItem);
        
        // ตรวจสอบว่ายังมีสินค้าเหลือหรือไม่
        var hasInventory = _machine.Inventory.Values.Any(i => i.Stock > 0);
        _machine.SetState(hasInventory ? new IdleState(_machine) : new OutOfStockState(_machine));
    }
    
    public void Cancel()
    {
        _machine.SelectedItem = -1;
        _machine.ReturnMoney();
        _machine.SetState(new IdleState(_machine));
    }
}

public class OutOfStockState : IVendingState
{
    private readonly VendingMachine _machine;
    
    public OutOfStockState(VendingMachine machine)
    {
        _machine = machine;
        Console.WriteLine("MACHINE OUT OF STOCK - Service required");
    }
    
    public void InsertCoin(decimal amount)
    {
        Console.WriteLine("Machine is out of stock. Money returned.");
        _machine.ReturnMoney();
    }
    
    public void SelectItem(int itemId) => Console.WriteLine("Machine is out of stock");
    public void DispenseItem() => Console.WriteLine("Machine is out of stock");
    public void Cancel() => _machine.ReturnMoney();
}

// การใช้งาน
var vm = new VendingMachine();
vm.InsertCoin(20m);
vm.SelectItem(1);    // Cola
vm.DispenseItem();

Console.WriteLine();
vm.InsertCoin(50m);
vm.SelectItem(3);    // Chips - out of stock
vm.SelectItem(2);    // Water
vm.DispenseItem();   // เหลือเงินทอน 35 บาท
```

## ขั้นตอนที่ 615: สรุป Design Patterns

```
Design Patterns ที่สำคัญที่สุดสำหรับ C# Developers:

Creational (สร้าง Objects):
─────────────────────────────────────────────────────────────────
Singleton   → ต้องการ 1 instance (Config, Logger, Cache)
             ✓ ใช้ DI แทน: AddSingleton<T>()

Factory     → ต้องการ create objects แบบ flexible
Method       ✓ ใช้บ่อยใน Framework, DI containers

Builder     → Object มีหลาย optional parameters
             ✓ HttpClient, StringBuilder, QueryBuilder

Structural (จัดโครงสร้าง):
─────────────────────────────────────────────────────────────────
Adapter     → ต้องการให้ incompatible interfaces ทำงานด้วยกัน
             ✓ Legacy code integration, 3rd party adapters

Decorator   → เพิ่ม behavior โดยไม่แก้ class เดิม
             ✓ Logging, Caching, Validation wrappers

Facade      → ซ่อน complexity ด้วย simple interface
             ✓ Service layer, SDK APIs

Behavioral (พฤติกรรม):
─────────────────────────────────────────────────────────────────
Observer    → Pub/Sub, event-driven architecture
             ✓ UI events, Domain events, SignalR

Strategy    → เปลี่ยน algorithm ที่ runtime
             ✓ Sorting, Pricing, Shipping calculations

Command     → Undo/Redo, Queue operations
             ✓ CQRS, Transaction scripts

Chain of    → Pipeline processing
Resp.        ✓ ASP.NET Middleware, Validators

Template    → Algorithm skeleton, customize steps
Method       ✓ Base classes, Framework hooks

State       → FSM, behavior ตาม state
             ✓ Order lifecycle, Game states

─────────────────────────────────────────────────────────────────
"Use patterns to solve problems, not to show off"
```

## สรุป Part 20

ใน Part นี้คุณได้เรียนรู้:
- **Creational**: Singleton, Factory Method, Abstract Factory, Builder
- **Structural**: Adapter, Decorator, Facade  
- **Behavioral**: Observer, Strategy, Command, Chain of Responsibility, Template Method, State
- แต่ละ pattern มีตัวอย่าง real-world use case ใน C#
- ใช้ pattern เพื่อแก้ปัญหา ไม่ใช่เพื่อเพิ่ม complexity

## แบบฝึกหัด

1. Implement **Proxy Pattern** สำหรับ lazy loading images ใน gallery app
2. สร้าง **Composite Pattern** สำหรับ file system tree (File/Folder)
3. ใช้ **Flyweight Pattern** เพื่อลด memory สำหรับ game objects (ต้นไม้ 1,000 ต้น)
4. สร้าง **Mediator Pattern** สำหรับ chat room ที่ users communicate ผ่าน mediator
5. Implement **Visitor Pattern** สำหรับ AST (Abstract Syntax Tree) ของ math expressions
