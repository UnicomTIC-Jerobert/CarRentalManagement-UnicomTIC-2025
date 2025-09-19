** USER : **
1. Great Shall we Implement The "Email Notifications" as well

---

Of course. Implementing email notifications is a crucial step that makes the application feel alive and professional. It provides essential feedback to users and keeps them engaged.

We will integrate **SendGrid**, a popular and robust email delivery service. It has an excellent .NET SDK and a free tier that is perfect for development and small-scale applications.

The plan is to replace our `MockEmailService` with a real implementation and then trigger emails from our existing business logic for key events.

---

### Part 1: Setting Up the Real Email Service (SendGrid)

#### Step 1: Get a SendGrid API Key

1.  Go to [SendGrid.com](https://sendgrid.com/) and sign up for a free account.
2.  Follow their instructions to create a "Single Sender Identity." This involves verifying an email address that you own (e.g., `your-email@gmail.com`). This will be your "From" address.
3.  Navigate to **Settings -> API Keys** in the SendGrid dashboard.
4.  Click "Create API Key," give it a name (e.g., "CraRental Project"), and choose "Full Access."
5.  **CRITICAL:** Copy the generated API key immediately and save it somewhere safe. You will **not** be able to see it again.

#### Step 2: Securely Store Your API Key

We will use the **.NET Secret Manager** to store your API key securely on your local machine, keeping it out of your source code.

1.  Open your terminal in the `CraRental.Web` project root.
2.  Initialize user secrets for the project (if you haven't already):
    ```bash
    dotnet user-secrets init
    ```
3.  Set your SendGrid API Key in the secret store:
    ```bash
    dotnet user-secrets set "SendGridKey" "PASTE_YOUR_API_KEY_HERE"
    ```

#### Step 3: Install the SendGrid SDK

1.  Run the following command in your terminal:
    ```bash
    dotnet add package SendGrid
    ```

#### Step 4: Implement the `SendGridEmailService`

Now, we'll create the real email service that implements our existing `IEmailService` interface.

1.  **Add your "From" email to `appsettings.json`**:
    ```json
    {
      // ... other settings ...
      "EmailSettings": {
        "FromAddress": "your-verified-email@example.com",
        "FromName": "CraRental Team"
      }
    }
    ```

2.  **Create a new file `Services/SendGridEmailService.cs`**:
    ```csharp
    using Microsoft.Extensions.Configuration;
    using SendGrid;
    using SendGrid.Helpers.Mail;
    using System.Threading.Tasks;

    namespace CraRental.Web.Services
    {
        public class SendGridEmailService : IEmailService
        {
            private readonly IConfiguration _configuration;

            public SendGridEmailService(IConfiguration configuration)
            {
                _configuration = configuration;
            }

            // A helper method to send any email
            private async Task SendEmailAsync(string toEmail, string subject, string htmlContent)
            {
                // The API Key is read from User Secrets, not appsettings.json
                var apiKey = _configuration["SendGridKey"];
                var client = new SendGridClient(apiKey);
                var from = new EmailAddress(
                    _configuration["EmailSettings:FromAddress"],
                    _configuration["EmailSettings:FromName"]
                );
                var to = new EmailAddress(toEmail);
                var msg = MailHelper.CreateSingleEmail(from, to, subject, "", htmlContent);
                var response = await client.SendEmailAsync(msg);

                // You can add logging here to check if the email was sent successfully
                // _logger.LogInformation(response.IsSuccessStatusCode ? "Email sent" : "Email failed to send");
            }

            // Implement the existing interface method
            public async Task SendPasswordResetEmailAsync(string toEmail, string resetLink)
            {
                var subject = "Reset Your CraRental Password";
                var htmlContent = $"<h1>Password Reset Request</h1><p>Please reset your password by clicking the following link:</p><p><a href='{resetLink}'>Reset Password</a></p><p>This link will expire in 15 minutes.</p>";
                await SendEmailAsync(toEmail, subject, htmlContent);
            }

            // Implement a new method for welcome emails
            public async Task SendWelcomeEmailAsync(string toEmail, string name)
            {
                var subject = "Welcome to CraRental!";
                var htmlContent = $"<h1>Welcome, {name}!</h1><p>Thank you for joining CraRental. We're excited to have you on board.</p><p>You can now browse cars and start your next adventure!</p>";
                await SendEmailAsync(toEmail, subject, htmlContent);
            }

            // Implement a new method for booking confirmations
            public async Task SendBookingConfirmationEmailAsync(string toEmail, string carName, DateTime startDate, DateTime endDate, decimal totalCost)
            {
                var subject = "Your CraRental Booking is Confirmed!";
                var htmlContent = $@"
                    <h1>Booking Confirmed!</h1>
                    <p>Your booking for the <strong>{carName}</strong> has been confirmed.</p>
                    <ul>
                        <li><strong>Start Date:</strong> {startDate:MMMM dd, yyyy}</li>
                        <li><strong>End Date:</strong> {endDate:MMMM dd, yyyy}</li>
                        <li><strong>Total Cost:</strong> ${totalCost:0.00}</li>
                    </ul>
                    <p>Thank you for choosing CraRental!</p>";
                await SendEmailAsync(toEmail, subject, htmlContent);
            }
        }
    }
    ```

3.  **Update `IEmailService.cs`** with the new method signatures:
    ```csharp
    public interface IEmailService
    {
        Task SendPasswordResetEmailAsync(string toEmail, string resetLink);
        Task SendWelcomeEmailAsync(string toEmail, string name);
        Task SendBookingConfirmationEmailAsync(string toEmail, string carName, DateTime startDate, DateTime endDate, decimal totalCost);
    }
    ```

#### Step 5: Register the Real Service in `Program.cs`

Now we swap the mock service for the real one.

1.  **Open `Program.cs`** and find the service registration.
    ```csharp
    // Comment out or delete the mock service registration
    // builder.Services.AddScoped<IEmailService, MockEmailService>();

    // Add the new, real service registration
    builder.Services.AddScoped<IEmailService, SendGridEmailService>();
    ```

---

### Part 2: Integrating Emails into Your Application's Logic

Now that the service is ready, we just need to call it from the right places.

#### Step 1: Send Welcome Email on Registration

1.  **Open `Services/AuthService.cs`**.
2.  Inject `IEmailService` in the constructor.
3.  In the `RegisterUserAsync` method, add the email call **after** the transaction has been successfully committed.
    ```csharp
    // In Services/AuthService.cs -> RegisterUserAsync
    public async Task<User> RegisterUserAsync(UserRegistrationDto registrationDto)
    {
        // ... existing logic to check for user and start transaction ...
        try
        {
            // ... logic to create User and Profile ...
            await _context.SaveChangesAsync();
            await transaction.CommitAsync();

            // --- ADD THIS LINE ---
            // Send welcome email after everything is successfully saved
            await _emailService.SendWelcomeEmailAsync(newUser.Email, registrationDto.FirstName);
            // --------------------

            return newUser;
        }
        catch (Exception)
        {
            // ... rollback logic ...
        }
    }
    ```

#### Step 2: Send Booking Confirmation Email

1.  **Open `Services/BookingService.cs`**.
2.  Inject `IEmailService` in the constructor.
3.  In the `CreateBookingAsync` method, add the email call **after** the booking is saved.
    ```csharp
    // In Services/BookingService.cs -> CreateBookingAsync
    public async Task<BookingViewDto> CreateBookingAsync(BookingCreateDto bookingDto, int renterUserId)
    {
        // ... all validation logic ...
        // ... logic to calculate cost and create newBooking object ...

        await using (var transaction = await _context.Database.BeginTransactionAsync())
        {
            await _bookingRepository.AddAsync(newBooking);
            await _context.SaveChangesAsync();
            await transaction.CommitAsync();
        }

        // --- ADD THIS BLOCK ---
        var renter = (await _carRenterRepository.FindAsync(r => r.UserId == renterUserId)).First();
        await _emailService.SendBookingConfirmationEmailAsync(
            renter.User.Email, // We need the user's email
            car.Make + " " + car.Model,
            newBooking.StartDate,
            newBooking.EndDate,
            newBooking.TotalCost
        );
        // ----------------------

        // ... return DTO ...
    }
    ```
    *Note: You may need to adjust the `CreateBookingAsync` method slightly to ensure you have all the required information (like the renter's email) readily available at the end.*

You have now fully integrated a powerful email notification system into your application. Users will receive immediate, professional feedback for critical actions like registering and booking a car, significantly improving the user experience and trust in your platform.

