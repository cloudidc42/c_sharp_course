# Part 25: ASP.NET Core MVC

## Steps 661-700: การสร้างเว็บแอปพลิเคชันด้วย ASP.NET Core MVC

---

## Step 661: MVC Pattern และโครงสร้างโปรเจค

```bash
# สร้าง MVC Project
dotnet new mvc -n MyMvcApp
cd MyMvcApp

# โครงสร้างโปรเจค
# Controllers/      - Controller classes
# Models/           - Data models และ ViewModels
# Views/            - Razor view files (.cshtml)
# wwwroot/          - Static files (CSS, JS, images)
# Views/Shared/     - Layout และ partial views
# appsettings.json  - Configuration
```

```csharp
// Program.cs - MVC Setup พื้นฐาน
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllersWithViews();
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("Default")));

var app = builder.Build();

if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/Home/Error");
    app.UseHsts();
}

app.UseHttpsRedirection();
app.UseStaticFiles();
app.UseRouting();
app.UseAuthentication();
app.UseAuthorization();

app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}");

app.MapControllerRoute(
    name: "areas",
    pattern: "{area:exists}/{controller=Home}/{action=Index}/{id?}");

app.Run();
```

---

## Step 662: Controllers พื้นฐาน

```csharp
// Controllers/HomeController.cs
using Microsoft.AspNetCore.Mvc;
using MyMvcApp.Models;
using System.Diagnostics;

namespace MyMvcApp.Controllers;

public class HomeController : Controller
{
    private readonly ILogger<HomeController> _logger;

    public HomeController(ILogger<HomeController> logger)
    {
        _logger = logger;
    }

    public IActionResult Index()
    {
        var model = new HomeViewModel
        {
            Title = "Welcome to My App",
            Message = "ASP.NET Core MVC Application"
        };
        return View(model);
    }

    public IActionResult About()
    {
        ViewData["Title"] = "About";
        ViewBag.Message = "Your application description page.";
        return View();
    }

    [ResponseCache(Duration = 0, Location = ResponseCacheLocation.None, NoStore = true)]
    public IActionResult Error()
    {
        return View(new ErrorViewModel 
        { 
            RequestId = Activity.Current?.Id ?? HttpContext.TraceIdentifier 
        });
    }
}
```

```csharp
// Controllers/ProductsController.cs
using Microsoft.AspNetCore.Mvc;
using MyMvcApp.Models;
using MyMvcApp.Services;

namespace MyMvcApp.Controllers;

public class ProductsController : Controller
{
    private readonly IProductService _productService;
    private readonly ILogger<ProductsController> _logger;

    public ProductsController(IProductService productService, ILogger<ProductsController> logger)
    {
        _productService = productService;
        _logger = logger;
    }

    // GET: /Products
    public async Task<IActionResult> Index(string? search, int page = 1)
    {
        var (products, total) = await _productService.GetPagedAsync(search, page, 10);
        
        var viewModel = new ProductListViewModel
        {
            Products = products,
            Search = search,
            CurrentPage = page,
            TotalPages = (int)Math.Ceiling(total / 10.0)
        };
        
        return View(viewModel);
    }

    // GET: /Products/Details/5
    public async Task<IActionResult> Details(int id)
    {
        var product = await _productService.GetByIdAsync(id);
        if (product is null)
            return NotFound();
        
        return View(product);
    }

    // GET: /Products/Create
    public IActionResult Create()
    {
        return View(new CreateProductViewModel());
    }

    // POST: /Products/Create
    [HttpPost]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> Create(CreateProductViewModel model)
    {
        if (!ModelState.IsValid)
            return View(model);

        await _productService.CreateAsync(model);
        TempData["Success"] = "Product created successfully!";
        return RedirectToAction(nameof(Index));
    }

    // GET: /Products/Edit/5
    public async Task<IActionResult> Edit(int id)
    {
        var product = await _productService.GetByIdAsync(id);
        if (product is null)
            return NotFound();

        var viewModel = new EditProductViewModel
        {
            Id = product.Id,
            Name = product.Name,
            Price = product.Price,
            Description = product.Description
        };
        
        return View(viewModel);
    }

    // POST: /Products/Edit/5
    [HttpPost]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> Edit(int id, EditProductViewModel model)
    {
        if (id != model.Id)
            return BadRequest();

        if (!ModelState.IsValid)
            return View(model);

        var success = await _productService.UpdateAsync(model);
        if (!success)
            return NotFound();

        TempData["Success"] = "Product updated successfully!";
        return RedirectToAction(nameof(Index));
    }

    // GET: /Products/Delete/5
    public async Task<IActionResult> Delete(int id)
    {
        var product = await _productService.GetByIdAsync(id);
        if (product is null)
            return NotFound();

        return View(product);
    }

    // POST: /Products/Delete/5
    [HttpPost, ActionName("Delete")]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> DeleteConfirmed(int id)
    {
        await _productService.DeleteAsync(id);
        TempData["Success"] = "Product deleted successfully!";
        return RedirectToAction(nameof(Index));
    }
}
```

---

## Step 663: Models และ ViewModels

```csharp
// Models/Product.cs
namespace MyMvcApp.Models;

public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string Description { get; set; } = string.Empty;
    public decimal Price { get; set; }
    public int Stock { get; set; }
    public string? ImageUrl { get; set; }
    public int CategoryId { get; set; }
    public Category? Category { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime UpdatedAt { get; set; }
    public bool IsActive { get; set; } = true;
}

// Models/Category.cs
public class Category
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string? Description { get; set; }
    public ICollection<Product> Products { get; set; } = new List<Product>();
}
```

```csharp
// Models/ViewModels/ProductListViewModel.cs
using System.ComponentModel.DataAnnotations;

namespace MyMvcApp.Models;

public class ProductListViewModel
{
    public IEnumerable<ProductSummary> Products { get; set; } = [];
    public string? Search { get; set; }
    public int CurrentPage { get; set; } = 1;
    public int TotalPages { get; set; }
    public bool HasPreviousPage => CurrentPage > 1;
    public bool HasNextPage => CurrentPage < TotalPages;
}

public class ProductSummary
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public decimal Price { get; set; }
    public int Stock { get; set; }
    public string? CategoryName { get; set; }
    public bool IsActive { get; set; }
}

// Models/ViewModels/CreateProductViewModel.cs
public class CreateProductViewModel
{
    [Required(ErrorMessage = "Product name is required")]
    [StringLength(200, MinimumLength = 2, ErrorMessage = "Name must be 2-200 characters")]
    [Display(Name = "Product Name")]
    public string Name { get; set; } = string.Empty;

    [StringLength(1000)]
    public string Description { get; set; } = string.Empty;

    [Required]
    [Range(0.01, 999999.99, ErrorMessage = "Price must be between 0.01 and 999,999.99")]
    [DataType(DataType.Currency)]
    public decimal Price { get; set; }

    [Required]
    [Range(0, int.MaxValue, ErrorMessage = "Stock cannot be negative")]
    public int Stock { get; set; }

    [Required]
    [Display(Name = "Category")]
    public int CategoryId { get; set; }

    public IEnumerable<CategoryOption> Categories { get; set; } = [];
}

// Models/ViewModels/EditProductViewModel.cs
public class EditProductViewModel : CreateProductViewModel
{
    public int Id { get; set; }
    public string? CurrentImageUrl { get; set; }
}

// Models/ViewModels/CategoryOption.cs
public class CategoryOption
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
}

// Models/ErrorViewModel.cs
public class ErrorViewModel
{
    public string? RequestId { get; set; }
    public bool ShowRequestId => !string.IsNullOrEmpty(RequestId);
}

// Models/HomeViewModel.cs
public class HomeViewModel
{
    public string Title { get; set; } = string.Empty;
    public string Message { get; set; } = string.Empty;
}
```

---

## Step 664: Razor Views - Layout

```html
<!-- Views/Shared/_Layout.cshtml -->
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>@ViewData["Title"] - MyMvcApp</title>
    <link rel="stylesheet" href="~/lib/bootstrap/dist/css/bootstrap.min.css" />
    <link rel="stylesheet" href="~/css/site.css" asp-append-version="true" />
    @await RenderSectionAsync("Styles", required: false)
</head>
<body>
    <header>
        <nav class="navbar navbar-expand-sm navbar-toggleable-sm navbar-light bg-white border-bottom box-shadow mb-3">
            <div class="container-fluid">
                <a class="navbar-brand" asp-area="" asp-controller="Home" asp-action="Index">MyMvcApp</a>
                <button class="navbar-toggler" type="button" data-bs-toggle="collapse" 
                    data-bs-target=".navbar-collapse">
                    <span class="navbar-toggler-icon"></span>
                </button>
                <div class="navbar-collapse collapse d-sm-inline-flex justify-content-between">
                    <ul class="navbar-nav flex-grow-1">
                        <li class="nav-item">
                            <a class="nav-link text-dark" asp-area="" asp-controller="Home" asp-action="Index">Home</a>
                        </li>
                        <li class="nav-item">
                            <a class="nav-link text-dark" asp-area="" asp-controller="Products" asp-action="Index">Products</a>
                        </li>
                    </ul>
                    <partial name="_LoginPartial" />
                </div>
            </div>
        </nav>
    </header>
    <div class="container">
        <main role="main" class="pb-3">
            @* Display TempData messages *@
            @if (TempData["Success"] != null)
            {
                <div class="alert alert-success alert-dismissible fade show" role="alert">
                    @TempData["Success"]
                    <button type="button" class="btn-close" data-bs-dismiss="alert"></button>
                </div>
            }
            @if (TempData["Error"] != null)
            {
                <div class="alert alert-danger alert-dismissible fade show" role="alert">
                    @TempData["Error"]
                    <button type="button" class="btn-close" data-bs-dismiss="alert"></button>
                </div>
            }
            @RenderBody()
        </main>
    </div>

    <footer class="border-top footer text-muted">
        <div class="container">
            &copy; @DateTime.Now.Year - MyMvcApp
        </div>
    </footer>
    <script src="~/lib/jquery/dist/jquery.min.js"></script>
    <script src="~/lib/bootstrap/dist/js/bootstrap.bundle.min.js"></script>
    <script src="~/js/site.js" asp-append-version="true"></script>
    @await RenderSectionAsync("Scripts", required: false)
</body>
</html>
```

```csharp
// Views/_ViewStart.cshtml
@{
    Layout = "_Layout";
}

// Views/_ViewImports.cshtml
@using MyMvcApp
@using MyMvcApp.Models
@addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers
```

---

## Step 665: Razor Views - Products Index

