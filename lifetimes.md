## A C++ comparison <sup id="a1">[1](#f1)</sup>


You already know that code like this

```C++
char* first_letter(std::string& s) {
    return &s[0];
}
...
char* name;
{
    string name_string = "hello";
    name = first_letter(name_string);
} // name_string goes out of scope here
cout << name[0];
```

is invalid, right? Rust adds extra annotations to let the compiler stop you from
doing that sort of thing.

Think of the difference between the following two functions:

```C++
const char& first_letter(std::string& s) {
    return s[0];
}
```

and

```C++
const char& first_letter(std::string& s) {
    static const char* letters = "abcdefghijklmnopqrstuvwxyz";
    return letters[s[0] - 'a']; // Pretend I know it starts with a lowercase letter
}
```

They have identical signatures, but very different behavior with respect to when
it's safe to call them. With the first one, you need to make sure you never
dereference the returned pointer after reallocating or freeing the string you
gave it. With the second one, you can do whatever you want with it, since it's
pointing to static memory instead. In Rust, these two functions have different
signatures.

```rust
fn first_letter<'a>(s: &'a String) -> &'a char
```

vs

```rust
fn first_letter<'a>(s: &'a String) -> &'static char
```

The `'a` is the name of a lifetime, which is some span of code between a
constructor and a destructor. The first says "If you give me
a reference to a String that is valid for a certain period, I'll give you a reference
to a character that is valid for that same period". The second says "I'll give
you a reference to a character that is valid for the entire length of the
program". If the compiler sees that a caller is violating the contract of the
first function signature, that the returned reference is forgotten before the
string given to it is destroyed, it will prevent the program from compiling.


##Example 2 <sup id="a2">[1](#f2)</sup>

```rust
fn main() { 
    let x = 1;
    let mut y = 2;
    let z = borrow(&x, &y);
    &mut y;
    z;
} // z goes out of scope here

fn borrow<'a, 'b>(u : &'a i32, v : &'a i32) -> &'a i32 { 
  // u and v are immutable borrows, which end when the variables u, v go out of scope
  &u 
} // u and v go out of scope here
  // the borrow of v ends, the ownership of u's borrow is transferred to the return value's binding, so
  // while u goes out of scope, the borrow lives on with another owner
```

Note that `main` is perfectly valid, because the reference given to `borrow` ends after the function is run, so a mutable
reference to `y` can exist after that. I does not compile, though, because the lifetimes in the function signature of
`borrow` claim "The lifetimes of the borrows for `u` and `v` are the same as the lifetime of the return value of that function",
and the return value's ownership is transferred to `z`. So the signature claims to the compiler that the borrow `&y` in the
call to `borrow` lives until `main` finishes, which conflicts with the mutable binding `&mut y;`.

In conclusion, the code is perfectly valid, the compiler just can't see this fact, because we made wrong claims about
lifetimes here.

We can fix this by changing `v: &'a i32` to `v: &'b i32`, that is to 

```rust
fn main() { 
    let x = 1;
    let mut y = 2;
    let z = borrow(&x, &y);
    &mut y;
    z;
} 

fn borrow<'a, 'b>(u : &'a i32, v : &'b i32) -> &'a i32 { 
  &u 
} 
```

The lifetime claims now are "The first argument's borrow lives as long as the return value, the second one has a lifetime
that is independent from that". Now the compiler can verify that the code is correct, and happily compiles for us.

## References
<b id="f1">1</b> https://www.reddit.com/r/rust/comments/587ecn/looking_to_possibly_learn_rust_i_come_from_a/d8y9gxl [↩](#a1)

<b id="f2">2</b> https://github.com/rust-lang/rust/pull/36997#issuecomment-254779796 [↩](#a2)
