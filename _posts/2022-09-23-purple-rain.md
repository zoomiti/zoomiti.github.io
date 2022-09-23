---
title: Embedding Rust WASM in Jekyll Posts
tags:
  - Rust
  - WASM
  - graphics
header:
  teaser: /assets/images/2022-09-23-purple-rain-teaser.png
toc: true
toc_sticky: true
custom_wasm:
  - purple_rain_bg.wasm
custom_preload_module:
  - purple_rain.js
---

So you're probably curious how this little animation was embedded into this
post. The journey from `winit` application to HTML canvas element is a long
one.

<div id="purple_rain_canvas"></div>
<script type="module">
import init from '/assets/js/purple_rain.js';init('/assets/wasm/purple_rain_bg.wasm');
</script>

## Why bother?
Honestly just for the novelty of it. I know I could've ported this to a small
JavaScript script on my own, but given `winit`'s commitment to support multiple
platforms it only seemed right to use `winit`'s inbuilt support for targeting 
`WASM`.

## Recommendations
This section includes personal recommendations that solved issues I had porting
my `winit` application to `WASM` but also made the development easier.

### Compatibility Crates
A lot of my issues come down to the Rust `std` library not being available on
the web. This should be surprising to no one but luckily a ton of crates exist
as a compatibility layer.

