---
layout: post
title: Rust with Ruby
category: code
---

I've been writing a bit of Rust lately and I was super curious to see if I could
embed Rust within Ruby to optimize some hot spots in a few programs I'd written.
To test this out I implemented a Sudoku solver in Ruby using a simple brute
force approach - very slow and very memory intensive. Nothing fancy or elegant.
But an excellent candidate to see how much faster I could make it go.

## A Simple Example

We'll start by writing a simple Rust function to add two numbers together.

~~~ rust
 #[no_mangle]
 pub extern "C" fn add(a: i32, b: i32) -> i32 {
    a + b
 }
~~~

The `no_mangle` attribute is required here. From the official Rust
documentation:

> When you create a Rust library, it changes the name of the function in the
> compiled output ... This attribute turns that behavior off.

We don't want the compiler changing the name of this function otherwise our
Ruby program won't be able to find it.

The `pub` declaration means that this function is callable from outside this
module. `extern "C"` tells the compiler that it should be callable from C. From
there it is just a normal Rust function called `add` which takes two variables
and returns their sum.

Modify Cargo.toml to specify that we are creating a dynamic library.

~~~
[lib]
name = "add"
crate-type = ["dylib"]
~~~

Go ahead and compile this program and hop over to Ruby. To use this Rust
function within our Ruby code we need to install the [FFI
gem](https://github.com/ffi/ffi){:target="_blank"}.

> Ruby-FFI is a ruby extension for programmatically loading dynamic libraries,
> binding functions within them, and calling those functions from Ruby code.

Within our program we create a module which will serve as the attachment point
for our Rust program. We extend FFI and then give it the path to the compiled
program. Then attach the function `add` from our Rust program to this module
and specify the function signature - parameters first then return value.

~~~ ruby
require "ffi"

module RustAdd
  extend FFI::Library
  ffi_lib "target/release/libadd.dylib"
  attach_function :add, [:int, :int], :int 
end

puts RustAdd.add(1336, 1)
# => 1337
~~~

_Note: if you are using Linux then you will want to use the `.so` extension
instead of the `.dylib`_

## More Advanced Example
Going back to my Sudoku program - it takes a string and returns a string back
out. Passing those through FFI is a little bit more work. Since strings don't
really exist in C (they are really arrays) we can't just pass a Ruby string
straight through to Rust. We have to deal with the pointer to the string. Yikes. 

The `solve` function below takes a variable called `c_ptr` which is a C pointer.
We'll use `std::ffi::CStr` to wrap the pointer since it could be `NULL`, then
convert it to a string slice so we can use it in our Rust program. On the way
out the function uses `CString` to turn the solution back into a C pointer.

~~~ rust
extern crate libc;

use libc::c_char;
use std::ffi::CStr;
use std::ffi::CString;

#[no_mangle]
pub extern "C" fn solve(c_ptr: *const c_char) -> *const c_char {
    let c_str = unsafe {
        assert!(!c_ptr.is_null());

        CStr::from_ptr(c_ptr)
    };

    let ruby_str = str::from_utf8(c_str.to_bytes()).unwrap();

    let mut sudoku = Sudoku::from_str(ruby_str).unwrap();
    sudoku.brute_force_solve();

    let solution = CString::new(format!("{}", sudoku)).unwrap();
    solution.into_raw()
}
~~~

On the Ruby side:

~~~ ruby
require "ffi"

class Sudoku
  def initialize(puzzle)
    @puzzle = puzzle
  end

  module RustSudoku
    extend FFI::Library
    ffi_lib "target/release/libsudoku.dylib"
    attach_function :solve, [:string], :string
  end


  def solve
    RustSudoku.solve(@puzzle)
  end
end
~~~

Now my brute force solver runs in about a hundredth of a second. 

~~~
rspec sudoku_spec.rb
.

Finished in 0.0118 seconds
1 example, 0 failures
~~~


## More Reading
* [http://blog.skylight.io/bending-the-curve-writing-safe-fast-native-gems-with-rust/](http://blog.skylight.io/bending-the-curve-writing-safe-fast-native-gems-with-rust/){:target="_blank"}
* [https://doc.rust-lang.org/book/rust-inside-other-languages.html](https://doc.rust-lang.org/book/rust-inside-other-languages.html){:target="_blank"}
* [https://github.com/steveklabnik/rust_example](https://github.com/steveklabnik/rust_example){:target="_blank"}
* [https://www.youtube.com/watch?v=IqrwPVtSHZI](https://www.youtube.com/watch?v=IqrwPVtSHZI){:target="_blank"}
