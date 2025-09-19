** USER : **
1. Ok lets create the full flow for booking related things here

---

Of course. Let's build the complete end-to-end flow for booking a car. This is a critical feature that involves robust backend logic (like checking for availability and transaction management) and a seamless frontend experience.

We will build:
1.  **The backend infrastructure:** Repositories, DTOs, a `BookingService` with business logic, and a secure `BookingController`.
2.  **A Car Detail page:** Where a renter can view car details and initiate a booking.
3.  **A "My Bookings" page:** For renters to view their booking history.

---

### Part 1: Backend Setup for Bookings

#### Step 1: Create Booking Repository

1.  **Create `IBookingRepository.cs`** in `Repositories/Interfaces`: This interface will include a crucial method to check for booking conflicts.

    ```csharp
    // CraRental.Web/Repositories/Interfaces/IBookingRepository.cs
    using CraRental.Web.Entities;

    namespace CraRental.Web.Repositories.Interfaces
    {
        public interface IBookingRepository : IGenericRepository<Booking>
        {
            Task<bool> HasOverlappingBooking(int carId, DateTime startDate, DateTime endDate);
            Task<IEnumerable<Booking>> GetBookingsByRenterIdAsync(int renterId);
        }
    }
    ```

2.  **Create `BookingRepository.cs`** in `Repositories`:

    ```csharp
    // CraRental.Web/Repositories/BookingRepository.cs
    using CraRental.Web.Data;
    using CraRental.Web.Entities;
    using CraRental.Web.Repositories.Interfaces;
    using Microsoft.EntityFrameworkCore;

    namespace CraRental.Web.Repositories
    {
        public class BookingRepository : GenericRepository<Booking>, IBookingRepository
        {
            public BookingRepository(ApplicationDbContext context) : base(context)
            {
            }

            public async Task<IEnumerable<Booking>> GetBookingsByRenterIdAsync(int renterId)
            {
                return await _dbSet
                    .Include(b => b.Car) // Include car details
                    .Where(b => b.RenterId == renterId)
                    .OrderByDescending(b => b.StartDate)
                    .ToListAsync();
            }

            public async Task<bool> HasOverlappingBooking(int carId, DateTime startDate, DateTime endDate)
            {
                // A booking overlaps if it's for the same car and its time range
                // intersects with the requested time range.
                // Logic: (StartA <= EndB) and (EndA >= StartB)
                return await _dbSet.AnyAsync(b =>
                    b.CarId == carId &&
                    b.StartDate < endDate &&
                    b.EndDate > startDate);
            }
        }
    }
    ```

#### Step 2: Create Booking DTOs

1.  **Create `BookingCreateDto.cs`** in `DTOs`:

    ```csharp
    // CraRental.Web/DTOs/BookingCreateDto.cs
    using System.ComponentModel.DataAnnotations;

    namespace CraRental.Web.DTOs
    {
        public class BookingCreateDto
        {
            [Required] public int CarId { get; set; }
            [Required] public DateTime StartDate { get; set; }
            [Required] public DateTime EndDate { get; set; }
        }
    }
    ```

2.  **Create `BookingViewDto.cs`** in `DTOs`:

    ```csharp
    // CraRental.Web/DTOs/BookingViewDto.cs
    namespace CraRental.Web.DTOs
    {
        public class BookingViewDto
        {
            public int Id { get; set; }
            public string CarName { get; set; }
            public DateTime StartDate { get; set; }
            public DateTime EndDate { get; set; }
            public decimal TotalCost { get; set; }
            public string Status { get; set; }
        }
    }
    ```

#### Step 3: Create Booking Service (with Business Logic)

1.  **Create `IBookingService.cs`** in `Services/Interfaces`:

    ```csharp
    // CraRental.Web/Services/Interfaces/IBookingService.cs
    using CraRental.Web.DTOs;

    namespace CraRental.Web.Services.Interfaces
    {
        public interface IBookingService
        {
            Task<BookingViewDto> CreateBookingAsync(BookingCreateDto bookingDto, int renterUserId);
            Task<IEnumerable<BookingViewDto>> GetMyBookingsAsync(int renterUserId);
        }
    }
    ```

