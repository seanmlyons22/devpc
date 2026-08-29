# devpc

Ansible playbook for setting up a development machine. Runs against `localhost`
and supports both Linux (Fedora/Arch/Ubuntu) and macOS.

## What it installs

| | Linux | macOS |
|---|---|---|
| Packages via | distro package manager | Homebrew |
| Shell | zsh + Oh My Zsh + plugins | same (zsh is already the default shell) |
| CLI tools | ripgrep, eza, procs, dust, fd, bat, fzf, diff-so-fancy, htop, stow, neofetch | same, but `fastfetch` instead of the archived `neofetch`, and no `unzip` (macOS ships one) |
| Rust | rustup, Cortex-M targets, `llvm-tools-preview`, cargo-binutils, cargo-generate | same |
| Docker | Docker Engine via `geerlingguy.docker` | Colima + Docker CLI, or Docker Desktop (see below) |
| Editor | Neovim + the LazyVim starter config | same, plus lazygit and a Nerd Font |
| System settings | - | Dock, Finder, trackpad, text input, menu bar (see below) |

## Installing Ansible

### macOS

macOS ships Python 3.9, which is older than modern `ansible-core` supports, so
install Ansible from Homebrew rather than the system Python.

1. Install the Xcode Command Line Tools (needed by Homebrew and by Rust):

   ```bash
   xcode-select --install
   ```

2. Install Homebrew:

   ```bash
   /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
   ```

   On Apple silicon it lands in `/opt/homebrew`, which is not on `PATH` by
   default. Follow the "Next steps" the installer prints, or run:

   ```bash
   eval "$(/opt/homebrew/bin/brew shellenv)"
   ```

3. Install Ansible:

   ```bash
   brew install ansible
   ```

The playbook verifies steps 1 and 2 and fails with instructions if either is
missing - it deliberately does not install Homebrew for you, since you already
need Homebrew to get this far.

### Linux

```bash
sudo dnf install ansible        # Fedora
sudo pacman -S ansible          # Arch
sudo apt install ansible        # Debian/Ubuntu
```

## Running it

Install the external role and collection dependencies first:

```bash
ansible-galaxy install -r requirements.yml
```

Then run the playbook.

On Linux, packages are installed with `sudo`, so pass `-K` to be prompted for
your password:

```bash
ansible-playbook -K devpc.yml
```

On macOS nothing runs under `sudo` - Homebrew refuses to run as root - so drop
the `-K`:

```bash
ansible-playbook devpc.yml
```

### Docker on macOS

macOS has no Docker Engine; containers run inside a Linux VM. The default here
is **Colima** - the Docker CLI plus a Lima VM. It is the closest analogue to
what `geerlingguy.docker` installs on Linux: no GUI, no licence strings, and it
installs entirely in user space, which is why the macOS run needs no password
at all. Start it after the playbook finishes:

```bash
colima start
```

No flags needed - the role writes a tuned instance template to
`~/.colima/_templates/default.yaml`, which `colima start` picks up when it
creates the VM.

#### Colima tuning on Apple silicon

Three of these differ from what Colima writes into its *own* default template,
and they are the ones that matter on an M-series Mac:

| Setting | Colima's default | Ours | Why |
|---|---|---|---|
| `vmType` | `qemu` | `vz` | Apple's Virtualization.framework instead of emulating hardware |
| `mountType` | `sshfs` | `virtiofs` | Fastest mount driver; bind-mounted source trees are the usual bottleneck |
| `rosetta` | `false` | `true` | amd64 images run through Rosetta's binary translation instead of qemu's interpreter |

Sizing is derived from host facts at run time, so it is right on whatever
machine you run it on:

| Setting | Value | Override |
|---|---|---|
| Memory | 50% of host RAM | `-e colima_memory_percent=75` |
| vCPUs | half the logical cores, floor, min 2 | `-e colima_cpu=8` |
| Disk | 100 GiB, sparse | `-e colima_disk_gib=200` |

Memory is a ceiling, not a reservation, but the guest cannot exceed it - half
leaves the host its own half. All the vCPUs are worth *not* allocating: under
`vz` they are host threads competing with everything else, so a heavy
`docker build` given every core fights your UI rather than finishing sooner.

Two more are off by default and worth knowing about:

* `colima_mount_inotify` - propagates inotify events into the VM so
  file-watchers and hot-reload work on bind mounts. Still experimental.
* `colima_nested_virtualization` - M3 or newer; only useful for running VMs
  inside containers.

`rosetta: true` needs **Rosetta 2 actually installed on the host**. Colima only
logs a warning if it is missing and starts anyway, so amd64 images quietly fall
back to qemu emulation. The role tests this properly - by running an x86_64
binary, since `/usr/libexec/rosetta/oahd` exists even when Rosetta is *not*
installed - and tells you if it is missing. Installing it needs your password
and a licence agreement, so the playbook cannot:

```bash
softwareupdate --install-rosetta
```

Then `colima delete && colima start` to pick it up.

The template also declares `mounts` explicitly. This is not cosmetic: leaving
the key out writes `mounts: null` into the instance config, which is *not* the
same as Colima's documented `[]` default and results in nothing being mounted -
`docker run -v $PWD:/x` then hands the container an empty directory with no
error. `$HOME` is mounted writable.

**`vmType`, `mountType` and `arch` are fixed when the VM is created.** The
template only affects VMs created after it is written, so changing them on an
existing VM means `colima delete` and `colima start` again. The role checks for
an existing VM and tells you if that applies.

