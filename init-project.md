# /init-project — Initialize or bootstrap a project

You are executing the `/init-project` skill. Follow these instructions precisely.

## Project name

The user provided: **$ARGUMENTS**

If `$ARGUMENTS` is empty, you MUST ask the user for the project name before proceeding.

## Instructions

1. Read the full workflow from the file at this path:
   `~/.claude/commands/init-project-templates/skill.md`

2. The templates referenced in the workflow are located at:
   `~/.claude/commands/init-project-templates/`

   Specifically:
   - `~/.claude/commands/init-project-templates/CLAUDE_MD.template.md`
   - `~/.claude/commands/init-project-templates/MEMORY_MD.template.md`
   - `~/.claude/commands/init-project-templates/settings.local.json.template`
   - `~/.claude/commands/init-project-templates/settings.json.template`
   - `~/.claude/commands/init-project-templates/feature-doc.template.md`
   - `~/.claude/commands/init-project-templates/gitignore.template`
   - `~/.claude/commands/init-project-templates/PRD/*.md.template`

3. Execute the workflow step by step (Etape 0 through Etape 10), using the templates directory above as the base path for all template reads.

4. The working directory for file generation is the **current project directory** (where the user invoked the command).

## Important

- Read the `skill.md` file FIRST — it contains the complete workflow with 11 steps.
- All template paths in `skill.md` that reference `./templates/` should be resolved to `~/.claude/commands/init-project-templates/`.
- Never skip the user confirmation steps defined in the workflow.
