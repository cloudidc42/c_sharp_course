# Part 02: ตัวแปรและชนิดข้อมูล (Variables & Data Types)
## Steps 11-30: รากฐานของการเขียนโปรแกรม C#

---

## Step 11: ชนิดข้อมูลพื้นฐาน (Primitive Types)

C# เป็น **strongly-typed language** หมายความว่าทุกตัวแปรต้องมีชนิดที่กำหนดไว้

### ตารางชนิดข้อมูลครบถ้วน

```csharp
// ============ Integer Types ============
byte   b = 255;          // 0 ถึง 255 (1 byte)
sbyte  sb = -128;        // -128 ถึง 127 (1 byte, signed)
short  s = -32768;       // -32,768 ถึง 32,767 (2 bytes)
ushort us = 65535;       // 0 ถึง 65,535 (2 bytes, unsigned)
int    i = 2147483647;   // -2,147,483,648 ถึง 2,147,483,647 (4 bytes) ← ใช้บ่อยที่สุด
uint   ui = 4294967295;  // 0 ถึง 4,294,967,295 (4 bytes, unsigned)
long   l = 9223372036854775807L; // -9.2 * 10^18 ถึง 9.2 * 10^18 (8 bytes)
ulong  ul = 18446744073709551615UL; // 0 ถึง 1.8 * 10^19 (8 bytes)

// ============ Floating-point Types ============
float  f = 3.14f;        // 7 digits precision (4 bytes) - suffix: f หรือ F
double d = 3.14159265358979; // 15-16 digits precision (8 bytes) ← ใช้บ่อย
decimal dec = 3.14159265358979323846m; // 28-29 digits (16 bytes) - suffix: m หรือ M

// ============ Text Types ============
char   c = 'A';          // Unicode character (2 bytes) - ใช้ single quotes
string str = "Hello";   // Unicode string - ใช้ double quotes ← ใช้บ่อยที่สุด

// ============ Boolean ============
bool   boolVal = true;   // true หรือ false (1 byte)

// ============ Special Types ============
object obj = 42;         // Base type ของทุกชนิด
var    v = "auto type";  // Type inference - compiler กำหนดให้

Console.WriteLine($"byte: {b}, max: {byte.MaxValue}");
Console.WriteLine($"int: {i}, min: {int.MinValue}, max: {int.MaxValue}");
Console.WriteLine($"long: {l}");
Console.WriteLine($"float: {f:F7}");
Console.WriteLine($"double: {d:F15}");
Console.WriteLine($"decimal: {dec:F20}");
Console.WriteLine($"char: {c}, Unicode: {(int)c}");
Console.WriteLine($"string: {str}, length: {str.Length}");
Console.WriteLine($"bool: {boolVal}");
```

---

## Step 12: การประกาศตัวแปร (Variable Declaration)

```csharp
// ============ วิธีประกาศตัวแปร ============

// 1. ประกาศพร้อมกำหนดค่า (Declaration with initialization)
int age = 25;
string name = "Alice";
bool isActive = true;

// 2. ประกาศก่อน กำหนดค่าทีหลัง
int score;
score = 100;

// 3. var - Type Inference (Compiler กำหนดชนิดให้)
var message = "Hello";     // string
var number = 42;           // int
var price = 9.99;          // double
var flag = false;          // bool

// ✅ var ถูกกำหนดชนิดเมื่อ compile ไม่ใช่ runtime
// ❌ var x; // Error! ต้องกำหนดค่าทันที

// 4. Multiple declarations
int x = 1, y = 2, z = 3;

// 5. Constants
const double PI = 3.14159265358979;
const string APP_NAME = "MyCSharpApp";
const int MAX_SIZE = 1000;

// ✅ const ต้องกำหนดค่าตอนประกาศ และเปลี่ยนแปลงไม่ได้
// ❌ PI = 3.14; // Error!

// 6. readonly - กำหนดค่าได้แค่ใน constructor
class Config
{
    public readonly string ConnectionString;
    public readonly int MaxRetries = 3;
    
    public Config(string connStr)
    {
        ConnectionString = connStr; // ✅ กำหนดค่าใน constructor ได้
    }
}

// 7. static readonly
public static class AppSettings
{
    public static readonly string Version = "1.0.0";
    public static readonly DateTime StartTime = DateTime.Now;
}

// แสดงผล
Console.WriteLine($"Age: {age}");
Console.WriteLine($"Name: {name}");
Console.WriteLine($"Message (var): {message}, Type: {message.GetType().Name}");
Console.WriteLine($"PI: {PI}");
Console.WriteLine($"Version: {AppSettings.Version}");
```

