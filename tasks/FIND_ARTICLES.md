# Action Plan: Find Missing Technical Guides and Update README

## Purpose
Template for finding missing technical guide files and ensuring all guides in the `/technical_guides/` directory are documented in README.md.

## Prerequisites
- Access to repository file system
- README.md with "In-Depth Technical Guides" section

## Step-by-Step Process

### 1. Inventory Existing Technical Guides
```bash
# List all technical guide files recursively
# Use Glob tool: technical_guides/**/*.md
```

### 2. Extract Current README Documentation
- Read "In-Depth Technical Guides" section in README.md
- Note all documented guides and their paths
- Pay attention to technology categories (AI, Rust, PostgreSQL, etc.)

### 3. Compare and Identify Gaps
- Create list of filesystem guides
- Create list of README documented guides  
- Identify missing guides: `filesystem_guides - documented_guides`
- Check for path errors and inconsistencies

### 4. Add Missing Guides to README
For each missing guide:
- Determine appropriate technology category
- Read guide file to understand content and scope
- Add entry with format:
  ```markdown
  *   [Guide Title](./technical_guides/category/filename.md)
      *   *Core Concepts: Key topics covered in the guide.*
  ```

### 5. Fix Path and Content Errors
Common issues to check:
- Typos in file paths (e.g., `gpostresql` → `postgresql`)
- Incorrect link paths in README
- Duplicate entries with wrong information
- Mismatched titles vs actual file content

### 6. Verification
- Ensure all filesystem guides are documented
- Verify all paths are correct and files exist
- Check descriptions accurately reflect guide content
- Confirm no broken links

## Technology Categories to Check
- **AI**: Machine learning, agents, frameworks
- **Rust**: Language fundamentals, patterns, applications
- **PostgreSQL**: Database concepts, administration, HA
- **React**: Frontend frameworks, state management
- **CSS**: Styling frameworks, design systems
- **Kafka**: Event streaming, fault tolerance
- **Observability**: Monitoring, logging, metrics

## Expected Outcomes
- Complete documentation coverage
- All technical guides discoverable via README
- Accurate paths and descriptions
- Consistent formatting across entries

## Common Issues to Watch For
- Typos in file paths or guide titles
- Missing or inaccurate descriptions
- Broken internal links
- Inconsistent formatting
- Duplicate or conflicting entries