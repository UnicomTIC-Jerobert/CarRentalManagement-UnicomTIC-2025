** USER : ***
1. ok then whats the next step we are going
2. lets proceed this User Profile Mangement as we agreed

---

Excellent. Now that the Car Owner Dashboard is complete, the most logical and impactful next step is to build the **User Profile Management** flow.

This is a fundamental feature that has been missing for all user roles. Currently, a user can register, but they have no way to view or update their own information. Implementing this will significantly enhance the user experience and the completeness of the application.

Here’s the plan for our next phase:

### Next Step: Implement User Profile Management

We will create a "My Profile" page where authenticated users can view their details and update them. This flow will touch every layer of our application.

1.  **Backend Development:**
    *   **Create DTOs:** We'll need a `ProfileViewDto` to display user data and a `ProfileUpdateDto` to receive changes.
    *   **Create a `UserService`:** A new service dedicated to user-related operations that are not strictly authentication. This service will handle fetching and updating profile information for both Car Owners and Car Renters.
    *   **Create a `UsersController`:** A new API controller with secure endpoints (`GET /api/users/my-profile`, `PUT /api/users/my-profile`) to manage profile data. The controller will identify the user via their JWT token.

2.  **Frontend Development:**
    *   **Create a "My Profile" Razor Page:** A new page (`Pages/Users/Profile.cshtml`) containing a form that will be pre-populated with the user's current information.
    *   **Develop JavaScript for the Profile Page:** We will write the AJAX calls to:
        *   Fetch the user's profile data when the page loads.
        *   Submit the updated profile information to the backend.
        *   Provide feedback to the user (e.g., "Profile updated successfully!").

3.  **UI and Navigation Polish:**
    *   **Implement a Logout Feature:** A proper profile section isn't complete without a secure way to log out. We'll add a "Logout" button that clears the JWT token from local storage and redirects the user.
    *   **Add a "My Profile" Link:** We will add a link to the new profile page in the main navigation bar, visible only to authenticated users.

This approach will create a complete and self-contained feature, adding immediate value for every registered user on the platform and paving the way for more advanced user-specific features later on. Shall we begin with the backend for the User Profile Management?

---

Of course. Let's proceed with building the User Profile Management flow as planned. This is a vital feature for a personalized user experience.

We'll start with the backend, creating the necessary DTOs, the new `UserService`, and the `UsersController`.

---

### Part 1: Backend for User Profile Management

#### Step 1: Create the DTOs

We need two DTOs: one to send profile data to the client, and one to receive updates.

1.  **Create `ProfileViewDto.cs`** in the `DTOs` folder. This will represent a user's complete profile.

    ```csharp
    // CraRental.Web/DTOs/ProfileViewDto.cs
    namespace CraRental.Web.DTOs
    {
        public class ProfileViewDto
        {
            public string Email { get; set; }
            public string Role { get; set; }
            public string FirstName { get; set; }
            public string LastName { get; set; }
            public string? PhoneNumber { get; set; }

            // Role-specific fields
            public string? Address { get; set; } // For CarOwner
            public string? DriverLicenseNumber { get; set; } // For CarRenter
        }
    }
    ```

2.  **Create `ProfileUpdateDto.cs`** in the `DTOs` folder. This contains only the fields a user is allowed to change.

    ```csharp
    // CraRental.Web/DTOs/ProfileUpdateDto.cs
    using System.ComponentModel.DataAnnotations;

    namespace CraRental.Web.DTOs
    {
        public class ProfileUpdateDto
        {
            [Required] public string FirstName { get; set; }
            [Required] public string LastName { get; set; }
            public string? PhoneNumber { get; set; }
            public string? Address { get; set; }
            public string? DriverLicenseNumber { get; set; }
        }
    }
    ```

#### Step 2: Create the User Service

This service will handle the business logic of fetching and updating profiles.

1.  **Create `IUserService.cs`** in `Services/Interfaces`:

    ```csharp
    // CraRental.Web/Services/Interfaces/IUserService.cs
    using CraRental.Web.DTOs;

    namespace CraRental.Web.Services.Interfaces
    {
        public interface IUserService
        {
            Task<ProfileViewDto> GetProfileAsync(int userId);
            Task<bool> UpdateProfileAsync(int userId, ProfileUpdateDto profileDto);
        }
    }
    ```