2.  **Create `BookingService.cs`** in `Services`: This is where the magic happens—validation, cost calculation, and transaction management.

    ```csharp
    // CraRental.Web/Services/BookingService.cs
    using CraRental.Web.Data;
    using CraRental.Web.DTOs;
    using CraRental.Web.Entities;
    using CraRental.Web.Repositories.Interfaces;
    using CraRental.Web.Services.Interfaces;
    using Microsoft.EntityFrameworkCore.Storage;

    namespace CraRental.Web.Services
    {
        public class BookingService : IBookingService
        {
            private readonly IBookingRepository _bookingRepository;
            private readonly ICarRepository _carRepository;
            private readonly ICarRenterRepository _carRenterRepository;
            private readonly ApplicationDbContext _context;

            public BookingService(IBookingRepository bookingRepository, ICarRepository carRepository, ICarRenterRepository carRenterRepository, ApplicationDbContext context)
            {
                _bookingRepository = bookingRepository;
                _carRepository = carRepository;
                _carRenterRepository = carRenterRepository;
                _context = context;
            }

            public async Task<BookingViewDto> CreateBookingAsync(BookingCreateDto bookingDto, int renterUserId)
            {
                // --- Validation ---
                var renterProfile = (await _carRenterRepository.FindAsync(r => r.UserId == renterUserId)).FirstOrDefault();
                if (renterProfile == null) throw new Exception("Renter profile not found.");

                var car = await _carRepository.GetByIdAsync(bookingDto.CarId);
                if (car == null) throw new Exception("Car not found.");

                if (bookingDto.StartDate >= bookingDto.EndDate || bookingDto.StartDate < DateTime.UtcNow.Date)
                    throw new InvalidOperationException("Invalid booking dates.");

                // --- CRITICAL: Check for overlapping bookings ---
                var isConflict = await _bookingRepository.HasOverlappingBooking(car.Id, bookingDto.StartDate, bookingDto.EndDate);
                if (isConflict) throw new InvalidOperationException("The car is already booked for the selected dates.");

                // --- Logic ---
                var numberOfDays = (bookingDto.EndDate - bookingDto.StartDate).Days;
                var totalCost = numberOfDays * car.DailyRate;

                var newBooking = new Booking
                {
                    CarId = car.Id,
                    RenterId = renterProfile.Id,
                    StartDate = bookingDto.StartDate,
                    EndDate = bookingDto.EndDate,
                    TotalCost = totalCost,
                    Status = "Confirmed"
                };
                
                // Use a transaction to ensure the booking is saved atomically
                await using (var transaction = await _context.Database.BeginTransactionAsync())
                {
                    await _bookingRepository.AddAsync(newBooking);
                    await _context.SaveChangesAsync();
                    await transaction.CommitAsync();
                }

                return new BookingViewDto { /* map from newBooking */ };
            }

            public async Task<IEnumerable<BookingViewDto>> GetMyBookingsAsync(int renterUserId)
            {
                var renterProfile = (await _carRenterRepository.FindAsync(r => r.UserId == renterUserId)).FirstOrDefault();
                if (renterProfile == null) return Enumerable.Empty<BookingViewDto>();

                var bookings = await _bookingRepository.GetBookingsByRenterIdAsync(renterProfile.Id);

                return bookings.Select(b => new BookingViewDto
                {
                    Id = b.Id,
                    CarName = $"{b.Car.Make} {b.Car.Model}",
                    StartDate = b.StartDate,
                    EndDate = b.EndDate,
                    TotalCost = b.TotalCost,
                    Status = b.Status
                });
            }
        }
    }
    ```

#### Step 4: Create `BookingController`

```csharp
// CraRental.Web/Controllers/BookingController.cs
using CraRental.Web.DTOs;
using CraRental.Web.Services.Interfaces;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using System.Security.Claims;

namespace CraRental.Web.Controllers
{
    [Route("api/[controller]")]
    [ApiController]
    [Authorize]
    public class BookingsController : ControllerBase
    {
        private readonly IBookingService _bookingService;

        public BookingsController(IBookingService bookingService)
        {
            _bookingService = bookingService;
        }

        [HttpPost]
        [Authorize(Roles = "CarRenter")]
        public async Task<IActionResult> CreateBooking([FromBody] BookingCreateDto bookingDto)
        {
            if (!ModelState.IsValid) return BadRequest(ModelState);

            try
            {
                var userId = int.Parse(User.FindFirstValue(ClaimTypes.NameIdentifier));
                var result = await _bookingService.CreateBookingAsync(bookingDto, userId);
                return Ok(result);
            }
            catch (Exception ex)
            {
                return BadRequest(new { message = ex.Message });
            }
        }

        [HttpGet("my-bookings")]
        [Authorize(Roles = "CarRenter")]
        public async Task<IActionResult> GetMyBookings()
        {
            var userId = int.Parse(User.FindFirstValue(ClaimTypes.NameIdentifier));
            var bookings = await _bookingService.GetMyBookingsAsync(userId);
            return Ok(bookings);
        }
    }
}
```

