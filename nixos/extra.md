# Extras

Optional configurations not included in the base install. Add to `default.nix` unless noted.

---

## Audio — PipeWire

```nix
security.rtkit.enable = true;

services.pipewire = {
  enable = true;
  alsa.enable = true;
  alsa.support32Bit = true;
  jack.enable = true;
  pulse.enable = true;
  wireplumber.enable = true;
};
```

> `rtkit` enables real-time scheduling for audio processes. `pulse.enable` provides PulseAudio compatibility for applications that do not speak PipeWire natively.

---

## Unfree Packages (default.nix)

Enable proprietary software support in Nixpkgs (disabled by default for licensing policy reasons).

```nix
nixpkgs.config.allowUnfree = true;
```

> Allows installation of packages with restrictive or proprietary licenses (e.g. NVIDIA drivers, Steam, Chrome). Without this, such packages are blocked at evaluation time.

---

### Optional stricter alternative (recommended for controlled setups)

```nix
nixpkgs.config.allowUnfreePredicate = pkg:
  builtins.elem (pkgs.lib.getName pkg) [
    "steam"
    "nvidia-x11"
  ];
```

> Limits unfree access to explicitly approved packages instead of enabling all unfree software system-wide.

---

### Placement rule

- `nixpkgs.config.allowUnfree = true;` is a top-level NixOS option
- It is evaluated during package resolution for the entire system
