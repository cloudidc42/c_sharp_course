# Part 04: การควบคุมการทำงาน (Control Flow)
## Steps 51-80: if, switch, loops, และการจัดการโปรแกรม

---

## Step 51: if-else Statements

```csharp
// ============ Basic if-else ============
int age = 20;

// Simple if
if (age >= 18)
{
    Console.WriteLine("ผู้ใหญ่");
}

// if-else
if (age >= 18)
    Console.WriteLine("ผู้ใหญ่"); // ถ้า 1 บรรทัดไม่ต้องใช้ {}
else
    Console.WriteLine("ไม่ถึง 18");

// if-else if-else
int score = 85;
string grade;
if (score >= 90)
    grade = "A";
else if (score >= 80)
    grade = "B";
else if (score >= 70)
    grade = "C";
else if (score >= 60)
    grade = "D";
else
    grade = "F";

Console.WriteLine($"Score: {score}, Grade: {grade}");

// ============ Nested if ============
bool isLoggedIn = true;
bool isAdmin = false;
bool hasPermission = true;

if (isLoggedIn)
{
    if (isAdmin)
    {
        Console.WriteLine("Admin access: Full");
    }
    else if (hasPermission)
    {
        Console.WriteLine("User access: Limited");
    }
    else
    {
        Console.WriteLine("User access: Read-only");
    }
}
else
{
    Console.WriteLine("Not logged in");
}

// ============ Complex Conditions ============
int temperature = 28;
bool isRaining = false;
bool isWeekend = true;

if (temperature >= 25 && !isRaining && isWeekend)
{
    Console.WriteLine("ไปเที่ยวทะเลกันเถอะ!");
}
else if (temperature >= 25 && isRaining)
{
    Console.WriteLine("อยู่บ้านดีกว่า มีฝนตก");
}
else if (temperature < 25 && isWeekend)
{
    Console.WriteLine("อากาศดี ออกไปข้างนอกได้");
}
else
{
    Console.WriteLine("วันธรรมดา ทำงานกันเถอะ");
}

// ============ if with Pattern Matching (C# 9+) ============
object value = 42;

if (value is int n and > 0)
    Console.WriteLine($"Positive int: {n}");

if (value is int m and (> 10 and < 100))
    Console.WriteLine($"Two-digit number: {m}");

// ============ Guard Clauses (แนวทางที่ดี) ============
// แทนที่จะ nest หลายชั้น ใช้ return early

double CalculateBMI(double weightKg, double heightM)
{
    // Guard clauses - ตรวจสอบข้อผิดพลาดก่อน
    if (weightKg <= 0)
    {
        throw new ArgumentException("น้ำหนักต้องมากกว่า 0");
    }
    if (heightM <= 0)
    {
        throw new ArgumentException("ส่วนสูงต้องมากกว่า 0");
    }
    if (heightM > 3)
    {
        throw new ArgumentException("ส่วนสูงเกิน 3 เมตร");
    }
    
    return weightKg / (heightM * heightM);
}

try
{
    double bmi = CalculateBMI(70, 1.75);
    Console.WriteLine($"BMI: {bmi:F1}");
    
    string category = bmi switch
    {
        < 18.5 => "น้ำหนักน้อย",
        < 25.0 => "น้ำหนักปกติ",
        < 30.0 => "น้ำหนักเกิน",
        _ => "อ้วน"
    };
    Console.WriteLine($"หมวดหมู่: {category}");
}
catch (ArgumentException ex)
{
    Console.WriteLine($"Error: {ex.Message}");
}
```

---

## Step 52: switch Statement