---

## Step 13: Numeric Literals และ Operators

```csharp
// ============ Numeric Literals ============

// Decimal (ฐาน 10)
int decimal_num = 1_000_000; // _ เป็น separator ทำให้อ่านง่าย
long bigNum = 9_223_372_036_854_775_807L;

// Hexadecimal (ฐาน 16) - prefix: 0x
int hex = 0xFF;        // = 255
int hexColor = 0xFF5733; // = สี RGB
Console.WriteLine($"0xFF = {hex}");
Console.WriteLine($"0xFF5733 = {hexColor}");

// Binary (ฐาน 2) - prefix: 0b
int binary = 0b1010_1010; // = 170
int flags = 0b0000_1111;   // = 15
Console.WriteLine($"0b10101010 = {binary}");

// Octal ไม่มีใน C# โดยตรง แต่ทำได้ผ่าน Convert
int octal = Convert.ToInt32("777", 8); // = 511

// Infinity และ NaN
double posInf = double.PositiveInfinity;
double negInf = double.NegativeInfinity;
double nan = double.NaN;
Console.WriteLine($"Infinity: {posInf}");
Console.WriteLine($"NaN: {nan}");
Console.WriteLine($"IsNaN: {double.IsNaN(nan)}");

// ============ Arithmetic Operators ============
int a = 10, b = 3;

int sum = a + b;         // 13
int diff = a - b;        // 7
int product = a * b;     // 30
int quotient = a / b;    // 3 (integer division)
int remainder = a % b;   // 1 (modulo)
double divResult = (double)a / b; // 3.333...

Console.WriteLine($"{a} + {b} = {sum}");
Console.WriteLine($"{a} - {b} = {diff}");
Console.WriteLine($"{a} * {b} = {product}");
Console.WriteLine($"{a} / {b} = {quotient} (integer)");
Console.WriteLine($"{a} % {b} = {remainder}");
Console.WriteLine($"{a} / {b} = {divResult:F3} (double)");

// Power (Math.Pow)
double power = Math.Pow(2, 10); // 2^10 = 1024
Console.WriteLine($"2^10 = {power}");

// Increment/Decrement
int count = 0;
count++;    // post-increment: เพิ่มหลังใช้ค่า
++count;    // pre-increment: เพิ่มก่อนใช้ค่า
count--;    // post-decrement
--count;    // pre-decrement

int x = 5;
int postInc = x++;  // postInc = 5, x = 6
int preInc = ++x;   // preInc = 7, x = 7
Console.WriteLine($"After post++: x={x}, got={postInc}");
Console.WriteLine($"After pre++: x={x}, got={preInc}");

// Assignment Operators
int val = 100;
val += 10;  // val = val + 10 = 110
val -= 5;   // val = val - 5 = 105
val *= 2;   // val = val * 2 = 210
val /= 3;   // val = val / 3 = 70
val %= 9;   // val = val % 9 = 7
Console.WriteLine($"Final val: {val}");

// Math operations
Console.WriteLine($"Abs(-5): {Math.Abs(-5)}");
Console.WriteLine($"Sqrt(16): {Math.Sqrt(16)}");
Console.WriteLine($"Round(3.7): {Math.Round(3.7)}");
Console.WriteLine($"Floor(3.7): {Math.Floor(3.7)}");
Console.WriteLine($"Ceiling(3.2): {Math.Ceiling(3.2)}");
Console.WriteLine($"Max(10,20): {Math.Max(10, 20)}");
Console.WriteLine($"Min(10,20): {Math.Min(10, 20)}");
```

---

## Step 14: String ชนิดข้อมูลสำคัญ

