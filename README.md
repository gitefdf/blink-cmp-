# blink-cmp
## Installation

WARNING

Blink 使用一个预构建的二进制文件作为模糊匹配器，当你切换到标签时，这个二进制文件会自动下载。你可以使用 Rust nightly 构建源代码，或者使用 Lua 实现。更多信息请参见模糊匹配文档。

###  **Requirements**

- Neovim 0.10+ :     
- 使用预构建的二进制文件 :  
  - curl  
  - git  
- 从源代码构建 :  
  - Rust nightly 或 rustup 

Note : 默认 ，blink 尝试使用 rust 实现模糊匹配 。然而 ，lua实现不需要任何依赖 。详见 fuzzy documentation 。

lazy.nvim

```lua
{
	'saghen/blink.cmp',
-- optional: 为代码片段提供片段源
	dependencies = { 'rafamadriz/friendly-snippets' },
-- use a release tag to download pre-built binaries
	version = '1.*',
-- AND/OR build from source, requires nightly: https://rust-lang.github.io/rustup/concepts/channels.html#working-with-nightly-rust
-- build = 'cargo build --release',
-- If you use nix, you can build from source using latest nightly rust with:
-- build = 'nix run .#build-plugin',
---@module 'blink.cmp'
---@type blink.cmp.Config
	opts = {
    -- 'default' (recommended) for mappings similar to built-in completions (C-y to accept)
    -- 'super-tab' for mappings similar to vscode (tab to accept)
    -- 'enter' for enter to accept
    -- 'none' for no mappings
    --
    -- All presets have the following mappings:
    -- C-space: Open menu or open docs if already open
    -- C-n/C-p or Up/Down: Select next/previous item
    -- C-e: Hide menu
    -- C-k: Toggle signature help (if signature.enabled = true)
    --
    -- See :h blink-cmp-config-keymap for defining your own keymap
    keymap = { preset = 'default' },

    appearance = {
      -- 'mono' (default) for 'Nerd Font Mono' or 'normal' for 'Nerd Font'
      -- Adjusts spacing to ensure icons are aligned
      nerd_font_variant = 'mono'
    },

    -- (Default) Only show the documentation popup when manually triggered
    completion = { documentation = { auto_show = false } },

    -- Default list of enabled providers defined so that you can extend it
    -- elsewhere in your config, without redefining it, due to `opts_extend`
    sources = {
      default = { 'lsp', 'path', 'snippets', 'buffer' },
    },

    -- (Default) Rust fuzzy matcher for typo resistance and significantly better performance
    -- You may use a lua implementation instead by using `implementation = "lua"` or fallback to the lua implementation,
    -- when the Rust fuzzy matcher is not available, by using `implementation = "prefer_rust"`
    --
    -- See the fuzzy documentation for more information
    fuzzy = { implementation = "prefer_rust_with_warning" }
  },
  opts_extend = { "sources.default" }
}
```

## Configuration

### General

Blink cmp 有很多配置选项，以下代码块展示了一些你可能会关心的变化。更多信息，请查看其他页面。

对于更常见的配置，请参阅 recipes 。

**警告**

不要复制整个配置！它只包含非默认选项。

