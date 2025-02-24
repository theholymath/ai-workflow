# Mise and LLM Codebase Improvement Workflow

This document provides a detailed guide to using the `mise` task runner along with LLM tools to analyze, improve, and maintain your codebase. The workflow combines repository analysis, LLM-powered code review, test coverage improvement, and documentation generation into a systematic approach for codebase enhancement.

## Setup and Installation

### Prerequisites

1. **Install mise**
   ```bash
   # Using Homebrew (macOS/Linux)
   brew install mise
   
   # Using the installer script
   curl https://mise.run | sh
   ```

2. **Install repomix**
   ```bash
   # Using pip
   pip install repomix
   
   # Using npm
   npm install -g repomix
   ```

3. **Install LLM command-line tool**
   ```bash
   # Using pip
   pip install llm
   
   # Configure with your API keys
   llm keys set openai
   # Or for other providers
   llm keys set anthropic
   ```

4. **Install Aider (optional, for automated fixes)**
   ```bash
   pip install aider-chat
   ```

### Configuration Setup

Create a `.mise.toml` file in your project root with the following content:

```toml
[env]
# Set your preferred model here to use across all tasks
model = "anthropic/claude-3-sonnet"

[tasks.LLM.clean_bundles]
description = "Generate LLM bundle output file using repomix"
run = "repomix -o repo_output.txt"

[tasks.LLM.copy_buffer_bundle]
description = "Copy generated LLM bundle to clipboard"
run = "cat repo_output.txt | pbcopy"  # Use xclip for Linux

[tasks.LLM.generate_code_review]
description = "Generate code review from repository content"
run = "cat repo_output.txt | llm --system \"You are an expert software architect performing a thorough code review. Focus on architecture, patterns, potential bugs, and maintainability issues. Organize your review by file with clear headings.\" \"Perform a comprehensive code review of this repository. Highlight strengths, weaknesses, bugs, and opportunities for improvement. Include specific code references when possible.\" > code_review.md"

[tasks.LLM.generate_missing_tests]
description = "Generate suggestions for missing tests"
run = "cat repo_output.txt | llm --system \"You are an expert in test-driven development and test coverage analysis. Focus on identifying untested code and suggesting specific test cases with clear implementation notes.\" \"Analyze this codebase and identify areas lacking test coverage. For each area, create a separate section with a specific title for the missing test, detailed description of what should be tested, and sample test code where appropriate. Format each missing test as a separate task that could be implemented independently.\" > missing_tests.md"

[tasks.LLM.generate_readme]
description = "Generate a README.md from repository content"
run = "cat repo_output.txt | llm --system \"You are a technical documentation expert. Focus on clarity, completeness, and usability. Structure your documentation with clear sections, examples, and setup instructions.\" \"Generate a comprehensive README.md for this repository. Include project overview, setup instructions, usage examples, architecture explanation, and API documentation where relevant. Use proper markdown formatting with headings, lists, and code blocks.\" > README.md"
```

For global configuration, you can add these to `~/.config/mise/config.toml` instead.

## Detailed Workflow Steps

### 1. Codebase Bundling and Analysis

**Command**: `mise run LLM:clean_bundles`

**What happens**:
- `repomix` scans your repository and creates a single consolidated file (`repo_output.txt`) containing your code
- This process intelligently handles:
  - Directory structures
  - File relationships
  - Comments and documentation
  - Ignores files specified in `.gitignore`
  - Excludes binaries, large files, and other non-relevant content
- The output file is optimized for LLM consumption, with headers and structure that helps the model understand the codebase organization
- The file is saved as `repo_output.txt` in your project root

**Why this matters**:
- Creates a standardized, machine-readable version of your codebase
- Significantly reduces token usage by focusing only on relevant code
- Maintains sufficient context for LLMs to understand the overall project structure
- Provides a consistent snapshot that can be referenced by multiple LLM tasks

**Troubleshooting**:
- If `repo_output.txt` is too large, customize repomix arguments to exclude more files or directories
- If important files are missing, check your `.gitignore` and repomix configuration

### 2. Generating Code Review

**Command**: `mise run LLM:generate_code_review`

**What happens**:
- The bundled codebase is passed to the LLM tool with a specialized system prompt
- The system prompt sets the LLM in the role of an expert software architect
- The LLM analyzes the entire codebase, looking for:
  - Architectural patterns and anti-patterns
  - Code quality issues
  - Potential bugs and edge cases
  - Performance concerns
  - Security vulnerabilities
  - Style and consistency issues
- The analysis is structured by file/module with clear headings
- The output is saved as `code_review.md` in your project root

**Why this matters**:
- Provides an objective third-party review of your codebase
- Identifies issues that may be missed during development
- Creates a structured document that can be shared with the team
- Helps prioritize technical debt and refactoring efforts
- Serves as a learning tool for improving code quality

**How to use the output**:
1. Review the document carefully, looking for actionable feedback
2. Prioritize issues based on severity and impact
3. Create tickets or tasks for addressing each issue
4. Reference the code review in pull requests and team discussions

### 3. Identifying Missing Tests

**Command**: `mise run LLM:generate_missing_tests`

**What happens**:
- The bundled codebase is analyzed specifically for test coverage
- The system prompt positions the LLM as a testing expert
- The LLM identifies:
  - Functions and classes without tests
  - Edge cases that aren't covered
  - Integration points that need testing
  - Complex logic requiring thorough validation
