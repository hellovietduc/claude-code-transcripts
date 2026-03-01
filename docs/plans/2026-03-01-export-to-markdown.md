# Export to Markdown Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add a `--markdown` flag that outputs session transcripts as a single `.md` file instead of paginated HTML.

**Architecture:** Add a `generate_markdown()` function parallel to `generate_html()` that reuses the same parsing/conversation-grouping logic but renders each content block to Markdown text instead of HTML. Each CLI command (`local`, `json`, `web`) gets a `--markdown` flag that switches to the markdown codepath. Output is a single `.md` file (no pagination).

**Tech Stack:** Python, Click (CLI), existing `parse_session_file()` and `extract_text_from_content()` functions.

---

### Task 1: Add `render_content_block_markdown()` function

**Files:**
- Modify: `src/claude_code_transcripts/__init__.py`
- Test: `tests/test_generate_html.py`

**Step 1: Write the failing tests**

Add a new test class `TestMarkdownRendering` to `tests/test_generate_html.py`:

```python
from claude_code_transcripts import render_content_block_markdown


class TestMarkdownRendering:
    """Tests for Markdown rendering of content blocks."""

    def test_text_block(self):
        block = {"type": "text", "text": "Hello **world**"}
        result = render_content_block_markdown(block)
        assert result == "Hello **world**"

    def test_thinking_block(self):
        block = {"type": "thinking", "thinking": "Let me think about this"}
        result = render_content_block_markdown(block)
        assert "<details>" in result
        assert "Thinking" in result
        assert "Let me think about this" in result

    def test_tool_use_write(self):
        block = {
            "type": "tool_use",
            "id": "toolu_001",
            "name": "Write",
            "input": {"file_path": "/tmp/hello.py", "content": "print('hi')"},
        }
        result = render_content_block_markdown(block)
        assert "Write" in result
        assert "/tmp/hello.py" in result
        assert "print('hi')" in result

    def test_tool_use_edit(self):
        block = {
            "type": "tool_use",
            "id": "toolu_002",
            "name": "Edit",
            "input": {
                "file_path": "/tmp/hello.py",
                "old_string": "print('hi')",
                "new_string": "print('hello')",
            },
        }
        result = render_content_block_markdown(block)
        assert "Edit" in result
        assert "/tmp/hello.py" in result
        assert "print('hi')" in result
        assert "print('hello')" in result

    def test_tool_use_bash(self):
        block = {
            "type": "tool_use",
            "id": "toolu_003",
            "name": "Bash",
            "input": {"command": "ls -la", "description": "List files"},
        }
        result = render_content_block_markdown(block)
        assert "Bash" in result
        assert "ls -la" in result

    def test_tool_use_generic(self):
        block = {
            "type": "tool_use",
            "id": "toolu_004",
            "name": "Glob",
            "input": {"pattern": "**/*.py"},
        }
        result = render_content_block_markdown(block)
        assert "Glob" in result
        assert "**/*.py" in result

    def test_tool_result_string(self):
        block = {
            "type": "tool_result",
            "tool_use_id": "toolu_001",
            "content": "File written successfully",
        }
        result = render_content_block_markdown(block)
        assert "File written successfully" in result

    def test_tool_result_error(self):
        block = {
            "type": "tool_result",
            "tool_use_id": "toolu_001",
            "content": "Error: file not found",
            "is_error": True,
        }
        result = render_content_block_markdown(block)
        assert "Error" in result

    def test_tool_result_with_list_content(self):
        block = {
            "type": "tool_result",
            "tool_use_id": "toolu_001",
            "content": [{"type": "text", "text": "some output"}],
        }
        result = render_content_block_markdown(block)
        assert "some output" in result

    def test_image_block(self):
        block = {
            "type": "image",
            "source": {"media_type": "image/png", "data": "abc123base64"},
        }
        result = render_content_block_markdown(block)
        assert "[Image" in result

    def test_todo_write(self):
        block = {
            "type": "tool_use",
            "id": "toolu_005",
            "name": "TodoWrite",
            "input": {
                "todos": [
                    {"content": "First task", "status": "completed"},
                    {"content": "Second task", "status": "pending"},
                ]
            },
        }
        result = render_content_block_markdown(block)
        assert "First task" in result
        assert "Second task" in result
```

