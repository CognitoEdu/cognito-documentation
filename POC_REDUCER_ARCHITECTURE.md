# Reducer Architecture - Where to Use useReducer

**Question**: How many reducers should we have?  
**Answer**: 2 types of reducers at different levels

---

## Recommended Architecture

### Level 1: Activity Context (ONE reducer for whole activity)

**Location**: `ActivityContext.jsx` in SubtopicElements  
**Scope**: Activity-wide state  
**Instances**: 1 per activity

```javascript
// In SubtopicElements.jsx
<ActivityProvider>
  {/* Whole activity wrapped */}
</ActivityProvider>

// ActivityContext reducer manages:
const activityReducer = (state, action) => {
  switch (action.type) {
    case 'START_ACTIVITY':
    case 'SUBMIT_ELEMENT':
    case 'COMPLETE_ACTIVITY':
    case 'TOGGLE_DISPLAY_MODE':
    case 'RECORD_FLASHCARD_RESPONSE':
    // ... activity-level actions
  }
};
```

**State managed**:
- Activity lifecycle (started, complete)
- All element scores (entire activity's scores)
- Display mode (focus vs all-at-once)
- Activity progress
- Navigation (current element index)

---

### Level 2: Element Context (ONE reducer per element instance)

**Location**: Inside `ElementCard.jsx`  
**Scope**: Single element's state  
**Instances**: 10-20 per activity (one per element)

```javascript
// In ElementCard.jsx (each element creates its own)
function ElementCard({ element }) {
  // Each element has its own reducer!
  const [elementState, elementDispatch] = useReducer(elementReducer, initialState);
  
  return (
    <ElementContext.Provider value={{ state: elementState, dispatch: elementDispatch }}>
      {/* This element's subelements */}
    </ElementContext.Provider>
  );
}

// ElementContext reducer manages (PER ELEMENT):
const elementReducer = (state, action) => {
  switch (action.type) {
    case 'REVEAL_HINT':       // This element's hints
    case 'SHOW_SOLUTION':     // This element's solution
    case 'UPDATE_ANSWER':     // This element's answer
    case 'MARK_COMPLETE':     // This element's completion
    // ... element-specific actions
  }
};
```

**State managed** (per element):
- Hints revealed (for THIS element)
- Solution visibility (for THIS element)
- Answer state (for THIS element)
- Element completion

---

## Visual Comparison

### Current (11 Contexts - BAD!)

```
SubtopicElements
├── ElementsContext (activity scores + FLAGS)
├── ElementsProgressContext (buttons + FLAGS)
├── FlashcardContext (flip state)
├── ProgressBarHeaderContext (header values)
├── FlashcardsProgressBarHeaderContext (flashcard header)
├── MultipartElementsContext (multipart flags)
├── ExamQActivityResetContext (reset trigger)
├── NavigationContext (refresh flags)
├── SystemContext (UI state)
├── CourseContext (course data)
└── LoginContext (user data)

ElementCard
├── Reads from ALL 11 above
├── ElementContext (per element - hints, solution)
├── CarouselContext (per element)
└── HintsContext (per element - duplicate!)
```

**Problems**: 11+ contexts, flag coordination, confusion!

---

### Proposed (3 Contexts - GOOD!)

```
App Level
├── LoginContext (user data - simple state, no reducer)
└── SystemContext (UI state - simple state, no reducer)

Activity Level (SubtopicElements)
└── ActivityContext ← ONE reducer
    ├── Activity lifecycle
    ├── All element scores
    ├── Display mode
    ├── Activity progress
    └── Navigation

Element Level (ElementCard - one per element)
└── ElementContext ← One reducer PER ELEMENT
    ├── Hints (this element's hints)
    ├── Solution visibility
    ├── Answer state
    └── Element completion
```

**Benefits**: Clear hierarchy, related state together, independent where appropriate!

---

## Why Hints Get Their Own Reducer

### Example: Activity with 10 Elements

**Element 1**: User reveals 2 hints  
**Element 5**: User reveals 0 hints  
**Element 8**: User reveals 3 hints and shows solution

**Each element's hint state is INDEPENDENT!**

### ❌ DON'T do this (hints in ActivityContext):

```javascript
// BAD: Activity-level hints
const activityReducer = (state, action) => {
  case 'REVEAL_HINT':
    return {
      ...state,
      elementHints: {
        ...state.elementHints,
        [action.elementId]: {
          hintsRevealed: state.elementHints[action.elementId].hintsRevealed + 1
        }
      }
    };
};
```

**Problems**:
- Activity context knows about ALL element hints (coupling!)
- One element's hint affects activity state (unnecessary re-render of ALL elements)
- Can't reuse ElementCard independently

### ✅ DO this (hints in ElementContext):

```javascript
// GOOD: Element-level hints (in ElementCard)
function ElementCard({ element }) {
  const hints = useElementHints({
    totalHints: element.hints?.length || 0
  });
  
  // Element manages its own hints
  // Other elements don't know, don't care
  
  return (
    <div>
      <button onClick={hints.revealNextHint}>Show Hint</button>
      <p>Hints revealed: {hints.hintsRevealed}</p>
      <p>Marks deducted: {hints.marksDeducted}</p>
    </div>
  );
}
```

**Benefits**:
- Element is self-contained
- No global hint state
- Can reuse ElementCard anywhere
- Revealing hint in Element 1 doesn't affect Element 2

---

## The Rule: "Scope Your State"

### Put state at the LOWEST level that needs it

**If state is used by**:
- ✅ Whole activity → ActivityContext
- ✅ One element → ElementContext (per element)
- ✅ Whole app → App-level context

**Don't**:
- ❌ Put element hints in ActivityContext
- ❌ Put activity scores in ElementContext
- ❌ Put everything in one mega-context

---

## Code Example: Full Picture

### SubtopicElements.jsx (Activity level)

```javascript
function SubtopicElements() {
  // Activity-level state with ONE reducer
  return (
    <ActivityProvider>
      {elements.map(element => (
        <ElementCard key={element.id} element={element} />
      ))}
    </ActivityProvider>
  );
}
```

### ElementCard.jsx (Element level)

```javascript
function ElementCard({ element }) {
  // Element-level state with separate reducer (per element!)
  const hints = useElementHints({
    totalHints: element.hints?.length || 0
  });
  
  // This element manages its own hints
  // Revealing a hint here doesn't affect other elements
  
  return (
    <div>
      {hints.hasHints && (
        <button 
          onClick={hints.revealNextHint}
          disabled={!hints.canRevealMore}
        >
          Reveal Hint ({hints.hintsRevealed}/{hints.totalHints})
        </button>
      )}
      
      {hints.hintsRevealed > 0 && (
        <div>
          {/* Show the hints */}
          <p>Marks deducted: {hints.marksDeducted}</p>
        </div>
      )}
      
      {element.hasSolution && (
        <button onClick={hints.showSolution}>
          Show Solution
        </button>
      )}
      
      {hints.solutionVisible && (
        <div>{/* Show solution */}</div>
      )}
    </div>
  );
}
```

---

## Why This Architecture is Better

### Current (Confused)

```
HintsContext (shared across all elements) ← WRONG!
  - Element 1's hints
  - Element 5's hints  
  - Element 10's hints
  - All mixed together
```

**Problem**: Why does Element 1 need to know about Element 10's hints?

### Proposed (Clear)

```
Element 1: useElementHints → manages Element 1's hints only
Element 2: useElementHints → manages Element 2's hints only
Element 3: useElementHints → manages Element 3's hints only
```

**Benefit**: Independent, reusable, no coupling!

---

## Summary: Reducer Guidelines

### ONE Reducer For:
✅ **ActivityContext** - Activity-wide state (scores, progress, lifecycle)

### MULTIPLE Reducers For:
✅ **ElementContext** - One reducer per element instance (hints, solutions, answers)

### NO Reducer For:
✅ **LoginContext** - Simple user data (useState is fine)  
✅ **SystemContext** - Simple UI flags (useState is fine)

### The Pattern:

**Shared/coordinated state** → ONE reducer (ActivityContext)  
**Independent instance state** → Reducer PER INSTANCE (ElementContext per element)  
**Simple state** → useState (no reducer needed)

---

## This Solves the "11 Contexts" Problem

**Current**: 11 contexts because state is scattered everywhere  
**Proposed**: 3 context types because state is properly scoped

**Activity stuff** → ActivityContext  
**Element stuff** → ElementContext (per element)  
**App stuff** → LoginContext & SystemContext

---

**Does this make sense?** The key insight is: **hints are per-element, so each element should have its own hints reducer**.
