# Part 03: ตัวดำเนินการและนิพจน์ (Operators & Expressions)
## Steps 31-50: การจัดการนิพจน์และเงื่อนไขใน C#

---

## Step 31: Comparison Operators

```csharp
// ============ Comparison Operators ============
int a = 10, b = 20;

bool eq = (a == b);    // Equal: false
bool neq = (a != b);   // Not Equal: true
bool lt = (a < b);     // Less than: true
bool gt = (a > b);     // Greater than: false
bool lte = (a <= b);   // Less than or equal: true
bool gte = (a >= b);   // Greater than or equal: false

Console.WriteLine($"a={a}, b={b}");
Console.WriteLine($"a == b: {eq}");
Console.WriteLine($"a != b: {neq}");
Console.WriteLine($"a < b: {lt}");
Console.WriteLine($"a > b: {gt}");
Console.WriteLine($"a <= b: {lte}");
Console.WriteLine($"a >= b: {gte}");

// ============ String Comparison ============
string s1 = "Hello";
string s2 = "hello";
string s3 = "Hello";

Console.WriteLine($"\nString Comparison:");
Console.WriteLine($"s1 == s2: {s1 == s2}");          // false (case sensitive)
Console.WriteLine($"s1 == s3: {s1 == s3}");          // true
Console.WriteLine($"s1.Equals(s2): {s1.Equals(s2)}"); // false

// Case-insensitive comparison
Console.WriteLine($"OrdinalIgnoreCase: {string.Compare(s1, s2, StringComparison.OrdinalIgnoreCase) == 0}"); // true
Console.WriteLine($"EqualsIgnoreCase: {s1.Equals(s2, StringComparison.OrdinalIgnoreCase)}"); // true

// Ordinal vs Culture comparison
string str1 = "café";
string str2 = "cafe";
bool ordinal = string.Compare(str1, str2, StringComparison.Ordinal) == 0;
bool culture = string.Compare(str1, str2, StringComparison.InvariantCultureIgnoreCase) == 0;
Console.WriteLine($"Ordinal: {ordinal}, Culture: {culture}");

// ============ Reference Equality (object) ============
object obj1 = new object();
object obj2 = new object();
object obj3 = obj1; // Same reference

Console.WriteLine($"\nReference Equality:");
Console.WriteLine($"obj1 == obj2: {obj1 == obj2}");  // false (different objects)
Console.WriteLine($"obj1 == obj3: {obj1 == obj3}");  // true (same reference)
Console.WriteLine($"ReferenceEquals: {ReferenceEquals(obj1, obj2)}"); // false
Console.WriteLine($"ReferenceEquals: {ReferenceEquals(obj1, obj3)}"); // true

// ============ Comparing Numbers ============
double x = 0.1 + 0.2;
double y = 0.3;
Console.WriteLine($"\n0.1 + 0.2 == 0.3: {x == y}"); // false! floating point issue
Console.WriteLine($"0.1 + 0.2 = {x:R}");
Console.WriteLine($"0.3 = {y:R}");

// วิธีแก้: ใช้ epsilon comparison
double epsilon = 1e-10;
bool approxEqual = Math.Abs(x - y) < epsilon;
Console.WriteLine($"Approximately equal: {approxEqual}"); // true

// สำหรับ decimal ไม่มีปัญหานี้
decimal d1 = 0.1m + 0.2m;
decimal d2 = 0.3m;
Console.WriteLine($"\ndecimal: {d1 == d2}"); // true!
```

---

## Step 32: Logical Operators

