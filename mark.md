# Addis Temari: Codebase Guide & Technical Implementation

## 1. Project Overview

### **Brief**

Addis Temari is a full-stack web application that empowers students to discover, compare, and manage information about universities worldwide. It provides a personalized dashboard for users to track their application journey, supports a subscription model for premium features, and includes an admin panel for efficient data management. The system is designed for scalability, maintainability, and a seamless user experience.

### **Architecture Diagram**

```mermaid
graph TD
    A[User's Browser] --> B{Frontend (React)};
    B -- REST API --> C{Backend (Django REST API)};
    C -- ORM --> D[Database (PostgreSQL/SQLite)];
    C -- Payment API --> E[Chapa Payment Gateway];
    E -- Webhook --> C;
```

**Tech Stack Summary:**

| Layer      | Technology                                  | Notes                                                                        |
| ---------- | ------------------------------------------- | ---------------------------------------------------------------------------- |
| Frontend   | React, Vite, Tailwind CSS, Radix UI, Sonner | Modern SPA, utility-first CSS, accessible UI primitives, toast notifications |
| State Mgmt | React Context API                           | Handles auth, user, and UI state                                             |
| Routing    | React Router                                | Client-side routing                                                          |
| HTTP       | Fetch API                                   | API calls, JWT in headers                                                    |
| Backend    | Django, Django REST Framework               | RESTful API, authentication, admin logic                                     |
| Auth       | SimpleJWT (JWT)                             | Secure, stateless authentication                                             |
| Database   | SQLite (dev), PostgreSQL (prod)             | Relational, scalable                                                         |
| ORM        | Django ORM                                  | Model management, migrations                                                 |
| Payments   | Chapa API                                   | Subscription payments                                                        |

---

## 2. System Architecture & Deep Dive

### 2.1. Frontend Architecture (`/frontend`)

**Framework & Rationale:**  
React was chosen for its component-based architecture, rich ecosystem, and strong community support. This enables rapid development, code reuse, and easy state management.

**Key Libraries & Purposes:**

- **Radix UI:** Provides accessible, unstyled component primitives, allowing for custom design with accessibility out-of-the-box. Chosen over alternatives for its flexibility and accessibility guarantees.
- **Tailwind CSS:** Utility-first CSS framework that accelerates development and enforces design consistency. Reduces the need for custom CSS and makes responsive design trivial.
- **State Management:**
  - **React Context API** is used for global state (auth, user, subscription).
  - Example: `useAuth` context provides `authTokens`, `logoutUser`, and user info to all components. The `updateUserProfile` function within the context ensures that profile changes are reflected globally.
  - University data and UI state are managed locally in components or via context if shared.
- **Routing:**
  - **React Router** manages client-side navigation.
  - Main routes:
    - `/` (Home/University List)
    - `/university/:id` (University Detail)
    - `/dashboard` (User Dashboard)
    - `/admin` (Admin Panel)
    - `/privacy` (Privacy Policy)
    - `/terms` (Terms of Service)
- **HTTP Client:**
  - **Fetch API** is used for API calls.
  - JWT tokens are attached to the `Authorization` header.
  - API calls are wrapped in async functions, with error handling and token refresh logic as needed.

**Critical Component Breakdown:**

1. **UniversityList**

   - **Props:** None (fetches data internally)
   - **State:** List of universities, filters, loading/error state
   - **Key Functions:** Fetch universities, apply filters, handle pagination
   - **Backend Interaction:** Calls `/api/universities/` with query params

2. **UniversityDetail**

   - **Props:** `universityId` (from route)
   - **State:** University data, loading/error state
   - **Key Functions:** Fetch university details, add to user lists
   - **Backend Interaction:** Calls `/api/universities/:id`

3. **UserDashboard**

   - **Props:** None (uses context)
   - **State:** User lists (favorites, applied, etc.), subscription status
   - **Key Functions:** Move universities between lists, renew subscription
   - **Backend Interaction:** Calls `/api/dashboard/`, `/api/initialize-payment/`

4. **AdminUniversityPage**

   - **Props:** None
   - **State:** Universities, bulk upload state, dialog state
   - **Key Functions:** CRUD operations, bulk upload, sorting
   - **Backend Interaction:** Calls `/api/universities/`, `/api/universities/bulk_create/`, etc.

5. **SubscriptionModal**

   - **Props:** Open state, callbacks
   - **State:** Payment status, error state
   - **Key Functions:** Initiate payment, handle redirect
   - **Backend Interaction:** Calls `/api/initialize-payment/`

6. **ProfileModal**

   - **Props:** `isOpen`, `onOpenChange`
   - **State:** `profile`, `loading`, `isEditing`
   - **Key Functions:** Fetch user profile, handle profile updates (including image upload).
   - **Backend Interaction:** `GET` and `PATCH` to `/api/profile/`. Handles `multipart/form-data` for file uploads. On success, it calls `updateUserProfile` to sync the live profile state and `refreshAuthTokens` to update the JWT with the new `profile_picture` URL.

