# Recipes

Feel free to open a PR with any of your own recipes!

[[toc]]

## General

### Disable per filetype/buffer

You may change the `enabled` function to return `false` for any case you'd like to disable completion.

```lua
enabled = function() return not vim.tbl_contains({ "lua", "markdown" }, vim.bo.filetype) end,
```

or set `vim.b.completion = false` on the buffer

```lua
-- via an autocmd
vim.api.nvim_create_autocmd('BufEnter', {
  pattern = '*.lua',
  callback = function()
    vim.b.completion = false
  end,
})

-- or via ftplugin/some-filetype.lua
vim.b.completion = false
```

### Disable completion in *only* shell command mode

When inside of git bash or WSL on windows, you may experience a hang with shell commands. The following disables cmdline completions only when running shell commands (i.e. `[':!' , ':%!']`), but still allows completion in other command modes (i.e. `[':' , ':help', '/' , '?', ...]`).

```lua
sources = {
  providers = {
    cmdline = {
      -- ignores cmdline completions when executing shell commands
      enabled = function()
        return vim.fn.getcmdtype() ~= ':' or not vim.fn.getcmdline():match("^[%%0-9,'<>%-]*!")
      end
    }
  }
}
```

### Emacs behavior

Full discussion: https://github.com/Saghen/blink.cmp/issues/1367

```lua
local has_words_before = function()
  local col = vim.api.nvim_win_get_cursor(0)[2]
  if col == 0 then
    return false
  end
  local line = vim.api.nvim_get_current_line()
  return line:sub(col, col):match("%s") == nil
end

-- in your blink configuration
keymap = {
  preset = 'none',

  -- If completion hasn't been triggered yet, insert the first suggestion; if it has, cycle to the next suggestion.
  ['<Tab>'] = {
    function(cmp)
      if has_words_before() then
        return cmp.insert_next()
      end
    end,
    'fallback',
  },
  -- Navigate to the previous suggestion or cancel completion if currently on the first one.
  ['<S-Tab>'] = { 'insert_prev' },
},
completion = {
  menu = { enabled = false },
  list = { selection = { preselect = false }, cycle = { from_top = false } },
}
```

### Border

On neovim 0.11+, you may use the `vim.o.winborder` option to set the default border for all floating windows. You may override that option with your own border value as shown below.

```lua
completion = {
  menu = { border = 'single' },
  documentation = { window = { border = 'single' } },
},
signature = { window = { border = 'single' } },
```

### Select Nth item from the list