```html
<!-- Views/Products/Index.cshtml -->
@model ProductListViewModel

@{
    ViewData["Title"] = "Products";
}

<div class="row mb-3">
    <div class="col-md-8">
        <h1>Products</h1>
    </div>
    <div class="col-md-4 text-end">
        <a asp-action="Create" class="btn btn-primary">
            <i class="bi bi-plus-circle"></i> New Product
        </a>
    </div>
</div>

@* Search Form *@
<form asp-action="Index" method="get" class="mb-3">
    <div class="input-group">
        <input type="text" name="search" value="@Model.Search" 
               class="form-control" placeholder="Search products..." />
        <button type="submit" class="btn btn-outline-secondary">Search</button>
        @if (!string.IsNullOrEmpty(Model.Search))
        {
            <a asp-action="Index" class="btn btn-outline-danger">Clear</a>
        }
    </div>
</form>

@* Products Table *@
@if (Model.Products.Any())
{
    <table class="table table-striped table-hover">
        <thead class="table-dark">
            <tr>
                <th>Name</th>
                <th>Category</th>
                <th class="text-end">Price</th>
                <th class="text-center">Stock</th>
                <th class="text-center">Status</th>
                <th class="text-center">Actions</th>
            </tr>
        </thead>
        <tbody>
            @foreach (var product in Model.Products)
            {
                <tr>
                    <td>@product.Name</td>
                    <td>@product.CategoryName</td>
                    <td class="text-end">@product.Price.ToString("C")</td>
                    <td class="text-center">
                        @if (product.Stock > 10)
                        {
                            <span class="badge bg-success">@product.Stock</span>
                        }
                        else if (product.Stock > 0)
                        {
                            <span class="badge bg-warning">@product.Stock</span>
                        }
                        else
                        {
                            <span class="badge bg-danger">Out of Stock</span>
                        }
                    </td>
                    <td class="text-center">
                        @if (product.IsActive)
                        {
                            <span class="badge bg-success">Active</span>
                        }
                        else
                        {
                            <span class="badge bg-secondary">Inactive</span>
                        }
                    </td>
                    <td class="text-center">
                        <a asp-action="Details" asp-route-id="@product.Id" 
                           class="btn btn-sm btn-info">Details</a>
                        <a asp-action="Edit" asp-route-id="@product.Id" 
                           class="btn btn-sm btn-warning">Edit</a>
                        <a asp-action="Delete" asp-route-id="@product.Id" 
                           class="btn btn-sm btn-danger">Delete</a>
                    </td>
                </tr>
            }
        </tbody>
    </table>

    @* Pagination *@
    @if (Model.TotalPages > 1)
    {
        <nav>
            <ul class="pagination justify-content-center">
                <li class="page-item @(!Model.HasPreviousPage ? "disabled" : "")">
                    <a class="page-link" asp-action="Index" 
                       asp-route-page="@(Model.CurrentPage - 1)"
                       asp-route-search="@Model.Search">Previous</a>
                </li>
                @for (int i = 1; i <= Model.TotalPages; i++)
                {
                    <li class="page-item @(i == Model.CurrentPage ? "active" : "")">
                        <a class="page-link" asp-action="Index" 
                           asp-route-page="@i"
                           asp-route-search="@Model.Search">@i</a>
                    </li>
                }
                <li class="page-item @(!Model.HasNextPage ? "disabled" : "")">
                    <a class="page-link" asp-action="Index" 
                       asp-route-page="@(Model.CurrentPage + 1)"
                       asp-route-search="@Model.Search">Next</a>
                </li>
            </ul>
        </nav>
    }
}
else
{
    <div class="alert alert-info">
        No products found. <a asp-action="Create">Create the first product</a>.
    </div>
}
```

---

## Step 666: Razor Views - Create Product

```html
<!-- Views/Products/Create.cshtml -->
@model CreateProductViewModel

@{
    ViewData["Title"] = "Create Product";
}

<nav aria-label="breadcrumb">
    <ol class="breadcrumb">
        <li class="breadcrumb-item"><a asp-action="Index">Products</a></li>
        <li class="breadcrumb-item active">Create</li>
    </ol>
</nav>

<h1>Create Product</h1>

<div class="row">
    <div class="col-md-6">
        <form asp-action="Create" method="post" enctype="multipart/form-data">
            <div asp-validation-summary="ModelOnly" class="text-danger mb-3"></div>
            
            <div class="mb-3">
                <label asp-for="Name" class="form-label"></label>
                <input asp-for="Name" class="form-control" />
                <span asp-validation-for="Name" class="text-danger"></span>
            </div>
            
            <div class="mb-3">
                <label asp-for="Description" class="form-label"></label>
                <textarea asp-for="Description" class="form-control" rows="3"></textarea>
                <span asp-validation-for="Description" class="text-danger"></span>
            </div>
            
            <div class="row">
                <div class="col-md-6 mb-3">
                    <label asp-for="Price" class="form-label"></label>
                    <div class="input-group">
                        <span class="input-group-text">$</span>
                        <input asp-for="Price" class="form-control" />
                    </div>
                    <span asp-validation-for="Price" class="text-danger"></span>
                </div>
                
                <div class="col-md-6 mb-3">
                    <label asp-for="Stock" class="form-label"></label>
                    <input asp-for="Stock" class="form-control" />
                    <span asp-validation-for="Stock" class="text-danger"></span>
                </div>
            </div>
            
            <div class="mb-3">
                <label asp-for="CategoryId" class="form-label"></label>
                <select asp-for="CategoryId" class="form-select"
                    asp-items="@(new SelectList(Model.Categories, "Id", "Name"))">
                    <option value="">-- Select Category --</option>
                </select>
                <span asp-validation-for="CategoryId" class="text-danger"></span>
            </div>
            
            <div class="mb-3">
                <button type="submit" class="btn btn-primary">Create</button>
                <a asp-action="Index" class="btn btn-secondary">Cancel</a>
            </div>
        </form>
    </div>
</div>

@section Scripts {
    <partial name="_ValidationScriptsPartial" />
}
```

---

## Step 667: Tag Helpers

```csharp
// TagHelpers/ActiveRouteTagHelper.cs
using Microsoft.AspNetCore.Mvc.Rendering;
using Microsoft.AspNetCore.Mvc.ViewFeatures;
using Microsoft.AspNetCore.Razor.TagHelpers;

namespace MyMvcApp.TagHelpers;

[HtmlTargetElement("li", Attributes = "asp-active-route")]
public class ActiveRouteTagHelper : TagHelper
{
    private readonly IHttpContextAccessor _httpContextAccessor;

    [HtmlAttributeName("asp-active-route")]
    public string ActiveRoute { get; set; } = string.Empty;

    [ViewContext]
    [HtmlAttributeNotBound]
    public ViewContext ViewContext { get; set; } = default!;

    public ActiveRouteTagHelper(IHttpContextAccessor httpContextAccessor)
    {
        _httpContextAccessor = httpContextAccessor;
    }

    public override void Process(TagHelperContext context, TagHelperOutput output)
    {
        var currentPath = _httpContextAccessor.HttpContext?.Request.Path.Value ?? "";
        
        if (currentPath.StartsWith(ActiveRoute, StringComparison.OrdinalIgnoreCase))
        {
            var existingClass = output.Attributes["class"]?.Value?.ToString() ?? "";
            output.Attributes.SetAttribute("class", $"{existingClass} active".Trim());
        }
    }
}

// TagHelpers/GravatarTagHelper.cs
[HtmlTargetElement("gravatar")]
public class GravatarTagHelper : TagHelper
{
    [HtmlAttributeName("email")]
    public string Email { get; set; } = string.Empty;

    [HtmlAttributeName("size")]
    public int Size { get; set; } = 50;

    public override void Process(TagHelperContext context, TagHelperOutput output)
    {
        output.TagName = "img";
        output.TagMode = TagMode.SelfClosing;

        var hash = System.Security.Cryptography.MD5
            .HashData(System.Text.Encoding.UTF8.GetBytes(Email.ToLower().Trim()))
            .Select(b => b.ToString("x2"))
            .Aggregate(string.Concat);

        output.Attributes.SetAttribute("src", 
            $"https://www.gravatar.com/avatar/{hash}?s={Size}&d=mp");
        output.Attributes.SetAttribute("width", Size.ToString());
        output.Attributes.SetAttribute("height", Size.ToString());
        output.Attributes.SetAttribute("alt", Email);
    }
}

// TagHelpers/AlertTagHelper.cs
[HtmlTargetElement("alert")]
public class AlertTagHelper : TagHelper
{
    [HtmlAttributeName("type")]
    public string Type { get; set; } = "info";  // success, warning, danger, info

    [HtmlAttributeName("dismissible")]
    public bool Dismissible { get; set; } = true;

    public override async Task ProcessAsync(TagHelperContext context, TagHelperOutput output)
    {
        output.TagName = "div";
        output.TagMode = TagMode.StartTagAndEndTag;

        var classes = $"alert alert-{Type}";
        if (Dismissible)
            classes += " alert-dismissible fade show";

        output.Attributes.SetAttribute("class", classes);
        output.Attributes.SetAttribute("role", "alert");

        var content = await output.GetChildContentAsync();

        if (Dismissible)
        {
            output.PostContent.SetHtmlContent(
                @"<button type=""button"" class=""btn-close"" data-bs-dismiss=""alert""></button>");
        }
    }
}
```

```html
<!-- Usage in Views -->
@addTagHelper *, MyMvcApp

<!-- Active Route Tag Helper -->
<ul class="navbar-nav">
    <li class="nav-item" asp-active-route="/products">
        <a class="nav-link" asp-controller="Products" asp-action="Index">Products</a>
    </li>
</ul>

<!-- Gravatar Tag Helper -->
<gravatar email="user@example.com" size="80" />

<!-- Alert Tag Helper -->
<alert type="success">
    Product was created successfully!
</alert>
```

---

## Step 668: Partial Views

```html
<!-- Views/Shared/_ProductCard.cshtml -->
@model ProductSummary

<div class="card h-100">
    <div class="card-body">
        <h5 class="card-title">@Model.Name</h5>
        <p class="card-text text-muted small">@Model.CategoryName</p>
        <div class="d-flex justify-content-between align-items-center">
            <span class="h5 text-primary">@Model.Price.ToString("C")</span>
            @if (Model.Stock > 0)
            {
                <span class="badge bg-success">In Stock (@Model.Stock)</span>
            }
            else
            {
                <span class="badge bg-danger">Out of Stock</span>
            }
        </div>
    </div>
    <div class="card-footer">
        <a asp-controller="Products" asp-action="Details" asp-route-id="@Model.Id"
           class="btn btn-sm btn-outline-primary w-100">View Details</a>
    </div>
</div>

<!-- Views/Shared/_Pagination.cshtml -->
@model PaginationViewModel

@if (Model.TotalPages > 1)
{
    <nav aria-label="Page navigation">
        <ul class="pagination justify-content-center">
            <li class="page-item @(!Model.HasPreviousPage ? "disabled" : "")">
                <a class="page-link" href="@Model.GetPageUrl(Model.CurrentPage - 1)">
                    &laquo; Previous
                </a>
            </li>
            
            @for (int i = Math.Max(1, Model.CurrentPage - 2); 
                  i <= Math.Min(Model.TotalPages, Model.CurrentPage + 2); i++)
            {
                <li class="page-item @(i == Model.CurrentPage ? "active" : "")">
                    <a class="page-link" href="@Model.GetPageUrl(i)">@i</a>
                </li>
            }
            
            <li class="page-item @(!Model.HasNextPage ? "disabled" : "")">
                <a class="page-link" href="@Model.GetPageUrl(Model.CurrentPage + 1)">
                    Next &raquo;
                </a>
            </li>
        </ul>
    </nav>
}

<!-- Views/Shared/_LoginPartial.cshtml -->
@using Microsoft.AspNetCore.Identity

@inject SignInManager<ApplicationUser> SignInManager
@inject UserManager<ApplicationUser> UserManager

<ul class="navbar-nav">
    @if (SignInManager.IsSignedIn(User))
    {
        <li class="nav-item">
            <a class="nav-link text-dark" asp-area="" asp-controller="Manage" asp-action="Index">
                Hello, @UserManager.GetUserName(User)!
            </a>
        </li>
        <li class="nav-item">
            <form asp-area="" asp-controller="Account" asp-action="Logout" method="post">
                <button type="submit" class="nav-link btn btn-link text-dark">Logout</button>
            </form>
        </li>
    }
    else
    {
        <li class="nav-item">
            <a class="nav-link text-dark" asp-area="" asp-controller="Account" asp-action="Register">Register</a>
        </li>
        <li class="nav-item">
            <a class="nav-link text-dark" asp-area="" asp-controller="Account" asp-action="Login">Login</a>
        </li>
    }
</ul>
```

```csharp
// Calling Partial Views from Controllers
// Option 1: Return PartialViewResult
public IActionResult _ProductCard(int id)
{
    var product = _productService.GetSummaryById(id);
    return PartialView("_ProductCard", product);
}

// Option 2: Use in Razor View
// <partial name="_ProductCard" model="product" />
// <partial name="_Pagination" model="paginationModel" />

// Option 3: ViewComponent (more complex logic)
public class PaginationViewComponent : ViewComponent
{
    public IViewComponentResult Invoke(PaginationViewModel model)
    {
        return View(model);
    }
}
// Usage: @await Component.InvokeAsync("Pagination", new { model = paginationModel })
```