```csharp
// ============ String Declaration ============
string s1 = "Hello, World!";
string s2 = 'A'.ToString();  // char to string

// Empty string
string empty1 = "";
string empty2 = string.Empty; // แนะนำกว่า

// Null string
string? nullStr = null;

// ============ String Interpolation ($) ============
string name = "Alice";
int age = 30;
string greeting = $"สวัสดี, {name}! อายุ {age} ปี";
string formatted = $"Pi = {Math.PI:F4}";  // Pi = 3.1416
string dateStr = $"วันนี้: {DateTime.Now:dd/MM/yyyy}";
Console.WriteLine(greeting);
Console.WriteLine(formatted);
Console.WriteLine(dateStr);

// ============ Verbatim String (@) ============
string path = @"C:\Users\Alice\Documents\file.txt"; // ไม่ต้อง escape backslash
string multiLine = @"Line 1
Line 2
Line 3";

// ============ Raw String Literals (C# 11+) ============
string rawJson = """
    {
        "name": "Alice",
        "age": 30
    }
    """;
Console.WriteLine(rawJson);

string html = """
    <html>
        <body>
            <p>Hello</p>
        </body>
    </html>
    """;

// ============ String Operations ============
string text = "  Hello, World!  ";

// Length
Console.WriteLine($"Length: {text.Length}"); // 17

// Trim
Console.WriteLine($"Trim: '{text.Trim()}'");      // 'Hello, World!'
Console.WriteLine($"TrimStart: '{text.TrimStart()}'"); // 'Hello, World!  '
Console.WriteLine($"TrimEnd: '{text.TrimEnd()}'");   // '  Hello, World!'

// Case
string upper = text.Trim().ToUpper(); // HELLO, WORLD!
string lower = text.Trim().ToLower(); // hello, world!
Console.WriteLine($"Upper: {upper}");
Console.WriteLine($"Lower: {lower}");

// Contains, StartsWith, EndsWith
string sentence = "The quick brown fox";
Console.WriteLine($"Contains 'quick': {sentence.Contains("quick")}");    // True
Console.WriteLine($"StartsWith 'The': {sentence.StartsWith("The")}");    // True
Console.WriteLine($"EndsWith 'fox': {sentence.EndsWith("fox")}");        // True

// IndexOf, LastIndexOf
Console.WriteLine($"IndexOf 'o': {sentence.IndexOf('o')}");              // 11
Console.WriteLine($"LastIndexOf 'o': {sentence.LastIndexOf('o')}");      // 17

// Substring
string sub = sentence.Substring(4, 5); // "quick"
Console.WriteLine($"Substring(4,5): {sub}");

// Range (C# 8+)
string rangeStr = sentence[4..9]; // "quick"
string fromEnd = sentence[^3..];  // "fox"
Console.WriteLine($"Range [4..9]: {rangeStr}");
Console.WriteLine($"Range [^3..]: {fromEnd}");

// Replace
string replaced = sentence.Replace("fox", "cat");
Console.WriteLine($"Replace: {replaced}");

// Split
string csv = "apple,banana,cherry,date";
string[] fruits = csv.Split(',');
foreach (string fruit in fruits)
    Console.Write($"{fruit} | ");
Console.WriteLine();

// Join
string joined = string.Join(" - ", fruits);
Console.WriteLine($"Join: {joined}");

// Concat vs +
string c1 = string.Concat("Hello", " ", "World"); // "Hello World"
string c2 = "Hello" + " " + "World"; // เหมือนกัน แต่ Concat มักเร็วกว่า

// Repeat (C# .NET way)
string repeated = string.Concat(Enumerable.Repeat("abc", 3)); // "abcabcabc"
// หรือ
string repeated2 = new string('-', 20); // "--------------------"

// Format
string formatted2 = string.Format("Name: {0}, Age: {1}", name, age);
Console.WriteLine(formatted2);
```

---

## Step 15: char และ Unicode

```csharp
// ============ char ============
char letter = 'A';
char digit = '5';
char newline = '\n';
char tab = '\t';
char backslash = '\\';
char singleQuote = '\'';
char nullChar = '\0';

// char arithmetic
char next = (char)(letter + 1); // 'B'
Console.WriteLine($"A + 1 = {next}");

// char methods
Console.WriteLine($"IsLetter: {char.IsLetter('A')}");    // True
Console.WriteLine($"IsDigit: {char.IsDigit('5')}");      // True
Console.WriteLine($"IsWhiteSpace: {char.IsWhiteSpace(' ')}"); // True
Console.WriteLine($"IsUpper: {char.IsUpper('A')}");      // True
Console.WriteLine($"IsLower: {char.IsLower('a')}");      // True
Console.WriteLine($"ToUpper: {char.ToUpper('a')}");      // A
Console.WriteLine($"ToLower: {char.ToLower('A')}");      // a

// ============ Unicode ============
char thaiChar = 'ก';
Console.WriteLine($"Thai char: {thaiChar}, Code: {(int)thaiChar}"); // 3585

// Emoji (ต้องใช้ string เพราะเป็น surrogate pair)
string emoji = "😀";
Console.WriteLine($"Emoji: {emoji}, Length: {emoji.Length}"); // Length = 2!

// char array to string
char[] chars = { 'H', 'e', 'l', 'l', 'o' };
string word = new string(chars);
Console.WriteLine(word); // Hello

// string to char array
char[] charArray = "Hello".ToCharArray();
Console.WriteLine($"First char: {charArray[0]}"); // H

// ============ Escape Characters ============
Console.WriteLine("Tab:\tEnd");
Console.WriteLine("Newline:\nSecond line");
Console.WriteLine("Quote: \"quoted\"");
Console.WriteLine("Backslash: C:\\Windows");
Console.WriteLine("Null char test: 'A\0B' (null between A and B)");
```

