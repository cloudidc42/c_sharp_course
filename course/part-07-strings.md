# Part 07: Strings และการจัดการข้อความ (Steps 141-170)

## เป้าหมายการเรียนรู้
- เข้าใจ String เป็น Reference Type และ String Interning
- ใช้ String methods ทั้งหมดอย่างมืออาชีพ
- String formatting รูปแบบต่างๆ ใน .NET
- Regular Expressions สำหรับการค้นหาและแปลงข้อมูล
- StringBuilder สำหรับ performance
- Span\<char\> และ Memory\<char\> สำหรับ high-performance string processing
- C# 11+ Raw string literals และ UTF-8 string literals

---

## Step 141: String Fundamentals และ Immutability

```csharp
// String เป็น Reference Type แต่ทำงานเหมือน Value Type
// เพราะ String เป็น immutable (เปลี่ยนแปลงไม่ได้)

string s1 = "Hello";
string s2 = "Hello";
string s3 = new string("Hello"); // สร้าง instance ใหม่

// String Interning: .NET เก็บ string literal ไว้ใน intern pool
Console.WriteLine(object.ReferenceEquals(s1, s2)); // True (same interned reference)
Console.WriteLine(object.ReferenceEquals(s1, s3)); // False (new instance)

// แต่ == เปรียบเทียบ value
Console.WriteLine(s1 == s3); // True

// String.Intern
string s4 = string.Intern(new string("Hello"));
Console.WriteLine(object.ReferenceEquals(s1, s4)); // True

// Immutability: การ "เปลี่ยน" string จริงๆ สร้าง string ใหม่
string original = "Hello";
string modified = original + " World";
Console.WriteLine(original); // "Hello" (ไม่เปลี่ยน)
Console.WriteLine(modified); // "Hello World" (ใหม่)

// String length และ indexing
string text = "C# Programming";
Console.WriteLine(text.Length);    // 14
Console.WriteLine(text[0]);        // 'C'
Console.WriteLine(text[^1]);       // 'g' (จากท้าย C# 8+)
Console.WriteLine(text[3..8]);     // "Progr" (range C# 8+)

// Null vs Empty vs Whitespace
string? nullStr = null;
string emptyStr = "";
string whitespaceStr = "   ";

Console.WriteLine(string.IsNullOrEmpty(nullStr));      // True
Console.WriteLine(string.IsNullOrEmpty(emptyStr));     // True
Console.WriteLine(string.IsNullOrWhiteSpace(whitespaceStr)); // True
Console.WriteLine(string.IsNullOrEmpty(whitespaceStr));// False!
```

---

## Step 142: String Creation Methods

```csharp
// 1. String Literals
string literal = "Hello, World!";
string verbatim = @"C:\Users\file.txt";    // verbatim string
string multiline = @"Line 1
Line 2
Line 3";

// 2. String Interpolation (C# 6+)
string name = "Alice";
int age = 30;
string interpolated = $"Name: {name}, Age: {age}";
string withFormat = $"Pi = {Math.PI:F4}";
string withExpression = $"Sum: {1 + 2 + 3}";
string conditional = $"Status: {(age >= 18 ? "Adult" : "Minor")}";

// Nested interpolation
string nested = $"Result: {$"Inner {name}"}";

// 3. Raw String Literals (C# 11+)
string raw = """
    This is a raw string.
    No need to escape "quotes" or backslashes \.
    """;

// Interpolated raw string
string rawInterpolated = $"""
    Name: {name}
    Age: {age}
    """;

// Multiple $ for literal braces
string withBraces = $$"""
    JSON: {"key": "{{name}}"}
    """;

// 4. UTF-8 String Literals (C# 11+)
ReadOnlySpan<byte> utf8 = "Hello UTF-8"u8;
byte[] utf8Array = "Hello"u8.ToArray();

// 5. String.Create for high performance
string created = string.Create(5, "Hello", (span, state) =>
{
    for (int i = 0; i < state.Length; i++)
        span[i] = state[i];
});

// 6. new string(char, count)
string repeated = new string('*', 10);     // "**********"
string fromChars = new string(new char[] {'H', 'i'});

// 7. String.Join
string joined = string.Join(", ", "a", "b", "c");  // "a, b, c"
string joinList = string.Join(" | ", new[] { 1, 2, 3 }); // "1 | 2 | 3"

// 8. String.Concat
string concat = string.Concat("Hello", " ", "World");
string concatArray = string.Concat(new[] { "a", "b", "c" }); // "abc"

Console.WriteLine(interpolated);     // Name: Alice, Age: 30
Console.WriteLine(withFormat);       // Pi = 3.1416
Console.WriteLine(raw);
Console.WriteLine(repeated);         // **********
Console.WriteLine(joined);           // a, b, c
```

---

## Step 143: String Search Methods

```csharp
string text = "The quick brown fox jumps over the lazy dog";

// Contains
Console.WriteLine(text.Contains("fox"));                    // True
Console.WriteLine(text.Contains("FOX"));                    // False
Console.WriteLine(text.Contains("FOX", StringComparison.OrdinalIgnoreCase)); // True

// StartsWith / EndsWith
Console.WriteLine(text.StartsWith("The"));                  // True
Console.WriteLine(text.EndsWith("dog"));                    // True
Console.WriteLine(text.StartsWith("the", StringComparison.OrdinalIgnoreCase)); // True

// IndexOf / LastIndexOf
Console.WriteLine(text.IndexOf("o"));          // 11 (brown)
Console.WriteLine(text.LastIndexOf("o"));      // 41 (dog)
Console.WriteLine(text.IndexOf("o", 12));      // 17 (fox)
Console.WriteLine(text.IndexOf("xyz"));        // -1 (not found)

// IndexOf with StringComparison
int idx = text.IndexOf("FOX", StringComparison.OrdinalIgnoreCase); // 16

// IndexOfAny / LastIndexOfAny
char[] vowels = { 'a', 'e', 'i', 'o', 'u' };
Console.WriteLine(text.IndexOfAny(vowels));     // 2 (e in "The")
Console.WriteLine(text.LastIndexOfAny(vowels)); // 41 (o in "dog")

// Span-based search (C# 8+, high performance)
ReadOnlySpan<char> span = text.AsSpan();
bool containsFox = span.Contains("fox", StringComparison.Ordinal);
int foxIndex = span.IndexOf("fox");

// Count occurrences
string countTarget = "the";
int count = 0;
int pos = 0;
while ((pos = text.IndexOf(countTarget, pos, StringComparison.OrdinalIgnoreCase)) >= 0)
{
    count++;
    pos++;
}
Console.WriteLine($"'the' appears {count} times"); // 2 (The, the)

// MemoryExtensions for Span
var countBetter = text.AsSpan().Count("the", StringComparison.OrdinalIgnoreCase);
```