---

## Step 669: View Components

```csharp
// ViewComponents/CartSummaryViewComponent.cs
using Microsoft.AspNetCore.Mvc;

namespace MyMvcApp.ViewComponents;

public class CartSummaryViewComponent : ViewComponent
{
    private readonly ICartService _cartService;

    public CartSummaryViewComponent(ICartService cartService)
    {
        _cartService = cartService;
    }

    public async Task<IViewComponentResult> InvokeAsync()
    {
        if (!User.Identity?.IsAuthenticated ?? true)
            return View(new CartSummaryViewModel { ItemCount = 0, Total = 0 });

        var userId = HttpContext.User.FindFirst(System.Security.Claims.ClaimTypes.NameIdentifier)?.Value;
        var summary = await _cartService.GetSummaryAsync(userId!);
        
        return View(summary);
    }
}

// ViewComponents/RecentProductsViewComponent.cs
public class RecentProductsViewComponent : ViewComponent
{
    private readonly IProductService _productService;

    public RecentProductsViewComponent(IProductService productService)
    {
        _productService = productService;
    }

    public async Task<IViewComponentResult> InvokeAsync(int count = 5)
    {
        var products = await _productService.GetRecentAsync(count);
        return View(products);
    }
}

// Models/CartSummaryViewModel.cs
public class CartSummaryViewModel
{
    public int ItemCount { get; set; }
    public decimal Total { get; set; }
}
```

```html
<!-- Views/Shared/Components/CartSummary/Default.cshtml -->
@model CartSummaryViewModel

<div class="d-flex align-items-center">
    <a href="/Cart" class="btn btn-outline-primary btn-sm position-relative">
        <i class="bi bi-cart"></i> Cart
        @if (Model.ItemCount > 0)
        {
            <span class="position-absolute top-0 start-100 translate-middle badge rounded-pill bg-danger">
                @Model.ItemCount
            </span>
        }
    </a>
</div>

<!-- Views/Shared/Components/RecentProducts/Default.cshtml -->
@model IEnumerable<ProductSummary>

<div class="recent-products">
    <h5>Recent Products</h5>
    @foreach (var product in Model)
    {
        <div class="d-flex align-items-center mb-2">
            <div>
                <a asp-controller="Products" asp-action="Details" asp-route-id="@product.Id">
                    @product.Name
                </a>
                <small class="text-muted d-block">@product.Price.ToString("C")</small>
            </div>
        </div>
    }
</div>

<!-- Using View Components in Layout -->
<!-- In _Layout.cshtml: -->
@await Component.InvokeAsync("CartSummary")
@await Component.InvokeAsync("RecentProducts", new { count = 5 })
```

---

## Step 670: Razor Pages (ทางเลือกจาก MVC)

```csharp
// Program.cs สำหรับ Razor Pages
builder.Services.AddRazorPages(options =>
{
    options.Conventions.AuthorizeFolder("/Admin");
    options.Conventions.AllowAnonymousToPage("/Account/Login");
});

app.MapRazorPages();
```

```csharp
// Pages/Products/Index.cshtml.cs
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.RazorPages;

namespace MyMvcApp.Pages.Products;

public class IndexModel : PageModel
{
    private readonly IProductService _productService;

    public IndexModel(IProductService productService)
    {
        _productService = productService;
    }

    public IEnumerable<ProductSummary> Products { get; set; } = [];
    
    [BindProperty(SupportsGet = true)]
    public string? Search { get; set; }
    
    [BindProperty(SupportsGet = true)]
    public int PageNumber { get; set; } = 1;
    
    public int TotalPages { get; set; }

    public async Task OnGetAsync()
    {
        var (products, total) = await _productService.GetPagedAsync(Search, PageNumber, 10);
        Products = products;
        TotalPages = (int)Math.Ceiling(total / 10.0);
    }
}

// Pages/Products/Create.cshtml.cs
public class CreateModel : PageModel
{
    private readonly IProductService _productService;

    public CreateModel(IProductService productService)
    {
        _productService = productService;
    }

    [BindProperty]
    public CreateProductViewModel Product { get; set; } = new();

    public async Task<IActionResult> OnGetAsync()
    {
        await LoadCategoriesAsync();
        return Page();
    }

    public async Task<IActionResult> OnPostAsync()
    {
        if (!ModelState.IsValid)
        {
            await LoadCategoriesAsync();
            return Page();
        }

        await _productService.CreateAsync(Product);
        TempData["Success"] = "Product created!";
        return RedirectToPage("./Index");
    }

    private async Task LoadCategoriesAsync()
    {
        Product.Categories = await _productService.GetCategoriesAsync();
    }
}
```

```html
<!-- Pages/Products/Index.cshtml -->
@page
@model MyMvcApp.Pages.Products.IndexModel
@{
    ViewData["Title"] = "Products";
}

<h1>Products</h1>

<form method="get">
    <div class="input-group mb-3">
        <input type="text" asp-for="Search" class="form-control" placeholder="Search..." />
        <button type="submit" class="btn btn-primary">Search</button>
    </div>
</form>

@foreach (var product in Model.Products)
{
    <partial name="_ProductCard" model="product" />
}
```

---

## Step 671: Model Binding Advanced

```csharp
// Complex Model Binding
public class OrderFilterModel
{
    public string? CustomerName { get; set; }
    public decimal? MinAmount { get; set; }
    public decimal? MaxAmount { get; set; }
    public DateTime? FromDate { get; set; }
    public DateTime? ToDate { get; set; }
    public List<int> CategoryIds { get; set; } = [];
    public OrderStatus? Status { get; set; }
    public string SortBy { get; set; } = "date";
    public bool SortDescending { get; set; } = true;
}

// Controller Action with Complex Binding
public async Task<IActionResult> Orders([FromQuery] OrderFilterModel filter)
{
    var orders = await _orderService.GetFilteredAsync(filter);
    return View(orders);
}

// Custom Model Binder
public class DateRangeModelBinder : IModelBinder
{
    public Task BindModelAsync(ModelBindingContext bindingContext)
    {
        var fromValue = bindingContext.ValueProvider.GetValue("from");
        var toValue = bindingContext.ValueProvider.GetValue("to");

        if (fromValue == ValueProviderResult.None || toValue == ValueProviderResult.None)
        {
            bindingContext.Result = ModelBindingResult.Success(DateRange.Empty);
            return Task.CompletedTask;
        }

        if (DateTime.TryParse(fromValue.FirstValue, out var from) &&
            DateTime.TryParse(toValue.FirstValue, out var to))
        {
            bindingContext.Result = ModelBindingResult.Success(new DateRange(from, to));
        }
        else
        {
            bindingContext.ModelState.AddModelError(bindingContext.ModelName, "Invalid date range");
        }

        return Task.CompletedTask;
    }
}

public record DateRange(DateTime From, DateTime To)
{
    public static DateRange Empty => new(DateTime.MinValue, DateTime.MaxValue);
}

// Register Custom Binder
[ModelBinder(typeof(DateRangeModelBinder))]
public DateRange Period { get; set; } = DateRange.Empty;
```

---

## Step 672: Filters และ Action Filters

```csharp
// Filters/LogActionFilter.cs
using Microsoft.AspNetCore.Mvc.Filters;

namespace MyMvcApp.Filters;

public class LogActionFilter : IActionFilter
{
    private readonly ILogger<LogActionFilter> _logger;

    public LogActionFilter(ILogger<LogActionFilter> logger)
    {
        _logger = logger;
    }

    public void OnActionExecuting(ActionExecutingContext context)
    {
        var controllerName = context.RouteData.Values["controller"];
        var actionName = context.RouteData.Values["action"];
        _logger.LogInformation("Executing {Controller}.{Action}", controllerName, actionName);
    }

    public void OnActionExecuted(ActionExecutedContext context)
    {
        var controllerName = context.RouteData.Values["controller"];
        var actionName = context.RouteData.Values["action"];
        _logger.LogInformation("Executed {Controller}.{Action}", controllerName, actionName);
    }
}

// Filters/ValidateModelFilter.cs
public class ValidateModelFilter : IActionFilter
{
    public void OnActionExecuting(ActionExecutingContext context)
    {
        if (!context.ModelState.IsValid)
        {
            // For API
            context.Result = new BadRequestObjectResult(context.ModelState);
            
            // For MVC Views - need different approach
            // context.Controller is not easily accessible as Controller type
        }
    }

    public void OnActionExecuted(ActionExecutedContext context) { }
}

// Filters/AuditAttribute.cs
[AttributeUsage(AttributeTargets.Class | AttributeTargets.Method)]
public class AuditAttribute : ActionFilterAttribute
{
    public string EventName { get; set; } = string.Empty;

    public override async Task OnActionExecutionAsync(
        ActionExecutingContext context, ActionExecutionDelegate next)
    {
        var userId = context.HttpContext.User.Identity?.Name ?? "anonymous";
        var ip = context.HttpContext.Connection.RemoteIpAddress?.ToString();
        
        var result = await next();
        
        if (result.Exception is null)
        {
            var auditService = context.HttpContext.RequestServices
                .GetRequiredService<IAuditService>();
            
            await auditService.LogAsync(new AuditEntry
            {
                Event = EventName,
                UserId = userId,
                IpAddress = ip ?? "unknown",
                Timestamp = DateTime.UtcNow
            });
        }
    }
}

// Filters/RequirePermissionAttribute.cs
public class RequirePermissionAttribute : TypeFilterAttribute
{
    public RequirePermissionAttribute(string permission) 
        : base(typeof(PermissionAuthorizationFilter))
    {
        Arguments = [permission];
    }
}

public class PermissionAuthorizationFilter : IAuthorizationFilter
{
    private readonly string _permission;
    private readonly IPermissionService _permissionService;

    public PermissionAuthorizationFilter(string permission, IPermissionService permissionService)
    {
        _permission = permission;
        _permissionService = permissionService;
    }

    public void OnAuthorization(AuthorizationFilterContext context)
    {
        var userId = context.HttpContext.User.Identity?.Name;
        if (userId is null || !_permissionService.HasPermission(userId, _permission))
        {
            context.Result = new ForbidResult();
        }
    }
}

// Register filters globally
builder.Services.AddControllersWithViews(options =>
{
    options.Filters.Add<LogActionFilter>();
    options.Filters.Add<ValidateModelFilter>();
});

// Use filter attributes on controllers/actions
[Audit(EventName = "CreateProduct")]
[RequirePermission("product.create")]
public async Task<IActionResult> Create(CreateProductViewModel model) { ... }
```

---

## Step 673: Areas

```csharp
// Areas/Admin/Controllers/DashboardController.cs
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;

namespace MyMvcApp.Areas.Admin.Controllers;

[Area("Admin")]
[Authorize(Roles = "Admin")]
public class DashboardController : Controller
{
    private readonly IDashboardService _dashboardService;

    public DashboardController(IDashboardService dashboardService)
    {
        _dashboardService = dashboardService;
    }

    public async Task<IActionResult> Index()
    {
        var stats = await _dashboardService.GetStatsAsync();
        return View(stats);
    }

    public async Task<IActionResult> Users()
    {
        var users = await _dashboardService.GetUsersAsync();
        return View(users);
    }
}

// Areas/Admin/Controllers/ProductsController.cs
[Area("Admin")]
[Authorize(Roles = "Admin,ProductManager")]
public class ProductsController : Controller
{
    // Full CRUD with additional admin features
}
```