---

## Step 16: Type Conversion

```csharp
// ============ Implicit Conversion (ปลอดภัย, อัตโนมัติ) ============
int intVal = 42;
long longVal = intVal;     // int -> long (ขยายขนาด)
double doubleVal = intVal; // int -> double
float floatVal = intVal;   // int -> float

Console.WriteLine($"int to long: {longVal}");
Console.WriteLine($"int to double: {doubleVal}");

// ============ Explicit Conversion / Cast (อาจสูญเสียข้อมูล) ============
double pi = 3.14159;
int piInt = (int)pi;       // ตัดทศนิยมออก = 3 (ไม่ปัดขึ้น!)
Console.WriteLine($"double to int: {piInt}");

long bigLong = 10_000_000_000L;
int truncated = (int)bigLong; // Overflow! ค่าจะผิด
Console.WriteLine($"long to int (overflow): {truncated}");

// ============ Convert Class (ปลอดภัยกว่า) ============
string numStr = "42";
int converted = Convert.ToInt32(numStr);
double convertedDouble = Convert.ToDouble("3.14");
bool convertedBool = Convert.ToBoolean(1); // true
string convertedStr = Convert.ToString(12345);

Console.WriteLine($"Convert.ToInt32: {converted}");
Console.WriteLine($"Convert.ToDouble: {convertedDouble}");
Console.WriteLine($"Convert.ToBoolean(1): {convertedBool}");

// Convert ทุกชนิด
Console.WriteLine($"ToByte: {Convert.ToByte(255)}");
Console.WriteLine($"ToInt16: {Convert.ToInt16(32767)}");
Console.WriteLine($"ToInt64: {Convert.ToInt64(9999999999)}");
Console.WriteLine($"ToDecimal: {Convert.ToDecimal(9.99)}");

// ============ Parse Methods ============
int parsed = int.Parse("123");
double parsedDouble = double.Parse("3.14");
bool parsedBool = bool.Parse("true");
DateTime parsedDate = DateTime.Parse("2024-01-15");

Console.WriteLine($"int.Parse: {parsed}");
Console.WriteLine($"DateTime.Parse: {parsedDate:dd/MM/yyyy}");

// ============ TryParse (ปลอดภัย, ไม่ throw exception) ============
string input = "abc123";
if (int.TryParse(input, out int result))
{
    Console.WriteLine($"Parsed: {result}");
}
else
{
    Console.WriteLine($"'{input}' ไม่ใช่ตัวเลข");
}

// TryParse กับ nullable
if (double.TryParse("3.14", out double dResult))
{
    Console.WriteLine($"Double parsed: {dResult}");
}

// ============ Boxing and Unboxing ============
// Boxing: value type -> object (heap)
int boxInt = 42;
object boxed = boxInt;  // Boxing
Console.WriteLine($"Boxed: {boxed}, Type: {boxed.GetType()}");

// Unboxing: object -> value type
int unboxed = (int)boxed;  // Unboxing
Console.WriteLine($"Unboxed: {unboxed}");

// ✅ Boxing/Unboxing มีค่าใช้จ่าย performance ควรหลีกเลี่ยงในงาน performance-critical
```

---

## Step 17: Nullable Types

