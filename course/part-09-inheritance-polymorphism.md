# Part 09: Inheritance และ Polymorphism
## Steps 211-250: การสืบทอดและความหลากหลายใน C#

---

## Step 211: Inheritance พื้นฐาน

```csharp
// ============ Base Class ============
class Animal
{
    public string Name { get; protected set; }
    public string Species { get; protected set; }
    public int Age { get; protected set; }
    protected double Weight;
    
    public Animal(string name, string species, int age, double weight)
    {
        Name = name;
        Species = species;
        Age = age;
        Weight = weight;
    }
    
    // Virtual methods - สามารถ override ได้ใน subclass
    public virtual void MakeSound()
    {
        Console.WriteLine($"{Name} makes a generic sound");
    }
    
    public virtual void Move()
    {
        Console.WriteLine($"{Name} moves");
    }
    
    public virtual string Describe()
    {
        return $"{Name} ({Species}), Age: {Age}, Weight: {Weight}kg";
    }
    
    // Non-virtual - ไม่สามารถ override ได้
    public void Breathe()
    {
        Console.WriteLine($"{Name} breathes");
    }
    
    public override string ToString() => Describe();
}

// ============ Derived Classes ============
class Dog : Animal
{
    public string Breed { get; }
    public bool IsVaccinated { get; set; }
    
    public Dog(string name, string breed, int age, double weight, bool vaccinated = false)
        : base(name, "Canis lupus familiaris", age, weight)
    {
        Breed = breed;
        IsVaccinated = vaccinated;
    }
    
    // Override: เปลี่ยนพฤติกรรม
    public override void MakeSound()
    {
        Console.WriteLine($"{Name} says: Woof! Woof!");
    }
    
    public override void Move()
    {
        Console.WriteLine($"{Name} runs with 4 legs");
    }
    
    public override string Describe()
    {
        return base.Describe() + $", Breed: {Breed}, Vaccinated: {IsVaccinated}";
    }
    
    // New method specific to Dog
    public void Fetch()
    {
        Console.WriteLine($"{Name} fetches the ball! 🐾");
    }
}

class Cat : Animal
{
    public bool IsIndoor { get; }
    
    public Cat(string name, int age, double weight, bool isIndoor = true)
        : base(name, "Felis catus", age, weight)
    {
        IsIndoor = isIndoor;
    }
    
    public override void MakeSound() => Console.WriteLine($"{Name} says: Meow~");
    public override void Move() => Console.WriteLine($"{Name} moves gracefully");
    
    public void Purr() => Console.WriteLine($"{Name} purrs contentedly 😸");
}

class Bird : Animal
{
    public double WingSpan { get; }
    public bool CanFly { get; }
    
    public Bird(string name, string species, int age, double weight, double wingspan, bool canFly = true)
        : base(name, species, age, weight)
    {
        WingSpan = wingspan;
        CanFly = canFly;
    }
    
    public override void MakeSound() => Console.WriteLine($"{Name} chirps: Tweet tweet!");
    
    public override void Move()
    {
        if (CanFly)
            Console.WriteLine($"{Name} flies with wingspan {WingSpan}m");
        else
            Console.WriteLine($"{Name} walks (cannot fly)");
    }
}

// ============ Using Inheritance ============
var animals = new List<Animal>
{
    new Dog("Buddy", "Golden Retriever", 3, 30, true),
    new Dog("Rex", "German Shepherd", 5, 40),
    new Cat("Whiskers", 2, 4.5),
    new Bird("Tweety", "Canary", 1, 0.02, 0.15),
    new Bird("Penguin Pete", "Emperor Penguin", 4, 25, 1.2, false),
};

Console.WriteLine("=== All Animals ===");
foreach (var animal in animals)
{
    Console.WriteLine(animal.Describe());
    animal.MakeSound();
    animal.Move();
    Console.WriteLine();
}

// ============ Type Checking ============
foreach (var animal in animals)
{
    if (animal is Dog dog)
    {
        dog.Fetch();
        Console.WriteLine($"  Dog breed: {dog.Breed}");
    }
    else if (animal is Cat cat)
    {
        cat.Purr();
    }
    else if (animal is Bird bird)
    {
        Console.WriteLine($"  Can fly: {bird.CanFly}");
    }
}
```