```html
<!-- Areas/Admin/Views/Shared/_AdminLayout.cshtml -->
<!DOCTYPE html>
<html>
<head>
    <title>Admin - @ViewData["Title"]</title>
    <link rel="stylesheet" href="~/lib/bootstrap/dist/css/bootstrap.min.css" />
    <link rel="stylesheet" href="~/css/admin.css" />
</head>
<body class="admin-panel">
    <div class="wrapper">
        <!-- Sidebar -->
        <nav class="sidebar">
            <div class="sidebar-header">
                <h3>Admin Panel</h3>
            </div>
            <ul class="sidebar-nav">
                <li>
                    <a asp-area="Admin" asp-controller="Dashboard" asp-action="Index">
                        Dashboard
                    </a>
                </li>
                <li>
                    <a asp-area="Admin" asp-controller="Products" asp-action="Index">
                        Products
                    </a>
                </li>
                <li>
                    <a asp-area="Admin" asp-controller="Dashboard" asp-action="Users">
                        Users
                    </a>
                </li>
            </ul>
        </nav>
        
        <!-- Main Content -->
        <div class="content-wrapper">
            @RenderBody()
        </div>
    </div>
</body>
</html>

<!-- Areas/Admin/Views/_ViewStart.cshtml -->
@{
    Layout = "_AdminLayout";
}
```

```csharp
// Routing with Areas
app.MapControllerRoute(
    name: "admin",
    pattern: "Admin/{controller=Dashboard}/{action=Index}/{id?}",
    defaults: new { area = "Admin" });

// Or use attribute routing in Program.cs
app.MapControllerRoute(
    name: "areas",
    pattern: "{area:exists}/{controller=Home}/{action=Index}/{id?}");
```

---

## Step 674: Authentication และ Authorization ใน MVC

```csharp
// Program.cs - ASP.NET Core Identity Setup
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString));

builder.Services.AddIdentity<ApplicationUser, ApplicationRole>(options =>
{
    options.Password.RequiredLength = 8;
    options.Password.RequireDigit = true;
    options.Password.RequireUppercase = true;
    options.Password.RequireNonAlphanumeric = false;
    options.User.RequireUniqueEmail = true;
    options.SignIn.RequireConfirmedEmail = true;
    options.Lockout.MaxFailedAccessAttempts = 5;
    options.Lockout.DefaultLockoutTimeSpan = TimeSpan.FromMinutes(15);
})
.AddEntityFrameworkStores<AppDbContext>()
.AddDefaultTokenProviders();

builder.Services.ConfigureApplicationCookie(options =>
{
    options.LoginPath = "/Account/Login";
    options.LogoutPath = "/Account/Logout";
    options.AccessDeniedPath = "/Account/AccessDenied";
    options.Cookie.HttpOnly = true;
    options.Cookie.SameSite = SameSiteMode.Lax;
    options.ExpireTimeSpan = TimeSpan.FromDays(7);
    options.SlidingExpiration = true;
});
```

```csharp
// Models/ApplicationUser.cs
using Microsoft.AspNetCore.Identity;

public class ApplicationUser : IdentityUser
{
    public string FirstName { get; set; } = string.Empty;
    public string LastName { get; set; } = string.Empty;
    public DateTime DateOfBirth { get; set; }
    public string? ProfilePictureUrl { get; set; }
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    
    public string FullName => $"{FirstName} {LastName}";
}

public class ApplicationRole : IdentityRole
{
    public string? Description { get; set; }
}
```

```csharp
// Controllers/AccountController.cs
using Microsoft.AspNetCore.Identity;
using Microsoft.AspNetCore.Mvc;

namespace MyMvcApp.Controllers;

public class AccountController : Controller
{
    private readonly UserManager<ApplicationUser> _userManager;
    private readonly SignInManager<ApplicationUser> _signInManager;
    private readonly IEmailSender _emailSender;
    private readonly ILogger<AccountController> _logger;

    public AccountController(
        UserManager<ApplicationUser> userManager,
        SignInManager<ApplicationUser> signInManager,
        IEmailSender emailSender,
        ILogger<AccountController> logger)
    {
        _userManager = userManager;
        _signInManager = signInManager;
        _emailSender = emailSender;
        _logger = logger;
    }

    [HttpGet]
    public IActionResult Register() => View();

    [HttpPost]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> Register(RegisterViewModel model)
    {
        if (!ModelState.IsValid)
            return View(model);

        var user = new ApplicationUser
        {
            UserName = model.Email,
            Email = model.Email,
            FirstName = model.FirstName,
            LastName = model.LastName
        };

        var result = await _userManager.CreateAsync(user, model.Password);

        if (result.Succeeded)
        {
            await _userManager.AddToRoleAsync(user, "User");
            
            var token = await _userManager.GenerateEmailConfirmationTokenAsync(user);
            var confirmUrl = Url.Action("ConfirmEmail", "Account", 
                new { userId = user.Id, token }, Request.Scheme)!;
            
            await _emailSender.SendEmailAsync(model.Email, "Confirm Email",
                $"Please confirm your email by clicking <a href='{confirmUrl}'>here</a>.");

            TempData["Info"] = "Registration successful! Please check your email to confirm.";
            return RedirectToAction("Login");
        }

        foreach (var error in result.Errors)
            ModelState.AddModelError(string.Empty, error.Description);

        return View(model);
    }

    [HttpGet]
    public IActionResult Login(string? returnUrl = null)
    {
        ViewData["ReturnUrl"] = returnUrl;
        return View();
    }

    [HttpPost]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> Login(LoginViewModel model, string? returnUrl = null)
    {
        if (!ModelState.IsValid)
            return View(model);

        var result = await _signInManager.PasswordSignInAsync(
            model.Email, model.Password, model.RememberMe, lockoutOnFailure: true);

        if (result.Succeeded)
        {
            _logger.LogInformation("User {Email} logged in", model.Email);
            return LocalRedirect(returnUrl ?? "/");
        }

        if (result.RequiresTwoFactor)
        {
            return RedirectToAction("LoginWith2fa", new { returnUrl, model.RememberMe });
        }

        if (result.IsLockedOut)
        {
            return RedirectToAction("Lockout");
        }

        ModelState.AddModelError(string.Empty, "Invalid login attempt.");
        return View(model);
    }

    [HttpPost]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> Logout()
    {
        await _signInManager.SignOutAsync();
        _logger.LogInformation("User logged out");
        return RedirectToAction("Index", "Home");
    }

    public async Task<IActionResult> ConfirmEmail(string userId, string token)
    {
        var user = await _userManager.FindByIdAsync(userId);
        if (user is null)
            return NotFound();

        var result = await _userManager.ConfirmEmailAsync(user, token);
        return View(result.Succeeded ? "ConfirmEmailSuccess" : "ConfirmEmailFail");
    }
}
```

---

## Step 675: Authorization Policies

```csharp
// Program.cs - Custom Authorization Policies
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("AdminOnly", policy => 
        policy.RequireRole("Admin"));
    
    options.AddPolicy("SeniorEmployee", policy =>
        policy.RequireRole("Admin", "Manager")
              .RequireClaim("Department", "IT", "Finance"));
    
    options.AddPolicy("MinimumAge", policy =>
        policy.AddRequirements(new MinimumAgeRequirement(18)));
    
    options.AddPolicy("CanEditProduct", policy =>
        policy.AddRequirements(new ResourceOwnerRequirement()));
    
    options.FallbackPolicy = new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .Build();
});

builder.Services.AddSingleton<IAuthorizationHandler, MinimumAgeHandler>();
builder.Services.AddScoped<IAuthorizationHandler, ResourceOwnerHandler>();

// Requirements and Handlers
public class MinimumAgeRequirement : IAuthorizationRequirement
{
    public int MinimumAge { get; }
    public MinimumAgeRequirement(int minimumAge) => MinimumAge = minimumAge;
}

public class MinimumAgeHandler : AuthorizationHandler<MinimumAgeRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        MinimumAgeRequirement requirement)
    {
        var dateOfBirthClaim = context.User.FindFirst("DateOfBirth");
        if (dateOfBirthClaim is null)
            return Task.CompletedTask;

        if (DateTime.TryParse(dateOfBirthClaim.Value, out var dateOfBirth))
        {
            var age = DateTime.Today.Year - dateOfBirth.Year;
            if (age >= requirement.MinimumAge)
                context.Succeed(requirement);
        }

        return Task.CompletedTask;
    }
}

// Resource-based authorization
public class ResourceOwnerRequirement : IAuthorizationRequirement { }

public class ResourceOwnerHandler : AuthorizationHandler<ResourceOwnerRequirement, Product>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        ResourceOwnerRequirement requirement,
        Product resource)
    {
        var userId = context.User.FindFirst(System.Security.Claims.ClaimTypes.NameIdentifier)?.Value;
        
        if (userId == resource.OwnerId || context.User.IsInRole("Admin"))
            context.Succeed(requirement);

        return Task.CompletedTask;
    }
}

// Using resource-based auth in controller
[HttpPost]
public async Task<IActionResult> Edit(int id, EditProductViewModel model)
{
    var product = await _productService.GetByIdAsync(id);
    if (product is null)
        return NotFound();

    var authResult = await _authorizationService.AuthorizeAsync(User, product, "CanEditProduct");
    if (!authResult.Succeeded)
        return Forbid();

    // proceed with edit
}
```

---

## Step 676: Form Handling และ Validation

```csharp
// Models/ViewModels/RegistrationViewModel.cs
using System.ComponentModel.DataAnnotations;

public class RegisterViewModel
{
    [Required]
    [Display(Name = "First Name")]
    [StringLength(50, MinimumLength = 2)]
    public string FirstName { get; set; } = string.Empty;

    [Required]
    [Display(Name = "Last Name")]
    [StringLength(50, MinimumLength = 2)]
    public string LastName { get; set; } = string.Empty;

    [Required]
    [EmailAddress]
    [Display(Name = "Email")]
    public string Email { get; set; } = string.Empty;

    [Required]
    [StringLength(100, MinimumLength = 8)]
    [DataType(DataType.Password)]
    [Display(Name = "Password")]
    public string Password { get; set; } = string.Empty;

    [DataType(DataType.Password)]
    [Display(Name = "Confirm Password")]
    [Compare("Password", ErrorMessage = "The password and confirmation password do not match.")]
    public string ConfirmPassword { get; set; } = string.Empty;

    [Required]
    [Display(Name = "Date of Birth")]
    [DataType(DataType.Date)]
    [MinimumAge(18, ErrorMessage = "You must be at least 18 years old")]
    public DateTime DateOfBirth { get; set; }

    [Required]
    [Display(Name = "I agree to the Terms of Service")]
    [MustBeTrue(ErrorMessage = "You must accept the terms of service")]
    public bool AcceptTerms { get; set; }
}

// Custom Validation Attributes
public class MinimumAgeAttribute : ValidationAttribute
{
    private readonly int _minimumAge;

    public MinimumAgeAttribute(int minimumAge)
    {
        _minimumAge = minimumAge;
    }

    public override bool IsValid(object? value)
    {
        if (value is DateTime dateOfBirth)
        {
            var age = DateTime.Today.Year - dateOfBirth.Year;
            if (dateOfBirth.Date > DateTime.Today.AddYears(-age)) age--;
            return age >= _minimumAge;
        }
        return false;
    }
}

public class MustBeTrueAttribute : ValidationAttribute
{
    public override bool IsValid(object? value)
        => value is bool boolValue && boolValue;
}
```