```csharp
// ============ Traditional switch ============
int dayNumber = 3;
string dayName;

switch (dayNumber)
{
    case 1:
        dayName = "จันทร์";
        break;
    case 2:
        dayName = "อังคาร";
        break;
    case 3:
        dayName = "พุธ";
        break;
    case 4:
        dayName = "พฤหัสบดี";
        break;
    case 5:
        dayName = "ศุกร์";
        break;
    case 6:
    case 7: // Fall-through (หลาย case รัน code เดียวกัน)
        dayName = "วันหยุด";
        break;
    default:
        dayName = "ไม่ถูกต้อง";
        break;
}
Console.WriteLine($"Day {dayNumber}: {dayName}");

// ============ switch กับ string ============
string command = "help";
switch (command.ToLower())
{
    case "help":
    case "?":
        Console.WriteLine("แสดงความช่วยเหลือ");
        break;
    case "quit":
    case "exit":
    case "q":
        Console.WriteLine("ออกจากโปรแกรม");
        break;
    case "clear":
    case "cls":
        Console.Clear();
        break;
    default:
        Console.WriteLine($"ไม่รู้จักคำสั่ง: {command}");
        break;
}

// ============ switch Expression (C# 8+) ============
// กระชับกว่า, คืนค่าได้

string GetDayType(int day) => day switch
{
    1 or 7 => "วันหยุด",
    2 or 3 or 4 or 5 or 6 => "วันทำงาน",
    _ => throw new ArgumentOutOfRangeException($"Day {day} ไม่ถูกต้อง")
};

Console.WriteLine(GetDayType(1)); // วันหยุด
Console.WriteLine(GetDayType(3)); // วันทำงาน

// ============ switch กับ Type Pattern ============
string Classify(object obj) => obj switch
{
    int i when i < 0 => $"จำนวนเต็มลบ ({i})",
    int i when i == 0 => "ศูนย์",
    int i => $"จำนวนเต็มบวก ({i})",
    double d => $"จำนวนทศนิยม ({d:F2})",
    string s when s.Length == 0 => "สตริงว่าง",
    string s => $"สตริง: '{s}'",
    bool b => $"บูลีน: {b}",
    null => "Null",
    _ => $"ชนิดอื่น: {obj.GetType().Name}"
};

Console.WriteLine(Classify(-5));
Console.WriteLine(Classify(0));
Console.WriteLine(Classify(42));
Console.WriteLine(Classify(3.14));
Console.WriteLine(Classify("Hello"));
Console.WriteLine(Classify(""));
Console.WriteLine(Classify(true));

// ============ switch กับ Tuple Pattern ============
(bool isLoggedIn2, bool isAdmin2) user = (true, false);
string permission = user switch
{
    (false, _) => "ไม่มีสิทธิ์ (ยังไม่ล็อกอิน)",
    (true, true) => "สิทธิ์เต็ม (Admin)",
    (true, false) => "สิทธิ์จำกัด (User)",
};
Console.WriteLine($"Permission: {permission}");

// ============ switch กับ Property Pattern ============
record WeatherData(string Condition, double TempCelsius);

var weather = new WeatherData("Sunny", 32);
string advice = weather switch
{
    { Condition: "Rainy" } => "พกร่มไปด้วย",
    { Condition: "Sunny", TempCelsius: > 35 } => "ร้อนมาก ดื่มน้ำเยอะๆ",
    { Condition: "Sunny", TempCelsius: > 25 } => "อากาศดี แต่ครีมกันแดดด้วย",
    { Condition: "Cloudy" } => "อากาศเย็นสบาย",
    _ => "ตรวจสอบสภาพอากาศอีกครั้ง"
};
Console.WriteLine($"Weather: {weather.Condition}, {weather.TempCelsius}°C");
Console.WriteLine($"Advice: {advice}");
```

---

## Step 53: for Loop

