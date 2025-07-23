# Rust Architecture Patterns

Building software that stands the test of time requires more than just writing code that works—it demands thoughtful architecture, proven patterns, and disciplined engineering practices. This comprehensive guide distills years of experience into 10 battle-tested architectural patterns that transform complex business requirements into maintainable, scalable, and observable systems.

Whether you're architecting a new application or refactoring an existing codebase, these patterns provide a structured approach to common challenges: from organizing domain logic and managing data persistence to implementing comprehensive observability and handling complex business workflows. Each pattern includes detailed Rust implementations, real-world examples, and practical guidance for integration with modern frameworks like Axum, SQLx, and OpenTelemetry.

## Why These Patterns Matter

Modern applications face unprecedented complexity: distributed systems, event-driven architectures, microservices, and the constant pressure to deliver features while maintaining quality. These patterns address the fundamental challenges that every development team encounters:

- **Complexity Management**: Break down complex systems into manageable, well-defined components
- **Change Resilience**: Build systems that adapt to evolving business requirements without massive rewrites
- **Quality Assurance**: Implement testing strategies that provide confidence without slowing development
- **Operational Excellence**: Create systems that are observable, debuggable, and maintainable in production
- **Team Scalability**: Establish conventions that enable multiple developers to work effectively on the same codebase

## How to Use This Guide

Each pattern is self-contained but builds upon concepts from previous sections. The patterns are organized progressively, starting with foundational testing and data access patterns, then moving through business logic organization, and culminating in advanced patterns for event-driven architectures and comprehensive observability.

Code examples follow a consistent function-based approach that emphasizes simplicity and testability. All examples are production-ready and include proper error handling, logging, and performance considerations.

---

## Table of Contents

