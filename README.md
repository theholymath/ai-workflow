# Mise and LLM Codebase Improvement Workflow

This document provides a detailed guide to using the [`mise`](https://mise.jdx.dev) task runner along with [`repomix`](https://github.com/yamadashy/repomix) and [`LLM`](https://github.com/simonw/LLM) tools to analyze, improve, and maintain your codebase. The workflow combines repository analysis, LLM-powered code review, test coverage improvement, and documentation generation into a systematic approach for codebase enhancement.

The end goal is to use this analysis as input to [`aider`](https://aider.chat) (or cursor, etc.) so you have  **fast, stable, and repeatable** automation for improving codebases. 

## Workflow Goal
The goal is to have a FAST (10 minutes?) have a meta understanding of your codebase, have actionable code enhancements, and an ability to do all this from the command line. 

### Sample Flow
1. Go to root of the directory where the code lives
1. Run [`aider`]() (**always make sure you are on a new branch**)
1. run `mise run LLM:generate_missing_tests`
1. Look at the generated markdown file (`missing-tests.md`)
1. Paste the first missing test “issue” into aider
1. `aider` does `aider` things 🤖
1. Interact and get it right
1. ...
1. Run tests
1. Move to next test 

# Why? Why are we doing all this?
At first this may seem over-the-top but this is a workflow that allows a developer to be fast and repeatable. The packages all have logs so you know what prompts, changes, commits all happened. Everything can be **undone in an automated fashion**.

Highlights:
* `aider` uses a local git for all code changes. When a user accepts, a commit is created with diffs and documentation 
* `llm` has [logs](https://llm.datasette.io/en/stable/logging.html) for all steps. Nothing is hidden.  

```mermaid
flowchart TD
    A[Create repomap using repomix] -->|mise run LLM:clean_bundles| B[Analyze the repo or subset]
    B -->|mise run LLM:generate_code_review| C[Choose a task]
    B -->|mise run LLM:generate_missing_tests| C
    B -->|mise run LLM:generate_readme| C
    C -->|Unit Tests| D[Create focused steps for improvement]
    C -->|UI Enhancements| D
    C -->|Documentation| D
    C -->|Bug Fixes| D
    D -->|Create git branch for task| E[Use aider to improve code]
    E -->|aider with improvement message| F[Review and commit changes]
    F --> G{More tasks?}
    G -->|Yes| C
    G -->|No| H[Done]
```

## Setup and Installation

### Install Tools (mise, aider, repomix, llm)

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

   # using brew
   brew install repomix
   
   # Using npm
   npm install -g repomix
   ```

3. **Install LLM command-line tool**

   [`azure-llm`](https://github.com/fabge/llm-azure)

   ```bash
   # Using pip
   pip install llm

   # install provider plugins (if you want to use something besides OpenAI)
   llm install llm-openrouter
   llm install llm-azure 
   
   # Configure with your API keys
   llm keys set openai
   # Or for other providers
   llm keys set anthropic
   # for azure once installed llm-azure plugin (see below for setup)
   llm keys set azure
   ```

4. **Install Aider** 
   ```bash
   pip install aider-chat
   ```

   Go [here](https://aider.chat/docs/llms/azure.html) for setting up Aider for Azure
   ```bash
   python -m pip install -U aider-chat

   # Mac/Linux:                                           
   export AZURE_API_KEY=<key>
   export AZURE_API_VERSION=2023-05-15
   export AZURE_API_BASE=https://myendpt.openai.azure.com

   # Windows
   setx AZURE_API_KEY <key>
   setx AZURE_API_VERSION 2023-05-15
   setx AZURE_API_BASE https://myendpt.openai.azure.com
   # ... restart your shell after setx commands

   aider --model azure/<your_deployment_name>

   # List models available from Azure
   aider --list-models azure/
   ```

### Configure mise.toml

Create a `.mise.toml` file in your project root with the following content:

```toml
[env]
# Set your preferred model here to use across all tasks
reasoning_model = "your/reasoning/model" # "openrouter/google/gemini-2.0-flash-001"
coding_model = "your/coding/model"

[tasks."LLM:clean_bundles"]
description = "Generate LLM bundle output file using repomix"
run = "repomix -o repo_output.txt"

[tasks."LLM:copy_buffer_bundle"] #notice the quotes
description = "Copy generated LLM bundle to clipboard"
run = "cat repo_output.txt | pbcopy"  # Use xclip for Linux

[tasks."LLM:generate_code_review"]
description = "Generate code review from repository content"
run = "cat repo_output.txt | LLM -m ${reasoning_model} -t code-review-gen > code-review.md"

[tasks."LLM:generate_missing_tests"]
description = "Generate missing tests suggestions"
run = "cat repo_output.txt | LLM -m ${coding_model} -t test-coverage-gen > missing-tests.md"

[tasks."LLM:generate_readme"]
description = "Generate README.md from repository content"
run = "cat repo_output.txt | LLM -m ${reasoning_model} -t readme-gen > README.md"
```
### Setting up llm-azure
The documentation for [`azure-llm`](https://github.com/fabge/llm-azure) buries some details. Besides setting up keys there also needs to be a config file in your llm library.
```bash
# find where the directory for llm CLI is
dirname "$(llm logs path)"

# cd into that dir
cd "$(dirname "$(llm logs path)")"

# Create an azure config file for you particular deployments
mkdir azure
touch azure/config.yaml
```

Configure the yaml file to your deployments found in the portal or through `az cli` for example
```bash
- model_id: gpt-4-32k
  model_name: gpt-4-32k
  api_base: https://your_deployment.openai.azure.com/
  api_version: '2023-05-15'

- model_id: text-embedding-3-small
  embedding_model: true
  model_name: text-embedding-3-small
  api_base: https://your_deployment.openai.azure.com/
  api_version: '2023-05-14'
  ```

### The templates in `llm`
This took me a bit to wrap my head around. The flag `-t readme-gen` is a "template" that `llm` calls. It is the prompt you want to use for the task at hand.

```bash
# readme-gen
llm --system "Based on the codebase in this file, please generate a detailed README.md that includes an overview of the project, its main features, setup instructions, and usage examples." --save readme-gen
```

```bash
# code-review-gen
llm --system "You are a senior developer. Your job is to do a thorough code review of this code. You should write it up and output markdown. Include line numbers, and contextual info. Your code review will be passed to another teammate, so be thorough. Think deeply  before writing the code review. Review every part, and don't hallucinate." --save code-review-gen
```

```bash
# test-coverage-gen
llm --system "You are a senior developer. Your job is to review this code, and write out a list of missing test cases, and code tests that should exist. You should be specific, and be very good. Do Not Hallucinate. Think quietly to yourself, then act - write the issues. The issues  will be given to a developer to executed on, so they should be in a format that is compatible with github issues" --save test-coverage-gen
```

```bash
# check_bad_actors_in_code
llm --system "You are a multilingual programming assistant and translation specialist. Your task is to analyze a provided codebase, identify all comments, docstrings, variable names, and strings written in a language other than English, and translate them into English. The input will be a well-formatted text file containing code, comments, and docstrings. If the original language is unclear, infer the meaning as accurately as possible. Output the translated content, along with the original content for reference, in clean Markdown format.  " --save translate_to_english
```

```bash
# check_bad_actors_in_code
llm --system "You are a cybersecurity expert and code analysis specialist. Your task is to analyze a given codebase to identify malicious or suspicious code patterns, such as hardcoded credentials, obfuscated code, functions with potentially harmful behavior, unauthorized data access, or any other security vulnerabilities. The input will be provided as a well-formatted text file, containing headers and all the code in one place. Your output must include a list of findings in clear and organized Markdown format, with explanations for why the code is flagged as malicious or suspect.  ### Instructions:


1. Carefully review the provided codebase.

2. Identify and flag any sections of code that appear malicious, suspicious, or vulnerable.

3. For each flagged section, include:

    - A brief description of the issue.

    - The reason why it is flagged as malicious or suspicious.

    - The specific code snippet or line number (if available).

4. Present the findings in clean Markdown format, including headers, lists, and code blocks for clarity." --save check_bad_actors_in_code
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
[tasks."LLM:generate_swagger_docs"]
description = "Generate Swagger/OpenAPI documentation"
run = "cat repo_output.txt | llm --system \"You are an API documentation specialist.\" \"Generate Swagger/OpenAPI documentation for all API endpoints in this codebase. Include parameters, responses, and examples.\" > swagger_docs.md"

[tasks."LLM:security_audit"]
description = "Perform a security audit on the codebase"
run = "cat repo_output.txt | llm --system \"You are a cybersecurity expert with experience in identifying security vulnerabilities in code.\" \"Perform a comprehensive security audit of this codebase. Identify potential vulnerabilities, classify them by severity, and provide remediation steps for each issue.\" > security_audit.md"
```

### Using Different Models for Different Tasks

You can override the default model for specific tasks:

```toml
[tasks."LLM:security_audit"]
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
