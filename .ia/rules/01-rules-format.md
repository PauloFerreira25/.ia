# Rules Format

When creating a new rule:
- Identify the specific problem the rule solves
- Create a file named `XX-descriptive-name.md` with the next available number
- Use the imperative format: "When [situation]: Do X / Never do Y / If Z, then W"
- Register it in `00-index.md`

When writing rule content:
- Keep it minimal and direct
- Use "When / Do / Never / If" structure
- No prose, no explanations of why — just what to do
- Update examples when they become outdated

When rules overlap:
- Keep detailed content in one file
- Reference from others with "See XX-filename.md"
- Never duplicate content across files

Never remove content without explicit user confirmation.

## Numbering
- `01-29`: Base rules from this repository
- `30+`: Project-specific rules (added per project)
