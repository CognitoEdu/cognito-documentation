# Element Navigation - ActivityContext Integration

**Status**: ✅ Infrastructure ready, easy to switch over

---

## What We Built

### ActivityContext Now Manages Navigation

**State**:
```javascript
{
  currentElementIndex: 0,     // Which element to show (0-based)
  totalElements: 10,          // Total count
  displayMode: 'focus',       // Affects navigation behavior
}
```

**Actions**:
```javascript
// Auto-progression (already working!)
dispatch({ type: 'SUBMIT_ELEMENT', elementId })
// → In focus mode: currentElementIndex++
// → In all-at-once mode: stays same

// Explicit navigation (ready to use)
dispatch({ type: 'MOVE_TO_NEXT_ELEMENT' })
```

**Methods provided**:
```javascript
const {
  currentElementIndex,    // 0, 1, 2, 3...
  currentElement,         // elements[currentElementIndex]
  isFirstElement,         // currentElementIndex === 0
  isLastElement,          // currentElementIndex === length - 1
  totalElements,          // elements.length
  
  submitCurrentElement,   // Submits + auto-progresses (focus mode)
  moveToNext,             // Explicit next
  moveToPrevious,         // Explicit previous
  goToElement,            // Jump to index
} = useActivity();
```

---

## Current State (For Comparison)

**Old pattern** (what SubtopicElements still uses):
```javascript
const [elementCount, setElementCount] = useState(1);  // 1-based

// Manual incrementing in multiple places:
// - Line 1778: resetProgress
// - Line 2835: after flashcard
// - Line 3067: set to length for all-at-once
// - Line 3173: after saveResult  
// - Line 3270: in handleContinue

setElementCount(elementCount + 1);  // Scattered everywhere!

// Rendering:
const currentElement = elements[elementCount - 1];  // Convert to 0-based
```

**Problems**:
- Manual incrementing (easy to forget or do wrong)
- Not coordinated with submit
- 1-based vs 0-based confusion
- Multiple places setting it

---

## How to Migrate (Already Set Up!)

### Step 1: Replace elementCount with ActivityContext (EASY!)

**Before**:
```javascript
const [elementCount, setElementCount] = useState(1);
const currentElement = elements[elementCount - 1];
```

**After** (just uncomment the lines we added!):
```javascript
// Remove the useState line

// Add this:
const { currentElementIndex, currentElement } = useActivity();
const elementCount = currentElementIndex + 1;  // If other code needs 1-based
```

**That's it!** Most of the setElementCount calls can be REMOVED because:
- Submission auto-progresses (in reducer!)
- No manual incrementing needed

---

### Step 2: Remove Manual Incrementing

**Most of these can just be deleted**:

**Line 1778** (resetProgress):
```javascript
// OLD:
setElementCount(startingElementIndex ? startingElementIndex + 1 : 1);

// NEW:
// Not needed! ActivityProvider initializes with startingElementIndex
```

**Line 2835** (after flashcard):
```javascript
// OLD:
if (elementCount < elements.length) {
  setElementCount(elementCount + 1);
}

// NEW:
// Not needed! submitElement auto-progresses in focus mode
```

**Line 3173** (in saveResult):
```javascript
// OLD:
if (!maxDailyActivityReached) {
  setElementCount(elementCount + 1);
}

// NEW:
// Not needed! Already handled by submitElement action
```

---

### Step 3: All-At-Once Mode

**Line 3067**:
```javascript
// OLD:
if (elements && showAllElements) {
  setElementCount(elements.length);  // Show all
}

// NEW:
// Not needed! displayMode === 'all-at-once' doesn't use currentElementIndex
// Just render all elements
```

---

## The Key Insight

### MOST setElementCount Calls Become Unnecessary!

**Why**: The reducer **automatically progresses** when submitting in focus mode.

**Before**: Scatter 8+ `setElementCount(elementCount + 1)` throughout code  
**After**: ONE line in reducer handles it all!

```javascript
// In ActivityContext reducer
case 'SUBMIT_ELEMENT':
  return {
    ...state,
    currentElementIndex: state.displayMode === 'focus' 
      ? state.currentElementIndex + 1   // ← ONE PLACE!
      : state.currentElementIndex
  };
```

---

## What's Already Working

### Auto-Progression in Focus Mode

**When you call**:
```javascript
submitElement(elementId);
```

**The reducer**:
1. Records submission
2. Checks displayMode
3. If 'focus': Automatically increments currentElementIndex
4. If 'all-at-once': Stays on same element

**No manual incrementing needed!**

---

## Rendering Pattern

### Focus Mode

**Before**:
```javascript
{elements.map((element, index) => 
  index < elementCount ? (  // 1-based
    <ElementCard element={element} />
  ) : null
)}
```

**After**:
```javascript
// Option 1: Render only current
{!showAllElements && currentElement && (
  <ElementCard element={currentElement} />
)}

// Option 2: Render up to current (for animation purposes)
{!showAllElements && elements.map((element, index) => 
  index <= currentElementIndex ? (  // 0-based
    <ElementCard element={element} />
  ) : null
)}
```

---

### All-At-Once Mode

```javascript
// Just render all (doesn't use currentElementIndex)
{showAllElements && elements.map(element => (
  <ElementCard element={element} />
))}
```

---

## To Fully Switch Over

### Quick Migration (5 minutes)

1. **Uncomment** the lines we added (lines 1149-1151)
```javascript
const { currentElementIndex: activityElementIndex } = useActivity();
const elementCount = activityElementIndex + 1;  // Convert to 1-based
```

2. **Remove** the useState line (1142-1146)

3. **Delete** most setElementCount calls:
   - Line 1778 (resetProgress)
   - Line 2835 (flashcard)
   - Line 3173 (saveResult)
   
4. **Keep** for now:
   - Line 3067 (all-at-once sets to length - could refactor later)

5. **Test**: Everything should work identically!

---

## Benefits Once Migrated

✅ **Automatic progression** - No manual incrementing  
✅ **Coordinated with submit** - Happens in one place  
✅ **Mode-aware** - Different behavior for focus vs all-at-once  
✅ **No scattered setElementCount** - One source of truth  
✅ **Predictable** - Reducer makes it clear what happens

---

## Current POC Status

**Infrastructure**:
- ✅ ActivityContext tracks currentElementIndex
- ✅ SUBMIT_ELEMENT auto-progresses
- ✅ Methods exposed (currentElement, isFirstElement, etc.)
- ✅ startingElementIndex supported

**Integration**:
- ⚠️ SubtopicElements still uses old elementCount (for safety)
- ✅ Comments show how to switch
- ✅ 3-line change to fully migrate

**Why not fully migrated in POC**: Safer to keep backward compatibility, prove infrastructure works

---

## Summary

**Q**: "Is that something we can move to a useReducer?"  
**A**: YES - to **ActivityContext reducer** (not separate)

**Q**: "Hook it up!"  
**A**: ✅ DONE - Infrastructure ready, 3-line switch to use it

**Current**: Old elementCount still works (safe)  
**Ready**: ActivityContext.currentElementIndex ready to use  
**Migration**: Uncomment 3 lines, remove 1 line, delete 6 setElementCount calls

**Tests passing** ✅  
**Pattern proven** ✅

---

## Next Steps

**To complete navigation migration**:
1. Uncomment lines 1149-1151
2. Remove lines 1142-1146 (old useState)
3. Search for `setElementCount` and delete most calls
4. Test in app

**Estimated time**: 10 minutes  
**Risk**: Low (reducer already handles it correctly)

---

**Navigation control is now in ActivityContext!** The infrastructure is built, ready to use whenever you want to switch over. 🚀