```csharp
// ============ Basic for Loop ============
Console.WriteLine("For loop 1-10:");
for (int i = 1; i <= 10; i++)
{
    Console.Write($"{i} ");
}
Console.WriteLine();

// ============ Countdown ============
Console.WriteLine("\nCountdown:");
for (int i = 10; i >= 1; i--)
{
    Console.Write($"{i} ");
}
Console.WriteLine("🚀");

// ============ Step ต่างๆ ============
Console.WriteLine("\nStep 2 (even numbers):");
for (int i = 0; i <= 20; i += 2)
    Console.Write($"{i} ");
Console.WriteLine();

Console.WriteLine("\nStep 3:");
for (int i = 0; i <= 30; i += 3)
    Console.Write($"{i} ");
Console.WriteLine();

// ============ Nested for ============
Console.WriteLine("\nMultiplication Table:");
Console.Write("    ");
for (int i = 1; i <= 10; i++)
    Console.Write($"{i,4}");
Console.WriteLine();
Console.WriteLine(new string('-', 44));

for (int row = 1; row <= 10; row++)
{
    Console.Write($"{row,3}|");
    for (int col = 1; col <= 10; col++)
    {
        Console.Write($"{row * col,4}");
    }
    Console.WriteLine();
}

// ============ Triangle Pattern ============
Console.WriteLine("\nStar Triangle:");
int size = 5;
for (int row = 1; row <= size; row++)
{
    // Spaces
    for (int space = 0; space < size - row; space++)
        Console.Write(" ");
    // Stars
    for (int star = 0; star < 2 * row - 1; star++)
        Console.Write("*");
    Console.WriteLine();
}

// ============ Loop Control: break, continue ============
Console.WriteLine("\nBreak example (stop at 5):");
for (int i = 1; i <= 10; i++)
{
    if (i == 5) break;
    Console.Write($"{i} ");
}
Console.WriteLine();

Console.WriteLine("\nContinue example (skip even):");
for (int i = 1; i <= 10; i++)
{
    if (i % 2 == 0) continue;
    Console.Write($"{i} ");
}
Console.WriteLine();

// ============ Labeled break (ออกจาก nested loop) ============
// C# ไม่มี labeled break เหมือน Java แต่ใช้ goto ได้
Console.WriteLine("\nBreak outer loop:");
bool found = false;
for (int i = 0; i < 5 && !found; i++)
{
    for (int j = 0; j < 5; j++)
    {
        if (i == 2 && j == 3)
        {
            Console.WriteLine($"Found at ({i},{j})");
            found = true;
            break;
        }
    }
}

// ============ for กับ index และ length ============
string[] fruits = { "apple", "banana", "cherry", "date", "elderberry" };

// Forward
for (int i = 0; i < fruits.Length; i++)
    Console.WriteLine($"[{i}] {fruits[i]}");

// Backward
Console.WriteLine("\nReverse:");
for (int i = fruits.Length - 1; i >= 0; i--)
    Console.WriteLine($"[{i}] {fruits[i]}");

// ============ Multiple variables in for ============
for (int i = 0, j = 10; i < j; i++, j--)
{
    Console.Write($"({i},{j}) ");
}
Console.WriteLine();
```

---

## Step 54: while และ do-while Loops

```csharp
// ============ while Loop ============
// ตรวจสอบเงื่อนไขก่อน

int count = 0;
while (count < 5)
{
    Console.WriteLine($"Count: {count}");
    count++;
}

// ============ while กับ user input ============
Console.WriteLine("\nWhile with input:");
int sum = 0;
int num = 0;
int inputCount = 0;

Console.WriteLine("ใส่ตัวเลข (0 เพื่อหยุด):");
while (true)
{
    Console.Write("ตัวเลข: ");
    string input = Console.ReadLine() ?? "0";
    
    if (!int.TryParse(input, out num))
    {
        Console.WriteLine("กรุณาใส่ตัวเลขเท่านั้น");
        continue;
    }
    
    if (num == 0) break;
    
    sum += num;
    inputCount++;
    Console.WriteLine($"Running sum: {sum}");
}

if (inputCount > 0)
{
    Console.WriteLine($"Sum: {sum}");
    Console.WriteLine($"Average: {(double)sum / inputCount:F2}");
    Console.WriteLine($"Count: {inputCount}");
}

// ============ do-while Loop ============
// รัน code ก่อน แล้วค่อยตรวจเงื่อนไข (รัน อย่างน้อย 1 ครั้ง)

int n = 0;
do
{
    Console.WriteLine($"do-while: {n}");
    n++;
} while (n < 3);

// ============ do-while กับ Menu ============
int choice;
do
{
    Console.WriteLine("\n=== เมนู ===");
    Console.WriteLine("1. เพิ่มข้อมูล");
    Console.WriteLine("2. แก้ไขข้อมูล");
    Console.WriteLine("3. ลบข้อมูล");
    Console.WriteLine("4. แสดงข้อมูล");
    Console.WriteLine("0. ออก");
    Console.Write("เลือก: ");
    
    string inputChoice = Console.ReadLine() ?? "0";
    
    if (!int.TryParse(inputChoice, out choice))
    {
        Console.WriteLine("กรุณาใส่ตัวเลข");
        choice = -1; // loop again
        continue;
    }
    
    switch (choice)
    {
        case 1: Console.WriteLine("เพิ่มข้อมูล..."); break;
        case 2: Console.WriteLine("แก้ไขข้อมูล..."); break;
        case 3: Console.WriteLine("ลบข้อมูล..."); break;
        case 4: Console.WriteLine("แสดงข้อมูล..."); break;
        case 0: Console.WriteLine("ออกจากโปรแกรม"); break;
        default: Console.WriteLine("ตัวเลือกไม่ถูกต้อง"); break;
    }
} while (choice != 0);

// ============ Infinite Loops ============
// while (true) { ... } - ต้องมี break
// for (;;) { ... } - ต้องมี break

int fibonacci = 0, prev = 0, curr = 1;
Console.Write("\nFibonacci (<100): ");
while (true)
{
    if (curr > 100) break;
    Console.Write($"{curr} ");
    int temp = curr;
    curr = prev + curr;
    prev = temp;
}
Console.WriteLine();
```