---

## Step 144: String Manipulation Methods

```csharp
string text = "  Hello, World!  ";

// Trim, TrimStart, TrimEnd
Console.WriteLine(text.Trim());           // "Hello, World!"
Console.WriteLine(text.TrimStart());      // "Hello, World!  "
Console.WriteLine(text.TrimEnd());        // "  Hello, World!"
Console.WriteLine(text.Trim(' ', '!'));   // "Hello, World"
Console.WriteLine("***hello***".Trim('*')); // "hello"

// ToUpper, ToLower
Console.WriteLine("hello".ToUpper());     // "HELLO"
Console.WriteLine("HELLO".ToLower());     // "hello"
Console.WriteLine("hello".ToUpperInvariant()); // Culture-independent

// Substring / Slice (C# 8+)
string s = "Hello, World!";
Console.WriteLine(s.Substring(7));        // "World!"
Console.WriteLine(s.Substring(7, 5));     // "World"
Console.WriteLine(s[7..]);               // "World!" (range)
Console.WriteLine(s[7..12]);             // "World"
Console.WriteLine(s[..5]);              // "Hello"

// Replace
string original = "The cat sat on the mat";
Console.WriteLine(original.Replace("at", "og")); // "The cog sog on the mog"
Console.WriteLine(original.Replace("cat", "dog", StringComparison.OrdinalIgnoreCase));

// Insert, Remove
string inserted = "Hello World".Insert(5, ","); // "Hello, World"
string removed = "Hello World".Remove(5);        // "Hello"
string removedLen = "Hello World".Remove(5, 6);  // "Hello"

// PadLeft, PadRight
Console.WriteLine("42".PadLeft(6));        // "    42"
Console.WriteLine("42".PadRight(6));       // "42    "
Console.WriteLine("42".PadLeft(6, '0'));   // "000042"

// Split
string csv = "a,b,c,d,e";
string[] parts = csv.Split(',');          // ["a","b","c","d","e"]
string[] limited = csv.Split(',', 3);     // ["a","b","c,d,e"]
string[] trimmed = "  a , b , c  ".Split(',', StringSplitOptions.TrimEntries); // ["a","b","c"]
string[] noEmpty = "a,,b,,c".Split(',', StringSplitOptions.RemoveEmptyEntries);

// Split by multiple separators
string multi = "one; two, three | four";
string[] words = multi.Split(new[] { ';', ',', '|' }, StringSplitOptions.TrimEntries | StringSplitOptions.RemoveEmptyEntries);

// Reverse (no built-in, use LINQ or Span)
string reversed = new string("Hello".Reverse().ToArray()); // "olleH"
// Better: use span
char[] chars = "Hello".ToCharArray();
Array.Reverse(chars);
string reversedFast = new string(chars);

// String.Compare
int result = string.Compare("apple", "Banana", StringComparison.OrdinalIgnoreCase);
Console.WriteLine(result); // negative (apple < banana alphabetically)
```

---

## Step 145: String Formatting

```csharp
// Composite Formatting
Console.WriteLine("Hello, {0}! You are {1} years old.", "Alice", 30);
string formatted = string.Format("{0:C}", 1234.56);          // $1,234.56
string padded = string.Format("{0,10}", "right");             // "     right"
string leftPad = string.Format("{0,-10}|", "left");           // "left      |"

// Numeric Format Specifiers
double number = 1234567.891;
Console.WriteLine(number.ToString("C"));       // $1,234,567.89 (Currency)
Console.WriteLine(number.ToString("C2"));      // $1,234,567.89
Console.WriteLine(number.ToString("D"));       // Invalid for double
Console.WriteLine(1234567.ToString("D"));      // 1234567 (int only)
Console.WriteLine(number.ToString("E"));       // 1.234568E+006
Console.WriteLine(number.ToString("F2"));      // 1234567.89 (Fixed)
Console.WriteLine(number.ToString("G"));       // 1234567.891 (General)
Console.WriteLine(number.ToString("N2"));      // 1,234,567.89 (Number with comma)
Console.WriteLine(0.1234.ToString("P1"));      // 12.3% (Percent)
Console.WriteLine(255.ToString("X"));         // FF (Hexadecimal)
Console.WriteLine(255.ToString("X4"));        // 00FF

// Custom Numeric Format
Console.WriteLine(1234.5.ToString("000,000.00"));  // 001,234.50
Console.WriteLine(0.75.ToString("#0.0%"));          // 75.0%
Console.WriteLine(1234.ToString("#,##0"));          // 1,234
Console.WriteLine(12.3.ToString("000.00"));         // 012.30

// DateTime Formatting
DateTime now = DateTime.Now;
Console.WriteLine(now.ToString("d"));      // Short date: 9/27/2026
Console.WriteLine(now.ToString("D"));      // Long date: Saturday, September 27, 2026
Console.WriteLine(now.ToString("t"));      // Short time: 2:30 PM
Console.WriteLine(now.ToString("T"));      // Long time: 2:30:45 PM
Console.WriteLine(now.ToString("f"));      // Full (short time)
Console.WriteLine(now.ToString("F"));      // Full (long time)
Console.WriteLine(now.ToString("g"));      // General (short)
Console.WriteLine(now.ToString("G"));      // General (long)
Console.WriteLine(now.ToString("M"));      // Month/day
Console.WriteLine(now.ToString("R"));      // RFC1123
Console.WriteLine(now.ToString("s"));      // Sortable: 2026-09-27T14:30:45
Console.WriteLine(now.ToString("u"));      // Universal sortable
Console.WriteLine(now.ToString("Y"));      // Year/month

// Custom DateTime Format
Console.WriteLine(now.ToString("dd/MM/yyyy HH:mm:ss"));    // 27/09/2026 14:30:45
Console.WriteLine(now.ToString("yyyy-MM-ddTHH:mm:ss.fffZ")); // ISO 8601
Console.WriteLine(now.ToString("dddd, MMMM d, yyyy"));     // Saturday, September 27, 2026

// String Interpolation with format
double price = 29.99;
string result = $"Price: {price:C}";             // Price: $29.99
string result2 = $"Date: {now:yyyy-MM-dd}";      // Date: 2026-09-27
string result3 = $"Hex: {255:X4}";              // Hex: 00FF
string result4 = $"Percent: {0.75:P0}";         // Percent: 75%

// IFormattable
Console.WriteLine(FormattableString.Invariant($"{price:F2}")); // Always uses . as decimal
```