- For each identified gap, the LLM creates:
  - A specific title for the missing test
  - Detailed description of what should be tested
  - Expected inputs and outputs
  - Sample test code where appropriate
- The output is saved as `missing_tests.md` in your project root

**Why this matters**:
- Improves code reliability through better test coverage
- Creates a clear roadmap for enhancing testing
- Breaks down testing work into manageable, focused tasks
- Provides implementation guidance for developers
- Serves as documentation for testing strategy

**How to use the output**:
1. Review the missing tests document
2. For each missing test:
   - Create a new git branch (e.g., `git checkout -b test/user-authentication`)
   - Implement the test following the guidance
   - Submit a focused PR for just that test
   - Merge and move to the next test
3. Use continuous integration to verify test coverage improvement

### 4. Generating Project README

**Command**: `mise run LLM:generate_readme`

**What happens**:
- The bundled codebase is analyzed to create comprehensive documentation
- The system prompt positions the LLM as a documentation expert
- The LLM generates a structured README with:
  - Project overview and purpose
  - Architecture explanation
  - Setup instructions
  - Usage examples
  - API documentation
  - Contributing guidelines
- The output is saved as `README.md` in your project root

**Why this matters**:
- Creates consistent, comprehensive documentation
- Improves project onboarding for new developers
- Makes the project more accessible to users and contributors
- Serves as a central reference point for project information
- Enhances project professionalism and usability

**How to use the output**:
1. Review the generated README for accuracy
2. Add any missing information or project-specific details
3. Consider expanding sections that need more detail
4. Commit the README to your repository
5. Update periodically as the project evolves

## Automated Code Improvements

For repetitive tasks like adding docstrings or comments, you can use Aider in batch mode:

```bash
#!/bin/bash

# Create a log directory for Aider outputs
mkdir -p aider_logs

# Process Python files
echo "Processing Python files for docstring improvements..."
for file in $(find . -name "*.py" -not -path "*/\.*" -not -path "*/venv/*"); do
    echo "Adding docstrings to $file"
    aider --message "Add comprehensive docstrings to all functions and classes in this file. Follow PEP 257 conventions. Include parameter descriptions, return types, and examples where helpful." "$file" > "aider_logs/$(basename "$file").log" 2>&1
    echo "Completed $file"
    # Small pause to avoid rate limiting
    sleep 2
done

echo "All files processed. See logs in aider_logs directory."
```

Save this as `improve_docstrings.sh`, make it executable (`chmod +x improve_docstrings.sh`), and run it when needed.

## Best Practices for This Workflow

1. **Focused, Atomic Changes**:
   - Create a new git branch for each specific improvement
   - Keep pull requests small and focused on one concern
   - Use clear, descriptive commit messages

2. **Continuous Integration**:
   - Run tests automatically on each PR
   - Enforce code quality checks
   - Verify documentation is up to date

3. **Iterative Improvement**:
   - Don't try to fix everything at once
   - Prioritize issues based on impact and effort
   - Celebrate incremental progress

4. **Team Collaboration**:
   - Share the code review with the team
   - Assign missing tests to different developers
   - Use pair programming for complex improvements

5. **Regular Maintenance**:
   - Re-run the analysis periodically (e.g., monthly)
   - Compare results to track improvement over time
   - Update documentation as the codebase evolves

## Advanced Customization

### Creating Custom Tasks

You can create additional custom tasks in your `.mise.toml`:

```toml
[tasks.LLM.generate_swagger_docs]
description = "Generate Swagger/OpenAPI documentation"
run = "cat repo_output.txt | llm --system \"You are an API documentation specialist.\" \"Generate Swagger/OpenAPI documentation for all API endpoints in this codebase. Include parameters, responses, and examples.\" > swagger_docs.md"

[tasks.LLM.security_audit]
description = "Perform a security audit on the codebase"
run = "cat repo_output.txt | llm --system \"You are a cybersecurity expert with experience in identifying security vulnerabilities in code.\" \"Perform a comprehensive security audit of this codebase. Identify potential vulnerabilities, classify them by severity, and provide remediation steps for each issue.\" > security_audit.md"
```

### Using Different Models for Different Tasks

You can override the default model for specific tasks:

```toml
[tasks.LLM.security_audit]
description = "Perform a security audit on the codebase"
run = "cat repo_output.txt | llm --model openai/gpt-4 --system \"You are a cybersecurity expert with experience in identifying security vulnerabilities in code.\" \"Perform a comprehensive security audit of this codebase.\" > security_audit.md"
```

### Chaining Tasks Together

For tasks that logically follow each other, use dependencies:

```toml
[tasks.LLM.full_analysis]
description = "Run complete codebase analysis suite"
deps = ["LLM.clean_bundles", "LLM.generate_code_review", "LLM.generate_missing_tests", "LLM.generate_readme"]
run = "echo 'Full analysis complete! Check the generated markdown files.'"
```

## Conclusion

This workflow combines the power of task automation with LLM-powered code analysis to create a systematic approach to codebase improvement. By using mise to manage tasks and repomix to prepare your codebase for LLM consumption, you can leverage AI to improve code quality, test coverage, and documentation with minimal setup effort.

Remember that while LLMs provide valuable insights, they should complement rather than replace human expertise. Always review the generated content critically and use it as a starting point for your own analysis and improvements.