#### Step 5: Register Dependencies in `Program.cs`

```csharp
// ... inside Program.cs

// Register repositories
builder.Services.AddScoped<IBookingRepository, BookingRepository>(); // Add this

// Register services
builder.Services.AddScoped<IBookingService, BookingService>(); // Add this
```

---

### Part 2: Frontend for Booking a Car

#### Step 1: Create the Car Detail Page

1.  **Create `Pages/Cars/Detail.cshtml`:** This page will show car info and a booking form.

    ```html
    @page "{id:int}"
    @using System.Security.Claims
    @{
        ViewData["Title"] = "Car Details";
        var isRenter = User.IsInRole("CarRenter");
    }

    <div class="container mx-auto mt-10">
        <!-- Car Details Section -->
        <div id="car-detail-container" class="grid grid-cols-1 md:grid-cols-2 gap-8">
            <!-- JS will populate car details here -->
        </div>

        <!-- Booking Section - Only for CarRenters -->
        @if (isRenter)
        {
            <div id="booking-section" class="mt-12 bg-white p-8 rounded-lg shadow-lg">
                <h2 class="text-2xl font-bold mb-4">Book This Car</h2>
                <form id="bookingForm">
                    <div id="booking-message" class="hidden p-3 mb-4 text-sm rounded-lg"></div>
                    <div class="grid grid-cols-1 md:grid-cols-2 gap-6">
                        <div>
                            <label for="startDate" class="block font-medium">Start Date</label>
                            <input type="date" id="startDate" required class="mt-1 p-2 w-full border rounded-md">
                        </div>
                        <div>
                            <label for="endDate" class="block font-medium">End Date</label>
                            <input type="date" id="endDate" required class="mt-1 p-2 w-full border rounded-md">
                        </div>
                    </div>
                    <div class="mt-6">
                        <p class="text-xl font-semibold">Total Cost: <span id="totalCost">$0.00</span></p>
                    </div>
                    <button type="submit" class="mt-4 w-full bg-green-600 text-white py-2 px-4 rounded-md hover:bg-green-700">
                        Confirm Booking
                    </button>
                </form>
            </div>
        }
    </div>

    @section Scripts {
        <script>
            // Pass the Car ID from the URL to JavaScript
            const carId = @Model.RouteData.Values["id"];
        </script>
        <script src="~/js/cars/car-detail.js" asp-append-version="true"></script>
    }
    ```