---

## Step 146: StringBuilder — High Performance String Building

```csharp
using System.Text;

// ทำไมต้องใช้ StringBuilder?
// string concatenation ใน loop = O(n²) เพราะสร้าง string ใหม่ทุกครั้ง
// StringBuilder = O(n) เพราะใช้ buffer ที่ขยายได้

// สร้าง StringBuilder
StringBuilder sb1 = new StringBuilder();
StringBuilder sb2 = new StringBuilder(100);          // initial capacity
StringBuilder sb3 = new StringBuilder("Hello");      // initial value
StringBuilder sb4 = new StringBuilder("Hello", 100); // value + capacity

// Append Methods
sb1.Append("Hello");
sb1.Append(", ");
sb1.Append("World");
sb1.Append('!');
sb1.Append(42);
sb1.Append(3.14);
sb1.Append(true);
Console.WriteLine(sb1.ToString()); // Hello, World!42 3.14True

// AppendLine
StringBuilder lines = new StringBuilder();
lines.AppendLine("Line 1");
lines.AppendLine("Line 2");
lines.AppendLine("Line 3");
// Each line ends with \r\n (Windows) or \n (Unix)

// AppendFormat
StringBuilder formatted = new StringBuilder();
formatted.AppendFormat("Name: {0}, Age: {1}\n", "Alice", 30);
formatted.AppendFormat("Score: {0:F2}\n", 95.678);

// AppendJoin (C# 6+)
StringBuilder joined = new StringBuilder();
joined.AppendJoin(", ", new[] { "a", "b", "c" }); // "a, b, c"
joined.AppendJoin("|", 1, 2, 3, 4, 5);            // adds "1|2|3|4|5"

// Insert, Remove, Replace
StringBuilder edit = new StringBuilder("Hello World");
edit.Insert(5, ",");          // "Hello, World"
edit.Remove(6, 1);            // "Hello World" (remove space)
edit.Replace("World", "C#"); // "HelloC#"

// Chaining (Fluent)
string result = new StringBuilder()
    .Append("Hello")
    .Append(", ")
    .Append("World")
    .AppendLine("!")
    .ToString();

// Performance comparison
var stopwatch = System.Diagnostics.Stopwatch.StartNew();

// Bad: string concatenation
string bad = "";
for (int i = 0; i < 10000; i++)
    bad += i.ToString();

stopwatch.Stop();
long concatMs = stopwatch.ElapsedMilliseconds;

stopwatch.Restart();

// Good: StringBuilder
var good = new StringBuilder();
for (int i = 0; i < 10000; i++)
    good.Append(i);
string goodResult = good.ToString();

stopwatch.Stop();
long sbMs = stopwatch.ElapsedMilliseconds;

Console.WriteLine($"Concat: {concatMs}ms, StringBuilder: {sbMs}ms");
// StringBuilder is typically 10-100x faster for many concatenations

// StringBuilder properties
StringBuilder sb = new StringBuilder("Hello, World!");
Console.WriteLine(sb.Length);      // 13
Console.WriteLine(sb.Capacity);    // 16 (or more)
sb.Capacity = 100;                 // pre-allocate
sb.Length = 5;                     // truncate to "Hello"
Console.WriteLine(sb[0]);         // 'H'
sb[0] = 'h';                       // modify in place!

// Clear and reuse
sb.Clear(); // Faster than new StringBuilder()
```

---

## Step 147: Regular Expressions (Regex)

```csharp
using System.Text.RegularExpressions;

// Basic Pattern Matching
string text = "Hello, my email is alice@example.com and phone is 555-1234";

// IsMatch
bool hasEmail = Regex.IsMatch(text, @"\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b");
Console.WriteLine(hasEmail); // True

// Match (first match)
Match match = Regex.Match(text, @"\b\w+@\w+\.\w+\b");
if (match.Success)
    Console.WriteLine(match.Value); // alice@example.com

// Matches (all matches)
string emails = "Send to alice@test.com, bob@example.org, carol@web.net";
MatchCollection allEmails = Regex.Matches(emails, @"\b[\w.+-]+@[\w-]+\.[a-z]{2,}\b");
foreach (Match m in allEmails)
    Console.WriteLine(m.Value);
// alice@test.com
// bob@example.org
// carol@web.net

// Groups and Captures
string date = "Today is 2026-09-27 and tomorrow is 2026-09-28";
string datePattern = @"(\d{4})-(\d{2})-(\d{2})";
foreach (Match m in Regex.Matches(date, datePattern))
{
    Console.WriteLine($"Full: {m.Value}, Year: {m.Groups[1]}, Month: {m.Groups[2]}, Day: {m.Groups[3]}");
}

// Named Groups
string namedPattern = @"(?<year>\d{4})-(?<month>\d{2})-(?<day>\d{2})";
Match namedMatch = Regex.Match("2026-09-27", namedPattern);
if (namedMatch.Success)
{
    Console.WriteLine($"Year: {namedMatch.Groups["year"].Value}");
    Console.WriteLine($"Month: {namedMatch.Groups["month"].Value}");
    Console.WriteLine($"Day: {namedMatch.Groups["day"].Value}");
}

// Replace
string cleaned = Regex.Replace("Hello   World", @"\s+", " "); // "Hello World"
string masked = Regex.Replace("SSN: 123-45-6789", @"\d{3}-\d{2}-(\d{4})", "XXX-XX-$1");
// "SSN: XXX-XX-6789"

// Replace with MatchEvaluator
string result = Regex.Replace("hello world", @"\b\w", m => m.Value.ToUpper());
// "Hello World"

// Split
string[] words = Regex.Split("one,two;three|four", @"[,;|]");
// ["one", "two", "three", "four"]

// Compiled Regex (better performance when used repeatedly)
Regex emailRegex = new Regex(
    @"\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b",
    RegexOptions.Compiled | RegexOptions.IgnoreCase
);

// Source Generator Regex (C# 11+) - zero overhead at runtime
public partial class Validator
{
    [GeneratedRegex(@"^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$")]
    public static partial Regex EmailRegex();
    
    [GeneratedRegex(@"^\d{3}-\d{4}$")]
    public static partial Regex PhoneRegex();
}

// Common Regex Patterns
var patterns = new Dictionary<string, string>
{
    ["Email"]    = @"^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$",
    ["URL"]      = @"^https?://[\w.-]+(?:\.[\w\.-]+)+[\w\-\._~:/?#[\]@!\$&'\(\)\*\+,;=.]+$",
    ["Phone"]    = @"^\+?(\d[\s-]?)?(\(?\d{3}\)?[\s.-]?)(\d{3}[\s.-]?)(\d{4})$",
    ["IP"]       = @"^(\d{1,3}\.){3}\d{1,3}$",
    ["Zip"]      = @"^\d{5}(-\d{4})?$",
    ["Credit"]   = @"^\d{4}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}$",
    ["Password"] = @"^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&])[A-Za-z\d@$!%*?&]{8,}$",
};

foreach (var (name, pattern) in patterns)
{
    Console.WriteLine($"{name}: {pattern}");
}
```

