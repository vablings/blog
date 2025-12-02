---
title: "Creating a Dioxus fullstack application with Auth"
date: 2025-11-30
---


# How to make a fullstack dioxus application with Auth
Just some notes before we begin, This example is using sqlx + sqlite as my example. Under `axum_auth_slqx` there is a feature flag for each major database that sqlx supports so you can configure as you want. The important thing is to get a simple grasp of the high level concepts here and once you understand it you wont have much trouble swapping out say Sqlite for Postgres

## Creating a basic fullstack applicatoon

``` dx new fullstack-auth ```

Select the following : barebones -> Fullstack: True -> Router: True -> TailwindCSS: True -> Platform default -> Web

You then can strip down main.rs to the following 

```rs
/// main.rs
use dioxus::prelude::*;

const FAVICON: Asset = asset!("/assets/favicon.ico");
const MAIN_CSS: Asset = asset!("/assets/main.css");
const HEADER_SVG: Asset = asset!("/assets/header.svg");
const TAILWIND_CSS: Asset = asset!("/assets/tailwind.css");


#[derive(Debug, Clone, Routable, PartialEq)]
#[rustfmt::skip]
enum Route {
    #[route("/")]
    Home {},
}

fn main() {
    #[cfg(feature = "web")]
    dioxus::launch(App);

    #[cfg(feature = "server")]
    dioxus::serve(|| async move {
        let router = dioxus::guard::router(App);
        Ok(router)
    })
}


#[component]
fn App() -> Element {
    rsx! {
        document::Link { rel: "icon", href: FAVICON }
        document::Link { rel: "stylesheet", href: MAIN_CSS } document::Link { rel: "stylesheet", href: TAILWIND_CSS }
        Router::<Route> {}
    }
}

/// Home page
#[component]
fn Home() -> Element {
    rsx! {
        div { 
            "Homepage!"
        }
    }
}

```


## Fullstack splitting 

First we need to add `dioxus_fullstack` using `cargo add dioxus_fullstack` to get access to dioxus_fullstack

Once we have our basic application the first thing we needs to do is split up our fullstack application and served web. This is because we will be modifying the router to add a custom middleware to insert sessions, page guarding and auth

```rs
fn main() {
    #[cfg(feature = "web")]
    dioxus::launch(App);

    #[cfg(feature = "server")]
    dioxus::serve(|| async move {
        let router = dioxus::guard::router(App);
        Ok(router)
    })
}
```

So here we split the call to `App` into two conditionally compiled routes. The first route does the usual dixous magic and creates the web platform binary that has our WASM magic and the second does the same.

The difference here is that we are now exposing the router under `dioxus::serve` so we can add our own middleware.

### What is middleware?