---

## Step 212: Polymorphism

```csharp
// ============ Polymorphism ============
// เรียกใช้ method ผ่าน base class reference แต่ได้พฤติกรรมของ derived class

abstract class Shape
{
    public string Color { get; set; } = "Black";
    public abstract double Area { get; }
    public abstract double Perimeter { get; }
    public abstract void Draw();
    
    // Template method pattern
    public void Print()
    {
        Console.WriteLine($"{GetType().Name}:");
        Console.WriteLine($"  Color: {Color}");
        Console.WriteLine($"  Area: {Area:F4}");
        Console.WriteLine($"  Perimeter: {Perimeter:F4}");
        Draw();
    }
}

class Circle : Shape
{
    public double Radius { get; }
    
    public Circle(double radius, string color = "Black")
    {
        Radius = radius;
        Color = color;
    }
    
    public override double Area => Math.PI * Radius * Radius;
    public override double Perimeter => 2 * Math.PI * Radius;
    
    public override void Draw()
    {
        Console.WriteLine($"  Drawing circle with radius {Radius}");
        // ASCII art
        int r = Math.Min((int)(Radius / 2), 5);
        for (int y = -r; y <= r; y++)
        {
            for (int x = -r * 2; x <= r * 2; x++)
            {
                double dist = Math.Sqrt((double)(x * x) / 4 + y * y);
                Console.Write(Math.Abs(dist - r) < 0.7 ? "O" : " ");
            }
            Console.WriteLine();
        }
    }
}

class Rectangle : Shape
{
    public double Width { get; }
    public double Height { get; }
    
    public Rectangle(double width, double height, string color = "Black")
    {
        Width = width;
        Height = height;
        Color = color;
    }
    
    public override double Area => Width * Height;
    public override double Perimeter => 2 * (Width + Height);
    
    public override void Draw()
    {
        Console.WriteLine($"  Drawing rectangle {Width}x{Height}");
        int w = Math.Min((int)Width, 20);
        int h = Math.Min((int)Height, 10);
        for (int y = 0; y < h; y++)
        {
            for (int x = 0; x < w; x++)
                Console.Write(y == 0 || y == h - 1 || x == 0 || x == w - 1 ? "#" : ".");
            Console.WriteLine();
        }
    }
}

class Triangle : Shape
{
    public double A { get; }
    public double B { get; }
    public double C { get; }
    
    public Triangle(double a, double b, double c, string color = "Black")
    {
        // Validate triangle inequality
        if (a + b <= c || b + c <= a || a + c <= b)
            throw new ArgumentException("Invalid triangle");
        A = a; B = b; C = c;
        Color = color;
    }
    
    public override double Perimeter => A + B + C;
    
    public override double Area
    {
        get
        {
            double s = Perimeter / 2;
            return Math.Sqrt(s * (s - A) * (s - B) * (s - C)); // Heron's formula
        }
    }
    
    public override void Draw()
    {
        Console.WriteLine($"  Drawing triangle with sides {A}, {B}, {C}");
        int h = 6;
        for (int y = 0; y < h; y++)
        {
            int spaces = h - y - 1;
            int stars = 2 * y + 1;
            Console.Write(new string(' ', spaces));
            Console.WriteLine(y == h - 1 ? new string('*', stars) :
                              y == 0 ? "*" : "*" + new string('.', stars - 2) + "*");
        }
    }
}

// ============ Polymorphic behavior ============
var shapes = new List<Shape>
{
    new Circle(5, "Red"),
    new Rectangle(8, 4, "Blue"),
    new Triangle(3, 4, 5, "Green"),
    new Circle(3),
    new Rectangle(6, 6),
};

// Polymorphism: เรียก Shape method แต่ได้พฤติกรรมของ derived class
double totalArea = 0;
foreach (Shape shape in shapes)
{
    shape.Print();
    totalArea += shape.Area;
    Console.WriteLine();
}
Console.WriteLine($"Total area: {totalArea:F2}");

// Sort by area
var sorted = shapes.OrderByDescending(s => s.Area);
Console.WriteLine("\nSorted by area:");
foreach (var s in sorted)
    Console.WriteLine($"  {s.GetType().Name}: {s.Area:F2}");
```

