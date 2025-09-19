** USER : **
1. Then Great,  What do you going to do next
2. Ok then lets implement this

---
Excellent. We've just established the Admin's ability to oversee users. Now, we need to give them the same level of oversight for the core business objects: **Cars and Bookings**.

The Admin Dashboard currently has placeholder links for "Car Management" and "Booking Management." Our next step is to build the functional pages behind those links. This will complete the foundational set of tools for any administrator on the platform.

### Next Step: Implement System-Wide Car & Booking Management Panels

We will tackle this as a two-part feature to fully equip the Admin Panel.

#### Part 1: System-Wide Car Management Panel

The goal is to create a page where an administrator can view every single car listed on the platform, regardless of who owns it.

*   **Backend:** The good news is that we've already done most of the work here. Our existing `GET /api/cars` endpoint is secured for Admins and returns all the necessary data. We don't need to build anything new on the backend for this part.
*   **Frontend:** We will create a new Razor Page at `Pages/Admin/Cars.cshtml`. This page will feature a comprehensive table displaying all cars with key details like:
    *   Car ID
    *   Make & Model
    *   Owner's Name
    *   Daily Rate
    *   Availability Status
    *   Registration Number
*   **Integration:** We'll update the "Car Management" link on the Admin Dashboard to point to this new page.

#### Part 2: System-Wide Booking Management Panel

This is a crucial feature for customer support and oversight. An Admin needs to be able to look up any booking in the system.

*   **Backend:** This requires a new flow similar to the User Management one. We will:
    1.  Create an `AdminBookingViewDto` to hold detailed booking information (including car, owner, and renter names).
    2.  Add a `GetAllBookingsAsync()` method to our `AdminService`.
    3.  Create a new `GET /api/admin/bookings` endpoint in the `AdminController`.
*   **Frontend:** We will create a new Razor Page at `Pages/Admin/Bookings.cshtml`. This page will display a table of all bookings with columns for:
    *   Booking ID
    *   Car Name
    *   Renter's Name
    *   Owner's Name
    *   Booking Dates
    *   Total Cost
    *   Payment Status
*   **Integration:** We'll update the "Booking Management" link on the Admin Dashboard.

Completing these two panels will give the administrator a full 360-degree view of the platform's operations, making it a truly manageable system.

Let's begin by building the **System-Wide Car Management Panel**, as it's the quickest to implement and will give us an immediate win.

---
Of course. Let's implement the complete Car and Booking Management panels for the Admin. We'll start with the Car Management panel, which is more straightforward as the backend is already in place.

---

### Part 1: System-Wide Car Management Panel

#### Step 1: Verify Backend (No New Code Needed)

Our existing `CarsController` already has the endpoint we need:
```csharp
// In CraRental.Web/Controllers/CarsController.cs
[HttpGet]
[Authorize(Roles = "CarRenter, Admin")] // Already secured for Admins!
public async Task<IActionResult> GetAllCars()
{
    var cars = await _carService.GetAllCarsAsync();
    return Ok(cars);
}
```
The `CarViewDto` it returns (`Id`, `Make`, `Model`, `Year`, `DailyRate`, `IsAvailable`, `OwnerName`) is perfect for our admin table. We can proceed directly to the frontend.

#### Step 2: Create the Admin Car Management Page

1.  **Create `Pages/Admin/Cars.cshtml`**:
    ```html
    @page
    @{
        ViewData["Title"] = "Car Management";
    }

    <div class="container mx-auto mt-10">
        <div class="flex justify-between items-center mb-8">
            <h1 class="text-3xl font-bold">Car Management</h1>
            <a asp-page="/Admin/Dashboard" class="text-blue-600 hover:underline">&larr; Back to Dashboard</a>
        </div>

        <div class="bg-white p-6 rounded-lg shadow-md">
            <table class="min-w-full">
                <thead class="bg-gray-100">
                    <tr>
                        <th class="py-2 px-4 text-left">Car ID</th>
                        <th class="py-2 px-4 text-left">Make & Model</th>
                        <th class="py-2 px-4 text-left">Owner</th>
                        <th class="py-2 px-4 text-left">Rate</th>
                        <th class="py-2 px-4 text-left">Status</th>
                    </tr>
                </thead>
                <tbody id="cars-table-body">
                    <tr><td colspan="5" class="text-center py-10"><p>Loading car data...</p></td></tr>
                </tbody>
            </table>
        </div>
    </div>

    @section Scripts {
        <script src="~/js/admin/cars.js" asp-append-version="true"></script>
    }
    ```

