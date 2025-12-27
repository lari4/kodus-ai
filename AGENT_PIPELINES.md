# Agent Pipelines Documentation

This document describes all AI agent pipelines and workflows in the Kodus AI application. It includes the sequence of prompts, data flow between stages, and triggering mechanisms.

---

## Table of Contents

1. [Code Review Pipeline](#1-code-review-pipeline)
2. [Kody Rules Processing Pipeline](#2-kody-rules-processing-pipeline)
3. [Comment Analysis Pipeline](#3-comment-analysis-pipeline)
4. [ReWoo Strategy Pipeline](#4-rewoo-strategy-pipeline)
5. [ReAct Strategy Pipeline](#5-react-strategy-pipeline)
6. [Plan-Execute Strategy Pipeline](#6-plan-execute-strategy-pipeline)

---

## 1. Code Review Pipeline

**Location:** `libs/code-review/pipeline/strategy/code-review-pipeline.strategy.ts`

**Entry Point:** Webhook Event (PR opened/updated) → `CodeReviewJobProcessorService` → `RunCodeReviewAutomationUseCase` → `CodeReviewHandlerService` → `CodeReviewPipeline`

### 1.1 Pipeline Overview (14 Stages)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         CODE REVIEW PIPELINE                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Webhook Event (PR opened/updated/comment)                                  │
│                    │                                                        │
│                    ▼                                                        │
│  ┌─────────────────────────────────────┐                                    │
│  │ 1. ValidateNewCommitsStage          │ Check for new commits              │
│  └─────────────────────────────────────┘                                    │
│                    │                                                        │
│                    ▼                                                        │
│  ┌─────────────────────────────────────┐                                    │
│  │ 2. ResolveConfigStage               │ Load code review configuration     │
│  └─────────────────────────────────────┘                                    │
│                    │                                                        │
│                    ▼                                                        │
│  ┌─────────────────────────────────────┐                                    │
│  │ 3. ValidateConfigStage              │ Validate configuration             │
│  └─────────────────────────────────────┘                                    │
│                    │                                                        │
│                    ▼                                                        │
│  ┌─────────────────────────────────────┐                                    │
│  │ 4. FetchChangedFilesStage           │ Get modified files from PR         │
│  └─────────────────────────────────────┘                                    │
│                    │                                                        │
│                    ▼                                                        │
│  ┌─────────────────────────────────────┐                                    │
│  │ 5. LoadExternalContextStage         │ Load contextual information        │
│  └─────────────────────────────────────┘                                    │
│                    │                                                        │
│                    ▼                                                        │
│  ┌─────────────────────────────────────┐                                    │
│  │ 6. FileContextGateStage             │ Apply file filtering rules         │
│  └─────────────────────────────────────┘                                    │
│                    │                                                        │
│                    ▼                                                        │
│  ┌─────────────────────────────────────┐                                    │
│  │ 7. InitialCommentStage              │ Post initial status comment        │
│  └─────────────────────────────────────┘                                    │
│                    │                                                        │
│                    ▼                                                        │
│  ┌─────────────────────────────────────┐                                    │
│  │ 8. ProcessFilesPrLevelReviewStage   │ PR-level analysis (cross-file)     │
│  └─────────────────────────────────────┘                                    │
│                    │                                                        │
│                    ▼                                                        │
│  ╔═════════════════════════════════════╗                                    │
│  ║ 9. ProcessFilesReview               ║◄── MAIN LLM ANALYSIS STAGE        │
│  ║    (File-level code analysis)       ║                                    │
│  ╚═════════════════════════════════════╝                                    │
│                    │                                                        │
│                    ▼                                                        │
│  ┌─────────────────────────────────────┐                                    │
│  │ 10. CreatePrLevelCommentsStage      │ Create PR-level comments           │
│  └─────────────────────────────────────┘                                    │
│                    │                                                        │
│                    ▼                                                        │
│  ┌─────────────────────────────────────┐                                    │
│  │ 11. CreateFileCommentsStage         │ Create file-level comments         │
│  └─────────────────────────────────────┘                                    │
│                    │                                                        │
│                    ▼                                                        │
│  ┌─────────────────────────────────────┐                                    │
│  │ 12. AggregateResultsStage           │ Combine all results                │
│  └─────────────────────────────────────┘                                    │
│                    │                                                        │
│                    ▼                                                        │
│  ┌─────────────────────────────────────┐                                    │
│  │ 13. UpdateCommentsAndSummaryStage   │ Update comments, generate summary  │
│  └─────────────────────────────────────┘                                    │
│                    │                                                        │
│                    ▼                                                        │
│  ┌─────────────────────────────────────┐                                    │
│  │ 14. RequestChangesOrApproveStage    │ Final PR approval/rejection        │
│  └─────────────────────────────────────┘                                    │
│                    │                                                        │
│                    ▼                                                        │
│               COMPLETE                                                      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Stage 9: ProcessFilesReview (Detailed LLM Flow)

This is the main stage where AI analyzes code and generates suggestions.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│              STAGE 9: ProcessFilesReview - LLM ANALYSIS FLOW                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  changedFiles[]                                                             │
│       │                                                                     │
│       ▼                                                                     │
│  ┌────────────────────────────────┐                                         │
│  │ createOptimizedBatches()       │                                         │
│  │ (max concurrency: 20)          │                                         │
│  └────────────────────────────────┘                                         │
│       │                                                                     │
│       ▼                                                                     │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                    FOR EACH FILE BATCH                                 │ │
│  ├────────────────────────────────────────────────────────────────────────┤ │
│  │                                                                        │ │
│  │  ┌──────────────────────────────────┐                                  │ │
│  │  │ 1. selectReviewMode()            │                                  │ │
│  │  │    Prompt: prompt_selectorLight  │                                  │ │
│  │  │    OrHeavyMode                   │                                  │ │
│  │  └──────────────────────────────────┘                                  │ │
│  │       │                                                                │ │
│  │       ▼                                                                │ │
│  │  ┌──────────┐   ┌──────────────┐                                       │ │
│  │  │LIGHT_MODE│   │ HEAVY_MODE   │                                       │ │
│  │  │(diff only)│   │(full file +  │                                       │ │
│  │  └──────────┘   │ diff)        │                                       │ │
│  │       │         └──────────────┘                                       │ │
│  │       └─────────────┬──────────────┘                                   │ │
│  │                     ▼                                                  │ │
│  │  ┌──────────────────────────────────┐                                  │ │
│  │  │ 2. analyzeCodeWithAI()           │                                  │ │
│  │  │                                  │                                  │ │
│  │  │  System: prompt_codereview_      │                                  │ │
│  │  │          system_gemini           │                                  │ │
│  │  │                                  │                                  │ │
│  │  │  User:   prompt_codereview_      │                                  │ │
│  │  │          user_gemini             │                                  │ │
│  │  │                                  │                                  │ │
│  │  │  Input Data:                     │                                  │ │
│  │  │  - patchWithLinesStr (diff)      │                                  │ │
│  │  │  - fileContent (if HEAVY)        │                                  │ │
│  │  │  - languageResultPrompt          │                                  │ │
│  │  │  - v2PromptOverrides             │                                  │ │
│  │  │                                  │                                  │ │
│  │  │  Output: codeSuggestions[]       │                                  │ │
│  │  └──────────────────────────────────┘                                  │ │
│  │                     │                                                  │ │
│  │                     ▼                                                  │ │
│  │  ┌──────────────────────────────────┐                                  │ │
│  │  │ 3. severityAnalysisAssignment()  │                                  │ │
│  │  │                                  │                                  │ │
│  │  │  Prompt: prompt_severity_        │                                  │ │
│  │  │          analysis_user           │                                  │ │
│  │  │                                  │                                  │ │
│  │  │  Input: codeSuggestions[]        │                                  │ │
│  │  │                                  │                                  │ │
│  │  │  Output: severity assigned       │                                  │ │
│  │  │  (critical/high/medium/low)      │                                  │ │
│  │  └──────────────────────────────────┘                                  │ │
│  │                     │                                                  │ │
│  │                     ▼                                                  │ │
│  │  ┌──────────────────────────────────┐                                  │ │
│  │  │ 4. filterSuggestionsSafeGuard()  │                                  │ │
│  │  │                                  │                                  │ │
│  │  │  System: prompt_codeReviewSafe   │                                  │ │
│  │  │          guard_system            │                                  │ │
│  │  │                                  │                                  │ │
│  │  │  Five-Expert Panel Analysis:     │                                  │ │
│  │  │  - Edward (Special Cases)        │                                  │ │
│  │  │  - Alice (Syntax)                │                                  │ │
│  │  │  - Bob (Logic)                   │                                  │ │
│  │  │  - Charles (Style)               │                                  │ │
│  │  │  - Diana (Final)                 │                                  │ │
│  │  │                                  │                                  │ │
│  │  │  Action per suggestion:          │                                  │ │
│  │  │  - 'no_changes' (keep as-is)     │                                  │ │
│  │  │  - 'update' (modify suggestion)  │                                  │ │
│  │  │  - 'discard' (remove)            │                                  │ │
│  │  └──────────────────────────────────┘                                  │ │
│  │                     │                                                  │ │
│  │                     ▼                                                  │ │
│  │  ┌──────────────────────────────────┐                                  │ │
│  │  │ 5. validateImplementedSuggestions│                                  │ │
│  │  │                                  │                                  │ │
│  │  │  Prompt: prompt_validateImple    │                                  │ │
│  │  │          mentedSuggestions       │                                  │ │
│  │  │                                  │                                  │ │
│  │  │  Check if suggestion already     │                                  │ │
│  │  │  implemented in code patch       │                                  │ │
│  │  │                                  │                                  │ │
│  │  │  Output: implementationStatus    │                                  │ │
│  │  │  - 'implemented'                 │                                  │ │
│  │  │  - 'partially_implemented'       │                                  │ │
│  │  └──────────────────────────────────┘                                  │ │
│  │                     │                                                  │ │
│  └─────────────────────┼──────────────────────────────────────────────────┘ │
│                        ▼                                                    │
│  ┌────────────────────────────────────┐                                     │
│  │ Aggregate Results                  │                                     │
│  │ - validSuggestions[]               │                                     │
│  │ - discardedSuggestions[]           │                                     │
│  └────────────────────────────────────┘                                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.3 PR-Level Analysis (Stage 8)

Cross-file analysis for patterns spanning multiple files.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                   STAGE 8: PR-LEVEL ANALYSIS                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  All Files in PR                                                            │
│       │                                                                     │
│       ▼                                                                     │
│  ┌──────────────────────────────────────┐                                   │
│  │ Cross-File Analysis                  │                                   │
│  │                                      │                                   │
│  │  Prompt: prompt_codereview_cross_    │                                   │
│  │          file_analysis               │                                   │
│  │                                      │                                   │
│  │  Detects:                            │                                   │
│  │  - Duplicate implementations         │                                   │
│  │  - Inconsistent error handling       │                                   │
│  │  - Configuration drift               │                                   │
│  │  - Redundant operations              │                                   │
│  │  - Missing shared utilities          │                                   │
│  └──────────────────────────────────────┘                                   │
│       │                                                                     │
│       ▼                                                                     │
│  ┌──────────────────────────────────────┐                                   │
│  │ Kody Rules PR-Level Check            │                                   │
│  │                                      │                                   │
│  │  Prompt: prompt_kodyrules_prlevel_   │                                   │
│  │          analyzer                    │                                   │
│  │                                      │                                   │
│  │  Input:                              │                                   │
│  │  - PR title, description, author     │                                   │
│  │  - File list with status             │                                   │
│  │  - Cross-file Kody Rules             │                                   │
│  │                                      │                                   │
│  │  Detects:                            │                                   │
│  │  - Missing documentation files       │                                   │
│  │  - Missing test files                │                                   │
│  │  - PR metadata violations            │                                   │
│  └──────────────────────────────────────┘                                   │
│       │                                                                     │
│       ▼                                                                     │
│  validCrossFileSuggestions[]                                                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.4 Data Flow Summary

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     CODE REVIEW DATA FLOW                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  CodeReviewPipelineContext                                                  │
│  ├── correlationId                                                          │
│  ├── organizationAndTeamData                                                │
│  ├── repository                                                             │
│  ├── pullRequest                                                            │
│  │   ├── id, number, title                                                  │
│  │   ├── description                                                        │
│  │   └── author                                                             │
│  ├── changedFiles[]                                                         │
│  │   ├── filename                                                           │
│  │   ├── status (added/modified/deleted)                                    │
│  │   ├── patch (diff)                                                       │
│  │   ├── additions, deletions                                               │
│  │   └── content (full file)                                                │
│  ├── batches[] (grouped for parallel processing)                            │
│  ├── validSuggestions[]  ─────────────────────────────────────────┐         │
│  │   ├── id                                                       │         │
│  │   ├── relevantFile                                             │         │
│  │   ├── language                                                 │         │
│  │   ├── suggestionContent                                        │         │
│  │   ├── existingCode                                             │         │
│  │   ├── improvedCode                                             │         │
│  │   ├── oneSentenceSummary                                       │         │
│  │   ├── relevantLinesStart/End                                   ▼         │
│  │   ├── label (category)                                   Posted as       │
│  │   ├── severity                                           PR comments     │
│  │   └── llmPrompt                                                          │
│  ├── discardedSuggestions[]                                                 │
│  ├── validCrossFileSuggestions[]                                            │
│  └── statusInfo { status, message }                                         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.5 Triggering Mechanisms

| Trigger | Event | Description |
|---------|-------|-------------|
| PR Opened | Webhook | New PR created |
| PR Updated | Webhook | New commits pushed |
| Comment | Webhook | User requests review via comment |
| Manual | API | Admin triggers via dashboard |

---

## 2. Kody Rules Processing Pipeline

**Location:** `libs/kodyRules/application/use-cases/generate-kody-rules.use-case.ts`

Kody Rules are custom code rules that organizations define to enforce their coding standards. This pipeline generates, validates, and applies these rules.

### 2.1 Rule Generation Pipeline

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    KODY RULES GENERATION PIPELINE                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  API Request / Onboarding Trigger                                           │
│                    │                                                        │
│                    ▼                                                        │
│  ┌────────────────────────────────────┐                                     │
│  │ GenerateKodyRulesUseCase.execute() │                                     │
│  └────────────────────────────────────┘                                     │
│                    │                                                        │
│                    ▼                                                        │
│  ┌────────────────────────────────────┐                                     │
│  │ 1. Get Repositories by Filter      │                                     │
│  │    (date range: months/weeks/days) │                                     │
│  └────────────────────────────────────┘                                     │
│                    │                                                        │
│                    ▼                                                        │
│  ┌────────────────────────────────────┐                                     │
│  │ 2. For Each Repository:            │                                     │
│  │    - Get Pull Requests             │                                     │
│  │    - Fetch PR comments             │                                     │
│  │    - Fetch review comments         │                                     │
│  │    - Fetch file info               │                                     │
│  └────────────────────────────────────┘                                     │
│                    │                                                        │
│                    ▼                                                        │
│  ┌────────────────────────────────────┐                                     │
│  │ 3. CommentAnalysisService          │                                     │
│  │    .processComments()              │                                     │
│  │                                    │                                     │
│  │    - Flatten all comments          │                                     │
│  │    - Remove duplicates             │                                     │
│  │    - Filter bots                   │                                     │
│  │    - Min length: 100 chars         │                                     │
│  │    - Max total: 100 comments       │                                     │
│  └────────────────────────────────────┘                                     │
│                    │                                                        │
│                    ▼                                                        │
│  ╔════════════════════════════════════╗                                     │
│  ║ 4. generateKodyRules()             ║◄── MAIN LLM PROCESSING             │
│  ║    (4-Stage Filtering)             ║                                     │
│  ╚════════════════════════════════════╝                                     │
│                    │                                                        │
│                    ▼                                                        │
│  ┌────────────────────────────────────┐                                     │
│  │ 5. Standardize Rules               │                                     │
│  │    - Check against library         │                                     │
│  │    - Set origin (LIBRARY/GENERATED)│                                     │
│  │    - Status: PENDING               │                                     │
│  └────────────────────────────────────┘                                     │
│                    │                                                        │
│                    ▼                                                        │
│  ┌────────────────────────────────────┐                                     │
│  │ 6. CreateOrUpdateKodyRulesUseCase  │                                     │
│  │    - Save to database              │                                     │
│  │    - Send notification             │                                     │
│  └────────────────────────────────────┘                                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Four-Stage Filtering (Step 4 Detail)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    4-STAGE RULE FILTERING PROCESS                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Raw Comments (max 100)                                                     │
│       │                                                                     │
│       ▼                                                                     │
│  ┌──────────────────────────────────────┐                                   │
│  │ STAGE 1: IRRELEVANCE FILTER          │                                   │
│  │                                      │                                   │
│  │  System: prompt_CommentIrrelevance   │                                   │
│  │          FilterSystem                │                                   │
│  │                                      │                                   │
│  │  User:   prompt_CommentIrrelevance   │                                   │
│  │          FilterUser                  │                                   │
│  │                                      │                                   │
│  │  Input:  [{ id, body }]              │                                   │
│  │                                      │                                   │
│  │  Filters out:                        │                                   │
│  │  - Greetings, thank you messages     │                                   │
│  │  - Bot/template messages             │                                   │
│  │  - Non-actionable comments           │                                   │
│  │                                      │                                   │
│  │  Output: { ids: ["id1", "id2", ...] }│                                   │
│  └──────────────────────────────────────┘                                   │
│       │                                                                     │
│       ▼                                                                     │
│  Filtered Comments                                                          │
│       │                                                                     │
│       ▼                                                                     │
│  ┌──────────────────────────────────────┐                                   │
│  │ STAGE 2: RULE GENERATION             │                                   │
│  │                                      │                                   │
│  │  System: prompt_KodyRulesGenerator   │                                   │
│  │          System                      │                                   │
│  │                                      │                                   │
│  │  User:   prompt_KodyRulesGenerator   │                                   │
│  │          User                        │                                   │
│  │                                      │                                   │
│  │  Input:                              │                                   │
│  │  - Filtered comments                 │                                   │
│  │  - Library rules (pre-existing)      │                                   │
│  │                                      │                                   │
│  │  Output: Up to 12 rules              │                                   │
│  │  {                                   │                                   │
│  │    rules: [{                         │                                   │
│  │      uuid (if from library),         │                                   │
│  │      title,                          │                                   │
│  │      rule,                           │                                   │
│  │      severity,                       │                                   │
│  │      examples: [                     │                                   │
│  │        { snippet, isCorrect }        │                                   │
│  │      ]                               │                                   │
│  │    }]                                │                                   │
│  │  }                                   │                                   │
│  └──────────────────────────────────────┘                                   │
│       │                                                                     │
│       ▼                                                                     │
│  Candidate Rules (max 12)                                                   │
│       │                                                                     │
│       ▼                                                                     │
│  ┌──────────────────────────────────────┐                                   │
│  │ STAGE 3: DEDUPLICATION               │                                   │
│  │                                      │                                   │
│  │  System: prompt_KodyRulesGenerator   │                                   │
│  │          DuplicateFilterSystem       │                                   │
│  │                                      │                                   │
│  │  User:   prompt_KodyRulesGenerator   │                                   │
│  │          DuplicateFilterUser         │                                   │
│  │                                      │                                   │
│  │  Input:                              │                                   │
│  │  - New candidate rules               │                                   │
│  │  - Existing organization rules       │                                   │
│  │                                      │                                   │
│  │  Removes rules that duplicate        │                                   │
│  │  existing rules (by rule field)      │                                   │
│  │                                      │                                   │
│  │  Output: { uuids: ["..."] }          │                                   │
│  └──────────────────────────────────────┘                                   │
│       │                                                                     │
│       ▼                                                                     │
│  Deduplicated Rules                                                         │
│       │                                                                     │
│       ▼                                                                     │
│  ┌──────────────────────────────────────┐                                   │
│  │ STAGE 4: QUALITY FILTER              │                                   │
│  │                                      │                                   │
│  │  System: prompt_KodyRulesGenerator   │                                   │
│  │          QualityFilterSystem         │                                   │
│  │                                      │                                   │
│  │  User:   prompt_KodyRulesGenerator   │                                   │
│  │          QualityFilterUser           │                                   │
│  │                                      │                                   │
│  │  Filters by:                         │                                   │
│  │  - Clarity and conciseness           │                                   │
│  │  - Actionability                     │                                   │
│  │  - Specificity (not too broad)       │                                   │
│  │  - Impact                            │                                   │
│  │                                      │                                   │
│  │  Output: { uuids: ["..."] }          │                                   │
│  └──────────────────────────────────────┘                                   │
│       │                                                                     │
│       ▼                                                                     │
│  High-Quality Rules (Final Output)                                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.3 Rule Application During Code Review

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                KODY RULES APPLICATION IN CODE REVIEW                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Code Review Stage 9 (ProcessFilesReview)                                   │
│       │                                                                     │
│       ├──────────────────────────────────────────────────────────────┐      │
│       │                                                              │      │
│       ▼                                                              ▼      │
│  ┌─────────────────────┐                              ┌─────────────────────┐
│  │ Standard Code       │                              │ Kody Rules          │
│  │ Review Analysis     │                              │ Classification      │
│  │                     │                              │                     │
│  │ (General best       │                              │ Prompt:             │
│  │  practices)         │                              │ prompt_kodyrules_   │
│  │                     │                              │ classifier_system   │
│  └─────────────────────┘                              │                     │
│       │                                               │ Three-Expert Panel: │
│       │                                               │ Alice, Bob, Charles │
│       │                                               │                     │
│       │                                               │ Input:              │
│       │                                               │ - PR diff           │
│       │                                               │ - Organization rules│
│       │                                               │                     │
│       │                                               │ Output:             │
│       │                                               │ { rules: [          │
│       │                                               │   { uuid, reason }  │
│       │                                               │ ]}                  │
│       │                                               └─────────────────────┘
│       │                                                        │            │
│       │                                                        ▼            │
│       │                                               ┌─────────────────────┐
│       │                                               │ Suggestion          │
│       │                                               │ Generation          │
│       │                                               │                     │
│       │                                               │ Prompt:             │
│       │                                               │ prompt_kodyrules_   │
│       │                                               │ suggestiongeneration│
│       │                                               │                     │
│       │                                               │ Creates suggestions │
│       │                                               │ for each violation  │
│       │                                               └─────────────────────┘
│       │                                                        │            │
│       ▼                                                        ▼            │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                         MERGE SUGGESTIONS                             │  │
│  │                                                                       │  │
│  │  Standard Suggestions + Kody Rules Suggestions                        │  │
│  │                         │                                             │  │
│  │                         ▼                                             │  │
│  │  ┌───────────────────────────────────────────────────────────────┐    │  │
│  │  │ Kody Rules Guardian                                           │    │  │
│  │  │                                                               │    │  │
│  │  │ Prompt: prompt_kodyrules_guardian_system                      │    │  │
│  │  │                                                               │    │  │
│  │  │ Validates that standard suggestions don't                     │    │  │
│  │  │ violate any Kody Rules                                        │    │  │
│  │  │                                                               │    │  │
│  │  │ Output: { decisions: [{ id, shouldRemove }] }                 │    │  │
│  │  └───────────────────────────────────────────────────────────────┘    │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│       │                                                                     │
│       ▼                                                                     │
│  Final Validated Suggestions                                                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.4 External References Resolution

```
┌─────────────────────────────────────────────────────────────────────────────┐
│              EXTERNAL REFERENCES RESOLUTION FOR KODY RULES                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Kody Rule with file reference                                              │
│  e.g., "Follow patterns in @file:CONTRIBUTING.md"                           │
│       │                                                                     │
│       ▼                                                                     │
│  ┌──────────────────────────────────────┐                                   │
│  │ External Reference Detection         │                                   │
│  │                                      │                                   │
│  │  System: prompt_kodyrules_detect_    │                                   │
│  │          references_system           │                                   │
│  │                                      │                                   │
│  │  User:   prompt_kodyrules_detect_    │                                   │
│  │          references_user             │                                   │
│  │                                      │                                   │
│  │  Input:  Rule text                   │                                   │
│  │                                      │                                   │
│  │  Detects:                            │                                   │
│  │  - @file:path references             │                                   │
│  │  - [[file:path]] references          │                                   │
│  │  - Natural language references       │                                   │
│  │  - Line ranges (#L10-L50)            │                                   │
│  │                                      │                                   │
│  │  Output: {                           │                                   │
│  │    references: [{                    │                                   │
│  │      fileName,                       │                                   │
│  │      originalText,                   │                                   │
│  │      lineRange,                      │                                   │
│  │      description,                    │                                   │
│  │      repositoryName                  │                                   │
│  │    }]                                │                                   │
│  │  }                                   │                                   │
│  └──────────────────────────────────────┘                                   │
│       │                                                                     │
│       ▼                                                                     │
│  ┌──────────────────────────────────────┐                                   │
│  │ File Content Fetching                │                                   │
│  │                                      │                                   │
│  │  - Fetch file from repository        │                                   │
│  │  - Extract line range if specified   │                                   │
│  │  - Build externalReferencesMap       │                                   │
│  └──────────────────────────────────────┘                                   │
│       │                                                                     │
│       ▼                                                                     │
│  externalReferencesMap passed to                                            │
│  classification/generation prompts                                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Comment Analysis Pipeline

**Location:** `libs/code-review/infrastructure/adapters/services/commentAnalysis.service.ts`

Used for categorizing and processing code review comments.

### 3.1 Comment Categorization Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                   COMMENT CATEGORIZATION PIPELINE                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Raw Comments from PR                                                       │
│  [{ id, body, author, createdAt }]                                          │
│       │                                                                     │
│       ▼                                                                     │
│  ┌──────────────────────────────────────┐                                   │
│  │ STEP 1: IRRELEVANCE FILTER           │                                   │
│  │                                      │                                   │
│  │  System: prompt_CommentIrrelevance   │                                   │
│  │          FilterSystem                │                                   │
│  │                                      │                                   │
│  │  User:   prompt_CommentIrrelevance   │                                   │
│  │          FilterUser                  │                                   │
│  │                                      │                                   │
│  │  Input:                              │                                   │
│  │  [{ id: "...", body: "..." }]        │                                   │
│  │                                      │                                   │
│  │  Removes:                            │                                   │
│  │  - Greetings ("thanks!", "LGTM")     │                                   │
│  │  - Bot messages                      │                                   │
│  │  - Template comments                 │                                   │
│  │  - Non-actionable feedback           │                                   │
│  │                                      │                                   │
│  │  Output:                             │                                   │
│  │  { ids: ["id1", "id2", ...] }        │                                   │
│  └──────────────────────────────────────┘                                   │
│       │                                                                     │
│       ▼                                                                     │
│  Filtered Comments (relevant only)                                          │
│       │                                                                     │
│       ▼                                                                     │
│  ┌──────────────────────────────────────┐                                   │
│  │ STEP 2: CATEGORIZATION               │                                   │
│  │                                      │                                   │
│  │  System: prompt_CommentCategorizer   │                                   │
│  │          System                      │                                   │
│  │                                      │                                   │
│  │  User:   prompt_CommentCategorizer   │                                   │
│  │          User                        │                                   │
│  │                                      │                                   │
│  │  Categories assigned:                │                                   │
│  │  - security                          │                                   │
│  │  - error_handling                    │                                   │
│  │  - refactoring                       │                                   │
│  │  - performance_and_optimization      │                                   │
│  │  - maintainability                   │                                   │
│  │  - potential_issues                  │                                   │
│  │  - code_style                        │                                   │
│  │  - documentation_and_comments        │                                   │
│  │                                      │                                   │
│  │  Severity levels:                    │                                   │
│  │  - low                               │                                   │
│  │  - medium                            │                                   │
│  │  - high                              │                                   │
│  │  - critical                          │                                   │
│  │                                      │                                   │
│  │  Output:                             │                                   │
│  │  { suggestions: [{                   │                                   │
│  │      id, category, severity          │                                   │
│  │  }]}                                 │                                   │
│  └──────────────────────────────────────┘                                   │
│       │                                                                     │
│       ▼                                                                     │
│  ┌──────────────────────────────────────┐                                   │
│  │ STEP 3: MAP ORIGINAL CONTENT         │                                   │
│  │                                      │                                   │
│  │  Merge categorization with           │                                   │
│  │  original comment body               │                                   │
│  └──────────────────────────────────────┘                                   │
│       │                                                                     │
│       ▼                                                                     │
│  Categorized Comments                                                       │
│  [{ id, body, category, severity }]                                         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 Repeated Suggestion Clustering

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                REPEATED SUGGESTION CLUSTERING PIPELINE                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Code Suggestions (from review)                                             │
│  [{                                                                         │
│    id, relevantFile, suggestionContent,                                     │
│    existingCode, improvedCode, oneSentenceSummary                           │
│  }]                                                                         │
│       │                                                                     │
│       ▼                                                                     │
│  ┌──────────────────────────────────────┐                                   │
│  │ Repeated Suggestion Clustering       │                                   │
│  │                                      │                                   │
│  │  Prompt: prompt_repeated_suggestion_ │                                   │
│  │          clustering_system           │                                   │
│  │                                      │                                   │
│  │  Identifies suggestions that:        │                                   │
│  │  - Require the same code change      │                                   │
│  │  - Address similar patterns          │                                   │
│  │  - Have matching improvedCode        │                                   │
│  │                                      │                                   │
│  │  Groups by smallest UUID as primary  │                                   │
│  │                                      │                                   │
│  │  Output:                             │                                   │
│  │  { codeSuggestions: [{               │                                   │
│  │    id: "primary-uuid",               │                                   │
│  │    sameSuggestionsId: ["id2","id3"], │                                   │
│  │    problemDescription: "...",        │                                   │
│  │    actionStatement: "..."            │                                   │
│  │  }]}                                 │                                   │
│  └──────────────────────────────────────┘                                   │
│       │                                                                     │
│       ▼                                                                     │
│  Clustered Suggestions (deduplicated)                                       │
│  - Single comment addresses all instances                                   │
│  - Lists all affected files/lines                                           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 4. ReWoo Strategy Pipeline

**Location:** `packages/kodus-flow/src/engine/strategies/rewoo-strategy.ts`

ReWoo (Reasoning with Working Memory) is a pipeline-based strategy that separates planning, execution, and synthesis into distinct phases. Best for knowledge synthesis tasks.

### 4.1 Four-Phase Execution

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         REWOO STRATEGY PIPELINE                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  User Input + Available Tools                                               │
│       │                                                                     │
│       ▼                                                                     │
│  ╔═══════════════════════════════════════════════════════════════════════╗  │
│  ║ PHASE 1: SKETCH (Planning)                                            ║  │
│  ║                                                                       ║  │
│  ║  Prompt: getPlannerSystemPrompt()                                     ║  │
│  ║          getPlannerUserPrompt()                                       ║  │
│  ║                                                                       ║  │
│  ║  First Decision: Are tools actually needed?                           ║  │
│  ║  ┌───────────────────────────────────────────────────────────────┐    ║  │
│  ║  │ NO TOOLS NEEDED if:                                           │    ║  │
│  ║  │ - User is just greeting                                       │    ║  │
│  ║  │ - General questions about capabilities                        │    ║  │
│  ║  │ - Request can be answered with general knowledge              │    ║  │
│  ║  │                                                               │    ║  │
│  ║  │ TOOLS NEEDED if:                                              │    ║  │
│  ║  │ - User requests specific data retrieval                       │    ║  │
│  ║  │ - Task requires external tools                                │    ║  │
│  ║  │ - Task requires multiple steps with dependencies              │    ║  │
│  ║  └───────────────────────────────────────────────────────────────┘    ║  │
│  ║                                                                       ║  │
│  ║  If NO tools needed → { sketches: [] } → Skip to ORGANIZE            ║  │
│  ║                                                                       ║  │
│  ║  If tools needed → Create sketches (max 6):                          ║  │
│  ║  {                                                                    ║  │
│  ║    sketches: [{                                                       ║  │
│  ║      id: "S1",                                                        ║  │
│  ║      query: "Clear question to gather evidence",                      ║  │
│  ║      tool: "TOOL_NAME",                                               ║  │
│  ║      arguments: { param: "value" }                                    ║  │
│  ║    }]                                                                 ║  │
│  ║  }                                                                    ║  │
│  ╚═══════════════════════════════════════════════════════════════════════╝  │
│       │                                                                     │
│       ▼                                                                     │
│  ╔═══════════════════════════════════════════════════════════════════════╗  │
│  ║ PHASE 2: WORK (Parallel Execution)                                    ║  │
│  ║                                                                       ║  │
│  ║  Concurrency: maxParallelWork = 4                                     ║  │
│  ║                                                                       ║  │
│  ║  For each sketch (in parallel):                                       ║  │
│  ║  ┌───────────────────────────────────────────────────────────────┐    ║  │
│  ║  │                                                               │    ║  │
│  ║  │  1. Find tool in context.availableTools                       │    ║  │
│  ║  │  2. SharedStrategyMethods.executeTool(action, context)        │    ║  │
│  ║  │  3. Collect evidence:                                         │    ║  │
│  ║  │                                                               │    ║  │
│  ║  │  RewooEvidenceItem {                                          │    ║  │
│  ║  │    id: "E1",                                                  │    ║  │
│  ║  │    sketchId: "S1",                                            │    ║  │
│  ║  │    toolName: "TOOL_NAME",                                     │    ║  │
│  ║  │    input: { ... },                                            │    ║  │
│  ║  │    output: <tool_result>,                                     │    ║  │
│  ║  │    error?: "error message",                                   │    ║  │
│  ║  │    latencyMs: 1234                                            │    ║  │
│  ║  │  }                                                            │    ║  │
│  ║  │                                                               │    ║  │
│  ║  └───────────────────────────────────────────────────────────────┘    ║  │
│  ║                                                                       ║  │
│  ║  Timeout: perWorkTimeoutMs = 25,000                                   ║  │
│  ╚═══════════════════════════════════════════════════════════════════════╝  │
│       │                                                                     │
│       ▼                                                                     │
│  All Evidence Collected: [E1, E2, E3, ...]                                  │
│       │                                                                     │
│       ▼                                                                     │
│  ╔═══════════════════════════════════════════════════════════════════════╗  │
│  ║ PHASE 3: ORGANIZE (Synthesis)                                         ║  │
│  ║                                                                       ║  │
│  ║  Prompt: getOrganizerSystemPrompt()                                   ║  │
│  ║          getOrganizerUserPrompt()                                     ║  │
│  ║                                                                       ║  │
│  ║  Input:                                                               ║  │
│  ║  - Original goal/input                                                ║  │
│  ║  - All collected evidence [E1, E2, ...]                               ║  │
│  ║                                                                       ║  │
│  ║  Rules:                                                               ║  │
│  ║  - ONLY use information from provided evidence                        ║  │
│  ║  - CITE every claim with evidence IDs [E1]                            ║  │
│  ║  - STATE if evidence is insufficient                                  ║  │
│  ║  - NO external knowledge or assumptions                               ║  │
│  ║                                                                       ║  │
│  ║  Output:                                                              ║  │
│  ║  {                                                                    ║  │
│  ║    answer: "Comprehensive evidence-based answer",                     ║  │
│  ║    citations: ["E1", "E2", "E3"],                                     ║  │
│  ║    confidence: 0.85                                                   ║  │
│  ║  }                                                                    ║  │
│  ╚═══════════════════════════════════════════════════════════════════════╝  │
│       │                                                                     │
│       ▼                                                                     │
│  ┌───────────────────────────────────────┐                                  │
│  │ PHASE 4: VERIFY (Optional)            │                                  │
│  │                                       │                                  │
│  │ If maxVerifyPasses > 0:               │                                  │
│  │ - Validate answer against evidence    │                                  │
│  │ - Re-ORGANIZE if verification fails   │                                  │
│  └───────────────────────────────────────┘                                  │
│       │                                                                     │
│       ▼                                                                     │
│  Final Answer with Citations                                                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.2 Configuration

| Parameter | Default | Description |
|-----------|---------|-------------|
| topKSketches | 4 | Maximum sketches per plan |
| maxParallelWork | 4 | Concurrent tool executions |
| overallTimeoutMs | 120,000 | Total execution timeout (2 min) |
| perWorkTimeoutMs | 25,000 | Per-tool execution timeout |
| perLLMTimeoutMs | 20,000 | Per-LLM call timeout |
| maxVerifyPasses | 1 | Optional verification iterations |
| requireEvidenceAnchors | true | Require citations in answer |

---

## 5. ReAct Strategy Pipeline

**Location:** `packages/kodus-flow/src/engine/strategies/react-strategy.ts`

ReAct (Reasoning + Acting) combines reasoning and action in an iterative loop. The agent thinks about what to do, takes an action, observes results, and continues until the task is complete.

### 5.1 Iterative Think-Act-Observe Loop

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          REACT STRATEGY PIPELINE                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  User Input + Available Tools + Scratchpad (Working Memory)                 │
│       │                                                                     │
│       ▼                                                                     │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                        ITERATION LOOP                                 │  │
│  │                    (max: 10 iterations)                               │  │
│  ├───────────────────────────────────────────────────────────────────────┤  │
│  │                                                                       │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │  │ CHECK STOPPING CONDITIONS                                       │  │  │
│  │  │                                                                 │  │  │
│  │  │ 1. hasFinalAnswer? ──────────────────────────▶ EXIT (success)   │  │  │
│  │  │ 2. shouldStop()? ────────▶ forceFinalAnswer() ▶ EXIT            │  │  │
│  │  │ 3. detectLoop()? ────────▶ forceFinalAnswer() ▶ EXIT            │  │  │
│  │  │ 4. Max iterations? ──────▶ forceFinalAnswer() ▶ EXIT            │  │  │
│  │  └─────────────────────────────────────────────────────────────────┘  │  │
│  │       │                                                               │  │
│  │       ▼ (continue iteration)                                          │  │
│  │                                                                       │  │
│  │  ╔═════════════════════════════════════════════════════════════════╗  │  │
│  │  ║ THINK: generateThought()                                        ║  │  │
│  │  ║                                                                 ║  │  │
│  │  ║  Prompt: getSystemPrompt() + getTaskPrompt()                    ║  │  │
│  │  ║                                                                 ║  │  │
│  │  ║  Input:                                                         ║  │  │
│  │  ║  - User objective                                               ║  │  │
│  │  ║  - Scratchpad (working memory)                                  ║  │  │
│  │  ║  - Available tools                                              ║  │  │
│  │  ║  - Execution history (all previous steps)                       ║  │  │
│  │  ║                                                                 ║  │  │
│  │  ║  Output: AgentThought                                           ║  │  │
│  │  ║  {                                                              ║  │  │
│  │  ║    reasoning: "Why I'm taking this action" (max 200 chars),     ║  │  │
│  │  ║    confidence: 0.75,                                            ║  │  │
│  │  ║    scratchpadUpdate: "Updated working memory...",               ║  │  │
│  │  ║    hypotheses: [{                                               ║  │  │
│  │  ║      approach: "Primary approach",                              ║  │  │
│  │  ║      confidence: 0.8,                                           ║  │  │
│  │  ║      action: {                                                  ║  │  │
│  │  ║        type: "tool_call" | "final_answer",                      ║  │  │
│  │  ║        toolName: "TOOL_NAME",     // if tool_call               ║  │  │
│  │  ║        input: { ... },            // if tool_call               ║  │  │
│  │  ║        content: "Final answer"    // if final_answer            ║  │  │
│  │  ║      }                                                          ║  │  │
│  │  ║    }],                                                          ║  │  │
│  │  ║    reflection: {                                                ║  │  │
│  │  ║      shouldContinue: true,                                      ║  │  │
│  │  ║      reasoning: "Why continue or stop",                         ║  │  │
│  │  ║      alternatives: ["alt1", "alt2"]                             ║  │  │
│  │  ║    },                                                           ║  │  │
│  │  ║    earlyStopping: {                                             ║  │  │
│  │  ║      shouldStop: false,                                         ║  │  │
│  │  ║      reason: "Reason to stop early"                             ║  │  │
│  │  ║    }                                                            ║  │  │
│  │  ║  }                                                              ║  │  │
│  │  ╚═════════════════════════════════════════════════════════════════╝  │  │
│  │       │                                                               │  │
│  │       ▼                                                               │  │
│  │  ╔═════════════════════════════════════════════════════════════════╗  │  │
│  │  ║ ACT: executeAction()                                            ║  │  │
│  │  ║                                                                 ║  │  │
│  │  ║  ┌─────────────────────┐    ┌─────────────────────────────────┐ ║  │  │
│  │  ║  │ type: "tool_call"  │    │ type: "final_answer"            │ ║  │  │
│  │  ║  ├─────────────────────┤    ├─────────────────────────────────┤ ║  │  │
│  │  ║  │                     │    │                                 │ ║  │  │
│  │  ║  │ Find tool by name   │    │ Return ActionResult with        │ ║  │  │
│  │  ║  │        ↓            │    │ type='final_answer'             │ ║  │  │
│  │  ║  │ executeTool()       │    │ content=answer                  │ ║  │  │
│  │  ║  │        ↓            │    │                                 │ ║  │  │
│  │  ║  │ ActionResult        │    │        ↓                        │ ║  │  │
│  │  ║  │ type='tool_result'  │    │ EXIT LOOP                       │ ║  │  │
│  │  ║  │ output=result       │    │                                 │ ║  │  │
│  │  ║  │                     │    │                                 │ ║  │  │
│  │  ║  └─────────────────────┘    └─────────────────────────────────┘ ║  │  │
│  │  ╚═════════════════════════════════════════════════════════════════╝  │  │
│  │       │                                                               │  │
│  │       ▼                                                               │  │
│  │  ╔═════════════════════════════════════════════════════════════════╗  │  │
│  │  ║ OBSERVE: analyzeResult()                                        ║  │  │
│  │  ║                                                                 ║  │  │
│  │  ║  Determine:                                                     ║  │  │
│  │  ║  - isComplete: Did we achieve the goal?                         ║  │  │
│  │  ║  - isSuccessful: Was the action successful?                     ║  │  │
│  │  ║  - shouldContinue: Should we proceed?                           ║  │  │
│  │  ║                                                                 ║  │  │
│  │  ║  Generate feedback based on result type                         ║  │  │
│  │  ╚═════════════════════════════════════════════════════════════════╝  │  │
│  │       │                                                               │  │
│  │       ▼                                                               │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │  │ RECORD: ExecutionStep                                           │  │  │
│  │  │                                                                 │  │  │
│  │  │ {                                                               │  │  │
│  │  │   id, type: 'think', timestamp, duration,                       │  │  │
│  │  │   thought, action, result, observation,                         │  │  │
│  │  │   metadata: { iteration, strategy, ... }                        │  │  │
│  │  │ }                                                               │  │  │
│  │  │                                                                 │  │  │
│  │  │ → Added to execution history                                    │  │  │
│  │  │ → UPDATE SESSION via ContextService                             │  │  │
│  │  └─────────────────────────────────────────────────────────────────┘  │  │
│  │       │                                                               │  │
│  │       └────────────────────────────┐                                  │  │
│  │                                    │                                  │  │
│  │  iteration++                       │                                  │  │
│  │       │                            │                                  │  │
│  │       └───────────────────────────▶│                                  │  │
│  │                                    │                                  │  │
│  └────────────────────────────────────┼──────────────────────────────────┘  │
│                                       │                                     │
│                                       ▼                                     │
│  Final Answer                                                               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 5.2 Decision Matrix

| Confidence | Action | Description |
|------------|--------|-------------|
| > 0.8 | `final_answer` | Complete info available |
| 0.6 - 0.8 | `tool_call` | Need specific data |
| 0.3 - 0.6 | `multi_hypothesis` | Generate alternatives |
| < 0.3 | `early_stop` | Insufficient confidence |

### 5.3 Loop Detection

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           LOOP DETECTION                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  detectLoop() checks for:                                                   │
│                                                                             │
│  1. Identical Actions (last 3):                                             │
│     ┌─────┐    ┌─────┐    ┌─────┐                                           │
│     │  A  │ == │  A  │ == │  A  │  → LOOP DETECTED                          │
│     └─────┘    └─────┘    └─────┘                                           │
│                                                                             │
│  2. Alternating Pattern:                                                    │
│     ┌─────┐    ┌─────┐    ┌─────┐                                           │
│     │  A  │    │  B  │    │  A  │  → LOOP DETECTED                          │
│     └─────┘    └─────┘    └─────┘                                           │
│                                                                             │
│  3. Same Tool + Same Input (twice):                                         │
│     ┌──────────────────┐    ┌──────────────────┐                            │
│     │ tool: X          │ == │ tool: X          │ → LOOP DETECTED            │
│     │ input: { a: 1 }  │    │ input: { a: 1 }  │                            │
│     └──────────────────┘    └──────────────────┘                            │
│                                                                             │
│  On loop detection → forceFinalAnswer()                                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 5.4 Configuration

| Parameter | Default | Description |
|-----------|---------|-------------|
| maxIterations | 10 | Maximum think-act-observe cycles |
| maxToolCalls | 20 | Maximum total tool calls |
| maxExecutionTime | 300,000 | Total timeout (5 min) |
| stepTimeout | 60,000 | Per-step timeout (1 min) |

---

## 6. Plan-Execute Strategy Pipeline

**Location:** `packages/kodus-flow/src/engine/strategies/plan-execute-strategy.ts`

Plan-Execute is a two-phase strategy that first creates a complete plan, then executes it step by step. Best for structured, predictable tasks.

### 6.1 Two-Phase Execution

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      PLAN-EXECUTE STRATEGY PIPELINE                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  User Input + Available Tools                                               │
│       │                                                                     │
│       ▼                                                                     │
│  ╔═══════════════════════════════════════════════════════════════════════╗  │
│  ║ PHASE 1: PLAN                                                         ║  │
│  ║                                                                       ║  │
│  ║  Prompt: getSystemPrompt() + getUserPrompt()                          ║  │
│  ║                                                                       ║  │
│  ║  Planning Framework:                                                  ║  │
│  ║  1. Analyze - Understand task and available tools                     ║  │
│  ║  2. Break Down - Decompose into manageable steps                      ║  │
│  ║  3. Sequence - Order steps logically with dependencies                ║  │
│  ║  4. Validate - Ensure each step is executable                         ║  │
│  ║  5. Optimize - Keep plan concise and efficient                        ║  │
│  ║                                                                       ║  │
│  ║  Output: Plan                                                         ║  │
│  ║  {                                                                    ║  │
│  ║    goal: "Overall objective",                                         ║  │
│  ║    reasoning: "Why this approach works",                              ║  │
│  ║    steps: [                                                           ║  │
│  ║      {                                                                ║  │
│  ║        id: "step-1",                                                  ║  │
│  ║        type: "tool_call",                                             ║  │
│  ║        toolName: "TOOL_NAME",                                         ║  │
│  ║        description: "What this step does",                            ║  │
│  ║        input: { param: "value" }                                      ║  │
│  ║      },                                                               ║  │
│  ║      {                                                                ║  │
│  ║        id: "step-2",                                                  ║  │
│  ║        type: "tool_call",                                             ║  │
│  ║        toolName: "ANOTHER_TOOL",                                      ║  │
│  ║        description: "...",                                            ║  │
│  ║        input: { ... }                                                 ║  │
│  ║      },                                                               ║  │
│  ║      {                                                                ║  │
│  ║        id: "step-3",                                                  ║  │
│  ║        type: "final_answer",                                          ║  │
│  ║        content: "Final response to user"                              ║  │
│  ║      }                                                                ║  │
│  ║    ]                                                                  ║  │
│  ║  }                                                                    ║  │
│  ║                                                                       ║  │
│  ║  EMIT EVENT: agent.plan.created                                       ║  │
│  ╚═══════════════════════════════════════════════════════════════════════╝  │
│       │                                                                     │
│       ▼                                                                     │
│  Plan with N Steps                                                          │
│       │                                                                     │
│       ▼                                                                     │
│  ╔═══════════════════════════════════════════════════════════════════════╗  │
│  ║ PHASE 2: EXECUTE (Sequential)                                         ║  │
│  ║                                                                       ║  │
│  ║  ┌─────────────────────────────────────────────────────────────────┐  ║  │
│  ║  │                   FOR EACH STEP IN PLAN                         │  ║  │
│  ║  ├─────────────────────────────────────────────────────────────────┤  ║  │
│  ║  │                                                                 │  ║  │
│  ║  │  EMIT EVENT: agent.step.started                                 │  ║  │
│  ║  │       │                                                         │  ║  │
│  ║  │       ▼                                                         │  ║  │
│  ║  │  ┌─────────────────────────────────────────────────────────┐    │  ║  │
│  ║  │  │ CHECK LIMITS                                            │    │  ║  │
│  ║  │  │                                                         │    │  ║  │
│  ║  │  │ Timeout: Date.now() - startTime > maxExecutionTime?     │    │  ║  │
│  ║  │  │ Tool calls: toolCallsCount >= maxToolCalls?             │    │  ║  │
│  ║  │  │                                                         │    │  ║  │
│  ║  │  │ If exceeded → STOP EXECUTION                            │    │  ║  │
│  ║  │  └─────────────────────────────────────────────────────────┘    │  ║  │
│  ║  │       │                                                         │  ║  │
│  ║  │       ▼                                                         │  ║  │
│  ║  │  ┌─────────────────────────────────────────────────────────┐    │  ║  │
│  ║  │  │ EXECUTE STEP                                            │    │  ║  │
│  ║  │  │                                                         │    │  ║  │
│  ║  │  │  ┌───────────────────┐   ┌───────────────────────────┐  │    │  ║  │
│  ║  │  │  │ type: "tool_call" │   │ type: "final_answer"      │  │    │  ║  │
│  ║  │  │  ├───────────────────┤   ├───────────────────────────┤  │    │  ║  │
│  ║  │  │  │                   │   │                           │  │    │  ║  │
│  ║  │  │  │ Find tool by name │   │ Return static response    │  │    │  ║  │
│  ║  │  │  │        ↓          │   │                           │  │    │  ║  │
│  ║  │  │  │ executeTool()     │   │                           │  │    │  ║  │
│  ║  │  │  │        ↓          │   │                           │  │    │  ║  │
│  ║  │  │  │ Return result     │   │                           │  │    │  ║  │
│  ║  │  │  │                   │   │                           │  │    │  ║  │
│  ║  │  │  └───────────────────┘   └───────────────────────────┘  │    │  ║  │
│  ║  │  │                                                         │    │  ║  │
│  ║  │  └─────────────────────────────────────────────────────────┘    │  ║  │
│  ║  │       │                                                         │  ║  │
│  ║  │       ▼                                                         │  ║  │
│  ║  │  ┌─────────────────────────────────────────────────────────┐    │  ║  │
│  ║  │  │ BUILD EXECUTION STEP                                    │    │  ║  │
│  ║  │  │                                                         │    │  ║  │
│  ║  │  │ ExecutionStep {                                         │    │  ║  │
│  ║  │  │   id, timestamp, duration,                              │    │  ║  │
│  ║  │  │   status: 'completed' | 'failed',                       │    │  ║  │
│  ║  │  │   result: <from executeStepAction>,                     │    │  ║  │
│  ║  │  │   metadata: { planStep, stepIndex, success/error }      │    │  ║  │
│  ║  │  │ }                                                       │    │  ║  │
│  ║  │  └─────────────────────────────────────────────────────────┘    │  ║  │
│  ║  │       │                                                         │  ║  │
│  ║  │       ▼                                                         │  ║  │
│  ║  │  UPDATE SESSION via ContextService                              │  ║  │
│  ║  │  - completedSteps                                               │  ║  │
│  ║  │  - currentStep                                                  │  ║  │
│  ║  │  - stepsJournal                                                 │  ║  │
│  ║  │       │                                                         │  ║  │
│  ║  │       ▼                                                         │  ║  │
│  ║  │  EMIT EVENT: agent.step.completed or agent.step.failed          │  ║  │
│  ║  │                                                                 │  ║  │
│  ║  └─────────────────────────────────────────────────────────────────┘  ║  │
│  ║                                                                       ║  │
│  ╚═══════════════════════════════════════════════════════════════════════╝  │
│       │                                                                     │
│       ▼                                                                     │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │ PHASE 3: FINALIZE                                                     │  │
│  │                                                                       │  │
│  │ extractFinalResult() from all steps                                   │  │
│  │                                                                       │  │
│  │ buildFinalResult() {                                                  │  │
│  │   output: "Final answer",                                             │  │
│  │   success: true | false,                                              │  │
│  │   strategy: "plan-execute",                                           │  │
│  │   steps: [all executed steps],                                        │  │
│  │   executionTime: totalMs,                                             │  │
│  │   metadata: {                                                         │  │
│  │     planSteps: originalCount,                                         │  │
│  │     executedSteps: actualCount,                                       │  │
│  │     toolCallsCount                                                    │  │
│  │   }                                                                   │  │
│  │ }                                                                     │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│       │                                                                     │
│       ▼                                                                     │
│  Final Result with Execution Metadata                                       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.2 Configuration

| Parameter | Default | Description |
|-----------|---------|-------------|
| maxPlanningSteps | 10 | Maximum steps in plan |
| maxExecutionSteps | 15 | Maximum execution steps |
| maxToolCalls | 25 | Maximum total tool calls |
| maxExecutionTime | 300,000 | Total timeout (5 min) |
| enablePlanningValidation | true | Validate plan before execution |

---

## Strategy Comparison

| Aspect | ReWoo | ReAct | Plan-Execute |
|--------|-------|-------|--------------|
| **Execution** | Parallel work | Iterative loop | Sequential |
| **Planning** | Sketches upfront | Per-iteration | Complete plan upfront |
| **Best For** | Knowledge synthesis | Complex problem-solving | Structured tasks |
| **Tool Parallelism** | Yes (4 concurrent) | No | No |
| **Adaptability** | Low | High | Low |
| **Observability** | Evidence-based | Step-by-step | Plan + execution |

---