---

## Step 148: String Encoding และ Conversion

```csharp
using System.Text;

// Encoding Types
Encoding utf8 = Encoding.UTF8;
Encoding utf16 = Encoding.Unicode;     // UTF-16 LE
Encoding utf16be = Encoding.BigEndianUnicode; // UTF-16 BE
Encoding utf32 = Encoding.UTF32;
Encoding ascii = Encoding.ASCII;
Encoding latin1 = Encoding.Latin1;

// String to Bytes
string message = "Hello, สวัสดี! 🌟";
byte[] utf8Bytes = Encoding.UTF8.GetBytes(message);
byte[] utf16Bytes = Encoding.Unicode.GetBytes(message);
Console.WriteLine($"UTF-8 bytes: {utf8Bytes.Length}");    // 26+ (Thai chars = 3 bytes each)
Console.WriteLine($"UTF-16 bytes: {utf16Bytes.Length}");  // 34+ (2 bytes per char + BOM)

// Bytes to String
string decoded = Encoding.UTF8.GetString(utf8Bytes);
Console.WriteLine(decoded); // "Hello, สวัสดี! 🌟"

// BOM (Byte Order Mark)
Encoding utf8WithBom = new UTF8Encoding(encoderShouldEmitUTF8Identifier: true);
byte[] bom = utf8WithBom.GetPreamble(); // [0xEF, 0xBB, 0xBF]

// Base64 Encoding/Decoding
string original = "Hello, World!";
string base64 = Convert.ToBase64String(Encoding.UTF8.GetBytes(original));
string decoded64 = Encoding.UTF8.GetString(Convert.FromBase64String(base64));
Console.WriteLine(base64);    // SGVsbG8sIFdvcmxkIQ==
Console.WriteLine(decoded64); // Hello, World!

// Base64Url (no +, /, = chars)
string base64Url = base64.Replace('+', '-').Replace('/', '_').TrimEnd('=');

// Hex Encoding
string hex = Convert.ToHexString(utf8Bytes);            // uppercase
string hexLower = BitConverter.ToString(utf8Bytes).Replace("-", ""); // "48-65-..."
byte[] fromHex = Convert.FromHexString("48656C6C6F");

// HTML Encoding
using System.Net; // or System.Web
string html = "<script>alert('xss')</script>";
string encoded = System.Net.WebUtility.HtmlEncode(html);   // &lt;script&gt;...
string decoded2 = System.Net.WebUtility.HtmlDecode(encoded);

// URL Encoding
string url = "hello world & more";
string urlEncoded = Uri.EscapeDataString(url);    // hello%20world%20%26%20more
string urlDecoded = Uri.UnescapeDataString(urlEncoded);
string urlEscaped = Uri.EscapeUriString("https://example.com/path?key=value with space");

// Unicode normalization
string nfc = "café".Normalize(NormalizationForm.FormC);   // NFC: composed
string nfd = "café".Normalize(NormalizationForm.FormD);   // NFD: decomposed
bool same = string.Compare(nfc, nfd, StringComparison.Ordinal) == 0; // May be false!
bool sameNorm = string.Compare(nfc, nfd, StringComparison.CurrentCulture) == 0; // True

// Character categories
char c = 'A';
Console.WriteLine(char.IsLetter(c));      // True
Console.WriteLine(char.IsDigit('5'));     // True
Console.WriteLine(char.IsWhiteSpace(' ')); // True
Console.WriteLine(char.IsUpper('A'));     // True
Console.WriteLine(char.IsLower('a'));     // True
Console.WriteLine(char.IsPunctuation('.')); // True
Console.WriteLine(char.GetNumericValue('7')); // 7.0

// Converting chars
char upper = char.ToUpper('a'); // 'A'
char lower = char.ToLower('A'); // 'a'
```

---

## Step 149: Span\<char\> และ High-Performance String Processing

```csharp
using System;

// Span<char> - zero allocation string slicing
string text = "Hello, World! This is a test.";

// AsSpan() - no copy, no allocation
ReadOnlySpan<char> span = text.AsSpan();
ReadOnlySpan<char> hello = span[..5];        // "Hello" - no allocation!
ReadOnlySpan<char> world = span[7..12];      // "World" - no allocation!

Console.WriteLine(hello.ToString());          // "Hello"
Console.WriteLine(world.Length);              // 5
Console.WriteLine(hello.SequenceEqual("Hello")); // True

// Trimming with Span (no allocation)
ReadOnlySpan<char> padded = "  Hello  ".AsSpan();
ReadOnlySpan<char> trimmedSpan = padded.Trim();
Console.WriteLine(trimmedSpan.ToString()); // "Hello"

// Splitting with Span
ReadOnlySpan<char> csv = "a,b,c,d".AsSpan();
foreach (Range range in csv.Split(','))
{
    Console.Write(csv[range].ToString() + " ");
}
Console.WriteLine();

// stackalloc for temporary buffers
Span<char> buffer = stackalloc char[100];
int charsWritten = 0;

"Hello".AsSpan().CopyTo(buffer);
charsWritten += 5;
", World".AsSpan().CopyTo(buffer[charsWritten..]);
charsWritten += 7;

ReadOnlySpan<char> result = buffer[..charsWritten];
Console.WriteLine(result.ToString()); // "Hello, World"

// String processing without allocation
ReadOnlySpan<char> data = "Name:Alice,Age:30,City:Bangkok";
while (data.Length > 0)
{
    int commaIndex = data.IndexOf(',');
    ReadOnlySpan<char> field = commaIndex >= 0 ? data[..commaIndex] : data;
    
    int colonIndex = field.IndexOf(':');
    if (colonIndex >= 0)
    {
        Console.WriteLine($"Key: {field[..colonIndex].ToString()}, Value: {field[(colonIndex+1)..].ToString()}");
    }
    
    data = commaIndex >= 0 ? data[(commaIndex+1)..] : ReadOnlySpan<char>.Empty;
}

// Memory<char> for async scenarios
Memory<char> memory = new char[50];
"Hello".AsSpan().CopyTo(memory.Span);

// High-performance number parsing
ReadOnlySpan<char> numSpan = "12345".AsSpan();
if (int.TryParse(numSpan, out int parsed))
    Console.WriteLine(parsed); // 12345

// TryWrite / TryFormat (C# 10+)
Span<char> output = stackalloc char[50];
if (DateTime.Now.TryFormat(output, out int written, "yyyy-MM-dd"))
{
    Console.WriteLine(output[..written].ToString()); // "2026-09-27"
}
```