---

## Step 213: Abstract Classes และ Methods

```csharp
// ============ Abstract Class ============
// ไม่สามารถสร้าง instance ได้ แต่มี partial implementation

abstract class Vehicle
{
    // Properties
    public string Make { get; protected set; }
    public string Model { get; protected set; }
    public int Year { get; protected set; }
    public decimal Price { get; protected set; }
    
    // Abstract properties - ต้อง implement ใน subclass
    public abstract string FuelType { get; }
    public abstract int MaxSpeed { get; } // km/h
    
    protected Vehicle(string make, string model, int year, decimal price)
    {
        Make = make;
        Model = model;
        Year = year;
        Price = price;
    }
    
    // Abstract methods - ต้อง implement ใน subclass
    public abstract void StartEngine();
    public abstract double CalculateFuelCost(double distanceKm);
    
    // Virtual method - สามารถ override หรือใช้ default ได้
    public virtual string GetInfo()
    {
        return $"{Year} {Make} {Model} ({FuelType}) - ฿{Price:N0}";
    }
    
    // Non-abstract method - มี implementation
    public void Honk() => Console.WriteLine("Beep beep!");
    
    public string GetAgeCategory()
    {
        int age = DateTime.Now.Year - Year;
        return age switch
        {
            0 => "New",
            <= 3 => "Recent",
            <= 7 => "Used",
            _ => "Old"
        };
    }
}

class GasolineCar : Vehicle
{
    public double EngineSize { get; }  // Liters
    public double FuelEfficiency { get; }  // km per liter
    
    public GasolineCar(string make, string model, int year, decimal price, 
                        double engineSize, double fuelEfficiency)
        : base(make, model, year, price)
    {
        EngineSize = engineSize;
        FuelEfficiency = fuelEfficiency;
    }
    
    public override string FuelType => "Gasoline";
    public override int MaxSpeed => (int)(180 + EngineSize * 20);
    
    public override void StartEngine()
        => Console.WriteLine($"{Make} {Model}: Vroom! (Gasoline engine started)");
    
    public override double CalculateFuelCost(double distanceKm)
    {
        double litersNeeded = distanceKm / FuelEfficiency;
        double gasolinePrice = 42.0; // baht per liter
        return litersNeeded * gasolinePrice;
    }
    
    public override string GetInfo()
        => base.GetInfo() + $", Engine: {EngineSize}L, {FuelEfficiency}km/L";
}

class ElectricCar : Vehicle
{
    public double BatteryCapacity { get; }  // kWh
    public double RangeKm { get; }
    
    public ElectricCar(string make, string model, int year, decimal price,
                        double batteryKwh, double rangeKm)
        : base(make, model, year, price)
    {
        BatteryCapacity = batteryKwh;
        RangeKm = rangeKm;
    }
    
    public override string FuelType => "Electric";
    public override int MaxSpeed => 250;
    
    public override void StartEngine()
        => Console.WriteLine($"{Make} {Model}: *whirr* (Electric motor started silently)");
    
    public override double CalculateFuelCost(double distanceKm)
    {
        double kwhNeeded = distanceKm / (RangeKm / BatteryCapacity);
        double electricityPrice = 4.5; // baht per kWh
        return kwhNeeded * electricityPrice;
    }
    
    public override string GetInfo()
        => base.GetInfo() + $", Battery: {BatteryCapacity}kWh, Range: {RangeKm}km";
    
    public void Charge() => Console.WriteLine($"Charging {Make} {Model}...");
}

class HybridCar : GasolineCar
{
    public double ElectricRange { get; }
    
    public HybridCar(string make, string model, int year, decimal price,
                      double engineSize, double fuelEfficiency, double electricRange)
        : base(make, model, year, price, engineSize, fuelEfficiency)
    {
        ElectricRange = electricRange;
    }
    
    public override string FuelType => "Hybrid";
    
    public override void StartEngine()
        => Console.WriteLine($"{Make} {Model}: Electric mode first, then hybrid");
    
    public override double CalculateFuelCost(double distanceKm)
    {
        // Electric for first ElectricRange km, then gasoline
        double electricKm = Math.Min(distanceKm, ElectricRange);
        double gasolineKm = Math.Max(0, distanceKm - ElectricRange);
        
        double electricCost = electricKm * 1.5; // cheaper
        double gasolineCost = base.CalculateFuelCost(gasolineKm);
        
        return electricCost + gasolineCost;
    }
}

// ============ Using Abstract Classes ============
var vehicles = new List<Vehicle>
{
    new GasolineCar("Toyota", "Camry", 2024, 1350000, 2.5, 15),
    new GasolineCar("Honda", "Civic", 2023, 920000, 1.5, 18),
    new ElectricCar("Tesla", "Model 3", 2024, 1900000, 75, 570),
    new ElectricCar("BYD", "Seal", 2024, 1100000, 82, 650),
    new HybridCar("Toyota", "Prius", 2024, 1450000, 1.8, 22, 50),
};

Console.WriteLine("=== Vehicle Fleet ===");
foreach (var v in vehicles)
{
    Console.WriteLine(v.GetInfo());
    v.StartEngine();
    
    double distance = 200; // km
    double cost = v.CalculateFuelCost(distance);
    Console.WriteLine($"  Cost for {distance}km: ฿{cost:N2}");
    Console.WriteLine();
}

// Polymorphic comparison
Console.WriteLine("Most economical for 500km:");
var ranked = vehicles.OrderBy(v => v.CalculateFuelCost(500));
foreach (var v in ranked)
    Console.WriteLine($"  {v.Make} {v.Model}: ฿{v.CalculateFuelCost(500):N2}");
```