2.  **Create `UserService.cs`** in `Services`:

    ```csharp
    // CraRental.Web/Services/UserService.cs
    using CraRental.Web.Data;
    using CraRental.Web.DTOs;
    using CraRental.Web.Repositories.Interfaces;
    using CraRental.Web.Services.Interfaces;

    namespace CraRental.Web.Services
    {
        public class UserService : IUserService
        {
            private readonly IUserRepository _userRepository;
            private readonly ICarOwnerRepository _carOwnerRepository;
            private readonly ICarRenterRepository _carRenterRepository;
            private readonly ApplicationDbContext _context;

            public UserService(IUserRepository userRepository, ICarOwnerRepository carOwnerRepository, ICarRenterRepository carRenterRepository, ApplicationDbContext context)
            {
                _userRepository = userRepository;
                _carOwnerRepository = carOwnerRepository;
                _carRenterRepository = carRenterRepository;
                _context = context;
            }

            public async Task<ProfileViewDto> GetProfileAsync(int userId)
            {
                var user = await _userRepository.GetByIdAsync(userId);
                if (user == null) throw new Exception("User not found.");

                var profileDto = new ProfileViewDto { Email = user.Email, Role = user.Role };

                switch (user.Role)
                {
                    case "CarOwner":
                        var owner = (await _carOwnerRepository.FindAsync(o => o.UserId == userId)).FirstOrDefault();
                        if (owner != null)
                        {
                            profileDto.FirstName = owner.FirstName;
                            profileDto.LastName = owner.LastName;
                            profileDto.PhoneNumber = owner.PhoneNumber;
                            profileDto.Address = owner.Address;
                        }
                        break;
                    case "CarRenter":
                        var renter = (await _carRenterRepository.FindAsync(r => r.UserId == userId)).FirstOrDefault();
                        if (renter != null)
                        {
                            profileDto.FirstName = renter.FirstName;
                            profileDto.LastName = renter.LastName;
                            profileDto.PhoneNumber = renter.PhoneNumber;
                            profileDto.DriverLicenseNumber = renter.DriverLicenseNumber;
                        }
                        break;
                }
                return profileDto;
            }

            public async Task<bool> UpdateProfileAsync(int userId, ProfileUpdateDto profileDto)
            {
                var user = await _userRepository.GetByIdAsync(userId);
                if (user == null) return false;

                switch (user.Role)
                {
                    case "CarOwner":
                        var owner = (await _carOwnerRepository.FindAsync(o => o.UserId == userId)).FirstOrDefault();
                        if (owner == null) return false;
                        owner.FirstName = profileDto.FirstName;
                        owner.LastName = profileDto.LastName;
                        owner.PhoneNumber = profileDto.PhoneNumber;
                        owner.Address = profileDto.Address;
                        await _carOwnerRepository.UpdateAsync(owner);
                        break;
                    case "CarRenter":
                         var renter = (await _carRenterRepository.FindAsync(r => r.UserId == userId)).FirstOrDefault();
                        if (renter == null) return false;
                        renter.FirstName = profileDto.FirstName;
                        renter.LastName = profileDto.LastName;
                        renter.PhoneNumber = profileDto.PhoneNumber;
                        renter.DriverLicenseNumber = profileDto.DriverLicenseNumber;
                        await _carRenterRepository.UpdateAsync(renter);
                        break;
                    default:
                        return false;
                }

                await _context.SaveChangesAsync();
                return true;
            }
        }
    }
    ```

#### Step 3: Create the Users API Controller

