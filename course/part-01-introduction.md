# Part 01: บทนำและการติดตั้ง C# (.NET 8.0/9.0/10.0)
## Steps 1-10: เริ่มต้นโลก C# และ .NET

---

## Step 1: C# คืออะไร และทำไมต้องเรียน?

C# (อ่านว่า "ซี-ชาร์ป") คือภาษาโปรแกรมที่พัฒนาโดย Microsoft ในปี 2000 เป็นภาษาที่ทันสมัย, มีประสิทธิภาพสูง, และใช้งานได้หลากหลาย

### ทำไมต้องเรียน C#?

```
✅ ใช้งานได้จริงใน:
   - Web Applications (ASP.NET Core)
   - Desktop Apps (WPF, WinForms, MAUI)
   - Mobile Apps (Xamarin/MAUI)
   - Game Development (Unity)
   - Cloud Services (Azure)
   - Microservices
   - AI/ML (ML.NET)
   - IoT Applications

✅ ข้อดีของ C#:
   - Type-safe (ปลอดภัยด้านชนิดข้อมูล)
   - Object-Oriented + Functional + Modern
   - Performance สูงเทียบเท่า C/C++ ในบางกรณี
   - Cross-platform (Windows, Linux, macOS)
   - Community ใหญ่, Job market ดี
   - Microsoft สนับสนุนต่อเนื่อง
```

### .NET Versions Timeline

| Version | Release | Support | Features |
|---------|---------|---------|----------|
| .NET 8.0 | Nov 2023 | LTS (2026) | Performance++, Native AOT |
| .NET 9.0 | Nov 2024 | STS (2025) | LINQ++, Regex++, AI |
| .NET 10.0 | Nov 2025 | LTS (2027) | Latest Features |

---

## Step 2: การติดตั้ง .NET SDK

### Windows

```bash
# วิธีที่ 1: ดาวน์โหลดจาก Microsoft
# https://dotnet.microsoft.com/download

# วิธีที่ 2: ใช้ winget (Windows Package Manager)
winget install Microsoft.DotNet.SDK.9

# ตรวจสอบการติดตั้ง
dotnet --version
# Output: 9.0.xxx

dotnet --list-sdks
# Output:
# 8.0.xxx [C:\Program Files\dotnet\sdk]
# 9.0.xxx [C:\Program Files\dotnet\sdk]
```

### macOS

```bash
# วิธีที่ 1: Homebrew
brew install dotnet

# วิธีที่ 2: ดาวน์โหลดจาก Microsoft
# https://dotnet.microsoft.com/download

# ตรวจสอบ
dotnet --version
dotnet --info
```

### Linux (Ubuntu/Debian)

```bash
# Ubuntu 22.04/24.04
sudo apt-get update
sudo apt-get install -y dotnet-sdk-9.0

# หรือใช้ script จาก Microsoft
wget https://dot.net/v1/dotnet-install.sh
chmod +x dotnet-install.sh
./dotnet-install.sh --channel 9.0

# ตรวจสอบ
dotnet --version
```

### ตรวจสอบการติดตั้งสำเร็จ

```bash
dotnet --info
# จะแสดงข้อมูล:
# .NET SDK:
#   Version:           9.0.xxx
#   Commit:            xxxxxxxx
#
# Runtime Environment:
#   OS Name:     ubuntu (หรือ windows/mac)
#   OS Version:  xx.xx
#   OS Platform: Linux (หรือ Windows/macOS)
#   RID:         linux-x64
#   Base Path:   /usr/share/dotnet/sdk/9.0.xxx/
```

---

## Step 3: การติดตั้ง IDE

### Visual Studio Code (แนะนำสำหรับเริ่มต้น)

```bash
# ดาวน์โหลดจาก: https://code.visualstudio.com/

# Extensions ที่ต้องติดตั้ง:
# 1. C# Dev Kit (Microsoft) - หลัก
# 2. C# (OmniSharp) - สนับสนุนพิเศษ
# 3. .NET Install Tool
# 4. NuGet Package Manager GUI
# 5. GitLens
# 6. Prettier
```

### Visual Studio 2022 (แนะนำสำหรับ Windows)

```
ดาวน์โหลดจาก: https://visualstudio.microsoft.com/

Workloads ที่ต้องเลือก:
✅ ASP.NET and web development
✅ .NET desktop development
✅ Cross-platform .NET development
✅ Azure development (ถ้าต้องการ)
```

### JetBrains Rider (Professional)

```
ดาวน์โหลดจาก: https://www.jetbrains.com/rider/
- มีทดลองใช้ 30 วัน
- ฟีเจอร์ครบที่สุด
- เหมาะสำหรับมืออาชีพ
```

---

