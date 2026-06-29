## Fix: stop wiping the device registry on every options edit

`async_update_entry` removed **all** device-registry entries on each options change,
which nukes your area assignments and device-level customizations every time you
tweak anything. Now it reconciles: it only removes devices that no longer match a
configured sub-device and leaves the rest (and their settings) alone.
