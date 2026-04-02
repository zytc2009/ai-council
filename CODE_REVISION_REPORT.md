# Code Revision Report — AI Council

**Generated**: 2026-04-02  
**Scope**: Full codebase analysis for debugging code, temporary solutions, and incomplete features

---

## Executive Summary

| Category | Count | Priority |
|----------|-------|----------|
| Test Failures | 1 | **Critical** |
| Silent Exception Handlers | 6 | High |
| Low Test Coverage Areas | 3 | Medium |
| Code Duplication | 1 | Medium |
| Documentation Gaps | 2 | Low |

---

## 1. Test Failures (Critical)

### 1.1 Config Validation Test Failure

**Location**: `tests/unit/test_config.py:47`  
**Test**: `TestAgentConfigValidate::test_missing_prompt_file_raises`

```python
# Test expects this to raise ValueError, but it doesn't
cfg = AgentConfig(
    name="Test",
    cli="test",
    model="m1",
    command="claude --help",  # Missing {prompt_file} AND stdin mode
    ...
)
```

**Issue**: The `validate()` method in `lib/config.py` has flawed logic. It checks for:
- `is_file_mode`: `{prompt_file}` or `{prompt_content}` in command
- `is_stdin_mode`: ` -p -`, ` -q -`, or trailing `-` 
- `is_output_file_mode`: `{output_file}` in command

The command `"claude --help"` passes all checks incorrectly, allowing invalid configurations.

**Recommendation**: Strengthen validation logic or update test expectations.

---

## 2. Silent Exception Handlers (High Priority)

Empty `pass` statements in exception handlers hide errors and make debugging difficult.

### 2.1 Windows Bash Path Detection

**Location**: `council.py:27`
```python
try:
    result = _sp.run(["cygpath", "-w", bash_path], ...)
    if result.returncode == 0:
        bash_path = result.stdout.strip()
except Exception:
    pass  # Silently ignores cygpath failures
```

**Risk**: Windows users may experience CLI failures without knowing why bash wasn't detected.

### 2.2 File Cleanup Failures

**Locations**: 
- `lib/agent_runner.py:171, 176`
- `lib/streaming_runner.py:222`

```python
finally:
    if prompt_file:
        try:
            Path(prompt_file).unlink(missing_ok=True)
        except Exception:
            pass  # Silent cleanup failure - temp files may accumulate
```

**Risk**: Temporary files may accumulate in `/tmp` over time.

### 2.3 Meeting Load Fallback

**Location**: `council.py:663`
```python
try:
    discussion = load_discussion(topic_id, BASE_DIR)
    _show_discussion(discussion, config, output)
    return
except (FileNotFoundError, ValueError):
    pass  # Silently falls back to meeting load
```

**Risk**: Valid errors (corrupt data) are silently ignored.

### 2.4 Empty CLI Stub

**Location**: `council.py:952`
```python
@click.group()
def agent_cmd():
    """Agent management commands."""
    pass  # Legitimate but worth noting
```

---

## 3. Low Test Coverage Areas (Medium Priority)

### 3.1 Discussion Orchestrator (78% coverage)

**File**: `lib/discussion_orchestrator.py`  
**Missing Lines**: 87, 130-134, 151, 161, 201-207, 311-315, 319, 329-333, 344, 373, 392-443, 470-471, 538, 548-549, 574, 587

**Key Untested Paths**:
- Error handling for failed agent responses (lines 87, 130-134)
- User input validation edge cases (lines 201-207)
- Consensus detection failures (lines 311-315, 319)
- File output generation errors (lines 548-549)

### 3.2 Streaming Runner (83% coverage)

**File**: `lib/streaming_runner.py`  
**Missing Lines**: 19, 25-33, 41, 112, 133-134, 140, 157-158, 208-210, 221-222, 247

**Key Untested Paths**:
- Windows bash path detection fallback (lines 25-33)
- Stdin pipe mode (lines 112, 133-134)
- Exception handling during streaming (lines 208-210)

### 3.3 Agent Runner (89% coverage)

**File**: `lib/agent_runner.py`  
**Missing Lines**: 23, 29-37, 45, 125-126, 175-176

**Key Untested Paths**:
- Windows-specific bash detection (lines 29-37)
- File reading fallback on error (lines 125-126)

---

## 4. Code Duplication (Medium Priority)

### 4.1 Windows Bash Detection

**Duplicated in**:
- `lib/agent_runner.py:16-37` 
- `lib/streaming_runner.py:14-33`

**Function**: `_find_bash_win32()`

Identical logic for finding bash on Windows exists in two files. Changes need to be synchronized.

**Recommendation**: Extract to shared utility module.

---

## 5. Hardcoded Values

### 5.1 Summarizer Agent Selection

**Location**: `council.py:87-93`

```python
def _pick_summarizer(config: Config) -> str:
    """Pick cheapest available agent for summarization tasks."""
    preference = ["claude-sonnet", "codex-o4-mini", "kimi", "claude-opus", "codex-o3"]
    ...
```

**Issue**: Hardcoded agent IDs may not match actual configuration. If these agents aren't defined in `agents.yaml`, the function falls back to an arbitrary agent.

**Recommendation**: Add to model strategies configuration or validate against available agents.

---

## 6. Documentation Gaps

### 6.1 TODO in File Structure

**Location**: `README.md:707`
```
├── tests/                        # 测试（TODO）
```

Tests are implemented but documentation still shows as TODO.

### 6.2 Missing Integration Tests

**Location**: `tests/integration/`

The directory structure exists but integration tests are not fully implemented (based on TESTING.md structure vs actual test count).

---

## 7. Recommendations Summary

### Immediate Actions (This Week)

1. **Fix test failure** in `test_config.py` - align validation logic with test expectations
2. **Replace silent `pass` statements** with proper logging:
   ```python
   except Exception as e:
       logging.debug("Optional operation failed: %s", e)
   ```

### Short Term (Next Sprint)

3. **Extract Windows bash detection** to shared utility to eliminate duplication
4. **Increase test coverage** for error paths in discussion orchestrator
5. **Update README.md** to remove TODO references to tests

### Long Term (Next Month)

6. **Add integration tests** for CLI commands
7. **Review hardcoded agent IDs** - move to configuration
8. **Add temp file cleanup monitoring** or use `tempfile.TemporaryDirectory`

---

## Appendix: Full File Inventory

| File | Lines | Coverage | Status |
|------|-------|----------|--------|
| `council.py` | ~1100 | N/A | CLI entry - needs error handling improvements |
| `lib/agent_runner.py` | 101 | 89% | Good - minor gaps in Windows paths |
| `lib/streaming_runner.py` | 113 | 83% | Moderate - needs error path tests |
| `lib/discussion_orchestrator.py` | 226 | 78% | **Needs attention** - 49 lines uncovered |
| `lib/orchestrator.py` | 103 | 98% | Excellent |
| `lib/config.py` | 105 | 97% | Excellent |
| `lib/consensus.py` | 33 | 94% | Good |
| `lib/context.py` | 26 | 100% | Perfect |
| `lib/meeting.py` | 179 | 95% | Excellent |
| `lib/prompt_builder.py` | 65 | 100% | Perfect |
| `lib/summarizer.py` | 39 | 100% | Perfect |
| `lib/cli_detector.py` | 88 | 92% | Good |

**Total Production Code**: ~1078 lines  
**Overall Coverage**: 91%