1.  **Create `UsersController.cs`** in the `Controllers` folder:

    ```csharp
    // CraRental.Web/Controllers/UsersController.cs
    using CraRental.Web.DTOs;
    using CraRental.Web.Services.Interfaces;
    using Microsoft.AspNetCore.Authorization;
    using Microsoft.AspNetCore.Mvc;
    using System.Security.Claims;

    namespace CraRental.Web.Controllers
    {
        [Route("api/[controller]")]
        [ApiController]
        [Authorize] // All methods require authentication
        public class UsersController : ControllerBase
        {
            private readonly IUserService _userService;

            public UsersController(IUserService userService)
            {
                _userService = userService;
            }

            [HttpGet("my-profile")]
            public async Task<IActionResult> GetMyProfile()
            {
                var userId = int.Parse(User.FindFirstValue(ClaimTypes.NameIdentifier));
                var profile = await _userService.GetProfileAsync(userId);
                return Ok(profile);
            }

            [HttpPut("my-profile")]
            public async Task<IActionResult> UpdateMyProfile([FromBody] ProfileUpdateDto profileDto)
            {
                if (!ModelState.IsValid) return BadRequest(ModelState);

                var userId = int.Parse(User.FindFirstValue(ClaimTypes.NameIdentifier));
                var success = await _userService.UpdateProfileAsync(userId, profileDto);

                if (!success) return NotFound(new { message = "Profile could not be updated." });

                return Ok(new { message = "Profile updated successfully." });
            }
        }
    }
    ```

#### Step 4: Register New Dependencies in `Program.cs`

```csharp
// ... inside Program.cs

// Register services
// ... (previous registrations)
builder.Services.AddScoped<IUserService, UserService>(); // Add this
```

---

### Part 2: Frontend for Profile Management

#### Step 1: Create the "My Profile" Razor Page

1.  **Create `Pages/Users/Profile.cshtml`**:

    ```html
    @page
    @{
        ViewData["Title"] = "My Profile";
    }

    <div class="container mx-auto mt-10 max-w-2xl">
        <h1 class="text-3xl font-bold mb-6">My Profile</h1>
        <form id="profileForm" class="bg-white p-8 rounded-lg shadow-md">
            <div id="formMessage" class="hidden p-3 mb-4 text-sm rounded-lg" role="alert"></div>

            <!-- Common Fields -->
            <div class="grid grid-cols-1 md:grid-cols-2 gap-6">
                <div>
                    <label class="block font-medium">Email</label>
                    <input type="email" id="email" readonly class="mt-1 p-2 w-full bg-gray-100 border rounded-md cursor-not-allowed">
                </div>
                <div>
                    <label class="block font-medium">Role</label>
                    <input type="text" id="role" readonly class="mt-1 p-2 w-full bg-gray-100 border rounded-md cursor-not-allowed">
                </div>
                 <div>
                    <label for="firstName" class="block font-medium">First Name</label>
                    <input type="text" id="firstName" required class="mt-1 p-2 w-full border rounded-md">
                </div>
                 <div>
                    <label for="lastName" class="block font-medium">Last Name</label>
                    <input type="text" id="lastName" required class="mt-1 p-2 w-full border rounded-md">
                </div>
                 <div class="md:col-span-2">
                    <label for="phoneNumber" class="block font-medium">Phone Number</label>
                    <input type="tel" id="phoneNumber" class="mt-1 p-2 w-full border rounded-md">
                </div>
            </div>

            <!-- Role-Specific Fields (initially hidden) -->
            <div id="owner-fields" class="hidden mt-6">
                <label for="address" class="block font-medium">Address</label>
                <textarea id="address" rows="3" class="mt-1 p-2 w-full border rounded-md"></textarea>
            </div>

            <div id="renter-fields" class="hidden mt-6">
                 <label for="driverLicenseNumber" class="block font-medium">Driver's License Number</label>
                <input type="text" id="driverLicenseNumber" class="mt-1 p-2 w-full border rounded-md">
            </div>

            <div class="mt-8">
                <button type="submit" class="w-full bg-blue-600 text-white py-2 px-4 rounded-md hover:bg-blue-700">
                    Save Changes
                </button>
            </div>
        </form>
    </div>

    @section Scripts {
        <script src="~/js/users/profile.js" asp-append-version="true"></script>
    }
    ```

#### Step 2: Create the Profile JavaScript