To install Docker Desktop instead:

```bash
sudo -v && ansible-playbook devpc.yml -e docker_macos_flavor=desktop
```

The `sudo -v` matters: the `docker-desktop` cask symlinks into `/usr/local/bin`
with `sudo: if_needed`, and Homebrew's password prompt has no terminal to
appear on when it runs under Ansible. Priming the sudo timestamp first avoids a
hang. Note that Docker Desktop is a commercial product - a paid subscription is
required at larger companies.

### Neovim / LazyVim

The `lazyvim` role installs Neovim and clones the
[LazyVim starter](https://github.com/LazyVim/starter) into `~/.config/nvim`,
then deletes its `.git` so you can track the config yourself. Run `nvim` once
afterwards to let lazy.nvim bootstrap the plugins, then `:LazyHealth`.

If a Neovim config is already there, it is moved to
`~/.config/nvim.bak-<epoch>` (along with `~/.local/share/nvim`,
`~/.local/state/nvim` and `~/.cache/nvim`). To abort instead of backing up:

```bash
ansible-playbook devpc.yml -e lazyvim_backup_existing_config=false
```

LazyVim needs **Neovim >= 0.11.2**; the role checks and fails with a clear
message if your distro package is older, which it will be on Ubuntu LTS.

Two dependencies are handled differently per platform:

* **lazygit** - installed from Homebrew on macOS. Left out of the Linux list on
  purpose: it is not in Fedora's official repos (it needs a COPR) and is absent
  from older Ubuntu, so including it would break the run. Add it to
  `lazyvim_linux_packages` if your distro has it.
* **tree-sitter-cli** - a Homebrew formula on macOS, but not packaged on Linux,
  so there it is built with `cargo`. This is why `lazyvim` runs after `rust` in
  the playbook. Disable with `-e lazyvim_install_tree_sitter_cli=false`.

A Nerd Font (JetBrains Mono) is installed as a cask on macOS. On Linux you will
need to install one yourself for LazyVim's icons to render.

## macOS system settings

The `macos_defaults` role applies System Settings preferences with
`defaults write`, so a rebuilt machine does not have to be clicked through by
hand. It runs on macOS only.

Every entry in `roles/macos_defaults/defaults/main.yml` was **read off a real
machine** with `defaults read`, not copied from a generic list. Keys that read
back as unset are deliberately not listed - unset means "macOS stock default",
and writing the stock value explicitly just adds noise.

One honest caveat about what that list represents: a key being present in a
plist does not prove you set it deliberately, because macOS writes some of its
own defaults out too. Stock values are not introspectable, so the list is *what
your plists explicitly contain*, not *what you changed*. Prune anything you do
not actually care about.

The `type` of each entry must match what `defaults read-type` reports. Get it
wrong and the module rewrites the key on every run, and the role silently stops
being idempotent. Note `Clicking` (tap-to-click) is an **integer**, not a bool,
while its neighbours in the same domain are bools.

Both trackpad domains are set on purpose:
`com.apple.driver.AppleBluetoothMultitouch.trackpad` covers external Magic
Trackpads and `com.apple.AppleMultitouchTrackpad` the built-in one. macOS keeps
them separate, so setting only one leaves the other device unconfigured.

Apps are restarted only when their own settings actually changed - a no-op run
does not bounce your Dock and Finder. Trackpad and keyboard settings are read
at login, so those need a logout; the role says so when it changes one.

Two things are opt-in because they need sudo or are disruptive:

```bash
ansible-playbook devpc.yml -e macos_set_computer_name=true   # needs -K
ansible-playbook devpc.yml -e macos_defaults_apply_locale=false
```

### What this cannot capture

`defaults` covers preferences, not everything in System Settings. Still manual:
Dock contents, wallpaper, display arrangement and resolution, login items,
keyboard shortcuts (`com.apple.symbolichotkeys` is writable but its key format
is impractical), energy settings (`pmset`, needs sudo), FileVault, and the
firewall.

## The role does not write a `.zshrc`

Deliberately. It clones `~/.oh-my-zsh` and the three plugins, but never creates
`~/.zshrc` - so a freshly provisioned machine has Oh My Zsh on disk and *not*
active in your shell until you supply one.

This is what `stow` is installed for: bring your own dotfiles repo and stow it.
An Ansible-managed `.zshrc` would occupy the path and make `stow` refuse to
link yours, since it will not overwrite an existing file.

If you want one to start from, Oh My Zsh ships a template:

```bash
cp ~/.oh-my-zsh/templates/zshrc.zsh-template ~/.zshrc
```

Add the plugins the role installs to its `plugins=()` line:

```
plugins=(git zsh-autosuggestions zsh-completions zsh-syntax-highlighting)
```

## Layout

```
devpc.yml            the playbook
requirements.yml     external role (geerlingguy.docker) and collections
ansible.cfg          local-only defaults
roles/
  homebrew/          macOS only: verifies Xcode CLT + Homebrew, wires shellenv
  zsh/               shell and CLI tooling; tasks/{linux,darwin}.yml per OS
  rust/              rustup, embedded targets, cargo utilities
  docker_macos/      macOS only: Colima (+ tuned template) or Docker Desktop
  lazyvim/           Neovim + the LazyVim starter config
  macos_defaults/    macOS only: System Settings preferences
```

Roles that differ per platform branch on the `ansible_system` fact
(`Linux` / `Darwin`), so adding a third platform means adding a branch rather
than rewriting a role.
