# Winit 0.31 Migration Notes

Winit 0.31 upgrade completed in February 2026. Started with 95 compiler errors, ended with a clean build and all 264 tests passing.

## The Big Picture

Winit 0.31 removed the typed user event system (`EventLoop<T>`) and replaced it with a simpler wake-up model. The biggest change was refactoring event delivery from background threads (PTY, IPC, config monitor, etc) to the main event loop.

## Event System Rewrite

### What Changed

Old way: winit let you parameterize `EventLoop<Event>` with your custom event type and call `proxy.send_event(my_event)` from any thread. Your `ApplicationHandler::user_event()` callback would receive it.

New way: `EventLoop` has no generic parameter. You create your own `mpsc::channel`, send data through that, then call `proxy.wake_up()` to notify the event loop. Your `ApplicationHandler::proxy_wake_up()` callback drains the channel with `try_iter()`.

### Why It Matters

Order is critical: always `sender.send(event)` before `proxy.wake_up()`. If wake up happens first, the callback might check the channel, find nothing, and return before the event arrives.

Alacritty has 7+ background threads sending events (PTY reader, IPC socket, config file watcher, logger, etc). The implementation creates one mpsc channel in `main.rs` and clones the sender to each thread.

### Scheduler Special Case

The scheduler only runs on the main thread (updated from `about_to_wait()`), so it returns a `Vec<Event>` directly instead of going through the channel. This avoids the overhead of send/receive for timer events.

## Window and ActiveEventLoop Are Traits Now

Everything that was a concrete type is now `Box<dyn Window>` or `&dyn ActiveEventLoop`. Not a huge deal, just means adding `dyn` everywhere and storing windows as trait objects.

Also, window creation moved to the new `can_create_surfaces()` callback instead of `new_events()`. This is for Android compatibility - the old way conflated surface creation with app lifecycle events.

## Pointer Events

Winit unified mouse and touch into a single "pointer" abstraction. The old `WindowEvent::Touch` is gone - touch is now delivered through `PointerMoved` and `PointerButton` events with a `PointerSource::Touch` discriminant.

Touch handling code was updated to extract touch info from pointer events:

```rust
WindowEvent::PointerButton { button, state, position, .. } => {
    match button {
        ButtonSource::Touch { finger_id, .. } => {
            let touch = TouchPoint { id: finger_id, location: position };
            // handle touch
        },
        _ => {}
    }
}
```

The `primary` flag tells you if it's the first touch/click or a secondary one (multi-touch).

## Renamed Everything

- `inner_size()` → `surface_size()`
- `Resized` → `SurfaceResized`
- `CursorMoved` → `PointerMoved`
- `MouseInput` → `PointerButton`
- `DroppedFile(path)` → `DragDropped { paths, .. }` (all files at once now)
- `CursorIcon` moved from `winit::window` to `winit::cursor`
- `Fullscreen` moved to `winit::monitor`

KeyEvent also changed - `text_with_all_modifiers()` and `key_without_modifiers()` were methods on an extension trait, now they're just public fields on the struct.

## The Hyper Key Problem

The Kitty keyboard protocol defines separate keycodes for Super (57444), Hyper (57445), and Meta (57446), plus modifier bits for each. Winit deprecated `NamedKey::Hyper` following the W3C spec, which marks Hyper/Super as "legacy" keys that don't exist on modern keyboards.

Current support status:

**Works**: Pressing a Hyper key (Linux with custom XKB config only) sends Kitty keycodes 57445/57451 correctly.

**Doesn't work**: The Hyper modifier bit (16) doesn't get set for key combinations like Hyper+A. This is because winit's `ModifiersState` only has 4 modifiers (Shift, Control, Alt, Meta) and never queries XKB for Hyper state.

The implementation uses `#[allow(deprecated)]` on the two match arms that handle Hyper keycodes. The limitation is documented in code comments.

### Why Not Fix It Properly?

Three approaches were considered:

1. **Self-track Hyper**: Watch for Hyper press/release events and maintain our own boolean. Would let us set the modifier bit. About 50 lines of code, but fragile (misses focus changes, sticky modifiers, etc).

2. **Fork winit**: Add HYPER bit to ModifiersState, make the XKB backend query it properly. About 100 lines across multiple files, plus ongoing maintenance burden.

3. **Switch keyboard libraries**: Use something terminal-specific instead of winit's keyboard-types. Turns out there's no good alternative - crossterm, terminput, and termwiz all define their own types, and even WezTerm has an explicit TODO comment for Hyper/Meta modifier support.

The root issue is that no windowing abstraction exposes Hyper modifier state from XKB. Browsers merge everything into "Meta" (following W3C conventions), but terminal protocols need finer distinction. This gap exists across the Rust terminal ecosystem.

Decision: keep partial support because:
- Hyper key events (the main use case) work fine
- About 0.01% of users have Hyper configured in XKB
- WezTerm has the same limitation despite owning their keyboard types
- Forking for such a niche feature doesn't justify the maintenance cost

If you need Hyper modifier support, you're probably using Emacs with the kkp package on a custom XKB layout. This will work for Hyper key presses, but Hyper+letter combinations won't include the modifier bit in the escape sequence. The practical impact is small since most Emacs users bind Hyper as a key, not as a modifier for other keys.

## IME API Changes

The old methods (`set_ime_allowed`, `set_ime_purpose`, `set_ime_cursor_area`) were deprecated in favor of a unified `request_ime_update()` method. Migration to the new API uses `ImeEnableRequest` and `ImeRequestData` structs.

The new API is more flexible - declare what IME capabilities are supported (cursor area, surrounding text, hint/purpose) and bundle the initial state into a request. Updates go through the same path.

## Testing Notes

All tests pass. Test macros using `WindowEvent::MouseInput` were updated to use the new `PointerButton` events with proper `ButtonSource::Mouse()` wrapping.

The `DeviceId::dummy()` and `WindowId::dummy()` test helpers no longer exist. Use `None` for device ID and `WindowId::from_raw(0)` for window ID in tests.

## Files Changed

22 files total, +667/-426 lines. The big ones:
- `event.rs` (353 lines) - EventProxy rewrite, WindowEvent handlers
- `input/mod.rs` (131 lines) - Touch migration
- `display/window.rs` (105 lines) - Box<dyn Window>, IME migration
- `config/bindings.rs` (90 lines) - Super→Meta renames

Everything else was mostly mechanical renames and import updates.

## Build Commands

Standard workflow:
```bash
cargo check -p alacritty
cargo test --workspace --all-targets --all-features
cargo clippy --workspace --all-targets --all-features
cargo +nightly fmt --all
```

No special flags needed. Everything just works.
