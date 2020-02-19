---
layout: single
title: "Rust: Deserialize JSON with Serde"
header:
  teaser: /assets/images/rebuild.jpg
  imgcredit: Photo by Randy Fath on Unsplash
categories:
  - rust
  - serde
  - json
  - howto
---

JSON can be a complicated format to manipulate, even though it's well structured text. When using static type languages - like Rust - we want the almighty compiler on our side, no one wants to work with pure text... but it happens :)

The following JSON has a list of profiles:

```json
[
  {
    "id": 1,
    "type": "personal",
    "details": {
      "firstName": "Juliano",
      "lastName": "Alves",
      "primaryAddress": 7777777
    }
  },
  {
    "id": 2,
    "type": "business",
    "details": {
      "name": "Juliano Business",
      "companyRole": "OWNER",
      "primaryAddress": 8888888
    }
  }
]
```

They are not the same though. We have to handle two problems:

1. The field names should follow `snake_case` instead of `camelCase`;
2. The response has different fields

Let's see how we can parse this json into an strongly typed data structure using [Serde](https://serde.rs).

## The Data Structure

Both profiles have `id` and `type` fields, so instead of the similarities we should think about the differences between them first. Let's define `PersonalDetails` and `BusinessDetails`:

```rust
struct PersonalDetails {
    first_name: String,
    last_name: String,
    primary_address: i32
}

struct BusinessDetails {
    name: String,
    company_role: String,
    primary_address: i32
}
```

Now we have to make sure that serde will know how to use these structs.

### The derive macro

Based on Rust's `#[derive]` mechanism, [serde provides a handful macro](https://serde.rs/derive.html) that can be used to generate implementations of [`Serialize`](https://docs.serde.rs/serde/trait.Serialize.html) and [`Deserialize`](https://docs.serde.rs/serde/trait.Deserialize.html). We just need to deserialize, so let's add it. Once we are changing the code, we can derive [`Debug`](https://doc.rust-lang.org/std/fmt/trait.Debug.html) as well, to make it easier to print later:

```rust
#[derive(Deserialize, Debug)]
struct PersonalDetails {

#[derive(Deserialize, Debug)]
struct BusinessDetails {
```

### Renaming fields

In order to customize fields, serde provides [container attributes](https://serde.rs/container-attrs.html). To rename `camelCase` we can use [`#[serde(rename_all = "...")]`](https://serde.rs/container-attrs.html#rename_all):

```rust
#[derive(Deserialize, Debug)]
#[serde(rename_all = "camelCase")]
struct PersonalDetails {

#[derive(Deserialize, Debug)]
#[serde(rename_all = "camelCase")]
struct BusinessDetails {
```

### Parsing different objects

The distinction between the profiles is given by the `type` attribute. When the field identifying which variant we are dealing with is inside the content, serde calls it [internally tagged](https://serde.rs/enum-representations.html#internally-tagged).

Let's define an `enum Profile`, with two profiles `Personal` and `Business`. Using `tag` we will tell serde to use the `type` field in order to decide between the variants:

```rust
#[derive(Deserialize, Debug)]
#[serde(tag = "type", rename_all = "camelCase")]
enum Profile {
    Personal {
        id: i32,
        details: PersonalDetails,
    },
    Business {
        id: i32,
        details: BusinessDetails,
    },
}
```

Now we can parse the json with `serde_json`:

```rust
let profiles: Vec<Profile> = serde_json::from_str(data)?;
```

## Full example

`Cargo.toml`

```rust
[package]
name = "serde-example"
version = "0.1.0"
authors = ["Juliano Alves <von.juliano@gmail.com>"]
edition = "2018"

[dependencies]
serde = "1.0"
serde_derive = "1.0"
serde_json = "1.0"
```

`main.rs`

```rust
#[macro_use]
extern crate serde_derive;

use serde_json::Result;

fn main() -> Result<()>  {
    let data = r#"
    [
      {
        "id": 1,
        "type": "personal",
        "details": {
          "firstName": "Juliano",
          "lastName": "Alves",
          "primaryAddress": 7777777
        }
      },
      {
        "id": 2,
        "type": "business",
        "details": {
          "name": "Juliano Business",
          "companyRole": "OWNER",
          "primaryAddress": 8888888
        }
      }
    ]
    "#;

    let profiles: Vec<Profile> = serde_json::from_str(data)?;
    println!("{:#?}", profiles);

    Ok(())
}

#[derive(Deserialize, Debug)]
#[serde(rename_all = "camelCase")]
struct PersonalDetails {
    first_name: String,
    last_name: String,
    primary_address: i32
}

#[derive(Deserialize, Debug)]
#[serde(rename_all = "camelCase")]
struct BusinessDetails {
    name: String,
    company_role: String,
    primary_address: i32
}

#[derive(Deserialize, Debug)]
#[serde(tag = "type", rename_all = "camelCase")]
enum Profile {
    Personal {
        id: i32,
        details: PersonalDetails,
    },
    Business {
        id: i32,
        details: BusinessDetails,
    },
}
```
