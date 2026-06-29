## Fix: allow temperature ranges beyond 16–30 °C

You couldn't set a max temp above 30 °C even if your AC supports it — the config
field's own bounds were the default 16–30 range. Added absolute 5–40 °C limits for
the min/max (and preset) temperature selectors, while keeping 16/30 as the prefilled
defaults.