## Step 4: สร้างโปรแกรมแรก - Hello World

```bash
# สร้าง directory ใหม่
mkdir HelloWorld
cd HelloWorld

# สร้างโปรเจค Console Application
dotnet new console --name HelloWorld
cd HelloWorld

# ดูไฟล์ที่ถูกสร้าง
ls -la
# Program.cs      <- ไฟล์โค้ดหลัก
# HelloWorld.csproj  <- ไฟล์โปรเจค
```

### ไฟล์ Program.cs (Modern .NET 6+)

```csharp
// Program.cs - Top-level statements (NET 6+)
Console.WriteLine("Hello, World!");
Console.WriteLine("สวัสดีชาวโลก!");
Console.WriteLine("ยินดีต้อนรับสู่ C# .NET!");
```

### รันโปรแกรม

```bash
# รันโปรแกรม
dotnet run

# Output:
# Hello, World!
# สวัสดีชาวโลก!
# ยินดีต้อนรับสู่ C# .NET!
```

---

## Step 5: โครงสร้างโปรเจค .NET

### ไฟล์ .csproj

```xml
<!-- HelloWorld.csproj -->
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <!-- ชนิดของ Output -->
    <OutputType>Exe</OutputType>
    
    <!-- .NET Version -->
    <TargetFramework>net9.0</TargetFramework>
    
    <!-- C# Version -->
    <LangVersion>latest</LangVersion>
    
    <!-- Nullable Reference Types -->
    <Nullable>enable</Nullable>
    
    <!-- Implicit Usings -->
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>

</Project>
```

### โครงสร้าง Solution (สำหรับโปรเจคใหญ่)

```bash
# สร้าง Solution
dotnet new sln --name MyCourse
cd MyCourse

# สร้างหลาย Projects
dotnet new console -n ConsoleApp
dotnet new classlib -n CoreLibrary
dotnet new webapi -n WebAPI

# เพิ่มเข้า Solution
dotnet sln add ConsoleApp/ConsoleApp.csproj
dotnet sln add CoreLibrary/CoreLibrary.csproj
dotnet sln add WebAPI/WebAPI.csproj

# เพิ่ม Reference ระหว่าง Projects
cd ConsoleApp
dotnet add reference ../CoreLibrary/CoreLibrary.csproj
```

---

## Step 6: C# Versions และ Features

### C# Version ต่อ .NET Version

```
.NET 8.0  = C# 12
.NET 9.0  = C# 13
.NET 10.0 = C# 14
```

### Features ใหม่ใน C# 12 (.NET 8)

```csharp
// 1. Primary Constructors
public class Person(string name, int age)
{
    public string Name => name;
    public int Age => age;
    public override string ToString() => $"{Name} ({Age})";
}

// 2. Collection Expressions
int[] numbers = [1, 2, 3, 4, 5];
List<string> names = ["Alice", "Bob", "Charlie"];
string[] combined = [..numbers.Select(n => n.ToString()), ..names];

// 3. Inline Arrays
[System.Runtime.CompilerServices.InlineArray(10)]
public struct Buffer
{
    private int _element0;
}

// 4. Optional Lambda Parameters
var add = (int x, int y = 10) => x + y;
Console.WriteLine(add(5));    // 15
Console.WriteLine(add(5, 3)); // 8

// 5. Alias Any Type
using Point = (int X, int Y);
Point p = (10, 20);
Console.WriteLine($"X={p.X}, Y={p.Y}");
```

### Features ใหม่ใน C# 13 (.NET 9)

```csharp
// 1. params Collections
void PrintAll(params IEnumerable<string> items)
{
    foreach (var item in items) Console.WriteLine(item);
}
PrintAll("Hello", "World", "C#");

// 2. New Lock Type
private readonly System.Threading.Lock _lock = new();
using (_lock.EnterScope())
{
    // thread-safe code
}

// 3. New Escape Sequence \e
char esc = '\e'; // ESC character (ASCII 27)

// 4. Method group natural type improvements
Func<int, int> doubler = x => x * 2; // cleaner

// 5. Partial Properties
public partial class MyClass
{
    public partial string Name { get; set; }
}
```

---

## Step 7: .NET CLI Commands ที่ต้องรู้

