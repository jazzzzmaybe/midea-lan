# Phase 2: HA Mode Mapping Utilities

## Goal

Add bidirectional mapping methods to convert between Home Assistant modes and device attributes.

## Background

- Phase 1 (PR #92) added `supported_*_modes()` properties that return lists of available modes
- Phase 2 adds methods to:
  - Convert HA modes → device attributes (for set operations)
  - Convert device attributes → HA modes (for state reporting)

## Reference

midea_ac_lan HA integration climate.py:

- `hvac_mode` property: maps device state to HA hvac_mode
- `set_hvac_mode()`: converts HA hvac_mode to device attributes
- `fan_mode` property: maps device fan_speed to HA fan mode string
- `set_fan_mode()`: converts HA fan mode to device fan_speed value
- `swing_mode` property: maps device swing states to HA swing mode string
- `set_swing_mode()`: converts HA swing mode to device swing attributes
- `preset_mode` property: maps device preset flags to HA preset string
- `set_preset_mode()`: converts HA preset to device preset flags

## Proposed API

### 1. HVAC Mode Mapping

```python
@property
def current_hvac_mode(self) -> str:
    """Return current HVAC mode as HA string.

    Maps device power + mode to HA hvac_mode:
    - power=off → "off"
    - power=on + mode=1 → "auto"
    - power=on + mode=2 → "cool"
    - power=on + mode=3 → "heat"
    - power=on + mode=4 → "dry"
    - power=on + mode=5 → "fan_only" (if supported)

    Returns:
        HA hvac_mode string: "off", "auto", "cool", "heat", "dry", "fan_only"
    """


def set_hvac_mode(self, hvac_mode: str) -> None:
    """Set HVAC mode from HA string.

    Converts HA hvac_mode to device attributes:
    - "off" → power=False
    - "auto" → power=True, mode=1
    - "cool" → power=True, mode=2
    - "heat" → power=True, mode=3
    - "dry" → power=True, mode=4
    - "fan_only" → power=True, mode=5

    Args:
        hvac_mode: HA hvac_mode string

    Raises:
        ValueError: if hvac_mode not in supported_hvac_modes
    """
```

### 2. Fan Mode Mapping

```python
@property
def current_fan_mode(self) -> str:
    """Return current fan mode as HA string.

    Maps device fan_speed to HA fan mode:
    - 0-19 → "silent"
    - 20-39 → "low"
    - 40-59 → "medium"
    - 60-79 → "high"
    - 80-100 → "auto"
    - 102 → "custom" (fixed speed mode)

    Returns:
        HA fan_mode string
    """


def set_fan_mode(self, fan_mode: str) -> None:
    """Set fan mode from HA string.

    Converts HA fan mode to device fan_speed:
    - "auto" → 102
    - "silent" → 20
    - "low" → 40
    - "medium" → 60
    - "high" → 80
    - "custom" → 102

    Args:
        fan_mode: HA fan_mode string

    Raises:
        ValueError: if fan_mode not in supported_fan_modes
    """
```

### 3. Swing Mode Mapping

```python
@property
def current_swing_mode(self) -> str:
    """Return current swing mode as HA string.

    Maps device swing flags to HA swing mode:
    - vertical=False, horizontal=False → "off"
    - vertical=True, horizontal=False → "vertical"
    - vertical=False, horizontal=True → "horizontal"
    - vertical=True, horizontal=True → "both"

    Returns:
        HA swing_mode string
    """


def set_swing_mode(self, swing_mode: str) -> None:
    """Set swing mode from HA string.

    Converts HA swing mode to device swing flags:
    - "off" → vertical=False, horizontal=False
    - "vertical" → vertical=True, horizontal=False
    - "horizontal" → vertical=False, horizontal=True
    - "both" → vertical=True, horizontal=True

    Args:
        swing_mode: HA swing_mode string

    Raises:
        ValueError: if swing_mode not in supported_swing_modes
    """
```

### 4. Preset Mode Mapping

```python
@property
def current_preset_mode(self) -> str:
    """Return current preset mode as HA string.

    Maps device preset flags to HA preset mode (priority order):
    - boost_mode=True → "boost"
    - eco_mode=True → "eco"
    - ieco=True → "ieco"
    - sleep_mode=True → "sleep"
    - comfort_mode=True → "comfort"
    - else → "none"

    Returns:
        HA preset_mode string
    """


def set_preset_mode(self, preset_mode: str) -> None:
    """Set preset mode from HA string.

    Converts HA preset mode to device preset flags.
    Turns off all other presets (mutually exclusive).

    - "none" → all presets off
    - "boost" → boost_mode=True, others off
    - "eco" → eco_mode=True, others off
    - "ieco" → ieco=True, others off
    - "sleep" → sleep_mode=True, others off
    - "comfort" → comfort_mode=True, others off

    Args:
        preset_mode: HA preset_mode string

    Raises:
        ValueError: if preset_mode not in supported_preset_modes
    """
```

## Implementation Notes

1. **Validation**: All setters should validate input against `supported_*_modes()` lists
2. **Error Handling**: Raise `ValueError` for unsupported modes
3. **Atomicity**: Each setter should call `set_attribute()` to trigger device update
4. **Preset Exclusivity**: Only one preset can be active at a time (handled by existing `set_attribute()` logic)
5. **Fan Speed Mapping**: Use midea_ac_lan's fan speed ranges for consistency

## Testing

For each mapping method, add tests covering:

- Valid mode conversions
- Invalid mode raises ValueError
- Boundary cases (e.g., fan_speed ranges)
- Preset mutual exclusivity

## Next Steps

After Phase 2 completion:

- Phase 3: Enhance customize options for HA-specific overrides
- Phase 4: Update documentation with mapping examples
