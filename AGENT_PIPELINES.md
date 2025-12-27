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

