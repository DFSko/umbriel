# Game streaming

[Sunshine](https://github.com/LizardByte/Sunshine) streams a desktop to
[Moonlight](https://moonlight-stream.org) clients. Pointed at a
[virtual output](outputs.md#virtual-outputs), it streams a monitor that exists
only for the session, at the client's resolution and frame rate, while the
physical monitors keep their layout.

This setup needs Sunshine, `wlr-randr`, and `jq`.

## Session script

Sunshine runs a command before a stream starts and another after it ends.
Save this as `~/.config/sunshine/umbriel-output.sh` and make it executable:

```sh
#!/bin/sh
set -eu

name=sunshine

case "$1" in
    create)
        umbriel output-destroy "$name" >/dev/null 2>&1 || true
        umbriel output-create "$name" >/dev/null
        wlr-randr --output "$name" \
            --custom-mode "${SUNSHINE_CLIENT_WIDTH:-1920}x${SUNSHINE_CLIENT_HEIGHT:-1080}@${SUNSHINE_CLIENT_FPS:-60}Hz"
        ;;
    remove)
        # Give windows that the app's undo command is closing up to 5 seconds to
        # go away, so they do not flash onto a physical monitor.
        i=0
        while [ "$i" -lt 25 ]; do
            left="$(umbriel windows --json |
                jq --arg out "$name:" '[.[] | select(.workspace | startswith($out))] | length')"
            [ "$left" = 0 ] && break
            sleep 0.2
            i=$((i + 1))
        done
        umbriel output-destroy "$name" >/dev/null 2>&1 || true
        ;;
esac
```

Destroying the output moves any window still on it to another output, as when
a monitor is unplugged.

## Sunshine configuration

In `~/.config/sunshine/sunshine.conf`, capture through wlr-screencopy, select
the output by name, and run the script around every session:

```ini
capture = wlr
output_name = sunshine
global_prep_cmd = [{"do":"/home/user/.config/sunshine/umbriel-output.sh create","undo":"/home/user/.config/sunshine/umbriel-output.sh remove"}]
```

Replace `/home/user` with your home directory. Restart Sunshine after editing
`sunshine.conf` or `apps.json`; it reads both only at startup.

## Opening applications on the stream

New windows open on the focused output, which is usually a physical monitor.
Use a [window rule](window-rules.md) to send an application to the stream
instead. For Steam Big Picture:

```toml
[[window_rule]]
match.app_id = "^steam$"
match.title = "[Bb]ig [Pp]icture"
default_output = "sunshine"
default_fullscreen = true
```

When the output does not exist, `default_output` is ignored and the window
opens as usual. Rules also apply when a title arrives after the window maps, so
Big Picture lands on the stream even when Steam switches an existing window
into it. `default_fullscreen` covers the case where Steam, already running,
opens Big Picture without asking for fullscreen.

In Sunshine's `apps.json`, launch Big Picture and close it when the session
ends. Sunshine runs an app's undo command before the global one, so the script
waits for Big Picture to close before it destroys the output:

```json
{
    "name": "Steam Big Picture",
    "detached": ["steam steam://open/bigpicture"],
    "prep-cmd": [{"do": "", "undo": "steam steam://close/bigpicture"}],
    "image-path": "steam.png"
}
```

Closing Big Picture makes Steam map its main window again. A new window takes
focus by default, and Steam's may switch a physical monitor to its workspace.
To keep focus where it was, give that window `default_focused = false` only for
the length of a session: include a file that the script fills in `create` and
empties in `remove`, then reload with `umbriel msg config-reload`.