---

## Step 55: foreach Loop

```csharp
// ============ foreach ============
// วน loop ผ่าน collection (array, list, etc.)

// Array
string[] colors = { "แดง", "เขียว", "น้ำเงิน", "เหลือง" };
foreach (string color in colors)
{
    Console.WriteLine($"สี: {color}");
}

// List<T>
var numbers = new List<int> { 1, 2, 3, 4, 5 };
foreach (int num in numbers)
{
    Console.Write($"{num} ");
}
Console.WriteLine();

// Dictionary
var capitals = new Dictionary<string, string>
{
    { "ไทย", "กรุงเทพฯ" },
    { "ญี่ปุ่น", "โตเกียว" },
    { "สหรัฐ", "วอชิงตัน ดี.ซี." },
    { "อังกฤษ", "ลอนดอน" }
};

foreach (var (country, capital) in capitals) // Deconstruction
{
    Console.WriteLine($"{country}: {capital}");
}

// หรือ
foreach (KeyValuePair<string, string> entry in capitals)
{
    Console.WriteLine($"{entry.Key} => {entry.Value}");
}

// ============ foreach กับ index (Enumerable.Select หรือ เก็บเอง) ============
// C# ไม่มี index ใน foreach โดยตรง

// วิธีที่ 1: ใช้ counter
int index = 0;
foreach (string color in colors)
{
    Console.WriteLine($"[{index}] {color}");
    index++;
}

// วิธีที่ 2: LINQ Select with index
foreach (var (i, color) in colors.Select((c, i) => (i, c)))
{
    Console.WriteLine($"[{i}] {color}");
}

// ============ foreach กับ IEnumerable ============
IEnumerable<int> GetEvenNumbers(int max)
{
    for (int i = 2; i <= max; i += 2)
        yield return i;
}

Console.WriteLine("\nEven numbers 2-20:");
foreach (int even in GetEvenNumbers(20))
    Console.Write($"{even} ");
Console.WriteLine();

// ============ foreach กับ String (char by char) ============
string word = "Hello, สวัสดี!";
Console.WriteLine("\nChar by char:");
foreach (char ch in word)
{
    if (char.IsLetter(ch))
        Console.Write($"{ch}.");
}
Console.WriteLine();

// ============ Nested foreach ============
int[][] matrix = 
{
    new[] { 1, 2, 3 },
    new[] { 4, 5, 6 },
    new[] { 7, 8, 9 }
};

Console.WriteLine("\nMatrix:");
foreach (int[] row in matrix)
{
    foreach (int cell in row)
        Console.Write($"{cell,3}");
    Console.WriteLine();
}
```

---

## Step 56-60: goto, labeled statements, และ Pattern Matching