7. **Shared Components (Footer)**

   - A consistent `Footer` component is used across user-facing pages (`Dashboard`, `UniversityList`, `UniversityDetail`, etc.) to provide important links like the Privacy Policy and Terms of Service. This component is defined in `frontend/src/components/Footer.jsx`.

8. **Utilities (`/lib`)**
   - **`color.js`**: Contains a `getColorFromString` utility that generates a consistent, unique color from a user's username. This is used to create colored avatar fallbacks, similar to Telegram's UI.

---

### 2.2. Backend Architecture (`/backend`)

**Framework & Structure:**  
Django REST Framework is used for its robust, batteries-included approach. The project follows a modular structure with apps for `universities`, `profiles`, and `contacts`. The codebase uses a Model-View-Serializer pattern, with custom permissions and authentication.

**API Endpoint Map:**

| Method | Endpoint                         | Purpose                      | Authentication | Authorization |
| ------ | -------------------------------- | ---------------------------- | -------------- | ------------- |
| GET    | `/api/universities/`             | Search & filter universities | JWT (optional) | Subscription  |
| GET    | `/api/universities/:id/`         | Get university details       | JWT (optional) | Subscription  |
| POST   | `/api/auth/login/`               | User login                   | -              | -             |
| GET    | `/api/dashboard/`                | Get user dashboard           | Required (JWT) | User          |
| POST   | `/api/dashboard/`                | Add uni to a user list       | Required (JWT) | User          |
| DELETE | `/api/dashboard/`                | Remove uni from a user list  | Required (JWT) | User          |
| POST   | `/api/universities/create/`      | Create a university          | Required (JWT) | Admin         |
| POST   | `/api/universities/bulk_create/` | Bulk create universities     | Required (JWT) | Admin         |
| DELETE | `/api/universities/:id/delete/`  | Delete a university          | Required (JWT) | Admin         |
| POST   | `/api/initialize-payment/`       | Initiate Chapa payment       | Required (JWT) | User          |
| POST   | `/api/chapa-webhook/`            | Handle payment confirmation  | Chapa IPN      | -             |

**Authentication & Authorization Flow:**

- **JWT Authentication:**
  - On login, the backend issues an access and refresh token.
  - The frontend stores the tokens and attaches the access token to all API requests.
  - Middleware checks the token, decodes user info, and attaches it to the request.
- **Role-Based Access:**
  - Custom permissions (`IsAdminUser`, `HasActiveSubscription`) restrict access to sensitive endpoints.
  - Example middleware logic:
    ```python
    class HasActiveSubscription(BasePermission):
        def has_permission(self, request, view):
            if not request.user or not request.user.is_authenticated:
                return False
            if request.user.is_staff:
                return True
            dashboard = request.user.dashboard
            return dashboard.subscription_status == 'active' and dashboard.subscription_end_date >= timezone.now().date()
    ```

---

### 2.3. Database Schema & ORM

**ORM:**  
Django ORM is used for its tight integration with Django, migrations, and admin interface. (If using Prisma, the rationale would be type-safety and auto-generated types.)

**Schema Analysis:**

- **User**

  - Fields: `id`, `username`, `email`, `password`, `role` (user/admin), etc.
  - Relationships: One-to-one with `Profile`, one-to-one with `UserDashboard`

- **University**

  - Fields: `id`, `name`, `country`, `city`, `application_fee`, `tuition_fee`, `intakes` (JSONField), `scholarships` (JSONField), `university_link`, `application_link`, `description`, etc.

- **UserDashboard**

  - Fields: `id`, `user_id`, `subscription_status`, `subscription_end_date`

- **Profile**
  - Fields: `id`, `user_id`, `bio`, `phone_number`, `profile_picture`, `preferred_intakes`

**Relationships:**

- User `hasOne` Profile
- User `hasOne` UserDashboard
- UserDashboard `hasMany` Universities (via M2M fields: `favorites`, `planning_to_apply`, `applied`, `accepted`, `visa_approved`)
- University can be in many dashboards/lists

**Performance Notes:**

- Consider adding indexes on `University.name` and `country` for faster search. A GIN index on the `intakes` JSONField could also improve performance for intake-based searches in PostgreSQL.
- For large-scale, move to PostgreSQL and use full-text search for advanced queries.

---

## 3. Key Features: Implementation Logic

### Feature 1: University Search & Filtering

