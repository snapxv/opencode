# Feature: Reasoning-Phase Tool Calls

## Overview

Enable AI models to call tools during their reasoning/thinking phase, allowing them to enrich their reasoning with real-time data, research, and information gathering.

## Problem Statement

Currently, OpenCode handles reasoning and tool calls as **sequential phases**:

1. Model thinks internally (reasoning phase)
2. Reasoning completes
3. Model makes tool calls (response phase)

This limits the model's ability to:

- Research during thought process
- Verify assumptions with real data
- Gather information before committing to a solution
- Debug issues during reasoning

## Solution

Modified the session processor to allow **interleaved tool calls during reasoning**:

### Key Changes

1. **Track reasoning phase state**
   - Added `isReasoningPhase` flag to track when model is in reasoning
   - Maintains separate `reasoningToolCalls` map for reasoning-phase tool calls

2. **Allow tool calls during reasoning**
   - Modified `tool-call` handler to detect if we're in reasoning phase
   - Route tool calls to appropriate tracker (`reasoningToolCalls` vs `toolcalls`)
   - Mark reasoning-phase tool calls with `reasoningPhase: true` metadata

3. **Incorporate tool results into reasoning**
   - When reasoning-phase tool calls complete, results are automatically appended to current reasoning text
   - This allows the model to see tool results while still thinking

4. **Handle errors gracefully**
   - Tool errors during reasoning are incorporated into reasoning text
   - Reasoning continues even if a tool call fails

## Technical Implementation

### Modified Files

- `packages/opencode/src/session/processor.ts` - Core reasoning-phase tool call support

### New State Variables

```typescript
const reasoningToolCalls: Record<string, MessageV2.ToolPart> = {} // Track tool calls during reasoning phase
let isReasoningPhase = false // Track if we're currently in reasoning phase
```

### Tool Call Flow During Reasoning

1. Model enters reasoning phase (`reasoning-start` event)
2. Model initiates tool call (`tool-call` event)
3. System detects reasoning phase and routes to `reasoningToolCalls`
4. Tool executes and returns result
5. Result is automatically appended to reasoning text
6. Model continues reasoning with new information

### Metadata Enhancement

All reasoning-phase tool calls are marked with:

```typescript
metadata: {
  ...value.providerMetadata,
  reasoningPhase: true, // Marks this as a reasoning-phase tool call
}
```

## Benefits

### For Users

- **Smarter reasoning** - Models can research and verify during thought process
- **More accurate solutions** - Models can check facts before committing
- **Better debugging** - Models can run tests during reasoning
- **Research capabilities** - Models can search docs, Stack Overflow, etc. during thinking

### For Development

- **Context-aware AI** - Models don't need to guess, they can verify
- **Higher quality output** - Reasoning based on actual data
- **Reduced errors** - Verification before implementation

## Example Usage

### Scenario: Debugging a React Component

**Before (Current Behavior):**

1. Model thinks: "The error might be due to state management..."
2. Reasoning ends
3. Model calls `bash` tool to check the code
4. Model implements solution based on guess

**With Reasoning-Phase Tool Calls:**

1. Model thinks: "I need to see the actual error. Let me check the logs..."
2. Model calls `bash` tool during reasoning
3. Model sees error in reasoning: `[Tool: bash] Result: TypeError: Cannot read property 'x' of undefined`
4. Model continues reasoning: "Ah, the issue is undefined property access..."
5. Model implements targeted fix

### Scenario: Learning an API

**Before:**

1. Model thinks: "I should use the fetch API for this..."
2. Reasoning ends
3. Model calls `codesearch` to find examples
4. Model implements based on patterns

**With Reasoning-Phase Tool Calls:**

1. Model thinks: "Let me check the official docs for this API..."
2. Model calls `codesearch` during reasoning
3. Model sees docs in reasoning: `[Tool: codesearch] Result: fetch(url, {method: 'POST', body: ...})`
4. Model continues reasoning: "Perfect, POST with body is supported..."
5. Model implements correct solution

## Future Enhancements

- UI indicators showing reasoning-phase tool calls separately
- Cost tracking for reasoning vs regular tool calls
- Permission controls for reasoning-phase tool access
- Configuration to enable/disable per model
- Performance optimization for concurrent reasoning tool calls

## Compatibility

- ✅ Backward compatible - existing behavior unchanged
- ✅ No breaking changes to API
- ✅ Works with all providers that support thinking/reasoning
- ✅ Transparent to users - models just become more capable

## Testing

- Test with GLM 4.7 (thinking-enabled model)
- Test tool calls during reasoning phase
- Verify tool results incorporate into reasoning text
- Test error handling during reasoning
- Verify cleanup of incomplete reasoning tool calls

## Related Issues

- Addresses user feedback: "During the reasoning phase, the model could use a tool call to enrich its reasoning"
- Enables more intelligent AI assistants

---

**Status:** 🚧 In Development  
**Branch:** `feature/reasoning-phase-tool-calls`  
**Complexity:** Medium  
**Impact:** High
