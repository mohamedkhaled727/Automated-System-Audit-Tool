# Comprehensive WebSec Final Exam Validation Guide

This document is a highly detailed, step-by-step manual designed to prove that your project meets every single requirement of the Final Exam. You can use this guide to test the application yourself or to present your work to your instructor.

---

## Part 1: Step-by-Step Verification of Requirements (1-13)

### Requirement 1 & 2: Define Roles and Permissions
**Criteria:** Define permissions (`upload_content`, `enroll_course`, `submit_assignment`) and three roles (`course_admin`, `instructor`, `student`).
* **Where to find it:** `database/seeders/RolePermissionSeeder.php`
* **How to test it:** 
  1. Open your database GUI (like phpMyAdmin).
  2. Look at the `roles` table. You will see `course_admin`, `instructor`, and `student`.
  3. Look at the `permissions` table. You will see the three permissions.

### Requirement 3: Instructor Permissions
**Criteria:** Instructor has permission to `upload_content` and view student submissions.
* **Where to find it:** 
  * Permission: `RolePermissionSeeder.php` (Line 34: `$instructorRole->syncPermissions(['upload_content']);`)
  * View Submissions: `app/Http/Controllers/SubmissionController.php` (Line 17). If the user is NOT a student, the code queries `Submission::with('assignment.course', 'user')->get()`, retrieving all submissions.
* **How to test it:** Log in as an instructor. Go to the "Courses" tab. You will see the "Create New Course" button (which requires `upload_content`). Go to the "Submissions" tab, and you will see submissions from *all* students, not just your own.

### Requirement 4 & 5: Student Role, Permissions, and Default Assignment
**Criteria:** Student can `enroll_course` and `submit_assignment`. New users take the `student` role by default.
* **Where to find it:** `app/Listeners/AssignDefaultRole.php` and `RolePermissionSeeder.php`.
* **How to test it:**
  1. Go to the homepage and click **Register**.
  2. Create a brand new account (e.g., `teststudent@example.com`).
  3. Immediately upon login, go to the "Courses" tab. You will see "Enroll" buttons next to available courses, proving the system automatically granted you the `student` role and the `enroll_course` permission.

### Requirement 6: Admin Role Assignment
**Criteria:** Only an admin can assign the `instructor` or `course_admin` roles.
* **Where to find it:** `resources/views/dashboard.blade.php` and `app/Http/Controllers/AdminController.php`.
* **How to test it:**
  1. Log in with the default admin account (`admin@secure-study.com` / `Admin123!@#`).
  2. Go to the **Dashboard**.
  3. You will see an **Admin Panel: Role Management** table listing all registered users.
  4. Select "Instructor" from the dropdown next to a user and click "Assign".
  5. *Security Check:* If a regular student tries to post data to the assign role endpoint, `AdminController.php` line 11 explicitly aborts with a `403 Forbidden` error.

### Requirement 7, 8, & 9: Feature Implementation tied to Permissions
**Criteria:** Upload content, Enroll in courses, Submit assignments.
* **Where to find it:** `app/Http/Controllers/CourseController.php` and `app/Http/Controllers/SubmissionController.php`.
* **How to test it:** 
  * Every critical controller method starts with Laravel's authorization check: `$this->authorize('permission_name');`.
  * If a user without `upload_content` tries to forcibly navigate to `http://localhost/FinalExam/public/courses/create`, Laravel's core security will block them with a `403 Unauthorized` page.

### Requirement 10: Student Privacy Boundary
**Criteria:** Students can only view and manage their own enrollments and submissions.
* **Where to find it:** `app/Http/Controllers/SubmissionController.php`
* **How it works:**
  ```php
  if ($user->hasRole('student')) {
      // The ORM strictly locks the query to only the authenticated user's ID
      $submissions = $user->submissions()->with('assignment.course')->get();
  }
  ```

### Requirement 11, 12, & 13: The Grade Review Workflow
**Criteria:** Request Review button, Status visibility, and Instructor completion.
* **Where to find it:** `resources/views/submissions/index.blade.php` and `app/Http/Controllers/SubmissionController.php`.
* **How to test it:**
  1. **Student Side:** Log in as a student, submit a file link for an assignment. A blue **"Request Grade Review"** button appears. Click it. The status badge turns yellow (`Requested`). The button disables so you can't spam it.
  2. **Instructor Side:** Log in as an Instructor or Admin. Go to Submissions. You will see that student's submission with a yellow `Requested` badge. Beside it is a green **"Mark Review Completed"** button. Click it. The badge turns green (`Completed`).

---

## Part 2: Security Vulnerabilities (Requirement 16)

The exam requires checking for common web security vulnerabilities. Here is a deep dive into how the three major vulnerabilities are neutralized in this project.

### 1. SQL Injection (SQLi)
* **The Vulnerability:** Attackers inject malicious SQL queries into input fields (like the login email) to dump the database, bypass passwords, or delete tables.
* **The Defense Mechanism:** **PDO Parameter Binding via Eloquent ORM.**
  * In `SubmissionController.php` and `CourseController.php`, we never write raw SQL strings like `SELECT * FROM users WHERE email = '$email'`.
  * Instead, we use Eloquent: `Submission::updateOrCreate(['user_id' => auth()->id()])`.
  * Eloquent uses PHP Data Objects (PDO) to prepare statements. The user input is treated strictly as a *string literal*, never as an executable database command.
