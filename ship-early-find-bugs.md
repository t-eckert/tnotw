# Ship Early. Find Bugs.

As soon as I start a project, I get it shipped to production with a Git-push based pipeline. I do this even before the project _is_ anything. It means I can go through the motions of getting changes out to users when I don't even have any yet. What's more, there are problems that don't show up running on `localhost`. Solving those problems one-at-a-time is much less stressful than fixing them all at once on some pre-defined ship date.

Along with getting the project shipped early, I will instrument it to help me debug the code in production. That paid off with my latest project [Devy](https://devy.page) which I plan on sharing with early users next month.

The API for Devy runs on [fly.io](https://fly.io) which enables Grafana dashboards by default. Looking over these logs the other day, I noticed a `500` error. Oops.

A fact of life, running code open to the internet, is that when you configure an SSL cert, the public record of this act attracts bots. I have seen this with every project I've shipped that uses LetsEncrypt, but I'm certain it happens with any certificate authority.

Within milliseconds of deploying the API to Fly and getting the automatically generated SSL cert, I will see logs of bots hitting the API trying to extract credentials. In the screenshot below, a bot tried to hit an endpoint `api.devy.page/blogs/wp-login.php`.

<Image src={Logs} alt="Grafana logs from running Devy in production. The logs indicate a not found error returns a 500." class="object-cover rounded-md mb-4"/>

The `wp-login.php` page is a common target of brute force attacks trying to guess passwords to log in to Wordpress sites. Devy is not Wordpress based, but the cost to try for a security hole is essentially zero for these botnets.

Reading through the logs, which are in reverse chronological order, the API tries to look up a blog with the slug and instead of returning a `404` which it should, it returns a `500`. Why?

This request passes through two Rust crates: `router` and `db`.

First the `router` code.

```rust
/// GET /blogs/:blog_slug
///
/// Get a blog from the database given a blog slug.
async fn get_blog_by_blog_slug(
    State(store): State<Store>,
    Path(blog_slug): Path<String>,
) -> Result<Json<Blog>> {
    Ok(Json(blog::get_by_slug(&store.db, blog_slug).await?))
}
```

This calls calls the function `get_by_slug` in the `blog` module of the `db` crate.

```rust
pub async fn get_by_slug(db: &Database, slug: String) -> Result<Blog> {
    Ok(
        sqlx::query_file_as!(Blog, "src/blog/queries/get_by_slug.sql", slug)
            .fetch_one(db)
            .await?,
    )
}
```

The `?` in the fifth line is somewhat like throwing an error in other languages. In Rust, it means return the `Err` type on the `Result`. This `Result` is a custom type for the `db` crate.

```rust
pub type Result<T> = std::result::Result<T, Error>;

/// Errors that can occur while performing an action on an entity.
#[serde_as]
#[derive(Debug, Serialize)]
pub enum Error {
    /// The database configuration is invalid.
    ConfigurationError(String),

    /// The requested entity was not found.
    EntityNotFound,

    /// The request was malformed.
    Malformed(String),

    /// A field was missing from the request.
    MissingField(String),

    /// An error occurred while interacting with the database.
    Sqlx(#[serde_as(as = "DisplayFromStr")] sqlx::Error),
}
```

This is one of my favorite features in Rust (and the source of this bug) which is defining a custom `Error` enum and a `Result` the encompasses all "failable" states in a crate.

Similarly, the `router` crate has a defined `Error` enum and a way to automatically translate `db` errors into `router` errors.

```rust
pub type Result<T> = std::result::Result<T, Error>;

#[serde_as]
#[derive(Debug, Serialize)]
pub enum Error {
    StatusCode(#[serde_as(as = "DisplayFromStr")] StatusCode),
    ServeFailure,
}

// --snip--

impl From<db::Error> for Error {
    fn from(_: db::Error) -> Self {
        Self::StatusCode(StatusCode::INTERNAL_SERVER_ERROR)
    }
}
```

This allows the behavior seen in the `get_blog_by_blog_slug` function in the `router` where `?` is used on a function in the `db` package: `blog::get_by_slug(&store.db, blog_slug).await?`. As the error passes back from the `db` crate to the `router` crate, it will get automatically transformed from a `db::Error` to a `router::Error`. Neat!

Except the `db` error in question is a `NotFound` error... and when it gets returned I am telling the `router` to transform it into a `StatusCode::INTERNAL_SERVER_ERROR`. Really, I'm saying no matter what error the `db` crate returns, transform it into a `StatusCode::INTERNAL_SERVER_ERROR`. Not great.

The fix is to add a match case where the `db` error `EntitiyNotFound` is transformed into the `404` status code. I also added the "malformed" and "missing field" cases too.

```rust
// BEFORE
impl From<db::Error> for Error {
    fn from(_: db::Error) -> Self {
        Self::StatusCode(StatusCode::INTERNAL_SERVER_ERROR)
    }
}

// AFTER
impl From<db::Error> for Error {
    fn from(err: db::Error) -> Self {
        match err {
            db::Error::EntityNotFound => Self::StatusCode(StatusCode::NOT_FOUND),
            db::Error::Malformed { .. } => Self::StatusCode(StatusCode::BAD_REQUEST),
            db::Error::MissingField { .. } => Self::StatusCode(StatusCode::BAD_REQUEST),
            _ => Self::StatusCode(StatusCode::INTERNAL_SERVER_ERROR),
        }
    }
}
```

I committed the change and pushed it. Now the bug is gone. I also added "looking up a blog that doesn't exist returns 404" to my integration test suite. Perfect!

<Image src={LogsFixed} alt="Grafana logs from running Devy in production. The logs indicate a not found error returned 404." class="object-cover rounded-md mb-4"/>

We all write bugs. It's what we do when we're not fixing bugs. I overlooked this pretty basic case because I was restructuring my code. Shipping and observing your application before you have any real users has helped me find all sorts of bugs which is why I'll always insist on it. Hopefully this inspires you to try the same.
