
"**User:** 
Ok then, let's create all the entities we need for this project. Also, let's create `Data/ApplicationDbContext.cs` file and configure the entities relationship as well."

---
Excellent. Let's create all the entity classes for your project and then set up the `ApplicationDbContext` to manage them and define their relationships.

### Step 1: Install Entity Framework Core Packages

First, you need to add the necessary EF Core packages to your project. Open your terminal in the `CraRental.Web` directory and run these commands:

1.  **SQLite Provider:** This package allows EF Core to work with a SQLite database.
    ```bash
    dotnet add package Microsoft.EntityFrameworkCore.Sqlite
    ```

2.  **EF Core Tools:** This provides command-line tools for migrations and database updates.
    ```bash
    dotnet add package Microsoft.EntityFrameworkCore.Tools
    ```

### Step 2: Create All Entity Classes

We've already created `User.cs` and `CarOwner.cs`. Now, let's create the rest of the entities inside the `Entities` folder.

**1. Create `CarRenter.cs`**

This entity holds information specific to users who rent cars.

```csharp
// CraRental.Web/Entities/CarRenter.cs
using System.ComponentModel.DataAnnotations.Schema;

namespace CraRental.Web.Entities
{
    public class CarRenter
    {
        public int Id { get; set; }
        public string FirstName { get; set; }
        public string LastName { get; set; }
        public string? PhoneNumber { get; set; }
        public string? DriverLicenseNumber { get; set; }

        // Foreign Key to the User table
        [ForeignKey("User")]
        public int UserId { get; set; }
        public virtual User User { get; set; }
    }
}
```

**2. Create `Car.cs`**

This entity represents a car available for rent.

```csharp
// CraRental.Web/Entities/Car.cs
using System.ComponentModel.DataAnnotations.Schema;

namespace CraRental.Web.Entities
{
    public class Car
    {
        public int Id { get; set; }
        public string Make { get; set; } // e.g., Toyota
        public string Model { get; set; } // e.g., Camry
        public int Year { get; set; }
        public string RegistrationNumber { get; set; }
        public decimal DailyRate { get; set; }
        public bool IsAvailable { get; set; } = true;

        // Foreign Key to the CarOwner table
        [ForeignKey("Owner")]
        public int OwnerId { get; set; }
        public virtual CarOwner Owner { get; set; }
    }
}
```

**3. Create `Booking.cs`**

This entity tracks rental bookings.

```csharp
// CraRental.Web/Entities/Booking.cs
using System.ComponentModel.DataAnnotations.Schema;

namespace CraRental.Web.Entities
{
    public class Booking
    {
        public int Id { get; set; }
        public DateTime StartDate { get; set; }
        public DateTime EndDate { get; set; }
        public decimal TotalCost { get; set; }
        public string Status { get; set; } // e.g., "Confirmed", "Completed", "Cancelled"

        // Foreign Key to the Car table
        [ForeignKey("Car")]
        public int CarId { get; set; }
        public virtual Car Car { get; set; }

        // Foreign Key to the CarRenter table
        [ForeignKey("Renter")]
        public int RenterId { get; set; }
        public virtual CarRenter Renter { get; set; }
    }
}
```

**4. Create `Payment.cs`**

This entity stores payment details for each booking.

```csharp
// CraRental.Web/Entities/Payment.cs
using System.ComponentModel.DataAnnotations.Schema;

namespace CraRental.Web.Entities
{
    public class Payment
    {
        public int Id { get; set; }
        public decimal Amount { get; set; }
        public DateTime PaymentDate { get; set; }
        public string? PaymentMethod { get; set; }
        public string? TransactionId { get; set; }

        // Foreign Key to the Booking table (One-to-One)
        [ForeignKey("Booking")]
        public int BookingId { get; set; }
        public virtual Booking Booking { get; set; }
    }
}
```

### Step 3: Create and Configure ApplicationDbContext

Now, let's create the database context class that will manage all these entities.

1.  **Create `ApplicationDbContext.cs` inside the `Data` folder.**

2.  **Add the following code to the file.** This code defines the `DbSet` for each entity (which corresponds to a database table) and configures the relationships between them using the Fluent API in the `OnModelCreating` method.

