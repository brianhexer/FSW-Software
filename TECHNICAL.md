# Technical Documentation

## Architecture Overview

The Film Script Writer is a single-file React application that handles screenplay formatting, parsing, storage, and export.

```
┌─────────────────────────────────────────────────────────┐
│                  React Application                       │
│              (script-editor.html)                        │
└─────────────────────────────────────────────────────────┘
          ↓
    ┌─────────────────────────────────────────────┐
    │         State Management (React Hooks)      │
    │  ├─ scriptContent (string)                 │
    │  ├─ title (string)                         │
    │  ├─ isDirty (boolean)                      │
    │  └─ UI state (modals, toasts)              │
    └─────────────────────────────────────────────┘
          ↓
    ┌─────────────────────────────────────────────┐
    │      Script Processing Pipeline             │
    │  ├─ detectFormat()                         │
    │  ├─ parseScript()                          │
    │  ├─ extractScenes()                        │
    │  ├─ extractCharacters()                    │
    │  └─ getScriptStats()                       │
    └─────────────────────────────────────────────┘
          ↓
    ┌─────────────────────────────────────────────┐
    │         Storage & Export Layers             │
    │  ├─ localStorage (auto-save)               │
    │  ├─ PDF export (html2pdf.js)               │
    │  └─ Scene navigation (sidebar)             │
    └─────────────────────────────────────────────┘
```

## Core Modules

### 1. Script Formatting Utilities

#### `FORMAT_TYPES`
```javascript
const FORMAT_TYPES = {
    SCENE_HEADING: 'scene-heading',
    ACTION: 'action',
    CHARACTER: 'character',
    PARENTHETICAL: 'parenthetical',
    DIALOGUE: 'dialogue',
    TRANSITION: 'transition',
}
```

#### `detectFormat(text, previousFormat) → string`
Analyzes a line of text to determine its format type.

**Algorithm:**
```
1. If empty → return ACTION
2. Check for scene heading patterns:
   - Starts with INT., EXT., INT/EXT., I/E.
   - Contains timecode format (INT BEDROOM - DAY)
3. If all caps and ends with "TO:" → TRANSITION
4. If surrounded by parentheses → PARENTHETICAL
5. Context-based detection based on previous format
6. If all caps and short → CHARACTER
7. Default → ACTION
```

**Parameters:**
- `text` (string): Line content
- `previousFormat` (string): Format of previous line

**Returns:** Format type constant

**Example:**
```javascript
detectFormat("INT. BEDROOM - NIGHT", FORMAT_TYPES.ACTION)
// Returns: FORMAT_TYPES.SCENE_HEADING

detectFormat("JOHN", FORMAT_TYPES.ACTION)
// Returns: FORMAT_TYPES.CHARACTER

detectFormat("Hello, how are you?", FORMAT_TYPES.CHARACTER)
// Returns: FORMAT_TYPES.DIALOGUE
```

#### `applyFormatting(text, format) → string`
Applies visual formatting rules to text.

**Transforms:**
- SCENE_HEADING → `text.toUpperCase()`
- CHARACTER → `text.toUpperCase().trim()`
- Others → `text` (unchanged)

#### `getIndentation(format) → number`
Returns indentation in inches for each format type.

```javascript
{
    SCENE_HEADING: 0,
    ACTION: 0,
    CHARACTER: 3.7,
    DIALOGUE: 1.5,
    PARENTHETICAL: 2.2,
    TRANSITION: 0
}
```

### 2. Script Parser

#### `parseScript(text) → Array<ScriptLine>`
Converts raw text into structured script data.

**Returns:**
```javascript
[
    {
        index: 0,
        original: "INT. BEDROOM - NIGHT",
        formatted: "INT. BEDROOM - NIGHT",
        format: "scene-heading",
        characters: []
    },
    // ... more lines
]
```

**Algorithm:**
1. Split text by newlines
2. Iterate through each line
3. Detect format with context (previous line format)
4. Apply formatting rules
5. Extract characters if format is CHARACTER

### 3. Scene & Character Extraction

#### `extractScenes(parsed) → Array<Scene>`
Filters scene headings from parsed data.

```javascript
{
    index: 0,        // Line number in full text
    heading: "INT. BEDROOM - NIGHT",
    lineNumber: 1    // Sequential scene number
}
```

#### `extractCharacters(parsed) → Array<string>`
Deduplicates character names from parsed data.

```javascript
["JOHN", "SARAH", "OFFICER"]
```

### 4. Statistics System

#### `getScriptStats(parsed) → ScriptStats`
Calculates script metrics.

```javascript
{
    pages: 5,        // Estimated (line_count * 12 / 55)
    lines: 275,
    scenes: 8,
    characters: 3
}
```