---

## Step 214: Method Overriding และ new keyword

```csharp
// ============ override vs new ============

class Base
{
    public virtual void VirtualMethod()
        => Console.WriteLine("Base.VirtualMethod()");
    
    public void RegularMethod()
        => Console.WriteLine("Base.RegularMethod()");
}

class DerivedA : Base
{
    // override: ใช้ virtual dispatch (polymorphism)
    public override void VirtualMethod()
        => Console.WriteLine("DerivedA.VirtualMethod() [override]");
    
    // new: ซ่อน base method (method hiding) - ไม่ใช่ polymorphism!
    public new void RegularMethod()
        => Console.WriteLine("DerivedA.RegularMethod() [new - hiding]");
}

class DerivedB : Base
{
    public override void VirtualMethod()
    {
        Console.Write("DerivedB calls base: ");
        base.VirtualMethod(); // เรียก Base.VirtualMethod
        Console.WriteLine("DerivedB.VirtualMethod() [override with base call]");
    }
}

// ============ Polymorphism Demo ============
Base b1 = new DerivedA();
Base b2 = new DerivedB();
DerivedA d1 = new DerivedA();

Console.WriteLine("Through Base reference:");
b1.VirtualMethod();    // DerivedA.VirtualMethod() - polymorphism
b1.RegularMethod();    // Base.RegularMethod() - ไม่ใช่ polymorphism!

Console.WriteLine("\nThrough DerivedA reference:");
d1.VirtualMethod();    // DerivedA.VirtualMethod()
d1.RegularMethod();    // DerivedA.RegularMethod() - method hiding

Console.WriteLine("\nDerivedB:");
b2.VirtualMethod();    // DerivedB.VirtualMethod with base call

// ============ sealed override ============
class Sealed : DerivedA
{
    // sealed: ป้องกันไม่ให้ subclass override ต่อ
    public sealed override void VirtualMethod()
        => Console.WriteLine("Sealed.VirtualMethod() [sealed - cannot override further]");
}

// ❌ class FurtherDerived : Sealed
// ❌ {
// ❌     public override void VirtualMethod() { } // Error!
// ❌ }
```

---

## Step 215-220: Interface-based Polymorphism (Preview)

