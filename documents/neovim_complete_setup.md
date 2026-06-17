# Neovim Setup: Complete Configuration and Installation
## For the ESP32-C3 Assembly Course

---

## Prerequisites

Before installing Neovim, ensure the following are in place:

```bash
# Verify ESP-IDF is installed and sourced (provides riscv32-esp-elf-gdb and openocd)
idf.py --version
riscv32-esp-elf-gdb --version
openocd --version

# Verify Git
git --version

# Verify a C compiler (for Treesitter parser compilation)
cc --version       # macOS: part of Xcode Command Line Tools
                   # Linux: sudo apt install build-essential

# Verify Node.js 20+ (required for cdt-gdb-adapter, the debug adapter)
node --version
```

**Install cdt-gdb-adapter** (required for debugging — riscv32-esp-elf-gdb
does not include DAP support, so this Node.js adapter bridges nvim-dap
to GDB via the GDB/MI protocol):

```bash
npm install -g cdt-gdb-adapter

# Verify both executables installed (note the camelCase names)
ls $(npm prefix -g)/bin/cdt*
# Expected: cdtDebugAdapter  cdtDebugTargetAdapter
```

If the executables are not on your PATH, add npm's global bin directory
to your shell config:

```bash
echo 'export PATH="$(npm prefix -g)/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

**macOS — install Xcode Command Line Tools if missing:**
```bash
xcode-select --install
```

---

## Step 1: Install Neovim

### macOS (Homebrew — recommended)

```bash
brew install neovim

# Verify — must be 0.9.0 or later
nvim --version | head -1
# Expected: NVIM v0.10.x or later
```

### Linux (Ubuntu/Debian)

The system package is often outdated. Use the official AppImage or release binary:

```bash
# Download latest stable release
curl -LO https://github.com/neovim/neovim/releases/latest/download/nvim-linux-x86_64.appimage
chmod +x nvim-linux-x86_64.appimage

# Install to /usr/local/bin
sudo mv nvim-linux-x86_64.appimage /usr/local/bin/nvim

# Verify
nvim --version | head -1
```

For Apple Silicon Linux (ARM64):

```bash
curl -LO https://github.com/neovim/neovim/releases/latest/download/nvim-linux-arm64.appimage
chmod +x nvim-linux-arm64.appimage
sudo mv nvim-linux-arm64.appimage /usr/local/bin/nvim
```

---

## Step 2: Directory Structure

Create the full configuration directory layout:

```bash
mkdir -p ~/.config/nvim/lua/plugins
mkdir -p ~/.config/nvim/lua/config
```

You will create these files:

```
~/.config/nvim/
├── init.lua                  ← entry point: bootstraps lazy.nvim, loads modules
└── lua/
    ├── plugins/
    │   └── init.lua          ← all plugin declarations
    └── config/
        ├── options.lua       ← editor options
        ├── keymaps.lua       ← all keymaps (build, debug, navigation)
        └── dap.lua           ← OpenOCD + GDB debug adapter configuration
```

---

## Step 3: `init.lua` — Entry Point

```bash
cat > ~/.config/nvim/init.lua << 'EOF'
-- ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
-- init.lua — Neovim entry point for ESP32-C3 Assembly Course
-- ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

-- Leader key must be set before lazy.nvim loads plugins
vim.g.mapleader      = ' '   -- <Space> as leader key
vim.g.maplocalleader = ' '

-- ── Bootstrap lazy.nvim (plugin manager) ──────────────────────────
local lazypath = vim.fn.stdpath('data') .. '/lazy/lazy.nvim'
if not vim.loop.fs_stat(lazypath) then
  vim.fn.system({
    'git', 'clone', '--filter=blob:none',
    'https://github.com/folke/lazy.nvim.git',
    '--branch=stable',
    lazypath,
  })
end
vim.opt.rtp:prepend(lazypath)

-- ── Load modules ──────────────────────────────────────────────────
require('config.options')    -- editor settings
require('lazy').setup('plugins')  -- plugins from lua/plugins/init.lua
require('config.keymaps')    -- keymaps (after plugins are loaded)
require('config.dap')        -- debug adapter configuration
EOF
```

---

## Step 4: `lua/plugins/init.lua` — All Plugins

```bash
cat > ~/.config/nvim/lua/plugins/init.lua << 'EOF'
-- ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
-- plugins/init.lua — plugin declarations for lazy.nvim
-- ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

