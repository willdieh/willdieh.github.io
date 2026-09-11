This public repository serves as the source for my Github Pages personal blog.
Theme settings are found in _config.yml
Using Minima - the default, blogger friendly, Github Pages theme

## Some *claude* generated MD tips:

**1. Headers — use `#` symbols, and skip levels sparingly**
```markdown
# Title (usually skip this — front matter title already renders as H1)
## Section
### Subsection
```
Start your actual content at `##`, since Minima already renders `page.title` as an H1 above your content. Using `#` again inside the post creates a second, duplicate-looking H1.

**2. Lists**
```markdown
- Bullet one
- Bullet two
  - Nested bullet (indent 2 spaces)

1. Numbered step one
2. Numbered step two
```

**3. Emphasis**
```markdown
**bold** for important terms
*italic* for subtle emphasis or first-use terminology
```
Don't overuse both together (`***bold italic***`) — reads as shouting.

**4. Links**
```markdown
[link text](https://example.com)
```
Write descriptive link text ("see the WinRT documentation") rather than bare URLs or "click here" — this also matters for accessibility (screen readers announce link text out of context).

**5. Blockquotes** — for quoting a source or highlighting a note:
```markdown
> This is a quoted or highlighted passage.
```

**6. Horizontal rule** — a thin line to separate distinct sections:
```markdown
---
```
Careful: this must have a blank line before and after, or Jekyll may confuse it with front matter delimiters if placed near the top of a file.

**7. Tables** — only when data genuinely benefits from rows/columns (per general accessibility guidance, don't use tables just for layout):
```markdown
| Feature | Minima | Chirpy |
|---------|--------|--------|
| Search  | No     | Yes    |
```

**General habits that'll save you trouble:**
- One blank line between every block (paragraph, heading, list, code block) — Markdown parsers are picky about this.
- Keep headers in a logical hierarchy (`##` then `###`, not jumping straight to `####`).
- Write alt text for every image, always — it's both good practice and an accessibility requirement.
- Preview before publishing when possible, since Markdown's simplicity means small mistakes (a missing blank line, mismatched backticks) can silently break rendering rather than throwing an error.


## Code specific MD tips:

Here's what you need for code-heavy posts in Markdown:

**1. Inline code** — for short snippets mentioned in a sentence, wrap in single backticks:
```markdown
Use the `winrt::init_apartment()` function to initialize.
```
Renders as: Use the `winrt::init_apartment()` function to initialize.

**2. Code blocks** — for multi-line code, use triple backticks, with the language name right after the opening backticks for syntax highlighting:

````markdown
```cpp
winrt::init_apartment();
auto result = SomeAsyncCall().get();
```
````

Jekyll uses Rouge for highlighting and recognizes most common language names: `cpp`, `csharp`, `python`, `bash`, `xml`, `json`, `html`, `yaml`, etc. 

**3. Showing a filename or command context** — there's no official Markdown syntax for a "code block title," but a common convention is a bolded line just above the block:
````markdown
**`main.cpp`**
```cpp
// code here
```
````

**4. Long lines:** code blocks don't wrap by default — they scroll horizontally on narrow screens. For a blog, keep lines under ~80 characters where practical so mobile readers aren't stuck scrolling.
