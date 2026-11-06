# DineLeb: Lebanon Restaurant Review & Dining Feedback Platform

**Internship:** SQL Server, C# and OOP, ASP.NET Core Web API, Onion Architecture, MVC (.NET 10)  
**Developer:** Rawad Thebian  
**Submission Date:** November 6, 2026  
**Presentation Date:** November 14, 2026  

---

## 1. Project Overview
DineLeb is a localized dining review and guest feedback platform built specifically for diners and restaurant owners across Lebanon. It enables diners to explore restaurants by Lebanese regions/cities, read multi-factor reviews (Food, Service, Ambiance, Value), and file private operational complaints. It provides restaurant owners with an operational portal to claim venues, manage private complaints, and publicly reply to customer reviews.

---

## 2. Solution Architecture (Onion Architecture)
The solution strictly follows Onion Architecture with inward-pointing dependencies across 5 distinct projects:

1. **`DineLeb.Domain`**: Core domain entities, enums, custom domain exceptions, value objects, and repository interfaces. Has zero dependencies on any framework.
2. **`DineLeb.Application`**: Use cases, business logic, DTOs, service interfaces, FluentValidation rules, and scoring strategy implementations. References `Domain`.
3. **`DineLeb.Infrastructure`**: EF Core `ApplicationDbContext`, entity configurations (Fluent API), SQL Server migrations, identity store, and repository implementations. References `Application` and `Domain`.
4. **`DineLeb.WebApi`**: ASP.NET Core Web API controllers, JWT authentication, Swagger OpenAPI documentation, route constraints, and global exception handling middleware.
5. **`DineLeb.WebMvc`**: ASP.NET Core MVC presentation layer consuming the Web API via typed `HttpClient` instances.

---

## 3. Database Schema (SQL Server)
The data layer utilizes Entity Framework Core (.NET 10) Code-First with migrations and seeded data:

* **`ApplicationUser` (`AspNetUsers`)**: Inherits from `IdentityUser` with full name, role (`Admin`, `RestaurantOwner`, `Diner`), and registration timestamps.
* **`Cities`**: `CityId` (PK), `Name` (Unique seeded Lebanese cities: Beirut, Byblos, Tripoli, Batroun, Zahle, Saida, etc.).
* **`Cuisines`**: `CuisineId` (PK), `Name` (Unique seeded categories: Traditional Lebanese, Seafood, Burgers, Italian, Armenian, etc.).
* **`Restaurants`**: `RestaurantId` (PK), `Name`, `Description`, `Address`, `CityId` (FK), `CuisineId` (FK), `OwnerId` (FK to `AspNetUsers`, nullable until claimed), `AverageRating`, `TotalReviewsCount`, `CreatedAt`.
* **`ClaimRequests`**: `ClaimId` (PK), `RestaurantId` (FK), `UserId` (FK), `BusinessRegistrationNumber`, `Status` (`Pending`, `Approved`, `Rejected`), `AdminNotes`, `CreatedAt`, `ProcessedAt`.
* **`Reviews`**: `ReviewId` (PK), `RestaurantId` (FK), `UserId` (FK), `FoodRating`, `ServiceRating`, `AmbianceRating`, `ValueRating`, `OverallRating`, `Comment`, `VisitDate`, `CreatedAt`. *(Includes a composite unique index on `[RestaurantId, UserId]` allowing one review per diner).*
* **`ReviewReplies`**: `ReplyId` (PK), `ReviewId` (FK unique - 1:1), `OwnerId` (FK), `ReplyText`, `CreatedAt`.
* **`FeedbackTickets`**: `TicketId` (PK), `RestaurantId` (FK), `UserId` (FK), `Category` (Enum: Hygiene, Billing, Service, FoodQuality), `Description`, `Status` (Enum: `Open`, `InProgress`, `Resolved`, `Rejected`), `OwnerResponse`, `CreatedAt`, `ResolvedAt`.
* **`RestaurantBookmarks`**: `BookmarkId` (PK), `RestaurantId` (FK), `UserId` (FK), `CreatedAt`. *(Includes a composite unique index on `[RestaurantId, UserId]` allowing one bookmark per diner).*
* **`LeaderboardSnapshots`**: `SnapshotId` (PK), `RestaurantId` (FK), `CityId` (FK), `CuisineId` (FK), `RankScore`, `CalculatedAt`.

---

## 4. Key Object-Oriented & Design Patterns
* **Encapsulation:** Entities protect internal state. Rating calculations and review aggregations are encapsulated in domain service methods executed within database transactions.
* **Polymorphism via Strategy Pattern:** Leaderboard scoring uses `IScoringStrategy` with implementations:
  * `BayesianWeightedScoringStrategy`: Computes weighted rankings based on review counts and platform priors.
  * `ComplaintPenalizedScoringStrategy`: Penalizes venue leaderboard visibility according to active unresolved complaints:
    $$\text{FinalScore} = \text{BaseScore} - (\text{OpenComplaintsCount} \times 0.25)$$