```csharp
// ============ Nullable Value Types ============
// เพิ่ม ? หลังชนิดข้อมูล ทำให้รับค่า null ได้

int? nullableInt = null;
double? nullableDouble = 3.14;
bool? nullableBool = null;

Console.WriteLine($"Nullable int: {nullableInt}");           // (empty)
Console.WriteLine($"Nullable double: {nullableDouble}");     // 3.14
Console.WriteLine($"Has value: {nullableDouble.HasValue}");  // True
Console.WriteLine($"Value: {nullableDouble.Value}");         // 3.14

// GetValueOrDefault
int defaultVal = nullableInt.GetValueOrDefault();    // 0
int customDefault = nullableInt.GetValueOrDefault(99); // 99
Console.WriteLine($"Default: {defaultVal}");
Console.WriteLine($"Custom default: {customDefault}");

// Null-coalescing operator (??)
int result = nullableInt ?? -1;  // -1 ถ้า null
Console.WriteLine($"?? result: {result}");

string? nullStr = null;
string value = nullStr ?? "default value";
Console.WriteLine($"String ?? : {value}");

// Null-coalescing assignment (??=)
nullableInt ??= 42; // กำหนดค่าเฉพาะเมื่อ null
Console.WriteLine($"After ??= : {nullableInt}");

// ============ Nullable Reference Types (C# 8+) ============
// เปิดใช้ด้วย <Nullable>enable</Nullable> ใน .csproj

string nonNullable = "Hello";     // ❌ ไม่สามารถเป็น null
string? nullable = null;           // ✅ สามารถเป็น null

// ถ้าพยายาม assign null ให้ non-nullable จะมี warning
// nonNullable = null; // ⚠️ Warning CS8600

// Null-conditional operator (?.)
string? name = null;
int? length = name?.Length; // null ถ้า name เป็น null (ไม่ throw exception)
Console.WriteLine($"Nullable length: {length}");

// Null-conditional + null-coalescing
int safeLength = name?.Length ?? 0;
Console.WriteLine($"Safe length: {safeLength}"); // 0

// ?. chain
string? city = null;
string? upper = city?.Trim()?.ToUpper()?.Replace(" ", "_");
Console.WriteLine($"Chain ?.: {upper}"); // (null)

// ============ Null check patterns ============
object? obj = null;

// Pattern 1: is null check
if (obj is null)
    Console.WriteLine("obj is null");

// Pattern 2: is not null
if (obj is not null)
    Console.WriteLine("obj has value");

// Pattern 3: Null-forgiving operator (!)
string definitelyNotNull = nullable!; // บอก compiler ว่าไม่ null (ระวัง NullReferenceException)
```

---

## Step 18: Value Types vs Reference Types

```csharp
// ============ Value Types ============
// เก็บค่าใน Stack โดยตรง
// เมื่อ assign จะ copy ค่า

int a = 10;
int b = a;  // Copy ค่า
b = 20;     // เปลี่ยน b ไม่กระทบ a
Console.WriteLine($"a={a}, b={b}"); // a=10, b=20

// Value Types:
// - int, long, float, double, decimal
// - bool, char, byte
// - struct
// - enum

// ============ Reference Types ============
// เก็บ reference (pointer) ใน Stack
// ค่าจริงอยู่ใน Heap
// เมื่อ assign จะ copy reference (ชี้ไปที่เดียวกัน)

int[] arr1 = { 1, 2, 3 };
int[] arr2 = arr1;  // Copy reference! ชี้ที่เดียวกัน
arr2[0] = 99;       // เปลี่ยน arr2 กระทบ arr1!
Console.WriteLine($"arr1[0] = {arr1[0]}"); // 99!

// Reference Types:
// - class, interface, delegate
// - string (พิเศษ: immutable)
// - array
// - object

// ============ String เป็นพิเศษ (Immutable Reference Type) ============
string str1 = "Hello";
string str2 = str1;  // Copy reference
str2 = "World";      // สร้าง string ใหม่! ไม่กระทบ str1
Console.WriteLine($"str1={str1}, str2={str2}"); // Hello, World

// ============ Struct (Value Type) ============
struct Point
{
    public int X;
    public int Y;
    
    public Point(int x, int y) { X = x; Y = y; }
    public override string ToString() => $"({X}, {Y})";
}

Point p1 = new Point(1, 2);
Point p2 = p1;  // Copy ค่าทั้งหมด
p2.X = 99;      // ไม่กระทบ p1
Console.WriteLine($"p1={p1}, p2={p2}"); // (1,2), (99,2)

// ============ Class (Reference Type) ============
class Rectangle
{
    public int Width;
    public int Height;
    
    public Rectangle(int w, int h) { Width = w; Height = h; }
    public override string ToString() => $"{Width}x{Height}";
}

Rectangle r1 = new Rectangle(10, 20);
Rectangle r2 = r1;  // Copy reference
r2.Width = 99;      // กระทบ r1!
Console.WriteLine($"r1={r1}, r2={r2}"); // 99x20, 99x20

// Deep copy ต้องทำเอง
Rectangle r3 = new Rectangle(r1.Width, r1.Height);
r3.Width = 50;
Console.WriteLine($"r1={r1}, r3={r3}"); // 99x20, 50x20
```

---

## Step 19: var, dynamic, และ object