```csharp
// ============ Logical Operators ============
bool t = true, f = false;

// AND: && (short-circuit)
Console.WriteLine($"true && true = {t && t}");   // true
Console.WriteLine($"true && false = {t && f}");  // false
Console.WriteLine($"false && true = {f && t}");  // false
Console.WriteLine($"false && false = {f && f}"); // false

// OR: || (short-circuit)
Console.WriteLine($"\ntrue || true = {t || t}");   // true
Console.WriteLine($"true || false = {t || f}");  // true
Console.WriteLine($"false || true = {f || t}");  // true
Console.WriteLine($"false || false = {f || f}"); // false

// NOT: !
Console.WriteLine($"\n!true = {!t}");   // false
Console.WriteLine($"!false = {!f}");  // true

// XOR: ^ (exclusive or)
Console.WriteLine($"\ntrue ^ true = {t ^ t}");   // false (ต่างกัน = true)
Console.WriteLine($"true ^ false = {t ^ f}");  // true
Console.WriteLine($"false ^ false = {f ^ f}"); // false

// ============ Short-circuit Evaluation ============
// && หยุดประเมินถ้าด้านซ้าย false
// || หยุดประเมินถ้าด้านซ้าย true

int count = 0;
bool Increment() { count++; Console.Write($"[evaluated#{count}] "); return true; }

count = 0;
bool result1 = false && Increment(); // Increment ไม่ถูกเรียก!
Console.WriteLine($"\nfalse && ... : count={count}"); // count=0

count = 0;
bool result2 = true || Increment(); // Increment ไม่ถูกเรียก!
Console.WriteLine($"true || ... : count={count}"); // count=0

count = 0;
bool result3 = true && Increment(); // Increment ถูกเรียก
Console.WriteLine($"\ntrue && ... : count={count}"); // count=1

// ✅ ประโยชน์: ป้องกัน NullReferenceException
string? str = null;
if (str != null && str.Length > 0) // ปลอดภัย: Length ไม่ถูกเรียกถ้า str = null
    Console.WriteLine("Not empty");

// ============ Non-short-circuit (& และ |) ============
count = 0;
bool result4 = false & Increment(); // Increment ยังถูกเรียก!
Console.WriteLine($"\nfalse & ... : count={count}"); // count=1

// ✅ ใช้ & กับ | เฉพาะ bitwise หรือเมื่อต้องการ side effects ทั้งสองด้าน

// ============ Compound Conditions ============
int age = 25;
double salary = 50000;
bool hasExperience = true;

bool qualified = age >= 21 && age <= 35 && salary > 30000 || hasExperience;
Console.WriteLine($"\nQualified: {qualified}");

// ใช้วงเล็บให้ชัดเจน
bool qualifiedClear = (age >= 21 && age <= 35 && salary > 30000) || hasExperience;
Console.WriteLine($"Qualified (clear): {qualifiedClear}");

// ============ De Morgan's Laws ============
bool x1 = true, y1 = false;
// !(A && B) == (!A || !B)
Console.WriteLine($"\n!(x1 && y1) = {!(x1 && y1)}");
Console.WriteLine($"(!x1 || !y1) = {!x1 || !y1}");
Console.WriteLine($"Equal: {!(x1 && y1) == (!x1 || !y1)}");
```

---

## Step 33: Bitwise Operators