**Step 2: Run tests to verify they fail**

Run: `uv run pytest tests/test_generate_html.py::TestMarkdownRendering -v`
Expected: FAIL with `ImportError: cannot import name 'render_content_block_markdown'`

**Step 3: Write minimal implementation**

Add to `src/claude_code_transcripts/__init__.py` (after the existing `render_content_block` function):

```python
def render_content_block_markdown(block):
    """Render a single content block as Markdown text."""
    if not isinstance(block, dict):
        return str(block)

    block_type = block.get("type", "")

    if block_type == "text":
        return block.get("text", "")

    elif block_type == "thinking":
        thinking = block.get("thinking", "")
        return f"<details>\n<summary>Thinking</summary>\n\n{thinking}\n\n</details>"

    elif block_type == "image":
        return "[Image embedded in original transcript]"

    elif block_type == "tool_use":
        tool_name = block.get("name", "Unknown tool")
        tool_input = block.get("input", {})

        if tool_name == "TodoWrite":
            todos = tool_input.get("todos", [])
            lines = [f"**TodoWrite**\n"]
            for todo in todos:
                status = todo.get("status", "pending")
                content = todo.get("content", "")
                if status == "completed":
                    lines.append(f"- [x] {content}")
                elif status == "in_progress":
                    lines.append(f"- [ ] {content} *(in progress)*")
                else:
                    lines.append(f"- [ ] {content}")
            return "\n".join(lines)

        if tool_name == "Write":
            file_path = tool_input.get("file_path", "Unknown file")
            content = tool_input.get("content", "")
            return f"**Write** `{file_path}`\n\n```\n{content}\n```"

        if tool_name == "Edit":
            file_path = tool_input.get("file_path", "Unknown file")
            old_string = tool_input.get("old_string", "")
            new_string = tool_input.get("new_string", "")
            replace_all = tool_input.get("replace_all", False)
            header = f"**Edit** `{file_path}`"
            if replace_all:
                header += " *(replace all)*"
            return f"{header}\n\n```diff\n- {old_string}\n+ {new_string}\n```"

        if tool_name == "Bash":
            command = tool_input.get("command", "")
            description = tool_input.get("description", "")
            header = f"**Bash**"
            if description:
                header += f": {description}"
            return f"{header}\n\n```bash\n{command}\n```"

        # Generic tool
        display_input = {k: v for k, v in tool_input.items() if k != "description"}
        input_json = json.dumps(display_input, indent=2, ensure_ascii=False)
        description = tool_input.get("description", "")
        header = f"**{tool_name}**"
        if description:
            header += f": {description}"
        return f"{header}\n\n```json\n{input_json}\n```"

    elif block_type == "tool_result":
        content = block.get("content", "")
        is_error = block.get("is_error", False)
        prefix = "**Error:**\n" if is_error else ""

        if isinstance(content, str):
            return f"{prefix}```\n{content}\n```"
        elif isinstance(content, list):
            parts = []
            for item in content:
                if isinstance(item, dict):
                    if item.get("type") == "text":
                        parts.append(item.get("text", ""))
                    elif item.get("type") == "image":
                        parts.append("[Image embedded in original transcript]")
                    else:
                        parts.append(json.dumps(item, indent=2, ensure_ascii=False))
                else:
                    parts.append(str(item))
            result = "\n".join(parts)
            return f"{prefix}```\n{result}\n```"
        else:
            return f"{prefix}```\n{json.dumps(content, indent=2, ensure_ascii=False)}\n```"

    else:
        return f"```json\n{json.dumps(block, indent=2, ensure_ascii=False)}\n```"
```

**Step 4: Run tests to verify they pass**

Run: `uv run pytest tests/test_generate_html.py::TestMarkdownRendering -v`
Expected: PASS

**Step 5: Commit**

```bash
uv run black .
git add src/claude_code_transcripts/__init__.py tests/test_generate_html.py
git commit -m "feat: add render_content_block_markdown for Markdown export"
```

---

### Task 2: Add `generate_markdown()` function

**Files:**
- Modify: `src/claude_code_transcripts/__init__.py`
- Test: `tests/test_generate_html.py`

