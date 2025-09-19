"
** USER " ** its great lets now build the Interfaces and Implementations for the Appropriate Entities
"

---
Of course. Now that we have our entities and DbContext set up, let's build the abstraction layer using interfaces and then create their concrete implementations. This is a core part of building a clean, maintainable, and testable application.

We will create:
1.  A **generic repository** to handle common data access operations (Create, Read, Update, Delete) for any entity.
2.  Specific repository interfaces for entities that might need custom query methods.
3.  Service interfaces and their implementations where our business logic will reside.

### Step 1: Create the Generic Repository

A generic repository saves you from writing repetitive CRUD code for each entity.

1.  **Create an `Interfaces` subfolder** inside the `Repositories` folder.
2.  **Create the `IGenericRepository.cs` interface** inside `Repositories/Interfaces`:

    ```csharp
    // CraRental.Web/Repositories/Interfaces/IGenericRepository.cs
    using System.Linq.Expressions;

    namespace CraRental.Web.Repositories.Interfaces
    {
        public interface IGenericRepository<T> where T : class
        {
            Task<T> GetByIdAsync(int id);
            Task<IEnumerable<T>> GetAllAsync();
            Task<IEnumerable<T>> FindAsync(Expression<Func<T, bool>> predicate);
            Task AddAsync(T entity);
            Task UpdateAsync(T entity);
            Task RemoveAsync(T entity);
        }
    }
    ```

3.  **Create the `GenericRepository.cs` implementation** inside the `Repositories` folder:

    ```csharp
    // CraRental.Web/Repositories/GenericRepository.cs
    using CraRental.Web.Data;
    using CraRental.Web.Repositories.Interfaces;
    using Microsoft.EntityFrameworkCore;
    using System.Linq.Expressions;

    namespace CraRental.Web.Repositories
    {
        public class GenericRepository<T> : IGenericRepository<T> where T : class
        {
            protected readonly ApplicationDbContext _context;
            protected readonly DbSet<T> _dbSet;

            public GenericRepository(ApplicationDbContext context)
            {
                _context = context;
                _dbSet = context.Set<T>();
            }

            public async Task AddAsync(T entity)
            {
                await _dbSet.AddAsync(entity);
            }

            public async Task<IEnumerable<T>> FindAsync(Expression<Func<T, bool>> predicate)
            {
                return await _dbSet.Where(predicate).ToListAsync();
            }

            public async Task<IEnumerable<T>> GetAllAsync()
            {
                return await _dbSet.ToListAsync();
            }

            public async Task<T> GetByIdAsync(int id)
            {
                return await _dbSet.FindAsync(id);
            }

            public Task RemoveAsync(T entity)
            {
                _dbSet.Remove(entity);
                return Task.CompletedTask;
            }

            public Task UpdateAsync(T entity)
            {
                _context.Entry(entity).State = EntityState.Modified;
                return Task.CompletedTask;
            }
        }
    }
    ```

### Step 2: Create Specific Interfaces and Implementations

Even with a generic repository, you'll often need specific methods for certain entities (e.g., getting a user by email).

1.  **Create `IUserRepository.cs`** inside `Repositories/Interfaces`:

    This inherits the generic methods and adds a custom one.

    ```csharp
    // CraRental.Web/Repositories/Interfaces/IUserRepository.cs
    using CraRental.Web.Entities;

    namespace CraRental.Web.Repositories.Interfaces
    {
        public interface IUserRepository : IGenericRepository<User>
        {
            Task<User> GetByEmailAsync(string email);
        }
    }
    ```

2.  **Create `UserRepository.cs`** inside `Repositories`:

    ```csharp
    // CraRental.Web/Repositories/UserRepository.cs
    using CraRental.Web.Data;
    using CraRental.Web.Entities;
    using CraRental.Web.Repositories.Interfaces;
    using Microsoft.EntityFrameworkCore;

    namespace CraRental.Web.Repositories
    {
        public class UserRepository : GenericRepository<User>, IUserRepository
        {
            public UserRepository(ApplicationDbContext context) : base(context)
            {
            }

            public async Task<User> GetByEmailAsync(string email)
            {
                return await _dbSet.FirstOrDefaultAsync(u => u.Email == email);
            }
        }
    }
    ```

### Step 3: Create Service Layer Interfaces

The service layer will contain the core business logic of your application.

