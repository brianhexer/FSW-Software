# Advanced Features & Extensions

This guide provides code snippets to extend the Film Script Writer with advanced capabilities.

## 1. Character Autocomplete Feature

### Implementation

Add this state to `ScriptEditor` component:

```javascript
const [characterSuggestions, setCharacterSuggestions] = useState([])
const [showCharSuggestions, setShowCharSuggestions] = useState(false)
const [suggestionInput, setSuggestionInput] = useState('')

const handleEditorInput = (e) => {
    handleChange(e)
    
    const text = e.target.value
    const cursorPos = e.target.selectionStart
    const textBefore = text.substring(0, cursorPos)
    const currentLine = textBefore.split('\n').pop()
    
    // Show suggestions if typing a character
    if (detectFormat(currentLine, selectedFormat) === FORMAT_TYPES.CHARACTER) {
        const filtered = characters.filter(char =>
            char.toLowerCase().includes(currentLine.toLowerCase())
        )
        setCharacterSuggestions(filtered.slice(0, 5))
        setShowCharSuggestions(filtered.length > 0)
        setSuggestionInput(currentLine)
    } else {
        setShowCharSuggestions(false)
    }
}

const selectSuggestion = (character) => {
    const editor = editorRef.current
    const text = editor.value
    const cursorPos = editor.selectionStart
    const textBefore = text.substring(0, cursorPos)
    const currentLine = textBefore.split('\n').pop()
    const currentLineStart = cursorPos - currentLine.length
    
    const newText = 
        text.substring(0, currentLineStart) +
        character +
        text.substring(cursorPos)
    
    editor.value = newText
    setScriptContent(newText)
    setShowCharSuggestions(false)
    
    // Move cursor to end of inserted character
    setTimeout(() => {
        editor.selectionStart = editor.selectionEnd = currentLineStart + character.length
    }, 0)
}
```

Replace `onChange={handleChange}` with `onChange={handleEditorInput}` on textarea:

```jsx
<textarea
    ref={editorRef}
    value={scriptContent}
    onChange={handleEditorInput}  // Updated
    onKeyDown={handleKeyDown}
    className="editor-textarea w-full h-full"
/>
```

Add suggestion dropdown above editor:

```jsx
{showCharSuggestions && (
    <div className="absolute bottom-full left-40 right-0 bg-white border border-gray-300 rounded shadow-lg z-50">
        {characterSuggestions.map(char => (
            <div
                key={char}
                className="px-4 py-2 cursor-pointer hover:bg-gray-100 border-b"
                onClick={() => selectSuggestion(char)}
            >
                {char}
            </div>
        ))}
    </div>
)}
```

---

## 2. Firebase Cloud Sync

### Setup

1. Install Firebase (or use CDN):
```html
<script src="https://www.gstatic.com/firebasejs/9.23.0/firebase-app.js"></script>
<script src="https://www.gstatic.com/firebasejs/9.23.0/firebase-firestore.js"></script>
<script src="https://www.gstatic.com/firebasejs/9.23.0/firebase-auth.js"></script>
```

2. Add initialization code:

```javascript
// Your Firebase config from Firebase Console
const firebaseConfig = {
    apiKey: "YOUR_API_KEY",
    authDomain: "your-app.firebaseapp.com",
    projectId: "your-project-id",
    storageBucket: "your-project-id.appspot.com",
    messagingSenderId: "123456789",
    appId: "1:123456789:web:abcdefghijk"
}

firebase.initializeApp(firebaseConfig)
const db = firebase.firestore()
const auth = firebase.auth()
```

### Integration Code

Add to `ScriptEditor`:

```javascript
const [firebaseUser, setFirebaseUser] = useState(null)
const [cloudSyncing, setCloudSyncing] = useState(false)

useEffect(() => {
    // Auth state listener
    auth.onAuthStateChanged(user => {
        if (user) {
            setFirebaseUser(user)
            loadFromFirebase(user.uid)
        } else {
            setFirebaseUser(null)
        }
    })
}, [])

const saveToFirebase = async () => {
    if (!firebaseUser) return
    
    try {
        setCloudSyncing(true)
        await db.collection('scripts').doc(firebaseUser.uid).update({
            [title]: {
                content: scriptContent,
                updatedAt: new Date(),
                pages: stats.pages
            }
        })
        showToast('Cloud sync successful')
    } catch (error) {
        console.error('Firebase save error:', error)
        showToast('Cloud sync failed')
    } finally {
        setCloudSyncing(false)
    }
}

const loadFromFirebase = async (userId) => {
    try {
        const doc = await db.collection('scripts').doc(userId).get()
        if (doc.exists) {
            const data = doc.data()
            console.log('Cloud scripts found:', Object.keys(data))
        }
    } catch (error) {
        console.error('Firebase load error:', error)
    }
}

const signInWithGoogle = async () => {
    try {
        const provider = new firebase.auth.GoogleAuthProvider()
        await auth.signInWithPopup(provider)
    } catch (error) {
        console.error('Auth error:', error)
        showToast('Sign in failed')
    }
}
```

