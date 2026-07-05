# remotemouse-wayland-bridge

Fixes remote-mouse / remote-input apps (RemoteMouse, and anything else built
on the X11 XTest extension) that connect fine but don't move the cursor when
running under a native Wayland session on KDE Plasma (KWin).

## The problem

Apps like RemoteMouse move the cursor using the X11 **XTest** extension
(`XTestFakeMotionEvent`/`XTestFakeButtonEvent`), historically the standard
way for a remote-control app to inject input.

Modern KWin (Plasma 6) treats XTest-injected input as untrusted by default:
it updates the position XWayland reports back to X11 clients (so tools like
`xdotool getmouselocation` show it "working"), but it does **not** forward
that motion into the real Wayland input pipeline that actually drives the
visible cursor. This is a deliberate security restriction — it stops an X11
client from silently taking control of a Wayland session — but it silently
breaks any app that relies on XTest for input injection.

Symptoms:
- The remote app connects fine.
- Non-input actions (e.g. a "sleep" button that goes over D-Bus) work.
- The cursor position changes at the X11 protocol level (visible to
  `xdotool getmouselocation`) but **the on-screen cursor never moves**.

## The fix

This bridge watches X11's **XInput2** event stream, filtered to only the
`Virtual core XTEST pointer` device (the device XTest-based input always
shows up as — never anything else, so this cannot create a feedback loop
with its own output), and replays every motion/click through a real
`/dev/uinput` virtual mouse device. Input injected via uinput is
indistinguishable from a real physical mouse at the kernel/libinput level,
so KWin moves the actual visible cursor.

Motion is replayed as **absolute** positioning (like a graphics tablet)
rather than relative deltas. XTest's reported position is already
pointer-accel-adjusted, so replaying it as relative motion would run it
through libinput's acceleration curve a second time, producing sluggish,
non-linear cursor movement. Absolute devices bypass acceleration entirely.

## Requirements

```
sudo apt-get install python3-xlib python3-evdev
```

Your user needs write access to `/dev/uinput`. Check with:

```
getfacl /dev/uinput
```

If your user isn't listed, add a udev rule granting your user/group access
(e.g. via the `input` group), then reload udev rules and reboot or replug.

## Install

```
git clone https://github.com/<you>/remotemouse-wayland-bridge.git ~/remotemouse-wayland-bridge
mkdir -p ~/.config/systemd/user
cp ~/remotemouse-wayland-bridge/remotemouse-cursor-bridge.service ~/.config/systemd/user/
systemctl --user daemon-reload
systemctl --user enable --now remotemouse-cursor-bridge.service
```

Check it's running:

```
systemctl --user status remotemouse-cursor-bridge.service
```

## Bonus fix: Wi-Fi power-saving lag

If input still feels laggy (periodic ~100-300ms stalls in bursts) even
after the cursor actually moves, the cause is very likely **Wi-Fi
power-save mode** batching radio transmissions — either on the PC
receiving the input, or on the phone/device sending it. This is unrelated
to the XTest/Wayland issue above and needs a separate fix.

### On the Linux PC

Check current state:

```
iwconfig <interface>   # e.g. wlp3s0 — look for "Power Management:on"
```

Turn it off immediately (until reboot/reconnect):

```
sudo iwconfig <interface> power off
```

Make it permanent via NetworkManager (survives reboots/reconnects):

```
nmcli -t -f NAME,DEVICE connection show --active   # find your connection name
sudo nmcli connection modify "<connection-name>" 802-11-wireless.powersave 2
```

`802-11-wireless.powersave` values: `0` = use driver default, `1` = ignore/no
change, `2` = disable (what you want), `3` = enable.

### On the phone (Android)

- Settings → Wi-Fi → Advanced → disable "Wi-Fi power saving mode" /
  "Wi-Fi optimization" (naming varies by manufacturer).
- Settings → Apps → (the remote-control app) → Battery → set to
  "Unrestricted" so Android doesn't throttle its background networking.

## Diagnosing further

If you want to confirm which side (PC vs phone) is causing stalls, capture
traffic on the app's port while triggering input and look for gaps with no
packets at all — that points to whichever side stopped transmitting:

```
sudo tcpdump -i any -tt "port <app-port>"
```

Gaps that show up **in the raw packets themselves** mean it's a
network/radio issue upstream of any software fix on the receiving machine.