Based on [#382](https://github.com/Saghen/blink.cmp/issues/382)

```lua
keymap = {
  preset = 'default',
  ['<A-1>'] = { function(cmp) cmp.accept({ index = 1 }) end },
  ['<A-2>'] = { function(cmp) cmp.accept({ index = 2 }) end },
  ['<A-3>'] = { function(cmp) cmp.accept({ index = 3 }) end },
  ['<A-4>'] = { function(cmp) cmp.accept({ index = 4 }) end },
  ['<A-5>'] = { function(cmp) cmp.accept({ index = 5 }) end },
  ['<A-6>'] = { function(cmp) cmp.accept({ index = 6 }) end },
  ['<A-7>'] = { function(cmp) cmp.accept({ index = 7 }) end },
  ['<A-8>'] = { function(cmp) cmp.accept({ index = 8 }) end },
  ['<A-9>'] = { function(cmp) cmp.accept({ index = 9 }) end },
  ['<A-0>'] = { function(cmp) cmp.accept({ index = 10 }) end },
},
completion = {
  menu = {
    draw = {
      columns = { { 'item_idx' }, { 'kind_icon' }, { 'label', 'label_description', gap = 1 } },
      components = {
        item_idx = {
          text = function(ctx) return ctx.idx == 10 and '0' or ctx.idx >= 10 and ' ' or tostring(ctx.idx) end,
          highlight = 'BlinkCmpItemIdx' -- optional, only if you want to change its color
        }
      }
    }
  }
}
```

### Hide Copilot on suggestion

```lua
vim.api.nvim_create_autocmd('User', {
  pattern = 'BlinkCmpMenuOpen',
  callback = function()
    require("copilot.suggestion").dismiss()
    vim.b.copilot_suggestion_hidden = true
  end,
})

vim.api.nvim_create_autocmd('User', {
  pattern = 'BlinkCmpMenuClose',
  callback = function()
    vim.b.copilot_suggestion_hidden = false
  end,
})
```

### Avoid multi-line completion ghost text

See [nvim-cmp#1955](https://github.com/hrsh7th/nvim-cmp/pull/1955#issue-2341857764) for an example of what this looks like.

When ghost text is enabled (`completion.ghost_text.enabled = true`), you may want the menu to avoid overlapping with the ghost text. You may provide a custom `completion.menu.direction_priority` function to achieve this

```lua
completion = {
  menu = {
    direction_priority = function()
      local ctx = require('blink.cmp').get_context()
      local item = require('blink.cmp').get_selected_item()
      if ctx == nil item == nil then return { 's', 'n' } end

      local item_text = item.textEdit ~= nil and item.textEdit.newText or item.insertText or item.label
      local is_multi_line = item_text:find('\n') ~= nil

      -- after showing the menu upwards, we want to maintain that direction
      -- until we re-open the menu, so store the context id in a global variable
      if is_multi_line or vim.g.blink_cmp_upwards_ctx_id == ctx.id then
        vim.g.blink_cmp_upwards_ctx_id = ctx.id
        return { 'n', 's' }
      end
      return { 's', 'n' }
    end,
  },
},
```

### Show on newline, tab and space

::: warning
Not working as expected, see [#836](https://github.com/Saghen/blink.cmp/issues/836)
:::

Note that you may want to add the override to other sources as well, since if the LSP doesn't return any items, we won't show the menu if it was triggered by any of these three characters.

```lua
-- by default, blink.cmp will block newline, tab and space trigger characters, disable that behavior
completion.trigger.show_on_blocked_trigger_characters = {}

-- add newline, tab and space to LSP source trigger characters
sources.providers.lsp.override.get_trigger_characters = function(self)
  local trigger_characters = self:get_trigger_characters()
  vim.list_extend(trigger_characters, { '\n', '\t', ' ' })
  return trigger_characters
end
```

## Fuzzy (sorting/filtering)

[See the full docs](./configuration/fuzzy.md)

### Always prioritize exact matches

By default, the fuzzy matcher will give a bonus score of 4 to exact matches. If you want to ensure that exact matches are always prioritized, you may set:

```lua
fuzzy = {
  sorts = {
    'exact',
    -- defaults
    'score',
    'sort_text',
  },
}
```

### Deprioritize specific LSP

You may use a custom sort function to deprioritize LSPs such as Emmet Language Server (`emmet_ls`)

```lua
fuzzy = {
  sorts = {
    function(a, b)
      if (a.client_name == nil or b.client_name == nil) or (a.client_name == b.client_name) then
        return
      end
      return b.client_name == 'emmet_ls'
    end,
    -- default sorts
    'score',
    'sort_text',
}
```

### Exclude keywords/constants from autocomplete

Removes language keywords/constants (if, else, while, etc.) provided by the language server from completion results. Useful if you prefer to use builtin or custom snippets for such constructs.

```lua
sources = {
  providers = {
    lsp = {
      name = 'LSP',
      module = 'blink.cmp.sources.lsp',
      transform_items = function(_, items)
        return vim.tbl_filter(function(item)
          return item.kind ~= require('blink.cmp.types').CompletionItemKind.Keyword
        end, items)
      end,
    },
  },
}
```

## Completion menu drawing

[See the full docs](./configuration/completion.md#menu-draw)

### Kind icon background

You'll need to configure your highlights (`BlinkCmpKind` or `BlinkCmpKind<kind>`) to your desired background and foreground colors.

```lua
completion = {
  menu = {
    draw = {
      padding = { 0, 1 }, -- padding only on right side
      components = {
        kind_icon = {
          text = function(ctx) return ' ' .. ctx.kind_icon .. ctx.icon_gap .. ' ' end
        }
      }
    }
  }
}
```

### `mini.icons`

[Original discussion](https://github.com/Saghen/blink.cmp/discussions/458)

```lua
completion = {
  menu = {
    draw = {
      components = {
        kind_icon = {
          text = function(ctx)
            local kind_icon, _, _ = require('mini.icons').get('lsp', ctx.kind)
            return kind_icon
          end,
          -- (optional) use highlights from mini.icons
          highlight = function(ctx)
            local _, hl, _ = require('mini.icons').get('lsp', ctx.kind)
            return hl
          end,
        },
        kind = {
          -- (optional) use highlights from mini.icons
          highlight = function(ctx)
            local _, hl, _ = require('mini.icons').get('lsp', ctx.kind)
            return hl
          end,
        }
      }
    }
  }
}
```

### `nvim-web-devicons` + `lspkind`

[Original discussion](https://github.com/Saghen/blink.cmp/discussions/1146)

```lua
completion = {
  menu = {
    draw = {
      components = {
        kind_icon = {
          text = function(ctx)
            local icon = ctx.kind_icon
            if vim.tbl_contains({ "Path" }, ctx.source_name) then
                local dev_icon, _ = require("nvim-web-devicons").get_icon(ctx.label)
                if dev_icon then
                    icon = dev_icon
                end
            else
                icon = require("lspkind").symbolic(ctx.kind, {
                    mode = "symbol",
                })
            end

            return icon .. ctx.icon_gap
          end,

          -- Optionally, use the highlight groups from nvim-web-devicons
          -- You can also add the same function for `kind.highlight` if you want to
          -- keep the highlight groups in sync with the icons.
          highlight = function(ctx)
            local hl = ctx.kind_hl
            if vim.tbl_contains({ "Path" }, ctx.source_name) then
              local dev_icon, dev_hl = require("nvim-web-devicons").get_icon(ctx.label)
              if dev_icon then
                hl = dev_hl
              end
            end
            return hl
          end,
        }
      }
    }
  }
}
```

## Sources

[See the full docs](./configuration/sources.md)

### Buffer completion from all open buffers

The default behavior is to only show completions from **visible** "normal" buffers (i.e. it wouldn't include neo-tree). This will instead show completions from all buffers, even if they're not visible on screen. Note that the performance impact of this has not been tested.

```lua
sources = {
  providers = {
    buffer = {
      opts = {
        -- get all buffers, even ones like neo-tree
        get_bufnrs = vim.api.nvim_list_bufs
        -- or (recommended) filter to only "normal" buffers
        get_bufnrs = function()
          return vim.tbl_filter(function(bufnr)
            return vim.bo[bufnr].buftype == ''
          end, vim.api.nvim_list_bufs())
        end
      }
    }
  }
}
```

### Dynamically picking providers by treesitter node/filetype

```lua
sources.default = function(ctx)
  local success, node = pcall(vim.treesitter.get_node)
  if success and node and vim.tbl_contains({ 'comment', 'line_comment', 'block_comment' }, node:type()) then
    return { 'buffer' }
  elseif vim.bo.filetype == 'lua' then
    return { 'lsp', 'path' }
  else
    return { 'lsp', 'path', 'snippets', 'buffer' }
  end
end
```

### Hide snippets after trigger character

Trigger characters are defined by the sources. For example, for Lua, the trigger characters are `.`, `"`, `'`.

```lua
sources.providers.snippets.should_show_items = function(ctx)
  return ctx.trigger.initial_kind ~= 'trigger_character'
end
```

### Set source kind icon and name

```lua
sources.providers.copilot.transform_items = function(ctx, items)
  for _, item in ipairs(items) do
    item.kind_icon = ''
    item.kind_name = 'Copilot'
  end
  return items
end
```

### Disable all snippets

See the [relevant section in the snippets documentation](./configuration/snippets.md#disable-all-snippets)

### Set minimum keyword length by filetype

```lua
sources.min_keyword_length = function()
  return vim.bo.filetype == 'markdown' and 2 or 0
end
```

### Path completion from `cwd` instead of current buffer's directory

It's common to run code from the root of your repository, in which case relative paths will start from that directory. In that case, you may want path completions to be relative to your current working directory rather than the default, which is the current buffer's parent directory.

```lua
sources = {
  providers = {
    path = {
      opts = {
        get_cwd = function(_)
          return vim.fn.getcwd()
        end,
      },
    },
  },
},
```

This also makes it easy to `:cwd` to the desired base directory for path completion.

## For writers

When writing prose, you may want significantly different behavior than typical LSP completions. If you find any interesting configurations, please open a PR adding it here!

### Keep first letter capitalization on buffer source

```lua
sources = {
  providers = {
    buffer = {
      -- keep case of first char
      transform_items = function (a, items)
        local keyword = a.get_keyword()
        local correct, case
        if keyword:match('^%l') then
            correct = '^%u%l+$'
            case = string.lower
        elseif keyword:match('^%u') then
            correct = '^%l+$'
            case = string.upper
        else
            return items
        end

        -- avoid duplicates from the corrections
        local seen = {}
        local out = {}
        for _, item in ipairs(items) do
            local raw = item.insertText
            if raw:match(correct) then
                local text = case(raw:sub(1,1)) .. raw:sub(2)
                item.insertText = text
                item.label = text
            end
            if not seen[item.insertText] then
                seen[item.insertText] = true
                table.insert(out, item)
            end
        end
        return out
      end
    }
  }
}
```