Add button to header:

```jsx
{firebaseUser ? (
    <div className="flex items-center gap-2">
        <span className="text-sm text-gray-600">{firebaseUser.email}</span>
        <button
            className="button-base button-secondary"
            onClick={() => auth.signOut()}
        >
            Sign Out
        </button>
    </div>
) : (
    <button
        className="button-base button-primary"
        onClick={signInWithGoogle}
    >
        Sign In (Cloud Sync)
    </button>
)}
```

---

## 3. Advanced Scene Navigator with Search

### Enhanced Component

```javascript
const AdvancedSceneNavigator = ({ scenes, characters, onSceneClick, currentLineIndex, scriptContent }) => {
    const [filterType, setFilterType] = useState('all') // 'all', 'character', 'location'
    const [searchQuery, setSearchQuery] = useState('')
    const [selectedCharacter, setSelectedCharacter] = useState('')

    const getFilteredScenes = () => {
        let filtered = scenes

        if (filterType === 'character' && selectedCharacter) {
            filtered = scenes.filter(scene => {
                const scenEnd = scene.index + 100
                const sceneText = scriptContent.slice(scene.index, scenEnd)
                return sceneText.includes(selectedCharacter)
            })
        }

        if (searchQuery) {
            filtered = filtered.filter(scene =>
                scene.heading.toLowerCase().includes(searchQuery.toLowerCase())
            )
        }

        return filtered
    }

    const filteredScenes = getFilteredScenes()

    return (
        <div className="sidebar w-80 flex flex-col">
            <div className="p-4 border-b border-gray-600">
                <h2 className="text-sm font-bold uppercase tracking-wide mb-3">Scene Navigator</h2>
                
                <input
                    type="text"
                    placeholder="Search scenes..."
                    className="w-full px-3 py-2 bg-gray-700 text-white rounded text-xs mb-3 focus:outline-none focus:ring-2 focus:ring-blue-500"
                    value={searchQuery}
                    onChange={(e) => setSearchQuery(e.target.value)}
                />

                <div className="flex gap-2 mb-3">
                    <button
                        className={`px-3 py-1 text-xs rounded ${
                            filterType === 'all'
                                ? 'bg-blue-600 text-white'
                                : 'bg-gray-700 text-gray-300 hover:bg-gray-600'
                        }`}
                        onClick={() => setFilterType('all')}
                    >
                        All
                    </button>
                    <button
                        className={`px-3 py-1 text-xs rounded ${
                            filterType === 'character'
                                ? 'bg-blue-600 text-white'
                                : 'bg-gray-700 text-gray-300 hover:bg-gray-600'
                        }`}
                        onClick={() => setFilterType('character')}
                    >
                        By Character
                    </button>
                </div>

                {filterType === 'character' && (
                    <select
                        className="w-full px-3 py-2 bg-gray-700 text-white rounded text-xs focus:outline-none focus:ring-2 focus:ring-blue-500"
                        value={selectedCharacter}
                        onChange={(e) => setSelectedCharacter(e.target.value)}
                    >
                        <option value="">Select character...</option>
                        {characters.map(char => (
                            <option key={char} value={char}>{char}</option>
                        ))}
                    </select>
                )}
            </div>

            <div className="flex-1 overflow-y-auto">
                {filteredScenes.length === 0 ? (
                    <div className="p-4 text-xs text-gray-500">No scenes found</div>
                ) : (
                    filteredScenes.map((scene) => (
                        <div
                            key={scene.index}
                            className={`scene-item text-xs cursor-pointer border-l-3 transition-all ${
                                currentLineIndex >= scene.index
                                    ? 'active bg-gray-700 border-l-yellow-400'
                                    : 'border-l-transparent hover:bg-gray-700 hover:border-l-blue-400'
                            }`}
                            onClick={() => onSceneClick(scene.index)}
                        >
                            <span className="font-bold text-yellow-400 mr-2">
                                #{scene.lineNumber}
                            </span>
                            <span className="truncate">{scene.heading}</span>
                        </div>
                    ))
                )}
            </div>

            <div className="p-3 border-t border-gray-600 text-xs text-gray-400">
                Showing {filteredScenes.length} of {scenes.length} scenes
            </div>
        </div>
    )
}
```

---

## 4. Revision History System

### Local Revision Storage

