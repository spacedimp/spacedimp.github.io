+++
title = "Using Rust, Axum, PostgreSQL, and Tokio to build a Blog"
date = 2023-03-11
+++

### In this tutorial we'll be creating a very basic blog to get the hang of Axum.

### Sure, you could just use a static site generator and push the files up to Github pages, but where's the fun in that?

------

## Setting up the project


```bash
cargo new blog-rs --bin
```

The dependencies I'll be using go in `Cargo.toml`

```toml
[package]
name = "blog-rs"
version = "0.1.0"
edition = "2021"

[dependencies]
tokio = {version="1.13.0", features = ["macros", "rt-multi-thread"]}
axum = "0.6.4"
askama = {version="0.12.0", features=["markdown"]}
sqlx = {version = "0.6", features = ["runtime-tokio-rustls", "postgres", "macros", "time"]} 
tower-http = {version = "0.4", features=["full"]}
```

Edit `main.rs` and create a server at localhost:4000/

```rs
use axum::{http::StatusCode, routing::get, Router};

async fn index() -> String {
    String::from("homepage") 
}

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/", get(index));

    axum::Server::bind(&"0.0.0.0:4000".parse().unwrap())
        .serve(app.into_make_service())
        .await
        .unwrap();
}
```
Spin up the server with:

```bash
cargo run main.rs
```

<br>

## A brief introduction to Tokio and Axum

Let's unpack Axum and Tokio a bit.