---

## Step 150: StringComparison และ Culture-Aware Operations

```csharp
using System.Globalization;

// StringComparison enum
string a = "resume";
string b = "Résumé";

// Different comparison modes
Console.WriteLine(string.Compare(a, b, StringComparison.Ordinal));           // case-sensitive binary
Console.WriteLine(string.Compare(a, b, StringComparison.OrdinalIgnoreCase)); // case-insensitive binary
Console.WriteLine(string.Compare(a, b, StringComparison.CurrentCulture));    // locale-aware
Console.WriteLine(string.Compare(a, b, StringComparison.InvariantCulture));  // invariant locale

// Best practices for comparison
// Use Ordinal for internal identifiers (file paths, URLs, dictionary keys)
string path1 = "C:\\Users\\Alice";
string path2 = "c:\\users\\alice";
bool samePath = string.Equals(path1, path2, StringComparison.OrdinalIgnoreCase);
Console.WriteLine(samePath); // True on Windows

// Use InvariantCulture for user-visible data without locale
bool sameInvariant = string.Equals("hello", "HELLO", StringComparison.InvariantCultureIgnoreCase);

// Use CurrentCulture for user-facing string comparisons
CultureInfo thai = new CultureInfo("th-TH");
CultureInfo english = new CultureInfo("en-US");

// Sorting
string[] fruits = { "banana", "Apple", "cherry", "APPLE" };
Array.Sort(fruits, StringComparer.OrdinalIgnoreCase);
foreach (var f in fruits) Console.Write(f + " ");
// APPLE Apple banana cherry

// Culture-specific sorting
Array.Sort(fruits, StringComparer.Create(english, ignoreCase: true));

// String operations with culture
CultureInfo tr = new CultureInfo("tr-TR"); // Turkish
string upper_tr = "i".ToUpper(tr);          // "İ" (dotted I in Turkish!)
string upper_en = "i".ToUpper(CultureInfo.InvariantCulture); // "I"

// Avoid culture bugs: always specify when intention matters
bool tricky = "FILE".Equals("file", StringComparison.CurrentCultureIgnoreCase); // Culture dependent!
bool safe = "FILE".Equals("file", StringComparison.OrdinalIgnoreCase);           // Always works

// NumberFormatInfo
NumberFormatInfo nfi = CultureInfo.GetCultureInfo("de-DE").NumberFormat;
string german = (1234.56).ToString("N2", nfi); // "1.234,56" (German uses . for thousands, , for decimal)
string english2 = (1234.56).ToString("N2");    // "1,234.56" (English default)

// Parsing with culture
double num1 = double.Parse("1.234,56", nfi); // 1234.56 (German format)
double num2 = double.Parse("1,234.56", CultureInfo.InvariantCulture); // 1234.56

// DateTimeFormatInfo
DateTimeFormatInfo dtfi = CultureInfo.GetCultureInfo("th-TH").DateTimeFormat;
Console.WriteLine(DateTime.Now.ToString("D", dtfi)); // Thai date format

// Comparing strings correctly: DO and DON'T
// DON'T: "hello" == "HELLO" (ordinal, case-sensitive)
// DO:    string.Equals("hello", "HELLO", StringComparison.OrdinalIgnoreCase)

// DON'T: dict["KEY"] when keys might differ by case
// DO:    Dictionary<string, V>(StringComparer.OrdinalIgnoreCase)
var caseInsensitiveDict = new Dictionary<string, int>(StringComparer.OrdinalIgnoreCase)
{
    ["Hello"] = 1
};
Console.WriteLine(caseInsensitiveDict["HELLO"]); // 1 (works!)
Console.WriteLine(caseInsensitiveDict["hello"]); // 1 (works!)
```

---

## Step 151: String Parsing และ Conversion

```csharp
// Parse methods
int i = int.Parse("42");
double d = double.Parse("3.14");
bool b = bool.Parse("true");
DateTime dt = DateTime.Parse("2026-09-27");
Guid g = Guid.Parse("550e8400-e29b-41d4-a716-446655440000");

// TryParse (safe, no exceptions)
if (int.TryParse("abc", out int result))
    Console.WriteLine(result);
else
    Console.WriteLine("Parse failed");

// TryParse with span (C# 8+, no allocation)
ReadOnlySpan<char> numSpan = "12345".AsSpan();
int.TryParse(numSpan, out int spanResult);

// Convert class
int fromString = Convert.ToInt32("42");
int fromDouble = Convert.ToInt32(3.99);  // 4 (rounds!)
int fromBool = Convert.ToInt32(true);    // 1
string fromInt = Convert.ToString(42);   // "42"
bool fromZero = Convert.ToBoolean(0);   // false
bool fromOne = Convert.ToBoolean(1);    // true

// Convert to different bases
int hex = Convert.ToInt32("FF", 16);    // 255
int bin = Convert.ToInt32("1010", 2);   // 10
int oct = Convert.ToInt32("17", 8);     // 15
string hexStr = Convert.ToString(255, 16); // "ff"
string binStr = Convert.ToString(10, 2);   // "1010"

// String to various types
string[] values = { "42", "3.14", "true", "2026-09-27", "Alice" };
object[] parsed = new object[values.Length];

// Using TypeConverter
foreach (string v in values)
{
    if (int.TryParse(v, out int intVal)) { Console.WriteLine($"int: {intVal}"); continue; }
    if (double.TryParse(v, out double dblVal)) { Console.WriteLine($"double: {dblVal}"); continue; }
    if (bool.TryParse(v, out bool boolVal)) { Console.WriteLine($"bool: {boolVal}"); continue; }
    if (DateTime.TryParse(v, out DateTime dtVal)) { Console.WriteLine($"date: {dtVal}"); continue; }
    Console.WriteLine($"string: {v}");
}

// Enum.Parse and TryParse
enum Color { Red, Green, Blue }
Color color = Enum.Parse<Color>("Green");           // Color.Green
Color color2 = Enum.Parse<Color>("green", ignoreCase: true); // Color.Green
Enum.TryParse<Color>("Purple", out Color c);        // false
string colorName = color.ToString();                // "Green"
int colorValue = (int)color;                        // 1

// Number format parsing
double german = double.Parse("1.234,56", new System.Globalization.CultureInfo("de-DE"));
double invariant = double.Parse("1234.56", System.Globalization.CultureInfo.InvariantCulture);
```