```javascript
const [revisions, setRevisions] = useState([])
const MAX_REVISIONS = 20

const saveRevision = () => {
    const newRevision = {
        id: Date.now(),
        title: title,
        content: scriptContent,
        timestamp: new Date().toLocaleString(),
        stats: {
            pages: stats.pages,
            scenes: stats.scenes,
            characters: stats.characters,
            lines: stats.lines
        }
    }

    const updated = [newRevision, ...revisions].slice(0, MAX_REVISIONS)
    setRevisions(updated)
    
    // Also save to localStorage
    localStorage.setItem('scriptRevisions', JSON.stringify(updated))
    showToast('Revision saved')
}

const loadRevision = (revisionId) => {
    const revision = revisions.find(r => r.id === revisionId)
    if (revision) {
        setScriptContent(revision.content)
        setTitle(revision.title)
        showToast(`Loaded revision from ${revision.timestamp}`)
    }
}

const clearRevisions = () => {
    if (confirm('Clear all revision history?')) {
        setRevisions([])
        localStorage.removeItem('scriptRevisions')
        showToast('Revision history cleared')
    }
}

// Load revisions on mount
useEffect(() => {
    const saved = localStorage.getItem('scriptRevisions')
    if (saved) {
        try {
            setRevisions(JSON.parse(saved))
        } catch (e) {
            console.error('Failed to load revisions:', e)
        }
    }
}, [])
```

### Revisions UI Component

```jsx
{showRevisionsModal && (
    <div className="modal-overlay" onClick={() => setShowRevisionsModal(false)}>
        <div className="modal" style={{ maxHeight: '80vh', overflowY: 'auto' }} onClick={e => e.stopPropagation()}>
            <div className="modal-title">Revision History</div>
            
            {revisions.length === 0 ? (
                <p className="text-gray-600 mb-4">No revisions saved yet</p>
            ) : (
                <div className="space-y-2 mb-4">
                    {revisions.map(rev => (
                        <div key={rev.id} className="border border-gray-300 rounded p-3 hover:bg-gray-50">
                            <div className="flex justify-between items-start mb-2">
                                <div>
                                    <p className="font-semibold text-sm">{rev.title}</p>
                                    <p className="text-xs text-gray-600">{rev.timestamp}</p>
                                    <p className="text-xs text-gray-500">
                                        {rev.stats.pages}p • {rev.stats.scenes} scenes • {rev.stats.characters} chars
                                    </p>
                                </div>
                                <button
                                    className="button-base button-primary text-xs"
                                    onClick={() => loadRevision(rev.id)}
                                >
                                    Load
                                </button>
                            </div>
                        </div>
                    ))}
                </div>
            )}
            
            <div className="flex gap-3">
                <button
                    className="button-base button-secondary flex-1"
                    onClick={() => setShowRevisionsModal(false)}
                >
                    Close
                </button>
                {revisions.length > 0 && (
                    <button
                        className="button-base button-danger flex-1"
                        onClick={clearRevisions}
                    >
                        Clear All
                    </button>
                )}
            </div>
        </div>
    </div>
)}
```

---

## 5. Export to Multiple Formats

### Markdown Export

```javascript
const exportToMarkdown = () => {
    let markdown = `# ${title}\n\n`
    
    const lines = scriptContent.split('\n')
    let previousFormat = FORMAT_TYPES.ACTION
    
    lines.forEach(line => {
        const format = detectFormat(line, previousFormat)
        const formatted = applyFormatting(line, format)
        
        switch (format) {
            case FORMAT_TYPES.SCENE_HEADING:
                markdown += `## ${formatted}\n\n`
                break
            case FORMAT_TYPES.CHARACTER:
                markdown += `**${formatted}**\n`
                break
            case FORMAT_TYPES.DIALOGUE:
                markdown += `> ${formatted}\n`
                break
            case FORMAT_TYPES.PARENTHETICAL:
                markdown += `*${formatted}*\n`
                break
            case FORMAT_TYPES.ACTION:
                markdown += `${formatted}\n\n`
                break
            default:
                markdown += `${formatted}\n`
        }
        
        if (format !== FORMAT_TYPES.DIALOGUE && format !== FORMAT_TYPES.PARENTHETICAL) {
            previousFormat = format
        }
    })
    
    // Download markdown file
    const element = document.createElement('a')
    element.setAttribute('href', 'data:text/plain;charset=utf-8,' + encodeURIComponent(markdown))
    element.setAttribute('download', `${title.replace(/\s+/g, '-').toLowerCase()}.md`)
    element.style.display = 'none'
    document.body.appendChild(element)
    element.click()
    document.body.removeChild(element)
    
    showToast('Markdown exported')
}
```

### Plain Text Export

```javascript
const exportToPlainText = () => {
    const element = document.createElement('a')
    element.setAttribute('href', 'data:text/plain;charset=utf-8,' + encodeURIComponent(scriptContent))
    element.setAttribute('download', `${title.replace(/\s+/g, '-').toLowerCase()}.txt`)
    element.style.display = 'none'
    document.body.appendChild(element)
    element.click()
    document.body.removeChild(element)
    
    showToast('Text file exported')
}
```

---

## 6. Script Templates

### Template System

```javascript
const SCRIPT_TEMPLATES = {
    blankPage: '',
    
    shortFilm: `FADE IN:

INT. LOCATION - TIME OF DAY

Description of the scene.

CHARACTER
Dialogue here.

FADE OUT.

THE END`,

    actionSequence: `INT. LOCATION - DAY

Description of fast-paced action.

CHARACTER
Quick dialogue!

(action description)

ANOTHER_CHARACTER
Response!

CUT TO:

EXT. NEW_LOCATION - DAY

Continuation of action.

FADE OUT.`,

    dialogueScene: `INT. COFFEE SHOP - MORNING

Two characters sit across from each other.

CHARACTER_A
Opening line of dialogue.

CHARACTER_B
Response line.

CHARACTER_A
(emotional direction)
Continued dialogue.

FADE OUT.`
}