return {

  -- ── Syntax highlighting ────────────────────────────────────────
  {
    'nvim-treesitter/nvim-treesitter',
    build = ':TSUpdate',
    event = { 'BufReadPost', 'BufNewFile' },
    opts = {
      ensure_installed = {
        'c',        -- C files in the course scaffold
        'asm',      -- RISC-V assembly .S files
        'make',     -- Makefile
        'cmake',    -- CMakeLists.txt
        'lua',      -- this config file
        'bash',     -- shell scripts
      },
      highlight = { enable = true },
      indent    = { enable = true },
    },
    config = function(_, opts)
      require('nvim-treesitter.configs').setup(opts)
    end,
  },

  -- ── LSP framework (for C files alongside assembly) ─────────────
  {
    'neovim/nvim-lspconfig',
    lazy = true,
  },

  -- ── Status line ────────────────────────────────────────────────
  {
    'nvim-lualine/lualine.nvim',
    event = 'VimEnter',
    opts  = {
      options = {
        theme = 'auto',
        globalstatus = true,
        section_separators = '',
        component_separators = '│',
      },
      sections = {
        lualine_a = { 'mode' },
        lualine_b = { 'branch', 'diff' },
        lualine_c = { { 'filename', path = 1 } },  -- show relative path
        lualine_x = { 'filetype' },
        lualine_y = { 'progress' },
        lualine_z = { 'location' },
      },
    },
  },

  -- ── Comment toggling ───────────────────────────────────────────
  {
    'numToStr/Comment.nvim',
    event = 'BufReadPost',
    opts  = {
      -- Assembly files use # for comments
      -- Comment.nvim uses 'commentstring' set per-filetype in options.lua
    },
  },

  -- ── Keymap helper (shows available keymaps on partial press) ───
  {
    'folke/which-key.nvim',
    event = 'VeryLazy',
    opts  = {
      window = { border = 'rounded' },
    },
  },

  -- ── Debug Adapter Protocol — core ─────────────────────────────
  {
    'mfussenegger/nvim-dap',
    -- Note: loaded at startup by config/dap.lua (not truly lazy)
  },

  -- ── DAP UI — panels for registers, stack, breakpoints, REPL ───
  {
    'rcarriga/nvim-dap-ui',
    dependencies = {
      'mfussenegger/nvim-dap',
      'nvim-neotest/nvim-nio',   -- required by nvim-dap-ui v4+
    },
    -- Note: loaded at startup by config/dap.lua (not truly lazy)
    opts = {
      -- All debug panels open on the RIGHT side.
      -- This keeps the bottom row free for :terminal splits
      -- (OpenOCD pane and serial monitor pane created by <leader>dw).
      layouts = {
        {
          elements = {
            { id = 'repl',        size = 0.40 },  -- GDB prompt — primary for assembly
            { id = 'stacks',      size = 0.25 },  -- call stack
            { id = 'breakpoints', size = 0.20 },  -- breakpoint list
            { id = 'scopes',      size = 0.15 },  -- local variables
          },
          size     = 45,          -- columns wide
          position = 'right',
        },
        -- No bottom layout — :terminal splits occupy that space
      },
      floating = {
        border   = 'rounded',
        mappings = { close = { 'q', '<Esc>' } },
      },
      render = {
        max_type_length = nil,
      },
    },
  },

  -- ── DAP virtual text — shows variable values inline ───────────
  {
    'theHamsta/nvim-dap-virtual-text',
    dependencies = { 'mfussenegger/nvim-dap', 'nvim-treesitter/nvim-treesitter' },
    -- Note: loaded at startup by config/dap.lua (not truly lazy)
    opts = {
      enabled                = true,
      enabled_commands       = true,
      highlight_changed_variables = true,
      highlight_new_as_changed    = false,
      commented              = false,
      show_stop_reason       = true,
      virt_text_pos          = 'eol',    -- show at end of line
    },
  },

}
EOF
```

---

## Step 5: `lua/config/options.lua` — Editor Settings

```bash
cat > ~/.config/nvim/lua/config/options.lua << 'EOF'
-- ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
-- config/options.lua — editor settings
-- ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

local opt = vim.opt

-- ── Display ───────────────────────────────────────────────────────
opt.number         = true       -- show line numbers
opt.relativenumber = true       -- relative numbers for fast navigation
opt.cursorline     = true       -- highlight current line
opt.signcolumn     = 'yes'      -- always show sign column (breakpoints, git)
opt.termguicolors  = true       -- 24-bit colour
opt.scrolloff      = 8          -- keep 8 lines above/below cursor
opt.sidescrolloff  = 8
opt.wrap           = false      -- no line wrapping (assembly lines can be long)
opt.showmode       = false      -- mode shown by lualine instead