```bash
# ============ สร้างโปรเจค ============
dotnet new console          # Console App
dotnet new webapi           # Web API
dotnet new mvc              # MVC Web App
dotnet new blazorwasm       # Blazor WebAssembly
dotnet new blazorserver     # Blazor Server
dotnet new classlib         # Class Library
dotnet new xunit            # Unit Test (xUnit)
dotnet new nunit            # Unit Test (NUnit)
dotnet new sln              # Solution

# ดู templates ทั้งหมด
dotnet new list

# ============ Build และ Run ============
dotnet build                # Build โปรเจค
dotnet run                  # Build และ Run
dotnet run --project ./MyApp/MyApp.csproj  # Run specific project
dotnet watch run            # Hot Reload

# ============ Test ============
dotnet test                 # รัน Unit Tests
dotnet test --verbose       # แสดงผล verbose
dotnet test --filter "Category=Unit"  # Filter tests

# ============ Publish ============
dotnet publish -c Release -o ./publish  # Publish
dotnet publish -r win-x64 --self-contained  # Self-contained
dotnet publish -r linux-x64 -p:PublishSingleFile=true  # Single file

# ============ NuGet ============
dotnet add package Newtonsoft.Json              # เพิ่ม package
dotnet add package Microsoft.EntityFrameworkCore --version 9.0.0
dotnet remove package Newtonsoft.Json           # ลบ package
dotnet list package                             # ดู packages
dotnet restore                                  # Restore packages

# ============ Tools ============
dotnet tool install --global dotnet-ef          # EF Core tools
dotnet tool install --global dotnet-format      # Code formatter
dotnet tool list --global                       # ดู tools
dotnet format                                   # Format code
```

---

## Step 8: ความเข้าใจ Top-level Statements

### รูปแบบเก่า (Pre-.NET 6)

```csharp
// รูปแบบเก่าที่ต้องเขียน namespace, class, method
using System;

namespace HelloWorld
{
    class Program
    {
        static void Main(string[] args)
        {
            Console.WriteLine("Hello, World!");
        }
    }
}
```

### รูปแบบใหม่ (Top-level Statements - .NET 6+)

```csharp
// Program.cs - ไม่ต้องมี namespace, class, Main method
Console.WriteLine("Hello, World!");

// ยังสามารถใช้ args ได้
if (args.Length > 0)
{
    Console.WriteLine($"Arguments: {string.Join(", ", args)}");
}

// สามารถประกาศ method ได้
Greet("สวัสดี");
Greet("Hello");

void Greet(string message)
{
    Console.WriteLine($"[{DateTime.Now:HH:mm:ss}] {message}");
}

// สามารถประกาศ class ได้
var person = new Person("Alice", 30);
Console.WriteLine(person);

class Person
{
    public string Name { get; }
    public int Age { get; }
    
    public Person(string name, int age)
    {
        Name = name;
        Age = age;
    }
    
    public override string ToString() => $"{Name} ({Age})";
}
```

### ทั้งสองรูปแบบทำงานเหมือนกัน แต่ Top-level สะดวกกว่า

```bash
# รัน และส่ง arguments
dotnet run -- arg1 arg2 arg3
```

---

## Step 9: การ Debug ใน VS Code

### launch.json (ตั้งค่า Debug)

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": ".NET Core Launch (console)",
            "type": "coreclr",
            "request": "launch",
            "preLaunchTask": "build",
            "program": "${workspaceFolder}/bin/Debug/net9.0/HelloWorld.dll",
            "args": [],
            "cwd": "${workspaceFolder}",
            "console": "internalConsole",
            "stopAtEntry": false
        },
        {
            "name": ".NET Core Attach",
            "type": "coreclr",
            "request": "attach"
        }
    ]
}
```

### tasks.json (Build Task)

```json
{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "build",
            "command": "dotnet",
            "type": "process",
            "args": [
                "build",
                "${workspaceFolder}/HelloWorld.csproj",
                "/property:GenerateFullPaths=true",
                "/consoleloggerparameters:NoSummary;ForceNoAlign"
            ],
            "problemMatcher": "$msCompile"
        },
        {
            "label": "watch",
            "command": "dotnet",
            "type": "process",
            "args": [
                "watch",
                "run",
                "--project",
                "${workspaceFolder}/HelloWorld.csproj"
            ],
            "problemMatcher": "$msCompile",
            "isBackground": true
        }
    ]
}
```

### Breakpoints และ Debug

```csharp
// Program.cs - ตัวอย่างการ Debug
int x = 10;
int y = 20;
int result = Add(x, y);  // วาง Breakpoint ที่บรรทัดนี้ (F9)

Console.WriteLine($"Result: {result}");

int Add(int a, int b)
{
    // Step Into (F11) จะเข้ามาที่นี่
    var sum = a + b;  // ดูค่า variables ใน Debug panel
    return sum;
}

// เทคนิค Debug:
// F5  = Start Debugging
// F9  = Toggle Breakpoint
// F10 = Step Over
// F11 = Step Into
// Shift+F11 = Step Out
// F5  = Continue (ขณะ Debug)
```

---

## Step 10: โปรแกรมแรกที่สมบูรณ์

```csharp
// Program.cs - โปรแกรมแนะนำตัว
using System;