const loadTemplate = (templateName) => {
    if (scriptContent.trim() && !confirm('Replace current script with template?')) {
        return
    }
    
    setScriptContent(SCRIPT_TEMPLATES[templateName] || '')
    setTitle('New Script from Template')
    setSelectedFormat(FORMAT_TYPES.ACTION)
    showToast('Template loaded')
}
```

### Template Selector in Header

```jsx
<select
    className="px-3 py-2 border border-gray-300 rounded text-sm"
    defaultValue="blankPage"
    onChange={(e) => loadTemplate(e.target.value)}
>
    <option value="">Load Template...</option>
    <option value="blankPage">Blank Page</option>
    <option value="shortFilm">Short Film</option>
    <option value="actionSequence">Action Sequence</option>
    <option value="dialogueScene">Dialogue Scene</option>
</select>
```

---

## 7. Word Count & Time Analysis

### Analytics System

```javascript
const calculateAnalytics = () => {
    const words = scriptContent
        .trim()
        .split(/\s+/)
        .filter(w => w.length > 0).length
    
    const dialogueLines = parsed.filter(l => l.format === FORMAT_TYPES.DIALOGUE).length
    const actionLines = parsed.filter(l => l.format === FORMAT_TYPES.ACTION).length
    const characters = new Set(parsed.map(l => l.characters).flat()).size
    
    // Rough screen time estimation (1 line ≈ 0.86 seconds)
    const screenTimeSeconds = parsed.length * 0.86
    const screenTimeMinutes = Math.floor(screenTimeSeconds / 60)
    
    return {
        words,
        dialogueLines,
        actionLines,
        characters,
        screenTimeMinutes,
        screenTimeSeconds: Math.round(screenTimeSeconds % 60)
    }
}

const analytics = calculateAnalytics()
```

### Display in Footer

```jsx
<div className="stats-bar flex items-center justify-between">
    <div className="flex gap-6">
        <span>📄 Pages: ~{stats.pages}</span>
        <span>🎬 Est. Time: {analytics.screenTimeMinutes}:{String(analytics.screenTimeSeconds).padStart(2, '0')}</span>
        <span>📝 Words: {analytics.words}</span>
        <span>💬 Dialogue: {analytics.dialogueLines}L</span>
        <span>🎬 Action: {analytics.actionLines}L</span>
        <span>👥 Characters: {stats.characters}</span>
    </div>
</div>
```

---

## 8. Dark Mode Toggle

### Implementation

```javascript
const [darkMode, setDarkMode] = useState(false)

useEffect(() => {
    const saved = localStorage.getItem('darkMode')
    if (saved) {
        setDarkMode(JSON.parse(saved))
    }
}, [])

useEffect(() => {
    localStorage.setItem('darkMode', JSON.stringify(darkMode))
    if (darkMode) {
        document.documentElement.classList.add('dark')
    } else {
        document.documentElement.classList.remove('dark')
    }
}, [darkMode])

const toggleDarkMode = () => {
    setDarkMode(!darkMode)
}
```

### CSS Classes

```css
.dark .editor-textarea {
    background: #1a1a1a;
    color: #e0e0e0;
}

.dark .sidebar {
    background: #1a1a1a;
    color: #e0e0e0;
}

.dark .modal {
    background: #1a1a1a;
    color: #e0e0e0;
}
```

---

## Implementation Priority

**Phase 1 (High Impact):**
1. Character Autocomplete
2. Advanced Scene Navigator
3. Export to Multiple Formats

**Phase 2 (Important):**
4. Firebase Cloud Sync
5. Revision History
6. Script Templates

**Phase 3 (Nice-to-Have):**
7. Analytics Dashboard
8. Dark Mode
9. Collaboration Features

---

**Need Help?** Each feature can be implemented independently. Start with the ones that add the most value to your workflow!
