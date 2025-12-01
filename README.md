# Dome-Gym

A multi-solution .NET (7/8) workspace implementing a DDD-style gym platform. It is split into bounded contexts:

- GymManagement: manage gyms, rooms, and subscriptions
- SessionReservation: schedule sessions, list rooms, and create/cancel reservations
- SharedKernel: cross-context contracts (integration events, message broker abstractions)
- UserManagement: authentication and user profiles

## Prerequisites
- .NET SDK 8.0 (and optionally 7.0 for compatibility)
- IDE: VS Code or Visual Studio 2022+
- PowerShell 5.1+ (default shell)

## Folder Layout
- `GymManagement/` — API, Application, Domain, Infrastructure, tests, and sample HTTP requests
- `SessionReservation/` — API, Application, Domain, Infrastructure, tests, and sample HTTP requests
- `SharedKernel/` — shared contracts and messaging abstractions
- `UserManagement/` — API, Application, Domain, Infrastructure, tests, and sample HTTP requests

## Quick Start (PowerShell)
Restore all solutions:
```powershell
# From repo root
dotnet restore GymManagement/GymManagement.sln
dotnet restore SessionReservation/SessionReservation.sln
dotnet restore UserManagement/UserManagement.sln
dotnet restore SharedKernel/SharedKernel.csproj
```

Build:
```powershell
dotnet build GymManagement/GymManagement.sln -c Debug
dotnet build SessionReservation/SessionReservation.sln -c Debug
dotnet build UserManagement/UserManagement.sln -c Debug
dotnet build SharedKernel/SharedKernel.csproj -c Debug
```

Run APIs:
```powershell
# GymManagement API
dotnet run --project GymManagement/src/GymManagement.Api/GymManagement.Api.csproj

# SessionReservation API
dotnet run --project SessionReservation/src/SessionReservation.Api/SessionReservation.Api.csproj

# UserManagement API
dotnet run --project UserManagement/src/UserManagement.Api/UserManagement.Api.csproj
```

App settings:
- Each API has `appsettings.json` and `appsettings.Development.json`. Adjust connection strings, message broker settings, and ports as needed.

## HTTP Request Samples
Use the `.http` files under each `requests/` folder (VS Code REST Client or JetBrains HTTP Client) to exercise endpoints.

- `GymManagement/requests/`
  - Gyms: `CreateGym.http`, `ListGyms.http`
  - Rooms: `CreateRoom.http`, `DeleteRoom.http`
  - Subscriptions: `CreateSubscription.http`, `ListSubscriptions.http`
- `SessionReservation/requests/`
  - Gyms: `ListSessions.http`
  - Participants: `CreateReservation.http`, `CancelReservation.http`, `ListParticipantSessions.http`
  - Reservations: `CreateReservation.http`
  - Rooms: `GetRoom.http`, `ListRooms.http`
  - Sessions: `CreateSession.http`, `GetSession.http`
- `UserManagement/requests/`
  - Authentication: `Register.http`, `Login.http`
  - Profiles: `CreateProfile.http`, `ListProfiles.http`

## API Endpoints
Summaries inferred from the `.http` samples and typical DDD APIs. Actual routes may differ based on controllers and minimal APIs.

- GymManagement API
  - Subscriptions: `POST /subscriptions` (create), `GET /subscriptions` (list)
  - Gyms (scoped to subscription): `POST /subscriptions/{subscriptionId}/gyms` (create), `GET /subscriptions/{subscriptionId}/gyms` (list), `GET /subscriptions/{subscriptionId}/gyms/{gymId}` (get)
  - Gym Rooms: `POST /gyms/{gymId}/rooms` (create), `DELETE /gyms/{gymId}/rooms/{roomId}` (delete)

- SessionReservation API
  - Rooms (scoped to gym): `GET /gyms/{gymId}/rooms` (list), `GET /gyms/{gymId}/rooms/{roomId}` (get)
  - Sessions (scoped to room): `POST /rooms/{roomId}/sessions` (create), `GET /rooms/{roomId}/sessions/{sessionId}` (get)
  - Reservations (scoped to session): `POST /sessions/{sessionId}/reservations` (create)
  - Participants: `GET /participants/{participantId}/sessions` (list), `POST /participants/{participantId}/sessions/{sessionId}/reservation` (create), `DELETE /participants/{participantId}/sessions/{sessionId}/reservation` (cancel)

- UserManagement API
  - Authentication: `POST /Authentication/register`, `POST /Authentication/login`
  - Profiles (scoped to user): `POST /users/{userId}/profiles/admin`, `POST /users/{userId}/profiles/participant`, `POST /users/{userId}/profiles/trainer`, `GET /users/{userId}/profiles`

## Configuration & Environment
Each API uses `appsettings.json` and `appsettings.Development.json`. Common settings include:

- `ConnectionStrings`: database connections per bounded context
- `MessageBroker`: settings for publishing/consuming integration events via `SharedKernel/MessageBroker`
- `Logging`: log levels and sinks
- `AllowedHosts` and `urls`: server binding