```csharp
// ============ var (Compile-time type inference) ============
var intVar = 42;           // int
var strVar = "Hello";      // string
var doubleVar = 3.14;      // double
var listVar = new List<int>(); // List<int>

// var เป็น static typing - ชนิดถูกกำหนดตอน compile
// ❌ intVar = "text"; // Error! Cannot convert string to int

// ✅ ใช้ var เมื่อชนิดข้อมูลชัดเจนจากการกำหนดค่า
var customer = new Customer { Name = "Alice", Age = 30 };
var results = from x in Enumerable.Range(1, 10) where x % 2 == 0 select x;

// ============ dynamic (Runtime type) ============
dynamic dyn = 42;
Console.WriteLine($"dynamic int: {dyn}");

dyn = "Hello";  // เปลี่ยนชนิดได้!
Console.WriteLine($"dynamic string: {dyn}");

dyn = 3.14;
Console.WriteLine($"dynamic double: {dyn}");

// dynamic ใช้ DLR (Dynamic Language Runtime)
// ✅ ใช้กับ COM interop, reflection, JSON dynamic
// ❌ ไม่ type-safe, ช้ากว่า static typing

dynamic obj = GetDynamicObject();
// ไม่รู้ชนิดตอน compile แต่ใช้งานได้
// Console.WriteLine(obj.Name); // อาจ throw exception ถ้าไม่มี property Name

// ============ object (Base type ทุกอย่าง) ============
object objInt = 42;       // Boxing
object objStr = "Hello";  // Reference
object objArr = new int[] { 1, 2, 3 };

// ต้อง cast ก่อนใช้
int unboxedInt = (int)objInt;
string unboxedStr = (string)objStr;
Console.WriteLine($"object int: {unboxedInt}");
Console.WriteLine($"object string: {unboxedStr}");

// is operator - type check
if (objStr is string s)
    Console.WriteLine($"It's a string: {s}");

// as operator - safe cast (return null ถ้าล้มเหลว)
string? safeCast = objStr as string;
int[]? safeArr = objArr as int[];
Console.WriteLine($"Safe cast: {safeCast}");
Console.WriteLine($"Safe array: {safeArr?.Length}");

// Pattern matching (C# 7+)
void Describe(object item)
{
    string desc = item switch
    {
        int i => $"Integer: {i}",
        string s => $"String: {s} (length {s.Length})",
        double d => $"Double: {d:F2}",
        bool b => $"Boolean: {b}",
        null => "Null value",
        _ => $"Unknown: {item.GetType().Name}"
    };
    Console.WriteLine(desc);
}

Describe(42);
Describe("Hello");
Describe(3.14);
Describe(true);
Describe(null!);
Describe(DateTime.Now);

object GetDynamicObject() => new { Name = "Test", Value = 42 };

class Customer
{
    public string Name { get; set; } = "";
    public int Age { get; set; }
}
```

---

## Step 20: Tuples และ Anonymous Types

```csharp
// ============ Tuples (C# 7+) ============

// 1. ValueTuple - ประหยัด memory
(int X, int Y) point = (10, 20);
Console.WriteLine($"X={point.X}, Y={point.Y}");

// 2. Named Tuple
var person = (Name: "Alice", Age: 30, IsActive: true);
Console.WriteLine($"{person.Name}, {person.Age}, {person.IsActive}");

// 3. Unnamed Tuple
var tuple = (1, "Hello", 3.14);
Console.WriteLine($"Item1={tuple.Item1}, Item2={tuple.Item2}");

// 4. Tuple Deconstruction
var (name, age, active) = person;
Console.WriteLine($"Deconstructed: {name}, {age}, {active}");

// 5. Swap using Tuple
int a = 1, b = 2;
(a, b) = (b, a); // Swap!
Console.WriteLine($"After swap: a={a}, b={b}");

// 6. Return multiple values from method
(string First, string Last) GetName()
{
    return ("John", "Doe");
}

var (first, last) = GetName();
Console.WriteLine($"Name: {first} {last}");

// 7. _ เพื่อ discard
var (only, _) = GetName();
Console.WriteLine($"First only: {only}");

// ============ Anonymous Types ============
// ใช้กับ LINQ เป็นหลัก, immutable

var anon = new { Name = "Alice", Age = 30, Score = 95.5 };
Console.WriteLine($"{anon.Name}, {anon.Age}, {anon.Score}");

// ❌ anon.Name = "Bob"; // Error! Immutable

// Anonymous type ใน collection
var students = new[]
{
    new { Name = "Alice", Score = 95 },
    new { Name = "Bob", Score = 87 },
    new { Name = "Charlie", Score = 92 }
};

foreach (var student in students)
{
    Console.WriteLine($"{student.Name}: {student.Score}");
}

// เรียงลำดับ
var sorted = students.OrderByDescending(s => s.Score);
foreach (var s in sorted)
    Console.WriteLine($"  {s.Name}: {s.Score}");

// ============ Record Types (C# 9+) ============
// ดีกว่า anonymous type - มีชื่อ, ใช้ซ้ำได้
record Point2D(int X, int Y);
record Person2(string Name, int Age);

var p = new Point2D(10, 20);
var alice = new Person2("Alice", 30);

Console.WriteLine(p);        // Point2D { X = 10, Y = 20 }
Console.WriteLine(alice);    // Person2 { Name = Alice, Age = 30 }

// Records เปรียบเทียบด้วยค่า (Value equality)
var p1 = new Point2D(10, 20);
var p2 = new Point2D(10, 20);
Console.WriteLine($"p1 == p2: {p1 == p2}"); // True!

// with expression - copy with modification
var p3 = p1 with { X = 99 };
Console.WriteLine($"p3: {p3}"); // Point2D { X = 99, Y = 20 }
Console.WriteLine($"p1 unchanged: {p1}"); // Point2D { X = 10, Y = 20 }
```

