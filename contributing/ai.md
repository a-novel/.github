# AI usage

This is generic documentation about AI usage across A-Novel repositories.

Using AI assistants (ChatGPT, Claude, Copilot, etc.) is allowed. They can significantly speed up development when used
appropriately. However, AI-generated code is still **your responsibility** — you own it, you maintain it, you debug it.

## Core Principles

- **Skills are yours, not AI**: a bad coder will produce bad code, with or without AI assistance. We expect all
  the code you produce to be your own, i.e. code that you fully understand down to the smallest detail. If your output
  is not worth more than AI, your very participation is worthless to us.
- **Never generate code you can't review**: we don't enforce any rules about the scope of AI assistance. Generating
  basic code, test suites, or drafting more complex logic are all valid use cases. However, the rules that apply to
  human-written code also apply to AI-generated code. Your updates should be concise, documented, and to the point.
- **Don't**: refactor large parts of the codebase at once, replace well-maintained libraries with AI-generated helpers
  (unless you thoroughly review them)

## Guidelines

AI assistants are just like humans. The more precise and scoped your prompt is, the better the results and the less
likely your model is to make mistakes.

Don't ask `Do this and that`. Instead, use separate prompts for each step, and review thoroughly in between.

Another important thing is to **NEVER ALLOW ALL MODIFICATIONS / COMMANDS** during a session. Always make sure your
assistant asks for permissions each time and review the commands / changes before they are executed. If you don't
understand a command, or a proposed change seems strange to you, refuse it. Most assistants let you add explanations
when doing so.
