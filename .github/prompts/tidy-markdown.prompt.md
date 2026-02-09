---
agent: agent
---

Review the specified Markdown file and apply the following formatting cleanup:

1. **Remove trailing whitespace** - Trim any spaces or tabs at the end of lines
2. **Standardize blank lines** - Replace multiple consecutive blank lines with a single blank line
3. **Fix heading spacing** - Ensure there's exactly one blank line before headings (except at the start of a file) and one blank line after headings
4. **Standardize list formatting** - Ensure consistent spacing after list markers (-, *, or numbers)
5. **Fix inline code** - Ensure backticks are properly placed around code elements, file paths, commands, and technical terms
6. **Clean up link formatting** - Remove unnecessary spaces in Markdown link syntax [text](url)
7. **Ensure file ends with newline** - Each file should end with exactly one newline character
8. **Preserve intentional formatting** - Don't modify code blocks (content between ``` markers) or HTML elements

Apply these changes while preserving the semantic meaning and structure of the content.