---

## Step 21-30: การฝึกปฏิบัติจริง - Calculator Program

```csharp
// ============ โปรแกรมเครื่องคิดเลขครบถ้วน ============
// แสดงการใช้ variables, types, conversion, และ operators

using System;

// ============ Banner ============
Console.ForegroundColor = ConsoleColor.Cyan;
Console.WriteLine("╔═══════════════════════════════════╗");
Console.WriteLine("║    C# Calculator v1.0 (.NET 9)    ║");
Console.WriteLine("╚═══════════════════════════════════╝");
Console.ResetColor();

bool continueCalc = true;
while (continueCalc)
{
    Console.WriteLine();
    Console.WriteLine("เลือกประเภทการคำนวณ:");
    Console.WriteLine("1. การคำนวณพื้นฐาน (+, -, *, /)");
    Console.WriteLine("2. การคำนวณขั้นสูง (%, ^, √)");
    Console.WriteLine("3. แปลงเลข (Decimal ↔ Binary ↔ Hex)");
    Console.WriteLine("4. ตรวจสอบชนิดข้อมูล");
    Console.WriteLine("0. ออก");
    Console.Write("เลือก: ");
    
    string choice = Console.ReadLine() ?? "0";
    
    switch (choice)
    {
        case "1":
            BasicCalculator();
            break;
        case "2":
            AdvancedCalculator();
            break;
        case "3":
            NumberBaseConverter();
            break;
        case "4":
            DataTypeChecker();
            break;
        case "0":
            continueCalc = false;
            Console.WriteLine("ขอบคุณที่ใช้งาน!");
            break;
        default:
            Console.ForegroundColor = ConsoleColor.Red;
            Console.WriteLine("ตัวเลือกไม่ถูกต้อง");
            Console.ResetColor();
            break;
    }
}

// ============ Functions ============

void BasicCalculator()
{
    Console.WriteLine("\n--- การคำนวณพื้นฐาน ---");
    
    double num1 = ReadDouble("ใส่ตัวเลขที่ 1: ");
    
    Console.Write("เลือกตัวดำเนินการ (+, -, *, /): ");
    string op = Console.ReadLine() ?? "+";
    
    double num2 = ReadDouble("ใส่ตัวเลขที่ 2: ");
    
    double result = op switch
    {
        "+" => num1 + num2,
        "-" => num1 - num2,
        "*" => num1 * num2,
        "/" when num2 != 0 => num1 / num2,
        "/" => throw new DivideByZeroException("ไม่สามารถหารด้วยศูนย์!"),
        _ => throw new ArgumentException($"ตัวดำเนินการ '{op}' ไม่รองรับ")
    };
    
    Console.ForegroundColor = ConsoleColor.Green;
    Console.WriteLine($"\n✅ {num1} {op} {num2} = {result:G}");
    Console.ResetColor();
}

void AdvancedCalculator()
{
    Console.WriteLine("\n--- การคำนวณขั้นสูง ---");
    Console.WriteLine("1. Modulo (%)");
    Console.WriteLine("2. Power (^)");
    Console.WriteLine("3. Square Root (√)");
    Console.WriteLine("4. Absolute Value (|x|)");
    Console.Write("เลือก: ");
    
    string option = Console.ReadLine() ?? "1";
    
    switch (option)
    {
        case "1":
            double a = ReadDouble("ตัวตั้ง: ");
            double b = ReadDouble("ตัวหาร: ");
            Console.WriteLine($"{a} % {b} = {a % b}");
            break;
        case "2":
            double baseNum = ReadDouble("ฐาน: ");
            double exp = ReadDouble("เลขชี้กำลัง: ");
            Console.WriteLine($"{baseNum}^{exp} = {Math.Pow(baseNum, exp)}");
            break;
        case "3":
            double num = ReadDouble("ตัวเลข: ");
            if (num < 0)
                Console.WriteLine($"ไม่สามารถหา √ ของจำนวนลบได้");
            else
                Console.WriteLine($"√{num} = {Math.Sqrt(num):F6}");
            break;
        case "4":
            double absNum = ReadDouble("ตัวเลข: ");
            Console.WriteLine($"|{absNum}| = {Math.Abs(absNum)}");
            break;
    }
}

void NumberBaseConverter()
{
    Console.WriteLine("\n--- แปลงฐานเลข ---");
    int decNum = (int)ReadDouble("ใส่ตัวเลขฐาน 10 (0-1023): ");
    
    Console.ForegroundColor = ConsoleColor.Yellow;
    Console.WriteLine($"\nDecimal: {decNum}");
    Console.WriteLine($"Binary:  {Convert.ToString(decNum, 2).PadLeft(10, '0')}");
    Console.WriteLine($"Octal:   {Convert.ToString(decNum, 8)}");
    Console.WriteLine($"Hex:     0x{Convert.ToString(decNum, 16).ToUpper()}");
    Console.ResetColor();
    
    // แสดง bit representation
    Console.WriteLine("\nBit Pattern:");
    string binary = Convert.ToString(decNum, 2).PadLeft(16, '0');
    for (int i = 0; i < 16; i += 4)
    {
        Console.Write(binary.Substring(i, 4) + " ");
    }
    Console.WriteLine();
}

void DataTypeChecker()
{
    Console.WriteLine("\n--- ตรวจสอบชนิดข้อมูล ---");
    Console.Write("ใส่ค่าที่ต้องการตรวจสอบ: ");
    string input = Console.ReadLine() ?? "";
    
    Console.WriteLine($"\nค่า: '{input}'");
    Console.WriteLine($"ชนิด string: ✅ (ทุกอย่างเป็น string ก่อน)");
    Console.WriteLine($"ยาว: {input.Length} ตัวอักษร");
    
    if (int.TryParse(input, out int intResult))
        Console.WriteLine($"ชนิด int: ✅ ({intResult})");
    
    if (long.TryParse(input, out long longResult))
        Console.WriteLine($"ชนิด long: ✅ ({longResult})");
    
    if (double.TryParse(input, out double doubleResult))
        Console.WriteLine($"ชนิด double: ✅ ({doubleResult:F6})");
    
    if (bool.TryParse(input, out bool boolResult))
        Console.WriteLine($"ชนิด bool: ✅ ({boolResult})");
    
    if (DateTime.TryParse(input, out DateTime dateResult))
        Console.WriteLine($"ชนิด DateTime: ✅ ({dateResult:dd/MM/yyyy HH:mm:ss})");
    
    if (Guid.TryParse(input, out Guid guidResult))
        Console.WriteLine($"ชนิด Guid: ✅ ({guidResult})");
}

double ReadDouble(string prompt)
{
    while (true)
    {
        Console.Write(prompt);
        string input = Console.ReadLine() ?? "";
        if (double.TryParse(input, out double result))
            return result;
        Console.ForegroundColor = ConsoleColor.Red;
        Console.WriteLine("กรุณาใส่ตัวเลขที่ถูกต้อง");
        Console.ResetColor();
    }
}
```

