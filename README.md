# Testing Guide — WebSec Transport Booking System
## Testing All 16 Requirements

---

## Prerequisites

1. **Start XAMPP** → Enable **Apache** and **MySQL**
2. **Create database** `db_a1` in phpMyAdmin (if not already created)
3. **Run migrations and seeders:**
   ```
   php artisan migrate:fresh --seed
   ```
4. Open the app at: **http://localhost/Assignment_2/public**

### Seeded Accounts

| Email | Password | Role |
|---|---|---|
| admin@transport.com | Admin@123 | transport_admin |

---

## Requirement 1: Define permissions 'create_trip', 'book_trip', and 'cancel_trip'

### Test Steps:
1. Open **phpMyAdmin** → Select database `db_a1`
2. Open the `permissions` table
3. **Verify** that exactly 3 rows exist:

| id | name |
|---|---|
| 1 | create_trip |
| 2 | book_trip |
| 3 | cancel_trip |

### Expected Result:
✅ Three permissions exist in the `permissions` table.

---

## Requirement 2: Define three roles: transport_admin, driver, and passenger

### Test Steps:
1. Open **phpMyAdmin** → Select database `db_a1`
2. Open the `roles` table
3. **Verify** that exactly 3 rows exist:

| id | name |
|---|---|
| 1 | transport_admin |
| 2 | driver |
| 3 | passenger |

### Expected Result:
✅ Three roles exist in the `roles` table.

---

## Requirement 3: The driver role has permission to create trips and view passengers

### Test Steps:
1. Open **phpMyAdmin** → Open `permission_role` table
2. **Verify** that the `driver` role (id=2) is linked to `create_trip` permission (id=1)
3. **Login as admin** (admin@transport.com / Admin@123)
4. Go to **Manage Users** → Register a new user first, then assign them the `driver` role
5. **Logout** and **login as the driver user**
6. Go to **Trips** page
7. **Verify** that the driver can see a **"+ Create New Trip"** button
8. Create a trip, then have a passenger book it (test later)
9. Return to the **Trips** page as the driver
10. **Verify** the **Passengers** column shows booked passenger names

### Expected Result:
✅ Driver has `create_trip` permission in the database.
✅ Driver can see the "Create Trip" button.
✅ Driver can see passenger names for each trip in the Passengers column.

---

## Requirement 4: The passenger role has permission to book and cancel their own trips

### Test Steps:
1. Open **phpMyAdmin** → Open `permission_role` table
2. **Verify** that the `passenger` role (id=3) is linked to:
   - `book_trip` permission (id=2)
   - `cancel_trip` permission (id=3)