---

## Step 152: Advanced String Patterns

```csharp
// Pattern: Template Engine (simple)
string Template(string template, Dictionary<string, string> values)
{
    var sb = new StringBuilder(template);
    foreach (var (key, value) in values)
        sb.Replace($"{{{key}}}", value);
    return sb.ToString();
}

string emailTemplate = "Dear {Name},\nYour order {OrderId} for {Amount} has been placed.";
string email = Template(emailTemplate, new()
{
    ["Name"] = "Alice",
    ["OrderId"] = "ORD-001",
    ["Amount"] = "$99.99"
});
Console.WriteLine(email);

// Pattern: String Tokenizer
IEnumerable<string> Tokenize(string input, char separator = ' ')
{
    var span = input.AsSpan();
    while (span.Length > 0)
    {
        int idx = span.IndexOf(separator);
        if (idx < 0)
        {
            if (span.Length > 0) yield return span.ToString();
            yield break;
        }
        if (idx > 0) yield return span[..idx].ToString();
        span = span[(idx + 1)..];
    }
}

foreach (string token in Tokenize("hello world foo bar"))
    Console.WriteLine(token);

// Pattern: String Builder with indentation
class IndentedBuilder
{
    private readonly StringBuilder _sb = new();
    private int _indent = 0;
    
    public IndentedBuilder AppendLine(string line)
    {
        _sb.Append(new string(' ', _indent * 4));
        _sb.AppendLine(line);
        return this;
    }
    
    public IndentedBuilder Indent() { _indent++; return this; }
    public IndentedBuilder Dedent() { _indent--; return this; }
    public override string ToString() => _sb.ToString();
}

var code = new IndentedBuilder()
    .AppendLine("public class MyClass")
    .AppendLine("{")
    .Indent()
    .AppendLine("public void Method()")
    .AppendLine("{")
    .Indent()
    .AppendLine("Console.WriteLine(\"Hello\");")
    .Dedent()
    .AppendLine("}")
    .Dedent()
    .AppendLine("}");
Console.WriteLine(code.ToString());

// Pattern: Fluent String Parser
class StringParser
{
    private string _value;
    
    public StringParser(string value) => _value = value;
    
    public StringParser Trim() { _value = _value.Trim(); return this; }
    public StringParser ToLower() { _value = _value.ToLower(); return this; }
    public StringParser ToUpper() { _value = _value.ToUpper(); return this; }
    public StringParser Replace(string old, string @new) { _value = _value.Replace(old, @new); return this; }
    public StringParser RemoveWhitespace() { _value = Regex.Replace(_value, @"\s+", ""); return this; }
    public string Value => _value;
    public static implicit operator string(StringParser p) => p._value;
}

string cleaned = new StringParser("  Hello,   World!  ")
    .Trim()
    .Replace(",", "")
    .ToString();
Console.WriteLine(cleaned); // "Hello   World!"

// Pattern: Word Count
Dictionary<string, int> WordCount(string text)
{
    return Regex.Matches(text.ToLower(), @"\b\w+\b")
        .GroupBy(m => m.Value)
        .ToDictionary(g => g.Key, g => g.Count());
}

var counts = WordCount("the quick brown fox jumps over the lazy dog. the dog was lazy.");
foreach (var (word, count) in counts.OrderByDescending(x => x.Value).Take(5))
    Console.WriteLine($"{word}: {count}");
```

---

## Step 153: String Performance Best Practices

```csharp
using System.Buffers;

// 1. Use string.IsNullOrEmpty / IsNullOrWhiteSpace instead of length check
string s = GetSomeString();
// Bad:
if (s == null || s.Length == 0) { }
// Good:
if (string.IsNullOrEmpty(s)) { }

// 2. Use StringComparison explicitly
// Bad: (uses CurrentCulture which is locale-dependent)
if (s == "hello") { }
// Good:
if (s.Equals("hello", StringComparison.Ordinal)) { }

// 3. Interpolation vs Format
string name = "Alice";
int age = 30;
// These are equivalent in modern C#:
string a = $"Hello {name}, you are {age}";           // Preferred (readable)
string b = string.Format("Hello {0}, you are {1}", name, age); // Legacy

// 4. Avoid repeated string allocation in loops
// Bad:
for (int i = 0; i < 1000; i++)
    Console.WriteLine("Processing item " + i.ToString()); // New string each iter

// Good:
for (int i = 0; i < 1000; i++)
    Console.WriteLine($"Processing item {i}"); // Optimized by compiler

// 5. StringBuilder for building long strings
// Bad: O(n²)
string result = "";
for (int i = 0; i < 10000; i++)
    result += "x";

// Good: O(n)
var sb = new StringBuilder(10000);
for (int i = 0; i < 10000; i++)
    sb.Append('x');
string goodResult = sb.ToString();

// 6. Span<char> for parsing without allocation
void ParseCoordinates(string input)
{
    // Bad: creates substrings
    var parts = input.Split(',');
    double x = double.Parse(parts[0]);
    double y = double.Parse(parts[1]);
    
    // Good: no allocation
    var span = input.AsSpan();
    int comma = span.IndexOf(',');
    if (comma >= 0 && 
        double.TryParse(span[..comma], out double gx) && 
        double.TryParse(span[(comma+1)..], out double gy))
    {
        Console.WriteLine($"({gx}, {gy})");
    }
}

// 7. ArrayPool<char> for temporary buffers
void ProcessString(string input)
{
    char[] buffer = ArrayPool<char>.Shared.Rent(input.Length * 2);
    try
    {
        // Use buffer...
        int written = input.Length;
        input.AsSpan().CopyTo(buffer);
        var result = new string(buffer, 0, written);
        Console.WriteLine(result);
    }
    finally
    {
        ArrayPool<char>.Shared.Return(buffer);
    }
}

// 8. String.Create for building strings with known length
string BuildHex(byte[] bytes)
{
    return string.Create(bytes.Length * 2, bytes, (span, state) =>
    {
        const string hex = "0123456789ABCDEF";
        for (int i = 0; i < state.Length; i++)
        {
            span[i * 2]     = hex[state[i] >> 4];
            span[i * 2 + 1] = hex[state[i] & 0xF];
        }
    });
}

// 9. Avoid ToString() on StringBuilder if reading multiple times
// Store the result once:
StringBuilder builder = BuildSomething();
string str = builder.ToString();
// Then use str multiple times

// 10. Use const for string constants
const string PREFIX = "LOG: ";
string logMessage = PREFIX + "Starting..."; // Compiler may optimize
```

