```
### User
Great lets start to build this project , my plan is to build using VS code using dotnet commands , so lets start from asp.net core empty project , configure the things for razor pages
and lets follow MVC architecture and use Javascript AJAX code to interaract with Controllers.
Lets Create Entities , DTOs,Service , Repository , Controller level of folder structure and use Views folder as well as with proper _layout.cshtml file to handle UI things properly 
```

---

Of course. Let's start building the "CraRental" management system from the ground up using `dotnet` commands in VS Code.

This guide will walk you through creating an ASP.NET Core empty project, scaffolding the necessary folders for a clean architecture (Entities, DTOs, Services, Repositories, Controllers), and setting up Razor Pages with a proper layout file.

### Step 1: Create and Configure the ASP.NET Core Project

1.  **Open your terminal or command prompt** in the directory where you want to create your project.

2.  **Create a new ASP.NET Core Empty project:**

    ```bash
    dotnet new web -n CraRental.Web
    ```

    This command creates a new project named `CraRental.Web`.

3.  **Navigate into the project directory:**

    ```bash
    cd CraRental.Web
    ```

4.  **Open the project in Visual Studio Code:**

    ```bash
    code .
    ```

5.  **Configure `Program.cs` for MVC and Razor Pages:**

    Open the `Program.cs` file and replace its content with the following. This code adds the necessary services for controllers and Razor Pages, and configures the middleware to handle routing and serve static files.

    ```csharp
    var builder = WebApplication.CreateBuilder(args);

    // 1. Add services to the container.
    builder.Services.AddControllersWithViews(); // For MVC Controllers
    builder.Services.AddRazorPages();           // For Razor Pages

    var app = builder.Build();

    // 2. Configure the HTTP request pipeline.
    if (!app.Environment.IsDevelopment())
    {
        app.UseExceptionHandler("/Home/Error");
        app.UseHsts();
    }

    app.UseHttpsRedirection();
    app.UseStaticFiles(); // Enable serving static files like JS, CSS

    app.UseRouting();

    app.UseAuthorization();

    // 3. Map routes for controllers and razor pages
    app.MapControllerRoute(
        name: "default",
        pattern: "{controller=Home}/{action=Index}/{id?}");
    app.MapRazorPages();

    app.Run();
    ```

### Step 2: Create the Folder Structure

In the root of your `CraRental.Web` project, create the following folders to organize your code according to the planned architecture:

*   **Controllers:** To handle incoming HTTP requests.
*   **Data:** To hold database-related files, including your DbContext.
*   **DTOs (Data Transfer Objects):** To shape data for views.
*   **Entities:** For your database models (POCO classes).
*   **Repositories:** For the data access layer.
*   **Services:** For your business logic.
*   **Views:** For the UI, containing Razor files.
    *   Inside `Views`, create a **Shared** folder.
*   **wwwroot:** This folder should already exist. Inside it, create:
    *   **css:** For your stylesheets.
    *   **js:** For your JavaScript files.

Your folder structure should now look like this:

```
CraRental.Web/
├── Controllers/
├── Data/
├── DTOs/
├── Entities/
├── Repositories/
├── Services/
├── Views/
│   └── Shared/
├── wwwroot/
│   ├── css/
│   └── js/
├── appsettings.json
├── Program.cs
└── CraRental.Web.csproj
```

### Step 3: Define the Core Entities

Let's create the first two entity classes based on our database plan.

1.  **Create `User.cs` in the `Entities` folder:**

    ```csharp
    // CraRental.Web/Entities/User.cs
    namespace CraRental.Web.Entities
    {
        public class User
        {
            public int Id { get; set; }
            public string Email { get; set; }
            public string PasswordHash { get; set; }
            public string Role { get; set; } // "Admin", "Staff", "CarOwner", "CarRenter"
            public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
            public DateTime? UpdatedAt { get; set; }
        }
    }
    ```

2.  **Create `CarOwner.cs` in the `Entities` folder:**

    ```csharp
    // CraRental.Web/Entities/CarOwner.cs
    using System.ComponentModel.DataAnnotations.Schema;

    namespace CraRental.Web.Entities
    {
        public class CarOwner
        {
            public int Id { get; set; }
            public string FirstName { get; set; }
            public string LastName { get; set; }
            public string? PhoneNumber { get; set; }
            public string? Address { get; set; }

            // Foreign Key to the User table
            [ForeignKey("User")]
            public int UserId { get; set; }
            public virtual User User { get; set; }
        }
    }
    ```

### Step 4: Create the Main Layout and Home Page

Now, let's set up the basic UI structure.

1.  **Create `_Layout.cshtml` in the `Views/Shared` folder:**

    This file will serve as the master template for your application's pages, containing the common HTML structure, header, and footer.

    ```html
    <!DOCTYPE html>
    <html lang="en">
    <head>
        <meta charset="utf-8" />
        <meta name="viewport" content="width=device-width, initial-scale=1.0" />
        <title>@ViewData["Title"] - CraRental</title>
        <link rel="stylesheet" href="~/css/site.css" />
    </head>
    <body>
        <header>
            <nav>
                <div class="container">
                    <a class="navbar-brand" asp-area="" asp-controller="Home" asp-action="Index">CraRental</a>
                    <!-- Navigation links will go here -->
                </div>
            </nav>
        </header>
        <div class="container">
            <main role="main" class="pb-3">
                @RenderBody()
            </main>
        </div>

        <footer class="footer">
            <div class="container">
                &copy; 2025 - CraRental Management System
            </div>
        </footer>

        <script src="~/js/site.js" asp-append-version="true"></script>
        @await RenderSectionAsync("Scripts", required: false)
    </body>
    </html>
    ```

2.  **Create `_ViewStart.cshtml` in the `Views` folder:**

    This file ensures that the `_Layout.cshtml` is used for all views by default.

    ```csharp
    @{
        Layout = "_Layout";
    }
    ```

3.  **Create your first Controller:**

    Create a new file named `HomeController.cs` inside the `Controllers` folder. This will be the entry point of your application.

    ```csharp
    // CraRental.Web/Controllers/HomeController.cs
    using Microsoft.AspNetCore.Mvc;

    namespace CraRental.Web.Controllers
    {
        public class HomeController : Controller
        {
            public IActionResult Index()
            {
                return View();
            }
        }
    }
    ```

4.  **Create the corresponding View for the Index action:**

    *   Inside the `Views` folder, create a new folder named `Home`.
    *   Inside `Views/Home`, create a new file named `Index.cshtml`.

    ```html
    @{
        ViewData["Title"] = "Home Page";
    }

    <div class="text-center">
        <h1 class="display-4">Welcome to CraRental</h1>
        <p>The best car rental management system.</p>
    </div>
    ```

### Step 5: Add Basic Styling and JavaScript Files

1.  In `wwwroot/css`, create `site.css` with some basic styles.
2.  In `wwwroot/js`, create `site.js` for your global JavaScript.

You can leave these files empty for now.

### Step 6: Run the Application

You are now ready for the first run. Go to your terminal and execute:

```bash
dotnet watch run
```

The `watch` command will automatically rebuild and restart the application whenever you save a file.

Open your browser and navigate to the URL provided in the terminal (usually `https://localhost:7001` or similar). You should see your "Welcome to CraRental" home page with the layout you created.

You have successfully set up the foundation of your project with a clean architecture. The next steps will involve setting up the database with Entity Framework, creating repositories and services, and implementing the token-based authentication.