```csharp
// ============ goto (ใช้น้อยมาก - ส่วนใหญ่ใช้ใน switch) ============
int counter = 0;

start:
    if (counter < 3)
    {
        Console.WriteLine($"Counter: {counter}");
        counter++;
        goto start; // ⚠️ ควรหลีกเลี่ยง ใช้ loop แทนดีกว่า
    }

// goto ใน switch (fall-through)
int num = 2;
switch (num)
{
    case 1:
    case 2:
        Console.WriteLine("One or Two");
        goto case 3; // goto next case
    case 3:
        Console.WriteLine("Three");
        break;
}

// ============ throw expressions ============
string ValidateName(string? name)
{
    return name ?? throw new ArgumentNullException(nameof(name), "ชื่อต้องไม่เป็น null");
}

try
{
    string valid = ValidateName("Alice");
    Console.WriteLine($"Valid: {valid}");
    
    string invalid = ValidateName(null);
}
catch (ArgumentNullException ex)
{
    Console.WriteLine($"Error: {ex.Message}");
}

// ============ Advanced Pattern Matching (C# 9-11) ============

// Relational patterns
static string ClassifyAge(int age) => age switch
{
    < 0 => "ไม่ถูกต้อง",
    < 13 => "เด็ก",
    < 18 => "วัยรุ่น",
    < 65 => "ผู้ใหญ่",
    >= 65 => "ผู้สูงอายุ"
};

// Logical patterns (and, or, not)
static bool IsWeekday(DayOfWeek day) =>
    day is not (DayOfWeek.Saturday or DayOfWeek.Sunday);

static bool IsValidScore(int score) =>
    score is >= 0 and <= 100;

Console.WriteLine($"Age 5: {ClassifyAge(5)}");
Console.WriteLine($"Age 15: {ClassifyAge(15)}");
Console.WriteLine($"Age 25: {ClassifyAge(25)}");
Console.WriteLine($"Age 70: {ClassifyAge(70)}");

Console.WriteLine($"Monday is weekday: {IsWeekday(DayOfWeek.Monday)}");
Console.WriteLine($"Saturday is weekday: {IsWeekday(DayOfWeek.Saturday)}");

Console.WriteLine($"Score 75 valid: {IsValidScore(75)}");
Console.WriteLine($"Score 105 valid: {IsValidScore(105)}");

// List pattern (C# 11+)
static string DescribeList(int[] list) => list switch
{
    [] => "Empty list",
    [var single] => $"Single element: {single}",
    [var first, var second] => $"Two elements: {first}, {second}",
    [var first, .., var last] => $"Starts with {first}, ends with {last}",
};

Console.WriteLine(DescribeList(new int[] { }));
Console.WriteLine(DescribeList(new int[] { 42 }));
Console.WriteLine(DescribeList(new int[] { 1, 2 }));
Console.WriteLine(DescribeList(new int[] { 1, 2, 3, 4, 5 }));
```

---

## Step 61-70: Advanced Control Flow Patterns

```csharp
// ============ Iterator Methods (yield) ============
IEnumerable<int> Fibonacci()
{
    int a = 0, b = 1;
    while (true)
    {
        yield return a;
        (a, b) = (b, a + b);
    }
}

Console.Write("Fibonacci: ");
foreach (int f in Fibonacci().Take(15))
    Console.Write($"{f} ");
Console.WriteLine();

// ============ Generator Pattern ============
IEnumerable<int> Range(int start, int end, int step = 1)
{
    if (step == 0) throw new ArgumentException("step ต้องไม่เป็น 0");
    if (step > 0)
    {
        for (int i = start; i <= end; i += step)
            yield return i;
    }
    else
    {
        for (int i = start; i >= end; i += step)
            yield return i;
    }
}

Console.Write("Range(1,10,2): ");
foreach (int n in Range(1, 10, 2))
    Console.Write($"{n} ");
Console.WriteLine();

Console.Write("Range(10,1,-2): ");
foreach (int n in Range(10, 1, -2))
    Console.Write($"{n} ");
Console.WriteLine();

// ============ LINQ-like control ============
// Take, Skip, TakeWhile, SkipWhile
int[] nums = { 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };

Console.WriteLine("\nTake(3):");
foreach (int n in nums.Take(3))
    Console.Write($"{n} ");

Console.WriteLine("\nSkip(7):");
foreach (int n in nums.Skip(7))
    Console.Write($"{n} ");

Console.WriteLine("\nTakeWhile(n < 5):");
foreach (int n in nums.TakeWhile(n => n < 5))
    Console.Write($"{n} ");

Console.WriteLine("\nSkipWhile(n < 5):");
foreach (int n in nums.SkipWhile(n => n < 5))
    Console.Write($"{n} ");
Console.WriteLine();

// ============ Parallel.For / Parallel.ForEach ============
using System.Threading;
using System.Threading.Tasks;

Console.WriteLine("\nParallel.For:");
object lockObj = new object();
int total = 0;

Parallel.For(1, 11, i =>
{
    int result = i * i;
    lock (lockObj) // thread-safe
    {
        total += result;
        Console.Write($"{i}²={result} ");
    }
});
Console.WriteLine($"\nTotal: {total}"); // 385

// ============ for loop performance tips ============
int[] largeArray = Enumerable.Range(1, 1000000).ToArray();

// Performance: cache Length
var sw = System.Diagnostics.Stopwatch.StartNew();
int len = largeArray.Length;
for (int i = 0; i < len; i++)
{
    _ = largeArray[i] * 2;
}
sw.Stop();
Console.WriteLine($"\nfor (cache Length): {sw.ElapsedMilliseconds}ms");

// LINQ approach
sw.Restart();
var results = largeArray.Select(x => x * 2).ToArray();
sw.Stop();
Console.WriteLine($"LINQ Select: {sw.ElapsedMilliseconds}ms");
```