1.  **Create an `Interfaces` subfolder** inside the `Services` folder.
2.  **Create `IAuthService.cs`** inside `Services/Interfaces`:

    This service will handle registration, login, and token generation.

    ```csharp
    // CraRental.Web/Services/Interfaces/IAuthService.cs
    using CraRental.Web.DTOs; // We will create this DTO soon
    using CraRental.Web.Entities;

    namespace CraRental.Web.Services.Interfaces
    {
        public interface IAuthService
        {
            Task<User> RegisterUserAsync(UserRegistrationDto registrationDto);
            Task<string> LoginAsync(UserLoginDto loginDto);
            // We will add more methods later
        }
    }
    ```
    *(Note: We are referencing DTOs that we haven't created yet. We'll do that in the next step, but it's good to define the contract first.)*

### Step 4: Create DTOs (Data Transfer Objects)

DTOs are used to transfer data between layers, especially between the client and the server, to avoid exposing your entity models directly.

1.  **Create `UserRegistrationDto.cs`** in the `DTOs` folder:

    ```csharp
    // CraRental.Web/DTOs/UserRegistrationDto.cs
    using System.ComponentModel.DataAnnotations;

    namespace CraRental.Web.DTOs
    {
        public class UserRegistrationDto
        {
            [Required]
            [EmailAddress]
            public string Email { get; set; }

            [Required]
            [MinLength(6)]
            public string Password { get; set; }

            [Required]
            public string Role { get; set; } // "CarOwner" or "CarRenter"

            [Required]
            public string FirstName { get; set; }

            [Required]
            public string LastName { get; set; }
        }
    }
    ```

2.  **Create `UserLoginDto.cs`** in the `DTOs` folder:

    ```csharp
    // CraRental.Web/DTOs/UserLoginDto.cs
    using System.ComponentModel.DataAnnotations;

    namespace CraRental.Web.DTOs
    {
        public class UserLoginDto
        {
            [Required]
            [EmailAddress]
            public string Email { get; set; }

            [Required]
            public string Password { get; set; }
        }
    }
    ```

### Step 5: Create Service Layer Implementations

Now, let's implement the `IAuthService`.

1.  **Create `AuthService.cs`** inside the `Services` folder:
    *(For now, we'll leave the login method incomplete as it requires JWT setup, which is a larger topic.)*

    ```csharp
    // CraRental.Web/Services/AuthService.cs
    using CraRental.Web.DTOs;
    using CraRental.Web.Entities;
    using CraRental.Web.Repositories.Interfaces;
    using CraRental.Web.Services.Interfaces;

    namespace CraRental.Web.Services
    {
        public class AuthService : IAuthService
        {
            private readonly IUserRepository _userRepository;
            // We'll add repositories for CarOwner and CarRenter later

            public AuthService(IUserRepository userRepository)
            {
                _userRepository = userRepository;
            }

            public async Task<string> LoginAsync(UserLoginDto loginDto)
            {
                // 1. Find user by email
                var user = await _userRepository.GetByEmailAsync(loginDto.Email);
                if (user == null)
                {
                    throw new Exception("Invalid credentials");
                }

                // 2. Verify password (we'll add hashing later)
                // For now, a placeholder check. THIS IS INSECURE.
                // if (user.PasswordHash != loginDto.Password)
                // {
                //     throw new Exception("Invalid credentials");
                // }

                // 3. Generate JWT Token (to be implemented)
                var token = "dummy-jwt-token-for-now";

                return token;
            }

            public async Task<User> RegisterUserAsync(UserRegistrationDto registrationDto)
            {
                // 1. Check if user already exists
                var existingUser = await _userRepository.GetByEmailAsync(registrationDto.Email);
                if (existingUser != null)
                {
                    throw new Exception("Email is already registered.");
                }

                // 2. Create and save the new User entity
                var newUser = new User
                {
                    Email = registrationDto.Email,
                    // IMPORTANT: Hash the password before saving! We'll add this.
                    PasswordHash = registrationDto.Password, // Placeholder
                    Role = registrationDto.Role,
                };

                await _userRepository.AddAsync(newUser);
                // We need to save changes to the database here. Let's create a Unit of Work for that.

                // 3. Create CarOwner or CarRenter profile (to be implemented)

                return newUser;
            }
        }
    }
    ```    ***Note:** As you can see, the service logic reveals the need for password hashing and a "Unit of Work" pattern to save changes. We'll address these in subsequent steps.*

### Step 6: Register Services for Dependency Injection

Finally, register all your new interfaces and implementations in `Program.cs` so they can be injected where needed.

Open `Program.cs` and add the following before `var app = builder.Build();`:

```csharp
// ... after builder.Services.AddDbContext...

// 1. Add services to the container.
builder.Services.AddControllersWithViews();
builder.Services.AddRazorPages();

// --- Add this block for Dependency Injection ---
// Register repositories
builder.Services.AddScoped(typeof(IGenericRepository<>), typeof(GenericRepository<>));
builder.Services.AddScoped<IUserRepository, UserRepository>();

// Register services
builder.Services.AddScoped<IAuthService, AuthService>();
// ---------------------------------------------

var app = builder.Build();

// ... rest of the file
```

You have now successfully established the repository and service layers with proper interfaces and dependency injection. This structure is robust and ready for you to build out the business logic, JWT authentication, and controllers.
