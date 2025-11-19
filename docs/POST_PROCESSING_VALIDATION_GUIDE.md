# Post-Processing Validation Guide

## Overview

This document explains the post-processing validation strategy implemented in the notebook to ensure all generated clues meet the 15-20 word requirement. **Now using batch processing to avoid rate limits!**

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

### 3. **Batch Collection Phase (NEW!)**
During validation, collect all non-compliant clues with their metadata:
- Clue text
- Clue type (informed/misinformed/fake/extra)
- Round index and clue index for replacement
- Current word count

### 4. **Batch Correction**
For all non-compliant clues in one go:
```python
def batch_rewrite_clues_with_llm(invalid_clues, model, max_retries=3):
    """Ask the LLM to rewrite ALL invalid clues in a SINGLE call"""
    # Builds one prompt with all clues
    # Preserves meaning and clue type for each
    # Retries entire batch up to 3 times if needed
```

### 5. **Full Pipeline**
```python
def validate_and_fix_game_data(game_data, model, auto_fix=True):
    """
    - Validates all clues in game data
    - Collects all invalid clues
    - Batch fixes all at once (1 API call instead of N calls!)
    - Returns corrected data + validation report
    """
```

## Usage Examples

### Automatic Batch Fixing (Recommended)
```python
corrected_game_data, validation_report = validate_and_fix_game_data(
    game_data, 
    model, 
    auto_fix=True
)

# Check the report
print(f"Compliance rate: {validation_report['compliance_rate']}")
print(f"Fixed clues: {validation_report['fixed_clues']} (in single batch call!)")
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
    
    # 2. Validate and batch fix clues (single API call for all fixes!)
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

## Performance Comparison

**OLD Approach (Per-Clue Fixing):**
- Game with 10 invalid clues = 10 API calls
- Total for 10 games = ~100 API calls
- High rate limit risk
- Processing time: ~5 minutes (with delays)

**NEW Approach (Batch Fixing):**
- Game with 10 invalid clues = 1 API call
- Total for 10 games = ~10 API calls
- No rate limit issues
- Processing time: ~30 seconds
- **90% reduction in API calls!**


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
| `batch_rewrite_clues_with_llm(invalid_clues, model, max_retries)` | Rewrite ALL invalid clues in one call | `invalid_clues`: list of dicts<br>`model`: LLM<br>`max_retries`: int |
| `validate_and_fix_game_data(game_data, model, auto_fix)` | Validate all clues and batch fix issues | `game_data`: list<br>`model`: LLM<br>`auto_fix`: bool |
| `manual_fix_clues(game_data)` | Interactive manual fixing | `game_data`: list |

## Benefits of Batch Approach

- **Reliability**: Programmatic word counting is 100% accurate  
- **Efficiency**: Single API call instead of multiple calls per game  
- **Rate limit safe**: 90% reduction in API requests  
- **Faster**: Parallel processing by LLM for all clues at once  
- **Cost effective**: Fewer API calls = lower costs  
- **Transparency**: Detailed reports show what was fixed  
- **Flexibility**: Can disable auto-fix to review issues first  
- **Retry logic**: Handles cases where first batch attempt fails  
- **Fallback**: Keeps original if all retries fail (prevents data loss)

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

The execution now prints detailed progress with batch processing:

```
================================================================================
Running test 1/10: Movie - Star Wars Episode I: The Phantom Menace
================================================================================

Validating clues...

Validating Round 1...
  Issue: Round 1, informed_clues #3: 14 words
  Issue: Round 1, fake_clues #1: 22 words

Validating Round 2...
  Issue: Round 2, fake_clues #2: 23 words

Batch fixing 3 invalid clues...
  Batch rewrite successful (attempt 1): All 3 clues now valid

Validation Summary for Test 1:
  Total clues: 28
  Compliant clues: 28
  Fixed clues: 3 (in single batch call)
  Failed fixes: 0
  Compliance rate: 100.0%

Test 1 completed successfully
```

## Next Steps

1. **Run the refactored notebook** to generate validated clues
2. **Review validation_summary.csv** to check compliance rates
3. **Adjust retry logic** if needed (increase `max_retries` if fixes fail)
4. **Consider manual review** for test cases with low compliance rates

## Troubleshooting

**Issue**: Batch rewrite returns wrong number of clues  
**Solution**: Check JSON parsing in `extract_json_from_response()` - may need refinement

**Issue**: Some clues in batch still invalid after rewrite  
**Solution**: Increase `max_retries` from 3 to 5 in function call

**Issue**: LLM batch rewrites change meaning  
**Solution**: Review batch prompt clarity, or use `auto_fix=False` and `manual_fix_clues()` for more control

**Issue**: Rate limiting errors (should be rare now!)  
**Solution**: Increase `sleep()` time between test runs (currently 5s)
