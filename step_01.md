# CarRentalManagement-UnicomTIC-2025

### Database Blueprint for Your CraRental Management System

Here is a recommended database table structure for your ASP.NET Core Razor Pages project, designed to support a custom token-based role authentication system using JWT and separate tables for different user roles.

This schema is normalized to reduce data redundancy and improve data integrity. It features a central `Users` table for authentication and separate profile tables for `CarOwners` and `CarRenters` to hold role-specific information.

### Core Authentication and User Roles

The foundation of the system is a central `Users` table that stores the essential information for authentication and role management.

**1. Users Table:** This table contains the core credentials and role information for every user in the system.

| Column Name | Data Type | Constraints | Description |
|---|---|---|---|
| `Id` | `INTEGER` | Primary Key, Autoincrement | Unique identifier for each user. |
| `Email` | `TEXT` | Not Null, Unique | The user's email address, used for login. |
| `PasswordHash` | `TEXT` | Not Null | The hashed version of the user's password. |
| `Role` | `TEXT` | Not Null | The user's role (e.g., "Admin", "Staff", "CarOwner", "CarRenter"). |
| `CreatedAt` | `TEXT` | Not Null | Timestamp of when the user account was created. |
| `UpdatedAt` | `TEXT` | | Timestamp of the last update to the user's record. |

**2. UserRoles Table (Optional but Recommended):** For a more scalable solution, you can normalize the user roles into a separate table.

| Column Name | Data Type | Constraints | Description |
|---|---|---|---|
| `Id` | `INTEGER` | Primary Key, Autoincrement | Unique identifier for each role. |
| `RoleName` | `TEXT` | Not Null, Unique | The name of the role (e.g., "Admin", "Staff", "CarOwner", "CarRenter"). |

If you use this table, the `Role` column in the `Users` table would be replaced with a `RoleId` column, which would be a foreign key to the `UserRoles` table.

### User Profile Tables

To keep the `Users` table lean and focused on authentication, role-specific information is stored in separate tables. This design avoids having many nullable columns in the `Users` table for attributes that only apply to certain roles.

**3. CarOwners Table:** This table stores information specific to users who own cars available for rent.

| Column Name | Data Type | Constraints | Description |
|---|---|---|---|
| `Id` | `INTEGER` | Primary Key, Autoincrement | Unique identifier for each car owner. |
| `UserId` | `INTEGER` | Not Null, Foreign Key (Users.Id) | A one-to-one relationship with the Users table. |
| `FirstName` | `TEXT` | Not Null | The car owner's first name. |
| `LastName` | `TEXT` | Not Null | The car owner's last name. |
| `PhoneNumber` | `TEXT` | | The car owner's contact phone number. |
| `Address` | `TEXT` | | The car owner's address. |

**4. CarRenters Table:** This table holds information specific to users who rent cars.

| Column Name | Data Type | Constraints | Description |
|---|---|---|---|
| `Id` | `INTEGER` | Primary Key, Autoincrement | Unique identifier for each car renter. |
| `UserId` | `INTEGER` | Not Null, Foreign Key (Users.Id) | A one-to-one relationship with the Users table. |
| `FirstName` | `TEXT` | Not Null | The car renter's first name. |
| `LastName` | `TEXT` | Not Null | The car renter's last name. |
| `PhoneNumber` | `TEXT` | | The car renter's contact phone number. |
| `DriverLicenseNumber` | `TEXT` | | The car renter's driver's license number. |

### Car and Rental Management

The following tables are essential for managing the core business logic of the car rental system.

**5. Cars Table:** This table contains details about the vehicles available for rent.

| Column Name | Data Type | Constraints | Description |
|---|---|---|---|
| `Id` | `INTEGER` | Primary Key, Autoincrement | Unique identifier for each car. |
| `OwnerId` | `INTEGER` | Not Null, Foreign Key (CarOwners.Id) | The car owner who listed the car. |
| `Make` | `TEXT` | Not Null | The manufacturer of the car (e.g., Toyota). |
| `Model` | `TEXT` | Not Null | The model of the car (e.g., Camry). |
| `Year` | `INTEGER` | Not Null | The manufacturing year of the car. |
| `RegistrationNumber` | `TEXT` | Not Null, Unique | The vehicle's registration number. |
| `DailyRate` | `REAL` | Not Null | The cost to rent the car per day. |
| `IsAvailable` | `INTEGER` | Not Null, Default 1 | A flag to indicate if the car is currently available for rent (1 for true, 0 for false). |

**6. Bookings Table:** This table tracks all the rental bookings made by users.

| Column Name | Data Type | Constraints | Description |
|---|---|---|---|
| `Id` | `INTEGER` | Primary Key, Autoincrement | Unique identifier for each booking. |
| `CarId` | `INTEGER` | Not Null, Foreign Key (Cars.Id) | The car being booked. |
| `RenterId` | `INTEGER` | Not Null, Foreign Key (CarRenters.Id) | The user who is renting the car. |
| `StartDate` | `TEXT` | Not Null | The start date of the rental period. |
| `EndDate` | `TEXT` | Not Null | The end date of the rental period. |
| `TotalCost` | `REAL` | Not Null | The total cost of the rental. |
| `Status` | `TEXT` | Not Null | The status of the booking (e.g., "Confirmed", "Completed", "Cancelled"). |

**7. Payments Table:** This table records the payment details for each booking.

| Column Name | Data Type | Constraints | Description |
|---|---|---|---|
| `Id` | `INTEGER` | Primary Key, Autoincrement | Unique identifier for each payment. |
| `BookingId` | `INTEGER` | Not Null, Foreign Key (Bookings.Id) | The booking associated with the payment. |
| `Amount` | `REAL` | Not Null | The amount paid. |
| `PaymentDate` | `TEXT` | Not Null | The date the payment was made. |
| `PaymentMethod` | `TEXT` | | The method of payment (e.g., "Credit Card", "PayPal"). |
| `TransactionId` | `TEXT` | | The transaction identifier from the payment gateway. |

This database schema provides a solid foundation for your car rental management system, offering a clear separation of concerns between user authentication and role-specific data, while effectively managing the core entities of your application.