```lua
{
-- 启用按键映射、自动补全和签名帮助，默认为 true（不适用于命令行或终端）
-- 如果函数返回 'force'，默认的禁用插件条件将被忽略
-- 默认条件：(vim.bo.buftype ~= 'prompt' and vim.b.completion ~= false)
-- 注意：当 `vim.b.completion` 显式设置为 `true` 时，默认条件将被忽略
-- 特例：vim.bo.filetype == 'dap-repl'
	enabled = function() return not vim.tbl_contains({ "lua", "markdown" }, vim.bo.filetype) end,
-- 禁用cmdline
	cmdline = { enabled = false },

	completion = {
-- 'prefix' 将对光标前的文本进行模糊匹配
-- 'full' 将对光标前后文本进行模糊匹配
-- 示例：'foo_|_bar' 将会对 'foo_' 进行 'prefix' 匹配，对 'foo__bar' 进行 'full' 匹配
	keyword = { range = 'full' },
-- 禁用自动括号
-- 注意：某些 LSP 可能仍会自动添加括号
	accept = { auto_brackets = { enabled = false }, },
	-- 默认不选中，选择时自动插入
	list = { selection = { preselect = false, auto_insert = true } },
-- 或通过函数设置
	list = { selection = { preselect = function(ctx) return vim.bo.filetype ~= 'markdown' end } },
	menu = {
-- 不自动显示补全菜单
		auto_show = false,
-- nvim-cmp 风格菜单
		draw = {
			columns = {
				{ "label", "label_description", gap = 1 },
				{ "kind_icon", "kind" }
			},
		}
	},
-- 选择补全项时显示文档
	documentation = { auto_show = true, auto_show_delay_ms = 500 },

-- 在当前行显示选中项的预览
	ghost_text = { enabled = true },
},
sources = {
	-- 如果不需要文本补全，可以移除 'buffer'，默认情况下只有 LSP 返回无项时才会启用
	default = { 'lsp', 'path', 'snippets', 'buffer' },
},

-- 使用预设的片段，查看片段文档获取更多信息
snippets = { preset = 'default', 'luasnip', 'mini_snippets' },
-- 实验性签名帮助支持
signature = { enabled = true }
}
```

### Appearance