```html
<!-- Views/Account/Register.cshtml -->
@model RegisterViewModel

@{
    ViewData["Title"] = "Register";
}

<div class="row justify-content-center">
    <div class="col-md-6">
        <div class="card">
            <div class="card-header"><h4>Create Account</h4></div>
            <div class="card-body">
                <form asp-action="Register" method="post">
                    <div asp-validation-summary="All" class="alert alert-danger" role="alert"></div>
                    
                    <div class="row">
                        <div class="col-md-6 mb-3">
                            <label asp-for="FirstName" class="form-label"></label>
                            <input asp-for="FirstName" class="form-control" />
                            <span asp-validation-for="FirstName" class="text-danger small"></span>
                        </div>
                        <div class="col-md-6 mb-3">
                            <label asp-for="LastName" class="form-label"></label>
                            <input asp-for="LastName" class="form-control" />
                            <span asp-validation-for="LastName" class="text-danger small"></span>
                        </div>
                    </div>
                    
                    <div class="mb-3">
                        <label asp-for="Email" class="form-label"></label>
                        <input asp-for="Email" class="form-control" />
                        <span asp-validation-for="Email" class="text-danger small"></span>
                    </div>
                    
                    <div class="mb-3">
                        <label asp-for="Password" class="form-label"></label>
                        <input asp-for="Password" class="form-control" type="password" />
                        <span asp-validation-for="Password" class="text-danger small"></span>
                    </div>
                    
                    <div class="mb-3">
                        <label asp-for="ConfirmPassword" class="form-label"></label>
                        <input asp-for="ConfirmPassword" class="form-control" type="password" />
                        <span asp-validation-for="ConfirmPassword" class="text-danger small"></span>
                    </div>
                    
                    <div class="mb-3">
                        <label asp-for="DateOfBirth" class="form-label"></label>
                        <input asp-for="DateOfBirth" class="form-control" type="date" />
                        <span asp-validation-for="DateOfBirth" class="text-danger small"></span>
                    </div>
                    
                    <div class="mb-3 form-check">
                        <input asp-for="AcceptTerms" class="form-check-input" type="checkbox" />
                        <label asp-for="AcceptTerms" class="form-check-label">
                            I accept the <a href="/Terms">Terms of Service</a>
                        </label>
                        <span asp-validation-for="AcceptTerms" class="text-danger d-block small"></span>
                    </div>
                    
                    <button type="submit" class="btn btn-primary w-100">Register</button>
                </form>
                
                <div class="mt-3 text-center">
                    Already have an account? <a asp-action="Login">Login here</a>
                </div>
            </div>
        </div>
    </div>
</div>

@section Scripts {
    <partial name="_ValidationScriptsPartial" />
}
```

---

## Step 677: File Upload

```csharp
// Controllers/FileUploadController.cs
[HttpPost]
[ValidateAntiForgeryToken]
public async Task<IActionResult> UploadProductImage(int productId, IFormFile image)
{
    if (image is null || image.Length == 0)
    {
        ModelState.AddModelError("image", "Please select an image.");
        return RedirectToAction("Edit", new { id = productId });
    }

    var allowedExtensions = new[] { ".jpg", ".jpeg", ".png", ".webp" };
    var extension = Path.GetExtension(image.FileName).ToLowerInvariant();
    
    if (!allowedExtensions.Contains(extension))
    {
        ModelState.AddModelError("image", "Only JPG, PNG, and WebP images are allowed.");
        return RedirectToAction("Edit", new { id = productId });
    }

    const long maxSize = 5 * 1024 * 1024; // 5MB
    if (image.Length > maxSize)
    {
        ModelState.AddModelError("image", "Image must be less than 5MB.");
        return RedirectToAction("Edit", new { id = productId });
    }

    var fileName = $"{Guid.NewGuid()}{extension}";
    var uploadPath = Path.Combine(_environment.WebRootPath, "uploads", "products");
    
    Directory.CreateDirectory(uploadPath);
    
    var filePath = Path.Combine(uploadPath, fileName);
    using var stream = new FileStream(filePath, FileMode.Create);
    await image.CopyToAsync(stream);

    await _productService.UpdateImageAsync(productId, $"/uploads/products/{fileName}");
    
    TempData["Success"] = "Image uploaded successfully!";
    return RedirectToAction("Edit", new { id = productId });
}

// Multiple file upload
[HttpPost]
public async Task<IActionResult> UploadMultiple(List<IFormFile> files)
{
    var results = new List<UploadResult>();
    
    foreach (var file in files)
    {
        if (file.Length > 0)
        {
            var fileName = Path.GetRandomFileName();
            var filePath = Path.Combine(_environment.WebRootPath, "uploads", fileName);
            
            using var stream = File.Create(filePath);
            await file.CopyToAsync(stream);
            
            results.Add(new UploadResult
            {
                FileName = file.FileName,
                StoredName = fileName,
                Size = file.Length
            });
        }
    }
    
    return Json(results);
}
```

---

## Step 678: Caching ใน MVC

```csharp
// Program.cs
builder.Services.AddMemoryCache();
builder.Services.AddResponseCaching();
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = builder.Configuration.GetConnectionString("Redis");
    options.InstanceName = "MyMvcApp_";
});

// In middleware pipeline
app.UseResponseCaching();

// Controller Level Caching
[ResponseCache(Duration = 300, Location = ResponseCacheLocation.Any)]
public async Task<IActionResult> Index()
{
    // Response cached for 5 minutes
}

// No Cache
[ResponseCache(Duration = 0, Location = ResponseCacheLocation.None, NoStore = true)]
public IActionResult Error() { ... }

// Using IMemoryCache
public class CachedProductService
{
    private readonly IMemoryCache _cache;
    private readonly IProductRepository _repository;
    private static readonly TimeSpan CacheDuration = TimeSpan.FromMinutes(10);

    public CachedProductService(IMemoryCache cache, IProductRepository repository)
    {
        _cache = cache;
        _repository = repository;
    }

    public async Task<IEnumerable<ProductSummary>> GetAllAsync()
    {
        return await _cache.GetOrCreateAsync("all_products", async entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = CacheDuration;
            entry.SlidingExpiration = TimeSpan.FromMinutes(2);
            entry.Priority = CacheItemPriority.Normal;
            
            return await _repository.GetAllAsync();
        }) ?? [];
    }

    public void InvalidateProductCache()
    {
        _cache.Remove("all_products");
    }
}

// Using IDistributedCache
public class DistributedCachedService
{
    private readonly IDistributedCache _cache;
    private readonly JsonSerializerOptions _jsonOptions = new() 
    { 
        PropertyNamingPolicy = JsonNamingPolicy.CamelCase 
    };

    public async Task<T?> GetAsync<T>(string key)
    {
        var cached = await _cache.GetStringAsync(key);
        return cached is null ? default : JsonSerializer.Deserialize<T>(cached, _jsonOptions);
    }

    public async Task SetAsync<T>(string key, T value, TimeSpan expiry)
    {
        var json = JsonSerializer.Serialize(value, _jsonOptions);
        var options = new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = expiry
        };
        await _cache.SetStringAsync(key, json, options);
    }
}
```

---

## Step 679: Localization

```csharp
// Program.cs - Localization Setup
builder.Services.AddLocalization(options => options.ResourcesPath = "Resources");

builder.Services.AddControllersWithViews()
    .AddViewLocalization()
    .AddDataAnnotationsLocalization();

builder.Services.Configure<RequestLocalizationOptions>(options =>
{
    var supportedCultures = new[] { "en-US", "th-TH", "ja-JP" };
    
    options.SetDefaultCulture(supportedCultures[0])
        .AddSupportedCultures(supportedCultures)
        .AddSupportedUICultures(supportedCultures);
    
    options.RequestCultureProviders.Insert(0, 
        new QueryStringRequestCultureProvider { QueryStringKey = "lang" });
});

app.UseRequestLocalization();

// Resources/Controllers/HomeController.th-TH.resx (XML resource file)
// Key: Welcome, Value: ยินดีต้อนรับ

// Resources/Views/Home/Index.th-TH.resx
// Key: Title, Value: หน้าแรก

// Using IStringLocalizer in Controller
public class HomeController : Controller
{
    private readonly IStringLocalizer<HomeController> _localizer;
    private readonly IStringLocalizer<SharedResource> _sharedLocalizer;

    public HomeController(
        IStringLocalizer<HomeController> localizer,
        IStringLocalizer<SharedResource> sharedLocalizer)
    {
        _localizer = localizer;
        _sharedLocalizer = sharedLocalizer;
    }

    public IActionResult Index()
    {
        ViewData["Title"] = _localizer["Title"];
        ViewData["Welcome"] = _localizer["Welcome", "John"];  // With parameter
        return View();
    }
}
```

```html
<!-- Using IHtmlLocalizer in View -->
@inject IHtmlLocalizer<HomeController> HtmlLocalizer
@inject IStringLocalizer<SharedResource> SharedLocalizer

<h1>@SharedLocalizer["AppName"]</h1>
<p>@HtmlLocalizer["WelcomeMessage", User.Identity!.Name]</p>

<!-- Language Switcher Partial -->
@using Microsoft.AspNetCore.Localization
@using Microsoft.AspNetCore.Mvc.Localization

@{
    var requestCulture = Context.Features.Get<IRequestCultureFeature>();
    var cultures = new[] 
    { 
        ("en-US", "English"), 
        ("th-TH", "ภาษาไทย"),
        ("ja-JP", "日本語")
    };
}

<div class="dropdown">
    <button class="btn btn-sm btn-outline-secondary dropdown-toggle">
        @requestCulture?.RequestCulture.Culture.DisplayName
    </button>
    <ul class="dropdown-menu">
        @foreach (var (code, name) in cultures)
        {
            <li>
                <a class="dropdown-item" 
                   href="?lang=@code&returnUrl=@Context.Request.Path">@name</a>
            </li>
        }
    </ul>
</div>
```

---

## Step 680: Scaffolding

```bash
# Install scaffolding tools
dotnet tool install --global dotnet-aspnet-codegenerator
dotnet add package Microsoft.VisualStudio.Web.CodeGeneration.Design

# Scaffold Controller with Views (MVC)
dotnet aspnet-codegenerator controller \
    -name ProductsController \
    -m Product \
    -dc AppDbContext \
    --relativeFolderPath Controllers \
    --useDefaultLayout \
    --referenceScriptLibraries

# Scaffold Razor Pages
dotnet aspnet-codegenerator razorpage \
    -m Product \
    -dc AppDbContext \
    --relativeFolderPath Pages/Products \
    --useDefaultLayout

# Scaffold Identity
dotnet aspnet-codegenerator identity \
    -dc AppDbContext \
    --files "Account.Register;Account.Login;Account.Logout"
```

---

## Step 681: Middleware ใน MVC

```csharp
// Middleware/TenantMiddleware.cs
namespace MyMvcApp.Middleware;

public class TenantMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<TenantMiddleware> _logger;

    public TenantMiddleware(RequestDelegate next, ILogger<TenantMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var tenantId = context.Request.Headers["X-Tenant-Id"].FirstOrDefault()
            ?? context.Request.Host.Host.Split('.').FirstOrDefault()
            ?? "default";

        context.Items["TenantId"] = tenantId;
        _logger.LogInformation("Request from tenant: {TenantId}", tenantId);

        await _next(context);
    }
}

// Middleware/MaintenanceModeMiddleware.cs
public class MaintenanceModeMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IConfiguration _config;

    public MaintenanceModeMiddleware(RequestDelegate next, IConfiguration config)
    {
        _next = next;
        _config = config;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        if (_config.GetValue<bool>("MaintenanceMode"))
        {
            if (!context.Request.Path.StartsWithSegments("/admin") &&
                !context.User.IsInRole("Admin"))
            {
                context.Response.StatusCode = StatusCodes.Status503ServiceUnavailable;
                await context.Response.WriteAsync("Site is under maintenance. Please try again later.");
                return;
            }
        }

        await _next(context);
    }
}

// Extension methods for clean registration
public static class MiddlewareExtensions
{
    public static IApplicationBuilder UseTenant(this IApplicationBuilder app)
        => app.UseMiddleware<TenantMiddleware>();
    
    public static IApplicationBuilder UseMaintenanceMode(this IApplicationBuilder app)
        => app.UseMiddleware<MaintenanceModeMiddleware>();
}

// Program.cs
app.UseTenant();
app.UseMaintenanceMode();
```

---

## Step 682: Error Handling ใน MVC