3. **Register a new user** (they will automatically be a passenger — tested in Req #5)
4. **Login as the passenger**
5. Go to **Trips** page → **Verify** a "Book" button appears next to each trip
6. Go to **My Bookings** → **Verify** a "Cancel" button appears next to booked trips

### Expected Result:
✅ Passenger has `book_trip` and `cancel_trip` permissions in the database.
✅ Passenger can see "Book" and "Cancel" buttons.

---

## Requirement 5: Any newly registered users should take the passenger role by default

### Test Steps:
1. Go to **http://localhost/Assignment_2/public/register**
2. Register a new user:
   - Name: `Test Passenger`
   - Email: `passenger@test.com`
   - Password: `Test@1234`
   - Confirm Password: `Test@1234`
3. Click **Register**
4. **Login** with the new credentials
5. **Check the navbar** → The user's name should show with a badge: **passenger**
6. **Verify in phpMyAdmin** → Open `role_user` table → The new user should have `role_id = 3` (passenger)

### Expected Result:
✅ New user is automatically assigned the `passenger` role.
✅ Navbar shows "passenger" badge next to the user's name.

---

## Requirement 6: Only an admin can assign the driver or transport_admin roles

### Test Steps:

**Test A — Admin CAN assign roles:**
1. **Login as admin** (admin@transport.com / Admin@123)
2. Go to **Manage Users** page
3. Find the test user (`passenger@test.com`)
4. Select `driver` from the dropdown → Click **Assign**
5. **Verify** the user now shows both `passenger` and `driver` badges

**Test B — Non-admin CANNOT access role management:**
1. **Logout** and **login as a passenger** (passenger@test.com / Test@1234)
2. Try to access **http://localhost/Assignment_2/public/admin/users** directly in the URL bar
3. **Verify** that a **403 Forbidden** error page is returned

### Expected Result:
✅ Admin can assign roles via the Manage Users page.
✅ Non-admin users get a 403 error when trying to access the admin page.

---

## Requirement 7: The create_trip permission allows users to create new trips

### Test Steps:
1. **Login as a driver** (user with driver role assigned in Req #6)
2. Click **Create Trip** in the navbar
3. Fill in the form:
   - Departure Location: `Cairo`
   - Destination: `Alexandria`
   - Departure Time: (any future date/time)
   - Available Seats: `4`
   - Price: `150.00`
4. Click **Create Trip**
5. **Verify** redirect to Trips page with success message: "Trip created successfully!"
6. **Verify** the new trip appears in the list

**Negative Test — Passenger cannot create trips:**
1. **Login as a passenger** (without driver role)
2. Try to access **http://localhost/Assignment_2/public/trips/create** directly
3. **Verify** that a **403 Forbidden** error is returned

### Expected Result:
✅ Driver can create trips.
✅ Passenger gets 403 when trying to create a trip.

---

## Requirement 8: The book_trip permission allows users to book available trips

### Test Steps:
1. **Login as a passenger** (passenger@test.com / Test@1234)
2. Go to **Trips** page
3. Find the trip created in Req #7
4. Click the **Book** button
5. **Verify** redirect to My Bookings with message: "Trip booked successfully!"
6. **Verify** the booking appears with status **Booked**

**Negative Test — Try to book a full trip:**
1. Create a trip with only 1 seat (as a driver)
2. Book it as passenger A
3. Login as passenger B and try to book the same trip
4. **Verify** error message: "No available seats for this trip."

### Expected Result:
✅ Passenger can book trips.
✅ Full trips show "Full" badge instead of Book button.

---

## Requirement 9: The cancel_trip permission allows users to cancel trips

### Test Steps:
1. **Login as a passenger** with a booked trip
2. Go to **My Bookings**
3. Find the booked trip → Click **Cancel**
4. Confirm the cancellation when prompted
5. **Verify** the booking status changes to **Cancelled** (red badge)

### Expected Result:
✅ Passenger can cancel their own bookings.
✅ Status badge changes from green "Booked" to red "Cancelled".

---

## Requirement 10: A user with the passenger role can only view and manage their own bookings

### Test Steps:
1. Register **two different passengers**:
   - Passenger A: `passengerA@test.com` / `Test@1234`
   - Passenger B: `passengerB@test.com` / `Test@1234`
2. **Login as Passenger A** → Book a trip → Go to **My Bookings**
3. **Verify** Passenger A can see ONLY their own booking
4. **Logout** → **Login as Passenger B** → Go to **My Bookings**
5. **Verify** Passenger B sees an empty bookings page (or only their own bookings)
6. Passenger B should NOT see Passenger A's booking

### Expected Result:
✅ Each passenger only sees their own bookings in My Bookings.
✅ No cross-user booking visibility.

---

## Requirement 11: Add a 'Request Refund' button beside cancelled trips

### Test Steps:
1. **Login as a passenger** who has a cancelled booking (from Req #9)
2. Go to **My Bookings**
3. Find the **cancelled** booking
4. **Verify** a red **"Request Refund"** button appears next to the cancelled booking
5. Click **Request Refund**
6. **Verify** success message: "Refund request submitted successfully."
7. **Verify** the button is replaced by an info badge: **"Refund Pending"**

**Note:** The "Request Refund" button should NOT appear next to booked (active) trips — only cancelled ones.

### Expected Result:
✅ "Request Refund" button visible only beside cancelled bookings.
✅ After clicking, button changes to "Refund Pending" badge.
✅ Button does NOT appear for active bookings.

---

## Requirement 12: Drivers and transport_admins can see the refund request status beside each trip

### Test Steps:
1. **Login as a driver** (or admin)
2. Click **Refund Requests** in the navbar
3. **Verify** the refund requests table shows:
   - Passenger name
   - Trip details (From → To)
   - Trip driver name
   - Booking date
   - **Refund Status** column showing **"Pending"** (yellow badge)
4. **Verify** all refund requests from all passengers are visible

**Negative Test — Passenger cannot access refunds page:**
1. **Login as a passenger**
2. Try to access **http://localhost/Assignment_2/public/refunds** directly
3. **Verify** a **403 Forbidden** error is returned

### Expected Result:
✅ Drivers/admins can see all refund requests with status badges.
✅ Passengers cannot access the refunds management page.

---

## Requirement 13: If a driver approves the refund, the refund request status changes to 'closed'

### Test Steps:
1. **Login as a driver** (or admin)
2. Go to **Refund Requests**
3. Find the pending refund request from Req #11
4. Click the **"Approve Refund"** button
5. Confirm when prompted
6. **Verify** success message: "Refund approved. Status changed to closed."
7. **Verify** the status badge changes from **"Pending" (yellow)** to **"Closed" (green)**
8. **Verify** the "Approve Refund" button is replaced by text: "Approved"

**Database Verification:**
1. Open **phpMyAdmin** → `refund_requests` table
2. **Verify** the refund record now has `status = 'closed'`

### Expected Result:
✅ Driver can approve refund requests.
✅ Status changes from "pending" to "closed" in both UI and database.

---

## Requirement 14: Using OpenSSL, create a new Certificate Authority named 'Transport Root'

### Test Steps:
1. Navigate to `Assignment_2/ssl/certs/` folder
2. **Verify** the following CA files exist:
   - `transport_root_ca.key` — CA private key (4096-bit RSA)
   - `transport_root_ca.crt` — CA certificate
3. **Inspect the CA certificate** by running:
   ```
   C:\xampp\apache\bin\openssl.exe x509 -in ssl\certs\transport_root_ca.crt -text -noout
   ```
4. **Verify** the output shows:
   - `Issuer: ... O = Transport Root, ... CN = Transport Root`
   - `Subject: ... O = Transport Root, ... CN = Transport Root`
   - It is self-signed (Issuer == Subject)

### Expected Result:
✅ CA certificate exists with organization name "Transport Root".
✅ CA certificate is self-signed.
✅ CA private key is 4096-bit RSA.

---

## Requirement 15: Make a website certificate for www.secure-transport.com that expires in one year

### Test Steps:
1. Navigate to `Assignment_2/ssl/certs/` folder
2. **Verify** the following server certificate files exist:
   - `secure_transport.key` — Server private key
   - `secure_transport.csr` — Certificate Signing Request
   - `secure_transport.crt` — Signed server certificate
3. **Inspect the server certificate:**
   ```
   C:\xampp\apache\bin\openssl.exe x509 -in ssl\certs\secure_transport.crt -text -noout
   ```
4. **Verify** the output shows:
   - `Subject: ... CN = www.secure-transport.com`
   - `Issuer: ... O = Transport Root, CN = Transport Root` (signed by our CA)
   - `Not Before:` and `Not After:` dates are exactly **365 days apart** (1 year)
   - `Subject Alternative Name: DNS:www.secure-transport.com, DNS:secure-transport.com`

5. **Verify the certificate chain:**
   ```
   C:\xampp\apache\bin\openssl.exe verify -CAfile ssl\certs\transport_root_ca.crt ssl\certs\secure_transport.crt
   ```
   Expected output: `ssl\certs\secure_transport.crt: OK`

### Expected Result:
✅ Server certificate has CN = www.secure-transport.com.
✅ Certificate is signed by "Transport Root" CA.
✅ Certificate expires in exactly 1 year (365 days).
✅ Certificate chain verification passes: **OK**.

---

## Requirement 16: Implement a check for common web security vulnerabilities

### 16a. Authentication Security

**Test — Auth middleware protects routes:**
1. **Logout** (or use incognito browser)
2. Try to access **http://localhost/Assignment_2/public/trips** directly
3. **Verify** you are redirected to the **login page**

**Test — Session fixation prevention:**
1. Open browser Developer Tools → Application → Cookies
2. Note the session cookie value before login
3. **Login** with valid credentials
4. **Verify** the session cookie value has **changed** (session regenerated)

### 16b. SQL Injection

**Test — Login form SQL injection:**
1. Go to **Login** page
2. Enter:
   - Email: `' OR 1=1 --`
   - Password: `anything`
3. Click **Login**
4. **Verify** login fails with error: "The email field must be a valid email address."
   (Laravel validates email format before query)

**Test — Trip creation SQL injection:**
1. Login as a **driver**
2. Go to **Create Trip**
3. Enter destination: `'; DROP TABLE trips; --`
4. Submit the form
5. **Verify** the trip is created with the literal text as destination (stored safely)
6. **Verify** in phpMyAdmin that the `trips` table still exists and is intact

### 16c. Weak Passwords

**Test — Weak password rejected:**
1. Go to **Register** page
2. Try registering with weak passwords:

| Password | Expected Error |
|---|---|
| `123` | "The password field must be at least 8 characters." |
| `abcdefgh` | "Password must contain at least one uppercase letter, one lowercase letter, one digit, and one special character" |
| `ABCDEFGH` | Same error (no lowercase, digit, or special char) |
| `Abcdefg1` | Same error (no special character) |
| `Abcdef@1` | ✅ Accepted (meets all requirements) |

3. **Verify** only passwords with 8+ characters, uppercase, lowercase, digit, and special character are accepted

### 16d. Cross-Site Scripting (XSS)

**Test — XSS in trip creation:**
1. Login as a **driver**
2. Go to **Create Trip**
3. Enter destination: `<script>alert('XSS')</script>`
4. Submit the form
5. Go to **Trips** page
6. **Verify** the script tag is displayed as **plain text**, NOT executed
7. **Verify** no JavaScript alert box pops up
8. View page source → **Verify** the output is escaped: `&lt;script&gt;alert('XSS')&lt;/script&gt;`

**Test — XSS in registration:**
1. Go to **Register** page
2. Enter name: `<img src=x onerror=alert('XSS')>`
3. Complete registration with valid email and password
4. **Login** → Check the navbar
5. **Verify** the name is displayed as plain text, no image error or alert

### 16e. CSRF Protection

**Test — CSRF token present:**
1. Go to any page with a form (Login, Register, Create Trip)
2. Right-click → **View Page Source**
3. Search for `_token`
4. **Verify** every `<form>` contains a hidden input: `<input type="hidden" name="_token" value="...">`

**Test — CSRF token validation:**
1. Open browser Developer Tools → Network tab
2. Submit a form (e.g., login)
3. Inspect the request → **Verify** `_token` is included in the POST data
4. (Advanced) Use a tool like Postman to send a POST request WITHOUT a CSRF token
5. **Verify** Laravel returns a **419 Page Expired** error

---

## Quick Test Checklist

| # | Requirement | Test Action | Expected Result | Pass? |
|---|---|---|---|---|
| 1 | Permissions defined | Check `permissions` table | 3 rows: create_trip, book_trip, cancel_trip | ☐ |
| 2 | Roles defined | Check `roles` table | 3 rows: transport_admin, driver, passenger | ☐ |
| 3 | Driver → create trips + view passengers | Login as driver, check Trips page | Create Trip button + Passengers column visible | ☐ |
| 4 | Passenger → book + cancel | Login as passenger, check Trips + My Bookings | Book + Cancel buttons visible | ☐ |
| 5 | Auto passenger role | Register new user, check navbar | "passenger" badge shown | ☐ |
| 6 | Admin-only role assignment | Login as admin vs passenger, try /admin/users | Admin: OK, Passenger: 403 | ☐ |
| 7 | create_trip works | Driver creates a trip | Trip appears in list | ☐ |
| 8 | book_trip works | Passenger books a trip | Booking appears in My Bookings | ☐ |
| 9 | cancel_trip works | Passenger cancels booking | Status → Cancelled | ☐ |
| 10 | Own bookings only | Check My Bookings with 2 passengers | Each sees only their own | ☐ |
| 11 | Request Refund button | Check cancelled booking | "Request Refund" button visible | ☐ |
| 12 | Refund status visible | Login as driver, check /refunds | Status badges visible | ☐ |
| 13 | Approve → closed | Driver approves refund | Status → Closed | ☐ |
| 14 | Transport Root CA | Check ssl/certs/ + inspect cert | CN = Transport Root | ☐ |
| 15 | www.secure-transport.com cert | Inspect + verify chain | CN = www.secure-transport.com, 365 days, chain OK | ☐ |
| 16 | Security audit | Test SQLi, XSS, weak passwords, auth | All attacks blocked | ☐ |
