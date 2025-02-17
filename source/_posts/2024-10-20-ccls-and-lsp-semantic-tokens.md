---
layout: post
title: ccls and LSP Semantic Tokens
author: MaskRay
tags: [ccls]
---

I've spent countless hours writing and reading C++ code. For many years, Emacs has been my primary editor, and I leverage [ccls](https://github.com/MaskRay/ccls/)' (my C++ language server) rainbow semantic highlighting feature.

The feature relies on two custom notification messages `$ccls/publishSemanticHighlight` and `$ccls/publishSkippedRanges`.
`$ccls/publishSemanticHighlight` provides a list of symbols, each with kind information (function, type, or variable) of itself and its semantic parent (e.g. a member function's parent is a class), storage duration, and a list of ranges.

```cc
struct CclsSemanticHighlightSymbol {
  int id = 0;
  SymbolKind parentKind;
  SymbolKind kind;
  uint8_t storage;
  std::vector<std::pair<int, int>> ranges;

  std::vector<lsRange> lsRanges; // Only used by vscode-ccls
};

struct CclsSemanticHighlight {
  DocumentUri uri;
  std::vector<CclsSemanticHighlightSymbol> symbols;
};
```

An editor can use consistent colors to highlight different occurrences of a symbol.
Different colors can be assigned to different symbols.

Tobias Pisani created emacs-cquery (the predecessor to emacs-ccls) in Nov 2017.
Despite not being a fan of Emacs Lisp, I added the rainbow semantic highlighting feature for my own use in early 2018.
My setup also relied heavily on these two settings:

* Bolding and underlining variables of static duration storage
* Italicizing member functions and variables

```lisp
(setq ccls-sem-highlight-method 'font-lock)
(ccls-use-default-rainbow-sem-highlight)
```

Key symbol properties (member, static) were visually prominent in my Emacs environment.

![](/static/2024-10-20-ccls-and-lsp-semantic-tokens/emacs-ccls.webp)

My Emacs hacking days are a distant memory – beyond basic configuration tweaks, I haven't touched elisp code since 2018.
As my Elisp skills faded, I increasingly turned to Neovim for various editing tasks.
Naturally, I wanted to migrate my C++ development workflow to Neovim as well.
However, a major hurdle emerged: Neovim lacked the beloved rainbow highlighting I enjoyed in Emacs.

Thankfully, Neovim supports "semantic tokens" from LSP 3.16, a standardized approach adopted by many editors.

I've made changes to ccls (available on [a branch](https://github.com/MaskRay/ccls/tree/semantic-tokens); [PR](https://github.com/MaskRay/ccls/pull/972)) to support semantic tokens.
This involves adapting the `$ccls/publishSemanticHighlight` code to additionally support `textDocument/semanticTokens/full` and `textDocument/semanticTokens/range`.

I utilize a few token modifiers (`static`, `classScope`, `functionScope`, `namespaceScope`) for highlighting:

```lua
vim.cmd([[
hi @lsp.mod.classScope.cpp gui=italic
hi @lsp.mod.static.cpp gui=bold
hi @lsp.typemod.variable.namespaceScope.cpp gui=bold,underline
]])
```

![treesitter, tokyonight-moon](/static/2024-10-20-ccls-and-lsp-semantic-tokens/neovim-semantic-tokens-basic.webp)

While this approach is a significant improvement over relying solely on nvim-treesitter, I'm still eager to implement rainbow semantic tokens.
Although LSP semantic tokens don't directly distinguish symbols, we can create custom modifiers to achieve similar results.

```javascript
tokenModifiers: {
  "declaration", "definition", "static", ...

  "id0", "id1", ... "id9",
}
```

In the user-provided initialization options, I set `highlight.rainbow` to 10.

ccls assigns the same modifier ID to tokens belonging to the same symbol, aiming for unique IDs for different symbols.
While we only have a few predefined IDs (each linked to a specific color), there's a slight possibility of collisions.
However, this is uncommon and generally acceptable.

For a token with type `variable`, Neovim's built-in LSP plugin assigns a highlight group `@lsp.typemod.variable.id$i.cpp` where `$i` is an integer between 0 and 9.
This allows us to customize a unique foreground color for each modifier ID.

```lua
local func_colors = {
  '#e5b124', '#927754', '#eb992c', '#e2bf8f', '#d67c17',
  '#88651e', '#e4b953', '#a36526', '#b28927', '#d69855',
}
local type_colors = {
  '#e1afc3', '#d533bb', '#9b677f', '#e350b6', '#a04360',
  '#dd82bc', '#de3864', '#ad3f87', '#dd7a90', '#e0438a',
}
local param_colors = {
  '#e5b124', '#927754', '#eb992c', '#e2bf8f', '#d67c17',
  '#88651e', '#e4b953', '#a36526', '#b28927', '#d69855',
}
local var_colors = {
  '#429921', '#58c1a4', '#5ec648', '#36815b', '#83c65d',
  '#419b2f', '#43cc71', '#7eb769', '#58bf89', '#3e9f4a',
}
local all_colors = {
  class = type_colors,
  constructor = func_colors,
  enum = type_colors,
  enumMember = var_colors,
  field = var_colors,
  ['function'] = func_colors,
  method = func_colors,
  parameter = param_colors,
  struct = type_colors,
  typeAlias = type_colors,
  typeParameter = type_colors,
  variable = var_colors
}
for type, colors in pairs(all_colors) do
  for i = 1,#colors do
    for _, lang in pairs({'c', 'cpp'}) do
      vim.api.nvim_set_hl(0, string.format('@lsp.typemod.%s.id%s.%s', type, i-1, lang), {fg=colors[i]})
    end
  end
end

vim.cmd([[
hi @lsp.mod.classScope.cpp gui=italic
hi @lsp.mod.static.cpp gui=bold
hi @lsp.typemod.variable.namespaceScope.cpp gui=bold,underline
]])
```

Now, let's analyze the C++ code above using this configuration.

![tokyonight-moon](/static/2024-10-20-ccls-and-lsp-semantic-tokens/neovide-semantic-tokens.webp)

While the results are visually pleasing, I need help implementing code lens functionality.

## Inactive code highlighting

Inactive code regions (skipped ranges in Clang) are typically displayed in grey.
While this can be helpful for identifying unused code, it can sometimes hinder understanding the details.
I simply disabled the inactive code feature.

```c
#ifdef X
... // colorful
#else
... // normal instead of grey
#endif
```

## Refresh

When opening a large project, the initial indexing or cache loading process can be time-consuming, often leading to empty lists of semantic tokens for the initially opened files.
While ccls prioritizes indexing these files, it's unclear how to notify the client to refresh the files.
The existing `workspace/semanticTokens/refresh` request, unfortunately, doesn't accept text document parameters.

In contrast, with `$ccls/publishSemanticHighlight`, ccls proactively sends the notification after an index update (see `main_OnIndexed`).

```cpp
void main_OnIndexed(DB *db, WorkingFiles *wfiles, IndexUpdate *update) {
  ...

  db->applyIndexUpdate(update);

  // Update indexed content, skipped ranges, and semantic highlighting.
  if (update->files_def_update) {
    auto &def_u = *update->files_def_update;
    if (WorkingFile *wfile = wfiles->getFile(def_u.first.path)) {
      wfile->setIndexContent(g_config->index.onChange ? wfile->buffer_content
                                                      : def_u.second);
      QueryFile &file = db->files[update->file_id];
      // Publish notifications to the file.
      emitSkippedRanges(wfile, file);
      emitSemanticHighlight(db, wfile, file);
      // But how do we send a workspace/semanticTokens/refresh request?????
    }
  }
}
```

While the semantic token request supports partial results in the specification, Neovim lacks this implementation.
Even if it were, I believe a notification message with a text document parameter would be a more efficient and direct approach.

```typescript
export interface SemanticTokensParams extends WorkDoneProgressParams,
	PartialResultParams {
	/**
	 * The text document.
	 */
	textDocument: TextDocumentIdentifier;
}
```

## Other clients

### [emacs-ccls](https://github.com/emacs-lsp/emacs-ccls)

Once this feature branch is merged, Emacs users can simply remove the following lines:

```lisp
(setq ccls-sem-highlight-method 'font-lock)
(ccls-use-default-rainbow-sem-highlight)
```

How to change `lsp-semantic-token-modifier-faces` to support rainbow semantic tokens in lsp-mode and emacs-ccls?

The general approach is similar to the following, but we need a feature from lsp-mode (<https://github.com/emacs-lsp/lsp-mode/issues/4590>).

```lisp
(setq lsp-semantic-tokens-enable t)
(defface lsp-face-semhl-namespace-scope
         '((t :weight bold)) "highlight for namespace scope symbols" :group 'lsp-semantic-tokens)
(cl-loop for color in '("#429921" "#58c1a4" "#5ec648" "#36815b" "#83c65d"
                        "#417b2f" "#43cc71" "#7eb769" "#58bf89" "#3e9f4a")
       for i = 0 then (1+ i)
       do (custom-declare-face (intern (format "lsp-face-semhl-id%d" i))
                               `((t :foreground ,color))
                               "" :group 'lsp-semantic-tokens))
(setq lsp-semantic-token-modifier-faces
      `(("declaration" . lsp-face-semhl-interface)
        ("definition" . lsp-face-semhl-definition)
        ("implementation" . lsp-face-semhl-implementation)
        ("readonly" . lsp-face-semhl-constant)
        ("static" . lsp-face-semhl-static)
        ("deprecated" . lsp-face-semhl-deprecated)
        ("abstract" . lsp-face-semhl-keyword)
        ("async" . lsp-face-semhl-macro)
        ("modification" . lsp-face-semhl-operator)
        ("documentation" . lsp-face-semhl-comment)
        ("defaultLibrary" . lsp-face-semhl-default-library)
        ("classScope" . lsp-face-semhl-member)
        ("namespaceScope" . lsp-face-semhl-namespace-scope)
        ,@(cl-loop for i from 0 to 10
                   collect (cons (format "id%d" i)
                                 (intern (format "lsp-face-semhl-id%d" i))))
        ))
```

### [vscode-ccls](https://github.com/MaskRay/vscode-ccls/)

We require assistance to eliminate the `$ccls/publishSemanticHighlight` feature and adopt built-in semantic tokens support.
Due to the lack of active maintenance for vscode-ccls, I'm unable to maintain this plugin for an editor I don't frequently use.

## Misc

I use a trick to switch ccls builds without changing editor configurations.

```zsh
#!/bin/zsh
#export CCLS_TRACEME=s
export LD_PRELOAD=/usr/lib/libmimalloc.so

type=
[[ -f /tmp/ccls-build ]] && type=$(</tmp/ccls-build)

case $type in
  strace)
    exec strace -s999 -e read,write -o /tmp/strace.log -f ~/ccls/out/debug/ccls --log-file=/tmp/cc.log -v=1 "$@";;
  debug)
    exec ~/ccls/out/debug/ccls --log-file=/tmp/cc.log -v=2 "$@";;
  release)
    exec ~/ccls/out/release/ccls --log-file=/tmp/cc.log -v=1 "$@";;
  *)
    exec /usr/bin/ccls --log-file=/tmp/cc.log -v=1 "$@";;
esac
```

Usage:

```sh
echo debug > /tmp/ccls-build
nvim  # out/debug/ccls is now used
```
