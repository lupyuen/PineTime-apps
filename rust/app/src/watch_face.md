# Porting PineTime Watch Face from C to Rust On RIOT with LVGL

![Rust on RIOT on PineTime Smart Watch](https://lupyuen.github.io/images/rust-on-riot-small.jpg)

_This article is presented in CINEMASCOPE... Rotate your phone to view the C and Rust source code side by side... Or better yet, read this article on a desktop computer_

We'll learn step by step to convert this [Embedded C code](https://github.com/bosmoment/PineTime-apps/blob/master/widgets/home_time/screen_time.c) (based on LVGL) to [Embedded Rust](https://github.com/lupyuen/pinetime-rust-riot/blob/master/rust/app/src/watch_face.rs) on RIOT...

| __Original C Code__ | __Converted Rust Code__ |
| :--- | :--- |
| `lv_obj_t *screen_time_create(home_time_widget_t *ht) {` <br><br>&nbsp;&nbsp;`    //  Create a label for time (00:00)` <br>&nbsp;&nbsp;`    lv_obj_t *scr = lv_obj_create(NULL, NULL);` <br>&nbsp;&nbsp;`    lv_obj_t *label1 = lv_label_create(scr, NULL);` <br><br>&nbsp;&nbsp;`    lv_label_set_text(label1, "00:00");` <br>&nbsp;&nbsp;`    lv_obj_set_width(label1, 240);` <br>&nbsp;&nbsp;`    lv_obj_set_height(label1, 200);` <br>&nbsp;&nbsp;`    ht->lv_time = label1;` <br>&nbsp;&nbsp;`    ...` <br>&nbsp;&nbsp;`    return scr;` <br>`}` <br> | `fn create_widgets(widgets: &mut WatchFaceWidgets) -> ` <br>&nbsp;&nbsp;`    LvglResult<()> {` <br><br>&nbsp;&nbsp;`    //  Create a label for time (00:00)` <br>&nbsp;&nbsp;`    let scr = widgets.screen;` <br>&nbsp;&nbsp;`    let label1 = label::create(scr, ptr::null()) ? ;` <br><br>&nbsp;&nbsp;`    label::set_text(label1, strn!("00:00")) ? ;` <br>&nbsp;&nbsp;`    obj::set_width(label1, 240) ? ;` <br>&nbsp;&nbsp;`    obj::set_height(label1, 200) ? ;` <br>&nbsp;&nbsp;`    widgets.time_label = label1;` <br>&nbsp;&nbsp;`    ...` <br>&nbsp;&nbsp;`    Ok(())` <br>`}` <br> |
| _From [widgets/home_time/screen_time.c](https://github.com/bosmoment/PineTime-apps/blob/master/widgets/home_time/screen_time.c)_ | _From [rust/app/src/watch_face.rs](https://github.com/lupyuen/pinetime-rust-riot/blob/master/rust/app/src/watch_face.rs)_ |

We'll also learn how Rust handles memory safety when calling C functions...

| __Original C Code__ | __Converted Rust Code__ |
| :--- | :--- |
|
`int set_time_label(home_time_widget_t *ht) {` <br><br>&nbsp;&nbsp;`    //  Create a string buffer on stack` <br>&nbsp;&nbsp;`    char time[6];` <br><br>&nbsp;&nbsp;`    //  Format the time` <br>&nbsp;&nbsp;`    int res = snprintf(time, ` <br>&nbsp;&nbsp;&nbsp;&nbsp;`        sizeof(time), ` <br>&nbsp;&nbsp;&nbsp;&nbsp;`        "%02u:%02u", ` <br>&nbsp;&nbsp;&nbsp;&nbsp;`        ht->time.hour,` <br>&nbsp;&nbsp;&nbsp;&nbsp;`        ht->time.minute);` <br><br>&nbsp;&nbsp;`if (res != sizeof(time) - 1) {` <br>&nbsp;&nbsp;&nbsp;&nbsp;`LOG_ERROR("overflow");` <br>&nbsp;&nbsp;&nbsp;&nbsp;`return -1;` <br>&nbsp;&nbsp;`}` <br><br>&nbsp;&nbsp;`//  Set the label` <br>&nbsp;&nbsp;`lv_label_set_text(ht->lv_time, time);` <br><br>&nbsp;&nbsp;`//  Return OK` <br>&nbsp;&nbsp;`return 0;` <br>`}` <br>|`fn set_time_label(` <br>&nbsp;&nbsp;`    widgets: &WatchFaceWidgets, ` <br>&nbsp;&nbsp;`    state: &WatchFaceState) -> ` <br>&nbsp;&nbsp;`    LvglResult<()> {` <br><br>&nbsp;&nbsp;`    //  Create a static string buffer` <br>&nbsp;&nbsp;`    static mut TIME_BUF: String =`<br>&nbsp;&nbsp;&nbsp;&nbsp;` new_string();` <br><br>&nbsp;&nbsp;`    unsafe {` <br>&nbsp;&nbsp;&nbsp;&nbsp;`        //  Format the time` <br>&nbsp;&nbsp;&nbsp;&nbsp;`        TIME_BUF.clear();` <br>&nbsp;&nbsp;&nbsp;&nbsp;`        write!(&mut TIME_BUF, ` <br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`            "{:02}:{:02}\0",` <br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`            state.time.hour,` <br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`            state.time.minute)` <br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`            .expect("overflow");` <br><br>&nbsp;&nbsp;&nbsp;&nbsp;`        //  Set the label` <br>&nbsp;&nbsp;&nbsp;&nbsp;`        label::set_text(widgets.time_label, ` <br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`            &to_strn(&TIME_BUF) ? ;` <br>&nbsp;&nbsp;`    }` <br><br>&nbsp;&nbsp;`    //  Return OK` <br>&nbsp;&nbsp;`    Ok(())` <br>`}` <br>
|
| _From [widgets/home_time/screen_time.c](https://github.com/bosmoment/PineTime-apps/blob/master/widgets/home_time/screen_time.c)_ | _From [rust/app/src/watch_face.rs](https://github.com/lupyuen/pinetime-rust-riot/blob/master/rust/app/src/watch_face.rs)_ |

# Function Declaration

Here's a C function that calls the [LVGL](https://lvgl.io/) library to create a Label Widget.  The Label Widget displays the time of the day (like `23:59`).  This code was taken from the [bosmoment /
PineTime-apps](https://github.com/bosmoment/PineTime-apps/blob/master/widgets/home_time/screen_time.c) port of [RIOT](https://www.riot-os.org/) to the [PineTime Smart Watch](https://wiki.pine64.org/index.php/PineTime).

```c
lv_obj_t *screen_time_create(home_time_widget_t *ht) {
    //  Create a label for time (00:00)
    lv_obj_t *scr = lv_obj_create(NULL, NULL);
    lv_obj_t *label1 = lv_label_create(scr, NULL);

    lv_label_set_text(label1, "00:00");
    lv_obj_set_width(label1, 240);
    lv_obj_set_height(label1, 200);
    ht->lv_time = label1;
    return scr;
}
```
_From [widgets/home_time/screen_time.c](https://github.com/bosmoment/PineTime-apps/blob/master/widgets/home_time/screen_time.c)_

Functions whose names start with `lv_` (like `lv_obj_create`) are defined in the LVGL library. `lv_obj_t` is a C Struct exposed by the LVGL library. `home_time_widget_t` is a custom C Struct defined by the RIOT application.

Let's start by converting this function declaration from C to Rust...

```c
lv_obj_t *screen_time_create(home_time_widget_t *ht) { ...
```

This function accepts a pointer and returns another pointer. In Rust, functions are defined with the `fn` keyword...

```rust
fn screen_time_create( ...
```

The return type `lv_obj_t` goes to the end of the function declaration, marked by `->`...

```rust
fn screen_time_create(ht: *mut home_time_widget_t) 
    -> *mut lv_obj_t { ...
```

Note that the names and types have been flipped, also for pointers...

| __Original C Code__ | __Converted Rust Code__ |
| :--- | :--- |
| `lv_obj_t *` | `*mut lv_obj_t` |
| `home_time_widget_t *ht` | `ht: *mut home_time_widget_t` |
| `lv_obj_t *screen_time_create(...)` | `fn screen_time_create(...)` <br> &nbsp;&nbsp;`-> *mut lv_obj_t` |

As we convert code from C to Rust, we'll find ourselves doing a lot of this Name/Type Flipping.

Rust is strict about Mutability of variables (whether a variable's value may be modified). `*mut` declares that the pointer refers to an object that is Mutable (i.e. may be modified). For objects that may not be modified, we write `*const` (similar to C).

Here's the C function declaration converted to Rust...

| __Original C Code__ | __Converted Rust Code__ |
| :--- | :--- |
| `lv_obj_t *screen_time_create(` <br> &nbsp;&nbsp;`home_time_widget_t *ht)` | `fn screen_time_create(` <br> &nbsp;&nbsp;`ht: *mut home_time_widget_t)` <br> &nbsp;&nbsp;`-> *mut lv_obj_t` |
| _From [widgets/home_time/screen_time.c](https://github.com/bosmoment/PineTime-apps/blob/master/widgets/home_time/screen_time.c)_ | _From [rust/app/src/watch_face.rs](https://github.com/lupyuen/pinetime-rust-riot/blob/master/rust/app/src/watch_face.rs)_ |

# Variable Declaration

Now let's convert this variable declaration from C to Rust...

```c
lv_obj_t *scr = lv_obj_create( ... ); 
```

`scr` is a pointer to a C Struct `lv_obj_t`. `scr` is set to the value returned by the C function LVGL `lv_obj_create` (which creates a LVGL Screen).

In Rust, variables are declared with the `let` keyword, followed by the variable name and type...

```rust
let scr: *mut lv_obj_t = lv_obj_create( ... );
```

_(Yep we did the Name/Type Flipping again)_

Here's a really cool thing about Rust... Types are optional in variable declarations!

We may drop the type `*mut lv_obj_t`, resulting in this perfectly valid Rust declaration...

```Rust
let scr = lv_obj_create( ... );
```

_What is this type dropping magic? Won't Rust complain about the missing type?_

If we think about it... `lv_obj_create` is a C function already declared somewhere. The Rust Compiler already knows that `lv_obj_create` returns a value of type `*mut lv_obj_t`.

Thus the Rust Compiler uses __Type Inference__ to deduce that `scr` must have type `*mut lv_obj_t`!

This saves us a lot of rewriting when we convert C code to Rust.

Here's how it looks when we convert to Rust the two variable declarations from our C function...

| __Original C Code__ | __Converted Rust Code__ |
| :--- | :--- |
| `lv_obj_t *screen_time_create(` <br> &nbsp;&nbsp;`home_time_widget_t *ht) {` | `fn screen_time_create(` <br> &nbsp;&nbsp;`ht: *mut home_time_widget_t)` <br> &nbsp;&nbsp;`-> *mut lv_obj_t {` <br> |
| &nbsp;&nbsp;`//  Create a label for time (00:00)` | &nbsp;&nbsp;`//  Create a label for time (00:00)` |
| &nbsp;&nbsp;`lv_obj_t *scr = lv_obj_create( ... );` | &nbsp;&nbsp;`let scr = lv_obj_create( ... );` |
| &nbsp;&nbsp;`lv_obj_t *label1 = lv_label_create(scr, ... );` | &nbsp;&nbsp;`let label1 = lv_label_create(scr, ... );` |
| _From [widgets/home_time/screen_time.c](https://github.com/bosmoment/PineTime-apps/blob/master/widgets/home_time/screen_time.c)_ | _From [rust/app/src/watch_face.rs](https://github.com/lupyuen/pinetime-rust-riot/blob/master/rust/app/src/watch_face.rs)_ |
<br>

The parameters are missing from the above code... Let's learn to convert `NULL` to Rust.

# Null Pointers

`NULL` is an unfortunate fact of life for C coders. In our C code we pass two `NULL` pointers to `lv_obj_create`...

```c
//  In C: Call lv_obj_create passing 2 NULL pointers
lv_obj_t *scr = lv_obj_create(NULL, NULL); 
```

Both `NULL`s look the same to C... But not to Rust! Let's look at the function declaration in C...

```c
//  In C: Function declaration for lv_obj_create
lv_obj_t * lv_obj_create(lv_obj_t *parent, const lv_obj_t *copy);
```
_From https://github.com/littlevgl/lvgl/blob/master/src/lv_core/lv_obj.h_

See the difference? The first parameter is a non-`const` pointer (i.e. it's Mutable), whereas the second parameter is a `const` pointer.

Here's how we pass the two `NULL` pointers in Rust...

```rust
//  In Rust: Call lv_obj_create passing 2 NULL pointers: 1 mutable, 1 const
let scr = lv_obj_create(ptr::null_mut(), ptr::null());
```

`null_mut` creates a `NULL` Mutable pointer, `null` creates a Non-Mutable `const NULL` pointer.

`ptr` references the Rust Core Library, which we import like this...

```rust
//  In Rust: Import the Rust Core Library for pointer handling
use core::ptr;
```

When we insert the `NULL` parameters into the converted Rust code, we get this...

| __Original C Code__ | __Converted Rust Code__ |
| :--- | :--- |
| `lv_obj_t *screen_time_create(` <br> &nbsp;&nbsp;`home_time_widget_t *ht) {` | `fn screen_time_create(` <br> &nbsp;&nbsp;`ht: *mut home_time_widget_t)` <br> &nbsp;&nbsp;`-> *mut lv_obj_t {` <br> |
| &nbsp;&nbsp;`//  Create a label for time (00:00)` | &nbsp;&nbsp;`//  Create a label for time (00:00)` |
| &nbsp;&nbsp;`lv_obj_t *scr = lv_obj_create(` | &nbsp;&nbsp;`let scr = lv_obj_create(` |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;__`NULL,`__ | &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;__`ptr::null_mut(),`__ |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;__`NULL`__ | &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;__`ptr::null()`__ |
| &nbsp;&nbsp;`);` | &nbsp;&nbsp;`);` |
| &nbsp;&nbsp;`lv_obj_t *label1 = lv_label_create(`&nbsp;&nbsp;&nbsp;&nbsp; | &nbsp;&nbsp;`let label1 = lv_label_create(` |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`scr,` | &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`scr,` |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;__`NULL`__ | &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;__`ptr::null()`__ |
| &nbsp;&nbsp;`);` | &nbsp;&nbsp;`);` |
| _From [widgets/home_time/screen_time.c](https://github.com/bosmoment/PineTime-apps/blob/master/widgets/home_time/screen_time.c)_ | _From [rust/app/src/watch_face.rs](https://github.com/lupyuen/pinetime-rust-riot/blob/master/rust/app/src/watch_face.rs)_ |
<br>

# Import C Functions into Rust

Let's look back at the C code that we're convering to Rust...

```c
//  In C: Create a label for time (00:00)
lv_obj_t *scr = lv_obj_create(NULL, NULL);
lv_obj_t *label1 = lv_label_create(scr, NULL);

//  Set the text, width and height of the label
lv_label_set_text(label1, "00:00");
lv_obj_set_width(label1, 240);
lv_obj_set_height(label1, 200);
```

The `lv_...` functions called above come from the LVGL library. Here are the function declarations in C...

```c
//  In C: LVGL Function Declarations
lv_obj_t * lv_obj_create(lv_obj_t *parent, const lv_obj_t *copy);
lv_obj_t * lv_label_create(lv_obj_t *par, const lv_obj_t *copy);
void lv_label_set_text(lv_obj_t *label, const char *text);
void lv_obj_set_width(lv_obj_t *obj, int16_t w);
void lv_obj_set_height(lv_obj_t *obj, int16_t h);
```
_From https://github.com/littlevgl/lvgl/blob/master/src/lv_core/lv_obj.h, https://github.com/littlevgl/lvgl/blob/master/src/lv_objx/lv_label.h_

To call these C functions from Rust, we need to import them with `extern "C"` like this...

```rust
//  In Rust: Import LVGL Functions
extern "C" {
    fn lv_obj_create(parent: *mut lv_obj_t, copy: *const lv_obj_t) -> *mut lv_obj_t;
    fn lv_label_create(par: *mut lv_obj_t, copy: *const lv_obj_t) -> *mut lv_obj_t;
    fn lv_label_set_text(label: *mut lv_obj_t, text: *const u8);
    fn lv_obj_set_width(obj: *mut lv_obj_t, w: i16);
    fn lv_obj_set_height(obj: *mut lv_obj_t, h: i16);
}
```
_From https://github.com/lupyuen/pinetime-rust-riot/blob/master/rust/lvgl/src/core/obj.rs, https://github.com/lupyuen/pinetime-rust-riot/blob/master/rust/lvgl/src/objx/label.rs_

_See the Name/Type Flipping? We did it again!_

Take note of the `*mut` and `*const` pointers... Rust is very picky about Mutability!

What's `*const u8`? It's complicated... We'll talk about strings in a while.

Once the C functions have been imported, we may call them in Rust like this...

| __Original C Code__ | __Converted Rust Code__ |
| :--- | :--- |
| `lv_obj_t *screen_time_create(` <br> &nbsp;&nbsp;`home_time_widget_t *ht) {` | `fn screen_time_create(` <br> &nbsp;&nbsp;`ht: *mut home_time_widget_t)` <br> &nbsp;&nbsp;`-> *mut lv_obj_t {` <br> |
| &nbsp;&nbsp;`//  Create a label for time (00:00)` | &nbsp;&nbsp;`//  Create a label for time (00:00)` |
| &nbsp;&nbsp;`lv_obj_t *scr = lv_obj_create(` | &nbsp;&nbsp;`let scr = lv_obj_create(` |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`NULL, NULL` | &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`ptr::null_mut(), ptr::null()` |
| &nbsp;&nbsp;`);` | &nbsp;&nbsp;`);` |
| &nbsp;&nbsp;`lv_obj_t *label1 = lv_label_create(`&nbsp;&nbsp;&nbsp;&nbsp; | &nbsp;&nbsp;`let label1 = lv_label_create(` |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`scr, NULL` | &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`scr, ptr::null()` |
| &nbsp;&nbsp;`);` | &nbsp;&nbsp;`);` |
| &nbsp;&nbsp;`//  Set the text, width and height` | &nbsp;&nbsp;`//  Set the text, width and height` |
| &nbsp;&nbsp;`lv_label_set_text(` | &nbsp;&nbsp;`lv_label_set_text(` |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`label1, "00:00"` | &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`label1, //  TODO` |
| &nbsp;&nbsp;`);` | &nbsp;&nbsp;`);` |
| &nbsp;&nbsp;`lv_obj_set_width(` | &nbsp;&nbsp;`lv_obj_set_width(` |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`label1, 240` | &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`label1, 240` |
| &nbsp;&nbsp;`);` | &nbsp;&nbsp;`);` |
| &nbsp;&nbsp;`lv_obj_set_height(` | &nbsp;&nbsp;`lv_obj_set_height(` |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`label1, 200` | &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`label1, 200` |
| &nbsp;&nbsp;`);` | &nbsp;&nbsp;`);` |
| _From [widgets/home_time/screen_time.c](https://github.com/bosmoment/PineTime-apps/blob/master/widgets/home_time/screen_time.c)_ | _From [rust/app/src/watch_face.rs](https://github.com/lupyuen/pinetime-rust-riot/blob/master/rust/app/src/watch_face.rs)_ |
<br>

# Numeric Types

Something interesting happened when we took this C function declaration...

```c
//  In C: Function declaration for lv_obj_set_width
void lv_obj_set_width(lv_obj_t *obj, int16_t w);
```

And imported it into Rust...

```rust
//  In Rust: Import lv_obj_set_width function from C
extern "C" {
    fn lv_obj_set_width(obj: *mut lv_obj_t, w: i16);
}
```

_Look at the second parameter... How did `int16_t` in C (16-bit signed integer) become `i16` in Rust?_

You might have guessed... Numeric Types in Rust have no-nonsense, super-compact names!

So `int16_t` gets shortened to `i16`. `uint16_t` (unsigned 16-bit integer) gets shortened to `u16`.

Numeric Types are such a joy to write!  And there's no need to `#include <stdint.h>`

| __C Numeric Type__ &nbsp;&nbsp; | __Rust Numeric Type__ |
| :--- | :---: |
| `int8_t` | `i8` |
| `uint8_t` | `u8` |
| `int16_t` | `i16` |
| `uint16_t` | `u16` |
| `int32_t` | `i32` |
| `uint32_t` | `u32` |
| `int64_t` | `i64` |
| `uint64_t` | `u64` |
| `float` | `f32` |
| `double` | `f64` |
<br>

In Rust we use `u8` to refer to a byte.

# Pass Strings from Rust to C

Rust has a powerful `String` type for manipulating strings (stored in heap memory)... But we'll look at a simpler way to pass strings from Rust to C.

This is our original C code...

```c
//  In C: Declare function lv_label_set_text
void lv_label_set_text(lv_obj_t *label, const char *text);
...
//  Set the text of the label to "00:00"
lv_label_set_text(label1, "00:00");
```

Here's how we pass the string `"00:00"` from Rust to C...

```rust
//  In Rust: Import function lv_label_set_text from C
extern "C" {
    fn lv_label_set_text(label: *mut lv_obj_t, text: *const u8);
}
...
//  Set the text of the label to "00:00"
lv_label_set_text(
    label1,
    b"00:00\0".as_ptr()
);
```

Remember that `u8` in Rust means unsigned byte, so `*const u8` in Rust is similar to `const char *` in C.

Let's compare the C string and its Rust equivalent...

| __C String__ &nbsp;&nbsp; | __Rust Equivalent__ |
| :--- | :--- |
| `"00:00"`&nbsp;&nbsp;&nbsp;&nbsp; | `b"00:00\0".as_ptr()` |
<br>

The `b"`...`"` notation creates a Rust [Byte String](https://doc.rust-lang.org/reference/tokens.html#byte-string-literals). A Byte String is an array of bytes, similar to strings in C.

Unlike C, strings in Rust don't have a terminating null. So we manually added the null: `\0`

In C, arrays and pointers are interchangeable, so `char *` behaves like `char[]`... But not in Rust! 

Rust arrays have an internal counter that remembers the length of the array. Which explains why Rust strings don't have a terminating null... Rust internally tracks the length of each string.

To convert a Rust array to a pointer, we use `as_ptr()` as shown above.

_What happens if we forget to add the terminating null `\0`? Catastrophe!_

The C function `lv_label_set_text` will get very confused without the terminating null. So the above Byte String notation `b"`...`"` is prone to problems.

Later we'll see an easier, safer way to write strings... With a Rust Macro.

```rust
//  In Rust: Set the label text with a macro
lv_label_set_text(
    label1,
    strn!("00:00")
);
```

# Pointer Dereferencing

In C we write `->` to dereference a pointer and access a Struct field...

```c
//  In C: Dereference the pointer ht and set the lv_time field
ht->lv_time = label1;
```

Rust doesn't have a combined operator for dereferencing pointers and accessing Struct fields. Instead, we use the `*` and `.` operators, which have the same meanings as in C...

```rust
//  In Rust: Dereference the pointer ht and set the lv_time field
(*ht).lv_time = label1;
```

# Return Value

In C we use the `return` keyword to set the return value of the current function...

```c
lv_obj_t *screen_time_create(home_time_widget_t *ht) {
    ...
    //  In C: Return scr as the value of the function
    return scr;
}
```

In Rust the `return` keyword works the same way...

```rust
fn screen_time_create(ht: *mut home_time_widget_t) -> *mut lv_obj_t { 
    ...
    //  In Rust: Return scr as the value of the function
    return scr;
}
```

Another way to set the return value in Rust: Just write the value as the last expression of the function...

```rust
fn screen_time_create(ht: *mut home_time_widget_t) -> *mut lv_obj_t { 
    ...
    //  In Rust: Return scr as the value of the function. Note: No semicolon ";" at the end
    scr
}
```

If we use this convention, the last expression of the function should not end with a semicolon.

# C to Rust Conversion: First Version

Following the steps above, we'll get this line-by-line conversion from C to Rust...

| __Original C Code__ | __Converted Rust Code__ |
| :--- | :--- |
| `lv_obj_t *screen_time_create(` <br> &nbsp;&nbsp;`home_time_widget_t *ht) {` | `fn screen_time_create(` <br> &nbsp;&nbsp;`ht: *mut home_time_widget_t)` <br> &nbsp;&nbsp;`-> *mut lv_obj_t {` <br> |
| &nbsp;&nbsp;`//  Create a label for time (00:00)` | &nbsp;&nbsp;`//  Create a label for time (00:00)` |
| &nbsp;&nbsp;`lv_obj_t *scr = lv_obj_create(` | &nbsp;&nbsp;`let scr = lv_obj_create(` |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`NULL, NULL` | &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`ptr::null_mut(), ptr::null()` |
| &nbsp;&nbsp;`);` | &nbsp;&nbsp;`);` |
| &nbsp;&nbsp;`lv_obj_t *label1 = lv_label_create(`&nbsp;&nbsp;&nbsp;&nbsp; | &nbsp;&nbsp;`let label1 = lv_label_create(` |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`scr, NULL` | &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`scr, ptr::null()` |
| &nbsp;&nbsp;`);` | &nbsp;&nbsp;`);` |
| &nbsp;&nbsp;`//  Set the text, width and height` | &nbsp;&nbsp;`//  Set the text, width and height` |
| &nbsp;&nbsp;`lv_label_set_text(` | &nbsp;&nbsp;`lv_label_set_text(` |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`label1, "00:00"` | &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`label1, b"00:00\0".as_ptr()` |
| &nbsp;&nbsp;`);` | &nbsp;&nbsp;`);` |
| &nbsp;&nbsp;`lv_obj_set_width(` | &nbsp;&nbsp;`lv_obj_set_width(` |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`label1, 240` | &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`label1, 240` |
| &nbsp;&nbsp;`);` | &nbsp;&nbsp;`);` |
| &nbsp;&nbsp;`lv_obj_set_height(` | &nbsp;&nbsp;`lv_obj_set_height(` |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`label1, 200` | &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`label1, 200` |
| &nbsp;&nbsp;`);` | &nbsp;&nbsp;`);` |
| &nbsp;&nbsp;`ht->lv_time = label1;` | &nbsp;&nbsp;`(*ht).lv_time = label1;` |
| &nbsp;&nbsp;`return scr;` | &nbsp;&nbsp;`scr` |
| `}` | `}` |
| _From [widgets/home_time/screen_time.c](https://github.com/bosmoment/PineTime-apps/blob/master/widgets/home_time/screen_time.c)_ | _From [rust/app/src/watch_face.rs](https://github.com/lupyuen/pinetime-rust-riot/blob/master/rust/app/src/watch_face.rs)_ |
<br>

The importing of C functions into Rust has been omitted from the code above. Now let's learn to import C Structs and Enums into Rust.

# Import C Structs into Rust

`home_time_widget_t` is a C Struct that's passed as a parameter into our Rust function. Here's how we import `home_time_widget_t` into Rust...

| __Original C Code__ | __Converted Rust Code__ |
| :--- | :--- |
| `typedef struct _home_time_widget {` | `#[repr(C)]` <br> `struct home_time_widget_t {` |
|    &nbsp;&nbsp;`widget_t widget;` | &nbsp;&nbsp;`widget: widget_t,` |
|    &nbsp;&nbsp;`control_event_handler_t handler;` | &nbsp;&nbsp;`handler: control_event_handler_t,` |
|    &nbsp;&nbsp;`lv_obj_t *screen;` | &nbsp;&nbsp;`screen:   *mut lv_obj_t,` |
|    &nbsp;&nbsp;`lv_obj_t *lv_time;` | &nbsp;&nbsp;`lv_time:  *mut lv_obj_t,` |
|    &nbsp;&nbsp;`lv_obj_t *lv_date;` | &nbsp;&nbsp;`lv_date:  *mut lv_obj_t,` |
|    &nbsp;&nbsp;`lv_obj_t *lv_ble;` | &nbsp;&nbsp;`lv_ble:   *mut lv_obj_t,` |
|    &nbsp;&nbsp;`lv_obj_t *lv_power;` | &nbsp;&nbsp;`lv_power: *mut lv_obj_t,` |
|    &nbsp;&nbsp;`bleman_ble_state_t ble_state;` | &nbsp;&nbsp;`ble_state: bleman_ble_state_t,` |
|    &nbsp;&nbsp;`controller_time_spec_t time;` | &nbsp;&nbsp;`time: controller_time_spec_t,` |
|    &nbsp;&nbsp;`uint32_t millivolts;` | &nbsp;&nbsp;`millivolts: u32,` |
|    &nbsp;&nbsp;`bool charging;` | &nbsp;&nbsp;`charging: bool,` |
|    &nbsp;&nbsp;`bool powered;` | &nbsp;&nbsp;`powered: bool,` |
|`} home_time_widget_t;` | `}` |
| _From https://github.com/bosmoment/PineTime-apps/blob/master/widgets/home_time/include/home_time.h_ | _From [rust/app/src/watch_face.rs](https://github.com/lupyuen/pinetime-rust-riot/blob/master/rust/app/src/watch_face.rs)_ |
<br>

Note the Name/Type Flipping. Also semicolons "`;`" have been replaced by commas "`,`".

We'll need to import the C types `widget_t`, `control_event_handler_t`, `lv_obj_t`, `bleman_ble_state_t` and `controller_time_spec_t` the same way.

_What's `#[repr(C)]`?_

The Rust Compiler is really clever in [laying out Struct fields to save storage space](https://doc.rust-lang.org/nomicon/repr-rust.html).  Unfortunately this optimised layout is not compatible with C... Rust would not be able to access correctly the Struct fields passed from C.

To fix this, we specify `#[repr(C)]`. This tells the Rust Compiler that the Struct uses the C layout for fields instead of the Rust layout.

# Import C Enums into Rust

The Struct above contains a C Enum `bleman_ble_state_t`. Here's how we import `bleman_ble_state_t` into Rust...

| __Original C Code__ | __Converted Rust Code__ |
| :--- | :--- |
| `typedef enum {` | `#[repr(u8)]` <br> `#[derive(PartialEq)]` <br> `enum bleman_ble_state_t {` |
| &nbsp;&nbsp;`BLEMAN_BLE_STATE_INACTIVE,` | &nbsp;&nbsp;`BLEMAN_BLE_STATE_INACTIVE = 0,` |
| &nbsp;&nbsp;`BLEMAN_BLE_STATE_ADVERTISING,` | &nbsp;&nbsp;`BLEMAN_BLE_STATE_ADVERTISING = 1,` |
| &nbsp;&nbsp;`BLEMAN_BLE_STATE_DISCONNECTED,` | &nbsp;&nbsp;`BLEMAN_BLE_STATE_DISCONNECTED = 2,` |
| &nbsp;&nbsp;`BLEMAN_BLE_STATE_CONNECTED,` | &nbsp;&nbsp;`BLEMAN_BLE_STATE_CONNECTED = 3,` |
| `} bleman_ble_state_t;` | `}` |
| _From https://github.com/bosmoment/PineTime-apps/blob/master/modules/bleman/include/bleman.h_ | _From [rust/app/src/watch_face.rs](https://github.com/lupyuen/pinetime-rust-riot/blob/master/rust/app/src/watch_face.rs)_ |
<br>

Note that we specified in Rust the Enum values `0, 1, 2, 3` to avoid any possible ambiguity.

_What's `#[repr(u8)]`?_

Recall that `u8` refers to an unsigned byte. When we specify `#[repr(u8)]`, we tell the Rust Compiler that this Enum uses 8 bits to [store the value of the Enum](https://doc.rust-lang.org/nomicon/other-reprs.html#repru-repri).

Thus the code above assumes that the C Enum value passed into our Rust function is 8 bits wide.

_What's the size of a C Enum? 8 bits, 16 bits, 32 bits, ...?_

That depends on the values in the C Enum. Check this article for details: [_"How Big Is An Enum?"_](https://embedded.fm/blog/2016/6/28/how-big-is-an-enum)

_What's `#[derive(PartialEq)]`?_

`#[derive(PartialEq)]` is needed so that we may [compare Enum values](https://doc.rust-lang.org/std/cmp/trait.PartialEq.html) like this...

```rust
//  In Rust: Compare an enum value
if state.ble_state == bleman_ble_state_t::BLEMAN_BLE_STATE_DISCONNECTED { ...
```
_From [rust/app/src/watch_face.rs](https://github.com/lupyuen/pinetime-rust-riot/blob/master/rust/app/src/watch_face.rs)_

Note that Enum values are prefixed by the Enum type name, like `bleman_ble_state_t::...`

_Importing of C functions and types looks tedious and error-prone... Is there a better way to import C functions and types into Rust?_

Yes! Later we'll look at an automated way to import C functions and types: `bindgen`

# Unsafe Code in Embedded Rust

Earlier we took this C code...

```c
//  In C: Declare function lv_label_set_text
void lv_label_set_text(lv_obj_t *label, const char *text);
...
//  Set the text of the label to "00:00"
lv_label_set_text(label1, "00:00");
```

And converted it to Rust...

```rust
//  In Rust: Import function lv_label_set_text from C
extern "C" {
    fn lv_label_set_text(label: *mut lv_obj_t, text: *const u8);
}
...
//  Set the text of the label to "00:00"
lv_label_set_text(
    label1,
    b"00:00\0".as_ptr()
);
```

Recall that `b"00:00\0".as_ptr()` is the Rust Byte String equivalent of `"00:00"` in C.  This is the string that's passed by the above Rust code to the C function `lv_label_set_text`.

_What happens when we remove `\0` from the Rust Byte String?_

`lv_label_set_text` will receive an invalid string that's not terminated by null.

`lv_label_set_text` may get stuck forever searching for the terminating null. Or it may attempt to copy a ridiculously huge string and corrupt the system memory.

_Surely the Rust Compiler can verify that all Rust Byte Strings as null terminated... Right?_

Well if we look at the calling contract that we have agreed with C...

```c
//  In C: Declare function lv_label_set_text
void lv_label_set_text(lv_obj_t *label, const char *text);
```

It doesn't say that `text` requires a terminating null... Legally we may pass in any `const char *` pointer!

Calling `lv_label_set_text` is an example of __Unsafe Code__ in Rust.  That's the Rust Compiler saying...

> I'm sorry, Dave. I'm afraid I can't do that. I won't let you call function `lv_label_set_text` because I'm not sure whether the C function will cause memory corruption or cause the system to crash. I'm not even sure if the function `lv_label_set_text` will ever return!

To override HAL... er... the Rust Compiler, we need to wrap the Unsafe Code with the `unsafe` keyword...

```rust
//  In Rust: Set the text of the label to "00:00"
unsafe {
    lv_label_set_text(
        label1,
        b"00:00\0".as_ptr()
    );
}
```

This needs to be done for _every C function_ that we call from Rust. Which will look incredibly messy.

Later we'll see the fix for this: Safe Wrappers. 

# Import C Types and Functions into Rust with `bindgen`

Earlier we used this Rust code to import C functions from the LVGL library into Rust...

```rust
//  In Rust: Import LVGL Functions
extern "C" {
    fn lv_obj_create(parent: *mut lv_obj_t, copy: *const lv_obj_t) -> *mut lv_obj_t;
    fn lv_label_create(par: *mut lv_obj_t, copy: *const lv_obj_t) -> *mut lv_obj_t;
    fn lv_label_set_text(label: *mut lv_obj_t, text: *const u8);
    fn lv_obj_set_width(obj: *mut lv_obj_t, w: i16);
    fn lv_obj_set_height(obj: *mut lv_obj_t, h: i16);
}
```
_From https://github.com/lupyuen/pinetime-rust-riot/blob/master/rust/lvgl/src/core/obj.rs, https://github.com/lupyuen/pinetime-rust-riot/blob/master/rust/lvgl/src/objx/label.rs_

The above Rust code was automatically generated by a [command-line tool named `bindgen`](https://rust-lang.github.io/rust-bindgen/command-line-usage.html). We install `bindgen` and run it like this...

```bash
cargo install bindgen
bindgen lv_obj.h -o obj.rs
```

`bindgen` takes a C Header File (like [`lv_obj.h`](https://github.com/littlevgl/lvgl/blob/master/src/lv_core/lv_obj.h) from LVGL) and generates the Rust code (like in `obj.rs` above) to import the C types and functions declared in the Header File.

Thus `bindgen` is a tool that generates __Rust Bindings__ for C types and functions...

- Take a peek at this [C Header File from LitlevGL](https://github.com/littlevgl/lvgl/blob/master/src/lv_core/lv_obj.h)

- And the [Rust Bindings generated by `bindgen`](https://github.com/lupyuen/pinetime-rust-riot/blob/master/rust/lvgl/src/core/obj.rs)

_What if the C Header File includes other Header Files?_

Yep that makes `bindgen` more complicated... Because `bindgen` can't generate bindings unless it knows the definition of every C type referenced by our Header File.

Here's how we specify the Include Folders for the Header Files...

```bash
bindgen lv_obj.h -o obj.rs \
    -- \
    -Ibaselibc/include/ \
    -Iapps/pinetime/bin/pkg/pinetime/ \
    -Iapps/pinetime \
    -DRIOT_BOARD=BOARD_PINETIME \
    -DRIOT_CPU=CPU_NRF52 \
    -DRIOT_MCU=MCU_NRF52 \
    -std=c99 \
    -fno-common
```
_From https://github.com/lupyuen/pinetime-rust-riot/blob/master/scripts/gen-bindings.sh_

After `--`, we add the same `gcc` options we would use for compiling the Embedded C code (for RIOT in this case)...

- `-I` for Include Folders

- `-D` for C Preprocessor Definitions (because they may affect the size of C types)

- Other `gcc` options like `-std=c99` and `-fno-common` so that `bindgen` understands how to parse our Header Files

Take a peek at the complete list of `bindgen` options we used to create Rust Bindings for the LVGL library: [gen-bindings.sh](https://github.com/lupyuen/pinetime-rust-riot/blob/master/scripts/gen-bindings.sh)

_How did we get that awfully long list of `bindgen` options?_

When we build the Embedded C code with `make --trace`, we'll see the options passed to `gcc`. These are the options that we should pass to `bindgen` as well.

# Whitelist and Blacklist C Types and Functions in `bindgen`

To build Watch Faces on PineTime Smart Watch, we need to call two groups of functions in LVGL...

1. [Base Object Functions `lv_obj_*`](https://docs.littlevgl.com/en/html/object-types/obj.html): Set the width and height of Widgets (like Labels). Also to create the Screen object. Defined in [`lv_obj.h`](https://github.com/littlevgl/lvgl/blob/master/src/lv_core/lv_obj.h)

1. [Label Functions `lv_label_*`](https://docs.littlevgl.com/en/html/object-types/label.html): Create Label Widgets and set the text of the Labels. Defined in [`lv_label.h`](https://github.com/littlevgl/lvgl/blob/master/src/lv_objx/lv_label.h)

To call both groups of functions from Rust, we need to run `bindgen` twice...

```bash
# Generate Rust Bindings for LVGL Base Object Functions lv_obj_*
bindgen lv_obj.h   -o obj.rs   -- -Ibaselibc/include/ ...

# Generate Rust Bindings for LVGL Label Functions lv_label_*
bindgen lv_label.h -o label.rs -- -Ibaselibc/include/ ...
```

_There's a problem with duplicate definitions... Do you see the problem?_

[`lv_label.h`](https://github.com/littlevgl/lvgl/blob/master/src/lv_objx/lv_label.h) includes [`lv_obj.h`](https://github.com/littlevgl/lvgl/blob/master/src/lv_core/lv_obj.h). So `bindgen` helpfully creates Rust Bindings for the Base Object Functions _twice_: In `obj.rs` and again in `label.rs`

The Rust Compiler is not gonna like this. To solve this, we __Whitelist and Blacklist__ the items that we should include ([Whitelist](https://rust-lang.github.io/rust-bindgen/whitelisting.html)) and exclude ([Blacklist](https://rust-lang.github.io/rust-bindgen/blacklisting.html))...

```bash
# Generate Rust Bindings for LVGL Base Object Functions lv_obj_*
bindgen lv_obj.h   -o obj.rs \
    --whitelist-function '(?i)lv_.*' \
    --whitelist-type     '(?i)lv_.*' \
    --whitelist-var      '(?i)lv_.*' \
    -- -Ibaselibc/include/ ...

# Generate Rust Bindings for LVGL Label Functions lv_label_*
bindgen lv_label.h -o label.rs \
    --whitelist-function '(?i)lv_label.*' \
    --whitelist-type     '(?i)lv_label.*' \
    --whitelist-var      '(?i)lv_label.*' \
    --blacklist-item     _lv_obj_t \
    --blacklist-item     lv_style_t \
    -- -Ibaselibc/include/ ...
```

`whitelist-function`, `whitelist-type` and `whitelist-var` tells `bindgen` to generate bindings only for C functions, types and variables that match a pattern.

`blacklist-item` tells `bindgen` to suppress bindings for functions, types and variables with that name.

`(?i)` tells `bindgen` to ignore the case and match both uppercase and lowercase versions of the name. [More about Rust Regular Expressions](https://docs.rs/regex/1.3.6/regex/)

When we write...

```
bindgen lv_label.h -o label.rs \
    --blacklist-item _lv_obj_t
```

We tell `bindgen` not to create Rust Bindings for `_lv_obj_t` even though it's included by `lv_label.h`. This solves our problem of duplicate Rust Bindings. And the Rust Compiler loves us for doing that!

- Take a peek at this [`lv_label.h`](https://github.com/littlevgl/lvgl/blob/master/src/lv_objx/lv_label.h)

- And the [Rust Bindings generated by `bindgen`](https://github.com/lupyuen/pinetime-rust-riot/blob/master/rust/lvgl/src/objx/label.rs)

No more duplicate Rust Bindings!

When using `bindgen` in real projects we'll need to add more [command-line options](https://rust-lang.github.io/rust-bindgen/command-line-usage.html). Here's how we actually used `bindgen` to create the Rust Bindings in our PineTime Watch Face project...

```bash
# Generate Rust Bindings for LVGL Base Object Functions lv_obj_*
bindgen --verbose --use-core --ctypes-prefix ::cty --with-derive-default --no-derive-copy --no-derive-debug --no-layout-tests --raw-line use --raw-line 'super::*;' --whitelist-function '(?i)lv_.*' --whitelist-type '(?i)lv_.*' --whitelist-var '(?i)lv_.*' -o rust/lvgl/src/core/obj.tmp apps/pinetime/bin/pkg/pinetime/lvgl/src/lv_core/lv_obj.h -- -Ibaselibc/include/ ...

# Generate Rust Bindings for LVGL Label Functions lv_label_*
bindgen --verbose --use-core --ctypes-prefix ::cty --with-derive-default --no-derive-copy --no-derive-debug --no-layout-tests --raw-line use --raw-line 'super::*;' --whitelist-function '(?i)lv_label.*' --whitelist-type '(?i)lv_label.*' --whitelist-var '(?i)lv_label.*' --blacklist-item _lv_obj_t --blacklist-item lv_style_t -o rust/lvgl/src/objx/label.tmp apps/pinetime/bin/pkg/pinetime/lvgl/src/lv_objx/lv_label.h -- -Ibaselibc/include/ ...
```

The shell script used to create the Rust Bindings is here: [gen-bindings.sh](https://github.com/lupyuen/pinetime-rust-riot/blob/master/scripts/gen-bindings.sh)

Here's the output log for the script: [gen-bindings.log](https://github.com/lupyuen/pinetime-rust-riot/blob/master/logs/gen-bindings.log)

# Safe Wrappers for Imported C Functions

To display the current time in our PineTime Watch Face, we need to call `lv_label_set_text` imported from the LVGL library...

```rust
//  In Rust: Import function lv_label_set_text from C
extern "C" {
    fn lv_label_set_text(label: *mut lv_obj_t, text: *const u8);
}
...
//  Set the text of the label to "00:00"
unsafe {
    lv_label_set_text(
        label1,
        b"00:00\0".as_ptr()
    );
}
```

It's not surprising that the Rust Compiler considers this code `unsafe`... If we forget to add the terminating null `\0`, `lv_label_set_text` might behave strangely and cause our watch to crash!

_Can we exploit the power of Type Checking in the Rust Compiler to make this code safer?_

Yes we can! Check this out...

```rust
//  In Rust: Wrapper function to set the text of a label
fn set_text(label: *mut lv_obj_t, text: &Strn) {
    text.validate();  //  Validate that the string is null-terminated
    unsafe {
        lv_label_set_text(
            label,
            text.as_ptr()
        );
    }
}
...
//  Set the text of the label to "00:00", the safe way
set_text(
    label1,
    strn!("00:00")
);
```
_From https://github.com/lupyuen/pinetime-rust-riot/blob/master/logs/liblvgl-expanded.rs#L9239-L9259_

`set_text` is a Wrapper Function that provides a safe way to call `lv_label_set_text`

Now we simply call `set_text` instead of `lv_label_set_text`... No more `unsafe` code!

Instead of passing unsafe C pointers to the text string, we now pass an `Strn` object. 

`Strn` is a Rust Struct that we have defined to [pass null-terminated strings to C functions](https://github.com/lupyuen/pinetime-rust-riot/blob/master/rust/lvgl/src/lib.rs#L82-L211).  We create an `Strn` object with the [Rust Macro `strn!`](https://github.com/lupyuen/pinetime-rust-riot/blob/master/rust/macros/src/lib.rs#L62-L128)...

```rust
strn!("00:00")
```

Note the validation done in the Wrapper Function...

```rust
//  Wrapper function to set the text of a label
fn set_text(label: *mut lv_obj_t, text: &Strn) {
    text.validate();  //  Validate that the string is null-terminated
    ...
```

The Wrapper Function always checks to ensure that [the string is null-terminated](https://github.com/lupyuen/pinetime-rust-riot/blob/master/rust/lvgl/src/lib.rs#L181-L190) before calling the C function. Crashing Watches Averted!

_But do we need to write this Wrapper Function ourselves for every C function?_

Not necessary... The Safe Wrappers may be automatically generated! Let's learn how with a Rust Procedural Macro.

# Generate Safe Wrappers with Rust Procedural Macro

As we have seen, to create a PineTime Watch Face we need to...

1. Run `bindgen` to import the LVGL function `lv_label_set_text` from C into Rust

1. Create a Safe Wrapper function in Rust to call `lv_label_set_text` safely

Trick Question: What's the difference between this Rust Binding code generated by `bindgen`...

```rust
//  In Rust: Import function lv_label_set_text from C to set the text of a label
#[lvgl_macros::safe_wrap(attr)]
extern "C" {
    pub fn lv_label_set_text(
        label: *mut lv_obj_t, 
        text:  *const ::cty::c_char
    );
}
```
_From https://github.com/lupyuen/pinetime-rust-riot/blob/master/rust/lvgl/src/objx/label.rs_

And this Safe Wrapper function (that calls `lv_label_set_text` safely)?

```rust
//  In Rust: Safe Wrapper function to set the text of a label
pub fn set_text(
    label: *mut lv_obj_t, 
    text:  &Strn
) -> LvglResult< () > {
    extern "C" {
        pub fn lv_label_set_text(
            label: *mut lv_obj_t,
            text:  *const ::cty::c_char
        );
    }
    text.validate();  //  Validate that the string is null-terminated
    unsafe {
        lv_label_set_text(
            label as *mut lv_obj_t,
            text.as_ptr() as *const ::cty::c_char
        );
        Ok(())  //  Return OK
    }
}
```
_From https://github.com/lupyuen/pinetime-rust-riot/blob/master/logs/liblvgl-expanded.rs#L9239-L9259_

_Answer: They are exactly the same!_

The magic happens in this line of code...

```rust
#[lvgl_macros::safe_wrap(attr)]
```

This activates a [__Rust Procedural Macro__](https://blog.rust-lang.org/2018/12/21/Procedural-Macros-in-Rust-2018.html) `safe_wrap` that we have written.  The Rust Compiler calls our Rust function `safe_wrap` during compilation (instead of runtime). [`safe_wrap` is defined here](https://github.com/lupyuen/pinetime-rust-riot/blob/master/rust/macros/src/safe_wrap.rs#L116-L147)

Unlike C Macros, Rust Macros are allowed to __inspect the Rust code__ passed to the macro... And alter the code!

So this whole chunk of Rust code...

```rust
extern "C" {
    pub fn lv_label_set_text(
        label: *mut lv_obj_t, 
        text:  *const ::cty::c_char
        ...
```

Gets passed into our `safe_wrap` function for us to manipulate!

1. `safe_wrap` inspects the imported function name (`lv_label_set_text`), parameter types (`lv_obj_t`, `c_char`) and return type (none)

1. Then `safe_wrap` replaces the chunk of code by the Safe Wrapper function `set_text`, populated with the right parameter types and return type

1. `*const ::cty::c_char` (pointer to a C string, which may or may not be null-terminated) is replaced by the safer `&Strn` (reference to a null-terminated string object)

That's how we automatically generate Safe Wrapper functions (described in the previous section)... For every imported LVGL function.

`safe_wrap` is inserted into the Rust Bindings by the [gen-bindings.sh](https://github.com/lupyuen/pinetime-rust-riot/blob/master/scripts/gen-bindings.sh) script.

_What's `LvglResult< () >` and `Ok(())`?_

We'll find out in the next section: Rust Error Handling.

# Return Errors with the Rust Result Enum

Error Handling in C is kinda messy. Here's a problem that we see often in C...

```c
//  In C: Declare lv_obj_create function that creates a LVGL object
lv_obj_t *lv_obj_create(lv_obj_t *parent, const lv_obj_t *copy);
...
//  Create a screen object
lv_obj_t *screen = lv_obj_create(NULL, NULL); 
//  Get the coordinates of the screen object
lv_area_t coords = screen->coords;
//  Oops! This crashes if screen is NULL
```

This C code failed to check the value returned by `lv_obj_create`. The program crashes if the returned `screen` is `NULL`.

In Rust, we use the [__`Result` Enum__](https://doc.rust-lang.org/core/result/index.html) to ensure that all returned values are checked.

Here's a Safe Wrapper Function `create` that exposes a safer version of `lv_obj_create`. The Safe Wrapper Function uses the `Result` Enum to enforce checking of returned values...

```rust
//  In Rust: Import from C the lv_obj_create function that creates a LVGL object
extern "C" {
    pub fn lv_obj_create(parent: *mut lv_obj_t, copy: *const lv_obj_t)
        -> *mut lv_obj_t;
}

//  Safe Wrapper function to create a LVGL object
pub fn create(parent: *mut lv_obj_t, copy: *const lv_obj_t) 
    -> LvglResult< *mut lv_obj_t > {  //  Returns a lv_obj_t pointer wrapped in a Result Enum
    unsafe {
        //  Create the object by calling the imported C function
        let result = lv_obj_create(parent, copy);
        //  If result is null, return an error
        if result.is_null() { Err( LvglError::SYS_EUNKNOWN ) }
        //  Otherwise return the wrapped result
        else { Ok( result ) }
    }
}
```
_Based on https://github.com/lupyuen/pinetime-rust-riot/blob/master/logs/liblvgl-expanded.rs#L5942-L5967_

What happens in the `create` Safe Wrapper Function?

1. Note that the return type of the `create` function has been changed from `*mut lv_obj_t` (mutable pointer to `lv_obj_t`) to...

   ```rust
   LvglResult< *mut lv_obj_t >
   ```

   `LvglResult` is a `Result` Enum that we have created to [wrap safely all values returned by the LVGL C library](https://github.com/lupyuen/pinetime-rust-riot/blob/master/rust/lvgl/src/lib.rs#L29-L80).

   `LvglResult< *mut lv_obj_t >` says that the returned `LvglResult` Enum will wrap a mutable pointer to `lv_obj_t`.

1. Unlike C Enums, Rust Enums like `LvglResult` can have values inside. The expansion of `LvglResult< *mut lv_obj_t >` looks something like this...

    ```rust
    enum LvglResult< *mut lv_obj_t > {
        Ok( *mut lv_obj_t ),
        Err( LvglError ),
    }  //  This is not valid Rust syntax
    ```

1. The `LvglResult` Enum has two variants: `Ok` and `Err`. To return an error, we return the `Err` variant with an [error code inside](https://github.com/lupyuen/pinetime-rust-riot/blob/master/rust/lvgl/src/lib.rs#L35-L44) (like `SYS_EUNKNOWN`)...

    ```rust
    //  Create the object by calling the imported C function
    let result = lv_obj_create(parent, copy);
    //  If result is null, return an error
    if result.is_null() { Err( LvglError::SYS_EUNKNOWN ) }
    ```

    Here we return an `Err` if the call to `lv_obj_create` returns `NULL`.

1. To return a valid result, we return the `Ok` variant with the result value inside...

    ```rust
    //  Otherwise return the wrapped result
    else { Ok( result ) }
    ```

    Here we return the result of the call to `lv_obj_create`, since it's not `NULL`.

1. The `if else` syntax used above looks odd if you're new to Rust...

    ```rust
    if condition { true_value } 
    else { false_value }
    ```

    In Rust, `if else` evaluates to a value. So the above Rust code is equivalent to this C code with the Ternary Operator...

    ```c
    condition ? true_value : false_value
    ```

In summary: The `create` function calls the C function `lv_obj_create`. If the C function returns `NULL`, `create` returns `Err`. Otherwise `create` returns `Ok` with the result value inside.

All calls to the `create` function must be checked for errors. Let's find out how the Rust Compiler enforces the error checking...

# Check Errors with the Rust Result Enum

Let's learn how the Rust Compiler forces us to check for errors returned by C functions. We'll use this Safe Wrapper function that we have created in the last section...

```rust
//  In Rust: Safe Wrapper function to create a LVGL object
pub fn create(parent: *mut lv_obj_t, copy: *const lv_obj_t) 
    -> LvglResult< *mut lv_obj_t > {  //  Returns an lv_obj_t pointer wrapped in a Result Enum
    ...
```

The `create` function returns `LvglResult< *mut lv_obj_t >` which is a `Result` Enum that's either...

1. `Ok` with an `lv_obj_t` pointer wrapped inside, or

1. `Err` with an error code wrapped inside

Let's try calling `create` without checking the result...

```rust
//  In Rust: Create a LVGL screen object
let screen = create(ptr::null_mut(), ptr::null());
//  Get a reference to the coordinates of the screen object
let coords = &(*screen).coords;
//  Oops! Rust Compiler says result cannot be dereferenced
```

_The C Compiler would happily accept code like this... But not Rust!_

`screen` has become a `Result` Enum that can't be used directly. The Rust Compiler insists that we check for error in `screen` before unwrapping it, like this...

```rust
//  In Rust: We specify `unsafe` to dereference the pointer in `screen`
unsafe {
    //  Create a LVGL screen object and unwrap it
    let screen = create(ptr::null_mut(), ptr::null())
        .expect("no screen");  //  If error, show "no screen" and stop
    //  Get a reference to the coordinates of the screen object
    let coords = &(*screen).coords;
```

By adding `.expect` after `create`, we check for error before unwrapping the pointer inside the result.

If `create` returns an error, the program stops with the error "`no screen`"

There's a simpler way to handle errors in Rust... With the Try Operator "`?`"

```rust
//  In Rust: Create a LVGL screen object and check for error
fn create_screen() -> LvglResult< () > {  //  Returns Ok (with nothing inside) or Err
    //  We specify `unsafe` to dereference the pointer in `screen`
    unsafe {
        //  Create a LVGL screen object and unwrap it
        let screen = create(ptr::null_mut(), ptr::null()) ? ;  //  If error, stop and return the Err
        //  Get a reference to the coordinates of the screen object
        let coords = &(*screen).coords;
        ...
        Ok( () )  //  Return Ok with nothing inside
    }
}
```

Note that `.expect` has been replaced by "`?`"...

```rust
//  Create a LVGL screen object and unwrap it
let screen = create(ptr::null_mut(), ptr::null()) ? ;  //  If error, stop and return the Err
```

If `create` returns `Ok`, the result is unwrapped and assigned to `screen`.

But if `create` returns `Err`, the result is returned to the caller of `create_screen` immediately.

That's why "`?`" works only inside a function that returns a `Result` Enum...

```rust
//  Create a LVGL screen object and check for error
fn create_screen() -> LvglResult< () > {  //  Returns Ok (with nothing inside) or Err
```

The `()` in `LvglResult< () >` means "nothing". Thus `create_screen` returns either...

1. `Ok` with nothing `()` wrapped inside (because we don't need the return value), or

1. `Err` with an error code wrapped inside

If we look at the [Safe Wrappers](https://github.com/lupyuen/pinetime-rust-riot/blob/master/logs/liblvgl-expanded.rs#L5942-L5967) created by our `safe_wrap` macro, it's now obvious why we see so many `LvglResult< () >` and `Ok( () )` inside... That's how we handle errors in Rust.

# C to Rust Conversion: Final Version

Now that we understand `unsafe` code, Safe Wrappers, `Result` Enums and "`?`", this Rust code should make sense...

| __Original C Code__ | __Converted Rust Code__ |
| :--- | :--- |
| `lv_obj_t *screen_time_create(home_time_widget_t *ht) {` <br><br>&nbsp;&nbsp;`    //  Create a label for time (00:00)` <br>&nbsp;&nbsp;`    lv_obj_t *scr = lv_obj_create(NULL, NULL);` <br>&nbsp;&nbsp;`    lv_obj_t *label1 = lv_label_create(scr, NULL);` <br><br>&nbsp;&nbsp;`    lv_label_set_text(label1, "00:00");` <br>&nbsp;&nbsp;`    lv_obj_set_width(label1, 240);` <br>&nbsp;&nbsp;`    lv_obj_set_height(label1, 200);` <br>&nbsp;&nbsp;`    ht->lv_time = label1;` <br>&nbsp;&nbsp;`    ...` <br>&nbsp;&nbsp;`    return scr;` <br>`}` <br> | `fn create_widgets(widgets: &mut WatchFaceWidgets) -> ` <br>&nbsp;&nbsp;`    LvglResult<()> {` <br><br>&nbsp;&nbsp;`    //  Create a label for time (00:00)` <br>&nbsp;&nbsp;`    let scr = widgets.screen;` <br>&nbsp;&nbsp;`    let label1 = label::create(scr, ptr::null()) ? ;` <br><br>&nbsp;&nbsp;`    label::set_text(label1, strn!("00:00")) ? ;` <br>&nbsp;&nbsp;`    obj::set_width(label1, 240) ? ;` <br>&nbsp;&nbsp;`    obj::set_height(label1, 200) ? ;` <br>&nbsp;&nbsp;`    widgets.time_label = label1;` <br>&nbsp;&nbsp;`    ...` <br>&nbsp;&nbsp;`    Ok(())` <br>`}` <br> |
| _From [widgets/home_time/screen_time.c](https://github.com/bosmoment/PineTime-apps/blob/master/widgets/home_time/screen_time.c)_ | _From [rust/app/src/watch_face.rs](https://github.com/lupyuen/pinetime-rust-riot/blob/master/rust/app/src/watch_face.rs)_ |

`create`, `set_text`, `set_width` and `set_height` are Safe Wrapper functions, automatically generated by our `safe_wrap` macro.

For clarity, we have segregated the Safe Wrappers by module, hence we see module names like `obj::` and `label::`

Note that parameter type and return type have been changed...

| __Original C Code__ | __Converted Rust Code__ |
| :--- | :--- |
| `lv_obj_t *screen_time_create(home_time_widget_t *ht) {` <br>&nbsp;&nbsp;`    //  Create a label for time (00:00)` <br>&nbsp;&nbsp;`    lv_obj_t *scr = lv_obj_create(NULL, NULL);` | `fn create_widgets(widgets: &mut WatchFaceWidgets) -> ` <br>&nbsp;&nbsp;`    LvglResult<()> {` <br>&nbsp;&nbsp;`    //  Create a label for time (00:00)` <br>&nbsp;&nbsp;`    let scr = widgets.screen;`  |

We'll learn in a while why this was done: To make the code easier to maintain.

![Rust on RIOT](https://lupyuen.github.io/images/rust-on-riot-rust.png)

# Heapless Strings in Rust

Let's look at the C code for displaying the current time on PineTime Smart Watch. It calls [`snprintf`](http://www.cplusplus.com/reference/cstdio/snprintf/) to format the current time into a string buffer on the stack. Then it calls `lv_label_set_text` to set the text on the LittlebGL Label...

```c
/// In C: Populate the LVGL Time Label with the current time
static int set_time_label(home_time_widget_t *ht) {
    //  Create a string buffer on the stack with max size 6 to format the time
    char time[6];
    //  Format the time HH:MM into the string buffer
    int res = snprintf(
        time, 
        sizeof(time), 
        "%02u:%02u", 
        ht->time.hour,
        ht->time.minute
    );
    if (res != sizeof(time) - 1) {
        LOG_ERROR("[home_time]: error formatting time string %*s\n", res, time);
        return -1;  //  Return error to caller
    }
    //  Display the formatted time on the LVGL label
    lv_label_set_text(
        ht->lv_time, 
        time
    );
    return 0;  //  Return Ok
}
```
_From [widgets/home_time/screen_time.c](https://github.com/bosmoment/PineTime-apps/blob/master/widgets/home_time/screen_time.c)_

Here's the equivalent code in Rust...

```rust
/// In Rust: Populate the LVGL Time Label with the current time
fn set_time_label(widgets: &WatchFaceWidgets, state: &WatchFaceState) -> LvglResult<()> {  //  If error, return Err with error code inside
    //  Create a heapless string buffer on the stack with max size 6 to format the time
    type TimeBufSize = heapless::consts::U6;  //  Size of the string buffer
    let mut time_buf: heapless::String::<TimeBufSize> = 
        heapless::String::new();
    //  Format the time HH:MM into the string buffer
    write!(                 //  Macro writes a formatted string...
        &mut time_buf,      //  Into this buffer...
        "{:02}:{:02}\0",    //  With this format... (Must terminate Rust strings with null)
        state.time.hour,    //  With this hour value...
        state.time.minute   //  And this minute value
    ).expect("time fail");  //  Fail if the buffer is too small
    //  Display the formatted time on the LVGL label
    label::set_text(
        widgets.time_label, 
        &Strn::new( time_buf.as_bytes() )  //  Verifies that the string is null-terminated
    ) ? ;   //  If error, return Err to caller
    Ok(())  //  Return Ok
}
```
_Based on https://github.com/lupyuen/pinetime-rust-riot/blob/master/rust/app/src/watch_face.rs_

_Why do we use `heapless::String` instead of the usual `String` type in Rust?_

The usual [`String`](https://doc.rust-lang.org/std/string/struct.String.html) type in Rust uses Heap Memory... It allocates memory dynamically to store strings.  But we don't allow Heap Memory in our Rust program.

[`heapless::String`](https://docs.rs/heapless/0.5.3/heapless/struct.String.html) is a [__Heapless String__](https://docs.rs/heapless/0.5.3/heapless/struct.String.html) that doesn't use Heap Memory. It uses a fixed-size array stored on the stack (like above) or stored in Static Memory.

_Why can't we use Heap Memory?_

When writing embedded programs, it's good to budget in advance the memory needed to run the program and preallocate the memory needed from Static Memory. So that our program won't run out of Heap Memory while running and fail.  

[Heap Fragmentation](https://cpp4arduino.com/2018/11/06/what-is-heap-fragmentation.html) may also cause our programs to behave erratically.

_Why does the usual `String` type in Rust use Heap Memory?_

Using strings safely in C is hard... We have to watch the string size very carefully and make sure the strings don't overflow.

Rust makes string programming easier and safer... Rust `Strings` will grow dynamically when they run out of space! Unfortunately this means that Rust `Strings` need Heap Memory to make them grow. Which is a problem for embedded programs.

Here's how we allocate a Heapless String on the stack...

```rust
//  Create a heapless string buffer on the stack with max size 6 to format the time
type TimeBufSize = heapless::consts::U6;  //  Size of the string buffer
let mut time_buf: heapless::String::<TimeBufSize> = 
    heapless::String::new();
```

`let mut` works like `let`, except that it declares a mutable variable on the stack whose value may change.

```rust
//  Format the time HH:MM into the string buffer
write!(                 //  Macro writes a formatted string...
    &mut time_buf,      //  Into this buffer...
    "{:02}:{:02}\0",    //  With this format... (Must terminate Rust strings with null)
    state.time.hour,    //  With this hour value...
    state.time.minute   //  And this minute value
).expect("time fail");  //  Fail if the buffer is too small
```

`write!` is a Rust Macro that writes formatted strings into a string buffer. It's a macro, not a function, so that the paramaters are validated against the specified format at compile-time.

The Rust function `set_time_label` above looks OK. But there's a problem with this line of code...

```rust
//  Display the formatted time on the LVGL label
label::set_text(
    widgets.time_label, 
    &Strn::new( time_buf.as_bytes() )  //  Verifies that the string is null-terminated
) ? ;   //  If error, return Err to caller
```

_Do you see the problem?_

The problem becomes obvious when we learn in a while about the Lifetime of Rust variables.

![Rust on RIOT: C vs Rust](https://lupyuen.github.io/images/rust-on-riot-c.png)

# Lifetime of Rust Variables

In the last section we attempted to display the current time on PineTime Smart Watch inside a LVGL Widget (which we have imported from C).  We allocated a Heapless String on the stack...

```rust
//  In Rust: Create a heapless string buffer on the stack with max size 6 to format the time
type TimeBufSize = heapless::consts::U6;  //  Size of the string buffer
let mut time_buf: heapless::String::<TimeBufSize> = 
    heapless::String::new();
```

Then we formatted the current time into the Heapless String...

```rust
//  In Rust: Format the time HH:MM into the string buffer
write!(                 //  Macro writes a formatted string...
    &mut time_buf,      //  Into this buffer...
    "{:02}:{:02}\0",    //  With this format... (Must terminate Rust strings with null)
    state.time.hour,    //  With this hour value...
    state.time.minute   //  And this minute value
).expect("time fail");  //  Fail if the buffer is too small
```

And we passed the formatted time in the Heapless String to `set_text` to set the label text...

```rust
//  In Rust: Display the formatted time on the LVGL label
label::set_text(
    widgets.time_label, 
    &Strn::new( time_buf.as_bytes() )  //  Verifies that the string is null-terminated
) ? ;   //  If error, return Err to caller
```

`set_text` is a Safe Wrapper for the LVGL function `lv_label_set_text` that we have imported from C into Rust.

When we compile this code, the Rust Compiler draws a neat line diagram to point out a cryptic error...

```
error[E0597]: `time_buf` does not live long enough
  --> rust/app/src/watch_face.rs:25:52
   |
25 |     label::set_text(widgets.time_label, &Strn::new(time_buf.as_bytes())) ? ;
   |                                                    ^^^^^^^^-----------
   |                                                    |
   |                                                    borrowed value does not live long enough
   |                                                    argument requires that `time_buf` is borrowed for `'static`
26 |     Ok(())
27 | }
   | - `time_buf` dropped here while still borrowed
```

_Borrowed value does not live long enough... What is the meaning of this?_

Let's look at the declaration of the C function `lv_label_set_text` (from which `set_text` was derived)...

```c
//  In C: Declare function lv_label_set_text to set the text of a label
void lv_label_set_text(lv_obj_t *label, const char *text);
```

`lv_label_set_text` (same for `set_text`) sets the text on a LVGL `label` to a string `text` that's passed to the function.

_Question: What happens to the string in `text` AFTER the function `lv_label_set_text` returns?_

_What if `lv_label_set_text` has lazily copied the string pointer (instead of the string contents)?_

_Will some other LVGL function read the string pointer later?_

Well this will be a problem... If our string buffer was allocated on the stack!

Stack variables will magically disappear when we return from the function. If a LVGL function attempts to read the string buffer previously allocated on the stack... Strange things will happen!

But that's exactly what we did: Allocate the string buffer on the stack...

```rust
//  In Rust: Create a heapless string buffer on the stack with max size 6 to format the time
type TimeBufSize = heapless::consts::U6;  //  Size of the string buffer
let mut time_buf: heapless::String::<TimeBufSize> = 
    heapless::String::new();
```

So the Rust Compiler helpfully warns us that somebody could be using later the string buffer that we have passed to C. And that it's not safe to pass a string buffer on the stack. Let's reword it like this...

1. Our string buffer lives on the stack. It disappears when the function returns.

1. Thus our string buffer has a very short __Lifetime__... It's not meant to be used for a long time.

1. But we passed the string buffer to the C function `lv_label_set_text` (via the Safe Wrapper `set_text`)

1. The Rust Compiler doesn't know the expected Lifetime of the string buffer used by `lv_label_set_text`... The string buffer may still be used for a long time afterwards

1. Hence the Rust Compiler warns that the string buffer might not __live long enough__ to satisfy `lv_label_set_text`

The Rust Compiler is really that clever! These are typical bugs that we tend to miss in C... Passing values on the stack when we're not supposed to.  Which won't happen in Rust since the Lifetimes of variables will have to be stated clearly.

_FYI: The Lifetime of `lv_label_set_text` is stated verbally in the LVGL docs... `lv_label_set_text` will copy the contents of the string buffer, instead of copying the string pointer. Therefore the string buffer passed to `lv_label_set_text` is expected to have a short Lifetime._

There are two solutions to our Lifetime problem...

1. Tell the Rust Compiler the expected Lifetime of the string buffer in `lv_label_set_text`. (Using Lifetime specifiers like `'static`, which is kinda complicated for newbies)

1. Or make our string buffer live forever! When we turn our Stack Variable into a Static Variable, the string buffer outlives `lv_label_set_text`. And makes the Rust Compiler very happy!

We'll learn about Static Variables next...

![Safer Rust on RIOT](https://lupyuen.github.io/images/rust-on-riot-safer.png)

# Static Variables in Rust

Creating Static Variables in C is easy...

```c
/// In C: Populate the LVGL Time Label with the current time
static int set_time_label(home_time_widget_t *ht) {
    //  Create a string buffer in static memory with max size 6 to format the time
    static char time[6];
```
_Based on https://github.com/bosmoment/PineTime-apps/blob/master/widgets/home_time/screen_time.c_

Here we allocate a 6-byte string buffer in Static Memory to format the time for display on PineTime.

_What's the initial value of `time`?_

Static Memory (also known as BSS) is implicitly initialised with null bytes.  So `time` is initially set to 6 bytes of null.  Which also represents an empty string `""` in C (since C strings are terminated by null).

Let's do the same in Rust... Watch how Rust cares about our code safety.  Here's our original Rust code that allocates the string buffer on the stack...

```rust
/// In Rust (Stack Version): Populate the LVGL Time Label with the current time
fn set_time_label(widgets: &WatchFaceWidgets, state: &WatchFaceState) -> LvglResult<()> {  //  If error, return Err with error code inside
    //  Create a heapless string buffer on the stack with max size 6 to format the time
    type TimeBufSize = heapless::consts::U6;  //  Size of the string buffer
    let mut time_buf: heapless::String::<TimeBufSize> = 
        heapless::String::new();
```

And now we allocate the string buffer in Static Memory...

```rust
/// In Rust (Static Version): Populate the LVGL Time Label with the current time
fn set_time_label(widgets: &WatchFaceWidgets, state: &WatchFaceState) -> LvglResult<()> {  //  If error, return Err with error code inside
    //  Create a heapless string buffer in static memory with max size 6 to format the time
    type TimeBufSize = heapless::consts::U6;  //  Size of the string buffer
    static mut TIME_BUF: heapless::String::<TimeBufSize> = 
        heapless::String( heapless::i::String::new() );    
```

`let mut` has been changed to `static mut`.  This looks very similar to C, piece of cake!

Then comes the initialisation...

```rust
//  In Rust: Initialise the string buffer
static mut TIME_BUF: heapless::String::<TimeBufSize> = 
    heapless::String( heapless::i::String::new() );    
```

In Rust, __all Static Variables must be initialised explicitly__... Rust doesn't allow implicit initialisation like in C!

This prevents initialisation errors that we see in C _(phew!)_

Note that the initial value has been changed from `heapless::String::new()` to...

```rust
heapless::String( heapless::i::String::new() )
```

That's the proper way to initialise a Heapless String Static Variable, [according to the docs](https://docs.rs/heapless/0.5.3/heapless/struct.String.html).  (And if you think carefully, there's a very good reason why the value looks different)

Here's the entire function that creates a string buffer in Static Memory and uses the buffer...

```rust
/// In Rust (Static Version): Populate the LVGL Time Label with the current time
fn set_time_label(widgets: &WatchFaceWidgets, state: &WatchFaceState) -> LvglResult<()> {  //  If error, return Err with error code inside
    //  Create a heapless string buffer in static memory with max size 6 to format the time
    type TimeBufSize = heapless::consts::U6;  //  Size of the string buffer
    static mut TIME_BUF: heapless::String::<TimeBufSize> = 
        heapless::String( heapless::i::String::new() );    
    //  This code is unsafe because multiple threads may be updating the string buffer
    unsafe {
        TIME_BUF.clear();       //  Erase the string buffer
        //  Format the time HH:MM into the string buffer
        write!(                 //  Macro writes a formatted string...
            &mut TIME_BUF,      //  Into this buffer...
            "{:02}:{:02}\0",    //  With this format... (Must terminate Rust strings with null)
            state.time.hour,    //  With this hour value...
            state.time.minute   //  And this minute value
        ).expect("time fail");  //  Fail if the buffer is too small
        //  Display the formatted time on the LVGL label
        label::set_text(
            widgets.time_label, 
            &Strn::new( TIME_BUF.as_bytes() )  //  Verifies that the string is null-terminated
        ) ? ;  //  If error, return Err to caller
    }          //  End of unsafe code
    Ok(())     //  Return Ok
}
```
_From [rust/app/src/watch_face.rs](https://github.com/lupyuen/pinetime-rust-riot/blob/master/rust/app/src/watch_face.rs)_

_Why is the code marked `unsafe`?_

Unlike C, Rust is fully aware of multithreading... [Using multiple threads to run code simultaneously](https://doc.rust-lang.org/1.30.0/book/second-edition/ch16-01-threads.html).

When two threads read and write to the same Static Variable (like `TIME_BUF`), we will get inconsistent results (unless we do some locking).

Thus we need to flag the code as `unsafe` to say...

> Dear Rust Compiler: Thank you for warning us that Mutable Statics like `TIME_BUF` can be mutated by multiple threads and cause undefined behavior. We promise to take responsibility for any `unsafe` consequences. We hope you're happy now.

_But is this code really `unsafe`? Will we have multiple threads running the same code concurrently?_

Actually the code above will only be executed by a single thread... RIOT assures this on our PineTime Smart Watch.

The Rust Compiler doesn't know anything about RIOT. That's why we need to flag the code as `unsafe` and tell the compiler that it's really OK.

If there's a possibility that multiple threads will run the code, we will need to use the [Thread Synchronisation](https://riot-os.org/api/group__core__sync.html) functions provided by RIOT.

# Simplify Strings

In the code that we have seen, declaring and creating string buffers look cumbersome...

```rust
//  In Rust: Initialise the string buffer
static mut TIME_BUF: heapless::String::<TimeBufSize> = 
    heapless::String( heapless::i::String::new() );    
```

Let's simplify this! For a watch face we probably don't need to handle strings longer than 64 characters. Let's define our own `String` type limited to 64 characters...

```rust
/// Limit Strings to 64 chars (which may include multiple color codes like "#ffffff")
type String = heapless::String::<heapless::consts::U64>;
```
_From [rust/app/src/watch_face.rs](https://github.com/lupyuen/pinetime-rust-riot/blob/master/rust/app/src/watch_face.rs)_

Note that the Rust Standard Library already defines a `String` type that uses Heap Memory. But it should be OK for us to redefine the `String` type because...

1. We're using the Rust Core Library (`no_std`) instead of the Rust Standard Library. Rust Core Library doesn't define the `String` type.

1. Our `String` type works with the same functions in the standard `String` type, thanks to `heapless`

Let's add some helper functions...

```rust
/// Create a new String
const fn new_string() -> String {
    heapless::String(heapless::i::String::new())
}

/// Convert a static String to null-terminated Strn
fn to_strn(str: &'static String) -> Strn {
    Strn::new(str.as_bytes())
}
```
_From [rust/app/src/watch_face.rs](https://github.com/lupyuen/pinetime-rust-riot/blob/master/rust/app/src/watch_face.rs)_

Now our Rust code becomes a lot simpler...

```rust
/// Populate the Time and Date Labels with the time and date. Called by screen_time_update_screen() above.
pub fn set_time_label(widgets: &WatchFaceWidgets, state: &WatchFaceState) -> LvglResult<()> {
    //  Create a string buffer to format the time
    static mut TIME_BUF: String = new_string();
    //  Format the time and set the label
    unsafe {  //  Unsafe because TIME_BUF is a mutable static
        TIME_BUF.clear();
        write!(
            &mut TIME_BUF, 
            "{:02}:{:02}\0",  //  Must terminate Rust strings with null
            state.time.hour,
            state.time.minute
        ).expect("time fail");
        label::set_text(
            widgets.time_label, 
            &to_strn(&TIME_BUF)
        ) ? ;
    }
    Ok(())
}
```
_From [rust/app/src/watch_face.rs](https://github.com/lupyuen/pinetime-rust-riot/blob/master/rust/app/src/watch_face.rs)_

# VSCode Development and Debugging

The repository [`pinetime-rust-riot`](https://github.com/lupyuen/pinetime-rust-riot) has been configured to work with VSCode. Just open the VSCode Workspace [`workspace.code-workspace`](https://github.com/lupyuen/pinetime-rust-riot/blob/master/workspace.code-workspace)

The VSCode Workspace contains tasks for building and flashing RIOT to PineTime on macOS and Linux. Refer to the tasks `Build Application` and `Flash Application` in [`.vscode/tasks.json`](https://github.com/lupyuen/pinetime-rust-riot/blob/master/.vscode/tasks.json)

The VSCode Debugger (with the [Cortex Debug Extension](https://marketplace.visualstudio.com/items?itemName=marus25.cortex-debug)) has been configured to flash and debug RIOT on PineTime with [ST-Link v2](https://www.aliexpress.com/wholesale?catId=0&initiative_id=SB_20180924134644&SearchText=st-link+v2&switch_new_app=y). See the debugger configuration in [`.vscode/launch.json`](https://github.com/lupyuen/pinetime-rust-riot/blob/master/.vscode/launch.json)

[Video Demo of VSCode Debugger with RIOT on PineTime](https://youtu.be/U2okd7C8Q2A)

# RIOT Development on Windows

Building RIOT natively on Windows (without WSL and MinGW) can be difficult. We recommend building RIOT in the GitHub Cloud with GitHub Actions.  Then download the built binary `Pinetime.elf` for flashing and debugging with VSCode.

Check out the GitHub Actions Workflow for Rust on RIOT: [`.github/workflows/main.yml`](https://github.com/lupyuen/pinetime-rust-riot/blob/master/.github/workflows/main.yml)

![Rust on RIOT WebAssembly Simulator](https://lupyuen.github.io/images/rust-on-riot-simulator.png)

# WebAssembly Simulator

For quicker development, we have a __WebAssembly Simulator__ that will preview Rust on RIOT Watch Faces in a web browser...

[Online Demo of WebAssembly Simulator](https://lupyuen.github.io/pinetime-rust-riot/lvgl.html)

The repository [`pinetime-rust-riot`](https://github.com/lupyuen/pinetime-rust-riot) auto-builds the WebAssembly Simulator with GitHub Actions whenever source files are changed. 

To set up the WebAssembly Simulator for Rust on RIOT, see [`pinetime-rust-riot/README.md`](https://github.com/lupyuen/pinetime-rust-riot/blob/master/README.md)

The WebAssembly Simulator is built according to the following __GitHub Actions Workflow__...

[`.github/workflows/simulate.yml`](.github/workflows/simulate.yml)

[More about WebAssembly Simulator for Rust on RIOT](https://github.com/AppKaki/lvgl-wasm/blob/rust/README.md)

![Rust on RIOT Design](https://lupyuen.github.io/images/rust-on-riot-design.png)

# LVGL and RIOT Bindings for Rust

Today our Rust code uses a custom-generated Rust Safe Wrapper for the LVGL Libray. The wrapper supports a subset of the LVGL functions. In future we should migrate the LVGL wrapper to this wrapper that is properly maintained...

[`github.com/rafaelcaricio/lvgl-rs`](https://github.com/rafaelcaricio/lvgl-rs)

To call the RIOT API from Rust, we could use one of the following Rust Wrappers for RIOT...

[`gitlab.com/etonomy/riot-sys`](https://gitlab.com/etonomy/riot-sys)

[`gitlab.com/etonomy/riot-wrappers`](https://gitlab.com/etonomy/riot-wrappers)

![Druid with LVGL](https://lupyuen.github.io/images/druid-lvgl.png)

# Declarative User Interfaces with Rust and LVGL

Can we create Declarative User Interfaces in Rust and LVGL?

So instead of specifying precise row and column pixel positions of each widget... We just lay them out in a table like HTML?

We're extending the Druid UI library so that it renders with LVGL widgets...

[`github.com/AppKaki/druid-lvgl`](https://github.com/AppKaki/druid-lvgl/blob/master/README.md)

![Remote PineTime](https://lupyuen.github.io/images/remote-pinetime-arch.jpg)

# What's Next

How to test your firmware without a PineTime? We now have a Remote PineTime that you may flash your firmware remotely (by sending Telegram Bot commands) and watch the live video stream (on YouTube).

[Check out Remote PineTime](https://github.com/lupyuen/remote-pinetime-bot/blob/master/README.md)

In this article I have demonstrated that it's not that hard to convert C code to Rust... Even for coding watch faces on smart watches.

_Rust on RIOT has great potential to become the safer, modern replacement for Arduino!_

# References

For more about Rust on RIOT, check out the presentation at [RIOT Summit 2020](https://summit.riot-os.org/2020/)...

[Safer, Simpler Embedded Programs with Rust on RIOT](https://docs.google.com/presentation/d/1IgCsWJ5TYpPaHXZlaETlM2rYQrBmOpN2WeFsNjmYO_k/edit?usp=sharing)

[Video Presentation](https://youtu.be/rTxeXnlH-mM)

Check out my articles on PineTime and IoT...

[My Articles](https://lupyuen.github.io/)

[My RSS Feed](https://lupyuen.github.io/rss.xml)