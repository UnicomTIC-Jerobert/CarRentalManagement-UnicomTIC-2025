** USER : **
1. Great Progress , shall we implement the full flow of payements as well

---

Of course. Implementing a payment flow is a critical step that requires careful handling of security and state management. We will integrate **Stripe**, a world-class payment processor, because you should **never handle raw credit card information on your own server**.

We will use Stripe Checkout, which provides a secure, pre-built, and hosted payment page.

**The User Flow:**
1.  A user sees their "Confirmed" booking on the "My Bookings" page.
2.  They click a "Pay Now" button.
3.  Our backend creates a Stripe Checkout Session and redirects the user to Stripe's secure payment page.
4.  The user enters their payment details on Stripe's page.
5.  After payment, Stripe redirects the user back to our application (to a "Success" or "Cancel" page).
6.  In the background, our "Success" endpoint verifies the payment with Stripe and creates the `Payment` record in our database.

---

### Part 1: Backend Setup for Payments with Stripe

#### Step 1: Install the Stripe .NET SDK

Open your terminal in the project root and run:
```bash
dotnet add package Stripe.net
```

#### Step 2: Configure Stripe API Keys

1.  **Sign up for a Stripe account** at [stripe.com](https://stripe.com) and find your API keys in the Developer dashboard. You will need the **Publishable Key** and the **Secret Key**.
2.  **Add your keys to `appsettings.json`**. For production, use a more secure method like User Secrets or Azure Key Vault.

    ```json
    {
      // ... other settings ...
      "StripeSettings": {
        "PublishableKey": "pk_test_...",
        "SecretKey": "sk_test_..."
      }
    }
    ```

#### Step 3: Create Payment Repository

We already defined the `Payment` entity. Now let's create its repository.

1.  **Create `IPaymentRepository.cs`** in `Repositories/Interfaces`:
    ```csharp
    // CraRental.Web/Repositories/Interfaces/IPaymentRepository.cs
    using CraRental.Web.Entities;

    namespace CraRental.Web.Repositories.Interfaces
    {
        public interface IPaymentRepository : IGenericRepository<Payment>
        {
            // Custom methods if needed later
        }
    }
    ```

2.  **Create `PaymentRepository.cs`** in `Repositories`:
    ```csharp
    // CraRental.Web/Repositories/PaymentRepository.cs
    using CraRental.Web.Data;
    using CraRental.Web.Entities;
    using CraRental.Web.Repositories.Interfaces;

    namespace CraRental.Web.Repositories
    {
        public class PaymentRepository : GenericRepository<Payment>, IPaymentRepository
        {
            public PaymentRepository(ApplicationDbContext context) : base(context)
            {
            }
        }
    }
    ```

#### Step 4: Create DTOs and Enhance Existing Ones

1.  **Create `PaymentRequestDto.cs`** in `DTOs`:
    ```csharp
    // CraRental.Web/DTOs/PaymentRequestDto.cs
    namespace CraRental.Web.DTOs
    {
        public class PaymentRequestDto
        {
            public int BookingId { get; set; }
        }
    }
    ```

2.  **Update `BookingViewDto.cs`** to include payment status:
    ```csharp
    // CraRental.Web/DTOs/BookingViewDto.cs
    public class BookingViewDto
    {
        public int Id { get; set; }
        public string CarName { get; set; }
        public DateTime StartDate { get; set; }
        public DateTime EndDate { get; set; }
        public decimal TotalCost { get; set; }
        public string Status { get; set; }
        public bool HasPayment { get; set; } // Add this property
    }
    ```

#### Step 5: Create the Payment Service

This service will interact directly with the Stripe API.

1.  **Create `IPaymentService.cs`** in `Services/Interfaces`:
    ```csharp
    // CraRental.Web/Services/Interfaces/IPaymentService.cs
    using Stripe.Checkout;

    namespace CraRental.Web.Services.Interfaces
    {
        public interface IPaymentService
        {
            Task<Session> CreateCheckoutSessionAsync(int bookingId);
            Task FulfillOrderAsync(Session session);
        }
    }
    ```

2.  **Create `PaymentService.cs`** in `Services`:
    ```csharp
    // CraRental.Web/Services/PaymentService.cs
    using CraRental.Web.Data;
    using CraRental.Web.Entities;
    using CraRental.Web.Repositories.Interfaces;
    using CraRental.Web.Services.Interfaces;
    using Microsoft.Extensions.Configuration;
    using Stripe;
    using Stripe.Checkout;

    namespace CraRental.Web.Services
    {
        public class PaymentService : IPaymentService
        {
            private readonly IBookingRepository _bookingRepository;
            private readonly IPaymentRepository _paymentRepository;
            private readonly ApplicationDbContext _context;

            public PaymentService(IConfiguration config, IBookingRepository bookingRepository, IPaymentRepository paymentRepository, ApplicationDbContext context)
            {
                StripeConfiguration.ApiKey = config["StripeSettings:SecretKey"];
                _bookingRepository = bookingRepository;
                _paymentRepository = paymentRepository;
                _context = context;
            }

            public async Task<Session> CreateCheckoutSessionAsync(int bookingId)
            {
                var booking = await _bookingRepository.GetByIdAsync(bookingId);
                var car = await _context.Cars.FindAsync(booking.CarId); // Need car name
                if (booking == null) throw new Exception("Booking not found.");

                var options = new SessionCreateOptions
                {
                    PaymentMethodTypes = new List<string> { "card" },
                    LineItems = new List<SessionLineItemOptions>
                    {
                        new SessionLineItemOptions
                        {
                            PriceData = new SessionLineItemPriceDataOptions
                            {
                                UnitAmount = (long)(booking.TotalCost * 100), // Amount in cents
                                Currency = "usd",
                                ProductData = new SessionLineItemPriceDataProductDataOptions
                                {
                                    Name = $"Rental for {car.Make} {car.Model}",
                                    Description = $"From {booking.StartDate:d} to {booking.EndDate:d}"
                                },
                            },
                            Quantity = 1,
                        },
                    },
                    Mode = "payment",
                    // IMPORTANT: We pass the booking ID here to identify the order later
                    ClientReferenceId = booking.Id.ToString(),
                    SuccessUrl = "https://localhost:7001/Payment/Success?sessionId={CHECKOUT_SESSION_ID}",
                    CancelUrl = "https://localhost:7001/Payment/Cancel",
                };

                var service = new SessionService();
                return await service.CreateAsync(options);
            }

            public async Task FulfillOrderAsync(Session session)
            {
                var bookingId = int.Parse(session.ClientReferenceId);
                
                // Prevent creating duplicate payments
                var existingPayment = (await _paymentRepository.FindAsync(p => p.BookingId == bookingId)).FirstOrDefault();
                if (existingPayment != null) return;

                var payment = new Entities.Payment
                {
                    BookingId = bookingId,
                    Amount = session.AmountTotal.Value / 100m, // Convert from cents
                    PaymentDate = DateTime.UtcNow,
                    PaymentMethod = "Stripe",
                    TransactionId = session.Id // The Stripe session ID
                };

                await _paymentRepository.AddAsync(payment);
                await _context.SaveChangesAsync();
            }
        }
    }
    ```

#### Step 6: Create `PaymentController`

```csharp
// CraRental.Web/Controllers/PaymentController.cs
using CraRental.Web.DTOs;
using CraRental.Web.Services.Interfaces;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using Stripe.Checkout;

namespace CraRental.Web.Controllers
{
    [Route("api/[controller]")]
    [ApiController]
    [Authorize]
    public class PaymentsController : ControllerBase
    {
        private readonly IPaymentService _paymentService;

        public PaymentsController(IPaymentService paymentService)
        {
            _paymentService = paymentService;
        }

        [HttpPost("create-checkout-session")]
        public async Task<IActionResult> CreateCheckoutSession([FromBody] PaymentRequestDto request)
        {
            var session = await _paymentService.CreateCheckoutSessionAsync(request.BookingId);
            return Ok(new { url = session.Url });
        }
    }
}
```

#### Step 7: Update `BookingService` to Include Payment Status

Modify `GetMyBookingsAsync` in `BookingService.cs` to populate the `HasPayment` flag.

```csharp
// In CraRental.Web/Services/BookingService.cs
public async Task<IEnumerable<BookingViewDto>> GetMyBookingsAsync(int renterUserId)
{
    // ... logic to get renterProfile ...
    var bookings = await _bookingRepository.GetBookingsByRenterIdAsync(renterProfile.Id);

    // Get all payment records for these bookings in one query
    var bookingIds = bookings.Select(b => b.Id).ToList();
    var payments = await _context.Payments.Where(p => bookingIds.Contains(p.BookingId)).ToDictionaryAsync(p => p.BookingId);

    return bookings.Select(b => new BookingViewDto
    {
        // ... other properties ...
        Status = b.Status,
        HasPayment = payments.ContainsKey(b.Id) // Check if a payment exists for this booking
    });
}
```

---

### Part 2: Frontend for Handling Payments

#### Step 1: Create Payment Status Razor Pages

1.  **Create `Pages/Payment/Success.cshtml`**: This is the page Stripe redirects to on success.

    ```csharp
    // In the .cs file (Success.cshtml.cs)
    using CraRental.Web.Services.Interfaces;
    using Microsoft.AspNetCore.Mvc.RazorPages;
    using Stripe.Checkout;

    public class SuccessModel : PageModel
    {
        private readonly IPaymentService _paymentService;
        public SuccessModel(IPaymentService paymentService) { _paymentService = paymentService; }

        public async Task OnGetAsync(string sessionId)
        {
            var sessionService = new SessionService();
            var session = await sessionService.GetAsync(sessionId);
            await _paymentService.FulfillOrderAsync(session);
        }
    }
    ```

    ```html
    <!-- In the .cshtml file (Success.cshtml) -->
    @page
    @model CraRental.Web.Pages.Payment.SuccessModel
    @{ ViewData["Title"] = "Payment Successful"; }

    <div class="container mx-auto text-center mt-20">
        <h1 class="text-3xl font-bold text-green-600">Payment Successful!</h1>
        <p class="mt-4">Thank you for your payment. Your booking is confirmed and paid.</p>
        <a href="/Bookings/MyBookings" class="mt-6 inline-block bg-blue-600 text-white py-2 px-4 rounded">
            View My Bookings
        </a>
    </div>
    ```

2.  **Create `Pages/Payment/Cancel.cshtml`**:
    ```html
    @page
    @{ ViewData["Title"] = "Payment Cancelled"; }
    <div class="container mx-auto text-center mt-20">
        <h1 class="text-3xl font-bold text-red-600">Payment Cancelled</h1>
        <p class="mt-4">Your payment was not completed. You can try again from your bookings page.</p>
        <a href="/Bookings/MyBookings" class="mt-6 inline-block bg-blue-600 text-white py-2 px-4 rounded">
            Return to My Bookings
        </a>
    </div>
    ```

#### Step 2: Update "My Bookings" Page to Include a "Pay" Button

1.  **Modify `Pages/Bookings/MyBookings.cshtml`:** Add a new column for actions.

    ```html
    <!-- ... inside the table ... -->
    <thead class="bg-gray-100">
        <tr>
            <!-- ... other headers ... -->
            <th class="py-2 px-4 text-left">Status</th>
            <th class="py-2 px-4 text-left">Action</th>
        </tr>
    </thead>
    <tbody id="bookings-table-body"></tbody>
    ```

2.  **Modify `wwwroot/js/bookings/my-bookings.js`:**

    ```javascript
    // ... inside the fetch success block ...
    bookings.forEach(booking => {
        let actionButton = '';
        if (booking.status === 'Confirmed' && !booking.hasPayment) {
            actionButton = `<button class="pay-btn bg-green-500 text-white py-1 px-3 rounded hover:bg-green-600" data-booking-id="${booking.id}">Pay Now</button>`;
        } else if (booking.hasPayment) {
            actionButton = `<span class="text-gray-500">Paid</span>`;
        }

        const row = `
            <tr class="border-b">
                <!-- ... other cells ... -->
                <td class="py-3 px-4"><span class="bg-blue-200 text-blue-800 py-1 px-3 rounded-full text-xs">${booking.status}</span></td>
                <td class="py-3 px-4">${actionButton}</td>
            </tr>
        `;
        tableBody.innerHTML += row;
    });

    // Add event listener for the new buttons
    document.querySelectorAll('.pay-btn').forEach(button => {
        button.addEventListener('click', async function () {
            const bookingId = this.getAttribute('data-booking-id');
            const response = await fetch('/api/payments/create-checkout-session', {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json',
                    'Authorization': `Bearer ${token}`
                },
                body: JSON.stringify({ bookingId: parseInt(bookingId) })
            });

            if (response.ok) {
                const result = await response.json();
                window.location.href = result.url; // Redirect to Stripe
            } else {
                alert('Could not initiate payment. Please try again.');
            }
        });
    });
    ```

#### Step 3: Register All New Dependencies in `Program.cs`

```csharp
// ... inside Program.cs

// Register repositories
// ...
builder.Services.AddScoped<IPaymentRepository, PaymentRepository>(); // Add this

// Register services
// ...
builder.Services.AddScoped<IPaymentService, PaymentService>(); // Add this
```

You have now implemented a complete, secure payment flow. The system can create bookings, accept payments via a trusted third party, and reflect the paid status back to the user, all while keeping your application secure and out of PCI compliance scope.