```csharp
// Controllers/ErrorController.cs
[Route("[controller]")]
public class ErrorController : Controller
{
    private readonly ILogger<ErrorController> _logger;

    public ErrorController(ILogger<ErrorController> logger)
    {
        _logger = logger;
    }

    [Route("{statusCode}")]
    public IActionResult StatusCode(int statusCode)
    {
        var errorViewModel = new ErrorStatusCodeViewModel
        {
            StatusCode = statusCode,
            Message = statusCode switch
            {
                404 => "The page you requested could not be found.",
                403 => "You don't have permission to access this page.",
                500 => "An internal server error occurred.",
                _ => "An error occurred."
            }
        };

        return statusCode == 404 ? View("NotFound", errorViewModel) 
                                 : View("Error", errorViewModel);
    }
}

// Program.cs
app.UseStatusCodePagesWithReExecute("/Error/{0}");

if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage();
}
else
{
    app.UseExceptionHandler("/Home/Error");
}

// Global exception handler with custom error page
app.UseExceptionHandler(appError =>
{
    appError.Run(async context =>
    {
        context.Response.StatusCode = StatusCodes.Status500InternalServerError;
        context.Response.ContentType = "text/html";

        var contextFeature = context.Features.Get<IExceptionHandlerFeature>();
        if (contextFeature != null)
        {
            var logger = context.RequestServices.GetRequiredService<ILogger<Program>>();
            logger.LogError(contextFeature.Error, "Unhandled exception");

            // Return error page or JSON based on request
            if (context.Request.Headers.Accept.ToString().Contains("application/json"))
            {
                await context.Response.WriteAsJsonAsync(new 
                { 
                    error = "An internal error occurred.",
                    requestId = context.TraceIdentifier 
                });
            }
            else
            {
                context.Response.Redirect("/Home/Error");
            }
        }
    });
});
```

---

## Step 683: Real-time Features ด้วย HTMX

```html
<!-- wwwroot/js/htmx.min.js (download from htmx.org) -->

<!-- Views/Products/Index.cshtml - HTMX Integration -->
@model ProductListViewModel

<div id="product-list">
    <!-- Search with HTMX - no JavaScript needed! -->
    <input type="text" 
           name="search" 
           placeholder="Search products..." 
           class="form-control"
           hx-get="/Products/Search"
           hx-trigger="keyup changed delay:300ms"
           hx-target="#product-results"
           hx-indicator="#search-spinner" />
    <span id="search-spinner" class="htmx-indicator spinner-border spinner-border-sm"></span>
    
    <div id="product-results">
        <partial name="_ProductList" model="Model.Products" />
    </div>
</div>

<!-- Delete with confirmation using HTMX -->
<button hx-delete="/Products/@product.Id"
        hx-confirm="Are you sure you want to delete this product?"
        hx-target="closest tr"
        hx-swap="outerHTML"
        class="btn btn-sm btn-danger">Delete</button>

<!-- Infinite scroll -->
<div hx-get="/Products?page=@(Model.CurrentPage + 1)"
     hx-trigger="intersect once"
     hx-target="#product-results"
     hx-swap="beforeend">
</div>
```

```csharp
// Controllers/ProductsController.cs - HTMX endpoints
public async Task<IActionResult> Search(string? search)
{
    var products = await _productService.SearchAsync(search);
    return PartialView("_ProductList", products);
}

[HttpDelete("{id}")]
public async Task<IActionResult> Delete(int id)
{
    await _productService.DeleteAsync(id);
    // Return empty 200 to remove the element
    return Ok();
}
```

---

## Step 684: SignalR ใน MVC

```csharp
// Hubs/NotificationHub.cs
using Microsoft.AspNetCore.SignalR;

namespace MyMvcApp.Hubs;

public class NotificationHub : Hub
{
    public async Task SendNotification(string message)
    {
        await Clients.All.SendAsync("ReceiveNotification", message);
    }

    public async Task JoinGroup(string groupName)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, groupName);
    }

    public async Task SendToGroup(string groupName, string message)
    {
        await Clients.Group(groupName).SendAsync("ReceiveNotification", message);
    }

    public override async Task OnConnectedAsync()
    {
        var userId = Context.User?.FindFirst("sub")?.Value;
        if (userId != null)
            await Groups.AddToGroupAsync(Context.ConnectionId, $"user_{userId}");
        
        await base.OnConnectedAsync();
    }
}

// Program.cs
builder.Services.AddSignalR();
app.MapHub<NotificationHub>("/hubs/notifications");
```

```html
<!-- Views/Shared/_Layout.cshtml - SignalR Client -->
<script src="~/lib/microsoft/signalr/dist/browser/signalr.min.js"></script>
<script>
    const connection = new signalR.HubConnectionBuilder()
        .withUrl("/hubs/notifications")
        .withAutomaticReconnect()
        .build();

    connection.on("ReceiveNotification", (message) => {
        const toast = document.createElement("div");
        toast.className = "toast show";
        toast.innerHTML = `<div class="toast-body">${message}</div>`;
        document.getElementById("toast-container").appendChild(toast);
        setTimeout(() => toast.remove(), 5000);
    });

    connection.start().catch(err => console.error(err));
</script>

<div id="toast-container" class="position-fixed bottom-0 end-0 p-3" style="z-index: 11"></div>
```

---

## Step 685: SPA Integration

```csharp
// Program.cs - Serving a React/Vue/Angular SPA
builder.Services.AddSpaStaticFiles(configuration =>
{
    configuration.RootPath = "ClientApp/dist";
});

// After UseRouting:
app.UseSpaStaticFiles();

app.MapControllerRoute(
    name: "default",
    pattern: "{controller}/{action=Index}/{id?}");

app.UseSpa(spa =>
{
    spa.Options.SourcePath = "ClientApp";
    
    if (app.Environment.IsDevelopment())
    {
        spa.UseProxyToSpaDevelopmentServer("http://localhost:5173"); // Vite dev server
    }
});

// API Controller that the SPA calls
[ApiController]
[Route("api/[controller]")]
public class ApiProductsController : ControllerBase
{
    // Returns JSON, not views
    [HttpGet]
    public async Task<ActionResult<IEnumerable<ProductDto>>> GetAll()
    {
        var products = await _productService.GetAllAsync();
        return Ok(products);
    }
}
```

---

## Step 686: Testing MVC Controllers

```csharp
// Tests/Controllers/ProductsControllerTests.cs
using Microsoft.AspNetCore.Mvc;
using Moq;
using Xunit;

namespace MyMvcApp.Tests.Controllers;

public class ProductsControllerTests
{
    private readonly Mock<IProductService> _mockService;
    private readonly ProductsController _controller;

    public ProductsControllerTests()
    {
        _mockService = new Mock<IProductService>();
        _controller = new ProductsController(
            _mockService.Object,
            Mock.Of<ILogger<ProductsController>>());
    }

    [Fact]
    public async Task Index_ReturnsViewWithProducts()
    {
        // Arrange
        var products = new List<ProductSummary>
        {
            new() { Id = 1, Name = "Product A", Price = 10.00m },
            new() { Id = 2, Name = "Product B", Price = 20.00m }
        };

        _mockService.Setup(s => s.GetPagedAsync(null, 1, 10))
            .ReturnsAsync((products, 2));

        // Act
        var result = await _controller.Index(null, 1);

        // Assert
        var viewResult = Assert.IsType<ViewResult>(result);
        var model = Assert.IsType<ProductListViewModel>(viewResult.Model);
        Assert.Equal(2, model.Products.Count());
    }

    [Fact]
    public async Task Details_ReturnsNotFound_WhenProductMissing()
    {
        _mockService.Setup(s => s.GetByIdAsync(999))
            .ReturnsAsync((ProductSummary?)null);

        var result = await _controller.Details(999);

        Assert.IsType<NotFoundResult>(result);
    }

    [Fact]
    public async Task Create_RedirectsToIndex_OnSuccess()
    {
        var model = new CreateProductViewModel 
        { 
            Name = "New Product", 
            Price = 15.00m,
            Stock = 100,
            CategoryId = 1
        };

        _mockService.Setup(s => s.CreateAsync(model))
            .ReturnsAsync(new Product { Id = 1 });

        var result = await _controller.Create(model);

        var redirect = Assert.IsType<RedirectToActionResult>(result);
        Assert.Equal("Index", redirect.ActionName);
    }

    [Fact]
    public async Task Create_ReturnsView_WhenModelInvalid()
    {
        _controller.ModelState.AddModelError("Name", "Required");
        var model = new CreateProductViewModel();

        var result = await _controller.Create(model);

        var viewResult = Assert.IsType<ViewResult>(result);
        Assert.Equal(model, viewResult.Model);
    }
}

// Integration Tests
public class ProductsIntegrationTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;

    public ProductsIntegrationTests(WebApplicationFactory<Program> factory)
    {
        _client = factory.WithWebHostBuilder(builder =>
        {
            builder.ConfigureServices(services =>
            {
                services.AddScoped<IProductService, TestProductService>();
            });
        }).CreateClient();
    }

    [Fact]
    public async Task Index_ReturnsSuccessStatusCode()
    {
        var response = await _client.GetAsync("/Products");
        response.EnsureSuccessStatusCode();
        
        var content = await response.Content.ReadAsStringAsync();
        Assert.Contains("Products", content);
    }
}
```

---

## Step 687: Output Caching (.NET 7+)

```csharp
// Program.cs
builder.Services.AddOutputCache(options =>
{
    options.AddBasePolicy(builder => builder.Cache());
    
    options.AddPolicy("ProductsCache", builder =>
        builder.Expire(TimeSpan.FromMinutes(5))
               .Tag("products")
               .SetVaryByQuery("search", "page"));
    
    options.AddPolicy("StaticContent", builder =>
        builder.Expire(TimeSpan.FromHours(1))
               .SetVaryByHeader("Accept-Encoding"));
});

app.UseOutputCache();

// Controller usage
[OutputCache(PolicyName = "ProductsCache")]
public async Task<IActionResult> Index(string? search, int page = 1) { ... }

// Evict cache programmatically
public class ProductsController : Controller
{
    private readonly IOutputCacheStore _outputCacheStore;

    [HttpPost]
    public async Task<IActionResult> Create(CreateProductViewModel model)
    {
        // after creating product, invalidate the cache
        await _outputCacheStore.EvictByTagAsync("products", default);
        return RedirectToAction("Index");
    }
}
```

---

## Step 688: Bundling และ Minification

```json
// bundleconfig.json
[
    {
        "outputFileName": "wwwroot/css/bundle.min.css",
        "inputFiles": [
            "wwwroot/lib/bootstrap/dist/css/bootstrap.min.css",
            "wwwroot/css/site.css"
        ]
    },
    {
        "outputFileName": "wwwroot/js/bundle.min.js",
        "inputFiles": [
            "wwwroot/lib/jquery/dist/jquery.min.js",
            "wwwroot/lib/bootstrap/dist/js/bootstrap.bundle.min.js",
            "wwwroot/js/site.js"
        ],
        "minify": {
            "enabled": true
        }
    }
]
```

```xml
<!-- MyMvcApp.csproj -->
<Project Sdk="Microsoft.NET.Sdk.Web">
    <PropertyGroup>
        <TargetFramework>net9.0</TargetFramework>
    </PropertyGroup>
    
    <ItemGroup>
        <PackageReference Include="BuildBundlerMinifier" Version="3.*" />
        <PackageReference Include="Microsoft.AspNetCore.Mvc.Razor.RuntimeCompilation" Version="9.*" />
    </ItemGroup>
</Project>
```

```csharp
// Program.cs - Enable Razor Runtime Compilation for Development
if (builder.Environment.IsDevelopment())
{
    builder.Services.AddControllersWithViews()
        .AddRazorRuntimeCompilation();
}
```

---

## Step 689: SEO Optimization

```html
<!-- Views/Shared/_Layout.cshtml - SEO Meta Tags -->
<!DOCTYPE html>
<html lang="@CultureInfo.CurrentCulture.TwoLetterISOLanguageName">
<head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>@ViewData["Title"] - @ViewData["SiteName"]</title>
    
    @if (ViewData["Description"] != null)
    {
        <meta name="description" content="@ViewData["Description"]" />
    }
    
    @if (ViewData["Keywords"] != null)
    {
        <meta name="keywords" content="@ViewData["Keywords"]" />
    }
    
    <!-- Open Graph -->
    <meta property="og:title" content="@ViewData["OgTitle"] ?? ViewData["Title"]" />
    <meta property="og:description" content="@ViewData["OgDescription"] ?? ViewData["Description"]" />
    @if (ViewData["OgImage"] != null)
    {
        <meta property="og:image" content="@ViewData["OgImage"]" />
    }
    <meta property="og:url" content="@Context.Request.GetDisplayUrl()" />
    
    <!-- Canonical URL -->
    @if (ViewData["CanonicalUrl"] != null)
    {
        <link rel="canonical" href="@ViewData["CanonicalUrl"]" />
    }
    
    <link rel="stylesheet" href="~/css/bundle.min.css" asp-append-version="true" />
</head>

<!-- Structured Data (JSON-LD) -->
@if (ViewData["StructuredData"] != null)
{
    <script type="application/ld+json">
        @Html.Raw(ViewData["StructuredData"]?.ToString())
    </script>
}
```