2.  **Create `wwwroot/js/admin/cars.js`**:
    ```javascript
    document.addEventListener('DOMContentLoaded', async function() {
        const token = localStorage.getItem('jwt_token');
        if (!token) {
            window.location.href = '/login';
            return;
        }

        const tableBody = document.getElementById('cars-table-body');

        try {
            // We use the existing /api/cars endpoint
            const response = await fetch('/api/cars', {
                headers: { 'Authorization': `Bearer ${token}` }
            });

            if (!response.ok) throw new Error('Failed to fetch car data.');

            const cars = await response.json();
            tableBody.innerHTML = ''; // Clear loading message

            if (cars.length === 0) {
                tableBody.innerHTML = '<tr><td colspan="5" class="text-center py-10">No cars found in the system.</td></tr>';
                return;
            }

            cars.forEach(car => {
                const row = `
                    <tr class="border-b hover:bg-gray-50">
                        <td class="py-3 px-4">${car.id}</td>
                        <td class="py-3 px-4 font-medium text-gray-900">${car.make} ${car.model} (${car.year})</td>
                        <td class="py-3 px-4">${car.ownerName}</td>
                        <td class="py-3 px-4">$${car.dailyRate.toFixed(2)}</td>
                        <td class="py-3 px-4">
                            ${car.isAvailable
                                ? '<span class="text-green-600 font-semibold">Available</span>'
                                : '<span class="text-red-600 font-semibold">Rented</span>'
                            }
                        </td>
                    </tr>
                `;
                tableBody.innerHTML += row;
            });

        } catch (error) {
            tableBody.innerHTML = `<tr><td colspan="5" class="text-center py-10 text-red-500">Error: ${error.message}</td></tr>`;
        }
    });
    ```

3.  **Update the Admin Dashboard Link**: Open `Pages/Admin/Dashboard.cshtml` and change the link.
    ```html
    <a href="/Admin/Cars" class="bg-white p-6 rounded-lg shadow-md hover:shadow-xl transition-shadow">
        <h3 class="font-bold text-lg">Car Management</h3>
        <p class="text-gray-600 mt-2">View and manage all listed cars.</p>
    </a>
    ```

---

### Part 2: System-Wide Booking Management Panel

This requires a new backend flow.

#### Step 1: Backend Implementation

1.  **Create `AdminBookingViewDto.cs`** in the `DTOs` folder:
    ```csharp
    // CraRental.Web/DTOs/AdminBookingViewDto.cs
    namespace CraRental.Web.DTOs
    {
        public class AdminBookingViewDto
        {
            public int BookingId { get; set; }
            public string CarName { get; set; }
            public string RenterName { get; set; }
            public string OwnerName { get; set; }
            public DateTime StartDate { get; set; }
            public DateTime EndDate { get; set; }
            public decimal TotalCost { get; set; }
            public string PaymentStatus { get; set; }
        }
    }
    ```

2.  **Update `IAdminService.cs`**:
    ```csharp
    public interface IAdminService
    {
        Task<AdminDashboardStatsDto> GetDashboardStatsAsync();
        Task<IEnumerable<AdminUserViewDto>> GetAllUsersAsync();
        Task<IEnumerable<AdminBookingViewDto>> GetAllBookingsAsync(); // Add this
    }
    ```

3.  **Implement `GetAllBookingsAsync` in `AdminService.cs`**:
    ```csharp
    // In CraRental.Web/Services/AdminService.cs
    public async Task<IEnumerable<AdminBookingViewDto>> GetAllBookingsAsync()
    {
        var bookings = await _context.Bookings
            .Include(b => b.Car)
                .ThenInclude(c => c.Owner)
            .Include(b => b.Renter)
            .Include(b => b.Payment) // Include payment to check its existence
            .OrderByDescending(b => b.StartDate)
            .ToListAsync();

        return bookings.Select(b => new AdminBookingViewDto
        {
            BookingId = b.Id,
            CarName = $"{b.Car.Make} {b.Car.Model}",
            RenterName = $"{b.Renter.FirstName} {b.Renter.LastName}",
            OwnerName = $"{b.Car.Owner.FirstName} {b.Car.Owner.LastName}",
            StartDate = b.StartDate,
            EndDate = b.EndDate,
            TotalCost = b.TotalCost,
            PaymentStatus = b.Payment != null ? "Paid" : "Unpaid"
        });
    }
    ```