1. [Integration Test Architecture](#1-integration-test-architecture)
2. [Test Database Optimization](#2-test-database-optimization)  
3. [Repository Pattern](#3-repository-pattern)
4. [Shared Module Architecture](#4-shared-module-architecture)
5. [Unified Error Handling](#5-unified-error-handling)
6. [Domain Events System](#6-domain-events-system)
7. [Validation Rules Engine](#7-validation-rules-engine)
8. [Event Sourcing and CQRS](#8-event-sourcing-and-cqrs)
9. [Observability and Telemetry Architecture](#9-observability-and-telemetry-architecture)
10. [General Architecture Principles](#10-general-architecture-principles)

---

## Initial Project Structure

Before diving into patterns, here's a typical project structure for a web application with authentication and user management:

```
myapp/
├── Cargo.toml
├── .env.example
├── migrations/
│   ├── 001_create_users_table.sql
│   └── 002_create_sessions_table.sql
├── src/
│   ├── main.rs                 # Application entry point
│   ├── lib.rs                  # Library root, router setup
│   ├── config.rs               # Configuration management
│   ├── error.rs                # Unified error type
│   ├── state.rs                # Application state
│   ├── auth/                   # Authentication module
│   │   ├── mod.rs              # Module exports and router
│   │   ├── middleware.rs       # Auth middleware (optional)
│   │   ├── api/                # HTTP handlers
│   │   │   ├── mod.rs
│   │   │   ├── login.rs
│   │   │   ├── logout.rs
│   │   │   └── register.rs
│   │   ├── models/             # Data structures
│   │   │   ├── mod.rs
│   │   │   └── session.rs
│   │   ├── repositories/       # Data access layer
│   │   │   ├── mod.rs
│   │   │   └── session.rs
│   │   └── services/           # Business logic
│   │       ├── mod.rs
│   │       └── auth.rs
│   ├── users/                  # User management
│   │   ├── mod.rs
│   │   ├── api/
│   │   │   ├── mod.rs
│   │   │   ├── get.rs
│   │   │   ├── update.rs
│   │   │   └── delete.rs
│   │   ├── models/
│   │   │   ├── mod.rs
│   │   │   ├── user.rs
│   │   │   └── password.rs
│   │   ├── repositories/
│   │   │   ├── mod.rs
│   │   │   └── user.rs
│   │   └── services/
│   │       ├── mod.rs
│   │       └── user.rs
│   └── shared/                 # Shared utilities
│       ├── mod.rs
│       ├── types.rs            # Common types
│       ├── validation.rs       # Validation rules
│       └── events.rs           # Domain events
└── tests/
    ├── helpers/
    │   ├── mod.rs
    │   ├── test_app.rs
    │   └── test_data.rs
    └── auth/
        └── api.rs
```

### Core Application Setup

**src/main.rs**
```rust
use myapp::{AppConfig, AppState, run};
use sqlx::postgres::PgPoolOptions;
use tokio::net::TcpListener;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Initialize tracing
    tracing_subscriber::fmt::init();
    
    // Load configuration
    dotenvy::dotenv().ok();
    let config = AppConfig::load()?;
    
    // Create database pool
    let db_pool = PgPoolOptions::new()
        .max_connections(config.database_max_connections)
        .connect(&config.database_url)
        .await?;
    
    // Run migrations
    sqlx::migrate!("./migrations").run(&db_pool).await?;
    
    // Create app state
    let state = AppState::new(config, db_pool);
    
    // Start server
    let listener = TcpListener::bind(&state.config.server_address).await?;
    tracing::info!("Server listening on {}", state.config.server_address);
    
    run(state, listener).await?;
    Ok(())
}
```

**src/lib.rs**
```rust
mod auth;
mod config;
mod error;
mod shared;
mod state;
mod users;

pub use config::AppConfig;
pub use error::{Error, Result};
pub use state::AppState;

use axum::{
    Router,
    http::StatusCode,
    response::IntoResponse,
};
use tower_http::trace::TraceLayer;

pub async fn run(
    state: AppState,
    listener: tokio::net::TcpListener,
) -> Result<()> {
    let app = Router::new()
        .nest("/api/v1/auth", auth::router(state.clone()))
        .nest("/api/v1/users", users::router(state.clone()))
        .fallback(handler_404)
        .layer(TraceLayer::new_for_http())
        .with_state(state);

    axum::serve(listener, app).await?;
    Ok(())
}

async fn handler_404() -> impl IntoResponse {
    (StatusCode::NOT_FOUND, "Not Found")
}
```

**src/error.rs**
```rust
use axum::{
    response::{IntoResponse, Response},
    http::StatusCode,
    Json,
};
use serde_json::json;

#[derive(Debug, thiserror::Error)]
pub enum Error {
    #[error("Database error: {0}")]
    Database(#[from] sqlx::Error),
    
    #[error("Authentication required")]
    Unauthorized,
    
    #[error("Invalid credentials")]
    InvalidCredentials,
    
    #[error("Resource not found")]
    NotFound,
    
    #[error("Validation failed: {field}: {message}")]
    ValidationFailed { field: String, message: String },
    
    #[error("Internal server error")]
    Internal(String),
}

pub type Result<T> = std::result::Result<T, Error>;

impl IntoResponse for Error {
    fn into_response(self) -> Response {
        let (status, message) = match self {
            Error::Database(_) => (StatusCode::INTERNAL_SERVER_ERROR, "Database error"),
            Error::Unauthorized => (StatusCode::UNAUTHORIZED, "Authentication required"),
            Error::InvalidCredentials => (StatusCode::UNAUTHORIZED, "Invalid credentials"),
            Error::NotFound => (StatusCode::NOT_FOUND, "Resource not found"),
            Error::ValidationFailed { .. } => (StatusCode::BAD_REQUEST, self.to_string().as_str()),
            Error::Internal(_) => (StatusCode::INTERNAL_SERVER_ERROR, "Internal server error"),
        };
        
        let body = Json(json!({
            "error": message,
            "status": status.as_u16(),
        }));
        
        (status, body).into_response()
    }
}
```

**src/auth/mod.rs**
```rust
pub mod api;
pub mod middleware;
pub mod models;
pub mod repositories;
pub mod services;

use crate::AppState;
use axum::{
    routing::{post, get},
    Router,
};

pub fn router(state: AppState) -> Router<AppState> {
    Router::new()
        .route("/register", post(api::register::handler))
        .route("/login", post(api::login::handler))
        .route("/logout", post(api::logout::handler))
        .route("/me", get(api::me::handler))
}
```

### Complete Module Example: Authentication

**src/auth/models/session.rs**
```rust
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use sqlx::FromRow;
use uuid::Uuid;

#[derive(Debug, Clone, FromRow, Serialize, Deserialize)]
pub struct Session {
    pub id: Uuid,
    pub user_id: Uuid,
    pub token: String,
    pub expires_at: DateTime<Utc>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize)]
pub struct LoginRequest {
    pub username: String,
    pub password: String,
}

#[derive(Debug, Serialize)]
pub struct LoginResponse {
    pub token: String,
    pub user: UserInfo,
}

#[derive(Debug, Serialize)]
pub struct UserInfo {
    pub id: Uuid,
    pub username: String,
    pub email: String,
}
```

**src/auth/repositories/session.rs**
```rust
use crate::{DbConn, Error, Result};
use crate::auth::models::session::Session;
use chrono::{DateTime, Utc};
use uuid::Uuid;

/// Creates a new session in the database
pub async fn create_session(
    conn: &mut DbConn,
    user_id: Uuid,
    token: &str,
    expires_at: DateTime<Utc>
) -> Result<Session> {
    let session = sqlx::query_as!(
        Session,
        r#"
        INSERT INTO sessions (id, user_id, token, expires_at, created_at, updated_at)
        VALUES ($1, $2, $3, $4, NOW(), NOW())
        RETURNING *
        "#,
        Uuid::now_v7(),
        user_id,
        token,
        expires_at
    )
    .fetch_one(conn)
    .await?;

    Ok(session)
}

/// Finds a session by its token
pub async fn find_session_by_token(
    conn: &mut DbConn,
    token: &str
) -> Result<Option<Session>> {
    let session = sqlx::query_as!(
        Session,
        r#"
        SELECT * FROM sessions
        WHERE token = $1 AND expires_at > NOW()
        "#,
        token
    )
    .fetch_optional(conn)
    .await?;

    Ok(session)
}

/// Deletes a session by token
pub async fn delete_session(conn: &mut DbConn, token: &str) -> Result<()> {
    sqlx::query!(
        "DELETE FROM sessions WHERE token = $1",
        token
    )
    .execute(conn)
    .await?;

    Ok(())
}

/// Deletes all expired sessions
pub async fn delete_expired_sessions(conn: &mut DbConn) -> Result<u64> {
    let result = sqlx::query!(
        "DELETE FROM sessions WHERE expires_at <= NOW()"
    )
    .execute(conn)
    .await?;

    Ok(result.rows_affected())
}

/// Updates session last activity timestamp
pub async fn update_session_activity(
    conn: &mut DbConn,
    session_id: Uuid
) -> Result<()> {
    sqlx::query!(
        r#"
        UPDATE sessions 
        SET last_activity = NOW(), updated_at = NOW()
        WHERE id = $1
        "#,
        session_id
    )
    .execute(conn)
    .await?;

    Ok(())
}
```

**src/auth/services/auth.rs**
```rust
use crate::{DbConn, Error, Result};
use crate::auth::models::session::{LoginRequest, LoginResponse, UserInfo};
use crate::auth::repositories;
use crate::users::repositories as user_repos;
use crate::shared::password::verify_password;
use chrono::{Duration, Utc};
use rand::Rng;
use uuid::Uuid;

/// Handles user login and creates a new session
pub async fn login(
    conn: &mut DbConn,
    req: LoginRequest
) -> Result<LoginResponse> {
    // Find user by username
    let user = user_repos::find_user_by_username(conn, &req.username)
        .await?
        .ok_or(Error::InvalidCredentials)?;

    // Verify password
    if !verify_password(&req.password, &user.password_hash)? {
        return Err(Error::InvalidCredentials);
    }

    // Generate session token
    let token = generate_session_token();
    let expires_at = Utc::now() + Duration::days(7);

    // Create session
    repositories::create_session(conn, user.id, &token, expires_at).await?;

    Ok(LoginResponse {
        token,
        user: UserInfo {
            id: user.id,
            username: user.username,
            email: user.email,
        },
    })
}

/// Logs out a user by deleting their session
pub async fn logout(conn: &mut DbConn, token: &str) -> Result<()> {
    repositories::delete_session(conn, token).await
}

/// Validates a session token and returns the user ID if valid
pub async fn validate_token(
    conn: &mut DbConn,
    token: &str
) -> Result<Option<Uuid>> {
    let session = repositories::find_session_by_token(conn, token).await?;
    
    // Update last activity if session is valid
    if let Some(ref s) = session {
        repositories::update_session_activity(conn, s.id).await?;
    }
    
    Ok(session.map(|s| s.user_id))
}

/// Cleans up expired sessions
pub async fn cleanup_sessions(conn: &mut DbConn) -> Result<u64> {
    repositories::delete_expired_sessions(conn).await
}

/// Generates a cryptographically secure session token
fn generate_session_token() -> String {
    let mut rng = rand::thread_rng();
    std::iter::repeat_with(|| rng.sample(rand::distributions::Alphanumeric))
        .take(32)
        .map(char::from)
        .collect()
}

/// Creates a new user account
pub async fn register(
    conn: &mut DbConn,
    req: crate::users::models::CreateUserRequest
) -> Result<crate::users::models::User> {
    user_repos::create_user(conn, req).await
}
```

**src/auth/api/login.rs**
```rust
use crate::{AppState, Result};
use crate::auth::models::session::{LoginRequest, LoginResponse};
use crate::auth::services;
use axum::{
    extract::State,
    Json,
    response::IntoResponse,
};

pub async fn handler(
    State(state): State<AppState>,
    Json(req): Json<LoginRequest>,
) -> Result<impl IntoResponse> {
    let mut conn = state.db_pool.acquire().await?;
    let response = services::login(&mut conn, req).await?;
    
    Ok(Json(response))
}
```

**src/auth/middleware.rs**
```rust
use crate::{AppState, Error};
use crate::auth::services;
use axum::{
    extract::{Request, State},
    middleware::Next,
    response::Response,
};
use axum_extra::{
    headers::{authorization::Bearer, Authorization},
    TypedHeader,
};

pub struct AuthUser {
    pub user_id: uuid::Uuid,
}

pub async fn require_auth(
    State(state): State<AppState>,
    TypedHeader(auth): TypedHeader<Authorization<Bearer>>,
    mut request: Request,
    next: Next,
) -> Result<Response, Error> {
    let token = auth.token();
    
    let mut conn = state.db_pool.acquire().await
        .map_err(|_| Error::Unauthorized)?;
    
    let user_id = services::validate_token(&mut conn, token)
        .await?
        .ok_or(Error::Unauthorized)?;

    // Insert user info into request extensions
    request.extensions_mut().insert(AuthUser { user_id });
    
    Ok(next.run(request).await)
}
```

### Complete Module Example: Users

**src/users/models/user.rs**
```rust
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use sqlx::FromRow;
use uuid::Uuid;

#[derive(Debug, Clone, FromRow, Serialize)]
pub struct User {
    pub id: Uuid,
    pub username: String,
    pub email: String,
    #[serde(skip)]
    pub password_hash: String,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize)]
pub struct CreateUserRequest {
    pub username: String,
    pub email: String,
    pub password: String,
}

#[derive(Debug, Deserialize)]
pub struct UpdateUserRequest {
    pub email: Option<String>,
    pub password: Option<String>,
}
```

**src/users/repositories/user.rs**
```rust
use crate::{DbConn, Error, Result};
use crate::users::models::user::{User, CreateUserRequest};
use crate::shared::password::hash_password;
use crate::shared::validation::{validate_email, validate_password_strength};
use uuid::Uuid;

/// Creates a new user in the database
pub async fn create_user(
    conn: &mut DbConn,
    req: CreateUserRequest
) -> Result<User> {
    // Validate input
    validate_email(&req.email)?;
    validate_password_strength(&req.password)?;
    
    // Hash password
    let password_hash = hash_password(&req.password)?;
    
    let user = sqlx::query_as!(
        User,
        r#"
        INSERT INTO users (id, username, email, password_hash, created_at, updated_at)
        VALUES ($1, $2, $3, $4, NOW(), NOW())
        RETURNING *
        "#,
        Uuid::now_v7(),
        req.username,
        req.email,
        password_hash
    )
    .fetch_one(conn)
    .await
    .map_err(|e| match e {
        sqlx::Error::Database(db_err) if db_err.constraint() == Some("users_username_key") => {
            Error::ValidationFailed {
                field: "username".to_string(),
                message: "Username already exists".to_string(),
            }
        }
        sqlx::Error::Database(db_err) if db_err.constraint() == Some("users_email_key") => {
            Error::ValidationFailed {
                field: "email".to_string(),
                message: "Email already exists".to_string(),
            }
        }
        _ => Error::Database(e),
    })?;

    Ok(user)
}

/// Finds a user by ID
pub async fn find_user_by_id(
    conn: &mut DbConn,
    id: Uuid
) -> Result<Option<User>> {
    let user = sqlx::query_as!(
        User,
        "SELECT * FROM users WHERE id = $1",
        id
    )
    .fetch_optional(conn)
    .await?;

    Ok(user)
}

/// Finds a user by username
pub async fn find_user_by_username(
    conn: &mut DbConn,
    username: &str
) -> Result<Option<User>> {
    let user = sqlx::query_as!(
        User,
        "SELECT * FROM users WHERE username = $1",
        username
    )
    .fetch_optional(conn)
    .await?;

    Ok(user)
}

/// Finds a user by email
pub async fn find_user_by_email(
    conn: &mut DbConn,
    email: &str
) -> Result<Option<User>> {
    let user = sqlx::query_as!(
        User,
        "SELECT * FROM users WHERE email = $1",
        email
    )
    .fetch_optional(conn)
    .await?;

    Ok(user)
}

/// Updates user email
pub async fn update_user_email(
    conn: &mut DbConn,
    id: Uuid,
    email: &str
) -> Result<User> {
    validate_email(email)?;
    
    let user = sqlx::query_as!(
        User,
        r#"
        UPDATE users 
        SET email = $2, updated_at = NOW()
        WHERE id = $1
        RETURNING *
        "#,
        id,
        email
    )
    .fetch_one(conn)
    .await
    .map_err(|e| match e {
        sqlx::Error::Database(db_err) if db_err.constraint() == Some("users_email_key") => {
            Error::ValidationFailed {
                field: "email".to_string(),
                message: "Email already exists".to_string(),
            }
        }
        sqlx::Error::RowNotFound => Error::NotFound,
        _ => Error::Database(e),
    })?;

    Ok(user)
}

/// Updates user password
pub async fn update_user_password(
    conn: &mut DbConn,
    id: Uuid,
    password: &str
) -> Result<()> {
    validate_password_strength(password)?;
    let password_hash = hash_password(password)?;
    
    let result = sqlx::query!(
        r#"
        UPDATE users 
        SET password_hash = $2, updated_at = NOW()
        WHERE id = $1
        "#,
        id,
        password_hash
    )
    .execute(conn)
    .await?;

    if result.rows_affected() == 0 {
        return Err(Error::NotFound);
    }

    Ok(())
}

/// Deletes a user
pub async fn delete_user(conn: &mut DbConn, id: Uuid) -> Result<()> {
    let result = sqlx::query!(
        "DELETE FROM users WHERE id = $1",
        id
    )
    .execute(conn)
    .await?;

    if result.rows_affected() == 0 {
        return Err(Error::NotFound);
    }

    Ok(())
}
```

**src/users/services/user.rs**
```rust
use crate::{DbConn, Error, Result};
use crate::users::models::user::{User, UpdateUserRequest};
use crate::users::repositories;
use crate::shared::password::{verify_password, check_password_compromised};
use uuid::Uuid;

/// Gets a user by ID
pub async fn get_user(conn: &mut DbConn, id: Uuid) -> Result<User> {
    repositories::find_user_by_id(conn, id)
        .await?
        .ok_or(Error::NotFound)
}

/// Gets a user by username
pub async fn get_user_by_username(
    conn: &mut DbConn,
    username: &str
) -> Result<User> {
    repositories::find_user_by_username(conn, username)
        .await?
        .ok_or(Error::NotFound)
}

/// Updates user profile information
pub async fn update_user(
    conn: &mut DbConn,
    id: Uuid,
    req: UpdateUserRequest
) -> Result<User> {
    // Update email if provided
    let user = if let Some(email) = req.email {
        repositories::update_user_email(conn, id, &email).await?
    } else {
        repositories::find_user_by_id(conn, id)
            .await?
            .ok_or(Error::NotFound)?
    };

    // Update password if provided
    if let Some(password) = req.password {
        // Optionally check if password has been compromised
        if check_password_compromised(&password).await.unwrap_or(false) {
            return Err(Error::ValidationFailed {
                field: "password".to_string(),
                message: "This password has been found in data breaches".to_string(),
            });
        }
        
        repositories::update_user_password(conn, id, &password).await?;
    }

    Ok(user)
}

/// Changes user password (requires current password)
pub async fn change_password(
    conn: &mut DbConn,
    user_id: Uuid,
    current_password: &str,
    new_password: &str
) -> Result<()> {
    // Get user to verify current password
    let user = repositories::find_user_by_id(conn, user_id)
        .await?
        .ok_or(Error::NotFound)?;

    // Verify current password
    if !verify_password(current_password, &user.password_hash)? {
        return Err(Error::InvalidCredentials);
    }

    // Update to new password
    repositories::update_user_password(conn, user_id, new_password).await
}

/// Deletes a user account
pub async fn delete_user(conn: &mut DbConn, id: Uuid) -> Result<()> {
    repositories::delete_user(conn, id).await
}

/// Checks if a username is available
pub async fn check_username_available(
    conn: &mut DbConn,
    username: &str
) -> Result<bool> {
    let user = repositories::find_user_by_username(conn, username).await?;
    Ok(user.is_none())
}

/// Checks if an email is available
pub async fn check_email_available(
    conn: &mut DbConn,
    email: &str
) -> Result<bool> {
    let user = repositories::find_user_by_email(conn, email).await?;
    Ok(user.is_none())
}
```

**src/users/api/get.rs**
```rust
use crate::{AppState, Result};
use crate::auth::middleware::AuthUser;
use crate::users::services;
use axum::{
    extract::{Path, State},
    Extension,
    Json,
    response::IntoResponse,
};
use uuid::Uuid;

pub async fn get_current_user(
    State(state): State<AppState>,
    Extension(auth): Extension<AuthUser>,
) -> Result<impl IntoResponse> {
    let mut conn = state.db_pool.acquire().await?;
    let user = services::get_user(&mut conn, auth.user_id).await?;
    Ok(Json(user))
}

pub async fn get_user_by_id(
    State(state): State<AppState>,
    Extension(_auth): Extension<AuthUser>, // Require authentication
    Path(id): Path<Uuid>,
) -> Result<impl IntoResponse> {
    let mut conn = state.db_pool.acquire().await?;
    let user = services::get_user(&mut conn, id).await?;
    Ok(Json(user))
}
```

**src/state.rs**
```rust
use crate::{AppConfig, Error};
use sqlx::PgPool;

#[derive(Clone)]
pub struct AppState {
    pub config: AppConfig,
    pub db_pool: PgPool,
}

impl AppState {
    pub fn new(config: AppConfig, db_pool: PgPool) -> Self {
        Self { config, db_pool }
    }
}
```

### Shared Module Examples

**src/shared/validation.rs**
```rust
use crate::{Error, Result};
use once_cell::sync::Lazy;
use regex::Regex;

static EMAIL_REGEX: Lazy<Regex> = Lazy::new(|| {
    Regex::new(r"^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$").unwrap()
});

pub fn validate_email(email: &str) -> Result<()> {
    if !EMAIL_REGEX.is_match(email) {
        return Err(Error::ValidationFailed {
            field: "email".to_string(),
            message: "Invalid email format".to_string(),
        });
    }
    Ok(())
}

pub fn validate_non_empty_string(value: &str, field_name: &str) -> Result<()> {
    if value.trim().is_empty() {
        return Err(Error::ValidationFailed {
            field: field_name.to_string(),
            message: "Cannot be empty".to_string(),
        });
    }
    Ok(())
}

pub fn validate_string_length(
    value: &str, 
    field_name: &str, 
    min: usize, 
    max: usize
) -> Result<()> {
    let len = value.trim().len();
    if len < min || len > max {
        return Err(Error::ValidationFailed {
            field: field_name.to_string(),
            message: format!("Must be between {} and {} characters", min, max),
        });
    }
    Ok(())
}
```

**src/shared/password.rs**
```rust
use crate::{Error, Result};
use once_cell::sync::Lazy;
use regex::Regex;
use zxcvbn::zxcvbn;

// Password validation regexes
static HAS_UPPERCASE: Lazy<Regex> = Lazy::new(|| Regex::new(r"[A-Z]").unwrap());
static HAS_LOWERCASE: Lazy<Regex> = Lazy::new(|| Regex::new(r"[a-z]").unwrap());
static HAS_DIGIT: Lazy<Regex> = Lazy::new(|| Regex::new(r"\d").unwrap());
static HAS_SPECIAL: Lazy<Regex> = Lazy::new(|| Regex::new(r"[!@#$%^&*(),.?\":{}|<>]").unwrap());

#[derive(Debug)]
pub struct PasswordStrength {
    pub score: u8,           // 0-4 (0 = very weak, 4 = very strong)
    pub feedback: Vec<String>,
    pub time_to_crack: String,
}

/// Validate password meets minimum requirements
pub fn validate_password_strength(password: &str) -> Result<()> {
    let mut errors = Vec::new();

    if password.len() < 8 {
        errors.push("Password must be at least 8 characters long");
    }

    if !HAS_UPPERCASE.is_match(password) {
        errors.push("Password must contain at least one uppercase letter");
    }

    if !HAS_LOWERCASE.is_match(password) {
        errors.push("Password must contain at least one lowercase letter");
    }

    if !HAS_DIGIT.is_match(password) {
        errors.push("Password must contain at least one number");
    }

    if !errors.is_empty() {
        return Err(Error::ValidationFailed {
            field: "password".to_string(),
            message: errors.join(", "),
        });
    }

    Ok(())
}

/// Analyze password strength using zxcvbn
pub fn analyze_password_strength(password: &str, user_inputs: &[&str]) -> PasswordStrength {
    let result = zxcvbn(password, user_inputs).unwrap();
    
    let mut feedback = Vec::new();
    
    // Add specific feedback based on what's missing
    if !HAS_UPPERCASE.is_match(password) {
        feedback.push("Add uppercase letters".to_string());
    }
    if !HAS_LOWERCASE.is_match(password) {
        feedback.push("Add lowercase letters".to_string());
    }
    if !HAS_DIGIT.is_match(password) {
        feedback.push("Add numbers".to_string());
    }
    if !HAS_SPECIAL.is_match(password) {
        feedback.push("Add special characters for extra security".to_string());
    }
    
    // Add zxcvbn suggestions
    if let Some(suggestions) = result.feedback() {
        feedback.extend(suggestions.suggestions().iter().map(|s| s.to_string()));
    }

    PasswordStrength {
        score: result.score(),
        feedback,
        time_to_crack: format_crack_time(result.crack_times().offline_slow_hashing_1e4_per_second()),
    }
}

/// Hash password using Argon2
pub fn hash_password(password: &str) -> Result<String> {
    password_auth::generate_hash(password)
        .map_err(|_| Error::Internal("Failed to hash password".to_string()))
}

/// Verify password against hash
pub fn verify_password(password: &str, hash: &str) -> Result<bool> {
    match password_auth::verify_password(password, hash) {
        Ok(()) => Ok(true),
        Err(password_auth::VerifyError::PasswordInvalid) => Ok(false),
        Err(_) => Err(Error::Internal("Failed to verify password".to_string())),
    }
}

/// Check if password has been compromised (via HaveIBeenPwned API)
pub async fn check_password_compromised(password: &str) -> Result<bool> {
    use sha1::{Sha1, Digest};
    
    // Calculate SHA1 hash
    let mut hasher = Sha1::new();
    hasher.update(password.as_bytes());
    let hash = format!("{:X}", hasher.finalize());
    
    let prefix = &hash[..5];
    let suffix = &hash[5..];
    
    // Query HaveIBeenPwned API
    let url = format!("https://api.pwnedpasswords.com/range/{}", prefix);
    let response = reqwest::get(&url)
        .await
        .map_err(|_| Error::Internal("Failed to check password breach status".to_string()))?;
    
    let text = response.text()
        .await
        .map_err(|_| Error::Internal("Failed to read breach response".to_string()))?;
    
    // Check if our hash suffix appears in the response
    for line in text.lines() {
        if let Some((hash_suffix, _count)) = line.split_once(':') {
            if hash_suffix == suffix {
                return Ok(true);
            }
        }
    }
    
    Ok(false)
}

fn format_crack_time(seconds: f64) -> String {
    const MINUTE: f64 = 60.0;
    const HOUR: f64 = MINUTE * 60.0;
    const DAY: f64 = HOUR * 24.0;
    const MONTH: f64 = DAY * 30.0;
    const YEAR: f64 = DAY * 365.0;
    const CENTURY: f64 = YEAR * 100.0;

    if seconds < 1.0 {
        "instantly".to_string()
    } else if seconds < MINUTE {
        format!("{} seconds", seconds.round())
    } else if seconds < HOUR {
        format!("{} minutes", (seconds / MINUTE).round())
    } else if seconds < DAY {
        format!("{} hours", (seconds / HOUR).round())
    } else if seconds < MONTH {
        format!("{} days", (seconds / DAY).round())
    } else if seconds < YEAR {
        format!("{} months", (seconds / MONTH).round())
    } else if seconds < CENTURY {
        format!("{} years", (seconds / YEAR).round())
    } else {
        "centuries".to_string()
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_password_validation() {
        assert!(validate_password_strength("weak").is_err());
        assert!(validate_password_strength("NoDigits!").is_err());
        assert!(validate_password_strength("nouppercas3!").is_err());
        assert!(validate_password_strength("NOLOWERCASE3!").is_err());
        assert!(validate_password_strength("ValidPass123").is_ok());
    }

    #[test]
    fn test_password_strength_analysis() {
        let weak = analyze_password_strength("password", &[]);
        assert_eq!(weak.score, 0);
        
        let strong = analyze_password_strength("MyStr0ng!P@ssw0rd#2024", &[]);
        assert!(strong.score >= 3);
    }

    #[test]
    fn test_password_hashing() {
        let password = "MySecurePassword123!";
        let hash = hash_password(password).unwrap();
        
        assert!(verify_password(password, &hash).unwrap());
        assert!(!verify_password("WrongPassword", &hash).unwrap());
    }
}
```

## 1. Integration Test Architecture

**The Problem**: Unit tests are fast but limited—they can't catch integration failures, configuration issues, or database constraints. End-to-end tests are comprehensive but slow and brittle. How do you gain confidence that your entire system works without sacrificing development velocity?

**The Solution**: Integration tests that spin up real application instances with isolated databases, testing the full request-response cycle while maintaining speed and reliability. This approach catches 80% of production issues while running in seconds, not minutes.

**Why It Matters**: Integration tests are your safety net during refactoring, your documentation of system behavior, and your confidence boost during deployment. They prove that your API actually works with your database, your middleware actually enforces security, and your business logic actually handles edge cases.

### World-Class Insights

**🎯 The Testing Pyramid Misconception**: Many teams obsess over the "testing pyramid" and under-invest in integration tests. In reality, integration tests often provide the highest ROI—they catch the bugs that actually break production while remaining fast enough for continuous feedback.

**💡 Database as a Feature**: Don't mock your database in integration tests. Your database constraints, indexes, and triggers are part of your business logic. Mocking them means you're not testing your actual system—you're testing a fantasy version that doesn't exist in production.

**🔧 Port Isolation Strategy**: Using random ports isn't just for avoiding conflicts—it enables true parallel test execution. Each test gets its own universe, eliminating test pollution and race conditions that plague shared-resource testing.

**⚡ Performance Secrets**: 
- Keep connection pools small in tests (1-2 connections) to avoid overwhelming your test database
- Use `tokio::time::timeout()` to fail fast on hung operations
- Implement exponential backoff for flaky external dependencies
- Cache expensive setup operations (like JWT keys) as static variables

**🛡️ Production Parity Techniques**:
- Use the same Docker images in tests and production
- Test with realistic data volumes—your query works with 10 rows but fails with 10,000
- Include the same environment variables and configuration patterns
- Test deployment scenarios with actual migration scripts

### Solution: Standalone Test App Pattern
Create a dedicated test application that mirrors your production setup but optimized for testing:

```
tests/
├── helpers/
│   ├── test_app.rs     # Standalone server spawning
│   ├── db.rs           # Database setup/cleanup
│   ├── test_data.rs    # Test data factories
│   └── utils.rs        # Common test utilities
├── domain1/
│   ├── api.rs          # HTTP endpoint tests
│   └── mod.rs
├── domain2/
│   ├── api.rs
│   └── mod.rs
└── integration/
    └── end_to_end.rs   # Full workflow tests
```

### Core Implementation

**1. TestApp Structure and Setup:**
```rust
// tests/helpers/test_app.rs
use once_cell::sync::Lazy;
use reqwest::redirect::Policy;
use tokio::net::TcpListener;

#[derive(Debug, Clone)]
pub struct TestApp {
    pub address: String,        // Random port address (http://127.0.0.1:XXXX)
    pub client: reqwest::Client, // HTTP client with cookie store
    pub config: AppConfig,      // Test-specific configuration
    pub state: AppState,        // Application state for direct access
}

#[derive(Debug, Clone)]
pub struct BasicAuth {
    pub username: String,
    pub password: Option<String>,
}

// Global tracing setup - only initialize once
static TRACING: Lazy<()> = Lazy::new(|| {
    init_tracing(); // Your app's tracing initialization
});

pub async fn spawn_app() -> TestApp {
    // Initialize tracing once across all tests
    Lazy::force(&TRACING);

    // Load test configuration
    dotenvy::dotenv().ok();
    let config = AppConfig::load().expect("Failed to load config");
    
    // Prepare isolated test database
    let config = prepare_db(config)
        .await
        .expect("Failed to prepare test database");

    // Create application state
    let state = AppState::new(config.clone())
        .await
        .expect("Failed to create AppState");
    
    // Run database migrations
    migrate_db(&state).await;

    // Bind to random port
    let listener = TcpListener::bind("127.0.0.1:0")
        .await
        .expect("Failed to bind random port");
    let port = listener.local_addr().unwrap().port();
    let address = format!("http://127.0.0.1:{}", port);

    // Start server in background
    let server = run(state.clone(), listener); // Your app's main server function
    tokio::spawn(server);

    // Give server time to start
    tokio::time::sleep(std::time::Duration::from_millis(100)).await;

    // Create HTTP client with persistent cookies
    let client = reqwest::Client::builder()
        .redirect(Policy::none())           // Handle redirects manually
        .cookie_store(true)                 // Persist cookies for sessions
        .timeout(std::time::Duration::from_secs(30))
        .build()
        .expect("Failed to create HTTP client");

    TestApp {
        address,
        client,
        config,
        state,
    }
}
```

**2. HTTP Helper Methods:**
```rust
impl TestApp {
    // GET request with optional authentication
    pub async fn get(&self, path: &str, auth: Option<BasicAuth>) -> reqwest::Response {
        let url = format!("{}{}", self.address, path);
        let mut builder = self.client.get(url);
        if let Some(auth) = auth {
            builder = builder.basic_auth(auth.username, auth.password);
        }
        builder.send().await.expect("Failed to execute GET request")
    }

    // POST with JSON body
    pub async fn post_json(
        &self,
        path: &str,
        auth: Option<BasicAuth>,
        content: &serde_json::Value,
    ) -> reqwest::Response {
        let url = format!("{}{}", self.address, path);
        let mut builder = self.client.post(url);
        if let Some(auth) = auth {
            builder = builder.basic_auth(auth.username, auth.password);
        }
        builder
            .json(content)
            .send()
            .await
            .expect("Failed to execute POST request")
    }

    // PUT, PATCH, DELETE methods follow same pattern...

    // Extract session cookies for authentication
    pub fn extract_session_cookie(&self, response: &reqwest::Response) -> Option<String> {
        response
            .headers()
            .get("set-cookie")
            .and_then(|value| value.to_str().ok())
            .and_then(|cookie| {
                cookie
                    .split(';')
                    .find(|part| part.starts_with("session-token="))
                    .map(|s| s.strip_prefix("session-token=").unwrap_or(s).to_string())
            })
    }

    // Cleanup test database after test
    pub async fn cleanup(&self) {
        // Connect to postgres server (not test database)
        let db = PgPool::connect(self.config.database_url_without_db_name())
            .await
            .expect("Failed to connect for cleanup");

        // Force drop test database
        sqlx::query(&format!(
            "DROP DATABASE \"{}\" WITH (FORCE)",
            self.config.db_name()
        ))
        .execute(&db)
        .await
        .expect("Failed to cleanup test database");
    }
}
```

**3. Example Test Usage:**
```rust
// tests/auth/api.rs
use crate::helpers::test_app::{spawn_app, BasicAuth};
use serde_json::json;

#[tokio::test]
async fn test_user_registration_and_login() {
    let app = spawn_app().await;

    // Register new user
    let registration_data = json!({
        "username": "testuser",
        "email": "test@example.com",
        "password": "SecurePass123!"
    });

    let response = app
        .post_json("/api/v1/auth/register", None, &registration_data)
        .await;
    
    assert_eq!(response.status(), 201);

    // Login with credentials
    let login_data = json!({
        "username": "testuser",
        "password": "SecurePass123!"
    });

    let response = app
        .post_json("/api/v1/auth/login", None, &login_data)
        .await;
    
    assert_eq!(response.status(), 200);
    
    // Extract session cookie
    let session_cookie = app.extract_session_cookie(&response).unwrap();
    let auth = BasicAuth::from_token(&session_cookie);

    // Access protected endpoint
    let response = app.get("/api/v1/auth/me", Some(auth)).await;
    assert_eq!(response.status(), 200);

    // Cleanup
    app.cleanup().await;
}
```

**4. Test Data Factories:**
```rust
// tests/helpers/test_data.rs
use uuid::Uuid;
use serde_json::json;

pub struct TestDataFactory {
    pub app: TestApp,
}

impl TestDataFactory {
    pub fn new(app: TestApp) -> Self {
        Self { app }
    }

    pub async fn create_user(&self, username: &str) -> serde_json::Value {
        let user_data = json!({
            "username": username,
            "email": format("{}@example.com", username),
            "password": "SecurePass123!"
        });

        let response = self.app
            .post_json("/api/v1/auth/register", None, &user_data)
            .await;

        assert_eq!(response.status(), 201);
        response.json().await.unwrap()
    }

    pub async fn create_authenticated_user(&self, username: &str) -> BasicAuth {
        self.create_user(username).await;
        
        let login_data = json!({
            "username": username,
            "password": "SecurePass123!"
        });

        let response = self.app
            .post_json("/api/v1/auth/login", None, &login_data)
            .await;

        let session_cookie = self.app.extract_session_cookie(&response).unwrap();
        BasicAuth::from_token(&session_cookie)
    }
}
```

### Key Benefits

**Real Environment Testing:**
- Tests actual HTTP endpoints, not mocked interfaces
- Same request/response cycle as production
- Catches serialization, routing, and middleware issues

**Isolation & Parallelism:**
- Each test gets unique database and server port
- Tests can run concurrently without interference
- No shared state between tests

**Developer Productivity:**
- Easy to debug with real HTTP requests
- Can use browser dev tools or curl to inspect
- Helper methods reduce boilerplate

**Production Similarity:**
- Exercises same code paths as production
- Tests middleware, authentication, and error handling
- Validates complete request lifecycle

## 2. Test Database Optimization

**The Problem**: You've embraced integration testing, but now your test suite takes forever to run. Each test creates a fresh database, runs migrations, seeds data—multiplied by hundreds of tests, your CI pipeline crawls. Developers stop running tests locally because they're too slow.

**The First Principle**: Database setup is expensive, but database copying is cheap. Instead of paying the migration cost for every test, pay it once and clone the result. It's like having a template that you photocopy instead of rewriting from scratch every time.

**Why It Matters**: Fast tests change developer behavior. When tests run in seconds, developers run them constantly, catch bugs early, and feel confident making changes. Slow tests become obstacles that teams work around, defeating the entire purpose of having them.

### World-Class Insights

**📊 The 30-Second Rule**: Research shows that test suites taking longer than 30 seconds see a 40% drop in developer usage. Every second matters—optimize ruthlessly to stay under this threshold.

**🎭 Template vs. Transaction Rollback**: While transaction rollback is faster for individual tests, template databases scale better with parallel execution. Templates avoid lock contention and enable true test isolation across multiple processes.

**💾 Memory vs. Disk Trade-offs**: Template databases use more disk space but reduce memory pressure compared to in-memory databases. For large test suites, this trade-off often favors templates because they're more predictable and stable.

**🔄 Connection Pool Sizing Strategy**:
- Template creation: Single connection to avoid lock contention
- Test execution: 2-3 connections per test process maximum
- Monitor `pg_stat_activity` to catch connection leaks early
- Use `pgbouncer` in test environments for connection multiplexing

**⚙️ Advanced Optimization Techniques**:
- Pre-warm template databases with realistic data distributions
- Use `UNLOGGED` tables in templates for 3x faster writes (acceptable in tests)
- Partition large tables in templates to improve copy performance
- Consider `pg_dump`/`pg_restore` for extremely large datasets

**🔍 Debugging Template Issues**:
- Log template creation timestamps to catch recreation loops
- Monitor template database sizes to detect bloat
- Use `SELECT pg_database_size()` to track template growth over time
- Implement template health checks with connection tests

### Solution: Template Database Pattern

**tests/helpers/db.rs**
```rust
use once_cell::sync::Lazy;
use sqlx::PgPool;
use std::sync::atomic::{AtomicU32, AtomicBool, Ordering};
use tokio::sync::Semaphore;

// Global counter for unique database names
static DB_COUNTER: AtomicU32 = AtomicU32::new(0);

// Template database configuration
static TEMPLATE_DB_NAME: &str = "myapp_test_template";
static TEMPLATE_CREATED: AtomicBool = AtomicBool::new(false);

// Limit concurrent database operations
static DB_SEMAPHORE: Lazy<Semaphore> = Lazy::new(|| Semaphore::new(5));

/// Ensures template database exists with all migrations applied
async fn ensure_template_db(config: &AppConfig) -> Result<(), Error> {
    // Check if already created by another test
    if TEMPLATE_CREATED.load(Ordering::Acquire) {
        return Ok(());
    }

    let pool = PgPool::connect(&config.database_url_without_db()).await?;

    // Check if template exists in PostgreSQL
    let exists: bool = sqlx::query_scalar(
        "SELECT EXISTS(SELECT 1 FROM pg_database WHERE datname = $1)"
    )
    .bind(TEMPLATE_DB_NAME)
    .fetch_one(&pool)
    .await?;

    if exists {
        TEMPLATE_CREATED.store(true, Ordering::Release);
        return Ok(());
    }

    // Try to create template (handle race condition)
    match sqlx::query(&format!("CREATE DATABASE \"{}\"", TEMPLATE_DB_NAME))
        .execute(&pool)
        .await
    {
        Ok(_) => {
            // We created it, now run migrations
            let template_url = format!("{}/{}", config.database_url_without_db(), TEMPLATE_DB_NAME);
            let template_pool = PgPool::connect(&template_url).await?;
            
            sqlx::migrate!("./migrations")
                .run(&template_pool)
                .await?;
            
            TEMPLATE_CREATED.store(true, Ordering::Release);
            tracing::info!("Template database created and migrated");
        }
        Err(sqlx::Error::Database(e)) if e.code() == Some("23505".into()) => {
            // Another process created it
            TEMPLATE_CREATED.store(true, Ordering::Release);
        }
        Err(e) => return Err(e.into()),
    }

    Ok(())
}

/// Creates a test database by cloning the template
pub async fn create_test_db(config: &AppConfig) -> Result<TestDatabase, Error> {
    // Limit concurrent operations
    let _permit = DB_SEMAPHORE.acquire().await?;
    
    // Ensure template exists
    ensure_template_db(config).await?;
    
    // Generate unique database name
    let counter = DB_COUNTER.fetch_add(1, Ordering::SeqCst);
    let db_name = format!("test_{}_{}", std::process::id(), counter);
    
    let pool = PgPool::connect(&config.database_url_without_db()).await?;
    
    // Clone from template (much faster than create + migrate)
    sqlx::query(&format!(
        "CREATE DATABASE \"{}\" WITH TEMPLATE \"{}\"",
        db_name, TEMPLATE_DB_NAME
    ))
    .execute(&pool)
    .await?;
    
    // Connect to new test database
    let test_url = format!("{}/{}", config.database_url_without_db(), db_name);
    let test_pool = PgPool::connect(&test_url).await?;
    
    Ok(TestDatabase {
        name: db_name,
        pool: test_pool,
        _cleanup_pool: pool,
    })
}

pub struct TestDatabase {
    pub name: String,
    pub pool: PgPool,
    _cleanup_pool: PgPool,
}

impl Drop for TestDatabase {
    fn drop(&mut self) {
        // Schedule cleanup in background
        let name = self.name.clone();
        let pool = self._cleanup_pool.clone();
        
        tokio::spawn(async move {
            // Terminate connections
            let _ = sqlx::query(&format!(
                "SELECT pg_terminate_backend(pid) 
                 FROM pg_stat_activity 
                 WHERE datname = '{}' AND pid <> pg_backend_pid()",
                name
            ))
            .execute(&pool)
            .await;
            
            // Drop database
            let _ = sqlx::query(&format!("DROP DATABASE IF EXISTS \"{}\"", name))
                .execute(&pool)
                .await;
        });
    }
}
```

**Usage in Tests:**
```rust
#[tokio::test]
async fn test_user_creation() {
    let test_db = create_test_db(&config).await.unwrap();
    let mut conn = test_db.pool.acquire().await.unwrap();
    
    // Run your test
    let user = create_user(&mut conn, CreateUserRequest {
        username: "testuser".to_string(),
        email: "test@example.com".to_string(),
        password: "SecurePass123!".to_string(),
    }).await.unwrap();
    
    assert_eq!(user.username, "testuser");
    // Database automatically cleaned up when test_db drops
}
```

### Benefits
- **10x faster** - Database cloning vs create + migrate
- **Resource efficient** - Semaphore limits concurrent operations
- **Race condition safe** - Atomic checks handle concurrent template creation
- **Auto cleanup** - Drop trait handles database removal
- **PostgreSQL optimized** - Uses native template feature

## 3. Repository Pattern Implementation

**The Problem**: Your service functions are littered with SQL queries, making them hard to test and impossible to change without touching business logic. Database concerns leak into your domain, violating the single responsibility principle. When you want to switch from PostgreSQL to something else, you're facing a major refactor.

**The First Principle**: Separate what from how. Your business logic should care about users and orders, not JOIN clauses and database connections. The repository pattern creates a clean boundary between your domain and your data storage, making both sides easier to understand, test, and modify.

**Why It Matters**: Clean separation enables independent evolution. You can optimize queries without touching business logic, switch databases without changing domain code, and test business rules without setting up databases. It's the difference between a monolithic ball of mud and a well-architected system.

### World-Class Insights

**🏗️ Function vs. Struct Repositories**: While many examples show struct-based repositories, function-based approaches are often superior in Rust. They're easier to test, compose better, and avoid the "repository as a god object" antipattern.

**🔍 Query Optimization Strategy**: Keep repositories thin but smart. They should handle data access patterns (pagination, filtering, sorting) but delegate business logic to services. Think "smart queries, dumb transformations."

**💡 The N+1 Problem Solution**: Use `DataLoader` patterns or batching functions within repositories to avoid N+1 queries. A well-designed repository anticipates common access patterns and provides efficient bulk operations.

**🎯 Repository Granularity Rules**:
- One repository per aggregate root (DDD concept)
- Avoid cross-aggregate repositories that encourage business logic leakage
- Create specialized query repositories for complex read operations
- Use composition over inheritance for shared repository behavior

**⚡ Performance Optimization Secrets**:
- Use `EXPLAIN ANALYZE` to validate query plans during development
- Implement query result caching at the repository level
- Pre-load related data using `JOIN`s instead of multiple round trips
- Monitor slow query logs and optimize proactively

**🔧 Advanced Patterns**:
- **Repository Decorators**: Wrap repositories with caching, logging, or metrics
- **Specification Pattern**: Build complex queries using composable specifications
- **Unit of Work**: Coordinate transactions across multiple repositories
- **Read/Write Splitting**: Separate repositories for commands vs. queries

**🧪 Testing Strategies**:
- Test repositories against real databases, not mocks
- Use property-based testing for edge cases (empty results, large datasets)
- Verify database constraints are enforced correctly
- Test concurrent access patterns and race conditions

### Solution: Function-Based Repository Layer

**Example: Complete Blog Post Module**

**src/posts/models.rs**
```rust
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use sqlx::FromRow;
use uuid::Uuid;

#[derive(Debug, Clone, FromRow, Serialize)]
pub struct Post {
    pub id: Uuid,
    pub author_id: Uuid,
    pub title: String,
    pub slug: String,
    pub content: String,
    pub excerpt: Option<String>,
    pub published: bool,
    pub published_at: Option<DateTime<Utc>>,
    pub view_count: i32,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize)]
pub struct CreatePostRequest {
    pub title: String,
    pub content: String,
    pub excerpt: Option<String>,
    pub published: bool,
}

#[derive(Debug, Deserialize)]
pub struct UpdatePostRequest {
    pub title: Option<String>,
    pub content: Option<String>,
    pub excerpt: Option<String>,
    pub published: Option<bool>,
}

#[derive(Debug, Deserialize)]
pub struct PostQuery {
    pub author_id: Option<Uuid>,
    pub published: Option<bool>,
    pub search: Option<String>,
    pub page: Option<u32>,
    pub limit: Option<u32>,
}
```

**src/posts/repositories.rs**
```rust
use crate::{DbConn, Error, Result};
use crate::posts::models::{Post, PostQuery};
use crate::shared::pagination::{PageRequest, PaginatedResponse};
use slug::slugify;
use uuid::Uuid;

/// Creates a new post
pub async fn create_post(
    conn: &mut DbConn,
    author_id: Uuid,
    title: &str,
    content: &str,
    excerpt: Option<&str>,
    published: bool,
) -> Result<Post> {
    let slug = generate_unique_slug(conn, title).await?;
    let published_at = if published { Some(chrono::Utc::now()) } else { None };
    
    let post = sqlx::query_as!(
        Post,
        r#"
        INSERT INTO posts (
            id, author_id, title, slug, content, excerpt, 
            published, published_at, view_count, created_at, updated_at
        )
        VALUES ($1, $2, $3, $4, $5, $6, $7, $8, 0, NOW(), NOW())
        RETURNING *
        "#,
        Uuid::now_v7(),
        author_id,
        title,
        slug,
        content,
        excerpt,
        published,
        published_at
    )
    .fetch_one(conn)
    .await?;

    Ok(post)
}

/// Finds a post by ID
pub async fn find_post_by_id(
    conn: &mut DbConn,
    id: Uuid,
) -> Result<Option<Post>> {
    let post = sqlx::query_as!(
        Post,
        "SELECT * FROM posts WHERE id = $1",
        id
    )
    .fetch_optional(conn)
    .await?;

    Ok(post)
}

/// Finds a post by slug
pub async fn find_post_by_slug(
    conn: &mut DbConn,
    slug: &str,
) -> Result<Option<Post>> {
    let post = sqlx::query_as!(
        Post,
        "SELECT * FROM posts WHERE slug = $1",
        slug
    )
    .fetch_optional(conn)
    .await?;

    Ok(post)
}

/// Lists posts with pagination and filtering
pub async fn list_posts(
    conn: &mut DbConn,
    query: &PostQuery,
) -> Result<PaginatedResponse<Post>> {
    let page = PageRequest {
        page: query.page.unwrap_or(1),
        limit: query.limit.unwrap_or(20),
    };
    let offset = page.offset();
    let limit = page.limit();

    // Build dynamic query
    let mut conditions = vec!["1=1"];
    let mut params = vec![];
    
    if let Some(author_id) = query.author_id {
        conditions.push("author_id = $1");
        params.push(author_id.to_string());
    }
    
    if let Some(published) = query.published {
        conditions.push("published = $2");
        params.push(published.to_string());
    }
    
    if let Some(search) = &query.search {
        conditions.push("(title ILIKE $3 OR content ILIKE $3)");
        params.push(format!("%{}%", search));
    }

    let where_clause = conditions.join(" AND ");
    
    // Count total items
    let count_query = format!("SELECT COUNT(*) FROM posts WHERE {}", where_clause);
    let total: i64 = sqlx::query_scalar(&count_query)
        .fetch_one(&mut *conn)
        .await?;
    
    // Fetch paginated results
    let query = format!(
        "SELECT * FROM posts WHERE {} ORDER BY created_at DESC LIMIT {} OFFSET {}",
        where_clause, limit, offset
    );
    
    let posts = sqlx::query_as::<_, Post>(&query)
        .fetch_all(conn)
        .await?;

    Ok(PaginatedResponse::new(posts, total as u64, page))
}

/// Updates a post
pub async fn update_post(
    conn: &mut DbConn,
    id: Uuid,
    updates: UpdatePostFields,
) -> Result<Post> {
    let mut query = "UPDATE posts SET updated_at = NOW()".to_string();
    
    if let Some(title) = updates.title {
        query.push_str(&format!(", title = '{}'", title));
        if updates.update_slug {
            let slug = generate_unique_slug(conn, &title).await?;
            query.push_str(&format!(", slug = '{}'", slug));
        }
    }
    
    if let Some(content) = updates.content {
        query.push_str(&format!(", content = '{}'", content));
    }
    
    if let Some(excerpt) = updates.excerpt {
        query.push_str(&format!(", excerpt = '{}'", excerpt));
    }
    
    if let Some(published) = updates.published {
        query.push_str(&format!(", published = {}", published));
        if published && updates.update_published_at {
            query.push_str(", published_at = NOW()");
        }
    }
    
    query.push_str(&format!(" WHERE id = '{}' RETURNING *", id));
    
    let post = sqlx::query_as::<_, Post>(&query)
        .fetch_one(conn)
        .await
        .map_err(|e| match e {
            sqlx::Error::RowNotFound => Error::NotFound,
            _ => Error::Database(e),
        })?;

    Ok(post)
}

/// Increments post view count
pub async fn increment_view_count(
    conn: &mut DbConn,
    id: Uuid,
) -> Result<()> {
    sqlx::query!(
        "UPDATE posts SET view_count = view_count + 1 WHERE id = $1",
        id
    )
    .execute(conn)
    .await?;
    
    Ok(())
}

/// Deletes a post
pub async fn delete_post(
    conn: &mut DbConn,
    id: Uuid,
) -> Result<()> {
    let result = sqlx::query!(
        "DELETE FROM posts WHERE id = $1",
        id
    )
    .execute(conn)
    .await?;
    
    if result.rows_affected() == 0 {
        return Err(Error::NotFound);
    }
    
    Ok(())
}

/// Generates a unique slug for a title
async fn generate_unique_slug(
    conn: &mut DbConn,
    title: &str,
) -> Result<String> {
    let base_slug = slugify(title);
    let mut slug = base_slug.clone();
    let mut counter = 1;
    
    // Keep trying until we find a unique slug
    while find_post_by_slug(conn, &slug).await?.is_some() {
        slug = format!("{}-{}", base_slug, counter);
        counter += 1;
    }
    
    Ok(slug)
}

pub struct UpdatePostFields {
    pub title: Option<String>,
    pub content: Option<String>,
    pub excerpt: Option<String>,
    pub published: Option<bool>,
    pub update_slug: bool,
    pub update_published_at: bool,
}
```

**src/posts/services.rs**
```rust
use crate::{DbConn, Error, Result};
use crate::posts::models::{Post, CreatePostRequest, UpdatePostRequest, PostQuery};
use crate::posts::repositories::{self, UpdatePostFields};
use crate::shared::pagination::PaginatedResponse;
use crate::shared::validation::{validate_non_empty_string, validate_string_length};
use uuid::Uuid;

/// Creates a new blog post
pub async fn create_post(
    conn: &mut DbConn,
    author_id: Uuid,
    req: CreatePostRequest,
) -> Result<Post> {
    // Validate input
    validate_non_empty_string(&req.title, "title")?;
    validate_string_length(&req.title, "title", 1, 200)?;
    validate_non_empty_string(&req.content, "content")?;
    
    if let Some(excerpt) = &req.excerpt {
        validate_string_length(excerpt, "excerpt", 0, 500)?;
    }
    
    // Create post
    repositories::create_post(
        conn,
        author_id,
        &req.title,
        &req.content,
        req.excerpt.as_deref(),
        req.published,
    ).await
}

/// Gets a post by ID (increments view count if published)
pub async fn get_post(
    conn: &mut DbConn,
    id: Uuid,
    viewer_id: Option<Uuid>,
) -> Result<Post> {
    let post = repositories::find_post_by_id(conn, id)
        .await?
        .ok_or(Error::NotFound)?;
    
    // Only increment views for published posts by other users
    if post.published && viewer_id != Some(post.author_id) {
        repositories::increment_view_count(conn, id).await?;
    }
    
    Ok(post)
}

/// Gets a post by slug (for public viewing)
pub async fn get_post_by_slug(
    conn: &mut DbConn,
    slug: &str,
) -> Result<Post> {
    let post = repositories::find_post_by_slug(conn, slug)
        .await?
        .ok_or(Error::NotFound)?;
    
    // Only return published posts for public access
    if !post.published {
        return Err(Error::NotFound);
    }
    
    // Increment view count
    repositories::increment_view_count(conn, post.id).await?;
    
    Ok(post)
}

/// Lists posts with filtering
pub async fn list_posts(
    conn: &mut DbConn,
    query: PostQuery,
    viewer_id: Option<Uuid>,
) -> Result<PaginatedResponse<Post>> {
    // Non-authors can only see published posts
    let mut filtered_query = query;
    if viewer_id.is_none() || filtered_query.author_id != viewer_id {
        filtered_query.published = Some(true);
    }
    
    repositories::list_posts(conn, &filtered_query).await
}

/// Updates a post
pub async fn update_post(
    conn: &mut DbConn,
    id: Uuid,
    author_id: Uuid,
    req: UpdatePostRequest,
) -> Result<Post> {
    // Verify ownership
    let post = repositories::find_post_by_id(conn, id)
        .await?
        .ok_or(Error::NotFound)?;
    
    if post.author_id != author_id {
        return Err(Error::Unauthorized);
    }
    
    // Validate updates
    if let Some(title) = &req.title {
        validate_non_empty_string(title, "title")?;
        validate_string_length(title, "title", 1, 200)?;
    }
    
    if let Some(content) = &req.content {
        validate_non_empty_string(content, "content")?;
    }
    
    // Update post
    let update_fields = UpdatePostFields {
        title: req.title,
        content: req.content,
        excerpt: req.excerpt,
        published: req.published,
        update_slug: req.title.is_some(),
        update_published_at: req.published == Some(true) && !post.published,
    };
    
    repositories::update_post(conn, id, update_fields).await
}

/// Deletes a post
pub async fn delete_post(
    conn: &mut DbConn,
    id: Uuid,
    author_id: Uuid,
) -> Result<()> {
    // Verify ownership
    let post = repositories::find_post_by_id(conn, id)
        .await?
        .ok_or(Error::NotFound)?;
    
    if post.author_id != author_id {
        return Err(Error::Unauthorized);
    }
    
    repositories::delete_post(conn, id).await
}
```

### Benefits
- **Clear separation** - Repository functions handle SQL, service functions handle business logic
- **Testability** - Easy to mock repository functions in tests
- **Reusability** - Repository functions can be used by multiple services
- **Type safety** - SQLx compile-time query verification
- **Flexibility** - Can easily add caching or change data source

## 4. Shared Module Architecture

**The Problem**: Every domain module is reinventing the wheel. Password hashing logic appears in three places with subtle differences. Pagination is implemented six different ways. Error handling is inconsistent across endpoints. Your codebase is drowning in duplication, and maintaining consistency feels impossible.

**The First Principle**: DRY isn't just about reducing lines of code—it's about creating single sources of truth. When business rules, utilities, or cross-cutting concerns exist in multiple places, they inevitably diverge, creating bugs and confusion. Centralization creates consistency and reduces cognitive load.

**Why It Matters**: Shared modules are force multipliers for your team. Fix a bug once, it's fixed everywhere. Improve performance once, everything gets faster. Add a feature once, all domains benefit. It's the difference between managing ten different password policies and managing one authoritative implementation.

### World-Class Insights

**🎭 The Shared Module Paradox**: Shared code reduces duplication but increases coupling. The key is sharing behavior and interfaces, not implementations. A well-designed shared module provides flexible abstractions that domains can adapt, not rigid implementations they must accept.

**📦 Dependency Direction Rules**: Shared modules should depend on nothing except standard library and fundamental crates. Domains depend on shared modules, never the reverse. This prevents circular dependencies and keeps shared code focused and reusable.

**🔄 Evolution Strategy**: Start with duplication, extract patterns once you see them three times, then continuously refactor toward more general solutions. Premature abstraction is worse than no abstraction—let usage patterns emerge naturally.

**🎯 What Belongs in Shared Modules**:
- **Pure functions** with no side effects (validation, formatting, calculation)
- **Data types** used across multiple domains (errors, pagination, common DTOs)
- **Infrastructure concerns** (database connections, HTTP clients, configuration)
- **Cross-cutting behaviors** (logging, metrics, authentication)

**🚫 What Doesn't Belong**:
- Domain-specific business logic (even if similar across domains)
- Stateful objects that maintain domain state
- Implementation details that frequently change
- Code used by only one domain (even if it might be reused later)

**⚙️ Advanced Organization Techniques**:
- **Feature flags**: Control shared behavior rollout across domains
- **Versioning**: Use semantic versioning for breaking changes in shared APIs
- **Interface segregation**: Provide multiple small interfaces instead of one large one
- **Plugin architecture**: Allow domains to extend shared behavior through traits

**🔧 Performance Optimization**:
- Use `Arc<T>` for expensive-to-clone shared resources
- Implement lazy initialization for optional dependencies
- Cache computationally expensive shared operations
- Profile shared code paths as they affect all domains

### Solution: Comprehensive Shared Module

**src/shared/types.rs**
```rust
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use uuid::Uuid;

/// Wrapper type for user IDs with additional validation
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub struct UserId(pub Uuid);

impl UserId {
    pub fn new() -> Self {
        Self(Uuid::now_v7())
    }
}

impl From<Uuid> for UserId {
    fn from(id: Uuid) -> Self {
        Self(id)
    }
}

/// Common timestamp fields for all entities
#[derive(Debug, Clone, Serialize)]
pub struct Timestamps {
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

/// Represents a date range for filtering
#[derive(Debug, Deserialize)]
pub struct DateRange {
    pub start: Option<DateTime<Utc>>,
    pub end: Option<DateTime<Utc>>,
}

impl DateRange {
    pub fn validate(&self) -> Result<(), crate::Error> {
        if let (Some(start), Some(end)) = (self.start, self.end) {
            if start > end {
                return Err(crate::Error::ValidationFailed {
                    field: "date_range".to_string(),
                    message: "Start date must be before end date".to_string(),
                });
            }
        }
        Ok(())
    }
}

/// Represents a monetary amount in cents
#[derive(Debug, Clone, Copy, Serialize, Deserialize)]
pub struct Money {
    cents: i64,
}

impl Money {
    pub fn from_cents(cents: i64) -> Self {
        Self { cents }
    }
    
    pub fn from_dollars(dollars: f64) -> Self {
        Self {
            cents: (dollars * 100.0).round() as i64,
        }
    }
    
    pub fn to_dollars(&self) -> f64 {
        self.cents as f64 / 100.0
    }
    
    pub fn add(&self, other: Money) -> Money {
        Money::from_cents(self.cents + other.cents)
    }
}

/// Represents a percentage (0-100)
#[derive(Debug, Clone, Copy, Serialize, Deserialize)]
pub struct Percentage(f64);

impl Percentage {
    pub fn new(value: f64) -> Result<Self, crate::Error> {
        if !(0.0..=100.0).contains(&value) {
            return Err(crate::Error::ValidationFailed {
                field: "percentage".to_string(),
                message: "Percentage must be between 0 and 100".to_string(),
            });
        }
        Ok(Self(value))
    }
    
    pub fn value(&self) -> f64 {
        self.0
    }
    
    pub fn as_decimal(&self) -> f64 {
        self.0 / 100.0
    }
}
```

**src/shared/pagination.rs**
```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Deserialize)]
pub struct PageRequest {
    pub page: u32,
    pub limit: u32,
}

impl PageRequest {
    pub fn offset(&self) -> u32 {
        (self.page.saturating_sub(1)) * self.limit
    }
    
    pub fn limit(&self) -> u32 {
        self.limit.min(100) // Cap at 100
    }
}

impl Default for PageRequest {
    fn default() -> Self {
        Self { page: 1, limit: 20 }
    }
}

#[derive(Debug, Serialize)]
pub struct PaginatedResponse<T> {
    pub items: Vec<T>,
    pub total: u64,
    pub page: u32,
    pub limit: u32,
    pub total_pages: u32,
}

impl<T> PaginatedResponse<T> {
    pub fn new(items: Vec<T>, total: u64, page_request: PageRequest) -> Self {
        let limit = page_request.limit();
        let total_pages = ((total as f64) / (limit as f64)).ceil() as u32;
        
        Self {
            items,
            total,
            page: page_request.page,
            limit,
            total_pages,
        }
    }
}
```

**src/shared/events.rs**
```rust
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use serde_json::Value;
use uuid::Uuid;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct DomainEvent {
    pub id: Uuid,
    pub event_type: EventType,
    pub aggregate_id: Uuid,
    pub user_id: Option<Uuid>,
    pub payload: Value,
    pub metadata: EventMetadata,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct EventMetadata {
    pub ip_address: Option<String>,
    pub user_agent: Option<String>,
    pub correlation_id: Option<Uuid>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum EventType {
    // User events
    UserRegistered,
    UserLoggedIn,
    UserLoggedOut,
    UserUpdated,
    UserDeleted,
    PasswordChanged,
    
    // Post events
    PostCreated,
    PostPublished,
    PostUpdated,
    PostDeleted,
    PostViewed,
    
    // System events
    SystemStarted,
    SystemShutdown,
    DatabaseMigrated,
}

impl DomainEvent {
    pub fn new(
        event_type: EventType,
        aggregate_id: Uuid,
        user_id: Option<Uuid>,
        payload: Value,
    ) -> Self {
        Self {
            id: Uuid::now_v7(),
            event_type,
            aggregate_id,
            user_id,
            payload,
            metadata: EventMetadata::default(),
            created_at: Utc::now(),
        }
    }
    
    pub fn with_metadata(mut self, metadata: EventMetadata) -> Self {
        self.metadata = metadata;
        self
    }
}

impl Default for EventMetadata {
    fn default() -> Self {
        Self {
            ip_address: None,
            user_agent: None,
            correlation_id: None,
        }
    }
}

/// Event publisher trait for different implementations
#[async_trait::async_trait]
pub trait EventPublisher: Send + Sync {
    async fn publish(&self, event: DomainEvent) -> Result<(), crate::Error>;
    async fn publish_batch(&self, events: Vec<DomainEvent>) -> Result<(), crate::Error>;
}

/// Database-backed event store
pub struct DatabaseEventPublisher {
    pool: sqlx::PgPool,
}

#[async_trait::async_trait]
impl EventPublisher for DatabaseEventPublisher {
    async fn publish(&self, event: DomainEvent) -> Result<(), crate::Error> {
        sqlx::query!(
            r#"
            INSERT INTO domain_events (
                id, event_type, aggregate_id, user_id, 
                payload, metadata, created_at
            )
            VALUES ($1, $2, $3, $4, $5, $6, $7)
            "#,
            event.id,
            serde_json::to_string(&event.event_type)?,
            event.aggregate_id,
            event.user_id,
            event.payload,
            serde_json::to_value(&event.metadata)?,
            event.created_at
        )
        .execute(&self.pool)
        .await?;
        
        Ok(())
    }
    
    async fn publish_batch(&self, events: Vec<DomainEvent>) -> Result<(), crate::Error> {
        let mut tx = self.pool.begin().await?;
        
        for event in events {
            sqlx::query!(
                r#"
                INSERT INTO domain_events (
                    id, event_type, aggregate_id, user_id, 
                    payload, metadata, created_at
                )
                VALUES ($1, $2, $3, $4, $5, $6, $7)
                "#,
                event.id,
                serde_json::to_string(&event.event_type)?,
                event.aggregate_id,
                event.user_id,
                event.payload,
                serde_json::to_value(&event.metadata)?,
                event.created_at
            )
            .execute(&mut *tx)
            .await?;
        }
        
        tx.commit().await?;
        Ok(())
    }
}
```

**src/shared/cache.rs**
```rust
use std::sync::Arc;
use std::time::Duration;
use moka::future::Cache;
use serde::{Deserialize, Serialize};

/// Generic cache wrapper for any serializable type
pub struct TypedCache<T> {
    inner: Cache<String, Arc<T>>,
}

impl<T: Send + Sync + 'static> TypedCache<T> {
    pub fn new(max_capacity: u64, ttl: Duration) -> Self {
        let cache = Cache::builder()
            .max_capacity(max_capacity)
            .time_to_live(ttl)
            .build();
            
        Self { inner: cache }
    }
    
    pub async fn get(&self, key: &str) -> Option<Arc<T>> {
        self.inner.get(key).await
    }
    
    pub async fn insert(&self, key: String, value: T) {
        self.inner.insert(key, Arc::new(value)).await;
    }
    
    pub async fn invalidate(&self, key: &str) {
        self.inner.invalidate(key).await;
    }
    
    pub async fn clear(&self) {
        self.inner.invalidate_all().await;
    }
}

/// Cache key builder for consistent key generation
pub struct CacheKey;

impl CacheKey {
    pub fn user(id: Uuid) -> String {
        format!("user:{}", id)
    }
    
    pub fn post(id: Uuid) -> String {
        format!("post:{}", id)
    }
    
    pub fn post_by_slug(slug: &str) -> String {
        format!("post:slug:{}", slug)
    }
    
    pub fn user_posts(user_id: Uuid, page: u32) -> String {
        format!("user:{}:posts:page:{}", user_id, page)
    }
}
```

### Usage Examples

**Using shared types in services:**
```rust
use crate::shared::types::{UserId, Money, DateRange};
use crate::shared::events::{DomainEvent, EventType, EventPublisher};

pub async fn process_payment(
    conn: &mut DbConn,
    user_id: UserId,
    amount: Money,
    event_publisher: &dyn EventPublisher,
) -> Result<Payment> {
    // Process payment...
    let payment = create_payment(conn, user_id.0, amount).await?;
    
    // Publish event
    let event = DomainEvent::new(
        EventType::PaymentProcessed,
        payment.id,
        Some(user_id.0),
        serde_json::json!({
            "amount_cents": amount.to_cents(),
            "amount_dollars": amount.to_dollars(),
        }),
    );
    
    event_publisher.publish(event).await?;
    
    Ok(payment)
}
```

### Benefits
- **Type safety** - Strongly typed wrappers prevent errors
- **Consistency** - Same validation and behavior everywhere
- **Reusability** - Share complex logic across modules
- **Maintainability** - Single source of truth for common code

## 5. Unified Error Handling

**The Problem**: Your application has dozens of error types scattered across modules. Converting between them requires endless `From` implementations and `.map_err()` calls. API clients receive inconsistent error formats. Debugging requires hunting through multiple error hierarchies to understand what actually went wrong.

**The First Principle**: Errors are data, and like all data, they benefit from consistency. A unified error type creates a common language for failure across your entire application. When everything speaks the same error protocol, composition becomes natural and debugging becomes straightforward.

**Why It Matters**: Error handling quality directly impacts user experience and developer productivity. Consistent errors mean consistent responses, better logging, and easier debugging. When errors follow patterns, both humans and machines can understand and respond to them predictably.

### World-Class Insights

**🎯 Error Design Philosophy**: Errors are part of your API contract. They should be as carefully designed as your success responses. Great error handling makes debugging feel like detective work with good clues, not archaeology with broken artifacts.

**📊 The Error Information Hierarchy**:
1. **Context**: What were you trying to do?
2. **Cause**: What specifically went wrong? 
3. **Effect**: What's the impact on the user/system?
4. **Resolution**: What can be done about it?

**🔒 Security-First Error Design**: Never leak implementation details in user-facing errors. Create two error representations—one rich for logging/debugging, one sanitized for users. Attackers often probe error messages for system information.

**⚡ Performance Considerations**:
- Error creation should be zero-allocation in hot paths
- Use `&'static str` for fixed error messages instead of `String`
- Implement `Clone` and `Copy` where possible to avoid heap allocations
- Consider error codes instead of messages for high-frequency errors

**🔧 Advanced Error Patterns**:
- **Error Aggregation**: Collect multiple validation errors into a single response
- **Error Recovery**: Implement retry logic with exponential backoff
- **Error Contextualization**: Add operation-specific context as errors bubble up
- **Error Metrics**: Track error frequencies and patterns for system health

**🎭 Error User Experience**:
- Use error codes that support internationalization
- Provide actionable error messages ("check your email format" vs "invalid input")
- Include correlation IDs for support requests
- Differentiate between temporary and permanent failures

**🧪 Testing Error Paths**:
- Test error serialization/deserialization thoroughly
- Verify error logging doesn't leak sensitive data
- Use property-based testing for error edge cases
- Simulate failure modes in integration tests

### Solution: Single Error Enum

**src/error.rs**
```rust
use axum::{
    response::{IntoResponse, Response},
    http::StatusCode,
    Json,
};
use serde::Serialize;
use std::fmt;
use uuid::Uuid;

#[derive(Debug, thiserror::Error)]
pub enum Error {
    // Database errors
    #[error("Database error")]
    Database(#[from] sqlx::Error),
    
    #[error("Database connection failed")]
    DatabaseConnection(String),
    
    // Authentication & Authorization
    #[error("Authentication required")]
    Unauthorized,
    
    #[error("Invalid credentials")]
    InvalidCredentials,
    
    #[error("Insufficient permissions")]
    Forbidden,
    
    #[error("Session expired")]
    SessionExpired,
    
    // Validation errors
    #[error("Validation failed for {field}: {message}")]
    ValidationFailed {
        field: String,
        message: String,
    },
    
    #[error("Multiple validation errors")]
    ValidationErrors(Vec<ValidationError>),
    
    // Resource errors
    #[error("Resource not found")]
    NotFound,
    
    #[error("Resource already exists")]
    Conflict(String),
    
    // Business logic errors
    #[error("Operation not allowed: {0}")]
    BusinessRule(String),
    
    #[error("Rate limit exceeded")]
    RateLimitExceeded {
        retry_after_seconds: u64,
    },
    
    // External service errors
    #[error("External service error: {service}")]
    ExternalService {
        service: String,
        message: String,
    },
    
    // System errors
    #[error("Internal server error")]
    Internal(String),
    
    #[error("Service temporarily unavailable")]
    ServiceUnavailable,
    
    // Serialization errors
    #[error("Serialization error")]
    Serialization(#[from] serde_json::Error),
}

#[derive(Debug, Serialize)]
pub struct ValidationError {
    pub field: String,
    pub message: String,
}

// Convert to HTTP responses
impl IntoResponse for Error {
    fn into_response(self) -> Response {
        let (status, error_response) = match self {
            // 400 Bad Request
            Error::ValidationFailed { field, message } => (
                StatusCode::BAD_REQUEST,
                ErrorResponse::new("validation_failed")
                    .with_field_error(&field, &message),
            ),
            Error::ValidationErrors(errors) => (
                StatusCode::BAD_REQUEST,
                ErrorResponse::new("validation_failed")
                    .with_validation_errors(errors),
            ),
            
            // 401 Unauthorized
            Error::Unauthorized => (
                StatusCode::UNAUTHORIZED,
                ErrorResponse::new("unauthorized")
                    .with_message("Authentication required"),
            ),
            Error::InvalidCredentials => (
                StatusCode::UNAUTHORIZED,
                ErrorResponse::new("invalid_credentials")
                    .with_message("Invalid username or password"),
            ),
            Error::SessionExpired => (
                StatusCode::UNAUTHORIZED,
                ErrorResponse::new("session_expired")
                    .with_message("Your session has expired, please login again"),
            ),
            
            // 403 Forbidden
            Error::Forbidden => (
                StatusCode::FORBIDDEN,
                ErrorResponse::new("forbidden")
                    .with_message("You don't have permission to perform this action"),
            ),
            
            // 404 Not Found
            Error::NotFound => (
                StatusCode::NOT_FOUND,
                ErrorResponse::new("not_found")
                    .with_message("The requested resource was not found"),
            ),
            
            // 409 Conflict
            Error::Conflict(message) => (
                StatusCode::CONFLICT,
                ErrorResponse::new("conflict")
                    .with_message(&message),
            ),
            
            // 422 Unprocessable Entity
            Error::BusinessRule(message) => (
                StatusCode::UNPROCESSABLE_ENTITY,
                ErrorResponse::new("business_rule_violation")
                    .with_message(&message),
            ),
            
            // 429 Too Many Requests
            Error::RateLimitExceeded { retry_after_seconds } => {
                let mut response = (
                    StatusCode::TOO_MANY_REQUESTS,
                    ErrorResponse::new("rate_limit_exceeded")
                        .with_message("Too many requests, please slow down"),
                ).into_response();
                
                response.headers_mut().insert(
                    "Retry-After",
                    retry_after_seconds.to_string().parse().unwrap(),
                );
                
                return response;
            },
            
            // 500 Internal Server Error
            Error::Database(e) => {
                tracing::error!("Database error: {:?}", e);
                (
                    StatusCode::INTERNAL_SERVER_ERROR,
                    ErrorResponse::new("database_error")
                        .with_message("A database error occurred"),
                )
            },
            Error::Internal(message) => {
                tracing::error!("Internal error: {}", message);
                (
                    StatusCode::INTERNAL_SERVER_ERROR,
                    ErrorResponse::new("internal_error")
                        .with_message("An internal error occurred"),
                )
            },
            Error::Serialization(e) => {
                tracing::error!("Serialization error: {:?}", e);
                (
                    StatusCode::INTERNAL_SERVER_ERROR,
                    ErrorResponse::new("serialization_error")
                        .with_message("Failed to process data"),
                )
            },
            
            // 502 Bad Gateway
            Error::ExternalService { service, message } => {
                tracing::error!("External service error: {} - {}", service, message);
                (
                    StatusCode::BAD_GATEWAY,
                    ErrorResponse::new("external_service_error")
                        .with_message(&format!("{} service is unavailable", service)),
                )
            },
            
            // 503 Service Unavailable
            Error::ServiceUnavailable => (
                StatusCode::SERVICE_UNAVAILABLE,
                ErrorResponse::new("service_unavailable")
                    .with_message("Service temporarily unavailable, please try again later"),
            ),
            
            Error::DatabaseConnection(message) => {
                tracing::error!("Database connection error: {}", message);
                (
                    StatusCode::SERVICE_UNAVAILABLE,
                    ErrorResponse::new("database_connection_error")
                        .with_message("Unable to connect to database"),
                )
            },
        };
        
        (status, Json(error_response)).into_response()
    }
}

#[derive(Debug, Serialize)]
pub struct ErrorResponse {
    pub error: ErrorDetail,
    pub request_id: String,
}

#[derive(Debug, Serialize)]
pub struct ErrorDetail {
    pub code: String,
    pub message: String,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub field_errors: Option<Vec<ValidationError>>,
}

impl ErrorResponse {
    fn new(code: &str) -> Self {
        Self {
            error: ErrorDetail {
                code: code.to_string(),
                message: String::new(),
                field_errors: None,
            },
            request_id: Uuid::now_v7().to_string(),
        }
    }
    
    fn with_message(mut self, message: &str) -> Self {
        self.error.message = message.to_string();
        self
    }
    
    fn with_field_error(mut self, field: &str, message: &str) -> Self {
        self.error.field_errors = Some(vec![ValidationError {
            field: field.to_string(),
            message: message.to_string(),
        }]);
        self
    }
    
    fn with_validation_errors(mut self, errors: Vec<ValidationError>) -> Self {
        self.error.field_errors = Some(errors);
        self
    }
}

// Helper for converting database constraint violations
impl Error {
    pub fn from_database_error(e: sqlx::Error, context: &str) -> Self {
        match e {
            sqlx::Error::Database(db_err) => {
                if let Some(constraint) = db_err.constraint() {
                    match constraint {
                        "users_email_key" => Error::Conflict("Email already exists".to_string()),
                        "users_username_key" => Error::Conflict("Username already exists".to_string()),
                        _ => Error::Database(sqlx::Error::Database(db_err)),
                    }
                } else {
                    Error::Database(sqlx::Error::Database(db_err))
                }
            }
            sqlx::Error::RowNotFound => Error::NotFound,
            _ => Error::Database(e),
        }
    }
}

pub type Result<T> = std::result::Result<T, Error>;
```

### Usage Examples

**In repository functions:**
```rust
pub async fn create_user(conn: &mut DbConn, data: CreateUser) -> Result<User> {
    sqlx::query_as!(User, "INSERT INTO users ...")
        .fetch_one(conn)
        .await
        .map_err(|e| Error::from_database_error(e, "create_user"))
}
```

**In services with validation:**
```rust
pub async fn transfer_money(
    conn: &mut DbConn,
    from: Uuid,
    to: Uuid,
    amount: Money,
) -> Result<Transfer> {
    // Validation
    if amount.to_cents() <= 0 {
        return Err(Error::ValidationFailed {
            field: "amount".to_string(),
            message: "Amount must be positive".to_string(),
        });
    }
    
    // Business rule
    let balance = get_user_balance(conn, from).await?;
    if balance < amount {
        return Err(Error::BusinessRule(
            "Insufficient balance for transfer".to_string()
        ));
    }
    
    // Perform transfer...
}
```

### Benefits
- **Single error type** - Use `?` everywhere without conversions
- **Consistent API responses** - All errors follow same format
- **Detailed error info** - Request IDs for debugging
- **Proper HTTP codes** - Automatic status code mapping
- **Security conscious** - Don't leak internal details to clients

## 6. Domain Events System

**The Problem**: Your business logic is tangled with side effects. When a user registers, you need to send emails, update analytics, create audit logs, and trigger downstream processes. All this logic lives in your registration function, making it complex, slow, and impossible to test in isolation.

**The First Principle**: Decouple actions from reactions. Your core business logic should focus on business rules, not on orchestrating side effects. Events capture "what happened" and let other parts of the system decide "what to do about it" independently.

**Why It Matters**: Events enable composition and evolution. New requirements become new event handlers, not modifications to existing code. Audit trails emerge naturally. Analytics data flows automatically. Testing focuses on business logic, not integration complexity. Your system becomes more resilient and adaptable.

### World-Class Insights

**🎭 Event Granularity Strategy**: Events should capture business intentions, not technical operations. "UserRegistered" is meaningful; "DatabaseRowInserted" is not. Focus on what business people care about, not what programmers do.

**⚡ Performance Architecture**: Event publishing should never slow down business operations. Use async channels, background processors, and fire-and-forget patterns. If event processing fails, business operations should continue.

**🔍 Event Schema Evolution**: Events are permanent data—design them for change. Use versioning, optional fields, and backward-compatible schemas. Today's perfect event structure will be tomorrow's technical debt if not designed for evolution.

**🎯 Event Naming Conventions**:
- Use past tense: "UserRegistered", not "RegisterUser"
- Be specific: "PaymentCompleted", not "PaymentProcessed"
- Include context: "OrderCancelled" with reason, not just the fact
- Avoid generic names: "EntityUpdated" tells you nothing useful

**🔧 Advanced Event Patterns**:
- **Event Sourcing Light**: Store events alongside traditional state for audit trails
- **Saga Orchestration**: Use events to coordinate complex multi-step processes
- **Event Replay**: Re-process events to generate new projections or fix bugs
- **Event Compaction**: Compress event streams to reduce storage and improve performance

**🛡️ Reliability Strategies**:
- **At-least-once delivery**: Accept duplicate events, design for idempotency
- **Dead letter queues**: Handle permanently failed events gracefully
- **Circuit breakers**: Prevent cascade failures in event processing
- **Monitoring**: Track event processing lag and failure rates

**📊 Event Analytics Goldmine**:
- Business metrics emerge naturally from event streams
- A/B test results can be calculated from user behavior events
- Customer journey analysis becomes trivial with proper event tracking
- Revenue attribution flows directly from purchase and referral events

### Solution: Comprehensive Event System

**Database Migration:**
```sql
-- migrations/XXXX_create_domain_events.sql
CREATE TABLE domain_events (
    id UUID PRIMARY KEY,
    event_type VARCHAR(100) NOT NULL,
    aggregate_id UUID NOT NULL,
    user_id UUID,
    payload JSONB NOT NULL,
    metadata JSONB,
    created_at TIMESTAMPTZ NOT NULL,
    
    -- Indexes for querying
    INDEX idx_events_aggregate (aggregate_id),
    INDEX idx_events_user (user_id),
    INDEX idx_events_type (event_type),
    INDEX idx_events_created (created_at DESC)
);

-- Optional: Event projections for specific views
CREATE TABLE user_activity_stream (
    id UUID PRIMARY KEY,
    user_id UUID NOT NULL,
    event_type VARCHAR(100) NOT NULL,
    description TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL,
    
    INDEX idx_user_activity (user_id, created_at DESC)
);
```

**Event Publisher Implementation:**
```rust
// src/shared/events/publisher.rs
use crate::{DbConn, Error, Result};
use crate::shared::events::{DomainEvent, EventType};
use std::sync::Arc;
use tokio::sync::mpsc;

/// Async event publisher with batching
pub struct AsyncEventPublisher {
    sender: mpsc::Sender<DomainEvent>,
}

impl AsyncEventPublisher {
    pub fn new(pool: sqlx::PgPool) -> Self {
        let (tx, rx) = mpsc::channel(1000);
        
        // Spawn background task for batch processing
        tokio::spawn(event_processor(rx, pool));
        
        Self { sender: tx }
    }
    
    pub async fn publish(&self, event: DomainEvent) -> Result<()> {
        self.sender.send(event).await
            .map_err(|_| Error::Internal("Event channel closed".to_string()))?;
        Ok(())
    }
}

async fn event_processor(
    mut receiver: mpsc::Receiver<DomainEvent>,
    pool: sqlx::PgPool,
) {
    let mut batch = Vec::with_capacity(100);
    let mut interval = tokio::time::interval(Duration::from_millis(100));
    
    loop {
        tokio::select! {
            Some(event) = receiver.recv() => {
                batch.push(event);
                
                // Process immediately if batch is full
                if batch.len() >= 100 {
                    if let Err(e) = process_batch(&pool, &mut batch).await {
                        tracing::error!("Failed to process event batch: {:?}", e);
                    }
                }
            }
            _ = interval.tick() => {
                // Process any pending events every 100ms
                if !batch.is_empty() {
                    if let Err(e) = process_batch(&pool, &mut batch).await {
                        tracing::error!("Failed to process event batch: {:?}", e);
                    }
                }
            }
        }
    }
}

async fn process_batch(pool: &sqlx::PgPool, batch: &mut Vec<DomainEvent>) -> Result<()> {
    if batch.is_empty() {
        return Ok(());
    }
    
    let mut tx = pool.begin().await?;
    
    for event in batch.drain(..) {
        // Store event
        sqlx::query!(
            r#"
            INSERT INTO domain_events (
                id, event_type, aggregate_id, user_id, 
                payload, metadata, created_at
            )
            VALUES ($1, $2, $3, $4, $5, $6, $7)
            "#,
            event.id,
            event.event_type.to_string(),
            event.aggregate_id,
            event.user_id,
            event.payload,
            serde_json::to_value(&event.metadata)?,
            event.created_at
        )
        .execute(&mut *tx)
        .await?;
        
        // Update projections based on event type
        update_projections(&mut tx, &event).await?;
    }
    
    tx.commit().await?;
    Ok(())
}

async fn update_projections(conn: &mut DbConn, event: &DomainEvent) -> Result<()> {
    match event.event_type {
        EventType::UserRegistered => {
            if let Some(user_id) = event.user_id {
                sqlx::query!(
                    r#"
                    INSERT INTO user_activity_stream (id, user_id, event_type, description, created_at)
                    VALUES ($1, $2, $3, $4, $5)
                    "#,
                    Uuid::now_v7(),
                    user_id,
                    "user_registered",
                    "Welcome! Your account has been created.",
                    event.created_at
                )
                .execute(conn)
                .await?;
            }
        }
        EventType::PostPublished => {
            if let Some(user_id) = event.user_id {
                let title = event.payload.get("title")
                    .and_then(|v| v.as_str())
                    .unwrap_or("Untitled");
                    
                sqlx::query!(
                    r#"
                    INSERT INTO user_activity_stream (id, user_id, event_type, description, created_at)
                    VALUES ($1, $2, $3, $4, $5)
                    "#,
                    Uuid::now_v7(),
                    user_id,
                    "post_published",
                    format!("Published a new post: {}", title),
                    event.created_at
                )
                .execute(conn)
                .await?;
            }
        }
        // Add more projections as needed
        _ => {}
    }
    
    Ok(())
}
```

**Integration in Services:**
```rust
// src/auth/services/auth.rs
use crate::shared::events::{DomainEvent, EventType, EventMetadata};

pub async fn register_user(
    conn: &mut DbConn,
    req: RegisterRequest,
    event_publisher: Arc<AsyncEventPublisher>,
    request_metadata: RequestMetadata,
) -> Result<User> {
    // Create user
    let user = repositories::create_user(conn, req).await?;
    
    // Publish event
    let event = DomainEvent::new(
        EventType::UserRegistered,
        user.id,
        Some(user.id),
        serde_json::json!({
            "username": user.username,
            "email": user.email,
            "registration_source": "web",
        }),
    ).with_metadata(EventMetadata {
        ip_address: request_metadata.ip_address,
        user_agent: request_metadata.user_agent,
        correlation_id: request_metadata.correlation_id,
    });
    
    event_publisher.publish(event).await?;
    
    Ok(user)
}

pub async fn login_user(
    conn: &mut DbConn,
    req: LoginRequest,
    event_publisher: Arc<AsyncEventPublisher>,
    request_metadata: RequestMetadata,
) -> Result<LoginResponse> {
    let response = login(conn, req).await?;
    
    // Publish login event
    let event = DomainEvent::new(
        EventType::UserLoggedIn,
        response.user.id,
        Some(response.user.id),
        serde_json::json!({
            "login_method": "password",
            "session_id": response.session_id,
        }),
    ).with_metadata(EventMetadata {
        ip_address: request_metadata.ip_address,
        user_agent: request_metadata.user_agent,
        correlation_id: request_metadata.correlation_id,
    });
    
    event_publisher.publish(event).await?;
    
    Ok(response)
}
```

**Event Querying and Analytics:**
```rust
// src/analytics/repositories.rs
pub async fn get_user_activity(
    conn: &mut DbConn,
    user_id: Uuid,
    days: u32,
) -> Result<Vec<ActivitySummary>> {
    let since = Utc::now() - Duration::days(days as i64);
    
    let activities = sqlx::query_as!(
        ActivitySummary,
        r#"
        SELECT 
            DATE(created_at) as date,
            event_type,
            COUNT(*) as count
        FROM domain_events
        WHERE user_id = $1 AND created_at >= $2
        GROUP BY DATE(created_at), event_type
        ORDER BY date DESC, count DESC
        "#,
        user_id,
        since
    )
    .fetch_all(conn)
    .await?;
    
    Ok(activities)
}

pub async fn get_system_metrics(
    conn: &mut DbConn,
    hours: u32,
) -> Result<SystemMetrics> {
    let since = Utc::now() - Duration::hours(hours as i64);
    
    let metrics = sqlx::query!(
        r#"
        SELECT 
            COUNT(DISTINCT user_id) FILTER (WHERE event_type = 'user_registered') as new_users,
            COUNT(*) FILTER (WHERE event_type = 'user_logged_in') as logins,
            COUNT(*) FILTER (WHERE event_type = 'post_published') as posts_published,
            COUNT(DISTINCT user_id) as active_users
        FROM domain_events
        WHERE created_at >= $1
        "#,
        since
    )
    .fetch_one(conn)
    .await?;
    
    Ok(SystemMetrics {
        new_users: metrics.new_users.unwrap_or(0),
        logins: metrics.logins.unwrap_or(0),
        posts_published: metrics.posts_published.unwrap_or(0),
        active_users: metrics.active_users.unwrap_or(0),
    })
}
```

**Event Listeners Pattern:**
```rust
// src/notifications/event_listener.rs
pub async fn start_event_listener(
    pool: sqlx::PgPool,
    notification_service: Arc<NotificationService>,
) {
    let mut interval = tokio::time::interval(Duration::from_secs(5));
    let mut last_event_id = None;
    
    loop {
        interval.tick().await;
        
        // Poll for new events
        let events = fetch_new_events(&pool, last_event_id).await
            .unwrap_or_default();
        
        for event in events {
            if let Err(e) = handle_event(&notification_service, &event).await {
                tracing::error!("Failed to handle event {}: {:?}", event.id, e);
            }
            last_event_id = Some(event.id);
        }
    }
}

async fn handle_event(
    service: &NotificationService,
    event: &DomainEvent,
) -> Result<()> {
    match event.event_type {
        EventType::UserRegistered => {
            service.send_welcome_email(event.user_id.unwrap()).await?;
        }
        EventType::PostPublished => {
            let post_id = event.aggregate_id;
            service.notify_followers_of_new_post(post_id).await?;
        }
        _ => {}
    }
    Ok(())
}
```

### Benefits
- **Complete audit trail** - Every business operation is recorded
- **Real-time analytics** - Query events for business metrics
- **Loose coupling** - Services communicate through events
- **Event sourcing ready** - Can reconstruct state from events
- **Debugging power** - Full history of what happened when

## 7. Validation Rules Engine

**The Problem**: Validation logic is scattered everywhere—some in your API handlers, some in your services, some duplicated across multiple endpoints. Business rules like "premium users can have unlimited projects" exist in multiple places with subtle differences. Data integrity depends on developers remembering to apply the right validations.

**The First Principle**: Validation is business logic, and business logic should be centralized and composable. Rules should be declarative, reusable, and testable in isolation. The same validation should work whether data comes from an API, a background job, or a data migration.

**Why It Matters**: Centralized validation prevents bugs at the boundary and creates consistency across your application. Business rules become explicit, testable, and maintainable. When requirements change, you modify one place, not ten. Data integrity becomes automatic, not accidental.

### World-Class Insights

**🎯 Validation vs. Business Rules**: Validation checks format and structure; business rules enforce domain constraints. "Email must be valid" is validation; "Premium users can have unlimited projects" is a business rule. Both need centralization, but for different reasons.

**⚡ Performance-First Validation**: Fast validation enables real-time feedback. Order validations by cost—cheap format checks first, expensive database lookups last. Short-circuit on first failure to minimize processing time.

**🔄 Composition Over Inheritance**: Build complex validations from simple, composable pieces. A "user registration validator" combines email validation, password strength, uniqueness checks, and business rules—each testable in isolation.

**🎭 Context-Aware Validation**: The same data needs different validation in different contexts. An email might be "optional" during registration but "required" for email verification. Design validators that understand their context.

**🔧 Advanced Validation Patterns**:
- **Conditional Validation**: Apply rules based on other field values or user state
- **Cross-Field Validation**: Validate relationships between fields (start date < end date)
- **Async Validation**: Handle database checks and external API validations gracefully
- **Batch Validation**: Validate collections efficiently without N+1 problems

**🛡️ Security Through Validation**:
- Validate on every boundary (API, background jobs, data imports)
- Sanitize inputs to prevent injection attacks
- Rate limit expensive validations to prevent DoS
- Log validation failures for security monitoring

**📊 Validation Analytics**:
- Track which validations fail most frequently
- Monitor validation performance to catch expensive rules
- Use validation data to improve UX (common error patterns)
- A/B test validation strictness to optimize conversion vs. quality

**🧪 Validation Testing Strategy**:
- Test edge cases with property-based testing
- Verify error messages are user-friendly and actionable
- Test performance with realistic data volumes
- Ensure validators fail fast and provide clear feedback

### Implementation Examples

**src/shared/validation/rules.rs**
```rust
use crate::{Error, Result};
use once_cell::sync::Lazy;
use regex::Regex;

// Common regex patterns
static ALPHANUMERIC: Lazy<Regex> = Lazy::new(|| Regex::new(r"^[a-zA-Z0-9]+$").unwrap());
static SLUG_PATTERN: Lazy<Regex> = Lazy::new(|| Regex::new(r"^[a-z0-9-]+$").unwrap());
static USERNAME_PATTERN: Lazy<Regex> = Lazy::new(|| Regex::new(r"^[a-zA-Z0-9_]{3,30}$").unwrap());

/// Validation rule builder for chainable validations
pub struct Validator<'a, T> {
    value: &'a T,
    field_name: &'a str,
    errors: Vec<String>,
}

impl<'a, T> Validator<'a, T> {
    pub fn new(value: &'a T, field_name: &'a str) -> Self {
        Self {
            value,
            field_name,
            errors: Vec::new(),
        }
    }
    
    pub fn validate(self) -> Result<()> {
        if !self.errors.is_empty() {
            return Err(Error::ValidationFailed {
                field: self.field_name.to_string(),
                message: self.errors.join("; "),
            });
        }
        Ok(())
    }
}

impl<'a> Validator<'a, String> {
    pub fn required(mut self) -> Self {
        if self.value.trim().is_empty() {
            self.errors.push("Cannot be empty".to_string());
        }
        self
    }
    
    pub fn min_length(mut self, min: usize) -> Self {
        if self.value.len() < min {
            self.errors.push(format!("Must be at least {} characters", min));
        }
        self
    }
    
    pub fn max_length(mut self, max: usize) -> Self {
        if self.value.len() > max {
            self.errors.push(format!("Must be at most {} characters", max));
        }
        self
    }
    
    pub fn pattern(mut self, regex: &Regex, message: &str) -> Self {
        if !regex.is_match(self.value) {
            self.errors.push(message.to_string());
        }
        self
    }
    
    pub fn username(mut self) -> Self {
        if !USERNAME_PATTERN.is_match(self.value) {
            self.errors.push("Username must be 3-30 characters, alphanumeric and underscore only".to_string());
        }
        self
    }
    
    pub fn slug(mut self) -> Self {
        if !SLUG_PATTERN.is_match(self.value) {
            self.errors.push("Slug must contain only lowercase letters, numbers, and hyphens".to_string());
        }
        self
    }
}

impl<'a, T: PartialOrd> Validator<'a, T> {
    pub fn min(mut self, min: T) -> Self 
    where 
        T: std::fmt::Display + Copy
    {
        if *self.value < min {
            self.errors.push(format!("Must be at least {}", min));
        }
        self
    }
    
    pub fn max(mut self, max: T) -> Self
    where 
        T: std::fmt::Display + Copy
    {
        if *self.value > max {
            self.errors.push(format!("Must be at most {}", max));
        }
        self
    }
    
    pub fn range(mut self, min: T, max: T) -> Self
    where 
        T: std::fmt::Display + Copy
    {
        if *self.value < min || *self.value > max {
            self.errors.push(format!("Must be between {} and {}", min, max));
        }
        self
    }
}

// Multi-field validation
pub struct FormValidator {
    errors: Vec<ValidationError>,
}

impl FormValidator {
    pub fn new() -> Self {
        Self { errors: Vec::new() }
    }
    
    pub fn field<T>(&mut self, value: &T, field_name: &str) -> Validator<T> {
        Validator::new(value, field_name)
    }
    
    pub fn add_error(&mut self, field: &str, message: &str) {
        self.errors.push(ValidationError {
            field: field.to_string(),
            message: message.to_string(),
        });
    }
    
    pub fn validate(self) -> Result<()> {
        if !self.errors.is_empty() {
            return Err(Error::ValidationErrors(self.errors));
        }
        Ok(())
    }
}

// Business rule validations
pub mod business_rules {
    use super::*;
    use chrono::{DateTime, Utc};
    
    pub fn validate_age(birth_date: DateTime<Utc>, min_age: u32) -> Result<()> {
        let age = (Utc::now() - birth_date).num_days() / 365;
        if age < min_age as i64 {
            return Err(Error::BusinessRule(
                format!("Must be at least {} years old", min_age)
            ));
        }
        Ok(())
    }
    
    pub fn validate_business_hours(hour: u32) -> Result<()> {
        if hour < 9 || hour > 17 {
            return Err(Error::BusinessRule(
                "Operation only allowed during business hours (9 AM - 5 PM)".to_string()
            ));
        }
        Ok(())
    }
    
    pub fn validate_minimum_balance(balance: f64, minimum: f64) -> Result<()> {
        if balance < minimum {
            return Err(Error::BusinessRule(
                format!("Minimum balance of ${:.2} required", minimum)
            ));
        }
        Ok(())
    }
}
```

**Usage in Services:**
```rust
use crate::shared::validation::{Validator, FormValidator, business_rules};

pub async fn create_product(
    conn: &mut DbConn,
    req: CreateProductRequest,
) -> Result<Product> {
    // Single field validation
    Validator::new(&req.name, "name")
        .required()
        .min_length(3)
        .max_length(100)
        .validate()?;
    
    Validator::new(&req.price, "price")
        .min(0.01)
        .max(999999.99)
        .validate()?;
    
    // Multi-field validation
    let mut form = FormValidator::new();
    
    // Custom business rule
    if req.discount_percentage > req.max_discount {
        form.add_error("discount_percentage", 
            "Discount cannot exceed maximum allowed discount");
    }
    
    form.validate()?;
    
    // Create product...
}

pub async fn process_order(
    conn: &mut DbConn,
    order: OrderRequest,
) -> Result<Order> {
    // Validate business hours
    business_rules::validate_business_hours(Utc::now().hour())?;
    
    // Validate customer balance
    let balance = get_customer_balance(conn, order.customer_id).await?;
    business_rules::validate_minimum_balance(balance, 10.0)?;
    
    // Process order...
}
```

## 8. Event Sourcing and CQRS

**The Problem**: Traditional CRUD applications lose history. When a price changes, you can't see what it was before or why it changed. Audit requirements force you to bolt on change tracking. Complex business workflows require orchestrating multiple database updates, making transactions unwieldy and error-prone.

**The First Principle**: Store intentions, not just current state. Instead of updating a record, record that an update was intended. Instead of deleting data, record that deletion was requested. Events capture the "why" and "when" that traditional databases lose forever.

**Why It Matters**: Event sourcing provides perfect audit trails, enables time travel debugging, and naturally handles complex business workflows. CQRS optimizes reads and writes independently, allowing you to scale each dimension according to your needs. Together, they enable sophisticated business logic while maintaining data integrity and observability.

### World-Class Insights

**🎯 When to Choose Event Sourcing**: Perfect for domains with complex business rules, strict audit requirements, or temporal queries ("What was the price on March 15th?"). Avoid it for simple CRUD operations or read-heavy systems where current state is all that matters.

**⚡ Performance Realities**: Event sourcing trades write complexity for read flexibility. Writes become more complex (storing events + updating projections), but reads can be optimized independently. Plan for eventual consistency between command and query sides.

**🔍 Snapshot Strategy**: Don't replay thousands of events every time—use snapshots for performance. Take snapshots every N events or when aggregates get large. Store snapshots alongside events, not instead of them.

**🎭 Event Schema Evolution**: Events are your permanent data—design for change. Use techniques like event upcasting, schema versioning, and backward-compatible fields. Consider event migration strategies before you need them.

**🔧 Advanced CQRS Patterns**:
- **Multiple Read Models**: Create specialized projections for different use cases (reporting, search, caching)
- **Event Store Partitioning**: Partition events by aggregate ID for horizontal scaling
- **Read Model Rebuilding**: Design for complete read model reconstruction from events
- **Cross-Aggregate Queries**: Use process managers or sagas for operations spanning aggregates

**🛡️ Consistency Guarantees**:
- **Strong consistency**: Within aggregate boundaries only
- **Eventual consistency**: Between aggregates and read models
- **Causal consistency**: Events maintain happened-before relationships
- **Conflict resolution**: Handle concurrent modifications with business rules

**📊 Operational Excellence**:
- Monitor event store growth rates and implement archiving strategies
- Track read model staleness and rebuild times
- Implement event store backup and disaster recovery procedures
- Use correlation IDs to trace operations across read and write sides

**🧪 Testing Event-Sourced Systems**:
- Test aggregate behavior with given-when-then patterns
- Verify event serialization/deserialization is stable
- Test projection rebuilding with historical events
- Use property-based testing for concurrent event processing

### Core Components

**Event Store Implementation:**
```rust
// src/shared/events/store.rs
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use sqlx::types::Json;
use uuid::Uuid;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct EventMetadata {
    pub event_id: Uuid,
    pub aggregate_id: Uuid,
    pub aggregate_type: String,
    pub event_type: String,
    pub event_version: i32,
    pub occurred_at: DateTime<Utc>,
    pub user_id: Option<Uuid>,
    pub correlation_id: Option<Uuid>,
    pub causation_id: Option<Uuid>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct StoredEvent {
    pub metadata: EventMetadata,
    pub data: serde_json::Value,
}

// Event store operations
pub async fn append_events(
    conn: &mut DbConn,
    aggregate_id: Uuid,
    expected_version: i32,
    events: Vec<DomainEvent>,
) -> Result<(), Error> {
    let mut tx = conn.begin().await?;
    
    // Check current version for optimistic concurrency
    let current_version = sqlx::query_scalar!(
        "SELECT COALESCE(MAX(event_version), 0) FROM events WHERE aggregate_id = $1",
        aggregate_id
    )
    .fetch_one(&mut *tx)
    .await?
    .unwrap_or(0);
    
    if current_version != expected_version {
        return Err(Error::ConcurrencyConflict {
            expected: expected_version,
            actual: current_version,
        });
    }
    
    // Append events
    for (i, event) in events.into_iter().enumerate() {
        let event_version = expected_version + i as i32 + 1;
        let metadata = EventMetadata {
            event_id: Uuid::now_v7(),
            aggregate_id,
            aggregate_type: event.aggregate_type(),
            event_type: event.event_type(),
            event_version,
            occurred_at: Utc::now(),
            user_id: event.user_id(),
            correlation_id: event.correlation_id(),
            causation_id: event.causation_id(),
        };
        
        sqlx::query!(
            r#"
                INSERT INTO events (
                    event_id, aggregate_id, aggregate_type, event_type,
                    event_version, occurred_at, user_id, correlation_id,
                    causation_id, event_data
                ) VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10)
            "#,
            metadata.event_id,
            metadata.aggregate_id,
            metadata.aggregate_type,
            metadata.event_type,
            metadata.event_version,
            metadata.occurred_at,
            metadata.user_id,
            metadata.correlation_id,
            metadata.causation_id,
            Json(&event.data()) as _
        )
        .execute(&mut *tx)
        .await?;
    }
    
    tx.commit().await?;
    Ok(())
}

pub async fn load_events(
    conn: &mut DbConn,
    aggregate_id: Uuid,
    from_version: Option<i32>,
) -> Result<Vec<StoredEvent>, Error> {
    let from_version = from_version.unwrap_or(0);
    
    let rows = sqlx::query!(
        r#"
            SELECT event_id, aggregate_id, aggregate_type, event_type,
                   event_version, occurred_at, user_id, correlation_id,
                   causation_id, event_data
            FROM events
            WHERE aggregate_id = $1 AND event_version > $2
            ORDER BY event_version ASC
        "#,
        aggregate_id,
        from_version
    )
    .fetch_all(conn)
    .await?;
    
    let events = rows
        .into_iter()
        .map(|row| StoredEvent {
            metadata: EventMetadata {
                event_id: row.event_id,
                aggregate_id: row.aggregate_id,
                aggregate_type: row.aggregate_type,
                event_type: row.event_type,
                event_version: row.event_version,
                occurred_at: row.occurred_at,
                user_id: row.user_id,
                correlation_id: row.correlation_id,
                causation_id: row.causation_id,
            },
            data: row.event_data.0,
        })
        .collect();
    
    Ok(events)
}
```

**Domain Events Trait:**
```rust
// src/shared/events/domain.rs
use serde::{Deserialize, Serialize};
use uuid::Uuid;

pub trait DomainEvent: Clone + Send + Sync {
    fn aggregate_type(&self) -> String;
    fn event_type(&self) -> String;
    fn data(&self) -> serde_json::Value;
    fn user_id(&self) -> Option<Uuid>;
    fn correlation_id(&self) -> Option<Uuid>;
    fn causation_id(&self) -> Option<Uuid>;
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct UserRegistered {
    pub user_id: Uuid,
    pub email: String,
    pub full_name: String,
    pub registration_source: String,
    pub correlation_id: Option<Uuid>,
}

impl DomainEvent for UserRegistered {
    fn aggregate_type(&self) -> String {
        "User".to_string()
    }
    
    fn event_type(&self) -> String {
        "UserRegistered".to_string()
    }
    
    fn data(&self) -> serde_json::Value {
        serde_json::to_value(self).unwrap()
    }
    
    fn user_id(&self) -> Option<Uuid> {
        Some(self.user_id)
    }
    
    fn correlation_id(&self) -> Option<Uuid> {
        self.correlation_id
    }
    
    fn causation_id(&self) -> Option<Uuid> {
        None
    }
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct OrderPlaced {
    pub order_id: Uuid,
    pub customer_id: Uuid,
    pub items: Vec<OrderItem>,
    pub total_amount: rust_decimal::Decimal,
    pub currency: String,
    pub correlation_id: Option<Uuid>,
}

impl DomainEvent for OrderPlaced {
    fn aggregate_type(&self) -> String {
        "Order".to_string()
    }
    
    fn event_type(&self) -> String {
        "OrderPlaced".to_string()
    }
    
    fn data(&self) -> serde_json::Value {
        serde_json::to_value(self).unwrap()
    }
    
    fn user_id(&self) -> Option<Uuid> {
        Some(self.customer_id)
    }
    
    fn correlation_id(&self) -> Option<Uuid> {
        self.correlation_id
    }
    
    fn causation_id(&self) -> Option<Uuid> {
        None
    }
}
```

**Command and Query Handlers:**
```rust
// src/orders/commands.rs
use crate::shared::events::{append_events, DomainEvent};

#[derive(Debug)]
pub struct PlaceOrderCommand {
    pub customer_id: Uuid,
    pub items: Vec<OrderItem>,
    pub delivery_address: Address,
    pub correlation_id: Option<Uuid>,
}

pub async fn handle_place_order(
    conn: &mut DbConn,
    command: PlaceOrderCommand,
) -> Result<Uuid, Error> {
    // Load customer aggregate
    let customer = load_customer_aggregate(conn, command.customer_id).await?;
    
    // Validate business rules
    validate_customer_can_place_order(&customer)?;
    validate_order_items(&command.items).await?;
    
    let order_id = Uuid::now_v7();
    let total_amount = calculate_total(&command.items);
    
    // Create domain events
    let events: Vec<Box<dyn DomainEvent>> = vec![
        Box::new(OrderPlaced {
            order_id,
            customer_id: command.customer_id,
            items: command.items.clone(),
            total_amount,
            currency: "USD".to_string(),
            correlation_id: command.correlation_id,
        }),
        Box::new(InventoryReserved {
            order_id,
            reservations: create_inventory_reservations(&command.items),
            correlation_id: command.correlation_id,
        }),
    ];
    
    // Append events to store
    append_events(conn, order_id, 0, events).await?;
    
    // Publish events for projections and external systems
    publish_events_async(&events).await?;
    
    Ok(order_id)
}

// src/orders/queries.rs
#[derive(Debug, Serialize)]
pub struct OrderSummary {
    pub id: Uuid,
    pub customer_name: String,
    pub status: OrderStatus,
    pub total_amount: rust_decimal::Decimal,
    pub created_at: DateTime<Utc>,
    pub item_count: i32,
}

pub async fn get_customer_orders(
    conn: &mut DbConn,
    customer_id: Uuid,
    pagination: PaginationParams,
) -> Result<PaginatedResult<OrderSummary>, Error> {
    // Query from read model/projection
    let total = sqlx::query_scalar!(
        "SELECT COUNT(*) FROM order_summaries WHERE customer_id = $1",
        customer_id
    )
    .fetch_one(&mut *conn)
    .await?
    .unwrap_or(0);
    
    let orders = sqlx::query_as!(
        OrderSummary,
        r#"
            SELECT id, customer_name, status as "status: OrderStatus",
                   total_amount, created_at, item_count
            FROM order_summaries
            WHERE customer_id = $1
            ORDER BY created_at DESC
            LIMIT $2 OFFSET $3
        "#,
        customer_id,
        pagination.limit,
        pagination.offset()
    )
    .fetch_all(conn)
    .await?;
    
    Ok(PaginatedResult::new(orders, total, pagination))
}

pub async fn get_order_details(
    conn: &mut DbConn,
    order_id: Uuid,
) -> Result<Option<OrderDetails>, Error> {
    // Reconstruct from events or query projection
    let events = load_events(conn, order_id, None).await?;
    
    if events.is_empty() {
        return Ok(None);
    }
    
    let mut order = OrderAggregate::new();
    for event in events {
        order.apply_event(&event)?;
    }
    
    Ok(Some(order.to_details()))
}
```

**Event Projections:**
```rust
// src/shared/projections/mod.rs
use tokio::sync::mpsc;

pub trait Projection: Send + Sync {
    async fn handle_event(&mut self, event: &StoredEvent) -> Result<(), Error>;
    fn handles_event_type(&self, event_type: &str) -> bool;
}

pub struct ProjectionManager {
    projections: Vec<Box<dyn Projection>>,
    event_receiver: mpsc::Receiver<StoredEvent>,
}

impl ProjectionManager {
    pub async fn run(&mut self) -> Result<(), Error> {
        while let Some(event) = self.event_receiver.recv().await {
            for projection in &mut self.projections {
                if projection.handles_event_type(&event.metadata.event_type) {
                    if let Err(e) = projection.handle_event(&event).await {
                        tracing::error!(
                            event_id = %event.metadata.event_id,
                            projection = %std::any::type_name_of_val(projection.as_ref()),
                            error = %e,
                            "Projection failed to handle event"
                        );
                    }
                }
            }
        }
        Ok(())
    }
}

// Order summary projection
pub struct OrderSummaryProjection {
    pool: PgPool,
}

impl Projection for OrderSummaryProjection {
    async fn handle_event(&mut self, event: &StoredEvent) -> Result<(), Error> {
        match event.metadata.event_type.as_str() {
            "OrderPlaced" => {
                let order_placed: OrderPlaced = serde_json::from_value(event.data.clone())?;
                self.create_order_summary(&event.metadata, &order_placed).await?;
            }
            "OrderStatusChanged" => {
                let status_changed: OrderStatusChanged = serde_json::from_value(event.data.clone())?;
                self.update_order_status(&event.metadata, &status_changed).await?;
            }
            _ => {}
        }
        Ok(())
    }
    
    fn handles_event_type(&self, event_type: &str) -> bool {
        matches!(event_type, "OrderPlaced" | "OrderStatusChanged" | "OrderCancelled")
    }
}

impl OrderSummaryProjection {
    async fn create_order_summary(
        &self,
        metadata: &EventMetadata,
        event: &OrderPlaced,
    ) -> Result<(), Error> {
        let mut conn = self.pool.acquire().await?;
        
        sqlx::query!(
            r#"
                INSERT INTO order_summaries (
                    id, customer_id, customer_name, status, total_amount,
                    created_at, item_count, last_event_version
                ) VALUES ($1, $2, $3, $4, $5, $6, $7, $8)
                ON CONFLICT (id) DO NOTHING
            "#,
            event.order_id,
            event.customer_id,
            "Unknown", // Will be updated by customer projection
            OrderStatus::Pending as OrderStatus,
            event.total_amount,
            metadata.occurred_at,
            event.items.len() as i32,
            metadata.event_version
        )
        .execute(&mut *conn)
        .await?;
        
        Ok(())
    }
}
```

**Saga Pattern for Complex Workflows:**
```rust
// src/shared/sagas/mod.rs
use std::collections::HashMap;
use tokio::time::{sleep, Duration};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum SagaState {
    Started,
    PaymentProcessing,
    InventoryReserved,
    OrderConfirmed,
    Failed(String),
    Compensating,
    Completed,
}

pub struct OrderProcessingSaga {
    saga_id: Uuid,
    order_id: Uuid,
    state: SagaState,
    compensation_data: HashMap<String, serde_json::Value>,
}

impl OrderProcessingSaga {
    pub async fn handle_event(&mut self, event: &StoredEvent) -> Result<Vec<Command>, Error> {
        match (&self.state, event.metadata.event_type.as_str()) {
            (SagaState::Started, "OrderPlaced") => {
                self.state = SagaState::PaymentProcessing;
                Ok(vec![Command::ProcessPayment {
                    order_id: self.order_id,
                    amount: self.extract_order_amount(event)?,
                }])
            }
            
            (SagaState::PaymentProcessing, "PaymentSucceeded") => {
                self.state = SagaState::InventoryReserved;
                Ok(vec![Command::ReserveInventory {
                    order_id: self.order_id,
                    items: self.extract_order_items(event)?,
                }])
            }
            
            (SagaState::PaymentProcessing, "PaymentFailed") => {
                self.state = SagaState::Failed("Payment failed".to_string());
                Ok(vec![Command::CancelOrder {
                    order_id: self.order_id,
                    reason: "Payment failed".to_string(),
                }])
            }
            
            (SagaState::InventoryReserved, "InventoryReserved") => {
                self.state = SagaState::OrderConfirmed;
                Ok(vec![
                    Command::ConfirmOrder { order_id: self.order_id },
                    Command::SendOrderConfirmationEmail { order_id: self.order_id },
                ])
            }
            
            (SagaState::InventoryReserved, "InventoryReservationFailed") => {
                self.state = SagaState::Compensating;
                Ok(vec![
                    Command::RefundPayment { order_id: self.order_id },
                    Command::CancelOrder { 
                        order_id: self.order_id,
                        reason: "Insufficient inventory".to_string(),
                    },
                ])
            }
            
            _ => Ok(vec![])
        }
    }
}
```

## 9. Observability and Telemetry Architecture

**The Problem**: Your application works fine in development, but production is a black box. When something goes wrong, you're debugging with `println!` statements and prayer. Performance problems are invisible until users complain. Business metrics are trapped in database queries that only the data team can write.

**The First Principle**: You can't manage what you can't measure. Modern systems are too complex to understand through intuition alone. Observability isn't just monitoring—it's building systems that explain themselves through structured data, metrics, traces, and business events.

**Why It Matters**: Observability transforms firefighting into engineering. Instead of guessing why something broke, you have data. Instead of optimizing blindly, you measure impact. Instead of wondering how users behave, you track patterns. Observability turns operations from reactive crisis management into proactive system improvement.

### World-Class Insights

**🎯 The Three Pillars Strategy**: Metrics tell you what's broken, traces tell you where it's broken, logs tell you why it's broken. Design each pillar to answer specific questions, not to collect data for the sake of it.

**⚡ Cardinality Management**: High-cardinality metrics (user IDs, request IDs) will destroy your observability budget. Use sampling, aggregation, or separate storage for different cardinality levels. Plan for scale from day one.

**🔍 Context Propagation**: Correlation IDs are your best friend for distributed systems. Every log, metric, and trace should include context that lets you follow a request across services. Make context propagation automatic, not manual.

**🎭 The Sampling Dilemma**: 100% tracing is expensive but incomplete sampling misses rare bugs. Use intelligent sampling—always trace errors, sample successes by latency, and increase sampling for specific operations during debugging.

**🔧 Advanced Observability Patterns**:
- **Golden Signals**: Focus on latency, traffic, errors, and saturation as primary health indicators
- **SLI/SLO Design**: Define Service Level Indicators that matter to users, not just systems
- **Synthetic Monitoring**: Proactively test critical user journeys before real users encounter problems
- **Chaos Engineering**: Inject failures to validate your observability coverage

**🛡️ Production Readiness**:
- Implement graceful degradation when observability systems fail
- Use separate infrastructure for observability to avoid circular dependencies
- Design for observability from the beginning—retrofitting is expensive and incomplete
- Practice incident response with your observability tools before you need them

**📊 Business Observability**:
- Track business metrics alongside technical metrics
- Use real-time dashboards for business decision making
- Implement feature flag monitoring to measure rollout impact
- Connect technical performance to business outcomes (latency → conversion rate)

**🎪 Alerting Philosophy**:
- Alert on symptoms, not causes
- Every alert should be actionable—if you can't fix it, don't alert on it
- Use different alerting channels for different severity levels
- Practice alert fatigue prevention through intelligent grouping and suppression

### OpenTelemetry Integration

**Tracing Setup:**
```rust
// src/shared/observability/tracing.rs
use opentelemetry::{
    trace::{TraceContextExt, Tracer, TracerProvider},
    Context, KeyValue,
};
use opentelemetry_otlp::WithExportConfig;
use opentelemetry_sdk::{
    trace::{self, RandomIdGenerator, Sampler},
    Resource,
};
use tracing::{info_span, Instrument};
use tracing_opentelemetry::OpenTelemetrySpanExt;
use uuid::Uuid;

pub fn init_tracing() -> Result<(), Error> {
    let tracer = opentelemetry_otlp::new_pipeline()
        .tracing()
        .with_exporter(
            opentelemetry_otlp::new_exporter()
                .tonic()
                .with_endpoint("http://localhost:4317"),
        )
        .with_trace_config(
            trace::config()
                .with_sampler(Sampler::AlwaysOn)
                .with_id_generator(RandomIdGenerator::default())
                .with_resource(Resource::new(vec![
                    KeyValue::new("service.name", "bodisafari"),
                    KeyValue::new("service.version", env!("CARGO_PKG_VERSION")),
                ])),
        )
        .install_batch(opentelemetry_sdk::runtime::Tokio)?;

    tracing_subscriber::registry()
        .with(tracing_subscriber::EnvFilter::new("info"))
        .with(tracing_subscriber::fmt::layer())
        .with(tracing_opentelemetry::layer().with_tracer(tracer))
        .init();

    Ok(())
}

#[derive(Clone)]
pub struct TraceContext {
    pub trace_id: String,
    pub span_id: String,
    pub correlation_id: Option<Uuid>,
}

impl TraceContext {
    pub fn new() -> Self {
        let context = Context::current();
        let span = context.span();
        let span_context = span.span_context();
        
        Self {
            trace_id: span_context.trace_id().to_string(),
            span_id: span_context.span_id().to_string(),
            correlation_id: Some(Uuid::now_v7()),
        }
    }
    
    pub fn from_headers(headers: &HeaderMap) -> Self {
        // Extract from W3C trace context headers
        let trace_parent = headers
            .get("traceparent")
            .and_then(|h| h.to_str().ok());
            
        // Parse traceparent format: 00-{trace_id}-{span_id}-{flags}
        if let Some(trace_parent) = trace_parent {
            let parts: Vec<&str> = trace_parent.split('-').collect();
            if parts.len() == 4 {
                return Self {
                    trace_id: parts[1].to_string(),
                    span_id: parts[2].to_string(),
                    correlation_id: Some(Uuid::now_v7()),
                };
            }
        }
        
        Self::new()
    }
}
```

**Structured Logging with Correlation:**
```rust
// src/shared/observability/logging.rs
use serde_json::Value;
use tracing::{event, Level};
use uuid::Uuid;

#[derive(Debug, Clone)]
pub struct LogContext {
    pub user_id: Option<Uuid>,
    pub session_id: Option<Uuid>,
    pub request_id: Option<Uuid>,
    pub correlation_id: Option<Uuid>,
    pub operation: String,
    pub component: String,
}

impl LogContext {
    pub fn new(operation: &str, component: &str) -> Self {
        Self {
            user_id: None,
            session_id: None,
            request_id: Some(Uuid::now_v7()),
            correlation_id: Some(Uuid::now_v7()),
            operation: operation.to_string(),
            component: component.to_string(),
        }
    }
    
    pub fn with_user(mut self, user_id: Uuid) -> Self {
        self.user_id = Some(user_id);
        self
    }
    
    pub fn with_session(mut self, session_id: Uuid) -> Self {
        self.session_id = Some(session_id);
        self
    }
}

pub fn log_business_event(
    ctx: &LogContext,
    event_type: &str,
    details: Value,
    metrics: Option<Vec<(String, f64)>>,
) {
    event!(
        Level::INFO,
        user_id = ?ctx.user_id,
        session_id = ?ctx.session_id,
        request_id = ?ctx.request_id,
        correlation_id = ?ctx.correlation_id,
        operation = %ctx.operation,
        component = %ctx.component,
        event_type = %event_type,
        details = %details,
        metrics = ?metrics,
        "Business event occurred"
    );
}

pub fn log_performance_metric(
    ctx: &LogContext,
    metric_name: &str,
    value: f64,
    unit: &str,
    tags: Option<Vec<(String, String)>>,
) {
    event!(
        Level::INFO,
        user_id = ?ctx.user_id,
        correlation_id = ?ctx.correlation_id,
        operation = %ctx.operation,
        component = %ctx.component,
        metric_name = %metric_name,
        metric_value = %value,
        metric_unit = %unit,
        tags = ?tags,
        "Performance metric recorded"
    );
}

pub fn log_error_with_context(
    ctx: &LogContext,
    error: &Error,
    additional_context: Option<Value>,
) {
    event!(
        Level::ERROR,
        user_id = ?ctx.user_id,
        session_id = ?ctx.session_id,
        request_id = ?ctx.request_id,
        correlation_id = ?ctx.correlation_id,
        operation = %ctx.operation,
        component = %ctx.component,
        error = %error,
        error_type = %std::any::type_name_of_val(error),
        context = ?additional_context,
        "Error occurred with full context"
    );
}
```

**Custom Metrics Collection:**
```rust
// src/shared/observability/metrics.rs
use std::sync::Arc;
use std::time::{Duration, Instant};
use prometheus::{
    Counter, Histogram, HistogramOpts, IntCounter, IntGauge, Opts, Registry,
};
use tokio::sync::RwLock;
use uuid::Uuid;

#[derive(Clone)]
pub struct MetricsCollector {
    registry: Arc<Registry>,
    
    // HTTP metrics
    pub http_requests_total: Counter,
    pub http_request_duration: Histogram,
    pub http_response_size: Histogram,
    
    // Database metrics
    pub db_query_duration: Histogram,
    pub db_connections_active: IntGauge,
    pub db_query_errors: IntCounter,
    
    // Business metrics
    pub users_registered: IntCounter,
    pub orders_placed: IntCounter,
    pub revenue_total: Counter,
    
    // System metrics
    pub memory_usage: IntGauge,
    pub cpu_usage: Counter,
}

impl MetricsCollector {
    pub fn new() -> Result<Self, Error> {
        let registry = Arc::new(Registry::new());
        
        let http_requests_total = Counter::with_opts(Opts::new(
            "http_requests_total",
            "Total HTTP requests received"
        ).namespace("app"))?;
        
        let http_request_duration = Histogram::with_opts(HistogramOpts::new(
            "http_request_duration_seconds",
            "HTTP request duration in seconds"
        ).namespace("app"))?;
        
        let db_query_duration = Histogram::with_opts(HistogramOpts::new(
            "db_query_duration_seconds",
            "Database query duration in seconds"
        ).namespace("app"))?;
        
        let users_registered = IntCounter::with_opts(Opts::new(
            "users_registered_total",
            "Total users registered"
        ).namespace("app"))?;
        
        // Register all metrics
        registry.register(Box::new(http_requests_total.clone()))?;
        registry.register(Box::new(http_request_duration.clone()))?;
        registry.register(Box::new(db_query_duration.clone()))?;
        registry.register(Box::new(users_registered.clone()))?;
        
        Ok(Self {
            registry,
            http_requests_total,
            http_request_duration,
            http_response_size: Histogram::new(),
            db_query_duration,
            db_connections_active: IntGauge::new(),
            db_query_errors: IntCounter::new(),
            users_registered,
            orders_placed: IntCounter::new(),
            revenue_total: Counter::new(),
            memory_usage: IntGauge::new(),
            cpu_usage: Counter::new(),
        })
    }
    
    pub fn record_http_request(&self, method: &str, path: &str, status: u16, duration: Duration) {
        self.http_requests_total.inc();
        self.http_request_duration.observe(duration.as_secs_f64());
        
        tracing::info!(
            method = %method,
            path = %path,
            status = %status,
            duration_ms = %duration.as_millis(),
            "HTTP request completed"
        );
    }
    
    pub fn record_db_query(&self, query_type: &str, table: &str, duration: Duration, success: bool) {
        self.db_query_duration.observe(duration.as_secs_f64());
        
        if !success {
            self.db_query_errors.inc();
        }
        
        tracing::debug!(
            query_type = %query_type,
            table = %table,
            duration_ms = %duration.as_millis(),
            success = %success,
            "Database query executed"
        );
    }
    
    pub fn record_business_event(&self, event_type: &str, user_id: Option<Uuid>, value: Option<f64>) {
        match event_type {
            "user_registered" => {
                self.users_registered.inc();
            },
            "order_placed" => {
                self.orders_placed.inc();
                if let Some(amount) = value {
                    self.revenue_total.inc_by(amount);
                }
            },
            _ => {}
        }
        
        tracing::info!(
            event_type = %event_type,
            user_id = ?user_id,
            value = ?value,
            "Business event recorded"
        );
    }
}

// Middleware for automatic HTTP metrics
pub struct MetricsMiddleware {
    metrics: Arc<MetricsCollector>,
}

impl MetricsMiddleware {
    pub fn new(metrics: Arc<MetricsCollector>) -> Self {
        Self { metrics }
    }
}

#[axum::async_trait]
impl<S> FromRequestParts<S> for MetricsMiddleware
where
    S: Send + Sync,
{
    type Rejection = ();

    async fn from_request_parts(_parts: &mut Parts, _state: &S) -> Result<Self, Self::Rejection> {
        // This would be injected via layer
        unreachable!()
    }
}
```

**Health Checks and Circuit Breakers:**
```rust
// src/shared/observability/health.rs
use std::collections::HashMap;
use std::sync::Arc;
use std::time::{Duration, Instant};
use tokio::sync::RwLock;
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum HealthStatus {
    Healthy,
    Degraded,
    Unhealthy,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ComponentHealth {
    pub status: HealthStatus,
    pub last_check: chrono::DateTime<chrono::Utc>,
    pub response_time_ms: u64,
    pub details: Option<String>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SystemHealth {
    pub overall_status: HealthStatus,
    pub components: HashMap<String, ComponentHealth>,
    pub uptime_seconds: u64,
    pub version: String,
}

pub struct HealthChecker {
    components: Arc<RwLock<HashMap<String, ComponentHealth>>>,
    start_time: Instant,
}

impl HealthChecker {
    pub fn new() -> Self {
        Self {
            components: Arc::new(RwLock::new(HashMap::new())),
            start_time: Instant::now(),
        }
    }
    
    pub async fn check_database(&self, pool: &PgPool) -> ComponentHealth {
        let start = Instant::now();
        
        let result = sqlx::query("SELECT 1")
            .fetch_one(pool)
            .await;
            
        let response_time = start.elapsed();
        
        match result {
            Ok(_) => ComponentHealth {
                status: if response_time.as_millis() < 100 {
                    HealthStatus::Healthy
                } else {
                    HealthStatus::Degraded
                },
                last_check: chrono::Utc::now(),
                response_time_ms: response_time.as_millis() as u64,
                details: None,
            },
            Err(e) => ComponentHealth {
                status: HealthStatus::Unhealthy,
                last_check: chrono::Utc::now(),
                response_time_ms: response_time.as_millis() as u64,
                details: Some(format!("Database error: {}", e)),
            },
        }
    }
    
    pub async fn check_external_service(&self, service_name: &str, url: &str) -> ComponentHealth {
        let start = Instant::now();
        
        let client = reqwest::Client::builder()
            .timeout(Duration::from_secs(5))
            .build()
            .unwrap();
            
        let result = client.get(url).send().await;
        let response_time = start.elapsed();
        
        match result {
            Ok(response) if response.status().is_success() => ComponentHealth {
                status: HealthStatus::Healthy,
                last_check: chrono::Utc::now(),
                response_time_ms: response_time.as_millis() as u64,
                details: None,
            },
            Ok(response) => ComponentHealth {
                status: HealthStatus::Degraded,
                last_check: chrono::Utc::now(),
                response_time_ms: response_time.as_millis() as u64,
                details: Some(format!("HTTP {}", response.status())),
            },
            Err(e) => ComponentHealth {
                status: HealthStatus::Unhealthy,
                last_check: chrono::Utc::now(),
                response_time_ms: response_time.as_millis() as u64,
                details: Some(format!("Request failed: {}", e)),
            },
        }
    }
    
    pub async fn get_system_health(&self) -> SystemHealth {
        let components = self.components.read().await.clone();
        
        let overall_status = if components.values().all(|c| matches!(c.status, HealthStatus::Healthy)) {
            HealthStatus::Healthy
        } else if components.values().any(|c| matches!(c.status, HealthStatus::Unhealthy)) {
            HealthStatus::Unhealthy
        } else {
            HealthStatus::Degraded
        };
        
        SystemHealth {
            overall_status,
            components,
            uptime_seconds: self.start_time.elapsed().as_secs(),
            version: env!("CARGO_PKG_VERSION").to_string(),
        }
    }
}

// Circuit breaker implementation
#[derive(Debug, Clone)]
pub enum CircuitState {
    Closed,
    Open,
    HalfOpen,
}

pub struct CircuitBreaker {
    state: Arc<RwLock<CircuitState>>,
    failure_count: Arc<RwLock<u32>>,
    last_failure_time: Arc<RwLock<Option<Instant>>>,
    failure_threshold: u32,
    timeout: Duration,
}

impl CircuitBreaker {
    pub fn new(failure_threshold: u32, timeout: Duration) -> Self {
        Self {
            state: Arc::new(RwLock::new(CircuitState::Closed)),
            failure_count: Arc::new(RwLock::new(0)),
            last_failure_time: Arc::new(RwLock::new(None)),
            failure_threshold,
            timeout,
        }
    }
    
    pub async fn call<F, Fut, T, E>(&self, f: F) -> Result<T, CircuitBreakerError<E>>
    where
        F: FnOnce() -> Fut,
        Fut: std::future::Future<Output = Result<T, E>>,
    {
        // Check if circuit is open
        {
            let state = self.state.read().await;
            if matches!(*state, CircuitState::Open) {
                let last_failure = self.last_failure_time.read().await;
                if let Some(last_failure) = *last_failure {
                    if last_failure.elapsed() < self.timeout {
                        return Err(CircuitBreakerError::CircuitOpen);
                    }
                    // Transition to half-open
                    drop(state);
                    *self.state.write().await = CircuitState::HalfOpen;
                }
            }
        }
        
        match f().await {
            Ok(result) => {
                // Reset on success
                *self.failure_count.write().await = 0;
                *self.state.write().await = CircuitState::Closed;
                Ok(result)
            }
            Err(e) => {
                // Increment failure count
                let mut count = self.failure_count.write().await;
                *count += 1;
                
                if *count >= self.failure_threshold {
                    *self.state.write().await = CircuitState::Open;
                    *self.last_failure_time.write().await = Some(Instant::now());
                }
                
                Err(CircuitBreakerError::ServiceError(e))
            }
        }
    }
}

#[derive(Debug)]
pub enum CircuitBreakerError<E> {
    CircuitOpen,
    ServiceError(E),
}
```

**Business Metrics and KPIs:**
```rust
// src/shared/observability/business_metrics.rs
use chrono::{DateTime, Utc, Duration as ChronoDuration};
use serde::{Deserialize, Serialize};
use std::collections::HashMap;

#[derive(Debug, Serialize, Deserialize)]
pub struct BusinessMetrics {
    pub daily_active_users: i64,
    pub monthly_active_users: i64,
    pub conversion_rate: f64,
    pub average_order_value: rust_decimal::Decimal,
    pub customer_lifetime_value: rust_decimal::Decimal,
    pub churn_rate: f64,
    pub revenue_growth: f64,
}

#[derive(Debug, Serialize, Deserialize)]
pub struct UserEngagementMetrics {
    pub session_duration_avg: f64,
    pub page_views_per_session: f64,
    pub bounce_rate: f64,
    pub feature_adoption_rates: HashMap<String, f64>,
}

pub async fn calculate_daily_metrics(
    conn: &mut DbConn,
    date: DateTime<Utc>,
) -> Result<BusinessMetrics, Error> {
    let start_of_day = date.date_naive().and_hms_opt(0, 0, 0).unwrap().and_utc();
    let end_of_day = start_of_day + ChronoDuration::days(1);
    
    // Daily Active Users
    let dau = sqlx::query_scalar!(
        r#"
            SELECT COUNT(DISTINCT user_id) 
            FROM user_activities 
            WHERE activity_time >= $1 AND activity_time < $2
        "#,
        start_of_day,
        end_of_day
    )
    .fetch_one(&mut *conn)
    .await?
    .unwrap_or(0);
    
    // Monthly Active Users (last 30 days)
    let thirty_days_ago = start_of_day - ChronoDuration::days(30);
    let mau = sqlx::query_scalar!(
        r#"
            SELECT COUNT(DISTINCT user_id) 
            FROM user_activities 
            WHERE activity_time >= $1 AND activity_time < $2
        "#,
        thirty_days_ago,
        end_of_day
    )
    .fetch_one(&mut *conn)
    .await?
    .unwrap_or(0);
    
    // Conversion Rate (orders/sessions)
    let sessions = sqlx::query_scalar!(
        "SELECT COUNT(*) FROM sessions WHERE created_at >= $1 AND created_at < $2",
        start_of_day,
        end_of_day
    )
    .fetch_one(&mut *conn)
    .await?
    .unwrap_or(0);
    
    let orders = sqlx::query_scalar!(
        "SELECT COUNT(*) FROM orders WHERE created_at >= $1 AND created_at < $2",
        start_of_day,
        end_of_day
    )
    .fetch_one(&mut *conn)
    .await?
    .unwrap_or(0);
    
    let conversion_rate = if sessions > 0 {
        orders as f64 / sessions as f64 * 100.0
    } else {
        0.0
    };
    
    // Average Order Value
    let avg_order_value = sqlx::query_scalar!(
        r#"
            SELECT COALESCE(AVG(total_amount), 0) 
            FROM orders 
            WHERE created_at >= $1 AND created_at < $2 
            AND status = 'completed'
        "#,
        start_of_day,
        end_of_day
    )
    .fetch_one(&mut *conn)
    .await?
    .unwrap_or(rust_decimal::Decimal::ZERO);
    
    Ok(BusinessMetrics {
        daily_active_users: dau,
        monthly_active_users: mau,
        conversion_rate,
        average_order_value: avg_order_value,
        customer_lifetime_value: rust_decimal::Decimal::ZERO, // Complex calculation
        churn_rate: 0.0, // Requires historical analysis
        revenue_growth: 0.0, // Requires comparison with previous period
    })
}

pub async fn track_feature_usage(
    conn: &mut DbConn,
    user_id: Uuid,
    feature_name: &str,
    context: Option<serde_json::Value>,
) -> Result<(), Error> {
    sqlx::query!(
        r#"
            INSERT INTO feature_usage_events (
                user_id, feature_name, context, created_at
            ) VALUES ($1, $2, $3, NOW())
        "#,
        user_id,
        feature_name,
        context
    )
    .execute(conn)
    .await?;
    
    // Log for real-time analytics
    tracing::info!(
        user_id = %user_id,
        feature = %feature_name,
        context = ?context,
        "Feature usage tracked"
    );
    
    Ok(())
}
```

**Usage in Service Layer:**
```rust
// Example usage in service functions
use crate::shared::observability::{LogContext, MetricsCollector, log_business_event};

pub async fn create_user(
    conn: &mut DbConn,
    metrics: Arc<MetricsCollector>,
    req: CreateUserRequest,
) -> Result<User, Error> {
    let ctx = LogContext::new("create_user", "users_service");
    let start_time = Instant::now();
    
    // Create span for distributed tracing
    let span = info_span!("create_user", user_email = %req.email);
    
    async move {
        // Business logic
        let user = User {
            id: Uuid::now_v7(),
            email: req.email.clone(),
            created_at: Utc::now(),
            // ... other fields
        };
        
        // Database operation with metrics
        let query_start = Instant::now();
        let result = sqlx::query!(
            "INSERT INTO users (id, email, created_at) VALUES ($1, $2, $3)",
            user.id,
            user.email,
            user.created_at
        )
        .execute(conn)
        .await;
        
        let query_duration = query_start.elapsed();
        metrics.record_db_query("INSERT", "users", query_duration, result.is_ok());
        
        match result {
            Ok(_) => {
                // Record business metrics
                metrics.record_business_event("user_registered", Some(user.id), None);
                
                // Log business event
                log_business_event(
                    &ctx.with_user(user.id),
                    "user_registered",
                    serde_json::json!({
                        "email": user.email,
                        "registration_source": "api"
                    }),
                    Some(vec![("registration_time_ms".to_string(), start_time.elapsed().as_millis() as f64)])
                );
                
                Ok(user)
            }
            Err(e) => {
                log_error_with_context(&ctx, &e.into(), Some(serde_json::json!({
                    "attempted_email": req.email
                })));
                Err(e.into())
            }
        }
    }
    .instrument(span)
    .await
}
```

## 10. General Architecture Principles

### Domain Organization
```
src/
├── {domain}/           # Each domain follows same structure
│   ├── api/           # HTTP handlers (thin layer)
│   ├── models/        # Data structures, DTOs
│   ├── repositories/  # Database queries
│   ├── services/      # Business logic
│   └── middleware.rs  # Domain-specific middleware (optional)
├── shared/            # Cross-cutting concerns
└── main.rs           # Application entry
```

### Testing Strategy
```rust
// Unit tests for pure functions
#[cfg(test)]
mod tests {
    use super::*;
    
    #[test]
    fn test_validation_logic() {
        let result = validate_username("ab");
        assert!(result.is_err());
    }
}

// Integration tests with real database
#[tokio::test]
async fn test_user_creation_flow() {
    let app = spawn_app().await;
    // Test complete flow...
}
```

### Performance Considerations
- Use connection pooling with appropriate limits
- Implement caching for frequently accessed data
- Batch database operations where possible
- Use database indexes for common queries
- Consider read replicas for analytics queries

### Security Best Practices
- Never log sensitive data (passwords, tokens)
- Use parameterized queries (SQLx does this)
- Validate all user input
- Implement rate limiting
- Use secure session management
- Hash passwords with strong algorithms

## Implementation Roadmap

### Phase 1: Foundation (Week 1-2)
- [ ] Set up project structure
- [ ] Implement unified error handling
- [ ] Create shared validation module
- [ ] Set up integration test framework

### Phase 2: Core Features (Week 3-4)
- [ ] Implement authentication module
- [ ] Create user management
- [ ] Add repository pattern
- [ ] Set up domain events

### Phase 3: Advanced Features (Week 5-6)
- [ ] Add caching layer
- [ ] Implement analytics
- [ ] Create admin dashboard
- [ ] Add monitoring/metrics

### Phase 4: Production Ready (Week 7-8)
- [ ] Performance optimization
- [ ] Security audit
- [ ] Documentation
- [ ] Deployment setup

### Phase 5: Enterprise Features (Week 9-10)
- [ ] Event sourcing implementation
- [ ] CQRS pattern adoption
- [ ] Advanced observability
- [ ] Circuit breakers and resilience

## Measuring Success

### Performance Metrics
- Test execution time < 30 seconds
- API response time < 100ms (p99)
- Database query time < 50ms (p95)
- Event processing latency < 10ms (p95)

### Code Quality Metrics
- Test coverage > 80%
- Zero critical security issues
- Code duplication < 5%
- Cyclomatic complexity < 10

### Developer Experience
- New developer onboarding < 1 day
- Feature development cycle < 3 days
- Bug fix turnaround < 1 day

### Business Metrics
- System uptime > 99.9%
- Error rate < 0.1%
- Customer satisfaction > 4.5/5
- Feature adoption rate > 60%

---

This comprehensive architecture guide provides 10 battle-tested patterns for building maintainable, scalable, and observable web applications. Each pattern addresses specific challenges in modern software development:

1. **Integration Test Architecture** - Fast, reliable testing with real services
2. **Test Database Optimization** - Template databases for rapid test execution
3. **Repository Pattern** - Clean data access with function-based approach
4. **Shared Module Architecture** - Reusable cross-cutting concerns
5. **Unified Error Handling** - Consistent error management across the application
6. **Domain Events System** - Decoupled communication and audit trails
7. **Validation Rules Engine** - Centralized business rule validation
8. **Event Sourcing and CQRS** - Event-driven architecture with read/write separation
9. **Observability and Telemetry** - Comprehensive monitoring and business insights
10. **General Architecture Principles** - Foundational guidelines for consistent development

These patterns can be adapted to various business domains while maintaining consistency, quality, and production readiness.

### Testing Strategy
```rust
// Unit tests for pure functions
#[cfg(test)]
mod tests {
    use super::*;
    
    #[test]
    fn test_validation_logic() {
        let result = validate_username("ab");
        assert!(result.is_err());
    }
}

// Integration tests with real database
#[tokio::test]
async fn test_user_creation_flow() {
    let app = spawn_app().await;
    // Test complete flow...
}
```

### Performance Considerations
- Use connection pooling with appropriate limits
- Implement caching for frequently accessed data
- Batch database operations where possible
- Use database indexes for common queries
- Consider read replicas for analytics queries

### Security Best Practices
- Never log sensitive data (passwords, tokens)
- Use parameterized queries (SQLx does this)
- Validate all user input
- Implement rate limiting
- Use secure session management
- Hash passwords with strong algorithms

## Implementation Roadmap

### Phase 1: Foundation (Week 1-2)
- [ ] Set up project structure
- [ ] Implement unified error handling
- [ ] Create shared validation module
- [ ] Set up integration test framework

### Phase 2: Core Features (Week 3-4)
- [ ] Implement authentication module
- [ ] Create user management
- [ ] Add repository pattern
- [ ] Set up domain events

### Phase 3: Advanced Features (Week 5-6)
- [ ] Add caching layer
- [ ] Implement analytics
- [ ] Create admin dashboard
- [ ] Add monitoring/metrics

### Phase 4: Production Ready (Week 7-8)
- [ ] Performance optimization
- [ ] Security audit
- [ ] Documentation
- [ ] Deployment setup

## Measuring Success

### Performance Metrics
- Test execution time < 30 seconds
- API response time < 100ms (p99)
- Database query time < 50ms (p95)

### Code Quality Metrics
- Test coverage > 80%
- Zero critical security issues
- Code duplication < 5%
- Cyclomatic complexity < 10

### Developer Experience
- New developer onboarding < 1 day
- Feature development cycle < 3 days
- Bug fix turnaround < 1 day

---

This architecture guide provides a solid foundation for building maintainable, scalable web applications. The patterns are battle-tested and can be adapted to various business domains while maintaining consistency and quality.