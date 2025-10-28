# Post-Processing Validation Guide

## Overview

This document explains the post-processing validation strategy implemented in the notebook to ensure all generated clues meet the 15-20 word requirement.

## Why Post-Processing?

**Problem**: LLMs don't always follow word count instructions reliably, even with explicit prompts.

**Solution**: Let the LLM generate clues naturally, then validate and fix them programmatically.

## Implementation Strategy

### 1. **Initial Generation**
- LLM generates all clues in JSON format
- No changes to the original prompt structure
- System prompt still includes word count instructions (helps improve initial compliance)

### 2. **Validation Phase**
```python
def validate_clue_word_count(clue):
    """Check if a clue meets the 15-20 word requirement"""
    word_count = len(clue.split())
    return 15 <= word_count <= 20, word_count
```

### 3. **Automatic Correction**
For each non-compliant clue:
```python
def rewrite_clue_with_llm(clue, clue_type, model, max_retries=3):
    """Ask the LLM to rewrite a clue to be exactly 15-20 words"""
    # Makes a targeted LLM call
    # Preserves meaning and clue type
    # Retries up to 3 times if needed
```

### 4. **Full Pipeline**
```python
def validate_and_fix_game_data(game_data, model, auto_fix=True):
    """
    - Validates all clues in game data
    - Optionally fixes non-compliant clues
    - Returns corrected data + validation report
    """
```

## Usage Examples

### Automatic Fixing (Recommended)
```python
corrected_game_data, validation_report = validate_and_fix_game_data(
    game_data, 
    model, 
    auto_fix=True
)

# Check the report
print(f"Compliance rate: {validation_report['compliance_rate']}")
print(f"Fixed clues: {validation_report['fixed_clues']}")
```

### Validation Only (No Fixes)
```python
game_data, validation_report = validate_and_fix_game_data(
    game_data, 
    model, 
    auto_fix=False
)

# Review issues
for issue in validation_report['issues']:
    print(issue)
```

### Manual Interactive Fixing
```python
corrected_game_data = manual_fix_clues(game_data)
# Prompts you interactively for each non-compliant clue
# Options: skip, edit manually, or auto-fix
```

## Workflow in Main Execution

The main execution loop now follows this pattern:

```python
for run_number, topic in enumerate(test_topics, 1):
    # 1. Generate clues with LLM
    response = model.invoke(messages)
    game_data = extract_json_from_response(response.content)
    
    # 2. Validate and fix clues
    corrected_game_data, validation_report = validate_and_fix_game_data(
        game_data, 
        model, 
        auto_fix=True
    )
    
    # 3. Process corrected data
    all_rows.extend(process_game_data(corrected_game_data, topic, run_number))
    
    # 4. Track validation metrics
    validation_summary.append(validation_report)
```

## Output Files

The refactored notebook now generates:

1. **`10_rounds_clues_analysis(gemini).csv`** - All clues with validation status (existing)
2. **`validation_summary(gemini).csv`** - NEW: Summary of validation and fixes per test run
   - Columns: test_run, topic, total_clues, compliant_clues, fixed_clues, failed_fixes, compliance_rate, issues

## Validation Report Structure

```python
validation_report = {
    "total_clues": 28,           # Total clues checked
    "compliant_clues": 25,       # Clues meeting 15-20 word requirement
    "fixed_clues": 3,            # Clues successfully rewritten
    "failed_fixes": 0,           # Clues that couldn't be fixed after retries
    "compliance_rate": "89.3%",  # Percentage of compliant clues
    "issues": [                  # List of specific issues found
        "Round 1, informed_clues #3: 14 words",
        "Round 2, fake_clues #1: 22 words"
    ]
}
```

## Key Functions Reference

| Function | Purpose | Parameters |
|----------|---------|------------|
| `validate_clue_word_count(clue)` | Check if single clue is valid | `clue`: str |
| `rewrite_clue_with_llm(clue, clue_type, model, max_retries)` | Rewrite non-compliant clue | `clue`: str<br>`clue_type`: str<br>`model`: LLM<br>`max_retries`: int |
| `validate_and_fix_game_data(game_data, model, auto_fix)` | Validate all clues and optionally fix | `game_data`: list<br>`model`: LLM<br>`auto_fix`: bool |
| `manual_fix_clues(game_data)` | Interactive manual fixing | `game_data`: list |

## Benefits of This Approach

✅ **Reliability**: Programmatic word counting is 100% accurate  
✅ **Targeted fixes**: Only rewrites problematic clues, preserving good ones  
✅ **Transparency**: Detailed reports show what was fixed  
✅ **Flexibility**: Can disable auto-fix to review issues first  
✅ **Retry logic**: Handles cases where first rewrite attempt fails  
✅ **Fallback**: Keeps original if all retries fail (prevents data loss)

## Testing the Implementation

A test cell is included (commented out by default) to validate the functions:

```python
# Uncomment the test cell to run a quick validation test
# It will:
# 1. Generate clues for a single test case
# 2. Run validation and fixing
# 3. Print detailed report
```

## Monitoring Compliance

The execution now prints detailed progress:

```
================================================================================
Running test 1/10: Movie - Star Wars Episode I: The Phantom Menace
================================================================================

📋 Validating clues...

Validating Round 1...
  ⚠️ Round 1, informed_clues #3: 14 words
  🔧 Attempting to fix...
  ✅ Rewrite successful (attempt 1): 17 words

Validating Round 2...
  ⚠️ Round 2, fake_clues #2: 23 words
  🔧 Attempting to fix...
  ✅ Rewrite successful (attempt 2): 19 words

📊 Validation Summary for Test 1:
  Total clues: 28
  Compliant clues: 28
  Fixed clues: 2
  Failed fixes: 0
  Compliance rate: 100.0%

✅ Test 1 completed successfully
```

## Next Steps

1. **Run the refactored notebook** to generate validated clues
2. **Review validation_summary.csv** to check compliance rates
3. **Adjust retry logic** if needed (increase `max_retries` if fixes fail)
4. **Consider manual review** for test cases with low compliance rates

## Troubleshooting

**Issue**: Too many retries failing  
**Solution**: Check the rewrite prompt in `rewrite_clue_with_llm()` - may need refinement

**Issue**: LLM rewrites change meaning  
**Solution**: Use `auto_fix=False` and `manual_fix_clues()` for more control

**Issue**: Rate limiting errors  
**Solution**: Increase `sleep()` time or reduce `max_retries`