---

## Step 154: โปรแกรมตัวอย่าง — Text Processing Engine

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Text.RegularExpressions;

// ============================================================
// Text Processing Engine - ระบบประมวลผลข้อความ
// ============================================================

class TextDocument
{
    public string Title { get; init; }
    public string Content { get; init; }
    
    public TextDocument(string title, string content)
    {
        Title = title;
        Content = content;
    }
    
    // นับจำนวนคำ
    public int WordCount => Regex.Matches(Content, @"\b\w+\b").Count;
    
    // นับจำนวนประโยค
    public int SentenceCount => Regex.Matches(Content, @"[.!?]+").Count;
    
    // นับจำนวนย่อหน้า
    public int ParagraphCount => Content.Split(new[] {"\n\n", "\r\n\r\n"}, StringSplitOptions.RemoveEmptyEntries).Length;
    
    // อ่านเฉลี่ย (WPM = 200)
    public double ReadingTimeMinutes => WordCount / 200.0;
    
    // สรุปข้อมูลเอกสาร
    public string Summary => $"""
        Document: {Title}
        Words: {WordCount:N0}
        Sentences: {SentenceCount:N0}
        Paragraphs: {ParagraphCount:N0}
        Reading Time: {ReadingTimeMinutes:F1} minutes
        """;
}

static class TextProcessor
{
    // นับความถี่คำ
    public static Dictionary<string, int> GetWordFrequency(string text, int topN = 10)
    {
        return Regex.Matches(text.ToLower(), @"\b[a-z]+\b")
            .GroupBy(m => m.Value)
            .OrderByDescending(g => g.Count())
            .Take(topN)
            .ToDictionary(g => g.Key, g => g.Count());
    }
    
    // ค้นหาและแทนที่ด้วย context
    public static List<(int Line, int Col, string Context)> Search(string text, string query, 
        bool ignoreCase = true, int contextChars = 30)
    {
        var results = new List<(int, int, string)>();
        var comparison = ignoreCase ? StringComparison.OrdinalIgnoreCase : StringComparison.Ordinal;
        var lines = text.Split('\n');
        
        for (int lineIdx = 0; lineIdx < lines.Length; lineIdx++)
        {
            string line = lines[lineIdx];
            int start = 0;
            int found;
            
            while ((found = line.IndexOf(query, start, comparison)) >= 0)
            {
                int contextStart = Math.Max(0, found - contextChars);
                int contextEnd = Math.Min(line.Length, found + query.Length + contextChars);
                string context = line[contextStart..contextEnd];
                results.Add((lineIdx + 1, found + 1, context));
                start = found + 1;
            }
        }
        
        return results;
    }
    
    // ลบ HTML tags
    public static string StripHtml(string html)
        => Regex.Replace(html, @"<[^>]+>", string.Empty);
    
    // แปลงเป็น Title Case
    public static string ToTitleCase(string text)
        => Regex.Replace(text.ToLower(), @"\b\w", m => m.Value.ToUpper());
    
    // แปลงเป็น camelCase
    public static string ToCamelCase(string text)
    {
        string pascal = Regex.Replace(text, @"(?:^|\s+)(\w)", m => m.Groups[1].Value.ToUpper());
        return char.ToLower(pascal[0]) + pascal[1..];
    }
    
    // แปลงเป็น snake_case
    public static string ToSnakeCase(string text)
        => Regex.Replace(Regex.Replace(text, @"([A-Z])", "_$1"), @"[\s-]+", "_").Trim('_').ToLower();
    
    // แปลงเป็น kebab-case
    public static string ToKebabCase(string text)
        => Regex.Replace(Regex.Replace(text, @"([A-Z])", "-$1"), @"[\s_]+", "-").Trim('-').ToLower();
    
    // Highlight keywords ใน text
    public static string Highlight(string text, string keyword, string openTag = "**", string closeTag = "**")
    {
        return Regex.Replace(text, Regex.Escape(keyword), 
            m => $"{openTag}{m.Value}{closeTag}", RegexOptions.IgnoreCase);
    }
    
    // ตรวจสอบความแข็งแกร่งของ password
    public static (bool IsValid, string[] Issues) ValidatePassword(string password)
    {
        var issues = new List<string>();
        if (password.Length < 8) issues.Add("At least 8 characters required");
        if (!Regex.IsMatch(password, @"[A-Z]")) issues.Add("Must contain uppercase letter");
        if (!Regex.IsMatch(password, @"[a-z]")) issues.Add("Must contain lowercase letter");
        if (!Regex.IsMatch(password, @"\d")) issues.Add("Must contain digit");
        if (!Regex.IsMatch(password, @"[!@#$%^&*]")) issues.Add("Must contain special character");
        return (issues.Count == 0, issues.ToArray());
    }
    
    // สร้าง excerpt
    public static string CreateExcerpt(string text, int maxLength = 150, string ellipsis = "...")
    {
        string clean = Regex.Replace(text.Trim(), @"\s+", " ");
        if (clean.Length <= maxLength) return clean;
        
        int cutPoint = clean.LastIndexOf(' ', maxLength);
        if (cutPoint < 0) cutPoint = maxLength;
        return clean[..cutPoint] + ellipsis;
    }
    