```csharp
// ============ Bitwise Operators ============
// ทำงานกับ bits โดยตรง
int a = 0b1010;  // = 10
int b = 0b1100;  // = 12

// AND: &
int andResult = a & b; // 0b1000 = 8
Console.WriteLine($"  1010");
Console.WriteLine($"& 1100");
Console.WriteLine($"  ----");
Console.WriteLine($"  {Convert.ToString(andResult, 2).PadLeft(4, '0')} = {andResult}");

// OR: |
int orResult = a | b; // 0b1110 = 14
Console.WriteLine($"\n  1010");
Console.WriteLine($"| 1100");
Console.WriteLine($"  ----");
Console.WriteLine($"  {Convert.ToString(orResult, 2).PadLeft(4, '0')} = {orResult}");

// XOR: ^
int xorResult = a ^ b; // 0b0110 = 6
Console.WriteLine($"\n  1010");
Console.WriteLine($"^ 1100");
Console.WriteLine($"  ----");
Console.WriteLine($"  {Convert.ToString(xorResult, 2).PadLeft(4, '0')} = {xorResult}");

// NOT: ~
int notResult = ~a; // ≈ -11 (two's complement)
Console.WriteLine($"\n~ 1010 = {notResult} (two's complement)");

// Left Shift: <<
int leftShift = a << 2; // คูณด้วย 4
Console.WriteLine($"\n{a} << 2 = {leftShift} (= {a} * 4)");

// Right Shift: >>
int rightShift = a >> 1; // หารด้วย 2
Console.WriteLine($"{a} >> 1 = {rightShift} (= {a} / 2)");

// Unsigned Right Shift: >>> (C# 11+)
int negative = -1;
int unsignedShift = negative >>> 1; // ไม่รักษา sign bit
Console.WriteLine($"-1 >>> 1 = {unsignedShift}");

// ============ Flags Enum ============
[Flags]
enum Permission
{
    None    = 0b0000, // 0
    Read    = 0b0001, // 1
    Write   = 0b0010, // 2
    Execute = 0b0100, // 4
    Admin   = 0b1000, // 8
    
    // Combinations
    ReadWrite = Read | Write,           // 3
    All = Read | Write | Execute | Admin // 15
}

Permission userPerm = Permission.Read | Permission.Write;
Console.WriteLine($"\nUser permission: {userPerm}");  // Read, Write
Console.WriteLine($"Has Read: {(userPerm & Permission.Read) != 0}");     // true
Console.WriteLine($"Has Execute: {(userPerm & Permission.Execute) != 0}"); // false

// Add permission
userPerm |= Permission.Execute;
Console.WriteLine($"After adding Execute: {userPerm}"); // Read, Write, Execute

// Remove permission
userPerm &= ~Permission.Write;
Console.WriteLine($"After removing Write: {userPerm}"); // Read, Execute

// Toggle permission
userPerm ^= Permission.Admin;
Console.WriteLine($"After toggling Admin: {userPerm}"); // Read, Execute, Admin

// Check with HasFlag
Console.WriteLine($"HasFlag(Admin): {userPerm.HasFlag(Permission.Admin)}"); // true
```

---

## Step 34: Conditional Expressions

```csharp
// ============ Ternary Operator ============
// condition ? valueIfTrue : valueIfFalse

int age = 20;
string ageGroup = age >= 18 ? "ผู้ใหญ่" : "เด็ก";
Console.WriteLine(ageGroup); // ผู้ใหญ่

// Nested ternary (ควรหลีกเลี่ยง - อ่านยาก)
int score = 75;
string grade = score >= 90 ? "A" :
               score >= 80 ? "B" :
               score >= 70 ? "C" :
               score >= 60 ? "D" : "F";
Console.WriteLine($"Grade: {grade}"); // C

// ============ Null-conditional Operators ============
string? name = null;
int? length = name?.Length; // null ถ้า name เป็น null
Console.WriteLine($"Length: {length}"); // (null)

// Null-coalescing
string display = name ?? "Anonymous";
Console.WriteLine($"Display: {display}"); // Anonymous

// Null-coalescing assignment (??=)
string? title = null;
title ??= "Untitled";
Console.WriteLine($"Title: {title}"); // Untitled

title ??= "Changed"; // ไม่เปลี่ยนเพราะ title ไม่ใช่ null แล้ว
Console.WriteLine($"Title: {title}"); // Untitled

// ============ Switch Expression (C# 8+) ============
int day = 3;
string dayName = day switch
{
    1 => "จันทร์",
    2 => "อังคาร",
    3 => "พุธ",
    4 => "พฤหัสบดี",
    5 => "ศุกร์",
    6 => "เสาร์",
    7 => "อาทิตย์",
    _ => "ไม่ถูกต้อง"
};
Console.WriteLine($"Day {day}: {dayName}");

// Switch expression with guards
int temp = 35;
string weather = temp switch
{
    <= 0 => "หนาวมาก (น้ำแข็ง)",
    > 0 and <= 15 => "หนาว",
    > 15 and <= 25 => "เย็นสบาย",
    > 25 and <= 35 => "ร้อน",
    > 35 => "ร้อนมาก"
};
Console.WriteLine($"{temp}°C: {weather}");

// Switch expression with tuple
string GetTrafficLight(string color, bool isRaining) => (color, isRaining) switch
{
    ("red", _) => "หยุด",
    ("yellow", _) => "ระวัง",
    ("green", true) => "ไปช้าๆ (มีฝน)",
    ("green", false) => "ไปได้",
    _ => "ไม่ทราบ"
};

Console.WriteLine(GetTrafficLight("green", true));
Console.WriteLine(GetTrafficLight("red", false));

// ============ Pattern Matching (C# 9+) ============
object obj = 42;

// is pattern
if (obj is int n && n > 0)
    Console.WriteLine($"Positive int: {n}");

// Type pattern with when
void Describe(object item)
{
    switch (item)
    {
        case int i when i < 0:
            Console.WriteLine($"Negative integer: {i}");
            break;
        case int i when i == 0:
            Console.WriteLine("Zero");
            break;
        case int i:
            Console.WriteLine($"Positive integer: {i}");
            break;
        case string s when s.Length == 0:
            Console.WriteLine("Empty string");
            break;
        case string s:
            Console.WriteLine($"String: '{s}'");
            break;
        case null:
            Console.WriteLine("Null");
            break;
        default:
            Console.WriteLine($"Other: {item.GetType().Name}");
            break;
    }
}

Describe(42);
Describe(-5);
Describe("Hello");
Describe("");
Describe(null!);
Describe(3.14);
```

