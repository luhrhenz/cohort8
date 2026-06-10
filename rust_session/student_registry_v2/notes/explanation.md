# Student Registry V2 — Code Explanation

## What is this project?

This is a simple in-memory student registry built in Rust.
"In-memory" means the data only lives while the program is running — nothing is saved to a file or database.

The big difference from v1 is that instead of giving each student a simple number as their ID (`1`, `2`, `3`...),
we use a **UUID** — a randomly generated unique string like `f3dbcfa8-a037-4a8e-82d8-21b4b6235c2d`.

---

## Project Structure

```
student_registry_v2/
├── Cargo.toml              ← project config + dependencies
└── src/
    ├── main.rs             ← entry point, where execution starts
    ├── grade.rs            ← Grade and Sex enums
    ├── student_struct.rs   ← the Student data type
    └── registry.rs         ← the Registry that manages students
```

---

## File by File

---

### `Cargo.toml`

```toml
[dependencies]
uuid = { version = "1", features = ["v4"] }
```

This tells Cargo (Rust's package manager) to download and include the `uuid` library.
The `features = ["v4"]` part means we want version 4 UUIDs — the kind that are **randomly generated**.

---

### `grade.rs` — Enums

```rust
pub enum Grade {
    First,
    Second,
    Third,
}
```

An **enum** is a type that can only be one of a fixed set of values.
Think of it like a multiple-choice answer — a student's grade can only be First, Second, or Third, nothing else.

```rust
impl Grade {
    pub fn as_str(&self) -> &str {
        match self {
            Grade::First => "Cohort 1",
            Grade::Second => "Cohort 2",
            Grade::Third => "Cohort 3",
        }
    }
}
```

`impl Grade` adds methods to the enum — just like adding functions that belong to it.
`as_str()` takes the current value (`&self`) and uses `match` to return its text label.
`match` is Rust's version of a switch statement, but it must cover every possible case.

`&str` is a borrowed string slice — a reference to text. It doesn't own the string, just points to it.

```rust
pub enum Sex {
    Male,
    Female,
}
```

Same idea — Sex can only be Male or Female.

```rust
impl Sex {
    pub fn to_str(&self) -> &str {
        match self {
            Sex::Male => "male",
            Sex::Female => "female",
        }
    }
}
```

---

### `student_struct.rs` — The Student

```rust
#[derive(Debug)]
pub struct Student {
    pub id: String,
    pub name: String,
    pub age: u8,
    pub sex: Sex,
    pub grade: Grade,
    pub score: f32,
}
```

A **struct** is a custom data type that groups related fields together.
Think of it like a form — every student has the same fields, but different values.

| Field   | Type     | Meaning                                      |
|---------|----------|----------------------------------------------|
| `id`    | `String` | UUID — unique identifier (random string)     |
| `name`  | `String` | Heap-allocated text, owned by this struct    |
| `age`   | `u8`     | Unsigned 8-bit integer (0–255), enough for age |
| `sex`   | `Sex`    | Our own enum from grade.rs                   |
| `grade` | `Grade`  | Our own enum from grade.rs                   |
| `score` | `f32`    | 32-bit floating point number (e.g. 78.5)     |

`#[derive(Debug)]` automatically gives the struct the ability to be printed with `{:?}` or `{:#?}`.

`pub` on each field means code outside this file can read and write those fields directly.

```rust
impl Student {
    pub fn new(id: String, name: String, age: u8, sex: Sex, grade: Grade, score: f32) -> Student {
        Student { id, name, age, sex, grade, score }
    }
}
```

`new()` is a constructor — a function that builds and returns a `Student`.
Rust has no built-in constructor keyword, so by convention we write a `new()` function.
`-> Student` means this function returns a `Student` value.

---

### `registry.rs` — The Registry (the core logic)

```rust
pub struct Registry {
    pub students: Vec<Student>,
}
```

`Vec<Student>` is a growable list of students — like an array that can expand.
In v1 there was also a `next_id: u32` counter here. In v2 it's gone — UUID handles its own uniqueness.

---

#### `new()`
```rust
pub fn new() -> Registry {
    Registry {
        students: Vec::new(),
    }
}
```
Creates an empty registry with an empty list. `Vec::new()` gives you an empty vector.

---

#### `add()`
```rust
pub fn add(&mut self, name: &str, age: u8, sex: Sex, grade: Grade, score: f32) {
    let id = Uuid::new_v4().to_string();
    let student = Student::new(id, name.to_string(), age, sex, grade, score);
    println!("Added: {} (ID: {})", student.name, student.id);
    self.students.push(student);
}
```

- `&mut self` — the registry is borrowed mutably, meaning we're allowed to change it
- `Uuid::new_v4()` — generates a random UUID (128-bit number)
- `.to_string()` — converts it into a regular `String`
- `name.to_string()` — `name` comes in as `&str` (a reference), we convert it to an owned `String` so the student can own its name
- `.push(student)` — appends the student to the end of the vector

---

#### `list_all()`
```rust
pub fn list_all(&self) {
    if self.students.is_empty() { ... }
    for student in &self.students { ... }
}
```

- `&self` — borrowed immutably, we're only reading, not changing
- `&self.students` — we iterate over references to the students (we don't consume or move them)
- `for student in &self.students` — `student` here is `&Student`, a reference to each one

---

#### `find_by_id()`
```rust
pub fn find_by_id(&self, id: &str) -> Option<&Student> {
    self.students.iter().find(|s| s.id == id)
}
```

- `.iter()` — creates an iterator over references to the students
- `.find(|s| s.id == id)` — goes through each student and returns the first one where `s.id == id`
- `Option<&Student>` — the return type. In Rust there is no `null`. Instead:
  - `Some(&student)` is returned if a match is found
  - `None` is returned if nothing matches
- This forces you to handle both cases, preventing null pointer crashes

---

#### `update()`
```rust
pub fn update(&mut self, id: &str, name: &str, age: u8, score: f32) {
    if let Some(student) = self.students.iter_mut().find(|s| s.id == id) {
        student.name = name.to_string();
        student.age = age;
        student.score = score;
        println!("Updated student with ID: {}", id);
    } else {
        println!("Student with ID {} not found", id);
    }
}
```

- `.iter_mut()` — like `.iter()` but gives **mutable** references, so we can modify fields
- `if let Some(student)` — this is Rust's clean way of saying "if the Option has a value, unwrap it into `student` and run this block, otherwise go to `else`"
- We then directly reassign the fields on the found student

---

#### `delete()`
```rust
pub fn delete(&mut self, id: &str) {
    let before = self.students.len();
    self.students.retain(|s| s.id != id);
    if self.students.len() < before {
        println!("Deleted student with ID: {}", id);
    } else {
        println!("Student with ID {} not found", id);
    }
}
```

- `.retain(|s| s.id != id)` — keeps only the students where the condition is `true`
  - Students whose id does NOT match the given id are kept
  - The one whose id matches is silently dropped
- We compare `len()` before and after to know if anything was actually removed

---

### `main.rs` — Putting it all together

```rust
mod grade;
mod registry;
mod student_struct;
```

`mod` tells Rust to include these files as modules. Without this, the files are invisible to the compiler.

```rust
use grade::{Grade, Sex};
use registry::Registry;
```

`use` brings names into scope so you don't have to write `grade::Grade` every time.

```rust
let mut reg = Registry::new();
```

`let` declares a variable. `mut` means it's mutable (changeable). Without `mut`, Rust won't let you call methods that modify it.

```rust
reg.add("Victor", 20, Sex::Male, Grade::First, 78.5);
```

Adds Victor to the registry. Inside `add()`, a UUID is generated automatically for him.

```rust
let first_id = reg.students[0].id.clone();
```

Gets the id of the first student. `.clone()` is needed because `id` is a `String` — if we just wrote `reg.students[0].id`, we'd be moving it out of the struct (Rust won't allow that while the struct is still alive). `.clone()` makes a copy instead.

```rust
match reg.find_by_id(&first_id) {
    Some(s) => println!("Found: {:#?}", s),
    None => println!("Not found"),
}
```

`match` on an `Option` is the standard Rust pattern. We handle both outcomes:
- `Some(s)` — found, print the student with pretty debug formatting (`{:#?}`)
- `None` — not found, print a message

```rust
reg.update(&first_id, "Victor Updated", 21, 85.0);
```

Updates Victor's name, age, and score. The id stays the same — we're just changing the other fields.

```rust
reg.delete(&first_id);
```

Removes Victor from the registry entirely.

---

## v1 vs v2 — The key differences

| | v1 | v2 |
|---|---|---|
| Student ID type | `u32` (number) | `String` (UUID) |
| ID generation | `next_id` counter in Registry | `Uuid::new_v4()` |
| Predictable IDs | yes (0, 1, 2...) | no (random each run) |
| Works across systems | no | yes |
| find_by_id | not implemented | `Option<&Student>` |
| update | not implemented | `.iter_mut()` + field assignment |
| delete | not implemented | `.retain()` |
| External dependency | none | `uuid` crate |

---

## Key Rust Concepts Used

| Concept | Where | What it means |
|---|---|---|
| `struct` | `Student`, `Registry` | Custom data type grouping fields |
| `enum` | `Grade`, `Sex` | A type with a fixed set of variants |
| `impl` | all structs/enums | Adds methods to a type |
| `Vec<T>` | `students` field | A growable list |
| `Option<T>` | `find_by_id` return | Either `Some(value)` or `None` — no nulls |
| `&self` | read-only methods | Borrow the value without taking ownership |
| `&mut self` | mutating methods | Borrow the value and allow changes |
| `.iter()` | `list_all`, `find_by_id` | Loop without consuming the collection |
| `.iter_mut()` | `update` | Loop with ability to modify elements |
| `.retain()` | `delete` | Keep only elements matching a condition |
| `.clone()` | `main.rs` | Make a deep copy of an owned value |
| `if let Some(x)` | `update` | Unwrap an Option only if it has a value |
| `match` | everywhere | Pattern match — must handle all cases |