    // แยก emails จาก text
    public static IEnumerable<string> ExtractEmails(string text)
        => Regex.Matches(text, @"\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b")
            .Select(m => m.Value)
            .Distinct();
    
    // แยก URLs จาก text  
    public static IEnumerable<string> ExtractUrls(string text)
        => Regex.Matches(text, @"https?://[^\s<>""]+")
            .Select(m => m.Value)
            .Distinct();
    
    // Wrap text to specified width
    public static string WordWrap(string text, int width)
    {
        var sb = new StringBuilder();
        string[] words = text.Split(' ');
        int lineLength = 0;
        
        foreach (string word in words)
        {
            if (lineLength + word.Length > width && lineLength > 0)
            {
                sb.AppendLine();
                lineLength = 0;
            }
            if (lineLength > 0) { sb.Append(' '); lineLength++; }
            sb.Append(word);
            lineLength += word.Length;
        }
        return sb.ToString();
    }
}

// ============================================================
// Demo
// ============================================================

string sampleText = """
    C# is a modern, object-oriented, and type-safe programming language.
    It was designed by Anders Hejlsberg and first released in 2002.
    C# runs on the .NET platform and is widely used for building
    desktop, web, mobile, cloud, and gaming applications.
    
    C# has evolved significantly over the years, with C# 12 introducing
    primary constructors, collection expressions, and many other features.
    The language continues to grow with each .NET release.
    
    Contact us at info@csharp.com or support@dotnet.org for more information.
    Visit https://learn.microsoft.com/en-us/dotnet/csharp/ for documentation.
    """;

var doc = new TextDocument("C# Overview", sampleText);
Console.WriteLine(doc.Summary);
Console.WriteLine();

// Word frequency
Console.WriteLine("=== Top 10 Words ===");
foreach (var (word, count) in TextProcessor.GetWordFrequency(sampleText))
    Console.WriteLine($"{word,-15} {count}");
Console.WriteLine();

// Search
Console.WriteLine("=== Search: 'C#' ===");
foreach (var (line, col, ctx) in TextProcessor.Search(sampleText, "C#"))
    Console.WriteLine($"Line {line}, Col {col}: ...{ctx}...");
Console.WriteLine();

// Case conversions
string phrase = "hello world from c sharp";
Console.WriteLine($"Title:  {TextProcessor.ToTitleCase(phrase)}");
Console.WriteLine($"Camel:  {TextProcessor.ToCamelCase(phrase)}");
Console.WriteLine($"Snake:  {TextProcessor.ToSnakeCase(phrase)}");
Console.WriteLine($"Kebab:  {TextProcessor.ToKebabCase(phrase)}");
Console.WriteLine();

// Password validation
string[] passwords = { "abc", "Password1", "P@ssw0rd!", "weak" };
foreach (var pwd in passwords)
{
    var (valid, issues) = TextProcessor.ValidatePassword(pwd);
    Console.WriteLine($"{pwd}: {(valid ? "Valid" : "Invalid")} {(issues.Length > 0 ? $"- {string.Join(", ", issues)}" : "")}");
}
Console.WriteLine();

// Extract emails and URLs
Console.WriteLine("Emails: " + string.Join(", ", TextProcessor.ExtractEmails(sampleText)));
Console.WriteLine("URLs: " + string.Join(", ", TextProcessor.ExtractUrls(sampleText)));
Console.WriteLine();

// Word wrap
string longText = "This is a long text that needs to be wrapped at a specific column width for better readability in terminal or fixed-width displays.";
Console.WriteLine("=== Wrapped at 40 chars ===");
Console.WriteLine(TextProcessor.WordWrap(longText, 40));
```

---

## สรุป Part 07

### String Fundamentals
- ✅ String เป็น **immutable reference type** — การเปลี่ยนแปลงสร้าง instance ใหม่
- ✅ **String Interning** — .NET เก็บ literals ไว้ใน pool
- ✅ `string.IsNullOrEmpty` และ `string.IsNullOrWhiteSpace`

### String Creation
- ✅ Literal, @verbatim, `$"interpolated"`, `"""raw"""` (C# 11)
- ✅ `string.Join`, `string.Concat`, `string.Create`
- ✅ UTF-8 literals `"..."u8` (C# 11)

### Search & Manipulation
- ✅ `Contains`, `StartsWith`, `EndsWith`, `IndexOf`, `LastIndexOf`
- ✅ `Replace`, `Insert`, `Remove`, `Split`, `Trim`, `Pad`
- ✅ `Substring`, Range syntax `[..]`

### Formatting
- ✅ Numeric formats: C, D, E, F, G, N, P, X
- ✅ DateTime formats: d, D, t, T, f, F, s, custom
- ✅ `string.Format`, interpolation `$"..."`, `FormattableString.Invariant`

### StringBuilder
- ✅ `Append`, `AppendLine`, `AppendFormat`, `AppendJoin`
- ✅ `Insert`, `Remove`, `Replace`
- ✅ Performance: O(n) vs O(n²) for string concatenation

### Regex
- ✅ `IsMatch`, `Match`, `Matches`, `Replace`, `Split`
- ✅ Groups, Named Groups, MatchEvaluator
- ✅ `[GeneratedRegex]` source generator (C# 11)

### Span\<char\> & Performance
- ✅ `AsSpan()` for zero-allocation slicing
- ✅ `stackalloc` for temporary char buffers
- ✅ `string.Create`, `ArrayPool<char>`

### Encoding
- ✅ UTF-8, UTF-16, ASCII, Latin1
- ✅ Base64, Hex encoding
- ✅ HTML/URL encoding

### StringComparison
- ✅ Ordinal vs CurrentCulture vs InvariantCulture
- ✅ Dictionary with `StringComparer.OrdinalIgnoreCase`

---

## แบบฝึกหัด

1. สร้าง function `MaskSensitiveData(string text)` ที่แทนที่ credit card numbers และ SSNs ด้วย asterisks โดยใช้ Regex
2. เขียน function `WordWrapJustified(string text, int width)` ที่ justify text ทั้งสองด้าน
3. สร้าง `TextDiff` class ที่แสดงความแตกต่างระหว่าง 2 strings แบบ line-by-line
4. เขียน `CsvParser` ที่รองรับ quoted fields, escaped commas โดยใช้ `Span<char>` สำหรับ performance
5. สร้าง simple Markdown parser ที่แปลง `**bold**`, `*italic*`, `# heading` เป็น HTML