---

## Step 35: Operator Precedence

```csharp
// ============ Operator Precedence (ลำดับความสำคัญ) ============
// สูง -> ต่ำ:
// 1. Unary: !, ~, +, -, ++, --
// 2. Multiplicative: *, /, %
// 3. Additive: +, -
// 4. Shift: <<, >>
// 5. Relational: <, >, <=, >=, is, as
// 6. Equality: ==, !=
// 7. Bitwise AND: &
// 8. Bitwise XOR: ^
// 9. Bitwise OR: |
// 10. Logical AND: &&
// 11. Logical OR: ||
// 12. Null-coalescing: ??
// 13. Conditional: ?:
// 14. Assignment: =, +=, -=, ...

int result;

result = 2 + 3 * 4;        // = 14 (*, ก่อน +)
Console.WriteLine($"2 + 3 * 4 = {result}");

result = (2 + 3) * 4;      // = 20 (วงเล็บก่อน)
Console.WriteLine($"(2 + 3) * 4 = {result}");

result = 10 - 3 - 2;       // = 5 (left-to-right)
Console.WriteLine($"10 - 3 - 2 = {result}");

bool boolResult = 5 > 3 && 2 < 4 || 1 == 2;  // = true
// (5 > 3) = true, (2 < 4) = true, (1 == 2) = false
// true && true || false = true || false = true
Console.WriteLine($"5 > 3 && 2 < 4 || 1 == 2 = {boolResult}");

// ============ Associativity ============
// Left-to-right: +, -, *, /, %, <<, >>
result = 100 - 50 - 25;    // = 25 (left: (100-50)-25)
Console.WriteLine($"100 - 50 - 25 = {result}");

// Right-to-left: =, +=, ?:
int x, y, z;
x = y = z = 10;  // z=10, y=10, x=10 (right-to-left)
Console.WriteLine($"x={x}, y={y}, z={z}");

// ============ Common Mistakes ============

// ❌ ผิดพลาดบ่อย: ลำดับ bits
int bits = 1 | 2 + 3;     // = 1 | 5 = 5 (+ ก่อน |)
Console.WriteLine($"1 | 2 + 3 = {bits}");

bits = (1 | 2) + 3;       // = 3 + 3 = 6
Console.WriteLine($"(1 | 2) + 3 = {bits}");

// ❌ ผิดพลาดบ่อย: comparison chain
int n = 5;
// bool inRange = 1 < n < 10; // ❌ Compile error! ทำไม่ได้ใน C#

bool inRange = 1 < n && n < 10; // ✅
Console.WriteLine($"1 < {n} < 10: {inRange}");
```

---

## Step 36-40: String Formatting ครบถ้วน