-- ── Indentation (defaults — overridden for .S files below) ───────
opt.expandtab  = true           -- spaces by default
opt.tabstop    = 4
opt.shiftwidth = 4
opt.smartindent = true

-- ── Search ────────────────────────────────────────────────────────
opt.ignorecase = true
opt.smartcase  = true           -- case-sensitive when uppercase used
opt.hlsearch   = true
opt.incsearch  = true

-- ── Splits ────────────────────────────────────────────────────────
opt.splitright = true           -- vertical splits open to the right
opt.splitbelow = true           -- horizontal splits open below

-- ── Files ─────────────────────────────────────────────────────────
opt.updatetime  = 250           -- faster completion and diagnostics
opt.timeoutlen  = 400           -- shorter which-key timeout
opt.undofile    = true          -- persistent undo history
opt.swapfile    = false
opt.backup      = false

-- ── Terminal ──────────────────────────────────────────────────────
-- Start terminal buffers in insert mode automatically
vim.api.nvim_create_autocmd('TermOpen', {
  pattern = '*',
  callback = function()
    opt.number         = false
    opt.relativenumber = false
    vim.cmd('startinsert')
  end,
})

-- ── Assembly file settings (.S files) ────────────────────────────
-- Assembly uses real tabs (not spaces) and 8-space tab width.
-- Column guides at 40 and 60 mark the label/mnemonic/operand columns.
vim.api.nvim_create_autocmd('FileType', {
  pattern  = 'asm',
  callback = function()
    opt.expandtab    = false   -- real tabs
    opt.tabstop      = 8       -- 8-space tabs (conventional in GNU assembly)
    opt.shiftwidth   = 8
    opt.colorcolumn  = '40,60' -- visual guides: label | mnemonic | operand
    opt.commentstring = '# %s' -- assembly comment character
  end,
})

-- Treat .S files as assembly
vim.api.nvim_create_autocmd({ 'BufRead', 'BufNewFile' }, {
  pattern  = '*.S',
  callback = function()
    vim.bo.filetype = 'asm'
  end,
})

-- ── Clipboard ─────────────────────────────────────────────────────
opt.clipboard = 'unnamedplus'   -- sync with system clipboard
EOF
```

---

## Step 6: `lua/config/keymaps.lua` — All Keymaps

```bash
cat > ~/.config/nvim/lua/config/keymaps.lua << 'EOF'
-- ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
-- config/keymaps.lua — all keymaps for the assembly course
-- ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

local map = vim.keymap.set
local opts = { noremap = true, silent = true }

-- ── General navigation ────────────────────────────────────────────
-- Better split navigation (no Ctrl-W prefix needed)
map('n', '<C-h>', '<C-w>h', { desc = 'Move to left split' })
map('n', '<C-l>', '<C-w>l', { desc = 'Move to right split' })
map('n', '<C-j>', '<C-w>j', { desc = 'Move to lower split' })
map('n', '<C-k>', '<C-w>k', { desc = 'Move to upper split' })

-- Resize splits with arrow keys
map('n', '<C-Up>',    '<cmd>resize +2<cr>',          { desc = 'Increase split height' })
map('n', '<C-Down>',  '<cmd>resize -2<cr>',          { desc = 'Decrease split height' })
map('n', '<C-Left>',  '<cmd>vertical resize -2<cr>', { desc = 'Decrease split width' })
map('n', '<C-Right>', '<cmd>vertical resize +2<cr>', { desc = 'Increase split width' })

-- Clear search highlight
map('n', '<Esc>', '<cmd>nohlsearch<cr>', opts)

