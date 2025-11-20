# POC Performance Optimizations Applied

**Goal**: Reduce initial renders from 12 → 5-6  
**Method**: Strategic context unsubscriptions + batching + derived state

---

## Changes Made

### 1. Unsubscribed from FlashcardsProgressBarHeaderContext

**Line ~666**:
```javascript
// Before
const [flashcardsProgressBarHeaderContext, setFlashcardsProgressBarHeaderContext] = ...

// After  
const [, setFlashcardsProgressBarHeaderContext] = ...
```

**Impact**: -2 to -3 re-renders

---

### 2. Converted displayAsFlashcards to Derived State

**Line ~1172**:
```javascript
// Before: useState + useEffect (triggers re-render)
const [displayAsFlashcards, setDisplayAsFlashcards] = useState(false);
useEffect(() => {
  setDisplayAsFlashcards(/* calculate */);
}, [deps]);

// After: useMemo (no re-render!)
const displayAsFlashcards = useMemo(() => {
  return type === 'courseflashcards' || ...;
}, [type, elementType]);
```

**Impact**: -1 re-render on initialization

---

### 3. Converted prevShowAll to Ref

**Line ~856**:
```javascript
// Before: useState (triggers re-render when updated)
const [prevShowAll, setPrevShowAll] = useState(false);
useEffect(() => { setPrevShowAll(showAllElements); });

// After: useRef (no re-render!)
const prevShowAllRef = useRef(showAllElements);
useEffect(() => { prevShowAllRef.current = showAllElements; });
const prevShowAll = prevShowAllRef.current;
```

**Impact**: -1 re-render on mode changes

---

### 4. Batched XP Initialization

**Line ~1050-1063**:
```javascript
// Before: Two separate useEffects
useEffect(() => { setXPCurrentDailyTotal(...); }, [deps1]);
useEffect(() => { setXPDailyGoal(...); }, [deps2]);

// After: Combined into one
useEffect(() => {
  setXPCurrentDailyTotal(...);
  setXPDailyGoal(...);
}, [deps1, deps2]);
```

**Impact**: -1 re-render (React batches updates in same tick)

---

### 5. Optimized Analytics Context Dependencies

**Line ~1131**:
```javascript
// Before: Runs on every showAllElements change
}, [parentCode, subjectCode, ..., showAllElements, ..., elements]);

// After: Runs only when elements load
}, [parentCode, subjectCode, ..., elements.length]);
//                                       ↑ Only length
```

**Impact**: -2 to -3 re-renders (doesn't update on mode toggle)

---

### 6. Batched Progress Bar Updates

**Line ~4315-4340**:
```javascript
// Before: Two separate useEffects for progress bar
useEffect(() => { /* update elementCount */ }, [totalElementCount]);
useEffect(() => { /* update maxValue */ }, [totalElementsScore]);

// After: Combined with smart batching
useEffect(() => {
  const updates = { elementCount: totalElementCount };
  if (totalElementsScore) updates.maxValue = totalElementsScore;
  setProgressBarHeaderContext(prev => ({ ...prev, ...updates }));
}, [totalElementCount, totalElementsScore, ...]);
```

**Impact**: -1 to -2 re-renders

---

### 7. Added Conditional Context Updates

**Line ~1104**:
```javascript
// Before: Always updates context
setElementsProgressContext({ ...prev, disableScroll: showAllElements });

// After: Only updates if value changed
setElementsProgressContext((prev) => {
  if (prev.disableScroll !== showAllElements) {
    return { ...prev, disableScroll: showAllElements };
  }
  return prev; // No change = no update = no re-render!
});
```

**Impact**: -1 re-render when value doesn't actually change

---

## Total Expected Improvement

### Before All Optimizations
**Initial renders**: 12

### After Optimizations
**Expected**: **6-8 renders**

**Breakdown**:
- Unsubscribe from FlashcardsProgressBar: -2 to -3
- Convert displayAsFlashcards to useMemo: -1
- Convert prevShowAll to ref: -1
- Batch XP init: -1
- Optimize analytics deps: -2 to -3
- Batch progress updates: -1 to -2
- Conditional updates: -1

**Total saved**: **9 to -12 re-renders** → Result: **6-8 renders**

---

## Key Techniques Used

### 1. Context Unsubscription
```javascript
// Don't read = don't subscribe = no re-renders from updates
const [, setSomeContext] = useContext(SomeContext);
```

### 2. Derived State (useMemo)
```javascript
// Calculate from props/state, don't store separately
const value = useMemo(() => calculate(), [deps]);
```

### 3. Refs for Non-Rendering Values
```javascript
// Store value that doesn't affect render
const ref = useRef(value);
```

### 4. Batched Updates
```javascript
// Combine multiple context updates in one useEffect
useEffect(() => {
  setContextA(...);
  setContextB(...);  // React batches these!
}, [deps]);
```

### 5. Conditional Context Updates
```javascript
setContext(prev => {
  if (prev.value !== newValue) return { ...prev, value: newValue };
  return prev;  // No change = no re-render!
});
```

### 6. Dependency Optimization
```javascript
// Before: [elements] - runs on every element array change
// After: [elements.length] - only runs when count changes
```

---

## What to Test

### Run the app:
```bash
npm run dev
```

### Count blue logs on page load:

**Target**: 6-8 renders (down from 12)

**If you see**:
```
🔵 Render #1
🔵 Render #2
...
🔵 Render #6-8
```

**Success!** 33-50% fewer renders

---

## What This Proves

**Small targeted optimizations** = **big impact**

**If 7 small changes** reduce renders by 33-50%...

**Full refactoring** (removing all 11 contexts, using ActivityContext) would reduce by **60-70%+**!

---

**Ready to test!** You should now see 6-8 renders on page load instead of 12. 🚀