```csharp
// ============ Format Specifiers ============

double price = 1234567.89;
DateTime now = DateTime.Now;
int num = 42;

// Standard Numeric Format
Console.WriteLine($"C (Currency): {price:C}");           // $1,234,567.89
Console.WriteLine($"C2 (2 decimal): {price:C2}");        // $1,234,567.89
Console.WriteLine($"D (Decimal int): {num:D5}");         // 00042
Console.WriteLine($"E (Scientific): {price:E2}");        // 1.23E+006
Console.WriteLine($"F (Fixed): {price:F2}");             // 1234567.89
Console.WriteLine($"G (General): {price:G}");            // 1234567.89
Console.WriteLine($"N (Number): {price:N2}");            // 1,234,567.89
Console.WriteLine($"P (Percent): {0.75:P}");             // 75.00%
Console.WriteLine($"R (Round-trip): {1.0/3.0:R}");       // 0.3333333333333333
Console.WriteLine($"X (Hex): {255:X}");                  // FF
Console.WriteLine($"x (hex lower): {255:x4}");           // 00ff

// Custom Numeric Format
Console.WriteLine($"#,##0.00: {price:#,##0.00}");       // 1,234,567.89
Console.WriteLine($"0000: {num:0000}");                  // 0042
Console.WriteLine($"##.##: {3.14:##.##}");              // 3.14

// DateTime Format
Console.WriteLine($"\nDate Formats:");
Console.WriteLine($"d (Short date): {now:d}");           // 1/15/2024
Console.WriteLine($"D (Long date): {now:D}");            // Monday, January 15, 2024
Console.WriteLine($"t (Short time): {now:t}");           // 2:30 PM
Console.WriteLine($"T (Long time): {now:T}");            // 2:30:45 PM
Console.WriteLine($"f (Full short): {now:f}");           // Monday, January 15, 2024 2:30 PM
Console.WriteLine($"F (Full long): {now:F}");            // Monday, January 15, 2024 2:30:45 PM
Console.WriteLine($"g (General short): {now:g}");        // 1/15/2024 2:30 PM
Console.WriteLine($"G (General long): {now:G}");         // 1/15/2024 2:30:45 PM
Console.WriteLine($"M (Month/day): {now:M}");            // January 15
Console.WriteLine($"R (RFC 1123): {now:R}");             // Mon, 15 Jan 2024 14:30:45 GMT
Console.WriteLine($"s (Sortable): {now:s}");             // 2024-01-15T14:30:45
Console.WriteLine($"u (Universal): {now:u}");            // 2024-01-15 14:30:45Z
Console.WriteLine($"Y (Year/month): {now:Y}");           // January 2024

// Custom DateTime Format
Console.WriteLine($"\nCustom Date:");
Console.WriteLine($"dd/MM/yyyy: {now:dd/MM/yyyy}");
Console.WriteLine($"yyyy-MM-dd HH:mm:ss: {now:yyyy-MM-dd HH:mm:ss}");
Console.WriteLine($"HH:mm:ss.fff: {now:HH:mm:ss.fff}");
Console.WriteLine($"ddd, dd MMM yyyy: {now:ddd, dd MMM yyyy}");

// Thai date format
System.Globalization.CultureInfo thaiCulture = new("th-TH");
Console.WriteLine($"Thai format: {now.ToString("D", thaiCulture)}");

// ============ StringBuilder ============
using System.Text;

StringBuilder sb = new StringBuilder();
sb.Append("Hello");
sb.Append(", ");
sb.Append("World");
sb.AppendLine("!");
sb.AppendFormat("Today is {0:D}", DateTime.Now);
sb.Insert(0, ">>> ");  // แทรกที่ต้น

string result = sb.ToString();
Console.WriteLine(result);

// StringBuilder เหมาะสำหรับการต่อ string จำนวนมาก
StringBuilder longSb = new StringBuilder(capacity: 1000);
for (int i = 0; i < 100; i++)
{
    longSb.AppendLine($"Line {i + 1}");
}
Console.WriteLine($"Generated {longSb.Length} chars");

// ============ String.Format patterns ============
// Alignment: {index,width} - ขวา (บวก), ซ้าย (ลบ)
string header = string.Format("{0,-15} {1,10} {2,8}", "Name", "Score", "Grade");
string row1   = string.Format("{0,-15} {1,10:F1} {2,8}", "Alice Smith", 95.5, "A");
string row2   = string.Format("{0,-15} {1,10:F1} {2,8}", "Bob Johnson", 82.3, "B");

Console.WriteLine(header);
Console.WriteLine(new string('-', 35));
Console.WriteLine(row1);
Console.WriteLine(row2);
```

