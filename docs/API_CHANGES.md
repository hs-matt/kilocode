# API Changes - State Management Fixes

## Breaking Changes

### Task Provider Access Pattern

**Before**:

```typescript
// Old pattern using WeakRef (DEPRECATED)
const provider = task.providerRef.deref()
if (provider) {
	provider.someMethod()
}
```

**After**:

```typescript
// New pattern using strong reference with disposal checking
const provider = task.getProvider() // Throws if disposed
provider.someMethod()

// Or use the getter for null-safe access
const provider = task.provider // Returns null if disposed
if (provider) {
	provider.someMethod()
}
```

### Task Class Changes

**Removed/Changed**:

- `Task.providerRef: WeakRef<ClineProvider>` → Now private `_provider: ClineProvider`

**Added**:

```typescript
class Task {
	// New safe provider access
	get provider(): ClineProvider | null
	getProvider(): ClineProvider // Throws if disposed

	// Enhanced disposal tracking
	get isDisposed(): boolean
	dispose(): void
}
```

## New APIs

### StateRecoveryManager

```typescript
import { StateRecoveryManager } from "../recovery/StateRecoveryManager"

const recoveryManager = new StateRecoveryManager()

// Recover from various corruption types
await recoveryManager.recoverFromCorruption("STACK_CORRUPTION", {
	provider: clineProvider,
	error: new Error("Stack corruption detected"),
	context: { stackSize: 5, operation: "addTask" },
})
```

**Corruption Types**:

- `STACK_CORRUPTION`: Task stack integrity issues
- `MODE_SWITCH_FAILURE`: Mode switching errors
- `TASK_CREATION_FAILURE`: Task initialization failures
- `PROVIDER_DISPOSAL_FAILURE`: Provider cleanup errors
- `REFERENCE_CORRUPTION`: Reference integrity issues
- `STATE_VALIDATION_FAILURE`: State consistency errors

### CircuitBreaker

```typescript
import { CircuitBreakerManager } from "../recovery/CircuitBreaker"

const manager = new CircuitBreakerManager()
const breaker = manager.getBreaker("stackOperations")

// Execute operation with circuit breaker protection
const result = await breaker.execute(async () => {
	// Your operation here
	return await someRiskyOperation()
})
```

**Available Circuit Breakers**:

- `stackOperations`: Task stack operations
- `modeSwitch`: Mode switching operations
- `taskInitialization`: Task creation/initialization
- `providerOperations`: Provider-level operations
- `stateValidation`: State validation operations

### Enhanced Monitoring

```typescript
import { MonitoringHooks } from "../diagnostics/MonitoringHooks"

// Listen to new monitoring events
MonitoringHooks.on("ATOMIC_OPERATION_START", (data) => {
	console.log("Atomic operation started:", data)
})

MonitoringHooks.on("RECOVERY_TRIGGERED", (data) => {
	console.log("Recovery triggered:", data)
})
```

## Migration Guide

### For Tool Implementations

**Old Pattern**:

```typescript
// attemptCompletionTool.ts (OLD)
const provider = task.providerRef.deref()
if (!provider) {
	throw new Error("Provider reference lost")
}
provider.finishSubTask(result)
```

**New Pattern**:

```typescript
// attemptCompletionTool.ts (NEW)
try {
	const provider = task.getProvider()
	provider.finishSubTask(result)
} catch (error) {
	if (error.message.includes("disposed")) {
		throw new Error("Task has been disposed")
	}
	throw error
}
```

### For Mode Switching

**Old Pattern**:

```typescript
// newTaskTool.ts (OLD)
await provider.handleModeSwitch(mode)
await delay(100) // Insufficient synchronization
```

**New Pattern**:

```typescript
// newTaskTool.ts (NEW)
await provider.handleModeSwitch(mode)
await waitForModeSwitch(provider, mode) // Proper synchronization
```

### For Task Creation

**No Changes Required**: The `createTask` method signature remains the same, but now includes comprehensive error handling and recovery internally.

```typescript
// Still works the same way
const task = await provider.createTask("Some task text", images, parentTask, options)
```

## Error Handling

### New Error Types

The system now provides more specific error handling:

```typescript
try {
	await someOperation()
} catch (error) {
	if (error.message.includes("disposed")) {
		// Task has been disposed
	} else if (error.message.includes("Invalid stack state")) {
		// Stack corruption detected
	} else if (error.message.includes("Mode validation failed")) {
		// Mode switch error
	}
}
```

### Recovery Operations

Recovery is now automatic, but you can also trigger manual recovery:

```typescript
import { StateRecoveryManager } from "../recovery/StateRecoveryManager"

const recovery = new StateRecoveryManager()

try {
	await riskyOperation()
} catch (error) {
	// Manual recovery trigger
	await recovery.recoverFromCorruption("STACK_CORRUPTION", {
		provider: clineProvider,
		error,
		context: { operation: "riskyOperation" },
	})
}
```

## Performance Considerations

### Memory Usage

- **Before**: WeakRef could be garbage collected unexpectedly
- **After**: Strong references use more memory but prevent crashes
- **Mitigation**: Proper disposal tracking prevents memory leaks

### Concurrency

- **Before**: Race conditions in stack operations and mode switching
- **After**: Serialized operations through mutex and queuing
- **Impact**: Slight increase in operation time for improved reliability

### Recovery Overhead

- Circuit breakers add minimal overhead when healthy
- Recovery operations are designed to be fast and non-blocking
- Monitoring events are asynchronous and don't block operations

## Testing

### Unit Tests

```typescript
import { Task } from "../task/Task"
import { StateRecoveryManager } from "../recovery/StateRecoveryManager"

describe("Task Provider Access", () => {
	test("should throw when accessing disposed task provider", () => {
		const task = new Task(/* ... */)
		task.dispose()

		expect(() => task.getProvider()).toThrow("Task has been disposed")
		expect(task.provider).toBeNull()
	})
})
```

### Integration Tests

```typescript
describe("State Recovery", () => {
	test("should recover from stack corruption", async () => {
		const recovery = new StateRecoveryManager()

		// Simulate corruption
		// ...

		await expect(recovery.recoverFromCorruption("STACK_CORRUPTION", context)).resolves.not.toThrow()
	})
})
```

## Backwards Compatibility

### What Still Works

- All public API methods have the same signatures
- Existing task creation code works unchanged
- Mode switching continues to work with enhanced reliability

### What Changed

- Internal provider access pattern (not typically used in external code)
- Enhanced error messages (more specific and helpful)
- Better error recovery (transparent to most users)

## Support

If you encounter issues during migration:

1. Check that you're not directly accessing `task.providerRef`
2. Use `task.getProvider()` for required provider access
3. Use `task.provider` for optional provider access
4. Monitor circuit breaker states if experiencing failures
5. Check logs for recovery operation details

The new system is designed to be more robust while maintaining API compatibility for most use cases.
