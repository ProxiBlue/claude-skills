# Use the Magento Skills in Claude

This folder contains specialized Magento 2 development skills that extend Claude Code's capabilities with domain-specific knowledge for Magento 2 / Mage-OS development, particularly for Hyvä Themes projects. They have been created specifically during an ongoing task for a complex magento 1 to magento 2 migration project, to allow claude to migrate code more consistently and using Hyva conventions. It is an ongoing Work In Progress

There are some environment specific entries, for example paths to exampel files which is my dev environment. I don;t have enough knowledge yet on skills definition to know how to make that more dynamic. so check before use.


## What Are Skills?

Skills are reusable, context-aware prompts that guide Claude through complex, multi-step tasks. Each skill is stored in its own folder with a `SKILL.md` file containing specialized instructions, examples, and best practices.

## Available Magento Skills

### 1. **analyze-m1-module-for-migration**
Systematically analyzes a Magento 1 module to determine its purpose, usage, and migration requirements for Magento 2. Helps decide whether to migrate, find alternatives, or skip the module.

**Use when:** Planning M1 to M2 migration, evaluating legacy modules

### 2. **create-backend-controller**
Creates a backend (adminhtml) controller action in Magento 2 with proper ACL, routing, authorization, and admin UI integration.

**Use when:** Building admin pages, AJAX endpoints, form handlers, or mass actions

### 3. **create-frontend-controller**
Creates a frontend controller action in Magento 2 for the storefront with proper routing, dependency injection, and response handling.

**Use when:** Building custom frontend pages, AJAX endpoints, form submission handlers, or API-like endpoints for JavaScript

### 4. **hyva-tailwind-integration**
Comprehensive guidance on integrating Tailwind CSS and JavaScript in Hyvä Themes, including configuration merging, module registration, and build processes.

**Use when:** Adding custom Tailwind styles, creating new Hyvä modules, or debugging Tailwind builds

### 5. **magento-controller-refactor**
Scans and refactors deprecated Magento 2 controller patterns to modern HTTP verb interfaces. Updates controllers that extend deprecated `Action` base class to PHP 8.3+ compatible patterns.

**Use when:** Modernizing controllers, upgrading to PHP 8.3+, or fixing deprecation warnings

## How to Check If Skills Are Loaded

### Method 1: Use the Skill Tool (Direct Check)
In Claude Code, simply type:
```
list skills
```

You should see output listing all available skills with their descriptions.

### Method 2: Try Using a Skill
Simply ask Claude to use a skill by name:
```
Use the hyva-tailwind-integration skill
```

If the skill is loaded, Claude will activate it and follow its instructions. If not loaded, Claude will indicate the skill is not available.

### Method 3: Check the File System
Skills are loaded automatically if they exist in the `.claude/skills/` directory. Each skill must have:
- A folder with a kebab-case name (e.g., `hyva-tailwind-integration`)
- A `SKILL.md` file inside that folder

You can verify this structure with:
```bash
ls -la .claude/skills/
```

## How to Load These Skills Into Any Claude Instance

### Option 1: Copy Skills to Another Project (Recommended)
1. Copy the entire `.claude/skills/` folder to your target project:
   ```bash
   cp -r /path/to/ntotank/.claude/skills /path/to/your-project/.claude/
   ```

2. Claude Code will automatically detect and load all skills when you start a session in that project.

3. Verify by running `list skills` in Claude Code.

### Option 2: Use Skills as a Git Submodule
If you want to keep skills synchronized across multiple projects:

1. Initialize the skills folder as a Git repository (if not already done):
   ```bash
   cd .claude/skills
   git init
   git add .
   git commit -m "Initial skills repository"
   ```

2. Push to a remote repository (e.g., GitHub):
   ```bash
   git remote add origin <your-skills-repo-url>
   git push -u origin main
   ```

3. In other projects, add as a submodule:
   ```bash
   cd /path/to/other-project
   mkdir -p .claude
   git submodule add <your-skills-repo-url> .claude/skills
   git submodule update --init --recursive
   ```

### Option 3: Symlink for Local Development
If you work on multiple Magento projects locally:

1. Keep skills in a central location:
   ```bash
   mkdir -p ~/magento-claude-skills
   cp -r .claude/skills/* ~/magento-claude-skills/
   ```

2. Create symlinks in each project:
   ```bash
   cd /path/to/other-project
   mkdir -p .claude
   ln -s ~/magento-claude-skills .claude/skills
   ```

## How to Use Skills

### Direct Invocation
Simply ask Claude to use the skill by name:
```
Use the hyva-tailwind-integration skill to help me add custom button styles
```

### Implicit Invocation
Claude can automatically invoke skills when it detects relevant context:
```
I need to create a backend controller for managing custom product attributes
```
Claude may automatically use the `create-backend-controller` skill.

### With Specific Context
Provide additional context for better results:
```
Use the analyze-m1-module-for-migration skill to analyze the Uptactics_TextReplacement module at /path/to/m1/module
```

## Skill File Structure

Each skill follows this structure:

```
.claude/skills/
├── skill-name/
│   └── SKILL.md          # Main skill instructions
└── README.md             # This file
```

The `SKILL.md` file contains:
- **Purpose**: What the skill does
- **When to Use**: Triggering conditions
- **Instructions**: Step-by-step guidance
- **Examples**: Code samples and patterns
- **Best Practices**: Domain-specific recommendations

## Creating Your Own Skills

To create a custom skill:

1. Create a new folder in `.claude/skills/`:
   ```bash
   mkdir -p .claude/skills/my-custom-skill
   ```

2. Create a `SKILL.md` file with your instructions:
   ```bash
   touch .claude/skills/my-custom-skill/SKILL.md
   ```

3. Write clear, actionable instructions in the `SKILL.md` file:
   ```markdown
   # My Custom Skill

   ## Purpose
   Describe what this skill does

   ## When to Use
   Specify triggering conditions

   ## Instructions
   Step-by-step guidance...
   ```

4. Restart Claude Code or start a new conversation to load the skill.

## Troubleshooting

### Skills Not Loading
- **Check file structure**: Ensure each skill has a `SKILL.md` file in its folder
- **Check file permissions**: Skills folder must be readable by Claude Code
- **Restart Claude**: Start a new conversation to reload skills
- **Verify location**: Skills must be in `.claude/skills/` relative to project root

### Skills Not Working as Expected
- **Review SKILL.md**: Ensure instructions are clear and actionable
- **Provide context**: Give Claude enough information to apply the skill correctly
- **Check dependencies**: Some skills may require specific files or configurations to exist

### Skill Conflicts
- **Naming**: Ensure skill folder names are unique and kebab-case
- **Scope**: Keep skill instructions focused on a single responsibility
- **Precedence**: If multiple skills could apply, explicitly name the one you want

## Contributing

To improve or add new Magento skills:

1. Follow the existing folder structure
2. Write clear, actionable instructions
3. Include examples and best practices
4. Test the skill with various scenarios
5. Document any dependencies or prerequisites

## Resources

- [Claude Code Documentation](https://docs.claude.com/en/docs/claude-code)
- [Magento 2 Developer Documentation](https://developer.adobe.com/commerce/php/development/)
- [Hyvä Themes Documentation](https://docs.hyva.io/)
- [Mage-OS Documentation](https://mage-os.org/)

---

**Note:** These skills are specifically designed for Magento 2 / Mage-OS development with Hyvä Themes. They may reference project-specific patterns and conventions from the NTOTanks project.
