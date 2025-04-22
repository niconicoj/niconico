---
layout: ../../layouts/post-layout.astro
title: "Testing with setup and teardown with async rust"
publishedAt: "2023-11-18T10:00:00+00:00"
prelude:
  'I have recently started to follow Luca Palmieri''s "Zero to production" book.
  Quite early on in the book is the section about testing. The tests being
  presented are integration tests where each tests gets its own database
  instance. However, The book does not go into deleting the testing databases.
  this is not a bad decision but it was still bothering me that some unused
  resources were not freed. So I decided to program a setup/teardown system and
  went down on a little journey.'
---

I have recently started to follow
[Luca Palmieri's "Zero to production"](https://www.zero2prod.com/index.html?country=France&discount_code=VAT20&country_code=FR)
book. Quite early on in the book is the section about testing. The tests being
presented are integration tests where each tests gets its own database instance.
However, The book does not go into deleting the testing databases. this is not a
bad decision but it was still bothering me that some unused resources were not
freed. So I decided to program a setup/teardown system and went down on a little
journey.

## The starting point

What we have to start with is something that looks like this :

```rust
#[tokio::test]
async fn health_check_is_200() {
    let test_app = spawn_app().await;
    // Do some testing, using info provided by `test_app`
    // like the app address or the database connection
}

struct TestApp {
    pub app_address: String,
    pub db_pool: PgPool,
    pub db_name: String,
    pub db_address: String,
}

async fn spawn_app() -> TestApp {
    // Setup a listener on a random port and retrieve that port to build the full app address
    let listener = TcpListener::bind("127.0.0.1:0").expect("Failed to bind listener");
    let port = listener.local_addr().unwrap().port();
    let app_address = format!("http://127.0.0.1:{port}");

    // Retrieve a test configuration, the database name is randomized and is then
    // used to create a new database. A full migration is executed to bring it up to date.
    let test_config = zero2prod::configuration::get_test_configuration()
        .expect("Failed to read configuration");
    let db_pool = configure_database(&test_config).await;

    // spawn the test app
    tokio::spawn(zero2prod::spawn_app());
    TestApp {
        app_address,
        db_pool,
        db_name: test_config.database.name.clone(),
        db_address: test_config.database.address.clone()
    }
}
```

As you can see, we are creating a database for our test, and we actually do so
for each single test. But what you will not see is us cleaning up after
ourselves. Ideally we would not have to write any code regarding this cleanup in
any tests, it would be nicely abstracted away in some form. Let's work on that.

## First idea : Implement `Drop` for `TestApp`

So we want to delete our test database when we are done with the `TestApp`, and
what better moment do that than when we drop it ? I mean, when it gets dropped
we are absolutely certain we are done with it, and it would also mean that we
only write the cleanup code in a single location and our test would not have to
manually call anything to do the cleanup. So here is the code I first came up
with for the drop implementation :

```rust
impl Drop for TestApp {
    fn drop(&mut self) {
        // close the pool to allow us to delete the database
        self.db_pool.close().await;

        // connect to the root database
        let mut conn = PgConnection::connect(&self.db_address)
            .await
            .expect("Failed to connect to Postgres");

        // cleanup !
        conn.execute(format!(r#"DROP DATABASE "{}";"#, test_database_name).as_str())
            .await
            .unwrap();
        }
    }
}
```

Looks pretty good right ? Except... it does not work.

```bash
error[E0728]: `await` is only allowed inside `async` functions and blocks
  --> tests/common/test_app.rs:16:30
   |
15 | /     fn drop(&mut self) {
16 | |         self.db_pool.close().await;
   | |                              ^^^^^ only allowed inside `async` functions and blocks
17 | |         let mut conn = PgConnection::connect(&self.db_address)
18 | |             .await
...  |
22 | |             .unwrap();
23 | |     }
   | |_____- this is not `async`
```

`Drop` is not `async` and we can't make it so because that would not satisfy the
trait definition. but maybe there is something we can still try : using
`block_on`. What I came up with looks like this :

```rust
impl Drop for TestApp {
    fn drop(&mut self) {
        // get a handle on the current runtime and block on running cleanup code
        Handle::current().block_on(async {
            self.db_pool.close().await;
            let mut conn = PgConnection::connect(&self.db_address)
                .await
                .expect("Failed to connect to Postgres");
            conn.execute(format!(r#"DROP DATABASE "{}";"#, self.db_name).as_str())
                .await
                .unwrap();
        });
    }
}
```

This compiles ! So let's just run it, and...

```bash
$ cargo test
   Compiling zero2prod v0.1.0 (/home/niconico/dev/rust/zero2prod)
    Finished test [unoptimized + debuginfo] target(s) in 1.20s
     Running unittests src/lib.rs (target/debug/deps/zero2prod-57d994dc11bdee2e)

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

     Running unittests src/main.rs (target/debug/deps/zero2prod-2912ef6fce2a4714)

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

     Running tests/health_check.rs (target/debug/deps/health_check-1b3831da74fcd355)

running 1 test
test health_check_works ... FAILED

failures:

---- health_check_works stdout ----
thread 'health_check_works' panicked at tests/common/test_app.rs:17:27:
Cannot start a runtime from within a runtime. This happens because a
function (like `block_on`) attempted to block the current thread while the
thread is being used to drive asynchronous tasks.
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

Yeah, this does not work either. I am a bit foggy on the details here, async
rust is kind of hard sometimes and it is one of those times, but here is what I
think is happening. The message says we are trying to run blocking code on a
thread used to run asynchronous tasks which is not allowed. It seems like it
this is to prevent **very** easy deadlocks. I thought I could circumvent this
issue by running my test in a multi thread runtime instead of the current
thread, which is default when using `tokio::test` :

```rust
// instead of just #[tokio::test]
#[tokio::test(flavor = "multi_thread")]
async fn health_check_works() {
    let test_app = spawn_app().await;
    let client = reqwest::Client::new();

    let response = client
        .get(&format!("{}/health_check", test_app.app_address))
        .send()
        .await
        .expect("Failed to execute request");

    assert!(response.status().is_success());
    assert_eq!(Some(0), response.content_length());
}
```

This was throwing the same error... After digging around a little bit I quickly
found a
[draft story from the rust async working group](https://rust-lang.github.io/wg-async/vision/submitted_stories/status_quo/barbara_bridges_sync_and_async.html)
which describe pretty much the exact same thing I ran into and it does seems
like this is all about deadlocks. Also, it seems my instinct about using a
multithreaded runtime was somewhat correct but the tokio team probably decided
to still panic on block_on running on a worker thread to prevent inconsistent
behaviour between single and multi thread runtime.

Anyway, seems like we are at a dead end here. So lets explore an other avenue :
make a custom async test runner.

## Second idea : custom async test runner

Until now we have been using the `#[tokio::test]` attribute to mark our test.
This attribute desugars like this :

```rust
// this...
#[tokio::test]
async fn my_test() {
    assert!(true);
}

// ...desugars into this
#[test]
fn my_test() {
    tokio::runtime::Builder::new_current_thread()
        .enable_all()
        .build()
        .unwrap()
        .block_on(async {
            assert!(true);
        })
}
```

So we could build our own test macro to run that behaves similarly, except it
would run some more code before the test and would provide the test with some
data (namely, the `TestApp` struct). but macros are very daunting, so lets put
it off a little bit and just write a function for now. This would be easier and
would also act as a proof of concept before we dump any extra amount of time
into making a cool handy macro. Here is a first draft for our testing function :

```rust
pub fn run_test<T>(test: T) -> ()
where
    T: panic::UnwindSafe,
    T: FnOnce(TestApp) -> Pin<Box<dyn Future<Output = ()> + 'static + Send>>,
{
    let result = std::panic::catch_unwind(|| {
        tokio::runtime::Builder::new_current_thread()
            .enable_all()
            .build()
            .unwrap()
            .block_on(async {
                // spawning an app now returns a tuple so we can move `test_app` into the test
                // function while keeping everything we need to later delete the database
                let (test_app, test_database_name, db_conn_string) = spawn_app().await;
                let pool = test_app.db_pool.clone();

                // run some tests on the newly started app !
                test(test_app).await;

                pool.close().await;
                drop_database(test_database_name, db_conn_string).await;
            })
    });
    assert!(result.is_ok());
}

async fn drop_database(test_database_name: String, db_conn_string: String) {
    let mut conn = PgConnection::connect(&db_conn_string)
        .await
        .expect("Failed to connect to Postgres");

    conn.execute(format!(r#"DROP DATABASE "{}";"#, test_database_name).as_str())
        .await
        .unwrap();
}
```

I chose to move the `TestApp` into the test for simplicity and flexibility
regarding the test. So I had to make sure I kept all the info needed to later
delete the database on hand, hence the tuple and the cloning of the connection
pool. This was a bit finnicky to get to compile. In particular, I had to box the
future returned by the FnOnce because of the `dyn` part, and we also need `Pin`
because of async requirements.

Anyways, here is how we can now write our test (with a full example including
usage of the database connection pool) :

```rust
// using #[test] instead of #[tokio::test] because we are creating a runtime ourselves
#[test]
fn subscribe_returns_a_200_for_valid_form_data() {
    // using our test function !
    run_test(|test_app| {
        // we need our future to be in a pinned box
        Box::pin(async move {
            let client = reqwest::Client::new();

            let body = "name=John%20Doe&email=john.doe@gmail.com";
            let response = client
                .post(&format!("{}/subscriptions", test_app.app_address))
                .header("Content-Type", "application/x-www-form-urlencoded")
                .body(body)
                .send()
                .await
                .expect("Failed to execute request");

            assert_eq!(response.status(), StatusCode::OK);

            let saved = sqlx::query!("SELECT email, name FROM subscriptions")
                .fetch_one(&test_app.db_pool)
                .await
                .expect("Failed to fetch saved subscription.");

            assert_eq!(saved.email, "john.doe@gmail.com");
            assert_eq!(saved.name, "John Doe");
        })
    });
}
```

This _almost_ works but sometimes tests would fail with this message :
`database "newsletter-1faa4f82-d8ea-4665-9c01-058b90290ca0" is being accessed by other users`.
Seems like although we closed the pool some connection are being recorded by the
database. I banged my head for quite some time on this issue. The solution I
found was to run a sql query that would manually close the connection on the
database in addition to everything else.

```rust
async fn drop_database(test_database_name: String, db_conn_string: String) {
    let mut conn = PgConnection::connect(&db_conn_string)
        .await
        .expect("Failed to connect to Postgres");

    // the trick that makes everything works
    conn.execute(
        format!(
            r#"
            SELECT pg_terminate_backend(pg_stat_activity.pid)
            FROM pg_stat_activity
            WHERE pg_stat_activity.datname = '{}';
            "#,
            test_database_name
        )
        .as_str(),
    )
    .await
    .unwrap();

    conn.execute(format!(r#"DROP DATABASE "{}";"#, test_database_name).as_str())
        .await
        .unwrap();
}
```

And with that we are done ! Our test databases are being created and then
deleted when the tests are done ! As a note, if something goes wrong during a
test, then the database would not be delete. To me that is actually a very good
thing because it will surely help debugging failing tests.

I said we were done, right ? Well, not quite. We can still improve the
experience by providing a macro that would also take care of the box/pin stuff
in addition to everything else.

## Third (and final) idea : `#[integration_test]` macro

That would be so good, the idea is to make a macro that would allow writing this
:

```rust
#[integration_test]
async fn subscribe_returns_a_200_for_valid_form_data(test_app: TestApp) {
    // test some stuff on the app
}
```

And later expands into this :

```rust
#[test]
fn subscribe_returns_a_200_for_valid_form_data() {
    run_test(|test_app| {
        Box::pin(async move {
            // test some stuff on the app
        })
    });
}
```

Or at the very least somethine that _looks_ like this. The thing is that there
is something called **macro hygiene** which is a concept that can be summarized
as "make sure your macros can be used in as many contexts as possible". In our
case, this means that we have to use absolute path everywhere. so the actual
macro expansion would be more like this :

```rust
#[::core::prelude::v1::test]
fn subscribe_returns_a_200_for_valid_form_data() {
    ::zero2prod_core::testing::run_test(|test_app| {
        ::std::boxed::Box::pin(async move {
            // test some stuff on the app
        })
    });
}
```

When working on procedural macros three crates are pretty much ubiquitous :

- [`proc-macro2`](https://github.com/dtolnay/proc-macro2) which is a wrapper
  around rust's `proc_macro` crate.
- [`syn`](https://github.com/dtolnay/syn) which is a parser for rust token
  stream into syntax tree.
- [`quote`](https://github.com/dtolnay/quote) which does the opposite of `syn`.

I took **heavy** inspiration from tokio's `#[tokio::test]` macro when writing my
own integration test macro. I was able to simplify it to fit my use case. I
can't show the whole code since it is a bit too large, but here is the most
interesting snippet which shows how the output token stream is built :

```rust
fn parse_knobs(mut input: ItemFn) -> TokenStream {
    // alter the functions signature, it will be sync, without inputs and returns ()
    input.sig.asyncness = None;
    input.sig.inputs.clear();
    input.sig.output = ReturnType::Default;

    // the header is the standard test macro, except with absolute path
    let header = quote! {
        #[::core::prelude::v1::test]
    };

    // pull the function body and surround it with our handy `run_test` function
    // also notice absolute path everywhere
    let body = input.body();
    let body = quote! {
        ::zero2prod_core::testing::run_test(|test_app: ::zero2prod_core::testing::TestApp| {
            ::std::boxed::Box::pin(async move #body)
        });
    };

    // assemble all our token streams and return them
    input.into_tokens(header, body)
}
```

The whole code is available on
[my github](https://github.com/niconicoj/zero2prod/blob/2774140c4cb080bd8b0006ca200b7ef163acd01a/zero2prod-macros/src/entry.rs#L18)
!

## Conclusion

rust `async` is cool but some corners are still pretty sharp. Macro are cool
too, I really just dipped my little toe here but I am slightly more confident
with them now and I think I am more likely to consider using them in the future
when the opportunity presents itself.