---

## สรุป Part 02

```
✅ Step 11: ชนิดข้อมูลพื้นฐานทั้งหมด (int, double, string, bool, ฯลฯ)
✅ Step 12: การประกาศตัวแปร (var, const, readonly)
✅ Step 13: Numeric Literals และ Arithmetic Operators
✅ Step 14: String operations ครบถ้วน
✅ Step 15: char และ Unicode
✅ Step 16: Type Conversion (Implicit, Explicit, Convert, Parse, TryParse)
✅ Step 17: Nullable Types และ null operators
✅ Step 18: Value Types vs Reference Types
✅ Step 19: var, dynamic, object
✅ Step 20: Tuples, Anonymous Types, Records
✅ Steps 21-30: โปรแกรม Calculator จริง
```

## แบบฝึกหัด Part 02

1. สร้างโปรแกรมรับข้อมูลส่วนตัว (ชื่อ, อายุ, เงินเดือน) และแสดงผลแบบ formatted
2. เขียนโปรแกรมแปลง Celsius ↔ Fahrenheit ↔ Kelvin
3. สร้าง Tuple function ที่คืนค่า min, max, average ของ array
4. ทดสอบ overflow ของ int และ handle ด้วย checked keyword
5. สร้างโปรแกรม BMI Calculator ที่รับ น้ำหนัก และ ส่วนสูง

---

**ถัดไป: [Part 03 - ตัวดำเนินการและนิพจน์ →](part-03-operators-expressions.md)**