- **Backend Logic:**
  - Uses DjangoFilterBackend and DRF filters.
  - Queryset is dynamically filtered based on query params (country, city, degree_level, etc.).
  - Example:
    ```python
    # In universities/views.py
    # Custom filter for intakes JSONField
    intake_query = self.request.query_params.get('intakes')
    if intake_query:
        # Correctly query the JSON array for an object with a matching 'name' key.
        # This is more accurate than a simple substring search.
        # Note: This works best with PostgreSQL's JSONB field.
        queryset = queryset.filter(intakes__contains=[{'name': intake_query}])
    ```
- **Frontend Logic:**
  - Filter state is managed in component state.
  - Debouncing is used to avoid excessive API calls (e.g., with `setTimeout` or a debounce hook).
  - API calls are made with current filter values as query params.

### Feature 2: User University Lists

- **Database Design:**
  - Each list (favorites, applied, etc.) is a M2M field on `UserDashboard`.
  - This avoids a separate junction table but is functionally similar to a `UserUniversityList` with a `listType` enum.
- **API Interaction:**
  - Frontend sends a POST/DELETE to `/api/dashboard/` with `university_id` and `list_name`.
  - Backend adds or removes the university from the appropriate M2M field and returns the updated dashboard.

### Feature 3: Subscription & Payment (Chapa)

- **Payment Flow:**
  1. User clicks "Renew" or "Subscribe".
  2. Frontend calls `/api/initialize-payment/`.
  3. Backend creates a Chapa payment reference and returns a checkout URL.
  4. User is redirected to Chapa for payment.
  5. Chapa calls the backend webhook on payment completion.
  6. Backend verifies the webhook, updates the user's subscription status and expiry.
- **Status Management:**
  - Protected routes check `dashboard.subscription_status === 'active'` and expiry date.
  - Custom permission classes enforce this on the backend.

### Feature 4: Admin Panel & Bulk Upload

- **Bulk Upload Logic:**
  - Admin can upload a JSON file or paste JSON text.
  - Backend parses the file/text, validates each university, and uses `bulk_create` for efficiency.
  - Errors are collected and returned for invalid entries; successful entries are added to the database.

---

## 4. Development Guide

**Local Setup:**

1. Clone the repo:  
   `git clone <repo-url>`
2. Install dependencies:
   - Backend: `cd backend && pip install -r requirements.txt`
   - Frontend: `cd frontend && npm install`
3. Set up environment variables:
   - Backend: Copy `.env.example` to `.env` and fill in secrets (DB, JWT, Chapa keys)
4. Run migrations:  
   `python manage.py migrate`
5. Start servers:
   - Backend: `python manage.py runserver`
   - Frontend: `npm run dev`
6. (Optional) Seed database:  
   `python manage.py loaddata initial_data.json`

**Scripts:**

| Location | Script                             | Purpose                       |
| -------- | ---------------------------------- | ----------------------------- |
| frontend | `npm run dev`                      | Start frontend dev server     |
| frontend | `npm run build`                    | Build frontend for production |
| backend  | `python manage.py runserver`       | Start backend server          |
| backend  | `python manage.py migrate`         | Run DB migrations             |
| backend  | `python manage.py createsuperuser` | Create admin user             |

---

## 5. Future Implementation Notes & Roadmap

**Potential Optimizations:**

- Use Redis for caching university search results to reduce DB load.
- Integrate Elasticsearch for advanced, typo-tolerant search.
- Move to a microservices architecture if scaling demands (separate user, university, and payment services).

**Feature Ideas:**

- **Email Notifications:**
  - Requires a queue system (e.g., Celery, BullMQ) for async delivery.
- **University Reviews:**
  - Add a `Review` model, moderation tools, and user feedback features.
- **Advanced Analytics:**
  - Track user engagement, most-viewed universities, etc.

**Code Quality:**

- Add end-to-end tests with Cypress.
- Use Swagger/OpenAPI for API documentation.
- Enforce code linting and formatting (ESLint, Prettier, Black).

---

_Document generated for the Addis Temari codebase. Last updated: September 23, 2025._

# Project Folder & File Structure

Below is the actual folder and file structure of the UNI-FINDER-GIT project as of September 27, 2025:

```
UNI-FINDER-GIT/
│
├── backend/
│   ├── build.sh
│   ├── db.sqlite3
│   ├── manage.py
│   ├── requirements.txt
│   ├── contacts/
│   │   ├── migrations/
│   │   │   ├── __init__.py
│   │   │   └── 0001_initial.py
│   │   ├── __init__.py
│   │   ├── admin.py
│   │   ├── apps.py
│   │   ├── models.py
│   │   ├── serializers.py
│   │   ├── tests.py
│   │   └── views.py
│   ├── media/
│   │   └── profile_pics/
│   │       ├── bag.jpg
│   │       ├── portrait_2.jpg
│   │       ├── portrait_2_ehImDDO.jpg
│   │       └── portrait_2_JbtJIBY.jpg
│   ├── notifications/
│   │   ├── migrations/
│   │   │   └── __init__.py
│   │   ├── __init__.py
│   │   ├── admin.py
│   │   ├── apps.py
│   │   ├── models.py
│   │   ├── tests.py
│   │   └── views.py
│   ├── profiles/
│   │   ├── migrations/
│   │   │   ├── __init__.py
│   │   │   ├── 0001_initial.py
│   │   │   └── 0002_profile_preferred_intakes.py
│   │   ├── __init__.py
│   │   ├── admin.py
│   │   ├── apps.py
│   │   ├── models.py
│   │   ├── serializers.py
│   │   ├── signals.py
│   │   ├── tasks.py
│   │   ├── tests.py
│   │   ├── urls.py
│   │   └── views.py
│   ├── universities/
│   │   ├── migrations/
│   │   │   ├── __init__.py
│   │   │   ├── 0001_initial.py
│   │   │   ├── 0002_university_degree_level.py
│   │   │   ├── 0003_rename_website_university_university_link_and_more.py
│   │   │   ├── 0004_university_course_offered.py
│   │   │   ├── 0005_userdashboard.py
│   │   │   ├── 0006_userdashboard_subscription_end_date_and_more.py
│   │   │   ├── 0007_userdashboard_phone_number.py
│   │   │   ├── 0008_remove_userdashboard_phone_number.py
│   │   │   ├── 0009_university_degree_level_university_intake_months.py
│   │   │   ├── 0010_remove_university_deadline_grad_and_more.py
│   │   │   └── 0011_universityjsonimport_remove_university_degree_level.py
│   │   ├── __init__.py
│   │   ├── admin.py
│   │   ├── apps.py
│   │   ├── celery.py
│   │   ├── models.py
│   │   ├── permissions.py
│   │   ├── serializers.py
│   │   ├── tasks.py
│   │   ├── tests.py
│   │   ├── urls.py
│   │   └── views.py
│   └── university_api/
│       ├── __init__.py
│       ├── asgi.py
│       ├── settings.py
│       ├── urls.py
│       └── wsgi.py
│
├── docs/
│
├── frontend/
│   ├── .gitignore
│   ├── components.json
│   ├── eslint.config.js
│   ├── index.html
│   ├── jsconfig.json
│   ├── marquee.jsx
│   ├── package.json
│   ├── postcss.config.js
│   ├── README.md
│   ├── tailwind.config.js
│   ├── vercel.json
│   ├── vite.config.js
│   ├── public/
│   │   └── vite.svg
│   └── src/
│       ├── apiConfig.js
│       ├── App.jsx
│       ├── index.css
│       ├── main.jsx
│       ├── Navbar.jsx
│       ├── assets/
│       │   ├── home.jpg
│       │   ├── n.svg
│       │   ├── react.svg
│       │   └── user-avatar.jpg
│       ├── components/
│       │   ├── Footer.jsx
│       │   ├── LoadingSpinner.jsx
│       │   ├── ProfileModal.jsx
│       │   ├── Sidebar.jsx
│       │   ├── UserAvatar.jsx
│       │   ├── UserNav.jsx
│       │   └── ui/
│       │       ├── accordion.jsx
│       │       ├── avatar.jsx
│       │       ├── badge.jsx
│       │       ├── button.jsx
│       │       ├── card.jsx
│       │       ├── checkbox.jsx
│       │       ├── dialog.jsx
│       │       ├── dropdown-menu.jsx
│       │       ├── input.jsx
│       │       ├── label.jsx
│       │       ├── marquee.jsx
│       │       ├── select.jsx
│       │       ├── sheet.jsx
│       │       ├── switch.jsx
│       │       ├── table.jsx
│       │       ├── tabs.jsx
│       │       └── textarea.jsx
│       ├── context/
│       │   └── context.jsx
│       ├── lib/
│       │   ├── color.js
│       │   └── utils.js
│       ├── pages/
│       │   ├── AdminContacts.jsx
│       │   ├��─ AdminUniversityEditPage.jsx
│       │   ├── AdminUniversityPage.jsx
│       │   ├── ContactPage.jsx
│       │   ├── DashboardPage.jsx
│       │   ├── HomePage.jsx
│       │   ├── LoginPage.jsx
│       │   ├── PrivacyPolicyPage.jsx
│       │   ├── RegisterPage.jsx
│       │   ├── TermsOfServicePage.jsx
│       │   ├── UniversityDetail.jsx
│       │   ├── UniversityDetailPage.jsx
│       │   ├── UniversityList.jsx
│       │   ├── UserManagement.jsx
│       │   ├── serializers.py
│       │   └── spinner.css
│       └── utils/
│           └── ProtectedRoute.jsx
│
├── scripts/
│
├── apps.py
├── mark.md
├── signals.py
└── tasks.py
```

> This structure is synchronized with the current workspace. If you add, remove, or rename files/folders, update this section accordingly.