```csharp
// ============ Polymorphism ผ่าน Interface ============
interface IDrawable
{
    void Draw();
    void Resize(double factor);
}

interface IColorable
{
    string Color { get; set; }
    void ChangeColor(string color);
}

interface IShape : IDrawable, IColorable
{
    double Area { get; }
    double Perimeter { get; }
    (double X, double Y) Center { get; }
}

class ColoredCircle : IShape
{
    public double Radius { get; private set; }
    public string Color { get; set; } = "Black";
    public (double X, double Y) Center { get; private set; }
    
    public ColoredCircle(double radius, double x, double y, string color = "Black")
    {
        Radius = radius;
        Center = (x, y);
        Color = color;
    }
    
    public double Area => Math.PI * Radius * Radius;
    public double Perimeter => 2 * Math.PI * Radius;
    
    public void Draw()
        => Console.WriteLine($"Drawing {Color} circle at ({Center.X},{Center.Y}), r={Radius:F2}");
    
    public void Resize(double factor)
    {
        Radius *= factor;
        Console.WriteLine($"Circle resized to radius {Radius:F2}");
    }
    
    public void ChangeColor(string color)
    {
        Color = color;
        Console.WriteLine($"Circle color changed to {color}");
    }
}

// ============ Working with multiple interfaces ============
var shapes = new List<IShape>
{
    new ColoredCircle(5, 0, 0, "Red"),
    new ColoredCircle(3, 10, 5, "Blue"),
};

double totalArea = shapes.Sum(s => s.Area);
Console.WriteLine($"Total area: {totalArea:F2}");

foreach (IShape shape in shapes)
{
    shape.Draw();
    shape.Resize(1.5);
    shape.ChangeColor("Purple");
}
```

---

## Step 221-250: โปรแกรม Employee Hierarchy System

