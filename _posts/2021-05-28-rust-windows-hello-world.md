## Hello Windows with Rust

I've recently started to become interested in Rust amd games development. Mostly,
I like the idea of developing a game from scratch and learning how all of the
different software components fit together to make it work. I've predominantly
been using macOS as my daily driver for the past 15 years but am venturing back
into the world of Windows as I start my games development journey. I've also been
mostly writing software in Java and Python, with a little bit of C/C++ but have
been wanting to get my hands into a juicy Rust project. What better way than to
write a game from scratch in Rust.

In this article, I'm going to show how to display a window in Windows using the
[windows-rs](https://github.com/microsoft/windows-rs) crate. This is basically a
copy of Microsoft's [Your First Windows Program](https://docs.microsoft.com/en-us/windows/win32/learnwin32/your-first-windows-program).
Let's look at the code!

### Generate bindings

First thing's first, we need to create a new project to house our example:

```console
cargo new hello_windows
cd hello_windows
```

Next, we need to create a library project that will be used to generate our
bindings to the Windows API. We could do this in the same executable project but
using a seperate library will help speed up our compile times in larger projects.

```console
cargo new --lib windows_bindings
```

Include the `windows-rs` crate as a dependency and build dependency in our
library project's `Cargo.tmol`:

```toml
[dependencies]
windows = "0.10.0"

[build-dependencies]
windows = "0.10.0"
```

Rather than generate all of the possible bindings, we need to specify which of
the bindings we want to generate. We do this in `build.rs`, at the root of out
bindings crate, using the `windows::build!` macro:

```rust
fn main() {
    windows::build!(
        Windows::Win32::System::SystemServices::{HINSTANCE, GetModuleHandleW, PWSTR, LRESULT,},
        Windows::Win32::UI::WindowsAndMessaging::{WNDCLASSEXW, CS_HREDRAW, CS_VREDRAW, WS_OVERLAPPEDWINDOW,
            RegisterClassExW, DefWindowProcW, CreateWindowExW, HWND, CW_USEDEFAULT, ShowWindow, SW_SHOW, MSG,
            PeekMessageW, PM_REMOVE, TranslateMessage, DispatchMessageW, WM_CLOSE, WM_DESTROY, WM_QUIT,
            PostQuitMessage, UnregisterClassW,
        },
    );
}
```

Finally, we need to make sure to export the bindings in `lib.rs` using the `windows::include_bindings!`
macro:

```rust
windows::include_bindings!();
```

### Display window

With that done, we're ready to start using the bindings to open a native Windows
window. The first step is to include a dependency on the same version of the `windows-rs`
crate and on the bindings library crate in the top level `Cargo.toml`:

```toml
[dependencies]
windows = "0.10.0"
windows_bindings = { path = "windows_bindings" }
```

Now we're ready to display our window. I'm not going to step through the code
line-by-line; the original Microsoft documents give a good explanation of
what's going on. Our `main.rs` is as follows - note that we must remember
to include the bindings to bring them into scope:

```rust
use std::mem;
use std::ptr;
use windows_bindings::Windows::Win32::{System::SystemServices::*, UI::WindowsAndMessaging::*};

fn main() -> windows::Result<()> {
    let instance = unsafe { GetModuleHandleW(None) };
    debug_assert!(!instance.is_null());

    let class_name = PWSTR(b"HelloWindowsClass\0".as_ptr() as _);

    let wc = WNDCLASSEXW {
        cbSize: mem::size_of::<WNDCLASSEXW>() as u32,
        style: CS_HREDRAW | CS_VREDRAW,
        lpfnWndProc: Some(window_proc),
        hInstance: instance,
        lpszClassName: class_name,
        ..Default::default()
    };

    let _atom = unsafe { RegisterClassExW(&wc) };
    debug_assert!(atom != 0);

    let window = unsafe {
        CreateWindowExW(
            Default::default(),
            class_name,
            "Hello Windows",
            WS_OVERLAPPEDWINDOW,
            CW_USEDEFAULT,
            CW_USEDEFAULT,
            CW_USEDEFAULT,
            CW_USEDEFAULT,
            None,
            None,
            instance,
            ptr::null_mut(),
        )
    };
    debug_assert!(!window.is_null());

    unsafe { ShowWindow(&window, SW_SHOW) };

    let mut running = true;
    while running {
        let mut message = MSG::default();
        while unsafe { PeekMessageW(&mut message, None, 0, 0, PM_REMOVE) }.into() {
            match message.message {
                WM_CLOSE | WM_QUIT => running = false,
                _ => unsafe {
                    TranslateMessage(&message);
                    DispatchMessageW(&message);
                },
            }
        }
    }

    unsafe { UnregisterClassW(class_name, instance) };

    Ok(())
}

unsafe extern "system" fn window_proc(
    window: HWND,
    message: u32,
    w_param: WPARAM,
    l_param: LPARAM,
) -> LRESULT {
    match message {
        WM_DESTROY => {
            PostQuitMessage(0);
            LRESULT(0)
        }
        _ => DefWindowProcW(window, message, w_param, l_param),
    }
}
```

It is worth noting that we're using the [PeekMessageW](https://docs.microsoft.com/en-gb/windows/win32/api/winuser/nf-winuser-peekmessagew)
rather than the [GetMessageW](https://docs.microsoft.com/en-gb/windows/win32/api/winuser/nf-winuser-getmessagew)
function. These are similar functions, but `PeekMessage` gives us more control because
it does not wait for a message to be posted before returning.

### Comments on the development workflow

During development, it's helpful to use the [Microsoft docs](https://docs.microsoft.com/en-gb/)
website. However, we also need to translate the native C/C++ API's into their Rust
equivalent. Fortunately, the `windows-rs` crate's docs have a [tool](https://microsoft.github.io/windows-docs-rs/doc/bindings/Windows/)
to do just that. Simply search for the name of the function needed and the documentation
will return the relevant signatire and binding location to include.

### Conclusion

The `windows-rs` crate makes it trivial to interact with native Windows API's
from Rust. Hopefully this example mapping the original Microsoft example into
a Rust version has helped to show how easy it is to get started developing
native Windows applications in Rust.
