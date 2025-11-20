# Submit Flow Analysis - Focus Mode vs All-At-Once Mode

## The Current Mess

### Context 1: ElementsProgressContext (Being Removed)

**Current state** (Lines 1073-1079):
```javascript
const [elementsProgressContext, setElementsProgressContext] = useState({
	buttonLabel: null,              // What button should say
	progressFunction: null,
	elementType: elementType,
	disableScroll: showAllElements,
	activityElementCount: 0,
	
	// ACTION FLAGS (THE PROBLEM!)
	callContinueFromElement: false,  // FLAG: Trigger continue
	showNextElement: false,          // FLAG: Show next element
	submitElement: false,            // FLAG: Trigger submit  ← THIS ONE!
	activeElement: null,             // Which element
	enableSubmitAll: false,          // FLAG: Submit all parts
	submitAllElement: null,
	submitAll: false,
	includesExecuteFunction: false,
	executeFunction: false,
	shouldTextFocus: false,
	updatingElement: false,
});
```

**Used for**: Button coordination between SubtopicElements and ElementCard

---

### Context 2: ElementsContext (Being Partially Removed)

**Current state** (Lines 1060-1071):
```javascript
const [elementsContext, setElementsContext] = useState({
	hideVideoLinks: hideVideoLinks,
	
	// SCORES (KEEP BUT MOVE TO ActivityContext)
	currentScores: {},
	currentScoresTotal: 0,
	currentIncorrectScoresTotal: 0,
	totalScore: 0,
	
	// FLAGS (REMOVE!)
	updateCurrentScoresTotal: false,
	forceUpdateProgressBar: false,
	
	// MORE FLAGS (REMOVE!)
	forceContinue: false,            // FLAG: Auto-progress
	showNextElement: false,
	activeElement: null,
	saveElement: null,               // FLAG: Trigger save
	enableContinue: false,           // FLAG: Enable button
	
	// MULTIPART STATE (KEEP BUT MOVE TO ActivityContext)
	currentMultipartScores: {},
	// ...
});
```

**Used for**: Storing scores + coordinating actions via flags

---

## The Submit Flow Problem

### In Focus Mode (Button in Footer)

**Current flow**:
```
User fills in answer in ElementCard
  ↓
ElementCard updates elementsContext.currentScores (score data)
  ↓
User clicks "Continue" button in FOOTER (SubtopicElements)
  ↓
SubtopicElements.handleFooterButtonClick()
  ↓
Determines if element is complete
  ↓
If needs submission: calls handleContinue('submit')
  ↓
handleContinue() calls saveResult()
  ↓
saveResult() calls saveElementResult()
  ↓
Saves to database
  ↓
Progress to next element
```

**Coordination**: Footer button in SubtopicElements needs to know if current element is ready to submit. Uses `elementsProgressContext.buttonLabel` and flags.

---

### In All-At-Once Mode (Button on ElementCard)

**Current flow**:
```
User fills in answer in ElementCard
  ↓
ElementCard determines element is complete
  ↓
ElementCard enables its own submit button
  ↓
User clicks "Submit" button ON THE ELEMENT CARD
  ↓
ElementCard.handleContinue() called
  ↓
ElementCard SETS FLAG: setElementsProgressContext({
    submitElement: true,
    activeElement: element,
    enableSubmitAll: true
  })
  ↓
SubtopicElements WATCHES FLAG in useEffect
  ↓
useEffect detects submitElement === true
  ↓
Clears flags and calls handleContinue('submit', element)
  ↓
Saves to database
```

**Coordination**: ElementCard can't call save directly, must set flag and wait for SubtopicElements to notice.

---

## The Contexts Being Removed/Consolidated

### Context 1: ElementsProgressContext → ActivityContext

**What we're removing**:
- All the action flags (submitElement, callContinueFromElement, etc.)
- Button label coordination through flags

**What we're replacing with**:
- Direct callbacks in ActivityContext
- Props passing for button state

### Context 2: ElementsContext → ActivityContext

**What we're keeping** (but moving to ActivityContext):
- `currentScores` - The actual scores
- `currentScoresTotal` - Aggregated (now auto-calculated!)
- `totalScore` - Activity total

**What we're removing**:
- All the flags (updateCurrentScoresTotal, forceContinue, saveElement, etc.)

### Context 3: ProgressBarHeaderContext → Derived from ActivityContext

**What we're removing**: Entire context!

**Why**: Progress bar values are **derived** from scores:
```javascript
// Instead of separate context:
const { correctValue, incorrectValue } = useScoreAggregation(scores);
// Pass directly to progress bar as props
<ProgressBar correct={correctValue} incorrect={incorrectValue} />
```

### Context 4: FlashcardContext → ActivityContext.flashcards

**What we're consolidating**:
- Flashcard flip state
- Move into ActivityContext as `flashcards: { [id]: { side: 'front' } }`

---

## Proposed Submit Flow (Better!)

### In Focus Mode - Footer Button