```csharp
// Controller setting SEO metadata
public async Task<IActionResult> Details(int id)
{
    var product = await _productService.GetByIdAsync(id);
    if (product is null) return NotFound();

    ViewData["Title"] = product.Name;
    ViewData["Description"] = product.Description?.Substring(0, Math.Min(160, product.Description.Length));
    ViewData["OgImage"] = product.ImageUrl;
    ViewData["CanonicalUrl"] = Url.Action("Details", "Products", new { id }, Request.Scheme);
    
    // JSON-LD structured data for products
    var structuredData = new
    {
        @context = "https://schema.org/",
        @type = "Product",
        name = product.Name,
        description = product.Description,
        image = product.ImageUrl,
        offers = new
        {
            @type = "Offer",
            price = product.Price,
            priceCurrency = "USD",
            availability = product.Stock > 0 
                ? "https://schema.org/InStock" 
                : "https://schema.org/OutOfStock"
        }
    };
    
    ViewData["StructuredData"] = JsonSerializer.Serialize(structuredData);
    
    return View(product);
}
```

---

## Step 690: Performance Optimization

```csharp
// Program.cs - Performance Configuration
builder.Services.AddResponseCompression(options =>
{
    options.EnableForHttps = true;
    options.Providers.Add<BrotliCompressionProvider>();
    options.Providers.Add<GzipCompressionProvider>();
});

builder.Services.Configure<BrotliCompressionProviderOptions>(options =>
{
    options.Level = CompressionLevel.Fastest;
});

// Distributed SQL Server Session
builder.Services.AddDistributedSqlServerCache(options =>
{
    options.ConnectionString = connectionString;
    options.SchemaName = "dbo";
    options.TableName = "Cache";
});

builder.Services.AddSession(options =>
{
    options.IdleTimeout = TimeSpan.FromMinutes(20);
    options.Cookie.HttpOnly = true;
    options.Cookie.IsEssential = true;
});

// Minimize allocations with Span<T>
public static class StringHelper
{
    public static ReadOnlySpan<char> TruncateDescription(string description, int maxLength)
    {
        if (description.Length <= maxLength)
            return description.AsSpan();
        
        var span = description.AsSpan(0, maxLength);
        var lastSpace = span.LastIndexOf(' ');
        return lastSpace > 0 ? span[..lastSpace] : span;
    }
}
```

---

## Step 691: Email Service ใน MVC

```csharp
// Services/EmailService.cs
using MailKit.Net.Smtp;
using MimeKit;

namespace MyMvcApp.Services;

public interface IEmailService
{
    Task SendAsync(string to, string subject, string body, bool isHtml = true);
    Task SendTemplateAsync<T>(string to, string subject, string templateName, T model);
}

public class EmailService : IEmailService
{
    private readonly EmailSettings _settings;
    private readonly ILogger<EmailService> _logger;
    private readonly RazorViewToStringRenderer _renderer;

    public EmailService(
        IOptions<EmailSettings> settings, 
        ILogger<EmailService> logger,
        RazorViewToStringRenderer renderer)
    {
        _settings = settings.Value;
        _logger = logger;
        _renderer = renderer;
    }

    public async Task SendAsync(string to, string subject, string body, bool isHtml = true)
    {
        var message = new MimeMessage();
        message.From.Add(new MailboxAddress(_settings.SenderName, _settings.SenderEmail));
        message.To.Add(MailboxAddress.Parse(to));
        message.Subject = subject;
        message.Body = isHtml 
            ? new TextPart("html") { Text = body }
            : new TextPart("plain") { Text = body };

        using var client = new SmtpClient();
        await client.ConnectAsync(_settings.Host, _settings.Port, _settings.UseSsl);
        await client.AuthenticateAsync(_settings.Username, _settings.Password);
        await client.SendAsync(message);
        await client.DisconnectAsync(true);
        
        _logger.LogInformation("Email sent to {To}: {Subject}", to, subject);
    }

    public async Task SendTemplateAsync<T>(string to, string subject, string templateName, T model)
    {
        var body = await _renderer.RenderViewToStringAsync($"Emails/{templateName}", model);
        await SendAsync(to, subject, body);
    }
}

// RazorViewToStringRenderer.cs
public class RazorViewToStringRenderer
{
    private readonly IRazorViewEngine _viewEngine;
    private readonly ITempDataProvider _tempDataProvider;
    private readonly IServiceProvider _serviceProvider;

    public RazorViewToStringRenderer(
        IRazorViewEngine viewEngine,
        ITempDataProvider tempDataProvider,
        IServiceProvider serviceProvider)
    {
        _viewEngine = viewEngine;
        _tempDataProvider = tempDataProvider;
        _serviceProvider = serviceProvider;
    }

    public async Task<string> RenderViewToStringAsync<T>(string viewName, T model)
    {
        var httpContext = new DefaultHttpContext { RequestServices = _serviceProvider };
        var actionContext = new ActionContext(httpContext, new RouteData(), new ActionDescriptor());

        using var sw = new StringWriter();
        var viewResult = _viewEngine.FindView(actionContext, viewName, false);
        
        if (!viewResult.Success)
            throw new InvalidOperationException($"View '{viewName}' not found.");

        var viewDictionary = new ViewDataDictionary<T>(new EmptyModelMetadataProvider(), new ModelStateDictionary())
        {
            Model = model
        };

        var viewContext = new ViewContext(
            actionContext,
            viewResult.View,
            viewDictionary,
            new TempDataDictionary(httpContext, _tempDataProvider),
            sw,
            new HtmlHelperOptions());

        await viewResult.View.RenderAsync(viewContext);
        return sw.ToString();
    }
}
```

---

## Step 692: Background Jobs ใน MVC

```csharp
// Services/BackgroundJobService.cs
using Hangfire;

// Program.cs - Hangfire Setup
builder.Services.AddHangfire(config =>
    config.UseSqlServerStorage(connectionString));
builder.Services.AddHangfireServer();

app.UseHangfireDashboard("/admin/jobs", new DashboardOptions
{
    Authorization = [new HangfireAdminAuthFilter()]
});

// Job Definitions
public interface IEmailJob
{
    Task SendWelcomeEmailAsync(string userId);
    Task SendOrderConfirmationAsync(int orderId);
}

public class EmailJob : IEmailJob
{
    private readonly IEmailService _emailService;
    private readonly IUserService _userService;

    public EmailJob(IEmailService emailService, IUserService userService)
    {
        _emailService = emailService;
        _userService = userService;
    }

    public async Task SendWelcomeEmailAsync(string userId)
    {
        var user = await _userService.GetByIdAsync(userId);
        if (user is not null)
            await _emailService.SendTemplateAsync(user.Email, "Welcome!", "WelcomeEmail", user);
    }

    public async Task SendOrderConfirmationAsync(int orderId)
    {
        // ...
    }
}

// Scheduling Jobs
public class OrderService
{
    private readonly IBackgroundJobClient _backgroundJob;
    private readonly IRecurringJobManager _recurringJob;

    public OrderService(IBackgroundJobClient backgroundJob, IRecurringJobManager recurringJob)
    {
        _backgroundJob = backgroundJob;
        _recurringJob = recurringJob;
    }

    public async Task CreateOrderAsync(OrderDto dto)
    {
        var order = await SaveOrderAsync(dto);
        
        // Fire and forget
        _backgroundJob.Enqueue<IEmailJob>(job => job.SendOrderConfirmationAsync(order.Id));
        
        // Delayed job
        _backgroundJob.Schedule<IEmailJob>(
            job => job.SendOrderConfirmationAsync(order.Id),
            TimeSpan.FromMinutes(5));
    }

    public void SetupRecurringJobs()
    {
        // Daily at midnight
        _recurringJob.AddOrUpdate<IReportService>(
            "daily-report", 
            svc => svc.GenerateDailyReport(), 
            Cron.Daily);
        
        // Every hour
        _recurringJob.AddOrUpdate<ICacheService>(
            "cache-warmup",
            svc => svc.WarmUpAsync(),
            Cron.Hourly);
    }
}
```

---

## Step 693: Multi-tenant Architecture

```csharp
// Services/TenantContext.cs
public interface ITenantContext
{
    string TenantId { get; }
    TenantConfig Config { get; }
}

public class TenantContext : ITenantContext
{
    public string TenantId { get; private set; } = "default";
    public TenantConfig Config { get; private set; } = new();

    public void SetTenant(string tenantId, TenantConfig config)
    {
        TenantId = tenantId;
        Config = config;
    }
}

public record TenantConfig(
    string Name,
    string ConnectionString,
    string Theme,
    bool IsPremium);

// Middleware/TenantResolutionMiddleware.cs
public class TenantResolutionMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ITenantResolver _tenantResolver;

    public async Task InvokeAsync(HttpContext context, ITenantContext tenantContext)
    {
        var host = context.Request.Host.Host;
        var tenantId = host.Split('.').First();
        
        var config = await _tenantResolver.ResolveAsync(tenantId);
        ((TenantContext)tenantContext).SetTenant(tenantId, config);
        
        await _next(context);
    }
}

// Multi-tenant DbContext
public class TenantDbContext : DbContext
{
    private readonly ITenantContext _tenant;

    public TenantDbContext(DbContextOptions options, ITenantContext tenant) 
        : base(options) 
    {
        _tenant = tenant;
    }

    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        if (!optionsBuilder.IsConfigured)
            optionsBuilder.UseSqlServer(_tenant.Config.ConnectionString);
    }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Apply global query filter for tenant isolation
        foreach (var entityType in modelBuilder.Model.GetEntityTypes())
        {
            if (typeof(ITenantEntity).IsAssignableFrom(entityType.ClrType))
            {
                var method = typeof(TenantDbContext)
                    .GetMethod(nameof(ApplyTenantFilter), 
                               System.Reflection.BindingFlags.NonPublic | System.Reflection.BindingFlags.Static)!
                    .MakeGenericMethod(entityType.ClrType);
                
                method.Invoke(null, [modelBuilder, _tenant.TenantId]);
            }
        }
    }

    private static void ApplyTenantFilter<T>(ModelBuilder builder, string tenantId) 
        where T : class, ITenantEntity
    {
        builder.Entity<T>().HasQueryFilter(e => e.TenantId == tenantId);
    }
}

public interface ITenantEntity
{
    string TenantId { get; }
}
```

---

## Step 694: API + MVC Hybrid App

```csharp
// Program.cs - Hybrid App Setup
builder.Services.AddControllersWithViews();  // MVC Controllers + Views
builder.Services.AddControllers();           // API Controllers (JSON only)

// Swagger for API portion
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// Separate routes
app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}");

// API controllers at /api prefix
app.MapControllers();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI(c => c.SwaggerEndpoint("/swagger/v1/swagger.json", "API V1"));
}

// MVC Controller (returns Views)
public class ProductsController : Controller
{
    public IActionResult Index() => View();  // Returns HTML
}

// API Controller (returns JSON)
[ApiController]
[Route("api/[controller]")]
public class ApiProductsController : ControllerBase
{
    [HttpGet]
    public async Task<ActionResult<PagedResponse<ProductDto>>> GetAll([FromQuery] ProductFilter filter)
    {
        // Returns JSON
    }
}
```

---

## Step 695: Globalization - ระบบสกุลเงิน/วันที่