**Page Calculation Formula:**
```
pages = Math.ceil((lines * 12) / 55)
```
Based on industry standard: ~55 lines per page, 12pt font.

## State Management

### React Hooks Usage

```javascript
const [scriptContent, setScriptContent] = useState('')
const [title, setTitle] = useState('Untitled Script')
const [isDirty, setIsDirty] = useState(false)
const [showSaveModal, setShowSaveModal] = useState(false)
const [toast, setToast] = useState(null)
const [selectedFormat, setSelectedFormat] = useState(FORMAT_TYPES.ACTION)

const editorRef = useRef(null)
const cursorPositionRef = useRef(0)
```

### State Flow

```
User Input
    ↓
handleChange()
    ↓
setScriptContent() → Triggers re-render
    ↓
parseScript() → Memoized computation
    ↓
extractScenes/Characters/Stats
    ↓
Component Re-render
    ↓
Auto-save trigger (if isDirty)
```

## Event Handling

### Keyboard Events

#### `handleKeyDown(event)`

**Key Processing:**
1. **Tab Key**
   - Prevents default behavior
   - Gets current line and format
   - Adds smart indentation
   - Updates cursor position

2. **Enter Key**
   - If after CHARACTER → Auto-format for DIALOGUE
   - If in DIALOGUE → Stay in DIALOGUE
   - Otherwise → Return to ACTION
   - Adds proper indentation

3. **Ctrl+P / Cmd+P**
   - Triggers `exportToPDF()`

4. **Ctrl+Shift+S / Cmd+Shift+S**
   - Opens save modal

#### `handleChange(event)`
- Updates `scriptContent` state
- Sets `isDirty = true`
- Tracks cursor position

### Storage Events

```javascript
useEffect(() => {
    // Load on mount
    const saved = localStorage.getItem('filmScriptData')
    if (saved) {
        const { content, title } = JSON.parse(saved)
        setScriptContent(content)
        setTitle(title)
    }
}, [])

useEffect(() => {
    // Auto-save on dirty
    if (isDirty) {
        const timer = setTimeout(() => {
            localStorage.setItem('filmScriptData', 
                JSON.stringify({ content: scriptContent, title }))
            setIsDirty(false)
        }, 2000) // 2-second debounce
        
        return () => clearTimeout(timer)
    }
}, [scriptContent, title, isDirty])
```

## PDF Export System

### Export Flow

```
exportToPDF()
    ↓
Create container div
    ↓
Build HTML from script
    ↓
Parse script lines with format detection
    ↓
Apply CSS indentation and styling
    ↓
Convert HTML to PDF using html2pdf.js
    ↓
Download file with title as filename
    ↓
Show toast notification
```

### PDF Generation

```javascript
const opt = {
    margin: [0.5, 0.5, 0.5, 0.5],      // 0.5" margins
    filename: 'title.pdf',
    image: { type: 'jpeg', quality: 0.98 },
    html2canvas: { scale: 2 },
    jsPDF: { 
        unit: 'in', 
        format: 'letter', 
        orientation: 'portrait' 
    },
}
```

### CSS for PDF Styling

```css
.pdf-page {
    width: 8.5in;
    height: 11in;
    padding: 1in;
    font-family: 'Courier Prime', monospace;
    font-size: 12pt;
    line-height: 1.5;
}
```

## Component Hierarchy

```
ScriptEditor (Main Component)
    ├── SceneNavigator (Sidebar)
    │   └── scene-item (repeating)
    ├── Header
    │   ├── Title Display
    │   ├── Dirty State Indicator
    │   └── Action Buttons
    ├── Editor Area
    │   └── textarea (ref: editorRef)
    ├── Statistics Bar
    │   ├── Page Count
    │   ├── Scene Count
    │   ├── Character Count
    │   └── Line Count
    ├── Save Modal
    │   ├── Title Input
    │   └── Action Buttons
    └── Toast Notification
```

## Extending the Application

### Adding New Format Types

1. Add to `FORMAT_TYPES`:
```javascript
const FORMAT_TYPES = {
    // ... existing
    SLUGLINE: 'slugline',
}
```

2. Update `detectFormat()`:
```javascript
if (trimmed.match(/^SLUGLINE PATTERN/)) {
    return FORMAT_TYPES.SLUGLINE
}
```

3. Add formatting in `applyFormatting()`:
```javascript
case FORMAT_TYPES.SLUGLINE:
    return text.toUpperCase()
```

4. Add indentation in `getIndentation()`:
```javascript
case FORMAT_TYPES.SLUGLINE:
    return 0
```

