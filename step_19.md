** USER : **
1. Shall we proceed to build "Password Reset" feature as well
---
Of course. Implementing the "Forgot Password" feature is a critical security and usability step for any application. It's a multi-stage process involving token generation, email sending, and secure password updates.

Let's build the complete password reset flow.

---

### A Quick Note on Email Sending

A real password reset flow requires sending an email. Setting up a full email service (like SendGrid or Mailgun) is a separate, detailed topic. For this guide, we will create an **`IEmailService` interface** and a **mock implementation** that simply logs the reset link to the console. This allows us to build and test the entire flow perfectly, and you can swap in a real email service later without changing any of the core logic.

---

### Part 1: Backend - The Secure Reset Mechanism

#### Step 1: Update the `User` Entity

We need to store the secure reset token and its expiry date.

1.  **Open `Entities/User.cs`** and add two new nullable properties:
    ```csharp
    // CraRental.Web/Entities/User.cs
    public class User
    {
        // ... existing properties ...
        public bool IsActive { get; set; } = true;

        // Add these properties for password reset
        public string? PasswordResetToken { get; set; }
        public DateTime? ResetTokenExpires { get; set; }
    }
    ```

2.  **Create and Apply the Database Migration**:
    ```bash
    dotnet ef migrations add AddPasswordResetToUser
    dotnet ef database update
    ```

#### Step 2: Create the DTOs

1.  **Create `RequestPasswordResetDto.cs`** in the `DTOs` folder:
    ```csharp
    public class RequestPasswordResetDto
    {
        [Required, EmailAddress]
        public string Email { get; set; }
    }
    ```

2.  **Create `ResetPasswordDto.cs`** in the `DTOs` folder:
    ```csharp
    public class ResetPasswordDto
    {
        [Required]
        public string Token { get; set; }

        [Required, MinLength(6)]
        public string NewPassword { get; set; }

        [Required, Compare("NewPassword")]
        public string ConfirmPassword { get; set; }
    }
    ```

#### Step 3: Create the Mock Email Service

1.  **Create `Services/Interfaces/IEmailService.cs`**:
    ```csharp
    public interface IEmailService
    {
        Task SendPasswordResetEmailAsync(string toEmail, string resetLink);
    }
    ```

2.  **Create `Services/MockEmailService.cs`**:
    ```csharp
    // This mock service simulates sending an email by logging to the console.
    // In a real application, you would replace this with a service like SendGrid.
    public class MockEmailService : IEmailService
    {
        private readonly ILogger<MockEmailService> _logger;
        public MockEmailService(ILogger<MockEmailService> logger) { _logger = logger; }

        public Task SendPasswordResetEmailAsync(string toEmail, string resetLink)
        {
            _logger.LogWarning("--- MOCK EMAIL SERVICE ---");
            _logger.LogInformation($"To: {toEmail}");
            _logger.LogInformation("Subject: Reset Your Password");
            _logger.LogInformation($"Body: Please reset your password by clicking here: {resetLink}");
            _logger.LogWarning("--------------------------");
            return Task.CompletedTask;
        }
    }
    ```

#### Step 4: Enhance the `AuthService`

This is where the core logic will live.

1.  **Update `IAuthService.cs`** with the new method signatures:
    ```csharp
    public interface IAuthService
    {
        // ... existing methods ...
        Task<bool> RequestPasswordResetAsync(string email);
        Task<bool> ResetPasswordAsync(ResetPasswordDto dto);
    }
    ```

2.  **Implement the new methods in `AuthService.cs`**:
    ```csharp
    // In Services/AuthService.cs
    // First, inject IEmailService and IConfiguration in the constructor
    private readonly IEmailService _emailService;
    private readonly IConfiguration _configuration;

    public AuthService(/*...,*/ IEmailService emailService, IConfiguration configuration)
    {
        // ...
        _emailService = emailService;
        _configuration = configuration;
    }

    public async Task<bool> RequestPasswordResetAsync(string email)
    {
        var user = await _userRepository.GetByEmailAsync(email);
        if (user == null || !user.IsActive)
        {
            // Do not reveal if the user exists or not for security reasons.
            // Silently succeed, but do nothing.
            return true;
        }

        // Generate a secure, random token
        user.PasswordResetToken = Convert.ToHexString(RandomNumberGenerator.GetBytes(64));
        user.ResetTokenExpires = DateTime.UtcNow.AddMinutes(15); // Token is valid for 15 minutes

        await _context.SaveChangesAsync();

        var resetLink = $"{_configuration["AppBaseUrl"]}/ResetPassword?token={user.PasswordResetToken}";
        await _emailService.SendPasswordResetEmailAsync(user.Email, resetLink);

        return true;
    }

    public async Task<bool> ResetPasswordAsync(ResetPasswordDto dto)
    {
        var user = (await _userRepository.FindAsync(u => u.PasswordResetToken == dto.Token)).FirstOrDefault();

        if (user == null || user.ResetTokenExpires < DateTime.UtcNow)
        {
            throw new Exception("Invalid or expired password reset token.");
        }

        user.PasswordHash = BCrypt.Net.BCrypt.HashPassword(dto.NewPassword);
        
        // CRITICAL: Nullify the token after use to prevent reuse
        user.PasswordResetToken = null;
        user.ResetTokenExpires = null;

        await _context.SaveChangesAsync();
        return true;
    }
    ```
    *Note: Add `"AppBaseUrl": "https://localhost:7001"` to your `appsettings.json`.*