* **How to Test it live:**
  1. Go to the login page.
  2. In the Email field, type: `' OR 1=1 --`
  3. Enter any password and click login.
  4. The system will fail to log you in. The database safely queries for literally exactly those characters instead of evaluating the `OR 1=1` logic.

### 2. Weak Passwords (Brute Force / Dictionary Attacks)
* **The Vulnerability:** Users create passwords like "123456" or "password", making it trivial for attackers to guess or crack their accounts.
* **The Defense Mechanism:** **Global Password Defaults.**
  * Open `app/Providers/AppServiceProvider.php`. Inside the `boot()` method, we defined:
    ```php
    Password::defaults(function () {
        return Password::min(8)->mixedCase()->numbers()->symbols();
    });
    ```
* **How to Test it live:**
  1. Go to the Registration page.
  2. Try to register with the password `secret123`.
  3. The form will reject the submission and display a red error stating the password must contain at least one uppercase letter and one special character.

### 3. Cross-Site Scripting (XSS)
* **The Vulnerability:** Attackers insert malicious JavaScript into text fields (like a Course Title or Assignment Description). When other users view that course, the script executes in their browser, potentially stealing their session cookies.
* **The Defense Mechanism:** **Blade HTML Entity Escaping.**
  * In `resources/views/courses/index.blade.php`, data is output using double curly braces: `{{ $course->title }}`.
  * Laravel automatically runs this through PHP's `htmlspecialchars()` function before rendering.
* **How to Test it live:**
  1. Log in as an instructor and create a new course.
  2. For the Course Title, enter exactly this text: `<script>alert("XSS Attack!");</script>`
  3. Save the course.
  4. Go to the Courses list. Instead of popping up a javascript alert box, the browser will display the literal text `<script>...` on the screen. The angle brackets `< >` were safely converted to `&lt;` and `&gt;` behind the scenes.

---

## Part 3: Full Project Architecture Mapping (Where is Where)

This structure map details exactly where the custom logic was injected into the Laravel framework.

### 🗂️ HTTP & Routing Logic
* **`routes/web.php`**
  * The central nervous system. It contains all the URL endpoints for the application (e.g., `/courses`, `/submissions/review`). It wraps all these endpoints inside the `auth` middleware so no unauthorized person can access them.
* **`app/Http/Controllers/`**
  * **`AdminController.php`**: Contains the `assignRole` method. Verifies if the user is a `course_admin` and uses Spatie to grant roles to other users.
  * **`CourseController.php`**: Handles fetching courses from the database, storing new courses (checks `upload_content`), and the logic for a student enrolling in a course.
  * **`AssignmentController.php`**: Handles the creation of new assignments under a specific course.
  * **`SubmissionController.php`**: The most complex controller. It separates data visibility (students see their own, instructors see all) and handles the database updates for the Grade Review request workflow.

### 🗂️ Database Layer
* **`database/migrations/`**
  * Contains the structural blueprints for the database tables.
  * `create_courses_table.php`, `create_enrollments_table.php`, `create_assignments_table.php`, `create_submissions_table.php`.
* **`database/seeders/RolePermissionSeeder.php`**
  * The script that initializes the database with the core security requirements (Permissions, Roles, and the master Admin user).
* **`app/Models/`**
  * **`User.php`**: Upgraded with Spatie's `HasRoles` trait. Represents the authenticated person.
  * **`Course.php`, `Enrollment.php`, `Assignment.php`, `Submission.php`**: The Eloquent representations of the database tables, allowing us to build easy relationships (e.g., `$course->assignments`).

### 🗂️ Frontend Views (Blade)
* **`resources/views/welcome.blade.php`**
  * The custom, premium dark-mode landing page featuring glassmorphism and animated background blobs.
* **`resources/views/layouts/guest.blade.php`**
  * The wrapper layout for the Login and Register pages. Heavily customized with CSS to match the premium dark mode aesthetic.
* **`resources/views/dashboard.blade.php`**
  * The landing page after logging in. For `course_admin` users, this file contains the table UI for assigning roles to other users.
* **`resources/views/courses/index.blade.php`**
  * Displays the list of all courses, their respective assignments, and dynamic buttons based on Spatie permissions (e.g., Students see "Enroll" and "Submit", Instructors see "Create Course" and "Add Assignment").
* **`resources/views/submissions/index.blade.php`**
  * The data table tracking assignment submissions and the color-coded UI for the Grade Review workflow.

### 🗂️ Security & Core System
* **`app/Providers/AppServiceProvider.php`**
  * Boots up global configurations. We use this file to register our custom Event Listeners and define our strict Password security rules.
* **`app/Listeners/AssignDefaultRole.php`**
  * An event listener that waits in the background. The exact second a user finishes registering, this listener catches the `Registered` event and injects the `student` role into their database profile.
* **`certificates/`**
  * The directory containing the output of your OpenSSL commands (`education_root.crt`, `secure-study.crt`, etc.).
