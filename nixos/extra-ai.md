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
  pulse.enable = true;
};
```

> `rtkit` enables real-time scheduling for audio processes. `pulse.enable` provides
> PulseAudio compatibility for applications that do not speak PipeWire natively.
> `jack.enable` is omitted unless you do pro audio work — it adds complexity without
> benefit for general use. `wireplumber.enable` is omitted because PipeWire enables
> WirePlumber automatically since NixOS 23.05.

---

## Firmware

### Redistributable Firmware

Enables firmware blobs that are redistributable but not fully open source.
Required for most WiFi cards, Bluetooth adapters, and some GPUs on bare metal.

```nix
hardware.enableRedistributableFirmware = true;
```

### All Firmware (includes non-redistributable)

Broader coverage — includes firmware that cannot be freely redistributed.
Use this if `enableRedistributableFirmware` alone leaves hardware non-functional.

```nix
hardware.enableAllFirmware = true;
```

> `enableAllFirmware` implies `enableRedistributableFirmware` — you do not need both.
> Requires `nixpkgs.config.allowUnfree = true` since some blobs have restrictive licenses.

### Linux Firmware Package Explicitly

Both options above pull from `pkgs.linux-firmware` internally. You do not need to add
it to `systemPackages` manually.

---

## CPU Microcode

Microcode updates patch CPU bugs and security vulnerabilities at runtime.
NixOS applies them early in the boot process via the bootloader.

### Intel

```nix
hardware.cpu.intel.updateMicrocode = true;
```

### AMD

```nix
hardware.cpu.amd.updateMicrocode = true;
```

> Only enable the one matching your CPU. Both require `nixpkgs.config.allowUnfree = true`
> as the microcode blobs are proprietary. `nixos-generate-config` sets the correct one
> automatically based on detected hardware.

---

## GPU Drivers

### AMD

AMD GPUs are well-supported by the open source `amdgpu` kernel driver.
No proprietary driver is needed.

```nix
hardware.graphics = {
  enable = true;
  enable32Bit = true;   # required for 32-bit applications e.g. Steam, Wine
};

boot.initrd.kernelModules = [ "amdgpu" ];

services.xserver.videoDrivers = [ "amdgpu" ];
```

> `enable32Bit` has no effect without `nixpkgs.config.allowUnfree` if you are using
> Steam or Wine, since those packages are unfree. The `amdgpu` initrd module ensures
> the GPU is available early — useful for AMDGPU-based display output before X/Wayland.

### Intel (integrated graphics)

```nix
hardware.graphics = {
  enable = true;
  enable32Bit = true;
};

services.xserver.videoDrivers = [ "modesetting" ];

hardware.intel-gpu-tools.enable = true;   # optional: intel_gpu_top and diagnostic tools
```

> Intel integrated graphics use the `modesetting` driver, which is the current standard.
> The older `intel` Xorg driver is deprecated and should not be used.

### NVIDIA

NVIDIA on NixOS requires explicit configuration and unfree packages.

```nix
nixpkgs.config.allowUnfree = true;

hardware.graphics = {
  enable = true;
  enable32Bit = true;
};

services.xserver.videoDrivers = [ "nvidia" ];

hardware.nvidia = {
    modesetting.enable = true;       # required for Wayland
    powerManagement.enable = false;  # set true if on a laptop with suspend/resume issues
    open = false;                    # set true to use NVIDIA's open source kernel module
                                     # only recommended for Turing (RTX 20xx) and newer
    nvidiaSettings = true;           # installs the nvidia-settings GUI
    package = config.boot.kernelPackages.nvidiaPackages.stable;
};
```

> NVIDIA on Wayland requires `modesetting.enable = true`. Without it, only Xorg works.
> The `open` kernel module is NVIDIA's own open source variant — distinct from the
> community nouveau driver. It is stable on Turing and newer but still experimental
> on older hardware. Do not confuse it with nouveau.

#### NVIDIA + Optimus (laptop hybrid Intel/NVIDIA)

```nix
hardware.nvidia.prime = {
  offload = {
    enable = true;
    enableOffloadCmd = true;   # provides nvidia-offload helper command
  };
  intelBusId = "PCI:0:2:0";   # verify with: lspci | grep VGA
  nvidiaBusId = "PCI:1:0:0";
};
```

> Bus IDs must be verified on your specific hardware with `lspci`. The values above
> are examples only.

---

## Bluetooth

```nix
hardware.bluetooth = {
  enable = true;
  powerOnBoot = true;   # powers the adapter on at startup
};