Just a few examples for reference:
  - [`getrandom`](https://crates.io/crates/getrandom) with the `js` feature provides access to the rand
  crate
  - [`Instant`](https://crates.io/crates/instant) provides a compatibility layer
  for `std::time` because it stops working when using wasm-bindgen or stdweb
    - Install it and replace all uses of `std::time` with `instant::Instant`.
	Don't worry, if you aren't building for `WASM` it'll alias to `std::time`
	seamlessly

#### Fix for `web-gpu`
If you are doing any GPU rendering or one of your dependencies offloads its
rendering to the GPU and it supports `web-gpu` you may encounter an issue 
where the canvas is properly set up but there is no rendering happening.

The fix is to make sure your `Cargo.toml` includes a `web-gpu` feature. Find
what version you need and add this line to your `Cargo.toml`.
```toml
[target.'cfg(target_family = "wasm")'.dependencies] # If you have other targets too.
wgpu = { version = "0.12.0", features = ["webgl"]}
```

### Trunk
I recommend installing `trunk` which is a cargo utility that allows for
automatic building and deployment to `WASM` for rust projects. This'll make
development and debugging a lot easier.

You can install it using:
```sh
cargo install trunk
```
#### No Trunk
Proceeding without trunk is possible but not recommended. You can still build
your project without it but you won't be able to test your program without
deploying it to a website manually. This blog post will assume you are using it
so anytime you see 
```sh
trunk build
```
you can use 
```sh
cargo build --target wasm32-unknown-unknown
```
unless you specify the `wasm32-unknown-unknown` target
in your `.cargo/config` file. If you specify the target manually `cargo build`
works just fine.

### `WASM` Module
This tutorial will include writing a lot of functions only needed when
compiling to `WASM`. This is simple to set up in Rust using the `#[cfg()]`
attribute macro.
```rust
#[cfg(target_arch)]
fn foo() {
}
```

This can get incredibly tedious to write for every function but luckily
you can put all the functions in a `mod wasm` like this:
```rust
#[cfg(target_arch)]
mod wasm {
    fn foo() {
    }
}
```
And now the whole module will only be compiled to the `WASM` target. This
tutorial assumes you have done this so any function calls to functions we
create include the `wasm::` namespace.

## Using `wasm-bindgen` to build the WASM executable
This section goes over how to set up your rust project to be compiled to wasm
for web.
### New Dependencies
When targeting web alone just add the following dependencies using `cargo
add [dependency]`. If you have multiple targets make sure to only include
these dependencies for the `WASM` target by putting them under a `cfg` tag
like this:
```toml 
[target.'cfg(target_family = "wasm")'.dependencies]
foo = "0.1.0"
```

What dependencies you need really depend on your use case but I recommend at
least adding the following:
  - [`console_log`](https://crates.io/crates/console_log) allows you to use `console_log()` as an equivalent to JS's `console.log()`
  - [`wasm-bindgen`](https://crates.io/crates/wasm-bindgen) gives you procedural macros to make the inter op between JS and `WASM` much simpler.
  - [`wasm-bindgen-futures`](https://crates.io/crates/wasm-bindgen-futures) allows you to use JS's event loop to handle async code.
  - [`web-sys`](https://crates.io/crates/web-sys) gives you access to the DOM of the webpage in order to manipulate it (and add the canvas that `winit` will draw to).

Lastly as a personal recommendation, a lot of the crates in use tend to change a lot and you'll find your head against the wall as you see dependency chains break themselves. When it comes to these crates using a `*` version may be advised to have cargo do the heavy lifting as far as finding the version of the crate that happens to work with all your other dependencies
### Refactoring for `WASM` for web

Your first question hopefully is,

> Okay okay, I installed these new dependencies but what am I actually using
> them for?

Well I'm glad you asked! 

Turns out that web assembly does not have a defined start function like you
would expect coming from the variety of languages that use a `main` function.
Luckily `wasm-bindgen` has an attribute macro that allows us to tag a 
function to start our `WASM` program.

It looks like:
```rust
#[wasm_bindgen(start)]
pub fn run() {
}
```
Now you're wondering,

> But wait? Couldn't I use that macro on my `main` function and call it a day?

Well yes! Of course you can!

But wait! You're not seeing why defining a new function is useful. See when we
compile for `WASM` we lose a lot of things like the ability to print to a
console, or even a panic handler. This is where defining a different entry
point can be useful to allow you to set these things up.

> Okay you've convinced me... But then how do we set those things up?

Well that's actually exactly where the `console_log` crate comes in. It defines
two functions to set both those things up.

To set up printing to a console:
```rust
console_log::init_with_level(log::Level::Debug).expect("error initializing logger");
```
And to set up a panic handler:
```rust
std::panic::set_hook(Box::new(console_error_panic_hook::hook));
```

Altogether our function looks like:
```rust
#[wasm_bindgen(start)]
pub fn run() {
    console_log::init_with_level(log::Level::Debug).expect("error initializing logger");
    std::panic::set_hook(Box::new(console_error_panic_hook::hook));

    #[allow(clippy::main_recursion)]
    super::main();
}
```
And we're done. If you aren't doing any asynchronous code you can head over to
[the next section](#deploying-to-github-pages-using-jekyll). Otherwise stick around for the next subsection.


#### Using `wasm-bindgen-futures` for async code

Hopefully you are using [`tokio`](https://tokio.rs/) for your async code as this
will make this section much easier. You most likely are using `tokio`'s attribute
macro for your `async main` function aren't you?

> You caught me! But why does that matter?

Well that macro under the hood sets up `tokio`'s async runtime, which won't be
available in `WASM`. This means we need to use JavaScript's async runtime 
leveraging `wasm-bindgen-futures` to do so.

This is not too difficult although it requires unraveling `tokio`'s macro. This
is because we'll want to use Rust's `#[cfg()]` attribute to set up the
appropriate runtime.

Under the hood setting `tokio`'s macro creates a new `main` function with the
structure:

```rust
fn main() {
    tokio::runtime::Builder::new_multi_thread()
        .enable_all()
        .enable_time()
        .build()
        .unwrap()
        .block_on(async { run().await });
}
```
where the body of what was in your `async main` function is put inside the async block you see in the `block_on()` function.
But before we move on let's see how `wasm-bindgen-futures` sets up their
runtime.

```rust
wasm_bindgen_futures::spawn_local(run());
```
Well that's easy enough.

Now once we've rewritten the `async main` function like this we can add our
`#[cfg()]` attributes and we're running async code again!

```rust
fn main() {
    #[cfg(target_arch = "wasm32")]
    {
        wasm_bindgen_futures::spawn_local(run());
    }
    #[cfg(not(target_arch = "wasm32"))]
    {
        tokio::runtime::Builder::new_multi_thread()
            .enable_all()
            .enable_time()
            .build()
            .unwrap()
            .block_on(run());
    }
}
```

## Deploying to GitHub Pages using Jekyll
The following tutorial really only applies to projects built with
[`winit`](https://crates.io/crates/winit). Feel free to continue the tutorial
if it applies to you or see if it helps with your application anyway. If it
does, let me know and I'll consider adding it your discoveries to future
editions of this blog post!

### Testing your `WASM` Project
At this point you've most likely tried learning more of the `trunk` utility I
recommended installing and encountered the `trunk serve` command.

If you haven't I'll fill you in. `trunk serve` builds your project and then
serves an `index.html` file in your cargo project root modified to include your
`WASM` executable and the JavaScript script used to start it.

> But I don't have an `index.html` file in my project root!

Then make one! Here's a good enough template.
```html
<!doctype html>
<html lang=en>
    <head>
        <meta charset=utf-8>
        <title>Super Descriptive Title!</title>
    </head>
    <body>
    </body>
</html>
```

> Okay I've added the `index.html` but I still don't see my app on the webpage!

Well that's because our application does not add itself to the page's DOM yet!
Luckily it's an easy fix using the `web-sys` crate to handle our DOM
manipulation.

```rust
pub fn create_canvas(window: &Window) {
    use winit::platform::web::WindowExtWebSys;
    let canvas = window.canvas();

    canvas.style().set_css_text(
        "background-color: crimson; margin: auto; width: 100%; aspect-ratio: 4 / 3;",
    );

    let window = web_sys::window().unwrap();
    let document = window.document().unwrap();
    let body = document.body().unwrap();
    body.append_child(&canvas).unwrap();
}
```
> Woah! Woah! Woah! What's all this?

Okay I know a lot of stuff is happening so let's break it down.

```rust
let canvas = window.canvas();
```
This takes our `winit` window and creates an HTML canvas that we can now use to
put on the website.

```rust
canvas.style().set_css_text(
    "background-color: crimson; margin: auto; width: 100%; aspect-ratio: 4 / 3;",
);
```

This lines sets the CSS of the canvas so that we can see where in the canvas
`winit` is not drawing to. This will give you a hint about how to make sure 
the canvas is the proper size.

```rust
let window = web_sys::window().unwrap();
let document = window.document().unwrap();
let body = document.body().unwrap();
body.append_child(&canvas).unwrap();
```
And finally this block of code is the actual DOM manipulation. What impresses
me is how similar this looks like to the equivalent JS but that's just an
artifact of `web-sys`'s diligence in their APIs.

Now that we've set up this function make sure to call it at some point after 
creating your `winit` window and before starting your event loop, like this
to ensure the call only happens when compiling to `WASM`.

```rust
#[cfg(target_arch = "wasm32")]
wasm::create_canvas(&window);
```

#### Adding Event Loop Logging for Debug Builds
This part is optional since it is not needed for the final setup on your Jekyll
page, but it helps for debugging your build should it have issues on `WASM`.

Now that you know the basics of DOM manipulation using `web-sys`, we'll quickly
go over how to add a log for events in your event loop to the web-page. This
makes it easier to debug your mobile website since access to the console is
less convenient on mobile.

Here's the code for creating the log list. Using the `#[cfg(debug_assertions)]`
tags we ensure that this code is only compiled for debug builds.
```rust
#[cfg(debug_assertions)]
pub fn create_log_list(window: &Window) -> web_sys::Element {
    create_canvas(window);

    let window = web_sys::window().unwrap();
    let document = window.document().unwrap();
    let body = document.body().unwrap();

    let log_header = document.create_element("h2").unwrap();
    log_header.set_text_content(Some("Event Log"));
    body.append_child(&log_header).unwrap();

    let log_list = document.create_element("ul").unwrap();
    body.append_child(&log_list).unwrap();
    log_list
}
```
Now in the code where we add our canvas we want to change it slightly to set up
the DOM correctly when using debug or release builds. This'll do fine:
```rust
#[cfg(all(target_arch = "wasm32", not(debug_assertions)))]
wasm::create_canvas(&window);
#[cfg(all(target_arch = "wasm32", debug_assertions))]
let log_list = wasm::create_log_list(&window);
```

Now let's setup the logging function:
```rust
#[cfg(debug_assertions)]
pub fn log_event(log_list: &web_sys::Element, event: &Event<()>) {
    log::debug!("{:?}", event);

    // Getting access to browser logs requires a lot of setup on mobile devices.
    // So we implement this basic logging system into the page to give developers an easy alternative.
    // As a bonus its also kind of handy on desktop.
    if let Event::WindowEvent { event, .. } = &event {
        let window = web_sys::window().unwrap();
        let document = window.document().unwrap();
        let log = document.create_element("li").unwrap();
        log.set_text_content(Some(&format!("{:?}", event)));
        log_list
            .insert_before(&log, log_list.first_child().as_ref())
            .unwrap();
    }
}
```

And add logging in the event loop:
```rust
#[cfg(all(target_arch = "wasm32", debug_assertions))]
wasm::log_event(&log_list, &event);
```

Feel free to `trunk serve` and see the ever expanding log!

### Building and Including your Assets
At this point in the tutorial I bet you weren't expecting me to ask you to
install one more tool but this is only important for making running your `WASM`
executable easier. Go ahead and type

```bash
cargo install wasm-bindgen-cli
```
This tool lets you generate all the files you need to host your `WASM` project
on your Jekyll page. Create a directory to put the files into in your project,
and run the following.
```bash
wasm-bindgen --no-typescript --out-dir out/ --target web target/wasm32-unknown-unknown/{release||debug}/{application}.wasm
```

Inside the `out` directory you should see:
```bash
$ ls out/
.
..
{application}_bg.wasm
{application}.js
```

Inside your Jekyll project you can put both files inside the `assets/js/`
folder or separate them into `assets/js/` and `assets/wasm/`. Just make sure to
remember what choice you made.

### Header Links
Well now that we have our files uploaded to our website we need[^1] to tell Jekyll
our blog post needs to link the two files we generated in the last section to
make them available.

[^1]: I'm not actually completely sure this step is necessary but I found these in
	the source for the HTML that `trunk` serves, so I assume it at least is worth
	including.

Hopefully your Jekyll configuration/theme allows you to add stuff to the page's
`<head>` tag. Assuming you do, make sure that in the `<head>` tag you include 
these two `<link>` tags.

```html
<link rel="modulepreload" src="/assets/js/{application}.js">
<link rel="preload" src="/assets/wasm/{application}_bg.wasm" as="fetch" type="application/wasm" crossorigin="">
```
### Scripts in Markdown
Now we need to add a script to start the `WASM` project. Luckily, if you were
unaware, you can embed any HTML tags straight into Markdown. So make sure you
add this anywhere in your blog post.

```html
<script type="module">
import init from '/assets/js/{application}.js';init('/assets/wasm/{application}_bg.wasm');
</script>
```

### Placing the Canvas in your Post
You probably immediately noticed that your HTML canvas was put right at the
bottom of your whole website!

Well fret not! This is technically by design! Our code from [this
section](#testing-your-wasm-project) manipulates the DOM and adds our canvas to
the very end of our `<body>` tag. Let's refactor it to add it 
exactly where we want it.

First start by adding a `<div>` with and id exactly where you want
the canvas to be placed in the website. Something like this:

```html
<div id="{application}_canvas"></div>
```

Next go to your `create_canvas` function and refactor it to something like
this:

```rust
pub fn create_canvas(window: &Window) {
    use winit::platform::web::WindowExtWebSys;
    let canvas = window.canvas();

    canvas.style().set_css_text(
        "background-color: crimson; margin: auto; width: 100%; aspect-ratio: 4 / 3;",
    );

    let window = web_sys::window().unwrap();
    let document = window.document().unwrap();
    let body = document.body().unwrap();
    let canvas_div = document
        .get_element_by_id("{application}_canvas")
        .unwrap_or(From::from(body)); //Get the div
    canvas_div.append_child(&canvas).unwrap();
}
```

The notable line of code is:
```rust
let canvas_div = document
    .get_element_by_id("{application}_canvas")
    .unwrap_or(From::from(body)); 
```

Here we are grabbing the `<div>` we put in our website and using the `<body>` tag
as a fallback, in case we are debugging using `trunk serve`, and placing the
canvas inside that.

And *Voila!* after all this effort you should have a `winit` application
running within your blog post!

## Wrapping Up
I hope this blog post helps anyone trying to do anything remotely like I did.
If this post did not manage to help you enough feel free to consult both [my
application repository](https://github.com/zoomiti/purple_rain) and [my blog
repository](https://github.com/zoomiti/zoomiti.github.io).

---