---

## Step 71-80: โปรแกรม Quiz ครบถ้วน

```csharp
// ============ โปรแกรม Quiz Game ============
// ใช้ control flow ทุกรูปแบบ

using System;
using System.Collections.Generic;

// ============ Data ============
record Question(string Text, string[] Options, int CorrectIndex, string Explanation);

var questions = new List<Question>
{
    new("C# ถูกพัฒนาโดยบริษัทใด?",
        new[] { "Google", "Microsoft", "Apple", "Meta" }, 1,
        "Microsoft พัฒนา C# ในปี 2000 โดย Anders Hejlsberg"),
    
    new("int ใน C# มีขนาดกี่ bytes?",
        new[] { "2 bytes", "4 bytes", "8 bytes", "16 bytes" }, 1,
        "int (Int32) มีขนาด 4 bytes = 32 bits"),
    
    new("คำสั่งใดใช้สำหรับวน loop แบบตรวจเงื่อนไขก่อน?",
        new[] { "do-while", "for", "while", "foreach" }, 2,
        "while ตรวจเงื่อนไขก่อน ถ้าเท็จจะไม่รัน code เลย"),
    
    new("'var' ใน C# หมายถึงอะไร?",
        new[] { "Dynamic type", "Variant", "Type inference", "Variable" }, 2,
        "var คือ type inference - compiler กำหนดชนิดให้อัตโนมัติ"),
    
    new("String ใน C# เป็น Value type หรือ Reference type?",
        new[] { "Value type", "Reference type", "ทั้งสอง", "ขึ้นอยู่กับ context" }, 1,
        "String เป็น Reference type แต่ Immutable และมี value semantics"),
};

// ============ Game State ============
int score = 0;
int total = questions.Count;
var wrongAnswers = new List<(int QuestionIndex, int UserAnswer)>();
var random = new Random();

// ============ Shuffle questions ============
questions = questions.OrderBy(_ => random.Next()).ToList();

// ============ Display Welcome ============
Console.Clear();
Console.ForegroundColor = ConsoleColor.Cyan;
Console.WriteLine("╔══════════════════════════════════╗");
Console.WriteLine("║      C# Quiz - ทดสอบความรู้!     ║");
Console.WriteLine("╚══════════════════════════════════╝");
Console.ResetColor();
Console.WriteLine($"จำนวนคำถาม: {total} ข้อ");
Console.WriteLine("กด Enter เพื่อเริ่ม...");
Console.ReadLine();

// ============ Game Loop ============
for (int q = 0; q < questions.Count; q++)
{
    var question = questions[q];
    
    Console.Clear();
    Console.ForegroundColor = ConsoleColor.Yellow;
    Console.WriteLine($"คำถามที่ {q + 1}/{total}  |  คะแนน: {score}");
    Console.ResetColor();
    Console.WriteLine(new string('─', 50));
    Console.WriteLine();
    Console.WriteLine(question.Text);
    Console.WriteLine();
    
    for (int i = 0; i < question.Options.Length; i++)
    {
        Console.WriteLine($"  {i + 1}. {question.Options[i]}");
    }
    
    Console.WriteLine();
    
    // Get valid answer
    int userAnswer = -1;
    while (true)
    {
        Console.Write("คำตอบของคุณ (1-4): ");
        string input = Console.ReadLine() ?? "";
        
        if (int.TryParse(input, out int parsed) && parsed >= 1 && parsed <= question.Options.Length)
        {
            userAnswer = parsed - 1; // Convert to 0-indexed
            break;
        }
        
        Console.ForegroundColor = ConsoleColor.Red;
        Console.WriteLine($"กรุณาใส่ตัวเลข 1-{question.Options.Length}");
        Console.ResetColor();
    }
    
    // Check answer
    bool isCorrect = userAnswer == question.CorrectIndex;
    
    if (isCorrect)
    {
        score++;
        Console.ForegroundColor = ConsoleColor.Green;
        Console.WriteLine("\n✅ ถูกต้อง!");
    }
    else
    {
        wrongAnswers.Add((q, userAnswer));
        Console.ForegroundColor = ConsoleColor.Red;
        Console.WriteLine($"\n❌ ผิด! คำตอบที่ถูกต้องคือ: {question.Options[question.CorrectIndex]}");
    }
    
    Console.ForegroundColor = ConsoleColor.Gray;
    Console.WriteLine($"💡 {question.Explanation}");
    Console.ResetColor();
    
    Console.WriteLine("\nกด Enter เพื่อดำเนินต่อ...");
    Console.ReadLine();
}

// ============ Final Score ============
Console.Clear();
Console.ForegroundColor = ConsoleColor.Cyan;
Console.WriteLine("╔══════════════════════════════════╗");
Console.WriteLine("║           ผลคะแนน Quiz           ║");
Console.WriteLine("╚══════════════════════════════════╝");
Console.ResetColor();

double percentage = (double)score / total * 100;
Console.WriteLine($"\nคะแนน: {score}/{total} ({percentage:F1}%)");
Console.WriteLine();

// Grade based on score
string resultMessage = percentage switch
{
    >= 90 => "🏆 ยอดเยี่ยม! คุณเป็นมือโปร C#!",
    >= 80 => "🥇 ดีมาก! ความรู้ C# อยู่ในระดับดี",
    >= 70 => "🥈 ดี! ยังมีที่ต้องพัฒนา",
    >= 60 => "🥉 ผ่าน! ควรทบทวนเนื้อหาเพิ่ม",
    _ => "📚 ยังไม่ผ่าน กลับไปเรียนใหม่นะ"
};

Console.ForegroundColor = ConsoleColor.Yellow;
Console.WriteLine(resultMessage);
Console.ResetColor();

// Progress bar
Console.Write("\nความก้าวหน้า: [");
int filled = (int)(percentage / 5); // 20 chars wide
for (int i = 0; i < 20; i++)
{
    if (i < filled)
    {
        Console.ForegroundColor = ConsoleColor.Green;
        Console.Write("█");
    }
    else
    {
        Console.ForegroundColor = ConsoleColor.DarkGray;
        Console.Write("░");
    }
}
Console.ResetColor();
Console.WriteLine($"] {percentage:F0}%");

// Show wrong answers
if (wrongAnswers.Count > 0)
{
    Console.WriteLine("\n📝 ข้อที่ตอบผิด:");
    foreach (var (qIdx, userAns) in wrongAnswers)
    {
        var q = questions[qIdx];
        Console.ForegroundColor = ConsoleColor.Red;
        Console.WriteLine($"  ❌ {q.Text}");
        Console.ResetColor();
        Console.WriteLine($"     คุณตอบ: {q.Options[userAns]}");
        Console.ForegroundColor = ConsoleColor.Green;
        Console.WriteLine($"     ถูกต้อง: {q.Options[q.CorrectIndex]}");
        Console.ResetColor();
    }
}

Console.WriteLine("\nขอบคุณที่เล่น Quiz! 🎉");
```

---

## สรุป Part 04

```
✅ Step 51: if-else Statements และ Guard Clauses
✅ Step 52: switch Statement และ switch Expression
✅ Step 53: for Loop และ Nested Loops
✅ Step 54: while และ do-while Loops
✅ Step 55: foreach Loop
✅ Steps 56-60: goto, Pattern Matching ขั้นสูง
✅ Steps 61-70: Iterator Methods, Parallel.For
✅ Steps 71-80: โปรแกรม Quiz Game สมบูรณ์
```

## แบบฝึกหัด Part 04

1. เขียนโปรแกรมทายตัวเลข 1-100 ด้วย Binary Search approach
2. สร้าง pattern: วาด diamond ด้วย * ตามขนาดที่กำหนด
3. เขียน FizzBuzz 1-100 ด้วย switch expression
4. สร้าง iterator ที่คืน prime numbers
5. เขียนโปรแกรม ATM จำลอง ด้วย do-while menu

---

**ถัดไป: [Part 05 - เมธอดและฟังก์ชัน →](part-05-methods-functions.md)**