services.blueman.enable = true;   # GUI bluetooth manager, optional
```

> `blueman` provides a system tray applet and pairing GUI. Omit it if you intend to
> manage bluetooth via a desktop environment that has its own bluetooth integration
> (e.g. GNOME, KDE).

---

## Unfree Packages

Enable proprietary software support in Nixpkgs (disabled by default for licensing reasons).

### Allow all unfree

```nix
nixpkgs.config.allowUnfree = true;
```

> Required for: NVIDIA drivers, CPU microcode, Steam, Chrome, VS Code, and any other
> package with a restrictive or proprietary license. Without this, such packages are
> blocked at evaluation time.

### Allow specific unfree packages only (recommended for controlled setups)

```nix
nixpkgs.config.allowUnfreePredicate = pkg:
  builtins.elem (pkgs.lib.getName pkg) [
    "steam"
    "nvidia-x11"
    "nvidia-settings"
  ];
```

> Limits unfree access to explicitly named packages instead of enabling all unfree
> software system-wide. More auditable than the blanket option.

---

## Shells

NixOS requires both installing the shell package and enabling its system module.
The module handles environment variable injection, `/etc` integration, and completion setup.
Without `programs.<shell>.enable`, the shell works but NixOS environment variables
may not be sourced correctly.

### Zsh

```nix
programs.zsh = {
  enable = true;
  autosuggestions.enable = true;   # optional: fish-like inline suggestions
  syntaxHighlighting.enable = true;
};

users.users.yustinia = {
  shell = pkgs.zsh;
};
```

### Fish

```nix
programs.fish.enable = true;

users.users.yustinia = {
  shell = pkgs.fish;
};
```

> Fish does not need to be added to `environment.systemPackages` separately.
> `programs.fish.enable` handles installation and system integration together.

### Nushell

```nix
programs.nushell.enable = true;

users.users.yustinia = {
  shell = pkgs.nushell;
};
```

> Nushell is a structured data shell — pipelines operate on typed objects rather than
> raw text. Steeper learning curve than Fish or Zsh, but worth knowing exists.

---

## Virtualisation

### Docker

```nix
virtualisation.docker = {
  enable = true;
  autoPrune = {
    enable = true;     # automatically runs docker system prune on a schedule
    dates = "weekly";
  };
};

users.users.yustinia = {
  extraGroups = [ "wheel" "networkmanager" "docker" ];
};
```

> Adding your user to the `docker` group allows running `docker` without `sudo`.
> Note that membership in the `docker` group is effectively equivalent to root access
> — anyone in it can escape to root via a container. Accept that tradeoff consciously.

### Podman (rootless alternative to Docker)

```nix
virtualisation.podman = {
  enable = true;
  dockerCompat = true;   # provides a docker-compatible CLI alias
  defaultNetwork.settings.dns_enabled = true;
};
```

> Podman runs containers rootlessly by default, which is the more secure posture.
> `dockerCompat = true` lets existing `docker` commands work against Podman without
> changes. Docker and Podman should not both have `dockerCompat` enabled simultaneously.

### libvirt (QEMU/KVM virtual machines)

```nix
virtualisation.libvirtd = {
  enable = true;
  qemu = {
    package = pkgs.qemu_kvm;
    runAsRoot = false;
    swtpm.enable = true;     # virtual TPM — required for Windows 11 guests
    ovmf = {
      enable = true;         # UEFI firmware for VMs
      packages = [ pkgs.OVMFFull.fd ];
    };
  };
};

programs.virt-manager.enable = true;   # GUI for managing VMs

users.users.yustinia = {
  extraGroups = [ "wheel" "networkmanager" "libvirtd" ];
};
```

> `runAsRoot = false` runs QEMU processes as your user rather than root — more secure.
> `swtpm` and `ovmf` are needed for Windows 11 guests specifically; omit if not needed.
> `virt-manager` is the standard GUI frontend for libvirt.
