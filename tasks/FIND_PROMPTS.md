# Action Plan: Find Missing Prompts and Update README

## Purpose
Template for finding missing prompt files and ensuring all prompts in the `/prompts/` directory are documented in README.md.

## Prerequisites
- Access to repository file system
- README.md with "Prompt Engineering Templates" section

## Step-by-Step Process

### 1. Inventory Existing Prompts
```bash
# List all prompt files
ls prompts/
# Or use Glob tool: prompts/**/*.md
```

### 2. Extract Current README Documentation
```bash
# Read "Prompt Engineering Templates" section in README.md
# Note all documented prompts and their paths
```

### 3. Compare and Identify Gaps
- Create list of filesystem prompts
- Create list of README documented prompts
- Identify missing prompts: `filesystem_prompts - documented_prompts`

### 4. Add Missing Prompts to README
For each missing prompt:
- Determine appropriate category (Content Creation, Audio/Video, Engineering)
- Read prompt file to understand purpose
- Add entry with format:
  ```markdown
  *   **[Prompt Name](./prompts/filename.md)**: Brief description of what it does
  ```

### 5. Verification
- Ensure all filesystem prompts are documented
- Verify all paths are correct
- Check descriptions are accurate

## Expected Outcomes
- Complete documentation coverage
- All prompts discoverable via README
- Consistent formatting across entries

## Common Issues to Watch For
- Typos in file paths
- Missing or unclear descriptions
- Incorrect categorization
- Inconsistent formatting