5. Add CSS styling in `<style>`:
```css
.slugline {
    /* your styles */
}
```

### Adding Character Autocomplete

```javascript
const [suggestions, setSuggestions] = useState([])
const [showSuggestions, setShowSuggestions] = useState(false)

const handleCharacterInput = (e) => {
    const text = e.target.value
    const format = detectFormat(text, selectedFormat)
    
    if (format === FORMAT_TYPES.CHARACTER) {
        const filtered = characters.filter(char =>
            char.includes(text.toUpperCase())
        )
        setSuggestions(filtered)
        setShowSuggestions(true)
    } else {
        setShowSuggestions(false)
    }
}

const selectCharacter = (char) => {
    // Insert character, update textarea
}
```

### Adding Cloud Sync (Firebase Example)

```javascript
import { initializeApp } from 'firebase/app'
import { getFirestore, setDoc, getDoc } from 'firebase/firestore'

const firebaseConfig = { /* ... */ }
const app = initializeApp(firebaseConfig)
const db = getFirestore(app)

const saveToFirebase = async (userId, title, content) => {
    await setDoc(doc(db, 'scripts', userId), {
        title,
        content,
        timestamp: new Date()
    })
}

const loadFromFirebase = async (userId) => {
    const docSnap = await getDoc(doc(db, 'scripts', userId))
    return docSnap.data()
}
```

### Adding Scene Filtering

```javascript
const [filterCharacter, setFilterCharacter] = useState('')

const getFilteredScenes = () => {
    if (!filterCharacter) return scenes
    
    return scenes.filter(scene => {
        const sceneLines = parsed.slice(scene.index, nextSceneIndex)
        return sceneLines.some(line =>
            line.characters.includes(filterCharacter)
        )
    })
}
```

## Performance Considerations

### Optimization Techniques

1. **Debounced Auto-Save**
   - 2-second debounce prevents excessive localStorage writes
   - Improves performance for fast typers

2. **Memoized Computations**
   ```javascript
   const parsed = parseScript(scriptContent)  // Recalculates on content change
   const scenes = extractScenes(parsed)       // Dependent on parsed
   ```

3. **Ref-Based Cursor Tracking**
   ```javascript
   cursorPositionRef.current = e.target.selectionStart
   // Avoids state update for frequent cursor changes
   ```

4. **Efficient String Splitting**
   ```javascript
   const lines = scriptContent.split('\n')  // O(n) operation
   ```

### Scalability

- **Current limits:** Tested with scripts up to 50,000 lines
- **Format detection:** O(n) where n = lines in script
- **PDF export:** Linear time relative to script length
- **localStorage:** Typically 5-10MB limit per domain

## Testing

### Unit Test Examples

```javascript
// Format Detection
test('detects scene heading', () => {
    const result = detectFormat("INT. BEDROOM - NIGHT", FORMAT_TYPES.ACTION)
    expect(result).toBe(FORMAT_TYPES.SCENE_HEADING)
})

// Character Extraction
test('extracts characters', () => {
    const parsed = parseScript("JOHN\nHello\n\nSARAH\nHi")
    const characters = extractCharacters(parsed)
    expect(characters).toContain('JOHN')
    expect(characters).toContain('SARAH')
})

// Format Application
test('applies uppercase to scene headings', () => {
    const result = applyFormatting("int. bedroom - night", FORMAT_TYPES.SCENE_HEADING)
    expect(result).toBe("INT. BEDROOM - NIGHT")
})
```

## Browser APIs Used

1. **localStorage**
   - Persistent client-side storage
   - ~5-10MB limit
   - No expiration

2. **textarea.selectionStart/selectionEnd**
   - Get/set cursor position
   - Enable smart cursor placement

3. **document.createElement**
   - Create PDF export container
   - Hidden from view

## Dependencies

| Library | Purpose | CDN |
|---------|---------|-----|
| React 18 | UI framework | unpkg |
| React-DOM 18 | DOM rendering | unpkg |
| Tailwind CSS 3 | Styling | cdn.tailwindcss.com |
| html2pdf.js | PDF export | cdnjs.cloudflare.com |
| Courier Prime | Typography | fonts.googleapis.com |

## Future Roadmap

- [ ] Real-time collaboration (WebSocket)
- [ ] Multi-scene editing mode
- [ ] Script templates library
- [ ] Revision history (git-style)
- [ ] Advanced analytics dashboard
- [ ] Mobile app (React Native)
- [ ] AI-powered format suggestions
- [ ] Integration with production management software
- [ ] Voice-to-text scripting
- [ ] Multi-language support

---

**Version:** 1.0.0
**Last Updated:** May 7, 2026
**Maintainers:** Film Script Writer Community
