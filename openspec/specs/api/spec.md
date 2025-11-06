# API Capability


## Purpose

HTTP/S endpoints for all user-facing operations using H3/Nitro framework with TanStack Start.

## Requirements

This capability provides HTTP/S endpoints for all user-facing operations. All endpoints MUST validate inputs, handle errors consistently, support streaming uploads, and integrate with authentication middleware.

### Requirement: HTTP Server Initialization
The system SHALL initialize H3/Nitro HTTP server with configured port, middleware, and routes.

#### Scenario: Start server on configured port
- **GIVEN** configuration specifies port 3000
- **WHEN** starting server
- **THEN** HTTP server listens on localhost:3000

#### Scenario: Graceful shutdown
- **GIVEN** running HTTP server
- **WHEN** receiving SIGTERM signal
- **THEN** complete in-flight requests, close connections, exit cleanly

#### Scenario: HTTPS support
- **GIVEN** TLS certificate and key configuration
- **WHEN** starting server with HTTPS enabled
- **THEN** serve requests over HTTPS on port 443

### Requirement: Authentication Middleware
The system SHALL authenticate API requests using configured auth mechanism (JWT, API keys, or session).

#### Scenario: Validate JWT token
- **GIVEN** request with Authorization: Bearer <token>
- **WHEN** middleware validates token
- **THEN** extract user_id, attach to request context

#### Scenario: Reject invalid token
- **GIVEN** request with expired or malformed JWT
- **WHEN** middleware validates token
- **THEN** return 401 Unauthorized

#### Scenario: Public endpoint bypass
- **GIVEN** request to health check endpoint
- **WHEN** processing request
- **THEN** skip authentication middleware

### Requirement: Request Validation Middleware
The system SHALL validate request payloads against schemas before reaching route handlers.

#### Scenario: Validate project creation request
- **GIVEN** POST /projects with JSON body
- **WHEN** validation middleware processes request
- **THEN** validate required fields (name), reject if missing

#### Scenario: Validate ULID format in path
- **GIVEN** GET /projects/:id with id=invalid
- **WHEN** validation middleware checks path param
- **THEN** return 400 Bad Request with validation error

#### Scenario: Content-Type validation
- **GIVEN** POST request with Content-Type: text/plain
- **WHEN** expecting application/json
- **THEN** return 415 Unsupported Media Type

### Requirement: Error Handling Middleware
The system SHALL catch and format errors with appropriate HTTP status codes and error messages.

#### Scenario: Handle validation errors
- **GIVEN** validation middleware rejects request
- **WHEN** error handling middleware processes error
- **THEN** return 400 with structured error response

#### Scenario: Handle not found errors
- **GIVEN** request for non-existent project ID
- **WHEN** project not found in database
- **THEN** return 404 with error message

#### Scenario: Handle internal server errors
- **GIVEN** unexpected exception in route handler
- **WHEN** error middleware catches exception
- **THEN** log error details, return 500 with generic message (hide internals)

### Requirement: Project CRUD Endpoints
The system SHALL provide REST endpoints for creating, reading, updating, and deleting projects.

#### Scenario: Create project
- **GIVEN** POST /projects with valid JSON
- **WHEN** processing request
- **THEN** create project, return 201 Created with project metadata

#### Scenario: Get project by ID
- **GIVEN** GET /projects/:id
- **WHEN** fetching project
- **THEN** return 200 OK with full metadata + latest commit info

#### Scenario: List projects in space
- **GIVEN** GET /spaces/:space_id/projects
- **WHEN** querying projects
- **THEN** return 200 OK with paginated project list

#### Scenario: Update project metadata
- **GIVEN** PATCH /projects/:id with name/tags updates
- **WHEN** processing update
- **THEN** validate, update metadata, return 200 OK with updated project

### Requirement: Lease Check-Out Endpoint
The system SHALL provide endpoint for creating leases and initiating project check-out.

#### Scenario: Create full-scope lease
- **GIVEN** POST /projects/:id/leases with scope type=full
- **WHEN** creating lease
- **THEN** return 201 Created with lease metadata + download token

#### Scenario: Create sparse-scope lease
- **GIVEN** POST /projects/:id/leases with scope type=sparse, paths=["/docs/*"]
- **WHEN** creating lease
- **THEN** validate paths, create lease, return lease + download URL

#### Scenario: Prevent overlapping leases
- **GIVEN** active lease on project
- **WHEN** attempting second lease
- **THEN** return 409 Conflict with active lease details

### Requirement: Lease Check-In Endpoint
The system SHALL provide endpoint for uploading changes and completing leases with conflict resolution.

#### Scenario: Check-in with clean base
- **GIVEN** POST /projects/:id/leases/:lease_id/check-in with multipart upload
- **WHEN** base commit matches HEAD
- **THEN** create commit, close lease, return 200 OK with new commit metadata

#### Scenario: Check-in with stale base
- **GIVEN** check-in where base commit != HEAD
- **WHEN** processing check-in
- **THEN** attempt auto-rebase, return conflicts if any

#### Scenario: Handle conflict resolution
- **GIVEN** check-in with conflicts + resolution choices in request
- **WHEN** applying resolutions
- **THEN** complete rebase, create commit, return 200 OK

#### Scenario: Streaming upload support
- **GIVEN** large file upload in multipart request
- **WHEN** processing check-in
- **THEN** stream upload to temporary storage, process incrementally

### Requirement: Health Check Endpoint
The system SHALL provide health check endpoint for monitoring and load balancer probes.

#### Scenario: Basic health check
- **GIVEN** GET /health
- **WHEN** server is running
- **THEN** return 200 OK with status: "healthy"

#### Scenario: Deep health check
- **GIVEN** GET /health?deep=true
- **WHEN** checking subsystems
- **THEN** verify database connection, object storage, return detailed status

#### Scenario: Degraded health
- **GIVEN** database connection failing
- **WHEN** deep health check runs
- **THEN** return 503 Service Unavailable with degraded status

### Requirement: Response Formatting
The system SHALL format all API responses with consistent structure and content negotiation.

#### Scenario: JSON response format
- **GIVEN** successful request
- **WHEN** returning response
- **THEN** use Content-Type: application/json with structured data

#### Scenario: Error response format
- **GIVEN** request resulting in error
- **WHEN** formatting error response
- **THEN** return `{ "error": { "code": "...", "message": "...", "details": {} } }`

#### Scenario: Content negotiation
- **GIVEN** request with Accept: application/json
- **WHEN** server supports JSON
- **THEN** return JSON response with matching Content-Type
