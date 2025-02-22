# gitlinker.nvim

A lua [neovim](https://github.com/neovim/neovim) plugin to generate
shareable file permalinks (with line ranges) for several git web
frontend hosts. Inspired by
[tpope/vim-fugitive](https://github.com/tpope/vim-fugitive)’s `:GBrowse`

Example of a permalink:
<https://github.com/neovim/neovim/blob/2e156a3b7d7e25e56b03683cc6228c531f4c91ef/src/nvim/main.c#L137-L156>

Personally, I use this all the time to easily share code locations with
my co-workers.

## Supported git web hosts

- [github](https://github.com)
- [gitlab](https://gitlab.com)
- [gitea](https://try.gitea.io)
- [bitbucket](https://bitbucket.org)
- [gogs](https://gogs.io)
- [cgit](https://git.zx2c4.com/cgit) (includes
  [git.kernel.org](https://git.kernel.org) and
  [git.savannah.gnu.org](http://git.savannah.gnu.org))
- [launchpad](https://launchpad.net)
- [repo.or.cz](https://repo.or.cz)

**You can easily configure support for more hosts** by defining your own
host callbacks. It’s even easier if your host is just an
enterprise/self-hosted github/gitlab/gitea/gogs/cgit instance since you
can just use the same callbacks that already exist in gitlinker! See
[callbacks](#callbacks)

## Installation

Install it like any other vim plugin, just make sure
[plenary.nvim](https://github.com/nvim-lua/plenary.nvim) is also
installed.

- [packer.nvim](https://github.com/wbthomason/packer.nvim)

``` lua
use {
    'ruifm/gitlinker.nvim',
    requires = 'nvim-lua/plenary.nvim',
}
```

- [vim-plug](https://github.com/junegunn/vim-plug)

``` vim
Plug 'nvim-lua/plenary.nvim'
Plug 'ruifm/gitlinker.nvim'
```

### Requirements

- git
- neovim 0.5
- [plenary.nvim](https://github.com/nvim-lua/plenary.nvim)

## Usage

In your `init.lua` or in a lua-here-doc in your `init.vim`:

``` lua
require"gitlinker".setup()
```

By default, the following mappings are defined:

- `<leader>gy` for normal and visual mode

When used, it will copy the generated url to your clipboard and print it
in `:messages`.

- In normal mode, it will add the current line number to the url
- In visual mode , it will add the line range of the visual selection to
  the url

## Configuration

To configure `gitlinker.nvim`, call `require"gitlinker".setup(config)`
in your `init.lua` or in a lua-here-doc in your `init.vim`.

Here’s all the options with their defaults:

``` lua
require"gitlinker".setup({
  opts = {
    remote = nil, -- force the use of a specific remote
    -- adds current line nr in the url for normal mode
    add_current_line_on_normal_mode = true,
    -- callback for what to do with the url
    action_callback = require"gitlinker.actions".copy_to_clipboard,
    -- print the url after performing the action
    print_url = true
  },
  callbacks = {
        ["github.com"] = require"gitlinker.hosts".get_github_type_url,
        ["gitlab.com"] = require"gitlinker.hosts".get_gitlab_type_url,
        ["try.gitea.io"] = require"gitlinker.hosts"get_gitea_type_url,
        ["codeberg.org"] = require"gitlinker.hosts"get_gitea_type_url,
        ["bitbucket.org"] = require"gitlinker.hosts"get_bitbucket_type_url,
        ["try.gogs.io"] = require"gitlinker.hosts"get_gogs_type_url,
        ["git.sr.ht"] = require"gitlinker.hosts"get_srht_type_url,
        ["git.launchpad.net"] = require"gitlinker.hosts"get_launchpad_type_url,
        ["repo.or.cz"] = require"gitlinker.hosts"get_repoorcz_type_url,
        ["git.kernel.org"] = require"gitlinker.hosts"get_cgit_type_url,
        ["git.savannah.gnu.org"] = require"gitlinker.hosts"get_cgit_type_url
  },
  mappings = "<leader>gy" -- mapping to call url generation
})
```

When configuring `gitlinker.nvim`, you don’t need to copy-paste the
above, you just need to override/add what you want.

### callbacks

Besides the already configured hosts in the `callbacks` table, one can
add support for other git web hosts or self-hosted and enterprise
instances.

In the key, place a string with the hostname and in value a callback
function that constructs the url and receives:

``` lua
url_data = {
  host = "<host.tld>",
  repo = "<user/repo>",
  file = "<path/to/file/from/repo/root>",
  lstart = 42, -- the line start of the selected range / current line
  lend 57, -- the line end of the selected range
}
```

`lstart` and `lend` can be `nil` in case normal mode was used or
`opts.add_current_line_on_normal_mode = false`. Do not forget to check
for that in your callback.

As an example, here is the callback for github (**you don’t need this,
it’s already builtin**, it’s just an example):

``` lua
callbacks = {
  ["github.com"] = function(url_data)
      local url = "https://" .. url_data.host .. "/" .. url_data.repo .. "/blob/" ..
                      url_data.rev .. "/" .. url_data.file
      if url_data.lstart then
          url = url .. "#L" .. url_data.lstart
          if url_data.lend then url = url .. "-L" .. url_data.lend end
      end
      return url
    end
}
```

If you want to add support for your company’s gitlab instance:

``` lua
callbacks = {
  ["git.seriouscompany.com"] = require"gitlinker.hosts".get_gitlab_type_url
}
```

Here is my personal configuration for my personal self-hosted gitea
instance for which the `host` is a local one (since I can only access it
from my LAN) and but the web interface is public:

``` lua
callbacks = {
  ["192.168.1.2"] = function(url_data)
      url_data.host = "git.ruimarques.xyz"
      return
          require"gitlinker.hosts".get_gitea_type_url(url_data)
    end
}
```

### Options

- `remote`

If `remote = nil` (default), the relevant remote will be auto-detected.
If you have multiple git remotes configured and want to use a specific
one (e.g. `myfork`), do `remote = "myfork"`.

- `add_current_line_on_normal_mode`

If `true`, when invoking the mapping/command in normal mode, it adds the
current line to the url.

- `action_callback`

A function that receives a url string and decides which action to take.
By default set to `require"gitlinker.actions".copy_to_clipboard` which
copies to generated url to your system clipboard.

An alternative callback `require"gitlinker.actions".open_in_browser` is
provided which opens the url in your preferred browser using `xdg-open`
(linux only).

You can define your own action callback.

- `print_url`

If `true`, then print the url before performing the configured action.

- `mappings`

A string representing the keys you wish to map these plugin’s actions.
By default, a normal and a visual mapping is set up for `<leader>gy`.

**To disable mappings** just set `mappings = nil`.

If you want to disable mappings and set them on your own, the function
you are looking for is
`require"gitlinker".get_buf_range_url(mode, user_opts)` where `mode` is
the either `"n"` (normal) or `"v"` (visual) and `user_opts` is a table
of opts similar to the one passed in `setup()` (it can be `nil`, or not
passed), only `mode` is mandatory.

## Contributing

- Do you want to add support for another git web host?
- Another action callback?
- Fix a bug?
- Improve performance?
- Refactor?
- Improve docs?

All contributions are welcome, feel free to open a pull request.

Please read [CONTRIBUTING.md](./CONTRIBUTING.md) for general
contributing guidelines for this project.