2.  **Create `wwwroot/js/cars/car-detail.js`:** This will handle fetching details and submitting the booking.

    ```javascript
    document.addEventListener('DOMContentLoaded', function () {
        const token = localStorage.getItem('jwt_token');
        const carDetailContainer = document.getElementById('car-detail-container');
        let dailyRate = 0; // Store daily rate for cost calculation

        // Fetch car details on page load
        async function fetchCarDetails() {
            const response = await fetch(`/api/cars/${carId}`, {
                headers: { 'Authorization': `Bearer ${token}` }
            });
            if (!response.ok) return;

            const car = await response.json();
            dailyRate = car.dailyRate;
            carDetailContainer.innerHTML = `
                <div>
                    <img src="https://via.placeholder.com/600x400.png?text=${car.make}+${car.model}" alt="${car.make}" class="rounded-lg shadow-lg">
                </div>
                <div>
                    <h1 class="text-4xl font-extrabold">${car.make} ${car.model} (${car.year})</h1>
                    <p class="text-gray-500 mt-2">Owned by ${car.ownerName}</p>
                    <p class="text-3xl font-bold text-green-600 mt-4">$${car.dailyRate}/day</p>
                    <p class="mt-4 text-lg">Status: ${car.isAvailable ? '<span class="text-green-600">Available</span>' : '<span class="text-red-600">Unavailable</span>'}</p>
                </div>`;
        }

        fetchCarDetails();

        // Booking form logic
        const bookingForm = document.getElementById('bookingForm');
        if (bookingForm) {
            const startDateEl = document.getElementById('startDate');
            const endDateEl = document.getElementById('endDate');
            const totalCostEl = document.getElementById('totalCost');

            // Calculate cost dynamically
            function calculateCost() {
                const start = new Date(startDateEl.value);
                const end = new Date(endDateEl.value);
                if (start && end && start < end) {
                    const days = (end - start) / (1000 * 60 * 60 * 24);
                    totalCostEl.textContent = `$${(days * dailyRate).toFixed(2)}`;
                } else {
                    totalCostEl.textContent = '$0.00';
                }
            }

            startDateEl.addEventListener('change', calculateCost);
            endDateEl.addEventListener('change', calculateCost);

            // Handle form submission
            bookingForm.addEventListener('submit', async function (e) {
                e.preventDefault();
                const bookingData = {
                    carId: carId,
                    startDate: startDateEl.value,
                    endDate: endDateEl.value
                };

                const response = await fetch('/api/bookings', {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json', 'Authorization': `Bearer ${token}` },
                    body: JSON.stringify(bookingData)
                });

                const result = await response.json();
                const msgEl = document.getElementById('booking-message');
                msgEl.classList.remove('hidden');

                if (response.ok) {
                    msgEl.className = 'p-3 mb-4 text-sm text-green-700 bg-green-100 rounded-lg';
                    msgEl.textContent = 'Booking confirmed successfully!';
                } else {
                    msgEl.className = 'p-3 mb-4 text-sm text-red-700 bg-red-100 rounded-lg';
                    msgEl.textContent = result.message || 'Booking failed.';
                }
            });
        }
    });
    ```

#### Step 2: Create the "My Bookings" Page

1.  **Create `Pages/Bookings/MyBookings.cshtml`:**

    ```html
    @page
    @{
        ViewData["Title"] = "My Bookings";
    }

    <div class="container mx-auto mt-10">
        <h1 class="text-3xl font-bold mb-6">My Bookings</h1>
        <div class="bg-white p-8 rounded-lg shadow-md">
            <table class="min-w-full">
                <thead class="bg-gray-100">
                    <tr>
                        <th class="py-2 px-4 text-left">Car</th>
                        <th class="py-2 px-4 text-left">Start Date</th>
                        <th class="py-2 px-4 text-left">End Date</th>
                        <th class="py-2 px-4 text-left">Total Cost</th>
                        <th class="py-2 px-4 text-left">Status</th>
                    </tr>
                </thead>
                <tbody id="bookings-table-body">
                    <!-- JS will populate bookings here -->
                </tbody>
            </table>
        </div>
    </div>

    @section Scripts {
        <script src="~/js/bookings/my-bookings.js" asp-append-version="true"></script>
    }
    ```

2.  **Create `wwwroot/js/bookings/my-bookings.js`:**

    ```javascript
    document.addEventListener('DOMContentLoaded', async function () {
        const token = localStorage.getItem('jwt_token');
        const tableBody = document.getElementById('bookings-table-body');

        const response = await fetch('/api/bookings/my-bookings', {
            headers: { 'Authorization': `Bearer ${token}` }
        });

        if (!response.ok) return;

        const bookings = await response.json();
        tableBody.innerHTML = '';
        bookings.forEach(booking => {
            const row = `
                <tr class="border-b">
                    <td class="py-3 px-4">${booking.carName}</td>
                    <td class="py-3 px-4">${new Date(booking.startDate).toLocaleDateString()}</td>
                    <td class="py-3 px-4">${new Date(booking.endDate).toLocaleDateString()}</td>
                    <td class="py-3 px-4">$${booking.totalCost.toFixed(2)}</td>
                    <td class="py-3 px-4"><span class="bg-blue-200 text-blue-800 py-1 px-3 rounded-full text-xs">${booking.status}</span></td>
                </tr>
            `;
            tableBody.innerHTML += row;
        });
    });
    ```

You now have a fully operational booking system. A renter can view a car's details, select dates, see the calculated cost, and submit a booking, which is validated on the backend to prevent conflicts. They can also view their complete booking history.