```csharp
// Services/CurrencyService.cs
public interface ICurrencyService
{
    string Format(decimal amount, string currencyCode = "USD");
    decimal Convert(decimal amount, string from, string to);
}

public class CurrencyService : ICurrencyService
{
    private readonly Dictionary<string, decimal> _exchangeRates = new()
    {
        ["USD"] = 1.0m,
        ["THB"] = 35.5m,
        ["JPY"] = 149.5m,
        ["EUR"] = 0.92m,
    };

    public string Format(decimal amount, string currencyCode = "USD")
    {
        var culture = currencyCode switch
        {
            "THB" => new System.Globalization.CultureInfo("th-TH"),
            "JPY" => new System.Globalization.CultureInfo("ja-JP"),
            "EUR" => new System.Globalization.CultureInfo("de-DE"),
            _ => new System.Globalization.CultureInfo("en-US")
        };
        
        return amount.ToString("C", culture);
    }

    public decimal Convert(decimal amount, string from, string to)
    {
        var fromRate = _exchangeRates.GetValueOrDefault(from, 1m);
        var toRate = _exchangeRates.GetValueOrDefault(to, 1m);
        return amount / fromRate * toRate;
    }
}

// Extension Methods for Views
public static class HtmlHelperExtensions
{
    public static string FormatCurrency(this IHtmlHelper html, decimal amount, string currency = "USD")
    {
        var service = html.ViewContext.HttpContext.RequestServices
            .GetRequiredService<ICurrencyService>();
        return service.Format(amount, currency);
    }

    public static string FormatDate(this IHtmlHelper html, DateTime date, string format = "short")
    {
        var culture = System.Threading.Thread.CurrentThread.CurrentCulture;
        return format switch
        {
            "long" => date.ToString("D", culture),
            "time" => date.ToString("t", culture),
            _ => date.ToString("d", culture)
        };
    }
}
```

```html
<!-- Views/Products/Details.cshtml -->
@model Product
@inject ICurrencyService CurrencyService

<h2>@Model.Name</h2>
<p class="price">@Html.FormatCurrency(Model.Price)</p>
<p class="date">Added: @Html.FormatDate(Model.CreatedAt, "long")</p>

<!-- Using inject directly -->
<p>Price in THB: @CurrencyService.Format(CurrencyService.Convert(Model.Price, "USD", "THB"), "THB")</p>
```

---

## Step 696: SSR vs Client-side Rendering

```csharp
// Program.cs - Mixed SSR + CSR
// SSR routes go through MVC
app.MapControllerRoute(
    name: "ssr",
    pattern: "{controller=Home}/{action=Index}/{id?}");

// SPA routes serve index.html
app.MapFallbackToFile("index.html");  // For React/Vue/Angular routes

// HybridController: decides SSR or CSR based on request
public class HybridController : Controller
{
    public IActionResult Index()
    {
        // If request wants JSON (AJAX/SPA), return API response
        if (Request.Headers.Accept.Contains("application/json"))
            return Json(new { title = "Home", data = "..." });

        // Otherwise SSR HTML
        return View();
    }
}

// Turbo / Hotwire Integration (SSR streaming)
public class StreamingController : Controller
{
    [HttpGet]
    public async IAsyncEnumerable<string> Products(
        [System.Runtime.CompilerServices.EnumeratorCancellation] CancellationToken ct)
    {
        var products = _productService.GetStreamAsync(ct);
        
        await foreach (var product in products.WithCancellation(ct))
        {
            var html = await _renderer.RenderPartialAsync("_ProductCard", product);
            yield return $"<turbo-stream action=\"append\" target=\"products\"><template>{html}</template></turbo-stream>";
            await Task.Delay(10, ct);  // Simulate stream
        }
    }
}
```

---

## Step 697: Advanced Razor Features

```html
<!-- Custom Razor Directives -->
@functions {
    private string GetBadgeClass(int stock) => stock switch
    {
        0 => "danger",
        <= 5 => "warning",
        _ => "success"
    };
    
    private string FormatRelativeDate(DateTime date)
    {
        var diff = DateTime.UtcNow - date;
        return diff.TotalDays switch
        {
            < 1 => $"{(int)diff.TotalHours}h ago",
            < 7 => $"{(int)diff.TotalDays}d ago",
            < 30 => $"{(int)(diff.TotalDays / 7)}w ago",
            _ => date.ToString("MMM dd, yyyy")
        };
    }
}

<!-- Razor Component-like Patterns -->
@{
    void RenderCard(ProductSummary p)
    {
        <div class="card mb-2">
            <div class="card-body">
                <h6>@p.Name</h6>
                <span class="badge bg-@GetBadgeClass(p.Stock)">
                    @(p.Stock > 0 ? $"{p.Stock} in stock" : "Out of stock")
                </span>
            </div>
        </div>
    }
}

@foreach (var product in Model.Products)
{
    @{ RenderCard(product); }
}

<!-- Conditional Layout -->
@{
    Layout = User.IsInRole("Admin") ? "_AdminLayout" : "_Layout";
}

<!-- Section with Default Content -->
@section Scripts {
    @await RenderSectionAsync("PageScripts", required: false)
    @* Default scripts for all pages *@
    <script>
        // Common page JS
        document.querySelectorAll('[data-confirm]').forEach(el => {
            el.addEventListener('click', e => {
                if (!confirm(el.dataset.confirm)) e.preventDefault();
            });
        });
    </script>
}
```

---

## Step 698: Advanced Model Validation

```csharp
// Custom Validation with IValidatableObject
public class CreateOrderViewModel : IValidatableObject
{
    [Required]
    public int CustomerId { get; set; }
    
    [Required]
    [MinLength(1, ErrorMessage = "Order must have at least one item")]
    public List<OrderItemViewModel> Items { get; set; } = [];
    
    public DateTime? DeliveryDate { get; set; }
    public string? PromoCode { get; set; }

    public IEnumerable<ValidationResult> Validate(ValidationContext validationContext)
    {
        if (DeliveryDate.HasValue && DeliveryDate.Value < DateTime.Today.AddDays(1))
        {
            yield return new ValidationResult(
                "Delivery date must be at least tomorrow",
                [nameof(DeliveryDate)]);
        }

        if (Items.Sum(i => i.Quantity) > 100)
        {
            yield return new ValidationResult(
                "Total quantity cannot exceed 100 items",
                [nameof(Items)]);
        }

        if (!string.IsNullOrEmpty(PromoCode))
        {
            var promoService = validationContext.GetService<IPromoCodeService>();
            if (promoService?.IsValid(PromoCode) == false)
            {
                yield return new ValidationResult(
                    "Invalid promo code",
                    [nameof(PromoCode)]);
            }
        }
    }
}

// Remote Validation
public class ProductViewModel
{
    [Required]
    [Remote(action: "ValidateProductName", controller: "Products",
            AdditionalFields = nameof(Id),
            ErrorMessage = "Product name already exists")]
    public string Name { get; set; } = string.Empty;
    
    public int Id { get; set; }
}

// Validation action
[AcceptVerbs("GET", "POST")]
public async Task<IActionResult> ValidateProductName(string name, int id)
{
    var exists = await _productService.NameExistsAsync(name, excludeId: id);
    return exists 
        ? Json($"Product name '{name}' is already taken")
        : Json(true);
}
```

---

## Step 699: Performance Profiling

```csharp
// Program.cs - MiniProfiler for Development
if (builder.Environment.IsDevelopment())
{
    builder.Services.AddMiniProfiler(options =>
    {
        options.RouteBasePath = "/profiler";
        options.PopupRenderPosition = StackExchange.Profiling.RenderPosition.BottomLeft;
        options.PopupShowTimeWithChildren = true;
    }).AddEntityFramework();
}

// Using MiniProfiler in code
public async Task<IActionResult> ExpensiveAction()
{
    using (MiniProfiler.Current.Step("Loading Data"))
    {
        using (MiniProfiler.Current.Step("Database Query"))
        {
            var data = await _repository.GetExpensiveDataAsync();
        }
        
        using (MiniProfiler.Current.Step("Processing"))
        {
            // Process data
        }
    }
    return View();
}

// In _Layout.cshtml:
// @using StackExchange.Profiling
// @MiniProfiler.Current.RenderIncludes(Context)
```

---

## Step 700: Production Deployment

```bash
# Publish for production
dotnet publish -c Release -o ./publish

# Docker deployment
# Dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:9.0 AS base
WORKDIR /app
EXPOSE 8080

FROM mcr.microsoft.com/dotnet/sdk:9.0 AS build
WORKDIR /src
COPY ["MyMvcApp.csproj", "."]
RUN dotnet restore
COPY . .
RUN dotnet build -c Release -o /app/build

FROM build AS publish
RUN dotnet publish -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "MyMvcApp.dll"]
```

```yaml
# docker-compose.yml
version: '3.8'
services:
  web:
    build: .
    ports:
      - "80:8080"
    environment:
      - ASPNETCORE_ENVIRONMENT=Production
      - ConnectionStrings__Default=Server=db;Database=MyMvcApp;...
      - ASPNETCORE_URLS=http://+:8080
    depends_on:
      - db
      - redis
    
  db:
    image: mcr.microsoft.com/mssql/server:2022-latest
    environment:
      - SA_PASSWORD=YourPassword123!
      - ACCEPT_EULA=Y
    volumes:
      - sqldata:/var/opt/mssql

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  sqldata:
```

```csharp
// appsettings.Production.json
{
    "Logging": {
        "LogLevel": {
            "Default": "Warning",
            "Microsoft.AspNetCore": "Warning"
        }
    },
    "AllowedHosts": "*.myapp.com",
    "ConnectionStrings": {
        "Default": "Managed via secrets"
    },
    "Kestrel": {
        "Endpoints": {
            "Http": {
                "Url": "http://+:8080"
            },
            "Https": {
                "Url": "https://+:8443",
                "Certificate": {
                    "Path": "/certs/cert.pfx",
                    "Password": "cert-password"
                }
            }
        }
    }
}

// Health check endpoints for load balancer
app.MapHealthChecks("/health");
app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready")
});
app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = _ => false  // Always healthy if the app is running
});
```

---

## สรุป Part 25: ASP.NET Core MVC

ในส่วนนี้เราได้เรียนรู้:

| หัวข้อ | ขั้นตอน |
|--------|---------|
| MVC Pattern และ Project Structure | 661 |
| Controllers (Home, Products CRUD) | 662 |
| Models และ ViewModels | 663 |
| Razor Layout (_Layout.cshtml) | 664 |
| Views (Index, Create forms) | 665-666 |
| Tag Helpers (Custom, Active Route, Gravatar) | 667 |
| Partial Views และ View Components | 668-669 |
| Razor Pages (ทางเลือก MVC) | 670 |
| Model Binding Advanced | 671 |
| Filters (Action, Authorization) | 672 |
| Areas (Admin Area) | 673 |
| Authentication (ASP.NET Core Identity) | 674 |
| Authorization Policies (Resource-based) | 675 |
| Form Validation (DataAnnotations + Custom) | 676 |
| File Upload | 677 |
| Caching (Memory, Distributed, Output) | 678, 687 |
| Localization | 679 |
| Scaffolding | 680 |
| Middleware | 681 |
| Error Handling | 682 |
| HTMX Integration | 683 |
| SignalR Integration | 684 |
| SPA Integration | 685 |
| Testing Controllers | 686 |
| Bundling & Minification | 688 |
| SEO Optimization | 689 |
| Performance | 690 |
| Email Service (Razor Templates) | 691 |
| Background Jobs (Hangfire) | 692 |
| Multi-tenant Architecture | 693 |
| API + MVC Hybrid | 694 |
| Currency/Date Globalization | 695 |
| SSR vs CSR | 696 |
| Advanced Razor Features | 697 |
| Advanced Model Validation | 698 |
| Performance Profiling | 699 |
| Production Deployment | 700 |

**ขั้นตอนต่อไป**: Part 26 จะเรียนรู้ **Blazor** - การสร้างเว็บแอปแบบ interactive ด้วย C# แทน JavaScript