Middleware is pretty much what it says on the tin, it is in the middle of a request and response. It is executed on a per request basis. In axum the extractor will give your function access to `Request` and `Next` with a return type of `Response`. You can read more about it [here](https://docs.rs/axum/latest/axum/middleware/fn.from_fn.html)

### Creating a basic middleware 

We can add our middleware just anywhere into `main.rs` since dioxus fullstack will handle the conditionals for us, Although if we want to make use of code that is exclusively on  the server then it can be quite annoying to manage feature gating and very quickly it becomes a horrible mess of `#[cfg(feature = "server")]` all over your code on imports and function defintions/structures

So we will create a new source file that is conditionally included for our server compilation called `guard.rs`

```rs
// new file guard.rs
use dioxus_fullstack::{
    axum::middleware::Next,
    extract::Request,
};

pub async fn my_middleware(
    request: Request,
    next: Next,
) -> Response {

    let response = next.run(request).await;

    response
}
```

Now in our main.rs we can include `guard.rs` conditionally 
```rs
#[cfg(feature = "server")]
mod guard;
```

Going back to the example from axum of `async fn my_middleware(...)` we can see that we have the `Request` which is the origin for the request, then we have `next` which is where the request going to go and then the return type of `Response` which can be controller by the users. We then can use this to modify the flow by returning any type that has the trait of `Response` for example 


Lets just run a small test here to make sure our middleware is working 
```rs
//main.rs
#[derive(Debug, Clone, Routable, PartialEq)]
#[rustfmt::skip]
enum Route {
    #[route("/")]
    Home {},
    #[route("/test")]
    Test {},
}

#[component]
fn Test() -> Element {
    rsx! {
        div { 
            "If you are seeing this. You have been redirected by my middleware! Muahahah"
        }
    }
}

```

First we create a new Route using our shiny dioxus Rouer and add a new component.
Then in our logic we need to make the middleware modify our request flow, As long as our return type has the Response trait we can return pretty much whatever we want. In this case the most common example is gonna be a redirect 

```rs
//guard.rs
use dioxus_fullstack::{
    axum::middleware::Next,
    extract::Request,
    // Dont forget to import the trait for into_response and the Redirect 
    response::{IntoResponse, Response},
    Redirect, 
};

// Sinc we are in a new source file we need to include our Router
use crate::Route;


pub async fn my_middleware(
    request: Request,
    next: Next,
) -> Response {

    //example testing value hardcoded
    return Redirect::to(&Route::Test {}.to_string()).into_response();
 
}
```

So as the above code shows we simply Redirect to another Route on our Router and then call `into_response()` to convert the Redirect call into a valid `Response`

Finally all we have to do is now add our middleware into the application so that the axum router calls our code during the request flow. Since middleware requires alot of boilerplate to get working for our simple example we can make use of `from_fn` which automatically passes over the function arguments of `Request` and `Next`


```rs
//main.rs
fn main() {
    #[cfg(feature = "web")]
    dioxus::launch(App);

    #[cfg(feature = "server")]
    dioxus::serve(|| async move {
        use dioxus_fullstack::axum::middleware::from_fn;

        let router = dioxus::guard::router(App)
                        .layer(from_fn(crate::guard::my_middleware));
        Ok(router)
    })
}

```

Now if we run `dx serve` we see something pretty bad 
```
15:12:07 [server] [303] /test 
15:12:07 [server] [303] /test 
15:12:07 [server] [303] /test 
15:12:07 [server] [303] /test 
15:12:07 [server] [303] /test 
15:12:07 [server] [303] /test 
15:12:07 [server] [303] /test 
15:12:07 [server] [303] /test 
15:12:07 [server] [303] /test 
```

Oops, we are going in a loop forever and ever since we are redirecting to the page even if we are already on that page. We should probably check to see if we are on the page 

```rs
//guard.rs
fn redirect_page(path: &str) -> bool {
    let Ok(route) = path.parse::<Route>() else {
        return true;
    };
    matches!(route, Route::Test {})
}
``` 
So we are just parsing the given path as a `&str` into a valid `Route` and then check to see if our route is one of the excluded routes that are not to be matched on

```rs
//guard.rs
pub async fn my_middleware(
    request: Request,
    next: Next,
) -> Response {
    //get the expected path from the Request
    let path = request.uri().path();

    //example testing value hardcoded


    if redirect_page(path) { 
        // Continue the request as normal!
        return next.run(request).await;
    }


    return Redirect::to(&Route::Test {}.to_string()).into_response();
}


```

Now if we run `dx serve` again we should be greeted with our message 

```
If you are seeing this. You have been redirected by my middleware! Muahahah
```

So we can see that our middleware is working. And that our code is fairly straightforward and easy to manage. Lets do a small recap here to go over the key concepts
 - We define a middleware that takes `Request` and `Next` which can modify our requesat flow for ever request
 - We can then add that middleware to our `dioxus::serve` using `.layer` and `from_fn`


So based on this we can check every single request and determine if a user has the correct permissions to see a given page. If they dont we can even redirect them to a login!

## Creating a fullstack auth

First before we get started we are going to need a few tools to make our life much easier
- A database of our choosing (This example uses sqlite)
- A session manager
- Password hashing & Encryption 
- Some method to talk to our database with Rust (This example uses sqlx)

First we need to setup our database. This can all be managed by sqlx easily but first we need to install the cli to create and run our migrations
`cargo install sqlx-cli`

In the sqlx docs you are required to provide the database to sqlx-cli. The easiest way is to just create an .env file in the root of your project directory
```
DATABASE_URL="sqlite://fullstack.db?mode=rwc"
```
Here i use `mode=rwc` meaning read/write/create so that if the file doesnt exist yet it will create a new file for us. How convenient 

Next we need to head over to our `Cargo.toml` to add the dependencies needed
```toml
serde = {version = "1.0.159", features = ["derive"]}
anyhow = "1.0.71"


sqlx = { version = "0.8.6", features = ["sqlite", "runtime-tokio-native-tls"], optional = true }
async-trait = { version = "0.1.71", optional = true }
axum_session = { version = "0.17.1", optional = true } 
axum_session_auth = { version = "0.17.0", optional = true } 
axum_session_sqlx = { version = "0.6.1", features = ["sqlite"], optional = true } 
password-auth = { version = "1.0.0", optional = true }
```

Okay woah, This is alot of deps but most of them are pretty obvious if you have ever done anything web/rust related before but I will breakdown some of the more obscure ones

- `async-trait` We will need this later with creating our own session/auth because rust out of the box does not support trait implementations that are async in stable rust (This is a large oversimplification of the issue and you can read more about it [here](https://github.com/dtolnay/async-trait) but rest assured this is another banger crate from dtolnay)
- `password-auth` This is just a high-level opininated password authentication library that has as little user choice and complexity as possible. If you want a well rounded discussion on passwords then [Zero To Production In Rust](https://www.zero2prod.com/index.html?country_code=US) is an great book that for both non-web/non rust folks alike.

Something you will notice is that all my deps are tagged as optional. This is because they are crates for both `web` and `server` but since web is targeting `wasm32-unknown-unknown` some of the crates will not compile on that platform. 

```toml
# cargo.toml
# note: i removed desktop and mobile since thats not in scope for this
[features]
default = ["web"]
web = ["dioxus/web"]
server = ["dioxus/server", 
            "dep:async-trait",
            "sqlx",
            "tokio",
            "axum_session",
            "axum_session_auth",
            "axum_session_sqlx",
            "password-auth",
            ]

```

### Using the database in our code
Now that we have our deps included in the source tree for our web binary we need to talk to our database so we can create a new file `db.rs`

```rs
//db.rs
use sqlx::{Executor, Pool, Sqlite};
use tokio::sync::OnceCell;

pub static DB: OnceCell<Pool<Sqlite>> = OnceCell::const_new();


async fn db() -> Pool<Sqlite> {
    use sqlx::SqlitePool;

    let pool = sqlx::sqlite::SqlitePool::connect("sqlite://fullstack.db?mode=rwc")
        .await
        .unwrap();


    pool
}


pub async fn get_db() -> &'static Pool<Sqlite> {
    DB.get_or_init(db).await
}
```

The code here is pretty self explanatory. We use a OnceCell that creates our `SqlitePool`
Dont forget in our `main.rs` to include this and feature-gate it to `server`

```rs
//main.rs
#[cfg(feature = "server")]
mod guard;

#[cfg(feature = "server")]
mod db;
```


## Making use of axum_session and axum_session_auth

So axum session and axum session auth will be exposing some traits and types that allow us to then inject some middleware that automatically manages our session and authorisation. First go ahead and make a new file of `layer.rs`

```rs
//layer.rs
use axum_session_auth::*;
use axum_session_sqlx::SessionSqlitePool;

pub(crate) type Session = axum_session_auth::AuthSession<User, i64, SessionSqlitePool, SqlitePool>;
pub(crate) type AuthLayer = axum_session_auth::AuthSessionLayer<User, i64, SessionSqlitePool, SqlitePool>;

#[derive(Clone)]
pub struct User {
    pub id: i64,
    pub anonymous: bool,
    pub username: String,
}

#[derive(sqlx::FromRow, Clone)]
pub struct SqlUser {
    id: i32,
    username: String,
    password: String,
}

```

So we can just typedef our `Session` and `AuthLayer` then we need to create a `User` and `SqlUser`. We split it up so that our `User` is passed around to the client and `SqlUser` is controlled by our auth to validate the password etc. There are some cases where we might need to call an operation directly on our SqlUser outside of this module so we can expose that behaviour 

```rs
//layer.rs
impl SqlUser {
    pub fn check_password(&self, password: String) -> bool {
        let verify = password_auth::verify_password(password, &self.password);
        match verify {
            Ok(_) => true,
            Err(_) => false,
        }
    }
    pub fn id(&self) -> i64 {
        self.id as i64
    }
}
```

#### Implementing our `#[async_trait]`

```rs
//layer.rs
#[async_trait]
impl Authentication<User, i64, SqlitePool> for User {
    async fn load_user(userid: i64, pool: Option<&SqlitePool>) -> Result<User, anyhow::Error> {
        let db = pool.unwrap();

        let sqluser = sqlx::query_as::<_, SqlUser>("SELECT * FROM users WHERE id = $1")
            .bind(userid)
            .fetch_one(db)
            .await
            .unwrap();

        Ok(User {
            id: sqluser.id as i64,
            anonymous: if userid == 1 { true } else { false },
            username: sqluser.username,
        })
    }

    fn is_active(&self) -> bool {
        !self.anonymous
    }

    fn is_anonymous(&self) -> bool {
        self.anonymous
    }
    fn is_authenticated(&self) -> bool {
        !self.anonymous
    }
}
```
This example is directly ripped from [axum_session_auth](https://github.com/AscendingCreations/AxumSessionAuth) but the code is very straightforward and explains everything. Once these trait behaviours have been defined we then can call the functions on auth.

We also now need to create our users table in the database!
`sqlx migrate add create_users`


```sql
// migrations/20251201013417_create_users.sql
create table if not exists users
(
    id integer primary key not null,
    username text not null unique,
    password text not null
);

-- Insert "guest" user.
insert into users (id, username, password)
values (1, 'guest','thispasswordisimpossibletoinsert');

```
Note here that we plan to use password hashing so it will be impossible for any user to directly log into the guest user. It will be the default user (userid = 1) so it must exists hence why we create it during the migration.

`sqlx migrate run`

Back round to how we need to then access the defined trait behaviour for our `User` it must be added as a layer/middleware to our axum/dioxus_fullstack router then we can make use of a magic tool called an extractor (Small note here, you could also use the extractor to pass around an instance of your database! I have left this as an exercise for the reader)

Heading on over to `main.rs` we can modify our router as follows 

```rs
//main.rs
#[cfg(feature = "server")]
dioxus::serve(|| async move {
    use axum_session::SessionConfig;
    use axum_session::SessionStore;
    use axum_session_auth::AuthConfig;
    use axum_session_sqlx::SessionSqlitePool;
    use dioxus_fullstack::axum::middleware::from_fn;

    let pool = crate::backend::db::get_db().await.clone();

    let session_config = SessionConfig::default().with_table_name("sessions");
    let auth_config = AuthConfig::<i64>::default().with_anonymous_user_id(Some(1));
    let session_store =
        SessionStore::<SessionSqlitePool>::new(Some(pool.clone().into()), session_config)
            .await
            .unwrap();

    ...
```

So we need to create our `session_config` we can then predefine our session tablename as just `sessions` then create our auth config. where the turbofished `i64` is the type for our UserID. We then can set our default userid for an unauthenticated user as `1`

After that just creating a session store with an instance of our db and pass in our session config. Calling unwrap here is fine since if we cant even setup our session_store then we just blow the whole thing up 

```rs
//main.rs
        let router = dioxus::server::router(App)
            .layer(from_fn(crate::guard::my_middleware))
            .layer(
                axum_session_auth::AuthSessionLayer::<
                    backend::auth::layer::User,
                    i64,
                    SessionSqlitePool,
                    sqlx::SqlitePool,
                >::new(Some(pool.clone()))
                .with_config(auth_config),
            )
            .layer(axum_session::SessionLayer::new(session_store));
        Ok(router)
```
Then we just add our layers for our session auth and thats pretty much it right!
All we need to do now is define some server endpoints to call that have extractors injected. Dioxus fullstack makes this very very easy!

Go ahead and create our new `api.rs`

```rs
//api.rs

#[post("/api/user/register", auth: Session)]
pub async fn register(
    username: String, password: String, confirm_password: String,
) -> Result<String, HttpError> {
    ...
}
```
So notice here we have used the extractor to insert our session into the query. with `auth: Session` This is pretty much an argument to our function that is always saturated by our extractor, it will always be present so we can make use of it get information about our users session.


Okay so now we can see how the extractors work lets make a basic register function that creates us a new user account 

```rs
//api.rs
use dioxus::prelude::*;

#[cfg(feature = "server")]
use crate::layer::Session;

#[post("/api/user/register", auth: Session)]
pub async fn register(
    username: String, password: String, confirm_password: String,
) -> Result<String, HttpError> {
    use crate::layer::SqlUser;
    use crate::db::get_db;

    let pool = get_db().await.clone();

    let rows: Vec<SqlUser> = sqlx::query_as("SELECT * FROM users WHERE username = $1")
        .bind(&username)
        .fetch_all(&pool)
        .await
        .unwrap(); 

    if rows.len() != 0 {
        return HttpError::bad_request("User already exists!");
    } else {
        let hash = password_auth::generate_hash(password);
        let rowid = sqlx::query("INSERT INTO users (username, password) VALUES (?1, ?2)")
            .bind(&username)
            .bind(&hash)
            .bind(2)
            .execute(&pool)
            .await
            .unwrap()
            .last_insert_rowid();
        auth.login_user(rowid);
        Ok("Logged in!".to_string())
    }
}

```

So first we need to query our database to see if the username already exists. We can just return early a bad request. Since we are making use of `HttpError` we can actually pass a string into our error code and it supports `Display` meaning that later on we can reactively place the error somewhere on the webpage showing both the status code and also the inner error text value, Very ergonomic

At this point you should probably add some other checks such as password length requirements. In this example im just going to ensure that password, and confirm_password match 


```rs
//api.rs
...
if password != confirm_password {
    return HttpError::bad_request("Passwords do not match!");
}
...
```

I am not 100% sure if its best to do these kinds of checks on the server rather than checking them in the dioxus hooks later for the front end but it doesn't really matter for this application and we can deal with that error very easily.

Now that we have our register endpoint we can now create a component that submits the data to our server.

```rs
//main.rs

#[derive(Debug, Clone, Routable, PartialEq)]
#[rustfmt::skip]
enum Route {
    #[route("/")]
    Home {},
    #[route("/test")]
    Test {},
    #[route("/register")]
    Register {},
}
...
#[component]
pub fn Register() -> Element {
    let mut username = use_signal(|| String::new());
    let mut password = use_signal(|| String::new());
    let mut confirm_password = use_signal(|| String::new());
    let mut response = use_signal(|| String::new());

    rsx! {
        div { 
            "{username:?}",
        }
        input {
            placeholder: "Username",
            oninput: move |event| {
                username.set(event.value());
            },
        }
        input {
            placeholder: "Password",
            oninput: move |event| {
                password.set(event.value());
            },
        }
        input {
            placeholder: "confirm_password",
            oninput: move |event| {
                password.set(event.value());
            },
        }
    }
}
```

Lets also not forget to add our route to the exceptions list so we can visit the page without redirections. Now we can run `dx serve` and visit our new register page over at `http://127.0.0.1:8080/register`

```rs
fn redirect_page(path: &str) -> bool {
    let Ok(route) = path.parse::<Route>() else {
        return true;
    };
    matches!(route, Route::Test {} | Route::Register {})
}
```

Now when we go over to our register page we can see our login, password and confirm password!. So lets make an account to our new service. But wait, how can we submit our login, We need a button to submit the data and call our backend function to register the user. The dioxus documentation states that you can make use of a `Form<T>` but i consider this bad because its `url-encoded`

So lets just call the function with our `use_signal` hooks


```rs
button {
    onclick: move |_| async move {
        response.set(String::new());
        let server_response = register(
                username(),
                password(),
                confirm_password(),
            )
            .await;
        match server_response {
            Ok(_) => {
                navigator().replace(Route::JobList {});
            }
            Err(e) => {
                response.set(format!("{}", e).to_string());
            }
        }
    },
    "Sign up"
}
```
And just like that we have created our basic registration page! In the logic for our endpoint we take the rowid of the last insertion and just set that as the `auth.login_user` so we should be already logged into the account we just created! Lets quickly modify our homepage to display our current username. We are still being redirected by the middleware we created earlier. Lets change this to an actual auth check that validates the current user to see if they are `userid = 1` 

```rs
#[cfg(feature = "server")]
use crate::layer::*;

fn is_unsecured_route(path: &str) -> bool {
    let Ok(route) = path.parse::<Route>() else {
        return true;
    };
    matches!(route, Route::Register {})
}

fn is_unsecured_endpoint(path: &str) -> bool {
    matches!(path, "/api/user/register")
}

pub async fn auth_check(
    auth_session: Session, // New axum extractor for our session
    request: Request, 
    next: Next
) -> Response {
    let path = request.uri().path();

    let is_unsecured_route = is_unsecured_route(path)
        || is_unsecured_endpoint(path)
        || path.starts_with("/wasm/") // IMPORTANT; not allowing this will break any dioxus hooks and stop your website from working
        || path.starts_with("/assets/");


    if !is_unsecured_route && !auth_session.is_authenticated() {
        return Redirect::to(&Route::Register {}.to_string()).into_response();
    }

    next.run(request).await
}
```

Dont forget to rename our middleware in the `main.rs`

```rs
...
let router = dioxus::server::router(App)
        .layer(from_fn(crate::guard::auth_check))
...
```

Congratulations. You now have all the tools in the box to write your own auth

## Wrapping up and exercises for the reader

- Adding a login system with a login page and endpoints
- Whatever your heart can imagine.