---

## Step 41-50: Expressions ขั้นสูง

```csharp
// ============ Expression-bodied Members ============
class Circle
{
    private double _radius;
    
    public Circle(double radius) => _radius = radius;
    
    // Expression-bodied property
    public double Radius => _radius;
    public double Diameter => _radius * 2;
    public double Area => Math.PI * _radius * _radius;
    public double Circumference => 2 * Math.PI * _radius;
    
    // Expression-bodied method
    public bool Contains(double x, double y) => 
        Math.Sqrt(x * x + y * y) <= _radius;
    
    public override string ToString() => 
        $"Circle(r={_radius:F2}, A={Area:F2})";
}

var circle = new Circle(5.0);
Console.WriteLine(circle);
Console.WriteLine($"Area: {circle.Area:F4}");
Console.WriteLine($"Contains (3,4): {circle.Contains(3, 4)}");  // true (5^2 = 25)
Console.WriteLine($"Contains (4,4): {circle.Contains(4, 4)}");  // false (32 > 25)

// ============ Checked/Unchecked ============
// ป้องกัน integer overflow

int max = int.MaxValue; // 2,147,483,647
Console.WriteLine($"\nint.MaxValue: {max}");

// Unchecked (default): overflow ไม่ throw exception
int overflow = unchecked(max + 1);
Console.WriteLine($"Unchecked overflow: {overflow}"); // -2,147,483,648 (wraps around)

// Checked: throw OverflowException
try
{
    int checkedOverflow = checked(max + 1);
}
catch (OverflowException ex)
{
    Console.WriteLine($"Checked overflow: {ex.Message}");
}

// checked block
checked
{
    try
    {
        int x = int.MaxValue;
        x += 1; // OverflowException
    }
    catch (OverflowException)
    {
        Console.WriteLine("Overflow in checked block!");
    }
}

// ============ sizeof และ typeof ============
Console.WriteLine($"\nsizeof(int): {sizeof(int)} bytes");
Console.WriteLine($"sizeof(double): {sizeof(double)} bytes");
Console.WriteLine($"sizeof(bool): {sizeof(bool)} bytes");
Console.WriteLine($"sizeof(char): {sizeof(char)} bytes");
Console.WriteLine($"sizeof(long): {sizeof(long)} bytes");

Console.WriteLine($"\ntypeof(int): {typeof(int)}");
Console.WriteLine($"typeof(string): {typeof(string)}");
Console.WriteLine($"GetType(): {42.GetType()}");
Console.WriteLine($"Is int: {42 is int}");
Console.WriteLine($"Is string: {42 is string}");

// ============ nameof ============
string variableName = "Hello";
Console.WriteLine($"\nnameof(variableName): {nameof(variableName)}");
Console.WriteLine($"nameof(Console): {nameof(Console)}");
Console.WriteLine($"nameof(Math.PI): {nameof(Math.PI)}");
// ✅ ใช้ nameof เพื่อหลีกเลี่ยง magic strings

// ============ default ============
int defaultInt = default(int);       // 0
bool defaultBool = default(bool);    // false
string? defaultString = default(string); // null
double defaultDouble = default;       // 0.0 (C# 7.1+)
Console.WriteLine($"\ndefault(int): {defaultInt}");
Console.WriteLine($"default(bool): {defaultBool}");
Console.WriteLine($"default(string): {defaultString ?? "null"}");

// ============ Operator Overloading ============
struct Vector2D
{
    public double X, Y;
    
    public Vector2D(double x, double y) { X = x; Y = y; }
    
    // Overload +
    public static Vector2D operator +(Vector2D a, Vector2D b)
        => new Vector2D(a.X + b.X, a.Y + b.Y);
    
    // Overload -
    public static Vector2D operator -(Vector2D a, Vector2D b)
        => new Vector2D(a.X - b.X, a.Y - b.Y);
    
    // Overload * (scalar)
    public static Vector2D operator *(Vector2D v, double scalar)
        => new Vector2D(v.X * scalar, v.Y * scalar);
    
    // Overload == and !=
    public static bool operator ==(Vector2D a, Vector2D b)
        => a.X == b.X && a.Y == b.Y;
    public static bool operator !=(Vector2D a, Vector2D b)
        => !(a == b);
    
    public double Magnitude => Math.Sqrt(X * X + Y * Y);
    
    public override string ToString() => $"({X:F2}, {Y:F2})";
    public override bool Equals(object? obj) => obj is Vector2D v && this == v;
    public override int GetHashCode() => HashCode.Combine(X, Y);
}

var v1 = new Vector2D(3, 4);
var v2 = new Vector2D(1, 2);
var sum = v1 + v2;
var diff = v1 - v2;
var scaled = v1 * 2;

Console.WriteLine($"\nv1 = {v1}, |v1| = {v1.Magnitude}");
Console.WriteLine($"v2 = {v2}");
Console.WriteLine($"v1 + v2 = {sum}");
Console.WriteLine($"v1 - v2 = {diff}");
Console.WriteLine($"v1 * 2 = {scaled}");
Console.WriteLine($"v1 == v2: {v1 == v2}");

// ============ Implicit/Explicit Conversion Operators ============
struct Celsius
{
    public double Degrees;
    public Celsius(double degrees) => Degrees = degrees;
    
    // Implicit: Celsius -> Fahrenheit
    public static implicit operator Fahrenheit(Celsius c)
        => new Fahrenheit(c.Degrees * 9 / 5 + 32);
    
    public override string ToString() => $"{Degrees}°C";
}

struct Fahrenheit
{
    public double Degrees;
    public Fahrenheit(double degrees) => Degrees = degrees;
    
    // Explicit: Fahrenheit -> Celsius
    public static explicit operator Celsius(Fahrenheit f)
        => new Celsius((f.Degrees - 32) * 5 / 9);
    
    public override string ToString() => $"{Degrees}°F";
}

Celsius boiling = new Celsius(100);
Fahrenheit boilingF = boiling;  // Implicit conversion
Console.WriteLine($"\n{boiling} = {boilingF}");

Fahrenheit freezingF = new Fahrenheit(32);
Celsius freezingC = (Celsius)freezingF;  // Explicit conversion
Console.WriteLine($"{freezingF} = {freezingC}");
```

---

## สรุป Part 03

```
✅ Step 31: Comparison Operators (==, !=, <, >, <=, >=)
✅ Step 32: Logical Operators (&&, ||, !, ^, short-circuit)
✅ Step 33: Bitwise Operators (&, |, ^, ~, <<, >>)
✅ Step 34: Conditional Expressions (?:, ?., ??, ??=, switch)
✅ Step 35: Operator Precedence
✅ Steps 36-40: String Formatting ครบถ้วน
✅ Steps 41-50: Advanced Expressions (Operator Overloading, Conversions)
```

## แบบฝึกหัด Part 03

1. เขียนฟังก์ชัน `IsLeapYear(int year)` โดยใช้ logical operators
2. ใช้ bitwise operators สร้าง flags สำหรับ settings ของแอพ
3. เขียน `switch expression` แปลงคะแนน 0-100 เป็น เกรด A-F
4. สร้าง `struct Money` ที่ overload operators +, -, *, ==
5. เขียนโปรแกรมแสดงตาราง multiplication 10x10 ด้วย string formatting

---

**ถัดไป: [Part 04 - การควบคุมการทำงาน →](part-04-control-flow.md)**
