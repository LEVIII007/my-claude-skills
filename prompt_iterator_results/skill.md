# Prompt Iterator Results

```yaml
name: prompt-iterator-results
description: Diagnose prompt failures by comparing actual vs expected outputs. Trigger: 'why is this prompt failing', 'debug prompt outputs', 'analyse prompt test results'. Not for iterating the prompt — use prompt-iterator-feedback.
model: sonnet-4.5
```

## Philosophy

This skill takes a diagnostic, engineering-driven approach to prompt improvement. Unlike feedback-based iteration (Skill 9), which works from abstract user feedback, this skill operates on **concrete failure data**: actual outputs that differ from expected outputs.

Think of it as debugging a system:
- The prompt is your code
- Test entries are your test suite
- Evaluator scores are your test results
- Failures point to bugs (gaps, ambiguities, conflicts)

The goal is not to patch symptoms but to find **root causes**. A single failure might indicate a systemic issue affecting multiple test cases. Sometimes the fix is not in the prompt at all—it might be in the test harness (dataset or evaluator).

This skill is methodical and analytical, categorizing failures into actionable buckets and generating targeted fixes with clear reasoning chains.

## Workflow

### Input Discovery

1. **Locate the prompt project structure:**
   ```
   prompts/<prompt-name>/
   ├── prompt_prd.md          # Requirements
   ├── prompt/
   │   ├── v1.md, v2.md...    # Version history
   │   └── current.md         # Current version
   ├── dataset/
   │   └── entries.json       # Test cases with expected outputs
   └── evaluators/
       └── evaluator.md       # Scoring criteria
   ```

2. **Read the failure data:**
   - Ask user for the evaluation results or read from evaluator output
   - Identify which test entries failed
   - For each failure, note:
     - Input provided
     - Actual output produced
     - Expected output
     - Evaluator score and findings

3. **Read context documents:**
   - Current prompt version (`prompt/current.md` or highest version)
   - Prompt PRD (`prompt_prd.md`)
   - Dataset entries (`dataset/entries.json`)
   - Evaluator definition (`evaluators/evaluator.md`)

### Root-Cause Analysis

For each failing test case, diagnose the failure type:

#### 1. Prompt Instruction Gaps
- **Symptom:** The prompt lacks explicit guidance for a scenario the user expects it to handle
- **Example:** Prompt doesn't mention how to handle edge cases, but test expects specific edge case behavior
- **Fix:** Add missing instructions to prompt

#### 2. Prompt Ambiguity
- **Symptom:** The prompt instruction is vague or open to interpretation
- **Example:** "Be concise" without defining what concise means in this context
- **Fix:** Make instructions more specific and concrete

#### 3. Prompt Conflicts
- **Symptom:** Two or more prompt rules contradict each other
- **Example:** "Always include examples" vs "Keep responses under 50 words"
- **Fix:** Resolve the conflict by prioritizing or rewriting rules

#### 4. Dataset Issues
- **Symptom:** The expected output is actually incorrect or unrealistic
- **Example:** Expected output demands information not available in the input
- **Fix:** Flag dataset entry for revision, don't change prompt

#### 5. Evaluator Issues
- **Symptom:** The evaluator scoring is too strict, too lenient, or misaligned with the PRD
- **Example:** Evaluator penalizes valid alternative outputs, or passes outputs that violate requirements
- **Fix:** Flag evaluator for revision, don't change prompt

### Collaborative Diagnosis

After initial analysis:

1. **Present findings to user:**
   ```
   Found 5 failures across 3 categories:
   
   Category A: Missing instruction for X (3 cases)
   - Entry #2: Prompt doesn't specify how to handle empty inputs
   - Entry #7: Prompt doesn't define "concise" threshold
   - Entry #12: Prompt doesn't cover multi-part questions
   
   Category B: Ambiguous phrasing (1 case)
   - Entry #5: "Use simple language" is too vague
   
   Category C: Possible dataset issue (1 case)
   - Entry #9: Expected output seems to contradict input context
   ```

2. **Ask for user input:**
   - "Do you agree with these categorizations?"
   - "Any additional root-cause hypotheses?"
   - "Which category should we prioritize fixing first?"

3. **Gather constraints:**
   - Are there fixes the user wants to avoid?
   - Should we fix all categories or focus on one?
   - Should we update dataset/evaluator or only the prompt?

### Generate Fixes

#### For Prompt Fixes:

1. **Create a new prompt version:**
   - Increment version number: if current is v3, create v4
   - Start with the current prompt content
   - Apply targeted changes based on root-cause analysis

2. **Document changes:**
   - Add a changelog comment at the top of the new version
   - For each change, note:
     - What changed
     - Why (which failure it addresses)
     - Which test entries it should fix

3. **Show diff-style view:**
   ```
   ## Changes in v4
   
   ### Added: Edge case handling for empty inputs
   Addresses: Entry #2 failure
   Rationale: Prompt lacked guidance on empty input scenario
   
   + When the input is empty, respond with: "No input provided."
   
   ### Clarified: Definition of "concise"
   Addresses: Entry #7 failure
   Rationale: "Concise" was ambiguous
   
   - Keep responses concise
   + Keep responses under 100 words unless context requires more detail
   
   ### Added: Multi-part question handling
   Addresses: Entry #12 failure
   Rationale: Prompt didn't specify how to structure multi-part answers
   
   + When the input contains multiple questions, answer each one
   + in a numbered list format.
   ```

#### For Dataset Fixes:

If a test entry has an incorrect expected output:

```
DATASET ISSUE FLAGGED:

Entry #9:
Input: "What is the capital of France?"
Current Expected Output: "Paris, located in the United Kingdom"
Issue: Expected output contains factual error

Recommended Action: Update expected output to "Paris, France"
```

