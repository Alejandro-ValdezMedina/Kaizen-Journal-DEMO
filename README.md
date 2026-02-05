# Kaizen Journal DEMO

> **[Live Application](https://kaizen-journal.vercel.app/)** | **[Demo Video]()**

A Progressive Web App (PWA) for daily self-improvement journaling, inspired by the Japanese philosophy of Kaizen (continuous improvement). Track your daily progress in Learning, Exercise, and Mindfulness with real-time synchronization, social motivation features, and a beautiful, responsive UI.

## Project Overview

Kaizen Journal is a full-stack web application that helps users build consistent habits through daily journaling. The app features real-time data synchronization with Firebase, a social component where friends can send encouragement, and a modern, accessible user interface.

## Key Features

- **Daily Journal Entries** - Track learning, exercise, and mindfulness activities
- **Calendar View** - Browse and review past entries chronologically
- **Daily Motivational Quotes** - Inspiring quotes that change once per day
- **Social Motivation** - Friends can send personalized encouragement via shareable links (no account required)
- **Secure Authentication** - Email/Password and Google OAuth integration
- **Progressive Web App** - Installable on mobile and desktop with offline capabilities

## Tech Stack

- **Frontend:** React 19 + Vite 7
- **Routing:** React Router DOM v7
- **Backend:** Firebase (Authentication + Firestore)
- **Hosting:** Vercel
- **PWA:** vite-plugin-pwa
- **Styling:** CSS3 with CSS Variables, Animations, and Responsive Design

## Architecture & Design Decisions

### Component Structure

The application follows a component-based architecture with clear separation of concerns:

- **Pages** - Top-level route components (HomePage, CalendarPage, LoginPage, etc.)
- **Components** - Reusable UI components (CategoryInput, EntryCard)
- **Firebase Config** - Centralized Firebase initialization and exports

### State Management

- **Local State** - `useState` for component-level state
- **Real-time Data** - Firebase `onSnapshot` listeners for live updates
- **Memoization** - `useMemo` for expensive calculations (daily quote selection)

### Authentication Flow

Protected routes are handled at the App level using Firebase's `onAuthStateChanged` listener, ensuring users are authenticated before accessing journal features while allowing public access to the motivation page.

## Code Highlights

### 1. Protected Routes with Authentication

```
//App.jsx
function App() {
  const [user, setUser] = useState(null)
  const [loading, setLoading] = useState(true)
  
  useEffect(() => {
    const unsubscribe = onAuthStateChanged(auth, (currentUser) => {
      setUser(currentUser)
      setLoading(false)
    })
    return () => unsubscribe()
  }, [])

  return (
    <BrowserRouter>
      <Routes>
        <Route path="/motivate/:userId" element={<MotivatePage />} />
        <Route path="/" element={user ? <HomePage /> : <LoginPage />} />
        <Route path="/calendar" element={user ? <CalendarPage /> : <LoginPage />} />
      </Routes>
    </BrowserRouter>
  )
}
```

**Why this approach?** Centralizing auth state in the App component allows for consistent route protection and prevents code duplication. The loading state ensures a smooth user experience during authentication checks.

### 2. Real-time Data Synchronization

```
//HomePage.jsx
useEffect(() => {
  const user = auth.currentUser
  if (!user) return

  const q = query(
    collection(db, 'entries'),
    where('userId', '==', user.uid),
    orderBy('createdAt', 'desc')
  )

  const unsubscribe = onSnapshot(q, (snapshot) => {
    if (isMounted.current) {
      const entriesData = snapshot.docs.map(doc => ({
        id: doc.id,
        ...doc.data()
      }))
      setEntries(entriesData)
    }
  })

  return () => {
    isMounted.current = false
    if (unsubscribe) unsubscribe()
  }
}, [])
```

**Key considerations:**
- `isMounted` ref prevents state updates after component unmount
- Proper cleanup with `unsubscribe()` prevents memory leaks
- Real-time updates provide instant feedback without page refreshes

### 3. Auto-expanding Textarea Component

```
//CategoryInput.jsx (height adjustments)
function CategoryInput({ label, value, onChange, icon, placeholder }) {
  const textareaRef = useRef(null)

  const adjustHeight = () => {
    const textarea = textareaRef.current
    if (textarea) {
      textarea.style.height = 'auto'
      //max at 300px max, then scrolling kicks in
      textarea.style.height = `${Math.min(300, Math.max(80, textarea.scrollHeight))}px`
    }
  }

  useEffect(() => {
    adjustHeight()
  }, [value])

  const handleChange = (e) => {
    onChange(e.target.value)
    adjustHeight()
  }

  const handleFocus = () => {
    adjustHeight()
  }

  const handleBlur = () => {
    const textarea = textareaRef.current
    if (textarea && !value.trim()) {
      textarea.style.height = '80px' //collapse empty textareas
    }
  }

  return (
    <div className="category-section">
      <label className="category-label">
        <span className="category-icon">{icon}</span>
        {label}
      </label>
      <textarea
        ref={textareaRef}
        className="category-textarea"
        value={value}
        onChange={handleChange}
        onFocus={handleFocus}
        onBlur={handleBlur}
        placeholder={placeholder}
      />
    </div>
  )
}
```

**UX decision:** Auto-expanding textareas improve the writing experience by showing all content while typing, then collapsing when empty to maintain a clean interface.

### 4. Daily Quote with Deterministic Selection

```
//HomePage.jsx
const dailyQuote = useMemo(() => {
  const today = new Date()
  const dateString = `${today.getFullYear()}-${today.getMonth()}-${today.getDate()}`
  
  //hash function for deterministic selection
  let hash = 0
  for (let i = 0; i < dateString.length; i++) {
    hash = ((hash << 5) - hash) + dateString.charCodeAt(i)
    hash = hash & hash
  }
  
  const index = Math.abs(hash) % quotes.length
  return quotes[index]
}, [])
```

**Why this approach?** Using `useMemo` with a date-based hash ensures the quote only changes once per day (at midnight) without requiring server-side logic or localStorage. The hash function guarantees the same quote for the same date.

### 5. Collapsible Entry Cards

```
//EntryCard.jsx
function EntryCard({ entry, defaultExpanded = false }) {
  const [isExpanded, setIsExpanded] = useState(defaultExpanded)

  const getPreview = () => {
    if (entry.learning) return entry.learning
    if (entry.exercise) return entry.exercise
    if (entry.meditation) return entry.meditation
    return 'No entry'
  }

  return (
    <div className="entry-card">
      <div 
        className="entry-header" 
        onClick={() => setIsExpanded(!isExpanded)}
      >
        <div>
          <span className="entry-date">{entry.date}</span>
          {!isExpanded && (
            <span className="entry-preview"> — {getPreview()}</span>
          )}
        </div>
        <span className={`entry-toggle ${isExpanded ? 'expanded' : ''}`}>
          ▼
        </span>
      </div>
      
      <div className={`entry-content ${isExpanded ? 'expanded' : ''}`}>
        {entry.learning && (
          <div className="entry-item">
            <span className="entry-item-label">Mind:</span>
            <span className="entry-item-text">{entry.learning}</span>
          </div>
        )}
        {entry.exercise && (
          <div className="entry-item">
            <span className="entry-item-label">Body:</span>
            <span className="entry-item-text">{entry.exercise}</span>
          </div>
        )}
        {entry.meditation && (
          <div className="entry-item">
            <span className="entry-item-label">Mindfulness:</span>
            <span className="entry-item-text">{entry.meditation}</span>
          </div>
        )}
      </div>
    </div>
  )
}
```

**Design rationale:** Collapsible cards reduce visual clutter while allowing quick previews. Today's entry defaults to expanded for immediate visibility.

### 6. Google OAuth Authentication

```
//LoginPage.jsx (Google Sign in)
const handleGoogleSignIn = async () => {
  try {
    const provider = new GoogleAuthProvider()
    await signInWithPopup(auth, provider)
  } catch (err) {
    setError(err.message)
  }
}

return (
  <button className="btn-google" onClick={handleGoogleSignIn}>
    <svg width="18" height="18" viewBox="0 0 24 24">
      {/* Google logo SVG */}
    </svg>
    Continue with Google
  </button>
)
```

**Implementation note:** Google OAuth provides a seamless authentication experience without requiring users to create and remember passwords.

## Design System

**Design philosophy:** The color palette creates a calming, nature-inspired environment that supports reflection and mindfulness. Dark backgrounds reduce eye strain for daily use, while warm accent colors provide visual interest without distraction.

### Responsive Design

- **Mobile-first approach** - Base styles optimized for mobile
- **Breakpoints** - `@media (max-width: 600px)` for tablet/mobile adjustments
- **Touch-friendly** - Larger tap targets and optimized spacing
- **Progressive enhancement** - Core functionality works on all devices

## Project Structure

```
kaizen-journal/
├── public/
│   ├── icon-192.png          #PWA icons
│   ├── icon-512.png
│   └── manifest.json         #PWA manifest
├── src/
│   ├── pages/
│   │   ├── HomePage.jsx      #main journal interface
│   │   ├── CalendarPage.jsx  #calendar view
│   │   ├── LoginPage.jsx     #authentication
│   │   ├── SettingsPage.jsx  #user settings & share link
│   │   └── MotivatePage.jsx  #public motivation page
│   ├── components/
│   │   ├── CategoryInput.jsx #auto-expanding textarea
│   │   └── EntryCard.jsx     #collapsible entry display
│   ├── firebase/
│   │   └── config.js         #Firebase initialization
│   ├── data/
│   │   └── quotes.js         #motivational quotes array
│   ├── App.jsx               #root component & routing
│   ├── App.css               #global app styles
│   └── index.css             #CSS variables & base styles
├── vercel.json               #Vercel routing config
└── package.json              #dependencies
```

## Security & Best Practices

- **Environment Variables** - All Firebase credentials stored in `.env` (not committed)
- **Firebase Security Rules** - Database protected with user-based access control
- **Protected Routes** - Authentication required for journal features
- **Error Handling** - Try-catch blocks and user-friendly error messages
- **Memory Management** - Proper cleanup of subscriptions and event listeners
- **Input Validation** - Client-side validation before database writes

## Deployment

The application is deployed on **Vercel** with:
- Automatic deployments from GitHub
- Environment variable management
- Client-side routing support via `vercel.json`
- PWA manifest configuration for installability

## Key Metrics

- **Components:** 8+ reusable React components
- **Pages:** 5 main application pages
- **Firebase Collections:** 3 (entries, motivations, userPreferences)
- **Real-time Features:** Live data synchronization
- **PWA Features:** Installable with offline capabilities

## Future Enhancements

Potential improvements for future iterations:
- Data visualization (charts for progress tracking)
- Export functionality (PDF/CSV)
- Reminder notifications
- Social features (friend connections, shared goals)
- Dark/light theme toggle
- Multi-language support

---

*Built with React, Firebase, and modern web technologies*

**Note:** This is a portfolio demonstration. The live application showcases full functionality. Code snippets above represent key architectural decisions and implementation patterns.