**Step 1: Write the failing tests**

Add a new test class `TestGenerateMarkdown` to `tests/test_generate_html.py`:

```python
from claude_code_transcripts import generate_markdown


class TestGenerateMarkdown:
    """Tests for generate_markdown function."""

    def test_generates_markdown_file(self, output_dir):
        """Test that generate_markdown creates a .md file."""
        fixture_path = Path(__file__).parent / "sample_session.jsonl"
        result_path = generate_markdown(fixture_path, output_dir)
        assert result_path.exists()
        assert result_path.suffix == ".md"

    def test_markdown_contains_user_messages(self, output_dir):
        """Test that user messages appear in the Markdown output."""
        fixture_path = Path(__file__).parent / "sample_session.jsonl"
        result_path = generate_markdown(fixture_path, output_dir)
        content = result_path.read_text()
        assert "Create a hello world function" in content

    def test_markdown_contains_assistant_messages(self, output_dir):
        """Test that assistant messages appear in the Markdown output."""
        fixture_path = Path(__file__).parent / "sample_session.jsonl"
        result_path = generate_markdown(fixture_path, output_dir)
        content = result_path.read_text()
        assert "I'll create that function for you" in content

    def test_markdown_contains_tool_calls(self, output_dir):
        """Test that tool calls appear in the Markdown output."""
        fixture_path = Path(__file__).parent / "sample_session.jsonl"
        result_path = generate_markdown(fixture_path, output_dir)
        content = result_path.read_text()
        assert "Write" in content
        assert "hello.py" in content

    def test_markdown_has_role_headers(self, output_dir):
        """Test that messages have User/Assistant headers."""
        fixture_path = Path(__file__).parent / "sample_session.jsonl"
        result_path = generate_markdown(fixture_path, output_dir)
        content = result_path.read_text()
        assert "### User" in content
        assert "### Assistant" in content

    def test_markdown_has_timestamps(self, output_dir):
        """Test that timestamps appear in the output."""
        fixture_path = Path(__file__).parent / "sample_session.jsonl"
        result_path = generate_markdown(fixture_path, output_dir)
        content = result_path.read_text()
        assert "2025-12-24" in content
```

**Step 2: Run tests to verify they fail**

Run: `uv run pytest tests/test_generate_html.py::TestGenerateMarkdown -v`
Expected: FAIL with `ImportError: cannot import name 'generate_markdown'`

**Step 3: Write minimal implementation**

Add to `src/claude_code_transcripts/__init__.py` (after `generate_html`):

```python
def generate_markdown(json_path, output_dir, github_repo=None):
    """Generate a single Markdown file from a session file.

    Returns the Path to the generated .md file.
    """
    output_dir = Path(output_dir)
    output_dir.mkdir(exist_ok=True)

    data = parse_session_file(json_path)
    loglines = data.get("loglines", [])

    if github_repo is None:
        github_repo = detect_github_repo(loglines)

    conversations = _group_conversations(loglines)
    md_parts = []

    for conv in conversations:
        for log_type, message_json, timestamp in conv["messages"]:
            if not message_json:
                continue
            try:
                message_data = json.loads(message_json)
            except json.JSONDecodeError:
                continue

            if log_type == "user":
                content = message_data.get("content", "")
                if is_tool_result_message(message_data):
                    role = "Tool reply"
                else:
                    role = "User"
                md_parts.append(f"### {role}")
                md_parts.append(f"*{timestamp}*\n")
                md_parts.append(_render_message_content_markdown(message_data))
            elif log_type == "assistant":
                md_parts.append("### Assistant")
                md_parts.append(f"*{timestamp}*\n")
                md_parts.append(_render_message_content_markdown(message_data))

        md_parts.append("---\n")

    markdown_content = "\n\n".join(md_parts)
    md_path = output_dir / "transcript.md"
    md_path.write_text(markdown_content, encoding="utf-8")
    return md_path
```

Also add these helper functions:

```python
def _group_conversations(loglines):
    """Group loglines into conversations. Shared by HTML and Markdown generators."""
    conversations = []
    current_conv = None
    for entry in loglines:
        log_type = entry.get("type")
        timestamp = entry.get("timestamp", "")
        is_compact_summary = entry.get("isCompactSummary", False)
        message_data = entry.get("message", {})
        if not message_data:
            continue
        message_json = json.dumps(message_data)
        is_user_prompt = False
        user_text = None
        if log_type == "user":
            content = message_data.get("content", "")
            text = extract_text_from_content(content)
            if text:
                is_user_prompt = True
                user_text = text
        if is_user_prompt:
            if current_conv:
                conversations.append(current_conv)
            current_conv = {
                "user_text": user_text,
                "timestamp": timestamp,
                "messages": [(log_type, message_json, timestamp)],
                "is_continuation": bool(is_compact_summary),
            }
        elif current_conv:
            current_conv["messages"].append((log_type, message_json, timestamp))
    if current_conv:
        conversations.append(current_conv)
    return conversations


def _render_message_content_markdown(message_data):
    """Render a message's content blocks as Markdown."""
    content = message_data.get("content", "")
    if isinstance(content, str):
        return content
    elif isinstance(content, list):
        parts = []
        for block in content:
            rendered = render_content_block_markdown(block)
            if rendered:
                parts.append(rendered)
        return "\n\n".join(parts)
    return str(content)
```

**Step 4: Run tests to verify they pass**

Run: `uv run pytest tests/test_generate_html.py::TestGenerateMarkdown -v`
Expected: PASS

**Step 5: Commit**

```bash
uv run black .
git add src/claude_code_transcripts/__init__.py tests/test_generate_html.py
git commit -m "feat: add generate_markdown function for single-file markdown export"
```

---

### Task 3: Refactor `generate_html` to use `_group_conversations`

**Files:**
- Modify: `src/claude_code_transcripts/__init__.py`

**Step 1: Refactor `generate_html()` and `generate_html_from_session_data()` to use the shared `_group_conversations()` helper**

Replace the inline conversation-grouping loop in both functions with a call to `_group_conversations(loglines)`.

In `generate_html()`, replace lines ~1321-1352 with:
```python
    conversations = _group_conversations(loglines)
```

In `generate_html_from_session_data()`, replace lines ~1795-1826 with:
```python
    conversations = _group_conversations(loglines)
```

**Step 2: Run all existing tests to verify nothing broke**

Run: `uv run pytest -v`
Expected: All existing tests PASS

**Step 3: Commit**

```bash
uv run black .
git add src/claude_code_transcripts/__init__.py
git commit -m "refactor: extract _group_conversations helper shared by HTML and Markdown generators"
```

---

### Task 4: Add `--markdown` flag to `local` and `json` commands

**Files:**
- Modify: `src/claude_code_transcripts/__init__.py`
- Test: `tests/test_all.py`

**Step 1: Write the failing tests**

Add to `tests/test_all.py`:

```python
class TestMarkdownFlag:
    """Tests for --markdown flag on CLI commands."""

    def test_json_command_markdown_flag(self, output_dir):
        """Test that json command with --markdown produces a .md file."""
        jsonl_file = output_dir / "test.jsonl"
        jsonl_file.write_text(
            '{"type": "user", "timestamp": "2025-01-01T10:00:00.000Z", "message": {"role": "user", "content": "Hello"}}\n'
            '{"type": "assistant", "timestamp": "2025-01-01T10:00:05.000Z", "message": {"role": "assistant", "content": [{"type": "text", "text": "Hi there!"}]}}\n'
        )
        md_output = output_dir / "md_output"
        runner = CliRunner()
        result = runner.invoke(
            cli,
            ["json", str(jsonl_file), "-o", str(md_output), "--markdown"],
        )
        assert result.exit_code == 0
        assert (md_output / "transcript.md").exists()
        content = (md_output / "transcript.md").read_text()
        assert "Hello" in content
        assert "Hi there!" in content

    def test_json_command_markdown_no_html(self, output_dir):
        """Test that --markdown does not produce HTML files."""
        jsonl_file = output_dir / "test.jsonl"
        jsonl_file.write_text(
            '{"type": "user", "timestamp": "2025-01-01T10:00:00.000Z", "message": {"role": "user", "content": "Hello"}}\n'
        )
        md_output = output_dir / "md_output"
        runner = CliRunner()
        result = runner.invoke(
            cli,
            ["json", str(jsonl_file), "-o", str(md_output), "--markdown"],
        )
        assert result.exit_code == 0
        assert not (md_output / "index.html").exists()
```