// ============ ข้อมูลผู้ใช้ ============
Console.Write("กรุณาใส่ชื่อของคุณ: ");
string name = Console.ReadLine() ?? "ผู้ใช้";

Console.Write("กรุณาใส่อายุของคุณ: ");
string ageInput = Console.ReadLine() ?? "0";
int age = int.TryParse(ageInput, out int parsedAge) ? parsedAge : 0;

// ============ แสดงผล ============
Console.Clear();
Console.ForegroundColor = ConsoleColor.Cyan;
Console.WriteLine("╔══════════════════════════════╗");
Console.WriteLine("║     ยินดีต้อนรับสู่ C#!       ║");
Console.WriteLine("╚══════════════════════════════╝");
Console.ResetColor();

Console.WriteLine();
Console.ForegroundColor = ConsoleColor.Yellow;
Console.WriteLine($"👋 สวัสดี, {name}!");
Console.ResetColor();

Console.WriteLine($"📅 อายุ: {age} ปี");
Console.WriteLine($"🎂 เกิดปี: {DateTime.Now.Year - age} (โดยประมาณ)");
Console.WriteLine($"🕐 เวลาปัจจุบัน: {DateTime.Now:dd/MM/yyyy HH:mm:ss}");
Console.WriteLine($"💻 .NET Version: {Environment.Version}");
Console.WriteLine($"🖥️  OS: {Environment.OSVersion}");

// ============ แสดงข้อความต้อนรับ ============
Console.WriteLine();
if (age < 18)
{
    Console.ForegroundColor = ConsoleColor.Green;
    Console.WriteLine("🌱 คุณยังเด็ก เริ่มเรียน C# ได้เลย!");
}
else if (age < 30)
{
    Console.ForegroundColor = ConsoleColor.Blue;
    Console.WriteLine("🚀 วัยเยาว์เป็นช่วงเวลาที่ดีที่สุดในการเรียนรู้!");
}
else
{
    Console.ForegroundColor = ConsoleColor.Magenta;
    Console.WriteLine("💪 ไม่มีคำว่าสายเกินไปสำหรับการเรียน C#!");
}
Console.ResetColor();

Console.WriteLine();
Console.WriteLine("🎯 สิ่งที่คุณจะได้เรียนในหลักสูตรนี้:");
string[] topics = {
    "✅ C# Fundamentals",
    "✅ Object-Oriented Programming",
    "✅ LINQ & Functional Programming",
    "✅ Async/Await Programming",
    "✅ ASP.NET Core Web API",
    "✅ Entity Framework Core",
    "✅ Design Patterns",
    "✅ Cloud & Microservices"
};

foreach (string topic in topics)
{
    Console.WriteLine($"   {topic}");
    System.Threading.Thread.Sleep(100); // ทำให้ดู dynamic
}

Console.WriteLine();
Console.ForegroundColor = ConsoleColor.Green;
Console.WriteLine("🎉 พร้อมแล้ว! ไปต่อที่ Part 02 กันเลย!");
Console.ResetColor();
Console.WriteLine();
Console.WriteLine("กด Enter เพื่อออก...");
Console.ReadLine();
```

### รันและทดสอบ

```bash
# สร้างโปรเจค
dotnet new console -n WelcomeApp
cd WelcomeApp

# แทนที่ไฟล์ Program.cs ด้วยโค้ดด้านบน
# จากนั้นรัน

dotnet run
```

---

## สรุป Part 01

```
✅ Step 1:  เข้าใจ C# คืออะไรและประโยชน์
✅ Step 2:  ติดตั้ง .NET SDK สำเร็จ
✅ Step 3:  ติดตั้ง IDE (VS Code/VS 2022/Rider)
✅ Step 4:  สร้างและรัน Hello World
✅ Step 5:  เข้าใจโครงสร้างโปรเจค .NET
✅ Step 6:  รู้จัก C# Version features
✅ Step 7:  ใช้ .NET CLI commands
✅ Step 8:  เข้าใจ Top-level Statements
✅ Step 9:  Debug ใน VS Code
✅ Step 10: โปรแกรมแรกที่สมบูรณ์
```

## แบบฝึกหัด Part 01

1. ติดตั้ง .NET SDK และตรวจสอบด้วย `dotnet --version`
2. สร้างโปรเจค console ใหม่ชื่อ `MyFirstApp`
3. แก้ไข Program.cs ให้แสดงชื่อและวันเกิดของคุณ
4. รัน `dotnet watch run` และแก้ไขโค้ด ดู Hot Reload
5. สร้าง Solution และเพิ่ม 2 projects เข้าไป

---

**ถัดไป: [Part 02 - ตัวแปรและชนิดข้อมูล →](part-02-variables-datatypes.md)**
