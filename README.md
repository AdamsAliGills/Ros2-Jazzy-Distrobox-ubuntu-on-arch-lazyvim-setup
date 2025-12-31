# Ros2-Jazzy-Distrobox-ubuntu-on-arch-lazyvim-setup
Personal setup and tutorial 

## ROS 2 Jazzy Installation via Distrobox (Arch Linux Host)

This guide details setting up a ROS 2 Jazzy environment on Arch Linux using Distrobox and Podman, including GUI support and Neovim integration.

### 1. Install Dependencies

Install Podman and Distrobox on your Arch host.

```bash
sudo pacman -S distrobox podman

```

### 2. Configure X11 Permissions

Enable the container to access the host's X server for GUI applications.

```bash
sudo pacman -S xorg-xhost
xhost +local:docker

```

### 3. Create the Container

Initialize an Ubuntu 24.04 container with a dedicated home directory.

```bash
distrobox create --image ubuntu:24.04 --name ros-jazzy --home ~/distrobox/ubuntu-dot-files --init

```

*Note: Add `--nvidia` to the end of the command above if using an NVIDIA GPU.*

### 4. Enter the Container

Access the newly created environment.

```bash
distrobox enter ros-jazzy

```

### 5. Install ROS 2 Jazzy

Configure locales and the ROS 2 repository, then install the desktop suite.
System Files (/opt/ros/): These are the tools (like ros2, rviz2, and gazebo) installed by the apt commands.
They stay in the root directory to ensure stability and shared access.

```bash
sudo apt update && sudo apt install locales -y
sudo locale-gen en_US en_US.UTF-8
sudo update-locale LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8
export LANG=en_US.UTF-8

sudo apt update && sudo apt install curl software-properties-common -y
sudo add-apt-repository universe -y
sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key -o /usr/share/keyrings/ros-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] http://packages.ros.org/ros2/ubuntu $(. /etc/os-release && echo $UBUNTU_CODENAME) main" | sudo tee /etc/apt/sources.list.d/ros2.list > /dev/null

sudo apt update
sudo apt install ros-jazzy-desktop ros-jazzy-ros-gz ros-dev-tools -y

```

### 6. Configure Environment and GUI

Add ROS sourcing and GUI compatibility variables to the container's `.bashrc`.

```bash
echo "source /opt/ros/jazzy/setup.bash" >> ~/.bashrc
echo "export GZ_IP=127.0.0.1" >> ~/.bashrc
echo "export QT_QPA_PLATFORM=xcb" >> ~/.bashrc
echo "export QT_STYLE_OVERRIDE=Fusion" >> ~/.bashrc

source ~/.bashrc

```

### 7. Verify Installation

Run Gazebo Sim to ensure the GUI is rendering correctly.

```bash
gz sim empty.sdf

```

### 8. Set Up Neovim (LazyVim)

Install Neovim and dependencies inside the container.

```bash
# Add Neovim Unstable PPA and update
sudo add-apt-repository ppa:neovim-ppa/unstable -y
sudo apt update

# Install Neovim, Build Essentials, and Search Tools
# Added: cmake, gettext, ninja-build, tar, wget (essential for building/extracting plugins)
sudo apt install neovim git curl build-essential python3-venv python3-pip ripgrep fd-find unzip cmake gettext ninja-build tar wget -y

# Install Node.js & npm (Required for Mason and many LSPs)
sudo apt install nodejs npm -y

# Install C++ Development tools for ROS 2
# Added: clang, clang-tidy, gdb (for full development/debugging suite)
sudo apt install clang clang-tidy clang-format clangd gdb -y

# Fix the fd-find naming convention for Neovim plugin compatibility
sudo ln -s $(which fdfind) /usr/local/bin/fd

# Clone the LazyVim starter template
git clone https://github.com/LazyVim/starter ~/.config/nvim

# Clean up git history to allow for your own configuration tracking
rm -rf ~/.config/nvim/.git
```

### 9. Shared Clipboard Support

Install `wl-clipboard` on both systems and configure Neovim to use the system clipboard.

**On Arch (Host):**

```bash
sudo pacman -S wl-clipboard

```

**In Distrobox:**

```bash
sudo apt update && sudo apt install wl-clipboard -y

```

**Neovim Config (options.lua):**

```lua
vim.opt.clipboard = "unnamedplus"

```

### 10. Hyprland Keybinds

Add these shortcuts to your `hyprland.conf` or `bindings.conf` for quick access. A system restart is required for changes to take effect.

```bash
bindd = SUPER SHIFT, U, Ubuntu Neovim, exec, uwsm-app -- xdg-terminal-exec sh -c "distrobox enter ros-jazzy -- nvim"
bindd = SUPER, BACKSLASH, Ubuntu Shell, exec, uwsm-app -- xdg-terminal-exec sh -c "distrobox enter ros-jazzy"

```

### 11. Workspace Auto-Sourcing to have any packages ready

To automatically source your specific workspace (e.g., Armbot), add this logic to your container's `.bashrc`.

```bash
if [ -f "$HOME/armbot/install/setup.bash" ]; then
    source "$HOME/armbot/install/setup.bash"
fi

```

### 12. build complie commands so lsp works with this bash function. use the ubuntu container.
you could techincially keep the jsons in the home directory so all similar projects
have the lsp ready to go!

```bash
nvim ~/.bashrc
```
so in nvim copy the function belwo :w write it
and then run it in terminal lspbuild().
### pain point ..... have a c++ program ready as in some library headers with the cmake file.... then build
other wise ur json will just be empty or not exist (this messed me up before)

```bash
rosbuild() {
    # 1. Run the build with the special flag
    echo "Building with colcon..."
    colcon build --cmake-args -DCMAKE_EXPORT_COMPILE_COMMANDS=ON "$@"
    
    # 2. Handle C++ (compile_commands.json)
    # Uses fd if available (faster), falls back to find
    if command -v fd &> /dev/null; then
        target=$(fd -I "compile_commands.json" build/ | head -n 1)
    else
        target=$(find build -name "compile_commands.json" | head -n 1)
    fi

    if [ -n "$target" ]; then
        ln -sf "$target" ./compile_commands.json
        echo "Linked C++ compilation database."
    else
        echo "compile_commands.json not found!"
        echo "Make sure you create and build a C++ package first."
    fi

    # 3. Handle Python (pyrightconfig.json)
    if [ ! -f "pyrightconfig.json" ]; then
        echo "pyrightconfig.json missing. Creating it for ROS 2 Jazzy..."
        cat > pyrightconfig.json << EOF
{
  "extraPaths": [
    "/opt/ros/jazzy/lib/python3.12/site-packages",
    "install/lib/python3.12/site-packages"
  ],
  "pythonVersion": "3.12",
  "typeCheckingMode": "basic"
}
EOF
        echo "Created pyrightconfig.json"
    fi
}
```