```javascript
// SubtopicElements
function SubtopicElements() {
  const { submitCurrentElement, currentElement } = useActivity();
  
  return (
    <div>
      {/* Elements */}
      
      {/* Footer with submit button */}
      <button 
        onClick={() => submitCurrentElement()}
        disabled={!currentElement?.isReadyToSubmit}
      >
        Continue
      </button>
    </div>
  );
}

// ActivityContext provides submitCurrentElement method
const submitCurrentElement = () => {
  const element = elements[currentElementIndex];
  submitElement(element.id);  // Direct call!
  moveToNextElement();
};
```

**No flags!** Direct method call.

---

### In All-At-Once Mode - Button on Card

```javascript
// ElementCard
function ElementCard({ element }) {
  const { submitElement } = useActivity();  // Get from context
  
  return (
    <div>
      {/* Element content */}
      
      {/* Submit button on the card */}
      <button 
        onClick={() => submitElement(element.id)}
        disabled={!isElementComplete}
      >
        Submit
      </button>
    </div>
  );
}
```

**Same method** (`submitElement`), just called from different place!

**No flags!** ElementCard calls directly, SubtopicElements handles it immediately.

---

## How ActivityContext Handles Both Modes

```javascript
// ActivityContext.jsx
export function ActivityProvider({ children, onSubmitElement }) {
  const [state, dispatch] = useReducer(activityReducer, {
    displayMode: 'focus',  // or 'all-at-once'
    currentElementIndex: 0,
    elementScores: {},
  });
  
  // ONE method that works for BOTH modes
  const submitElement = useCallback((elementId, submitAll = false) => {
    // 1. Update state
    dispatch({ type: 'SUBMIT_ELEMENT', elementId });
    
    // 2. Save to database
    onSubmitElement(elementId, submitAll);
    
    // 3. If focus mode, auto-progress to next
    if (state.displayMode === 'focus') {
      dispatch({ type: 'MOVE_TO_NEXT_ELEMENT' });
    }
    
    // If all-at-once mode, just save (don't progress)
  }, [state.displayMode, onSubmitElement]);
  
  // For focus mode footer button
  const submitCurrentElement = useCallback(() => {
    const currentId = elements[state.currentElementIndex]?.id;
    if (currentId) {
      submitElement(currentId);
    }
  }, [state.currentElementIndex, elements, submitElement]);
  
  return (
    <ActivityContext.Provider value={{
      ...state,
      submitElement,           // For all-at-once mode (ElementCard calls this)
      submitCurrentElement,    // For focus mode (Footer calls this)
    }}>
      {children}
    </ActivityContext.Provider>
  );
}
```

---

## The Beauty of This Approach

### One Submit Method, Two Entry Points

**Focus mode**:
```javascript
// Footer button in SubtopicElements
<button onClick={submitCurrentElement}>Continue</button>
```

**All-at-once mode**:
```javascript
// Submit button in ElementCard
<button onClick={() => submitElement(element.id)}>Submit</button>
```

**Both end up calling the SAME LOGIC**, just different entry points!

---

## Contexts Consolidation Plan

### Current 11 Contexts → 3 Contexts

**Removing/Consolidating**:
1. ❌ **ElementsProgressContext** → ActivityContext (button coordination via callbacks)
2. ❌ **ElementsContext** (scores only) → ActivityContext.scores
3. ❌ **ProgressBarHeaderContext** → Derived from ActivityContext.scores (no separate context!)
4. ❌ **FlashcardsProgressBarHeaderContext** → Derived from ActivityContext.flashcards
5. ❌ **FlashcardContext** → ActivityContext.flashcards
6. ❌ **MultipartElementsContext** → ActivityContext.multipartState
7. ❌ **NavigationContext** (refresh flags) → Simple state or props
8. ✅ **ExamQActivityResetContext** → Keep (but could merge into ActivityContext)

**Keeping**:
9. ✅ **CourseContext** - Course metadata (app level)
10. ✅ **SystemContext** - UI state (app level)
11. ✅ **LoginContext** - User data (app level)

**New**:
12. ✅ **ActivityContext** - All activity state (ONE reducer)
13. ✅ **ElementContext** - Per-element state (one reducer per element, managed by hook)

**Final count**: **3 context types** (Activity, Login, System) + per-element contexts

---

## Which Context Should We Convert Next?

### Option A: ProgressBarHeaderContext → Derived State

**Current**: Separate context for progress bar values  
**Proposed**: Calculate from ActivityContext.scores automatically

**Benefit**: Removes entire context! Progress auto-updates.

**Code**:
```javascript
// No more ProgressBarHeaderContext!
// Just calculate from scores:
const { totalCorrect, totalIncorrect } = useScoreAggregation(scores);

<ProgressBar 
  correct={totalCorrect} 
  incorrect={totalIncorrect} 
  max={totalPossible}
/>
```

### Option B: FlashcardContext → ActivityContext.flashcards

**Current**: Separate context for flip state  
**Proposed**: Move into ActivityContext

**Benefit**: One less context, flashcard state in logical place

---

## My Recommendation for Next POC Step

**Convert ProgressBarHeaderContext to derived state**

**Why**:
- Demonstrates removing an entire context
- Shows derived state pattern (no context needed!)
- Quick win (30 mins)
- High impact (progress bar is critical)

**Want me to do that?** Or prefer to stop the POC here and test what we have?