#### For Evaluator Fixes:

If the evaluator scoring is misaligned:

```
EVALUATOR ISSUE FLAGGED:

Current Evaluator Behavior:
- Penalizes any response over 50 words, even when context requires detail
- Passes responses that omit required information

Recommended Action:
- Adjust word count threshold to be flexible based on input complexity
- Add strict checks for required information presence
```

### Output Delivery

1. **Save the new prompt version:**
   - Write to `prompts/<prompt-name>/prompt/v{N+1}.md`
   - Update `prompts/<prompt-name>/prompt/current.md` to point to new version (or copy content)

2. **Create a change summary document (optional):**
   ```markdown
   # Prompt Iteration Summary - v{N+1}
   
   ## Evaluation Results Analyzed
   - Total test entries: X
   - Failed entries: Y
   - Pass rate: Z%
   
   ## Root Causes Identified
   1. Missing instruction for edge cases (3 failures)
   2. Ambiguous phrasing in section B (1 failure)
   3. Dataset issue in entry #9 (1 failure)
   
   ## Changes Made
   - Added edge case handling
   - Clarified "concise" definition
   - Flagged dataset entry #9 for review
   
   ## Expected Impact
   - Should fix entries #2, #7, #12 (prompt gaps)
   - May require dataset revision for entry #9
   
   ## Next Steps
   - Re-run evaluator on all test entries
   - Review flagged dataset/evaluator issues
   - If failures persist, consider deeper structural changes
   ```

3. **Suggest next action:**
   - "Run `/prompt-evaluator-runner` to test the new prompt version"
   - "Review and fix flagged dataset entries, then re-test"
   - "If new failures appear, run this skill again for further diagnosis"

## Principles

### Be Systematic, Not Reactive
- Don't just fix the specific failure; understand the underlying pattern
- One failure in "edge case X" might indicate a broader gap in edge case handling

### Prioritize Root Causes Over Symptoms
- If 3 failures stem from the same ambiguous instruction, that's one root cause, not three
- Fix the root cause once rather than patching each symptom

### Validate Your Diagnosis
- Always ask: "Is this really a prompt issue, or is the test wrong?"
- Sometimes the expected output is unrealistic given the input
- Sometimes the evaluator is too strict or too lenient

### Show Your Work
- Present a clear reasoning chain: Failure → Diagnosis → Fix
- Use diff-style views to make changes transparent
- Explain why each change addresses a specific failure

### Recognize When the Prompt Isn't the Problem
- If expected outputs contradict the PRD, fix the dataset
- If the evaluator penalizes correct behavior, fix the evaluator
- Don't twist the prompt to match a broken test harness

### Iterate in Small, Testable Steps
- Don't change everything at once
- Each version should address a specific set of issues
- This makes it easier to isolate what worked and what didn't

### Maintain Backward Compatibility When Possible
- If the prompt already passes 80% of tests, don't break those to fix the other 20%
- Additive changes (new rules) are safer than destructive ones (removing rules)

### Document the Journey
- Future iterations will benefit from understanding past failures
- Changelog comments prevent repeating the same mistakes
- Version history shows how the prompt evolved

## Example Interaction

```
User: "I ran the evaluator and got 3 failures. Here's the output..."

Claude: Let me analyze these failures.

[Reads evaluation results, current prompt, dataset]

I found the following patterns:

CATEGORY 1: Missing instruction (2 failures)
- Entry #4: Input asks for pros AND cons, but output only lists pros
  Root cause: Prompt says "list the benefits" but doesn't mention drawbacks
  
- Entry #8: Input has no context, output says "based on the above"
  Root cause: Prompt assumes context is always present

CATEGORY 2: Possible dataset issue (1 failure)
- Entry #6: Expected output is 500 words, but PRD says "concise responses"
  Root cause: Dataset entry may contradict PRD requirements

Do these diagnoses match your understanding? Should I prioritize Category 1
(prompt fixes) or investigate Category 2 (dataset review) first?

User: "Fix Category 1 first, leave Category 2 for now."

Claude: Creating prompt v4 with targeted fixes...

[Generates new version with changes]

Changes made:
1. Added rule: "If the input asks for pros and cons, provide both sections"
2. Added rule: "If no context is provided, avoid referencing 'the above'"

New version saved to: prompts/feature-summarizer/prompt/v4.md

Diff view:
+ ## Handling pros/cons requests
+ If the input explicitly asks for both pros and cons, structure your
+ response with two sections: "Pros:" and "Cons:". Provide at least
+ 2 items in each section.

+ ## No-context scenarios
+ If the input does not provide background context, avoid phrases like
+ "based on the above" or "as mentioned earlier". Respond only to the
+ direct question using general knowledge.

Next steps:
1. Run `/prompt-evaluator-runner` to test v4 against all entries
2. If entries #4 and #8 now pass, review entry #6 dataset issue
3. If new failures appear, run this skill again for further iteration
```

## References

- Skill 9 (Prompt Iterator Feedback) — for abstract feedback-driven iteration
- Skill 7 (Prompt Evaluator Builder) — for evaluator structure and scoring logic
- Skill 8 (Prompt Evaluator Runner) — for running tests and getting failure data
- Skill 3 (Prompt PRD Creator) — for understanding original requirements

## Notes

- This skill is most effective when used AFTER running the evaluator (Skill 8)
- Works best with at least 3-5 test entries to identify patterns
- Can be run multiple times in a loop: Test → Diagnose → Fix → Test → Diagnose...
- If you're stuck in a loop (same failures keep appearing), consider:
  - More fundamental prompt restructuring (not just adding rules)
  - Dataset or evaluator revision
  - Re-examining the PRD to see if requirements are achievable