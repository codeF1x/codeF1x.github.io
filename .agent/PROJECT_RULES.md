# Project Rules for My Blog (Hugo)

## 1. Content Management

- **Language**: Default language is `zh-cn`. English content goes to `content/en`.
- **Frontmatter**: Always include `title`, `date`, `categories`, and `tags`.
- **Auto-Frontmatter**: When creating or editing a `.md` file in the `content/` directory, if Hugo frontmatter (TOML `+++` or YAML `---`) is missing, you MUST automatically generate and prepend it. Use the current date and infer title/tags from the content or filename.
- **Images**: Store images in `static/img/` and reference them as `/img/filename.jpg`.

## 2. Technical Stack

- **Theme**: Using the `stack` theme.
- **Build**: Use `hugo` to build, `hugo server -D` for development.

## 3. Antigravity Specifics

- **Verification**: After adding a new post, run `hugo` to ensure no build errors.
- **Formatting**: Use standard Markdown.