-- Better paste (don't overwrite register on paste over selection)
map('v', 'p', '"_dP', opts)

-- ── Terminal ──────────────────────────────────────────────────────
-- Open a terminal in a horizontal split
map('n', '<leader>t', '<cmd>split | terminal<cr>',
    { desc = 'Terminal: open horizontal split' })

-- Exit terminal insert mode more easily
map('t', '<Esc><Esc>', '<C-\\><C-n>', { desc = 'Terminal: exit insert mode' })

-- ── ESP-IDF Build and Flash ───────────────────────────────────────
-- These run idf.py commands in a :terminal split at the bottom.
-- The working directory must be a chapter directory (e.g. ch01/).

map('n', '<leader>bb', function()
  vim.cmd('botright split | resize 15 | terminal idf.py build')
end, { desc = 'ESP-IDF: build' })

map('n', '<leader>bf', function()
  vim.cmd('botright split | resize 15 | terminal idf.py build flash')
end, { desc = 'ESP-IDF: build and flash' })

map('n', '<leader>bm', function()
  vim.cmd('botright split | resize 15 | terminal idf.py monitor')
end, { desc = 'ESP-IDF: monitor (serial output)' })

map('n', '<leader>bd', function()
  vim.cmd('botright split | resize 15 | terminal idf.py build flash monitor')
end, { desc = 'ESP-IDF: build, flash, and monitor' })

map('n', '<leader>bc', function()
  vim.cmd('botright split | resize 15 | terminal idf.py fullclean')
end, { desc = 'ESP-IDF: full clean' })

-- ── Disassembly ───────────────────────────────────────────────────
map('n', '<leader>da', function()
  vim.cmd('botright split | resize 20 | ' ..
    'terminal riscv32-esp-elf-objdump -d -M no-aliases build/*.elf')
end, { desc = 'Disassemble: full ELF' })

map('n', '<leader>df', function()
  vim.ui.input({ prompt = 'Function name: ' }, function(name)
    if name and name ~= '' then
      vim.cmd(string.format(
        'botright split | resize 20 | terminal ' ..
        'riscv32-esp-elf-objdump -d -M no-aliases build/*.elf | ' ..
        'grep -A 50 "<%s>"', name
      ))
    end
  end)
end, { desc = 'Disassemble: specific function' })

map('n', '<leader>dh', function()
  vim.cmd('botright split | resize 10 | ' ..
    'terminal riscv32-esp-elf-objdump -h build/*.elf')
end, { desc = 'Disassemble: section headers' })

map('n', '<leader>ds', function()
  vim.cmd('botright split | resize 10 | ' ..
    'terminal riscv32-esp-elf-size -A build/*.elf')
end, { desc = 'Disassemble: section sizes' })

-- ── Debug workspace ───────────────────────────────────────────────
-- <leader>dw opens the three-pane debug layout:
--   Bottom-left  → OpenOCD  (hardware JTAG bridge, GDB server on :3333)
--   Bottom-right → idf.py monitor (serial output)
--   Focus returns to source pane
--
-- Workflow:
--   1. Press <leader>dw
--   2. Navigate to OpenOCD pane with <C-j>, wait for "Listening on port 3333"
--   3. Return to source with <C-k>
--   4. Set breakpoints with <leader>db
--   5. Press <F5> to start the debug session

map('n', '<leader>dw', function()
  local main_win = vim.api.nvim_get_current_win()

  -- Bottom-left: OpenOCD terminal (10 rows)
  vim.cmd('botright split')
  vim.cmd('resize 10')
  vim.cmd('terminal openocd -f board/esp32c3-builtin.cfg')
  -- Name the buffer so it is identifiable in :ls
  vim.api.nvim_buf_set_name(0, 'OpenOCD')

  -- Bottom-right: serial monitor (split the bottom pane)
  vim.cmd('vsplit')
  vim.cmd('terminal idf.py monitor')
  vim.api.nvim_buf_set_name(0, 'Monitor')

  -- Return focus to the original source pane
  vim.api.nvim_set_current_win(main_win)

  vim.notify(
    '[Debug] Workspace open.\n' ..
    'Wait for "Listening on port 3333" in OpenOCD pane (<C-j>),\n' ..
    'then set breakpoints (<leader>db) and press <F5>.',
    vim.log.levels.INFO
  )
end, { desc = 'Debug: Open workspace (OpenOCD + serial monitor)' })

-- ── Debug controls (loaded after nvim-dap) ────────────────────────
-- These are safe to define here; they resolve 'dap' lazily on first call.

local function dap()   return require('dap') end
local function dapui() return require('dapui') end

map('n', '<F5>',  function() dap().continue()         end, { desc = 'Debug: Continue / Start' })
map('n', '<F10>', function() dap().step_over()         end, { desc = 'Debug: Step Over (nexti)' })
map('n', '<F11>', function() dap().step_into()         end, { desc = 'Debug: Step Instruction (stepi)' })
map('n', '<F12>', function() dap().step_out()          end, { desc = 'Debug: Finish function' })

-- ── macOS alternatives ────────────────────────────────────────────
-- F10, F11, and F12 are intercepted by macOS (Mission Control /
-- media keys) and never reach Neovim. These leader-key bindings
-- work on all platforms without any system configuration changes.
--
-- To restore F-key behaviour in macOS instead:
--   System Settings → Keyboard → Keyboard Shortcuts →
--   Mission Control → uncheck "Show Desktop F11"
--
map('n', '<leader>dc', function() dap().continue()  end, { desc = 'Debug: Continue     (macOS alt: F5)' })
map('n', '<leader>dn', function() dap().step_over() end, { desc = 'Debug: Step Over    (macOS alt: F10)' })
map('n', '<leader>di', function() dap().step_into() end, { desc = 'Debug: Step Into    (macOS alt: F11)' })
map('n', '<leader>do', function() dap().step_out()  end, { desc = 'Debug: Step Out     (macOS alt: F12)' })

map('n', '<leader>db', function() dap().toggle_breakpoint() end,
    { desc = 'Debug: Toggle breakpoint' })

map('n', '<leader>dB', function()
  dap().set_breakpoint(vim.fn.input('Condition: '))
end, { desc = 'Debug: Conditional breakpoint' })

map('n', '<leader>du', function() dapui().toggle() end,
    { desc = 'Debug: Toggle UI panels' })

map('n', '<leader>dr', function() dap().repl.open() end,
    { desc = 'Debug: Open GDB REPL' })

map('n', '<leader>dx', function()
  dap().terminate()
  dapui().close()
end, { desc = 'Debug: Terminate session' })

-- ── which-key group labels ────────────────────────────────────────
-- Documents the <leader> key groups in the which-key popup
local ok, wk = pcall(require, 'which-key')
if ok then
  wk.add({
    { '<leader>b', group = 'Build / Flash' },
    { '<leader>d', group = 'Debug / Disassemble' },
    { '<leader>t', group = 'Terminal' },
    -- Register gc/gb as operator groups to silence the overlap warning
    { 'gc', group = 'Comment',         mode = { 'n', 'v' } },
    { 'gb', group = 'Comment (block)', mode = { 'n', 'v' } },
  })
end
EOF
```

---

## Step 7: `lua/config/dap.lua` — Debug Adapter Configuration

```bash
cat > ~/.config/nvim/lua/config/dap.lua << 'EOF'
-- ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
-- config/dap.lua — OpenOCD + GDB debug adapter for ESP32-C3
--
-- Architecture:
--   OpenOCD (started via <leader>dw) provides a GDB server on :3333.
--   nvim-dap launches riscv32-esp-elf-gdb, which connects to OpenOCD.
--   nvim-dap-ui displays debug state in the right-side panels.
--
-- Requirements:
--   - OpenOCD running: openocd -f board/esp32c3-builtin.cfg
--   - ELF flashed:    idf.py build flash
--   - GDB in PATH:    riscv32-esp-elf-gdb (via ESP-IDF export.sh)
--   - GDB version:    14.0+ (check: riscv32-esp-elf-gdb --version)
-- ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

local dap    = require("dap")
local dapui  = require("dapui")

-- ── cdt-gdb-adapter ───────────────────────────────────────────────
-- riscv32-esp-elf-gdb (esp-gdb) does not include DAP support —
-- confirmed: riscv32-esp-elf-gdb --interpreter=dap returns
-- "Interpreter `dap' unrecognized".
--
-- cdt-gdb-adapter is a Node.js DAP server that communicates with
-- GDB via GDB/MI protocol instead of DAP mode. It works with any
-- GDB version and is the adapter used by the ESP-IDF VS Code extension.
--
-- Install: npm install -g cdt-gdb-adapter
-- Requires Node.js 20.x LTS or later.
--
-- Two executables are installed:
--   cdtDebugAdapter        — local process debugging (sends 'target native';
--                            cannot work with a cross-debugger)
--   cdtDebugTargetAdapter  — remote target debugging ← REQUIRED for OpenOCD
--
-- VERIFIED WORKING: cdtDebugTargetAdapter + request="attach" +
-- target.connectCommands (see configurations below).

dap.adapters["cdt-gdb"] = {
  type    = "executable",
  command = "cdtDebugTargetAdapter",
  options = {
    initialize_timeout_sec = 30,
  },
}

-- ── Helper: locate the ELF produced by idf.py build ──────────────
local function find_elf()
  local function glob_first(pattern)
    local results = vim.fn.glob(pattern, false, true)
    return results[1]
  end

  local elf = glob_first(vim.fn.getcwd() .. "/build/*.elf")
           or glob_first(vim.fn.getcwd() .. "/../build/*.elf")

  if not elf or elf == "" then
    vim.notify(
      "[DAP] No .elf found in build/.\nRun: idf.py build flash\nThen press <F5> again.",
      vim.log.levels.ERROR
    )
    return nil
  end

  return elf
end

-- ── GDB init commands ─────────────────────────────────────────────
-- Plain strings — cdt-gdb-adapter passes these to GDB/MI.
-- Order follows Espressif JTAG Tips and Quirks documentation:
-- https://docs.espressif.com/projects/esp-idf/en/latest/esp32c3/api-guides/jtag-debugging/tips-and-quirks.html

-- initCommands run AFTER the connection is established by
-- target.connectCommands. Exactly the three commands from Espressif's
-- documented launch.json for the Eclipse CDT GDB Adapter.
local init_cmds = {
  "set remote hardware-watchpoint-limit 2",
  "mon reset halt",
  "maintenance flush register-cache",
}

-- connectCommands establish the OpenOCD connection. The MI form
-- (-target-select) is required by cdtDebugTargetAdapter.
-- remotetimeout 20 gives the USB-JTAG time to respond.
local connect_cmds = {
  "set remotetimeout 20",
  "-target-select extended-remote localhost:3333",
}

-- ── Debug configurations ──────────────────────────────────────────
-- VERIFIED WORKING configuration. The critical fields:
--   request = "attach"   — "launch" makes the adapter spawn gdbserver (fails)
--   target.connectCommands — explicit connection; without it the adapter
--                            attempts a bare 'target remote' (serial error)
-- Format mirrors Espressif's documented launch.json:
-- https://docs.espressif.com/projects/vscode-esp-idf-extension/en/latest/debugproject.html

local esp32c3_configs = {

  -- Built-in USB-JTAG — same USB-C cable as flash/monitor.
  -- Start OpenOCD with: openocd -f board/esp32c3-builtin.cfg
  --
  -- This is the only configuration for the SuperMini, which needs no
  -- external probe. With a single entry, <F5> launches it directly with
  -- no "Type number and <Enter>" picker prompt.
  {
    name         = "ESP32-C3 (Built-in USB-JTAG)",
    type         = "cdt-gdb",
    request      = "attach",
    gdb          = "riscv32-esp-elf-gdb",
    program      = find_elf,
    initCommands = init_cmds,
    target       = {
      connectCommands = connect_cmds,
    },
  },

  -- ── OPTIONAL: ESP-Prog external JTAG ────────────────────────────────
  -- Uncomment this block ONLY if you use an external ESP-Prog probe
  -- (GPIO4=TCK, 5=TDI, 6=TDO, 7=TMS). Adding a second entry brings back
  -- the configuration picker on every <F5>. Start OpenOCD with:
  --   openocd -f interface/ftdi/esp32_devkitj_v1.cfg -f target/esp32c3.cfg
  --
  -- {
  --   name         = "ESP32-C3 (ESP-Prog External JTAG)",
  --   type         = "cdt-gdb",
  --   request      = "attach",
  --   gdb          = "riscv32-esp-elf-gdb",
  --   program      = find_elf,
  --   initCommands = init_cmds,
  --   target       = {
  --     connectCommands = connect_cmds,
  --   },
  -- },

}

-- Apply to both assembly and C filetypes
dap.configurations.asm = esp32c3_configs
dap.configurations.c   = esp32c3_configs

-- ── nvim-dap-ui: auto open/close with debug sessions ─────────────
-- Listeners are registered here so dapui is available at this point.
-- The UI opens on the RIGHT (configured in plugins/init.lua).

dap.listeners.after.event_initialized['dapui_config'] = function()
  dapui.open()
end
dap.listeners.before.event_terminated['dapui_config'] = function()
  dapui.close()
end
dap.listeners.before.event_exited['dapui_config'] = function()
  dapui.close()
end

-- ── nvim-dap-virtual-text ─────────────────────────────────────────
-- Shows register/variable values inline as virtual text.
-- Wrapped in pcall so a missing plugin does not break startup.
local ok, dap_vtext = pcall(require, 'nvim-dap-virtual-text')
if ok then
  dap_vtext.setup({
    enabled                     = true,
    highlight_changed_variables = true,
    show_stop_reason            = true,
    virt_text_pos               = 'eol',
  })
end

-- ── Sign column icons ─────────────────────────────────────────────
-- Define highlight groups first so signs have colour regardless of
-- which colorscheme is active.

-- Breakpoint: red
vim.api.nvim_set_hl(0, 'DapBreakpoint',          { fg = '#e51400', default = true })
-- Conditional breakpoint: orange
vim.api.nvim_set_hl(0, 'DapBreakpointCondition', { fg = '#e5aa00', default = true })
-- Rejected breakpoint: grey
vim.api.nvim_set_hl(0, 'DapBreakpointRejected',  { fg = '#888888', default = true })
-- Log point: blue
vim.api.nvim_set_hl(0, 'DapLogPoint',            { fg = '#0070f3', default = true })
-- Current execution line: green text, subtle background
vim.api.nvim_set_hl(0, 'DapStopped',             { fg = '#00cc00', default = true })
vim.api.nvim_set_hl(0, 'DapStoppedLine',         { bg = '#1e3a1e', default = true })

vim.fn.sign_define('DapBreakpoint',          { text = '●', texthl = 'DapBreakpoint',          numhl = '' })
vim.fn.sign_define('DapBreakpointCondition', { text = '◉', texthl = 'DapBreakpointCondition', numhl = '' })
vim.fn.sign_define('DapBreakpointRejected',  { text = '○', texthl = 'DapBreakpointRejected',  numhl = '' })
vim.fn.sign_define('DapLogPoint',            { text = '◆', texthl = 'DapLogPoint',            numhl = '' })
vim.fn.sign_define('DapStopped',             { text = '▶', texthl = 'DapStopped', linehl = 'DapStoppedLine', numhl = '' })
EOF
```

---

## Step 8: First Launch and Plugin Installation

```bash
nvim
```

On first launch, lazy.nvim detects it is not installed and clones itself.
You will then see the lazy.nvim UI with all plugins queued for installation.
This takes 1–3 minutes depending on connection speed.

```
:Lazy    ← opens the plugin manager UI
```

After plugins install:

```
:Lazy sync          ← install/update all plugins
:TSUpdate           ← install/update all Treesitter parsers
:checkhealth        ← verify everything is working
```

You should see no errors in `:checkhealth` (warnings about optional
providers like Python or Node are safe to ignore for this course).

---

## Step 9: Verification

Run each of these checks after the first launch:

### Verify Treesitter parsers

```vim
:TSInstallInfo
```

Look for `asm`, `c`, `cmake`, `make` all showing as installed (`✓`).

To test syntax highlighting, open any assembly file:

```bash
nvim ~/esp32c3-assembly-course/ch01/main/my_func.S
```

Instructions, registers, and comments should each have distinct colours.

### Verify the leader key

```
Press: Space
```

The which-key popup should appear after 400ms showing the available groups
(`b` for Build, `d` for Debug, `t` for Terminal).

### Verify build keymaps (from inside a chapter directory)

```bash
cd ~/esp32c3-assembly-course/ch01
nvim main/my_func.S
```

Inside Neovim: `<leader>bb` should open a bottom terminal split running
`idf.py build`. Press `Ctrl+\ Ctrl+n` to exit insert mode in the terminal,
then `:q` to close it.

### Verify debug adapter

With OpenOCD running in a separate terminal:

```bash
openocd -f board/esp32c3-builtin.cfg
```

Inside Neovim:

```vim
<leader>dw    " opens OpenOCD + monitor panes
<leader>db    " set a breakpoint (cursor on any line)
<F5>          " starts session — dap-ui should open on the right
```

If the adapter fails to connect, check:

1. OpenOCD shows "Listening on port 3333" (not an error)
2. You ran `source ~/esp/esp-idf/export.sh` in the same shell as Neovim
3. `riscv32-esp-elf-gdb --version` works from that shell

### Verify GDB version (must be 14+)

```bash
riscv32-esp-elf-gdb --version | head -1
# Expected: GNU gdb (esp-gdb) 14.x.x
```

If the version is below 14, the `-i dap` flag will fail. Update ESP-IDF:

```bash
cd ~/esp/esp-idf
git checkout v6.0.1
git submodule update --init --recursive
./install.sh esp32c3
source export.sh
```

---

## Step 10: Updating Plugins

```vim
:Lazy update     " update all plugins to latest versions
:TSUpdate        " update Treesitter parsers
```

Run these periodically. After a Neovim major version upgrade (e.g. 0.10 →
0.11), run `:checkhealth` to catch any API changes.

---

## Complete Keymap Reference

### Build and Flash

| Keymap | Action |
|--------|--------|
| `<leader>bb` | `idf.py build` |
| `<leader>bf` | `idf.py build flash` |
| `<leader>bm` | `idf.py monitor` |
| `<leader>bd` | `idf.py build flash monitor` |
| `<leader>bc` | `idf.py fullclean` |

### Disassembly

| Keymap | Action |
|--------|--------|
| `<leader>da` | Full ELF disassembly |
| `<leader>df` | Disassemble named function (prompts for name) |
| `<leader>dh` | Section headers (`objdump -h`) |
| `<leader>ds` | Section sizes (`size -A`) |

### Debug Session

> **macOS:** `F10`, `F11`, `F12` are intercepted by the OS and never reach Neovim.
> Use the `<leader>d*` alternatives — they work on all platforms.
> To re-enable F-keys: **System Settings → Keyboard → Keyboard Shortcuts →
> Mission Control → uncheck `F11` (Show Desktop).**

| Keymap | macOS alternative | Action |
|--------|-----------------|--------|
| `<leader>dw` | — | Open debug workspace (OpenOCD + monitor panes) |
| `<F5>` | `<leader>dc` | Continue / start session |
| `<F10>` | `<leader>dn` | Step over instruction (`nexti`) |
| `<F11>` | `<leader>di` | Step one instruction (`stepi`) ← use most |
| `<F12>` | `<leader>do` | Finish current function |
| `<leader>db` | — | Toggle breakpoint at cursor |
| `<leader>dB` | — | Set conditional breakpoint |
| `<leader>du` | — | Toggle nvim-dap-ui panels |
| `<leader>dr` | — | Open GDB REPL pane |
| `<leader>dx` | — | Terminate debug session |

### Navigation

| Keymap | Action |
|--------|--------|
| `<C-h/j/k/l>` | Move between splits |
| `<C-Up/Down>` | Resize split height |
| `<C-Left/Right>` | Resize split width |
| `<Esc><Esc>` (terminal) | Exit terminal insert mode |
| `<leader>t` | Open terminal in horizontal split |

---

## Troubleshooting

**Treesitter `asm` parser not highlighting correctly**

```vim
:TSInstall asm
:TSUpdate asm
```

If `.S` files are not detected as `asm`, verify the autocmd in `options.lua`
is being loaded. Test: `nvim test.S` → `:set filetype?` → should say `asm`.

**`riscv32-esp-elf-gdb: command not found` when starting debug**

The ESP-IDF environment is not sourced in Neovim's shell. Add to your
shell config (`.bashrc` or `.zshrc`):

**`<F11>` / `<F10>` / `<F12>` do nothing on macOS**

These keys are bound to Mission Control and media functions at the OS level
and never reach Neovim. Use the leader-key alternatives instead:

| F-key | Use instead |
|-------|------------|
| `<F10>` | `<leader>dn` |
| `<F11>` | `<leader>di` |
| `<F12>` | `<leader>do` |

To restore F-key behaviour permanently: **System Settings → Keyboard →
Keyboard Shortcuts → Mission Control** and uncheck `Show Desktop F11`.
Also check the `Function Keys` tab — enabling "Use F1, F2, etc. as standard
function keys" restores all F-keys system-wide (you then hold `fn` for
brightness/volume instead).

```bash
source ~/esp/esp-idf/export.sh
```

Then restart your terminal before opening Neovim.

**nvim-dap-ui does not open when `<F5>` is pressed**

The auto-open listeners are in `dap.lua`. Verify it is being loaded:

```vim
:lua print(vim.inspect(require('dap').configurations.asm))
```

Should print the ESP32-C3 configuration table. If it prints `nil`, check
that `require('config.dap')` appears in `init.lua`.

**`<F5>` connects but shows "Connection refused"**

OpenOCD is not running. Press `<C-j>` to navigate to the OpenOCD pane
and check for errors. The most common cause is that `idf.py monitor` was
left running (press `Ctrl+]` there first to stop it).

**which-key popup does not appear**

Check that `folke/which-key.nvim` is installed (`:Lazy` should show it).
The popup appears after `timeoutlen` (400ms). If it still doesn't appear:

```vim
:WhichKey <leader>
```