```csharp
// CraRental.Web/Data/ApplicationDbContext.cs
using CraRental.Web.Entities;
using Microsoft.EntityFrameworkCore;

namespace CraRental.Web.Data
{
    public class ApplicationDbContext : DbContext
    {
        public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options) : base(options)
        {
        }

        // Define DbSets for each entity
        public DbSet<User> Users { get; set; }
        public DbSet<CarOwner> CarOwners { get; set; }
        public DbSet<CarRenter> CarRenters { get; set; }
        public DbSet<Car> Cars { get; set; }
        public DbSet<Booking> Bookings { get; set; }
        public DbSet<Payment> Payments { get; set; }

        protected override void OnModelCreating(ModelBuilder modelBuilder)
        {
            base.OnModelCreating(modelBuilder);

            // === Configure User Entity ===
            modelBuilder.Entity<User>(entity =>
            {
                // Make the Email field unique to prevent duplicate accounts
                entity.HasIndex(u => u.Email).IsUnique();
            });

            // === Configure Relationships ===

            // One-to-One: User -> CarOwner
            modelBuilder.Entity<User>()
                .HasOne<CarOwner>()
                .WithOne(co => co.User)
                .HasForeignKey<CarOwner>(co => co.UserId);

            // One-to-One: User -> CarRenter
            modelBuilder.Entity<User>()
                .HasOne<CarRenter>()
                .WithOne(cr => cr.User)
                .HasForeignKey<CarRenter>(cr => cr.UserId);

            // One-to-Many: CarOwner -> Car
            modelBuilder.Entity<CarOwner>()
                .HasMany<Car>()
                .WithOne(c => c.Owner)
                .HasForeignKey(c => c.OwnerId)
                .OnDelete(DeleteBehavior.Cascade); // If an owner is deleted, their cars are also deleted

            // One-to-Many: CarRenter -> Booking
            modelBuilder.Entity<CarRenter>()
                .HasMany<Booking>()
                .WithOne(b => b.Renter)
                .HasForeignKey(b => b.RenterId)
                .OnDelete(DeleteBehavior.Restrict); // Prevent deleting a renter if they have bookings

            // One-to-Many: Car -> Booking
            modelBuilder.Entity<Car>()
                .HasMany<Booking>()
                .WithOne(b => b.Car)
                .HasForeignKey(b => b.CarId)
                .OnDelete(DeleteBehavior.Restrict); // Prevent deleting a car if it has bookings

            // One-to-One: Booking -> Payment
            modelBuilder.Entity<Booking>()
                .HasOne<Payment>(b => b.Payment)
                .WithOne(p => p.Booking)
                .HasForeignKey<Payment>(p => p.BookingId);
        }
    }
}
```

### Step 4: Register the DbContext in `Program.cs`

Finally, you need to register `ApplicationDbContext` with the dependency injection container and configure it to use SQLite.

1.  **Add the connection string to `appsettings.json`:**

    ```json
    {
      "Logging": {
        "LogLevel": {
          "Default": "Information",
          "Microsoft.AspNetCore": "Warning"
        }
      },
      "AllowedHosts": "*",
      "ConnectionStrings": {
        "DefaultConnection": "Data Source=CraRental.db"
      }
    }
    ```

2.  **Update `Program.cs` to register the DbContext:**

    Open `Program.cs` and add the following lines just before `builder.Services.AddControllersWithViews();`.

    ```csharp
    using CraRental.Web.Data; // Add this using statement
    using Microsoft.EntityFrameworkCore; // Add this using statement

    var builder = WebApplication.CreateBuilder(args);

    // --- Add this block ---
    // Configure DbContext
    builder.Services.AddDbContext<ApplicationDbContext>(options =>
        options.UseSqlite(builder.Configuration.GetConnectionString("DefaultConnection")));
    // ----------------------

    // 1. Add services to the container.
    builder.Services.AddControllersWithViews(); // For MVC Controllers
    // ... rest of the file
    ```

You have now successfully defined your data models and configured the Entity Framework Core `DbContext`. The next logical step is to create a database from this model using migrations.