**Step 2: Run tests to verify they fail**

Run: `uv run pytest tests/test_all.py::TestMarkdownFlag -v`
Expected: FAIL (no such option `--markdown`)

**Step 3: Add `--markdown` flag to both commands**

Add `@click.option("--markdown", "use_markdown", is_flag=True, help="Output as a single Markdown file instead of HTML.")` to both `local_cmd` and `json_cmd`.

In `json_cmd`, add the branching logic:

```python
    if use_markdown:
        md_path = generate_markdown(json_file_path, output, github_repo=repo)
        click.echo(f"Generated {md_path.resolve()}")
    else:
        generate_html(json_file_path, output, github_repo=repo)
```

Same pattern in `local_cmd`. When `--markdown` is used, skip `--gist` and `--open` (they don't apply to markdown output).

**Step 4: Run tests to verify they pass**

Run: `uv run pytest tests/test_all.py::TestMarkdownFlag -v`
Expected: PASS

**Step 5: Commit**

```bash
uv run black .
git add src/claude_code_transcripts/__init__.py tests/test_all.py
git commit -m "feat: add --markdown flag to local and json commands"
```

---

### Task 5: Add `--markdown` flag to `web` command

**Files:**
- Modify: `src/claude_code_transcripts/__init__.py`

**Step 1: Add a `generate_markdown_from_session_data()` function**

This parallels `generate_html_from_session_data()`:

```python
def generate_markdown_from_session_data(session_data, output_dir, github_repo=None):
    """Generate Markdown from session data dict (instead of file path).

    Returns the Path to the generated .md file.
    """
    output_dir = Path(output_dir)
    output_dir.mkdir(exist_ok=True, parents=True)

    loglines = session_data.get("loglines", [])

    if github_repo is None:
        github_repo = detect_github_repo(loglines)

    conversations = _group_conversations(loglines)
    md_parts = []

    for conv in conversations:
        for log_type, message_json, timestamp in conv["messages"]:
            if not message_json:
                continue
            try:
                message_data = json.loads(message_json)
            except json.JSONDecodeError:
                continue

            if log_type == "user":
                if is_tool_result_message(message_data):
                    role = "Tool reply"
                else:
                    role = "User"
                md_parts.append(f"### {role}")
                md_parts.append(f"*{timestamp}*\n")
                md_parts.append(_render_message_content_markdown(message_data))
            elif log_type == "assistant":
                md_parts.append("### Assistant")
                md_parts.append(f"*{timestamp}*\n")
                md_parts.append(_render_message_content_markdown(message_data))

        md_parts.append("---\n")

    markdown_content = "\n\n".join(md_parts)
    md_path = output_dir / "transcript.md"
    md_path.write_text(markdown_content, encoding="utf-8")
    return md_path
```

**Step 2: Add `--markdown` flag to `web_cmd`**

Same `@click.option` decorator. Branch in the command body:

```python
    if use_markdown:
        md_path = generate_markdown_from_session_data(session_data, output, github_repo=repo)
        click.echo(f"Generated {md_path.resolve()}")
    else:
        generate_html_from_session_data(session_data, output, github_repo=repo)
```

**Step 3: Run all tests**

Run: `uv run pytest -v`
Expected: All PASS

**Step 4: Commit**

```bash
uv run black .
git add src/claude_code_transcripts/__init__.py
git commit -m "feat: add --markdown flag to web command"
```

---

### Task 6: Manual smoke test and final cleanup

**Step 1: Run the tool against a real session**

```bash
uv run claude-code-transcripts json tests/sample_session.jsonl -o /tmp/md-test --markdown
cat /tmp/md-test/transcript.md
```

Verify the output looks good — user prompts, assistant responses, tool calls, and tool results all render sensibly.

**Step 2: Run all tests one final time**

Run: `uv run pytest -v`
Expected: All PASS

**Step 3: Run black**

```bash
uv run black .
```

**Step 4: Final commit if any cleanup was needed**

```bash
git add -A
git commit -m "chore: final cleanup for markdown export feature"
```