```csharp
// ============ Employee Hierarchy ============

abstract class Employee
{
    public int Id { get; }
    public string Name { get; set; }
    public string Email { get; set; }
    public string Department { get; set; }
    public DateTime HireDate { get; }
    protected static int NextId = 1;
    
    protected Employee(string name, string email, string department)
    {
        Id = NextId++;
        Name = name;
        Email = email;
        Department = department;
        HireDate = DateTime.Now;
    }
    
    // Abstract - ต้อง implement
    public abstract decimal CalculateMonthlySalary();
    public abstract string JobTitle { get; }
    
    // Virtual - สามารถ override
    public virtual decimal CalculateBonus() => CalculateMonthlySalary() * 0.1m;
    
    public virtual string GetSummary()
    {
        return $"[{Id}] {Name} | {JobTitle} | {Department} | " +
               $"Salary: ฿{CalculateMonthlySalary():N0}";
    }
    
    // Non-virtual - ใช้ร่วมกัน
    public int YearsOfService => (DateTime.Now - HireDate).Days / 365;
    
    public override string ToString() => GetSummary();
}

class FullTimeEmployee : Employee
{
    public decimal BaseSalary { get; private set; }
    private List<string> _benefits = new();
    
    public FullTimeEmployee(string name, string email, string department, decimal salary)
        : base(name, email, department)
    {
        BaseSalary = salary;
    }
    
    public override string JobTitle => "Full-time Employee";
    
    public override decimal CalculateMonthlySalary()
    {
        decimal salary = BaseSalary;
        // Years of service bonus
        salary += YearsOfService * 500;
        return salary;
    }
    
    public override decimal CalculateBonus() => BaseSalary * 0.15m;
    
    public void AddBenefit(string benefit) => _benefits.Add(benefit);
    
    public void GiveRaise(decimal amount)
    {
        BaseSalary += amount;
        Console.WriteLine($"{Name} got a raise of ฿{amount:N0}. New salary: ฿{BaseSalary:N0}");
    }
    
    public override string GetSummary()
        => base.GetSummary() + $" | Benefits: {string.Join(", ", _benefits.DefaultIfEmpty("None"))}";
}

class PartTimeEmployee : Employee
{
    public decimal HourlyRate { get; }
    public int HoursPerMonth { get; set; }
    
    public PartTimeEmployee(string name, string email, string department, 
                             decimal hourlyRate, int hoursPerMonth)
        : base(name, email, department)
    {
        HourlyRate = hourlyRate;
        HoursPerMonth = hoursPerMonth;
    }
    
    public override string JobTitle => "Part-time Employee";
    public override decimal CalculateMonthlySalary() => HourlyRate * HoursPerMonth;
    public override decimal CalculateBonus() => CalculateMonthlySalary() * 0.05m;
    
    public override string GetSummary()
        => base.GetSummary() + $" | {HoursPerMonth}hrs/mo @ ฿{HourlyRate}/hr";
}

class Manager : FullTimeEmployee
{
    public List<Employee> DirectReports { get; } = new();
    public string ManagementLevel { get; }
    
    public Manager(string name, string email, string department, decimal salary, string level)
        : base(name, email, department, salary)
    {
        ManagementLevel = level;
    }
    
    public override string JobTitle => $"{ManagementLevel} Manager";
    
    public override decimal CalculateMonthlySalary()
    {
        decimal salary = base.CalculateMonthlySalary();
        // Management bonus per report
        salary += DirectReports.Count * 2000;
        return salary;
    }
    
    public override decimal CalculateBonus() => BaseSalary * 0.25m;
    
    public void AddReport(Employee emp)
    {
        DirectReports.Add(emp);
        Console.WriteLine($"{emp.Name} now reports to {Name}");
    }
    
    public override string GetSummary()
        => base.GetSummary() + $" | Reports: {DirectReports.Count}";
    
    public void PrintTeam(int indent = 0)
    {
        string prefix = new string(' ', indent * 2);
        Console.WriteLine($"{prefix}👤 {Name} ({JobTitle}) - ฿{CalculateMonthlySalary():N0}");
        foreach (var report in DirectReports)
        {
            if (report is Manager mgr)
                mgr.PrintTeam(indent + 1);
            else
                Console.WriteLine($"{prefix}  └─ {report.Name} ({report.JobTitle}) - ฿{report.CalculateMonthlySalary():N0}");
        }
    }
}

class Contractor : Employee
{
    public decimal DailyRate { get; }
    public int DaysWorkedThisMonth { get; set; }
    public string Agency { get; }
    
    public Contractor(string name, string email, string department, 
                       decimal dailyRate, string agency)
        : base(name, email, department)
    {
        DailyRate = dailyRate;
        Agency = agency;
        DaysWorkedThisMonth = 22; // default working days
    }
    
    public override string JobTitle => "Contractor";
    
    public override decimal CalculateMonthlySalary()
        => DailyRate * DaysWorkedThisMonth;
    
    public override decimal CalculateBonus() => 0; // Contractors don't get bonus
    
    public override string GetSummary()
        => base.GetSummary() + $" | Agency: {Agency} | Days: {DaysWorkedThisMonth}";
}

// ============ HR System ============
class HRSystem
{
    private List<Employee> _employees = new();
    
    public void Hire(Employee emp)
    {
        _employees.Add(emp);
        Console.WriteLine($"✅ Hired: {emp.Name} as {emp.JobTitle}");
    }
    
    public void PrintPayroll()
    {
        Console.WriteLine("\n📊 MONTHLY PAYROLL REPORT");
        Console.WriteLine(new string('═', 70));
        
        decimal totalPayroll = 0;
        decimal totalBonus = 0;
        
        var grouped = _employees.GroupBy(e => e.Department).OrderBy(g => g.Key);
        
        foreach (var dept in grouped)
        {
            Console.WriteLine($"\n🏢 {dept.Key}:");
            Console.WriteLine(new string('─', 60));
            
            foreach (var emp in dept.OrderBy(e => e.Name))
            {
                decimal salary = emp.CalculateMonthlySalary();
                decimal bonus = emp.CalculateBonus();
                totalPayroll += salary;
                totalBonus += bonus;
                
                Console.WriteLine($"  {emp.Name,-20} {emp.JobTitle,-25} ฿{salary,10:N0}");
            }
        }
        
        Console.WriteLine(new string('═', 70));
        Console.WriteLine($"  {'Total Payroll',-47} ฿{totalPayroll,10:N0}");
        Console.WriteLine($"  {'Total Bonus Pool',-47} ฿{totalBonus,10:N0}");
        Console.WriteLine($"  {'Grand Total',-47} ฿{totalPayroll + totalBonus,10:N0}");
        Console.WriteLine(new string('═', 70));
    }
    
    public void PrintStatsByType()
    {
        Console.WriteLine("\n📈 EMPLOYEE STATISTICS");
        
        var byType = _employees.GroupBy(e => e.GetType().Name);
        foreach (var group in byType)
        {
            decimal avgSalary = group.Average(e => e.CalculateMonthlySalary());
            Console.WriteLine($"\n{group.Key}: {group.Count()} employees");
            Console.WriteLine($"  Avg salary: ฿{avgSalary:N0}");
            Console.WriteLine($"  Total: ฿{group.Sum(e => e.CalculateMonthlySalary()):N0}");
        }
    }
    
    public IEnumerable<Employee> GetTopEarners(int count = 5)
        => _employees.OrderByDescending(e => e.CalculateMonthlySalary()).Take(count);
}

// ============ Demo ============
var hr = new HRSystem();

// Create managers
var ceo = new Manager("Sarah Chen", "sarah@company.com", "Executive", 200000, "Chief");
var cto = new Manager("David Kim", "david@company.com", "Technology", 150000, "Senior");
var cfo = new Manager("Emma Wilson", "emma@company.com", "Finance", 140000, "Senior");
var devMgr = new Manager("Mike Johnson", "mike@company.com", "Technology", 100000, "Mid");

// Create employees
var dev1 = new FullTimeEmployee("Alice Wong", "alice@company.com", "Technology", 75000);
var dev2 = new FullTimeEmployee("Bob Smith", "bob@company.com", "Technology", 70000);
var dev3 = new PartTimeEmployee("Charlie Lee", "charlie@company.com", "Technology", 500, 60);
var contractor1 = new Contractor("Dave Brown", "dave@company.com", "Technology", 3000, "TechStaff Agency");
var analyst = new FullTimeEmployee("Eve Martinez", "eve@company.com", "Finance", 65000);

// Benefits
dev1.AddBenefit("Health Insurance");
dev1.AddBenefit("Stock Options");
dev2.AddBenefit("Health Insurance");

// Setup hierarchy
ceo.AddReport(cto);
ceo.AddReport(cfo);
cto.AddReport(devMgr);
devMgr.AddReport(dev1);
devMgr.AddReport(dev2);
devMgr.AddReport(dev3);
cto.AddReport(contractor1);
cfo.AddReport(analyst);

// Hire all
foreach (var emp in new Employee[] { ceo, cto, cfo, devMgr, dev1, dev2, dev3, contractor1, analyst })
    hr.Hire(emp);

// Show org chart
Console.WriteLine("\n🏢 Organization Chart:");
ceo.PrintTeam();

// Payroll
hr.PrintPayroll();
hr.PrintStatsByType();

// Top earners
Console.WriteLine("\n🏆 Top Earners:");
foreach (var emp in hr.GetTopEarners(3))
    Console.WriteLine($"  {emp.Name}: ฿{emp.CalculateMonthlySalary():N0}");
```

---

## สรุป Part 09

```
✅ Step 211: Inheritance พื้นฐาน (Base/Derived classes)
✅ Step 212: Polymorphism ด้วย abstract classes
✅ Step 213: Abstract Classes และ Methods
✅ Step 214: Method Overriding, sealed, new keyword
✅ Steps 215-220: Interface-based Polymorphism
✅ Steps 221-250: Employee Hierarchy System สมบูรณ์
```

## แบบฝึกหัด Part 09

1. สร้าง `PaymentMethod` hierarchy: Cash, CreditCard, PromptPay, Crypto
2. Implement `Animal` hierarchy: Mammal, Bird, Fish, Reptile พร้อมพฤติกรรมเฉพาะ
3. สร้าง `Notification` system: Email, SMS, Push, Webhook
4. เขียน `Serializer` hierarchy รองรับ JSON, XML, CSV
5. สร้าง `TaxCalculator` สำหรับประเทศต่างๆ ด้วย polymorphism

---

**ถัดไป: [Part 10 - Interface และ Abstract Class →](part-10-interfaces-abstract.md)**