* **Repository & Unit of Work:** Generic repositories paired with specific extensions (`IRestaurantRepository`, `IReviewRepository`) coordinated by `IUnitOfWork` for transactional consistency.

---

## 5. Web API Specifications (Key Endpoints)

### Authentication
* `POST /api/auth/register` - Register Diner or Restaurant Owner
* `POST /api/auth/login` - Authenticate and receive JWT token

### Restaurants & Discovery
* `GET /api/restaurants` - Filter by City, Cuisine, Price, and Minimum Rating
* `GET /api/restaurants/{id:int}` - Detailed restaurant profile (restricted to integer IDs)
* `POST /api/restaurants` - Register venue (Admin / Pending Owner)
* `PUT /api/restaurants/{id:int}` - Update venue profile

### Venue Ownership & Verification
* `POST /api/restaurants/{id:int}/claim` - Submit claim request with `BusinessRegistrationNumber`
* `GET /api/admin/claims` - List pending claims (Admin only)
* `PUT /api/admin/claims/{id:int}/status` - Approve or reject claim; automatically assigns `Restaurant.OwnerId` upon approval

### Reviews & Owner Replies
* `GET /api/restaurants/{id:int}/reviews` - List published diner reviews
* `POST /api/reviews` - Submit review (1 per diner per venue constraint)
* `POST /api/reviews/{id:int}/reply` - Official restaurant owner response to a review

### Feedback & Complaints Channel
* `POST /api/feedback-tickets` - Submit private operational complaint
* `GET /api/owner/complaints` - View complaints filed for the authenticated owner's venue
* `PUT /api/owner/complaints/{id:int}/status` - Update ticket status (`InProgress`, `Resolved`, `Rejected`), submit `OwnerResponse`, and record `ResolvedAt`

### Diner Activity
* `POST /api/restaurants/{id:int}/bookmarks` - Toggle bookmark
* `GET /api/me/bookmarks` - List authenticated diner's saved venues
* `GET /api/me/complaints` - List authenticated diner's submitted complaints with owner responses

### Analytics & Leaderboard
* `GET /api/analytics/regional-performance` - Aggregated multi-criteria ratings and average resolution hours per city
* `GET /api/leaderboard/trending` - Current top-ranked snapshot

---

## 6. MVC Front-End Views
1. `Account/Login.cshtml` & `Account/Register.cshtml` - User onboarding and role selection.
2. `Home/Index.cshtml` - Regional explorer and trending leaderboard snapshot.
3. `Restaurants/Index.cshtml` - Search catalog with Lebanese city and cuisine dropdown filters.
4. `Restaurants/Details.cshtml` - Profile, photo gallery, score breakdowns, reviews, and owner reply threads.
5. `Reviews/Create.cshtml` - Multi-criteria rating submission form.
6. `Complaints/Create.cshtml` - Private complaint filing form.
7. `Diner/MyActivity.cshtml` - Saved bookmarks and filed complaint resolution tracking.
8. `Owner/Complaints.cshtml` - Owner dashboard for complaint management and reply actions.
9. `Admin/Claims.cshtml` - Admin moderation portal for business claim approvals.
10. `Analytics/Index.cshtml` - Visual analytics covering regional ratings and resolution performance.

---

## 7. Execution Timeline (07/11/2026 – 14/11/2026)
* **Day 1 (Nov 7):** Solution scaffolding (Onion Architecture), Domain entities, Fluent API configurations, and SQL Server initial migrations with Lebanese cities/cuisines seed data.
* **Day 2 (Nov 8):** Generic Repository, Unit of Work, ASP.NET Core Identity setup, JWT Bearer configuration, and Auth API endpoints.
* **Day 3 (Nov 9):** Application services, DTO mappings, FluentValidation, and implementation of `IScoringStrategy`.
* **Day 4 (Nov 10):** Core API endpoints (Restaurants, Reviews, Owner Replies, Bookmarks) verified via Swagger and Postman.
* **Day 5 (Nov 11):** Complaint ticketing endpoints, Admin claim approval pipeline, Diner activity routes, and Regional Analytics query.
* **Day 6 (Nov 12):** MVC Front-End Part 1: Typed `HttpClient` service, shared `_Layout`, Auth views, Restaurant catalog, and Review submission.
* **Day 7 (Nov 13):** MVC Front-End Part 2: Diner My Activity view, Complaint filing, Owner dashboard, Admin claim portal, and Analytics dashboard.
* **Day 8 (Nov 14):** End-to-end integration testing, error handling validation, commit history review, and live demo rehearsal.