Examples (override via command line or environment):
```powershell
# Override port when running
dotnet run --project GymManagement/src/GymManagement.Api/GymManagement.Api.csproj --urls http://localhost:5163

# Set environment (Windows PowerShell)
$env:ASPNETCORE_ENVIRONMENT = "Development"; 
$env:ConnectionStrings__Default = "Server=localhost;Database=DomeGym_Gym;Trusted_Connection=True;TrustServerCertificate=True"; 
# MessageBroker keys use HostName/UserName/Password/Port/QueueName/ExchangeName per appsettings.Development.json
$env:MessageBroker__HostName = "localhost"; $env:MessageBroker__UserName = "guest"; $env:MessageBroker__Password = "guest"; $env:MessageBroker__Port = "5672"; 

# Then run API
dotnet run --project GymManagement/src/GymManagement.Api/GymManagement.Api.csproj
```

Note: double underscores in env var names map to nested config keys (e.g., `MessageBroker__Host`).

## Swagger & API Exploration
- Swagger UI is enabled in `Development` for all APIs.
- After starting an API, navigate to `/swagger` on the base URL (e.g., `http://localhost:5163/swagger`).

## Auth Flow (UserManagement)
- Register: `POST /Authentication/register` with first/last name, email, password
- Login: `POST /Authentication/login` returns `AuthenticationResponse` with `token`
- Use Bearer token: include `Authorization: Bearer <token>` when calling protected endpoints, e.g., `POST /users/{userId}/profiles/*`

Tip: `userId` in profile routes must match the `id` claim in the token. Otherwise, API returns `403 Forbidden`.

## HTTPS Redirection
- APIs enable HTTPS redirection by default. If testing with plain `http`, either:
  - Use `--urls http://localhost:<port>` as shown above
  - Or configure Kestrel for local development to listen on HTTP in `appsettings.Development.json`

## Curl Examples
Replace base URLs with your local ports.

```powershell
# GymManagement: create subscription
curl -X POST "http://localhost:5163/subscriptions" ^
  -H "Content-Type: application/json" ^
  -d '{
    "subscriptionType": "Starter", // enum: Free|Starter|Pro
    "adminId": "00000000-0000-0000-0000-000000000001"
  }'

# GymManagement: list subscriptions
curl "http://localhost:5163/subscriptions"

# GymManagement: create gym under subscription
curl -X POST "http://localhost:5163/subscriptions/00000000-0000-0000-0000-000000000002/gyms" ^
  -H "Content-Type: application/json" ^
  -d '{ "name": "Downtown Gym" }'

# SessionReservation: list rooms in a gym
curl "http://localhost:5164/gyms/00000000-0000-0000-0000-000000000003/rooms"

# SessionReservation: create session in a room
curl -X POST "http://localhost:5164/rooms/00000000-0000-0000-0000-000000000004/sessions" ^
  -H "Content-Type: application/json" ^
  -d '{
    "name": "Morning Yoga",
    "description": "Beginner friendly",
    "maxParticipants": 20,
    "startDateTime": "2025-12-01T08:00:00Z",
    "endDateTime": "2025-12-01T09:00:00Z",
    "trainerId": "00000000-0000-0000-0000-000000000005",
    "categories": ["Yoga"]
  }'

# SessionReservation: participant reserves a session
curl -X POST "http://localhost:5164/participants/00000000-0000-0000-0000-000000000006/sessions/00000000-0000-0000-0000-000000000007/reservation"

# UserManagement: register and login
curl -X POST "http://localhost:5165/Authentication/register" ^
  -H "Content-Type: application/json" ^
  -d '{
    "firstName": "Jane",
    "lastName": "Doe",
    "email": "jane@example.com",
    "password": "P@ssw0rd!"
  }'

curl -X POST "http://localhost:5165/Authentication/login" ^
  -H "Content-Type: application/json" ^
  -d '{
    "email": "jane@example.com",
    "password": "P@ssw0rd!"
  }'

# UserManagement: create profiles (requires Authorization header with Bearer token)
curl -X POST "http://localhost:5165/users/00000000-0000-0000-0000-000000000008/profiles/participant" ^
  -H "Authorization: Bearer <token>"
```

## Sample DTOs
Reference shapes derived from `Contracts` projects.

- GymManagement
  - `CreateSubscriptionRequest`: `{ subscriptionType: "Free|Starter|Pro", adminId: "Guid" }`
  - `SubscriptionResponse`: `{ id: "Guid", subscriptionType: "Free|Starter|Pro" }`
  - `CreateGymRequest`: `{ name: "string" }`
  - `GymResponse`: `{ id: "Guid", name: "string" }`
  - `CreateRoomRequest`: `{ name: "string" }`
  - `RoomResponse`: `{ id: "Guid", name: "string" }`