4.  **Add the endpoint to `AdminController.cs`**:
    ```csharp
    // In CraRental.Web/Controllers/AdminController.cs
    [HttpGet("bookings")]
    public async Task<IActionResult> GetAllBookings()
    {
        var bookings = await _adminService.GetAllBookingsAsync();
        return Ok(bookings);
    }
    ```

#### Step 2: Frontend Implementation

1.  **Create `Pages/Admin/Bookings.cshtml`**:
    ```html
    @page
    @{
        ViewData["Title"] = "Booking Management";
    }
    <div class="container mx-auto mt-10">
        <div class="flex justify-between items-center mb-8">
            <h1 class="text-3xl font-bold">Booking Management</h1>
            <a asp-page="/Admin/Dashboard" class="text-blue-600 hover:underline">&larr; Back to Dashboard</a>
        </div>
        <div class="bg-white p-6 rounded-lg shadow-md">
            <table class="min-w-full">
                <thead class="bg-gray-100">
                    <tr>
                        <th class="py-2 px-4 text-left">Booking ID</th>
                        <th class="py-2 px-4 text-left">Car</th>
                        <th class="py-2 px-4 text-left">Renter</th>
                        <th class="py-2 px-4 text-left">Owner</th>
                        <th class="py-2 px-4 text-left">Dates</th>
                        <th class="py-2 px-4 text-left">Cost</th>
                        <th class="py-2 px-4 text-left">Payment</th>
                    </tr>
                </thead>
                <tbody id="bookings-table-body">
                    <tr><td colspan="7" class="text-center py-10"><p>Loading booking data...</p></td></tr>
                </tbody>
            </table>
        </div>
    </div>
    @section Scripts {
        <script src="~/js/admin/bookings.js" asp-append-version="true"></script>
    }
    ```

2.  **Create `wwwroot/js/admin/bookings.js`**:
    ```javascript
    document.addEventListener('DOMContentLoaded', async function() {
        const token = localStorage.getItem('jwt_token');
        if (!token) { window.location.href = '/login'; return; }

        const tableBody = document.getElementById('bookings-table-body');

        try {
            const response = await fetch('/api/admin/bookings', {
                headers: { 'Authorization': `Bearer ${token}` }
            });

            if (!response.ok) throw new Error('Failed to fetch booking data.');

            const bookings = await response.json();
            tableBody.innerHTML = '';

            if (bookings.length === 0) {
                tableBody.innerHTML = '<tr><td colspan="7" class="text-center py-10">No bookings found.</td></tr>';
                return;
            }

            bookings.forEach(b => {
                const row = `
                    <tr class="border-b hover:bg-gray-50">
                        <td class="py-3 px-4 font-semibold">#${b.bookingId}</td>
                        <td class="py-3 px-4">${b.carName}</td>
                        <td class="py-3 px-4">${b.renterName}</td>
                        <td class="py-3 px-4">${b.ownerName}</td>
                        <td class="py-3 px-4">${new Date(b.startDate).toLocaleDateString()} - ${new Date(b.endDate).toLocaleDateString()}</td>
                        <td class="py-3 px-4">$${b.totalCost.toFixed(2)}</td>
                        <td class="py-3 px-4">
                            ${b.paymentStatus === 'Paid'
                                ? '<span class="text-green-600 font-semibold">Paid</span>'
                                : '<span class="text-orange-600 font-semibold">Unpaid</span>'
                            }
                        </td>
                    </tr>
                `;
                tableBody.innerHTML += row;
            });
        } catch (error) {
            tableBody.innerHTML = `<tr><td colspan="7" class="text-center py-10 text-red-500">Error: ${error.message}</td></tr>`;
        }
    });
    ```

3.  **Update the Admin Dashboard Link**: Open `Pages/Admin/Dashboard.cshtml`.
    ```html
    <a href="/Admin/Bookings" class="bg-white p-6 rounded-lg shadow-md hover:shadow-xl transition-shadow">
        <h3 class="font-bold text-lg">Booking Management</h3>
        <p class="text-gray-600 mt-2">View all bookings in the system.</p>
    </a>
    ```

You have now fully equipped the Admin Panel with comprehensive management views for all major aspects of the platform: users, cars, and bookings. This provides the administrator with the necessary tools for oversight, support, and moderation.


