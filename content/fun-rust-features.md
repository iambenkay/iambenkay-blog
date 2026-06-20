+++
title = "Rust features that make programming fun again"
slug = "rust-features-that-make-programming-fun-again"
date = 2025-12-11
description = "The Rust programming language is a systems programming language that is fast, safe, and concurrent. It is a modern language that is designed to be a memory safe alternative to C and C++ but that is not all it brings to the table. There are a"

[extra]
header_img = "/images/header-rust.jpg"
header_alt = "Rust"
section = "Programming"
word_count = 1440
author = "Benjamin Chibuzor-Orie"
tags = ["Rust", "Exciting", "Syntax"]
+++
The **Rust programming language** is a systems programming language that is **fast**, **safe**, and **concurrent**. It is a modern language that is designed to be a memory safe alternative to **C** and **C++** but that is not all it brings to the table. There are a number of features that make Rust a fun language to work with and I am going to be discussing some of them that I find particularly interesting in this article so get ready to try out some of these samples on your own and see how enjoyable they are to use.

## Are we a match?
I'll start with one of **Rust's** most loved features. It is loved so much that the practice is now being adapted across many existing languages like Java and Kotlin because of it's expressiveness however none does it quite as well and beautifully as Rust. For our first example, we will use a Rust enum. I am going to assume you know what an enum in Rust is and how it works. It is a simple way to represent related complex types in one structure and the language provides facilities to maximise the use of this while writing programs. Consider the snippet below:
```rust
enum JsonValue {
    Number(f64),
    String(String),
    Boolean(bool),
    Array(Vec<JsonValue>),
    Object(HashMap<String, JsonValue>),
    Null,
}
```
With the above struct I can represent an entire JSON tree using this enum. Obviously you already knew this and this is not the reason we are here. The real reason is **pattern matching**. Now watch what I can do with pattern matching:
```rust
match json_value {
    JsonValue::Number(n) => println!("Number: {}", n),
    JsonValue::String(s) => println!("String: {}", s),
    JsonValue::Boolean(b) => println!("Boolean: {}", b),
    JsonValue::Array(a) => println!("Array: {:?}", a),
    JsonValue::Object(o) => println!("Object: {:?}", o),
    JsonValue::Null => println!("Null"),
}
```
It's like a switch statement but much more powerful because you can add extra conditions to the match and you can also match deeper more complex patterns in the same simple way like below:
```rust
let user: Result<Option<User>, Error> = mongo.users.find_one({ "username": "admin" }).await;

match user {
    Ok(Some(user)) if !user.is_deleted => println!("User: {:?}", user),
    Ok(None) => println!("User not found"),
    Err(e) => println!("Error: {:?}", e),
    _ => println!("Unknown error"),
}
```
The above could easily take up more lines than needed in a conventional programming language, but not my Rust! Let's move on to the next thing I love about the language
## Unwrap hell?
As part of a mission to make illegal states unrepresentable (a mission that was heralded by Scala for a long time before Rust toppled it and owned that domain), Rust introduced the concept of `Result` and `Option` types. These types are used to represent the possibility of a value being present or absent and the possibility of an error occurring. This allows you to write code that is more expressive and less error-prone. Consider the snippet below:
```rust
let user: Result<Option<User>, Error> = mongo.users.find_one({ "username": "admin" }).await;
```
Assuming you already know the basics of these types and how they operate, I will skim past them to the main point of this paragraph. If the database call fails for some reason then Rust does not throw instead it returns the `Result::Err` variant. Now let's say the call to the database succeeds then we will have a `Result::Ok` variant which in turn contains an `Option<User>` variant. If the user is found then we will have the `Some(User)` variant and if the user is not found then we will have the `None` variant. Now that you are up to speed let's talk about the problem.
A code smell I have seen too many times in Rust code is the use of `unwrap` to extract result/option values. Consider the snippet below:
```rust
async fn update_user_catalogues(user_id: &str) {
    let user: Result<Option<User>, Error> = mongo.users.find_one({ "id": user_id }).await;
    
    if user.is_ok() {
        let user = user.unwrap();

        if user.is_some() {
            let user = user.unwrap();

            // do something with the user
        }
    }
}
```
Even though the above code is safe to run and will not generate any panics because of the checks happening before the unwrap calls, I still think it is too verbose, dirty and prone to errors as the code evolves and stuff gets moved around. The ideal way that I found in Rust to do this is the `if/let` construct which combines well with pattern matching. It is really beautiful to watch. Let us rewrite the above function using the `if/let` construct:
```rust
async fn update_user_catalogues(user_id: &str) {
    let user: Result<Option<User>, Error> = mongo.users.find_one({ "id": user_id }).await;
    
    if let Ok(Some(user)) = user {
        // do something with the user
    }
```
Isn't that just a lot more beautiful and direct? No need to unwrap so many things especially since it's not even Boxing Day yet.
## Errors are values?
Unlike in conventional languages where errors are represented by exceptions and interrupts, in Rust errors are just values. This means that error handling can be executed in a very precise functional way. However, this manual propagation of errors may introduce a lot of boilerplate code by design. The maintainers recognized this hassle and came up with an error propagator, the `?` operator. This makes the propagation process semi-automatic as long as the error types match the one defined in the return signature of the containing function. Consider the following snippet:
```rust
async fn update_user_catalogues(user_id: &str) -> Result<(), Error> {
    let user: Option<User> = mongo.users.find_one({ "id": user_id }).await?;
    
    if let Some(user) = user {
        // do something with the user
    }
```
Thanks to the `?` operator, we have a circuit breaker which returns any `Result:Err` values it gets but if it is a `Result:Ok` value then we can continue with the execution of the function. This is a very powerful feature of Rust and it allows you to write code that is more expressive and people adore it so much.
## And then what?
When you have a chain of nested `Option` or `Result` types, either directly or as a result of extra function calls, that need to be unwrapped optionally, things will get really verbose fast if you don't utilize this golden bullet I am about to show you. Before I show it to you, consider the snippet below:
```rust
let authorization_header: Option<HeaderValue> = headers.get("Authorization");

if let Some(authorization_header) = authorization_header {
    if let Ok(header_value) = authorization_header.to_str() {
        if let Some(token) = header_value.strip_prefix("Bearer ") {
            if !token.is_empty() && *token != "null" {
                // do something with the token
            }
        }
    }
}
```
I am shedding righteous tears of war just staring at those nested `if` blocks and I hope you are too. This is no way to live and it is hardly a creative attempt at making the code readable. The more rusty way to do this would be to use the `Option::and_then` or the `Result::and_then` methods. It allows you to chain multiple `Option` and `Result` together and it also supports the error propagator `?`. Consider the snippet below:
```rust
let mut token: Option<String> = headers.get("Authorization").and_then(|h| {
        h.to_str()
            .ok()?
            .strip_prefix("Bearer ")
            .take_if(|s| !s.is_empty() && *s != "null")
    });
```
It really speaks for itself but for the sake of those who don't quite grasp it, the main thing happening here is the `and_then` call provides a scope that allows us to idiomatically `unwrap` the values safely and neatly while operating on them.
## Unwrap or?
This one is really nice and it's one of those features that make you wonder how you ever survived without them. Before I knew about the `unwrap_or` method, I used to handle configuring fallbacks like this:
```rust
let wallet = user.get_wallet();

if wallet.is_none() {
    wallet = Some(new_wallet());
}

let wallet = wallet.unwrap();
```
This is obviously horrendous and now I know better to use the `unwrap_or` method for this:
```rust
let wallet = user.get_wallet().unwrap_or(new_wallet());
```
That's more succinct, clean and comprehensible. It also comes with a lazy variant if that's the kind of juice you are looking for:
```rust
let wallet = user.get_wallet().unwrap_or_else(|| new_wallet());
```
## Serialization and Deserialization, why not?
Rust has a very powerful serialization and deserialization library called `serde`. It allows you to easily convert your structs to and from JSON, YAML, TOML and other formats. It also supports the `?` operator for error propagation. You can map various types of complex data sets to Rust types just like I have shown at the beginning of this post however that example was not type-safe. How about we explore a type-safe version where we know the shape of the data we are expecting:
```rust
#[serde(untagged)]
pub enum Input {
    Text(String),
    List(Vec<String>),
}
```
Since we marked our enum as `#[serde(untagged)]`, we can deserialize either a JSON string or a JSON array into the Rust type. It's really a great way to keep your type system in check as the code evolves. Below is another snippet that involves tagged enums:
```rust
#[serde(tag = "type")]
pub enum InnerItem {
    #[serde(rename = "input_text")]
    Input { text: String },

    #[serde(rename = "output_text")]
    Output { text: String },

    #[serde(rename = "input_image")]
    Image {
        image_url: String,
        #[serde(default)]
        detail: Detail,
    },
}
```
In this version, the serialization relies on the `#[serde(tag = "type")]` attribute. Whenever a `type` field is missing in the input then it will automatically fail serialization. It is used in conjunction with serde `rename` to locate the correct variant to store the data.

## Conclusion
In this post, we explored some fun Rust features that keep me coming back and I am sure you would love to try out. I will make a longer list for this in the future depending on how you respond to this one. Let me have your questions in the comments below.