[Axum](https://docs.rs/axum/latest/axum/) is a web framework built with Tokio, Hyper, and Tower. 

```rust
use axum::{http::StatusCode, routing::get, Router};
```  

<br>
    

[Tokio](https://docs.rs/tokio/latest/tokio/) allows us to run asynchronous non-blocking code (but it can also run blocking code if needed). Its componets include: 

* A scheduler that manages tasks pushed onto a run queue.
* An async I/O driver that enables using [net](https://docs.rs/tokio/latest/tokio/net/index.html), [process](https://docs.rs/tokio/latest/tokio/process/index.html), [signal](https://docs.rs/tokio/latest/tokio/signal/index.html). 
* A time driver that enables using `tokio::time` on the runtime.
* Core threads that should have no blocking code and [blocking](https://docs.rs/tokio/latest/tokio/task/fn.spawn_blocking.html) threads that can be spawned on demand to handle any blocking code.   

<br>

```rust
#[tokio::spawn]
async fn main() {
	// code here should never block
	// unless in a closure and passed to tokio::task::spawn_blocking()
}
```

This is the equivalent of 

```rust
fn main() {
	tokio::runtime::Builder::new_multi_thread()
		.enable_all()
		.build()
		.unwrap()
		.block_on(async {
			// Runtime's entry point
		})
}
```

Axum's Router matches a path to handler.

```rust
let app = Router::new()
        .route("/", get(index));
```

Handlers can accept zero or more extractors as arguments.  

The ordering of the extractors is important as only one extractor can consume the request's body. It should be placed as the last argument furthest to the right in your handler. 

Anything that implements the IntoResponse trait can be returned by handlers. Axum takes care of implementing it for common types.

```rust
async fn index() -> String {
    String::from("homepage") 
}
```

Just as an example, let's use Axum's TypedHeader extractor to send the user back their User-Agent (I'll remove this extractor and feature after this demonstration).

First I enable the headers feature in `Cargo.toml` 

```toml
axum = {version= "0.6.4", features = ["headers"]}
```

Next I import the extractor and edit the index handler to extract the user agent and send it back to the user as a response.

```rust
use axum::{
    http::StatusCode, routing::get, Router,
    extract::{TypedHeader},
    headers::UserAgent,
};

// go ahead and run "cargo run main.rs"
// localhost:4000 should now print out your user agent
async fn index(TypedHeader(user_agent): TypedHeader<UserAgent>) -> String {
    String::from(user_agent.as_str()) 
}
```


## Configuring the database

Let's get our database up and running. First make sure to [download](https://www.postgresql.org/download/) and install PostgreSQL. 

Make sure the service is started (I'm running linux so here's how I'd do it)

```bash
sudo systemctl start postgresql
``` 


Login using psql 

```bash
sudo -u postgres psql postgres
```

Setup a user and database (inside of psql run the following commands with your own username and password)

```sql
CREATE ROLE myuser LOGIN PASSWORD 'mypass';
CREATE DATABASE mydb WITH OWNER = myuser;
\q
```

Login with the new user and type in your password when prompted. In my case "mypass".

```bash
psql -h localhost -d mydb -U myuser
```


Create a table that will store our blog posts.

```sql
CREATE TABLE myposts(
post_id SERIAL PRIMARY KEY,
post_date DATE NOT NULL DEFAULT CURRENT_DATE,
post_title TEXT,
post_body TEXT)	
);
```

Great! Personally, I enjoy creating blog posts in [markdown](https://docs.github.com/en/get-started/writing-on-github/getting-started-with-writing-and-formatting-on-github/basic-writing-and-formatting-syntax) format. For my editor I use [Ghostwriter](https://ghostwriter.kde.org/). 

I say this because I'll be storing raw markdown into the field labeled post_body.

We can now connect our app to PostgreSQL

`main.rs`

```rust
use sqlx::postgres::PgPoolOptions;
use sqlx::FromRow;
use sqlx::types::time::Date;

use std::sync::Arc;

// the fields we'll be retrieving from an sql query

#[derive(FromRow, Debug, Clone)]
pub struct Post {
    pub post_title: String,
    pub post_date: Date,
    pub post_body: String,
}

#[tokio::main]
async fn main() {

    let pool = PgPoolOptions::new()
                .max_connections(5)
                // use your own credentials
                .connect("postgres://myuser:mypass@localhost/mydb")
                .await
                .expect("couldn't connect to the database");

	// I fetch all of the posts at the start of the program 
	// to avoid hitting the db for each page request
    let posts = sqlx::query_as::<_, Post>("select post_title, post_date, post_body from myposts") 
        .fetch_all(&pool)
        .await
        .unwrap();

	// Above we retrieved Vec<Post> 
	// We place it in an Arc for thread-safe referencing.  
    let shared_state = Arc::new(posts);

    let app = Router::new()
        .route("/", get(index))
        .route("/post/:query_title", get(post))
        // We pass the shared state to our handlers 
        .with_state(shared_state);
        
//

```

## Inserting markdown into the database

I suggest creating a new binary where we simply pass it a title and a markdown file as arguments.   

Edit Cargo.toml to include a second binary that will insert a markdown file into the database 

```toml
[[bin]]
name = "blog-rs"
path = "src/main.rs"

[[bin]]
name = "markd"
path = "src/bin/markd.rs"
```

Create a markdown file inside of src/bin/post.md with content of your choosing. Here's mine: 

src/bin/post.md

```md
# This is a post 

with some content 
```

Markd is very rudimentary. 

It lacks any capabilities besides inserting a single file into our database.  

Create src/bin/markd.rs

```rust
use std::env;
use sqlx::postgres::PgPoolOptions;
use std::fs::File;
use std::io::Read;

#[tokio::main]
async fn main() -> Result<(), sqlx::Error>{

	// collects the arguments when we run:
	// cargo run --bin markd "A title" ./post.md
	
    let args: Vec<String> = env::args().collect();

    let mut inserter;

	// argument 2 should contain the file name
    match File::open(&args[2]) {
        Ok(mut file) => {
            let mut content = String::new();
            file.read_to_string(&mut content).unwrap();
            inserter = content;
        },
        Err(error) => {panic!("could not insert into postgres")},
    }

    let pool = PgPoolOptions::new()
        .max_connections(3)
        // use your own credentials below
        .connect("postgres://myuser:mypass@localhost/mydb")
        .await
        .expect("couldn't create pool");

	// insert the title and file contents into the database
    let row: (i64,) = sqlx::query_as("insert into myposts (post_title, post_body) values ($1, $2) returning post_id")
        .bind(&args[1])
        .bind(inserter)
        .fetch_one(&pool)
        .await?;

    Ok(())
}
```

We can now use this separate binary to insert our posts into the database using the following command: 

```bash
cargo run --bin markd "My post's title" ./post.md
```

Of course you'd give a different title for each new post.

## Using Askama to render markdown into templates

So far so good. How about we add [Askama](https://github.com/djc/askama) template engine to render our markdown posts into html. 

edit `main.rs`

```rust
use askama::Template;

// Each post template will be populated with the values 
// located in the shared state of the handlers. 

#[derive(Debug)]
#[template(path = "posts.html")]
pub struct PostTemplate<'a> {
    pub post_title: &'a str,
    pub post_date: String,
    pub post_body: &'a str,
}
```

Askama looks for templates outside of the src folder. Create a folder called templates in the same spot that your Cargo.toml resides.

We should also make a base template that our post template can extend from. 

`templates/base.html`

```html
<!DOCTYPE html>
<html lang="en">
	<head>
		<title>{{ post_title }}</title>
		<!-- we'll use Tower middlewar middleware to serve this static content soon-->
		<link href="/assets/post.css rel="stylesheet" type="text/css">
	</head>

	<body>
		<div id="Post">
		{% block post %}
		{% endblock post %}
		</div>
	</body>
</html>
```


`templates/posts.html`

```html
{% extends "base.html" %}

{% block post %}
	<div class="post_title">
		{{ post_title }}
	</div>
	<div class="post_date">
		{{ post_date }}
	</div>
	<div class="post_body">
		{{ post_body|markdown }}
	</div>
{% endblock post %}
```

We need a handler to serve our static CSS. Fortunately, [Tower](https://docs.rs/tower/latest/tower/) has middleware we can use including [tower_http](https://docs.rs/tower-http/latest/tower_http/) to take care of this.

First create a folder titled `assets` in the same spot that main.rs resides. Inside of assets create `post.css` with some CSS.

`assets/post.css`

```css
body {
	background: #101010;
}
#Post {
	background: #D5D9E7;
}
```


edit `main.rs`

```rust
use tower_http::services::ServeDir;

// edit the router to serve static content from the assets folder

let app = Router::new()
        .route("/", get(index))
        .route("/post/:query_title", get(post))
        .with_state(shared_state)
        .nest_service("/assets", ServeDir::new("assets"));
        
```

We now need some logic in the post handler to match the user's query to any post with the same title.

edit `main.rs`

```rust
// We use two extractors in the arguments
// Path to grab the query and State that has all our posts 

async fn post(Path(query_title): Path<String>, State(state): State<Arc<Vec<Post>>>) -> impl IntoResponse {

    // A default template or else the compiler complains 
    let mut template = PostTemplate{post_title: "none", post_date: "none".to_string(), post_body: "none"};
    
    // We look for any post with the same title as the user's query
    for i in 0..state.len() {
        if query_title == state[i].post_title {
            // We found one so mutate the template variable and
            // populate it with the post that the user requested 
            template = PostTemplate{post_title: &state[i].post_title, 
                       post_date: state[i].post_date.to_string(), 
                       post_body: &state[i].post_body
            };
            break;
        } else {
            continue
        }
    }

    // 404 if no title found matching the user's query 
    if &template.post_title == &"none" {
        return (StatusCode::NOT_FOUND, "404 not found").into_response();
    }

    // render the template into HTML and return it to the user
    match template.render() {
        Ok(html) => Html(html).into_response(),
        Err(_) => (StatusCode::INTERNAL_SERVER_ERROR, "try again later").into_response()
    }
}
```

Ok great, but how will the user ever find our posts? 

How about sending them a list of links to all our posts.

edit `main.rs`

```rust
// create an Axum template for our homepage
// index_title is the html page's title 
// index_links are the titles of the blog posts 

#[derive(Template)]
#[template(path = "index.html")]
pub struct IndexTemplate<'a> {
    pub index_title: String,
    pub index_links: &'a Vec<String>,
}

// Then populate the template with all post titles

async fn index(State(state): State<Arc<Vec<Post>>>) -> impl IntoResponse{

    let s = state.clone();
    let mut plinks: Vec<String> = Vec::new();

    for i in 0 .. s.len() {
        plinks.push(s[i].post_title.clone());
    }

    let template = IndexTemplate{index_title: String::from("My blog"), index_links: &plinks};

    match template.render() {
            Ok(html) => Html(html).into_response(),
         Err(err) => (
                StatusCode::INTERNAL_SERVER_ERROR,
                format!("Failed to render template. Error {}", err),
            ).into_response(),
    }
}
```

Index template will loop through our Vec of titles and render them as anchor links.

`templates/index.html`

```html
<!DOCTYPE html> 
<html>
	<head> 
		<title> {{ index_title }} </title>
	</head>

	<body>
		<div class="links">
			<ul>
			{% for item in index_links %}
				<li><a href="/post/{{ item }}">{{ item }}</a></li>
			{% endfor %}
		   </ul>
		</div>
	</body>
</html>
```

Remember to insert your markdown into the database with this command 

```bash
cargo run --bin markd "Some title" ./post.md
```

Here's the repo of the full code 

And now we run the server

```bash
cargo run --bin blog-rs 
```

We're pretty much done, but I want to demonstrate how to create a custom Askama filter. 

I'll be adding dashes to the titles to make them more URL friendly.  

Because this:

`localhost:4000/post/Some-Title` 

is more readable than this:

`localhost:4000/post/Some&20Title`

However, this will also make each post title have dashes. My simple "rmdashes" filter will remove the dashes to make the titles appear more pleasant in the page. 

Askama searches for custom filters inside of `mod filters {}` 

edit `main.rs`

 ```rust 
mod filters {

    // This filter removes the dashes that I will be adding in main() 
    pub fn rmdashes(title: &str) -> askama::Result<String> {
        Ok(title.replace("-", " ").into())
     }
}

// I replace spaces with dashes so that the title appears
// easier to read in the URL. localhost:4000/post/a-title
async fn main() {
	
	for post in &mut posts {
       post.post_title = post.post_title.replace(" ", "-");
    }
    
    //
 ```
 
Now we use the rmdashes filter in `posts.html` as we don't
want the dashes in the web page. Only in the URL.
 
edit  `templates/posts.html`
 
```template
{% extends "base.html" %}

{% block post %}
	<div class="post_title">
		{{ post_title|rmdashes }}
	</div>
	<div class="post_date">
		{{ post_date }}
	</div>
	<div class="post_body">
		{{ post_body|markdown }}
	</div>
{% endblock post %}
 ```
 
## Optimizing the final binary
 
 Use this command to view file sizes, on linux: `ls -lh blog-rs`

 My binary, inside of `target/debug/blog-rs` , is at 126M.
 
[Here's](https://nnethercote.github.io/perf-book/build-configuration.html) an excellent guide on optimizing your binary.

Building my binary with the --release flag reduces the size to only 13M.

```bash
cargo build --release 
```

An optimized binary now resides in `target/release/blog-rs`

Want a smaller binary size? 

[UPX](https://upx.github.io/) gets my binary down further to 3.9M

```bash
upx /target/release/blog-rs
```