如果你想了解如何更改补全菜单的外观，可以查看[menu draw configuration](https://cmp.saghen.dev/configuration/completion.html#menu-draw).。

#### Highlight groups

|组名|默认值|描述|
|---|---|---|
|`BlinkCmpMenu`|`Pmenu`|补全菜单窗口|
|`BlinkCmpMenuBorder`|`Pmenu`|补全菜单窗口边框|
|`BlinkCmpMenuSelection`|`PmenuSel`|补全菜单窗口可选项|
|`BlinkCmpScrollBarThumb`|`PmenuThumb`|滚动条拇指|
|`BlinkCmpScrollBarGutter`|`PmenuSbar`|滚动条槽|
|`BlinkCmpLabel`|`Pmenu`|补全项的标签|
|`BlinkCmpLabelDeprecated`|`PmenuExtra`|补全项的已废弃标签|
|`BlinkCmpLabelMatch`|`Pmenu`|（当前未使用）当补全项与查询匹配时的标签|
|`BlinkCmpLabelDetail`|`PmenuExtra`|补全项的标签描述|
|`BlinkCmpLabelDescription`|`PmenuExtra`|补全项的标签描述|
|`BlinkCmpKind`|`PmenuKind`|补全项的类型：图标/文本|
|`BlinkCmpKind<kind>`|`PmenuKind`|补全项的类型图标：/文本|
|`BlinkCmpSource`|`PmenuExtra`|补全项的来源|
|`BlinkCmpGhostText`|`NonText`|预览项的幽灵文本|
|`BlinkCmpDoc`|`NormalFloat`|文档窗口|
|`BlinkCmpDocBorder`|`NormalFloat`|文档窗口边框|
|`BlinkCmpDocSeparator`|`NormalFloat`|文档与详情之间的分隔符|
|`BlinkCmpDocCursorLine`|`Visual`|文档窗口中的光标行|
|`BlinkCmpSignatureHelp`|`NormalFloat`|签名帮助窗口|
|`BlinkCmpSignatureHelpBorder`|`NormalFloat`|签名帮助窗口边框|
|`BlinkCmpSignatureHelpActiveParameter`|`LspSignatureActiveParameter`|签名帮助中的活动参数|

### Completion

Blink cmp 拥有许多配置选项，以下文档尝试突出显示每个部分最常见的配置项。有关所有选项，请单击每个标题旁的“Go to default configuration”按钮。

---

#### Keyword

控制插件认为的关键词，用于模糊匹配和触发补全。最显著的配置项是 `range`，它控制关键词是否应与光标前后的文本匹配，或仅与光标前的文本匹配。

|prefix|full|
|-|-|
|![[Pasted image 20250704175744.png]]|![[Pasted image 20250704175755.png]]|

---

#### Trigger

控制何时从源请求补全项并显示补全菜单。以下选项可用，排除了它们的 `show_on` 前缀：

- **显示关键词后，通常是字母数字字符、- 或 _**


### Fuzzy

### Keymap

Blink 使用一种特殊的方案来定义键映射，因为它需要处理回退到其他映射。然而，你也可以直接使用 `require('blink.cmp')` 并自己实现这些键映射。

你的自定义键映射会与预设进行合并，任何冲突的键都会覆盖预设的映射。The `fallback` command将运行下一个非 Blink 键映射。

---

#### Example

>**警告**  
这些键映射仅适用于默认模式，不适用于命令行模式或终端模式。有关更多信息，请参见 cmdline和 term 文档。

每个键映射可以是命令和/或函数的列表，其中命令直接映射到 `require('blink.cmp')[command]()`。如果命令/函数返回 `false` 或 `nil`，下一个命令/函数将会被执行。

```lua
keymap = {
-- 设置为 'none' 来禁用 'default' 预设
	preset = 'default',
	['<Up>'] = { 'select_prev', 'fallback' },
	['<Down>'] = { 'select_next', 'fallback' },
-- 禁用预设中的键映射
	  ['<C-e>'] = false, -- 或者 {}
-- 用提供列表显示
	['<C-space>'] = { function(cmp) cmp.show({ providers = { 'snippets' } }) end },
-- 控制在使用函数时下一个命令是否执行
	['<C-n>'] = { 
		 function(cmp)
			if some_condition then return end -- 执行下一个命令
			return true -- 不执行下一个命令
		end,
		'select_next'
	},
}
```

---

#### Commands

- show: 显示补全菜单  
  - 可选地使用 `function(cmp) cmp.show({ providers = { 'snippets' } }) end` 显示特定提供列表。  
- show_and_insert: 显示补全菜单并插入第一个项  
  - 如果 `auto_insert = true`，这是 `cmp.show({ initial_selected_item_idx = 1 })` 的简写。  
- hide: 隐藏补全菜单  
- cancel: 恢复 `completion.list.selection.auto_insert` 并隐藏补全菜单。 
- accept: 接受当前选中的项  
  - 可选地传递索引来选择列表中的特定项：`function(cmp) cmp.accept({ index = 1 }) end`  
  - 可选地传递回调函数，接受项后执行：`function(cmp) cmp.accept({ callback = function() some_function() end }) end`  
- accept_and_enter: 接受当前选中的项并输入回车  
  - 用于在命令行模式下，接受当前项并执行命令。  
- select_and_accept: 接受当前选中的项，若没有选中项，则接受第一个项。  
- select_accept_and_enter: 接受当前选中的项，若没有选中项，则接受第一个项，并输入回车  
  - 用于在命令行模式下，接受当前项并执行命令。  
- select_prev: 选择前一个项，若在顶部，则循环到底部（如果 `completion.list.cycle.from_top == true`）。  
  - 可选地控制 `completion.list.selection.auto_insert` 的 `auto_insert` 属性：`function(cmp) cmp.select_prev({ auto_insert = false }) end`  
  - 可选地在幽灵文本可见时运行，而不仅仅是菜单可见时：`function(cmp) cmp.select_prev({ on_ghost_text = true }) end`
- select_next: 选择下一个项，若在底部，则循环到顶部（如果 `completion.list.cycle.from_bottom == true`）。  
  - 可选地控制 `completion.list.selection.auto_insert` 的 `auto_insert` 属性：`function(cmp) cmp.select_next({ auto_insert = false }) end`  
  - 可选地在幽灵文本可见时运行，而不仅仅是菜单可见时：`function(cmp) cmp.select_next({ on_ghost_text = true }) end`  
- insert_prev: 插入前一个项（`auto_insert`），若在顶部，则循环到底部（如果 `completion.list.cycle.from_top == true`）。如果没有可用补全项，则会触发补全，不像 `select_prev` 会回退到下一个键映射。  
- insert_next: 插入下一个项（`auto_insert`），若在底部，则循环到顶部（如果 `completion.list.cycle.from_bottom == true`）。如果没有可用补全项，则会触发补全，不像 `select_next` 会回退到下一个键映射。  
- show_documentation: 显示当前选中项的文档  
- hide_documentation: 隐藏文档  
- scroll_documentation_up: 向上滚动文档 4 行  
  - 可选地使用 `function(cmp) cmp.scroll_documentation_up(4) end` 以特定行数滚动。  
- scroll_documentation_down: 向下滚动文档 4 行  
  - 可选地使用 `function(cmp) cmp.scroll_documentation_down(4) end` 以特定行数滚动。  
- show_signature: 显示签名帮助窗口  
- hide_signature: 隐藏签名帮助窗口  
- snippet_forward: 跳转到下一个代码片段占位符  
- snippet_backward: 跳转到上一个代码片段占位符  
- fallback: 运行下一个非 Blink 键映射，或者运行内建的 Neovim 键映射。  
- fallback_to_mappings: 运行下一个非 Blink 键映射（不是内建行为）。  

---

#### Cmdline and Terminal

See the respective cmdline documentation and terminal documentation for more information.

---

#### Presets

设置 preset to `none` 禁用预设

##### deafult

```lua
['<C-space>'] = { 'show', 'show_documentation', 'hide_documentation' },
['<C-e>'] = { 'hide' },
['<C-y>'] = { 'select_and_accept' },

['<Up>'] = { 'select_prev', 'fallback' },
['<Down>'] = { 'select_next', 'fallback' },
['<C-p>'] = { 'select_prev', 'fallback_to_mappings' },
['<C-n>'] = { 'select_next', 'fallback_to_mappings' },

['<C-b>'] = { 'scroll_documentation_up', 'fallback' },
['<C-f>'] = { 'scroll_documentation_down', 'fallback' },

['<Tab>'] = { 'snippet_forward', 'fallback' },
['<S-Tab>'] = { 'snippet_backward', 'fallback' },

['<C-k>'] = { 'show_signature', 'hide_signature', 'fallback' },
```

##### cmdline

详见 cmd 文档

##### super-tab

你可能希望设置 `completion.trigger.show_in_snippet = false` 或使用 `completion.list.selection.preselect = function(ctx) return not require('blink.cmp').snippet_active({ direction = 1 }) end` 详见 [[#Completion]]

```lua
['<C-space>'] = { 'show', 'show_documentation', 'hide_documentation' },
['<C-e>'] = { 'hide', 'fallback' },

['<Tab>'] = {
	function(cmp)
		if cmp.snippet_active() then return cmp.accept()
		telse return cmp.select_and_accept() end
	end,
	'snippet_forward',
	'fallback'
},
['<S-Tab>'] = { 'snippet_backward', 'fallback' },

['<Up>'] = { 'select_prev', 'fallback' },
['<Down>'] = { 'select_next', 'fallback' },
['<C-p>'] = { 'select_prev', 'fallback_to_mappings' },
['<C-n>'] = { 'select_next', 'fallback_to_mappings' },

['<C-b>'] = { 'scroll_documentation_up', 'fallback' },
['<C-f>'] = { 'scroll_documentation_down', 'fallback' },

['<C-k>'] = { 'show_signature', 'hide_signature', 'fallback' },
```

##### enter

> 你可能希望设置 `completion.list.selection.preselect = false`。有关更多信息，详见 [[#Completion]]