1.  **Create `wwwroot/js/users/profile.js`**:

    ```javascript
    document.addEventListener('DOMContentLoaded', function () {
        const token = localStorage.getItem('jwt_token');
        if (!token) {
            window.location.href = '/login';
            return;
        }

        const profileForm = document.getElementById('profileForm');
        const formMessage = document.getElementById('formMessage');

        // References to form fields
        const emailEl = document.getElementById('email');
        const roleEl = document.getElementById('role');
        const firstNameEl = document.getElementById('firstName');
        const lastNameEl = document.getElementById('lastName');
        const phoneNumberEl = document.getElementById('phoneNumber');
        const addressEl = document.getElementById('address');
        const driverLicenseEl = document.getElementById('driverLicenseNumber');
        const ownerFieldsDiv = document.getElementById('owner-fields');
        const renterFieldsDiv = document.getElementById('renter-fields');

        // Fetch profile data on page load
        async function loadProfile() {
            const response = await fetch('/api/users/my-profile', {
                headers: { 'Authorization': `Bearer ${token}` }
            });
            if (!response.ok) return;

            const profile = await response.json();
            
            // Populate common fields
            emailEl.value = profile.email;
            roleEl.value = profile.role;
            firstNameEl.value = profile.firstName;
            lastNameEl.value = profile.lastName;
            phoneNumberEl.value = profile.phoneNumber || '';

            // Show and populate role-specific fields
            if (profile.role === 'CarOwner') {
                addressEl.value = profile.address || '';
                ownerFieldsDiv.classList.remove('hidden');
            } else if (profile.role === 'CarRenter') {
                driverLicenseEl.value = profile.driverLicenseNumber || '';
                renterFieldsDiv.classList.remove('hidden');
            }
        }

        loadProfile();

        // Handle form submission for updates
        profileForm.addEventListener('submit', async function(e) {
            e.preventDefault();

            const profileData = {
                firstName: firstNameEl.value,
                lastName: lastNameEl.value,
                phoneNumber: phoneNumberEl.value,
                address: addressEl.value,
                driverLicenseNumber: driverLicenseEl.value
            };

            const response = await fetch('/api/users/my-profile', {
                method: 'PUT',
                headers: { 'Content-Type': 'application/json', 'Authorization': `Bearer ${token}` },
                body: JSON.stringify(profileData)
            });

            const result = await response.json();
            formMessage.classList.remove('hidden');
            if (response.ok) {
                formMessage.className = 'p-3 mb-4 text-sm text-green-700 bg-green-100 rounded-lg';
                formMessage.textContent = result.message;
            } else {
                formMessage.className = 'p-3 mb-4 text-sm text-red-700 bg-red-100 rounded-lg';
                formMessage.textContent = result.message || 'An error occurred.';
            }
        });
    });
    ```

---

### Part 3: Navigation and Logout Functionality

#### Step 1: Implement Logout

1.  **Add an ID to the Logout link** in `Views/Shared/_Layout.cshtml`:
    ```html
    <a href="#" id="logout-link" class="px-3 py-2">Logout</a>
    ```

2.  **Create `wwwroot/js/site.js`** (or add to it if it exists) to handle the global logout logic:
    ```javascript
    // wwwroot/js/site.js
    document.addEventListener('DOMContentLoaded', function() {
        const logoutLink = document.getElementById('logout-link');
        
        if (logoutLink) {
            logoutLink.addEventListener('click', function(e) {
                e.preventDefault();
                // Clear the token from storage
                localStorage.removeItem('jwt_token');
                // Redirect to the login page
                window.location.href = '/login';
            });
        }
    });
    ```

#### Step 2: Update Navigation in `_Layout.cshtml`

Add the "My Profile" link for logged-in users.

```html
<!-- Inside the @if (User.Identity.IsAuthenticated) block -->
<a asp-page="/Users/Profile" class="px-3 py-2">My Profile</a>
<a href="#" id="logout-link" class="px-3 py-2">Logout</a>
```

You have now built a complete and secure profile management system. Users can view their information, update it as needed, and securely log out of the application. This adds a crucial layer of personalization and functionality to the project.