- SessionReservation
  - `CreateSessionRequest`:
    ```json
    {
      "name": "string",
      "description": "string",
      "maxParticipants": 0,
      "startDateTime": "2025-12-01T08:00:00Z",
      "endDateTime": "2025-12-01T09:00:00Z",
      "trainerId": "Guid",
      "categories": ["string"]
    }
    ```
  - `SessionResponse`:
    ```json
    {
      "id": "Guid",
      "name": "string",
      "description": "string",
      "numParticipants": 0,
      "maxParticipants": 0,
      "startDateTime": "2025-12-01T08:00:00Z",
      "endDateTime": "2025-12-01T09:00:00Z",
      "categories": ["string"]
    }
    ```

- UserManagement
  - `RegisterRequest`: `{ firstName: "string", lastName: "string", email: "string", password: "string" }`
  - `LoginRequest`: `{ email: "string", password: "string" }`
  - `AuthenticationResponse`: `{ id: "Guid", firstName: "string", lastName: "string", email: "string", token: "string" }`
  - `ProfileResponse`: `{ id: "Guid" }`
  - `ListProfilesResponse`: `{ adminId: "Guid|null", participantId: "Guid|null", trainerId: "Guid|null" }`

## Testing
Run unit tests for each bounded context:
```powershell
# GymManagement domain tests
dotnet test GymManagement/tests/GymManagement.Domain.UnitTests

# SessionReservation domain tests
dotnet test SessionReservation/tests/SessionReservation.Domain.UnitTests
```

## Solutions Overview
- `GymManagement.sln`, `SessionReservation.sln`, `UserManagement.sln` group each context.
- `SharedKernel.csproj` provides shared contracts like `IntegrationEvents/` (e.g., `SessionScheduledIntegrationEvent.cs`, `RoomAddedIntegrationEvent.cs`).

## Development Notes
- Branch: `NET-8-upgraded`
- Style: keep changes minimal and aligned with DDD boundaries.
- Prefer messaging via SharedKernel integration events to coordinate between contexts.

## Troubleshooting
- If a project targets .NET 8, ensure SDK 8.0 is installed: `dotnet --list-sdks`.
- Use `--urls` to override API port, e.g.: `dotnet run --project GymManagement/src/GymManagement.Api/GymManagement.Api.csproj --urls http://localhost:5163`.
- Clear local builds: `dotnet clean` then `dotnet restore`.

## Ports & Environment Quick Reference
Suggested local development ports (adjust as needed):

- GymManagement API: `http://localhost:5163`
- SessionReservation API: `http://localhost:5164`
- UserManagement API: `http://localhost:5165`

PowerShell env var setup (copy/paste):
```powershell
# Common
$env:ASPNETCORE_ENVIRONMENT = "Development"

# GymManagement
$env:ConnectionStrings__Default = "Server=localhost;Database=DomeGym_Gym;Trusted_Connection=True;TrustServerCertificate=True"
$env:MessageBroker__HostName = "localhost"
$env:MessageBroker__UserName = "guest"
$env:MessageBroker__Password = "guest"
$env:MessageBroker__Port = "5672"
$env:MessageBroker__QueueName = "gym-management-integration-events-queue"
$env:MessageBroker__ExchangeName = "IntegrationEvents"

# SessionReservation
$env:ConnectionStrings__Default = "Server=localhost;Database=DomeGym_Session;Trusted_Connection=True;TrustServerCertificate=True"
$env:MessageBroker__HostName = "localhost"
$env:MessageBroker__UserName = "guest"
$env:MessageBroker__Password = "guest"
$env:MessageBroker__Port = "5672"
$env:MessageBroker__QueueName = "session-reservation-integration-events-queue"
$env:MessageBroker__ExchangeName = "IntegrationEvents"

# UserManagement
$env:ConnectionStrings__Default = "Server=localhost;Database=DomeGym_User;Trusted_Connection=True;TrustServerCertificate=True"
$env:MessageBroker__HostName = "localhost"
$env:MessageBroker__UserName = "guest"
$env:MessageBroker__Password = "guest"
$env:MessageBroker__Port = "5672"
$env:MessageBroker__QueueName = "user-management-integration-events-queue"
$env:MessageBroker__ExchangeName = "IntegrationEvents"
# JWT settings (required for auth tokens)
$env:JwtSettings__Secret = "a-very-super-secret-key-that-is-long-enough"
$env:JwtSettings__TokenExpirationInMinutes = "60"
$env:JwtSettings__Issuer = "UserManagement"
$env:JwtSettings__Audience = "DomeGym"

# Run with explicit port
# GymManagement
 dotnet run --project GymManagement/src/GymManagement.Api/GymManagement.Api.csproj --urls http://localhost:5163
# SessionReservation
 dotnet run --project SessionReservation/src/SessionReservation.Api/SessionReservation.Api.csproj --urls http://localhost:5164
# UserManagement
 dotnet run --project UserManagement/src/UserManagement.Api/UserManagement.Api.csproj --urls http://localhost:5165
```

Note: Environment variable names may differ if the APIs expect a specific connection string key (e.g., `ConnectionStrings:Default`). Adjust variable names to match `appsettings.json`.