#### Step 5: Enhance the `AuthController`

1.  **Add the new endpoints to `AuthController.cs`**:
    ```csharp
    [HttpPost("request-password-reset")]
    [AllowAnonymous] // Anyone can request a reset
    public async Task<IActionResult> RequestPasswordReset([FromBody] RequestPasswordResetDto dto)
    {
        await _authService.RequestPasswordResetAsync(dto.Email);
        // Always return a success message to prevent email enumeration attacks
        return Ok(new { message = "If an account with this email exists, a password reset link has been sent." });
    }

    [HttpPost("reset-password")]
    [AllowAnonymous]
    public async Task<IActionResult> ResetPassword([FromBody] ResetPasswordDto dto)
    {
        try
        {
            var success = await _authService.ResetPasswordAsync(dto);
            return Ok(new { message = "Password has been reset successfully." });
        }
        catch (Exception ex)
        {
            return BadRequest(new { message = ex.Message });
        }
    }
    ```

#### Step 6: Register Dependencies

1.  **Update `Program.cs`**:
    ```csharp
    // Register the mock email service
    builder.Services.AddScoped<IEmailService, MockEmailService>();
    ```

---

### Part 2: Frontend Implementation

#### Step 1: Update the Login Page

1.  **Open `Pages/Login.cshtml`** and add the "Forgot Password?" link.
    ```html
    <div class="flex items-baseline justify-between">
        <button type="submit" class="...">Login</button>
        <a href="/RequestPasswordReset" class="text-sm text-blue-600 hover:underline">Forgot Password?</a>
    </div>
    ```

#### Step 2: Create the "Request Reset" Page

1.  **Create `Pages/RequestPasswordReset.cshtml`**:
    ```html
    @page
    <h3 class="text-2xl font-bold text-center">Forgot Your Password?</h3>
    <p class="text-center text-gray-600 mt-2">Enter your email and we'll send you a link to reset it.</p>
    <form id="requestResetForm" class="mt-4">
        <!-- ... form with an email input and a submit button ... -->
    </form>
    ```
2.  **Create `wwwroot/js/auth/request-reset.js`**:
    ```javascript
    // ... add event listener to the form ...
    const response = await fetch('/api/auth/request-password-reset', { /*...*/ });
    const result = await response.json();
    // Use the toast notification to show the success message from the API
    showToast(result.message, 'info');
    ```

#### Step 3: Create the "Reset Password" Page

1.  **Create `Pages/ResetPassword.cshtml`**: This page needs to read the token from the URL.
    ```csharp
    // In ResetPassword.cshtml.cs
    public class ResetPasswordModel : PageModel
    {
        [BindProperty(SupportsGet = true)]
        public string Token { get; set; }
        public void OnGet() { }
    }
    ```
    ```html
    <!-- In ResetPassword.cshtml -->
    @page
    @model ResetPasswordModel
    <h3 class="text-2xl font-bold text-center">Create a New Password</h3>
    <form id="resetPasswordForm" class="mt-4">
        <input type="hidden" id="token" value="@Model.Token" />
        <!-- ... New Password and Confirm New Password fields ... -->
    </form>
    ```

2.  **Create `wwwroot/js/auth/reset-password.js`**:
    ```javascript
    // ... add event listener to the form ...
    const resetData = {
        token: document.getElementById('token').value,
        newPassword: document.getElementById('newPassword').value,
        confirmPassword: document.getElementById('confirmPassword').value
    };
    const response = await fetch('/api/auth/reset-password', { /*...*/ });
    if (response.ok) {
        showToast('Password reset successfully! Redirecting to login...', 'success');
        setTimeout(() => window.location.href = '/login', 2000);
    } else {
        // ... show error toast ...
    }
    ```

You have now built a complete, secure, and user-friendly password reset flow. Users who forget their password now have a clear path to regain access to their accounts, which is an essential feature for any real-world application.

