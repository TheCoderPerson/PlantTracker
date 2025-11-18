# PlantTracker Technical Design Document

## Table of Contents

1. [Overview](#overview)
2. [Technology Stack](#technology-stack)
3. [Project Structure](#project-structure)
4. [Architecture](#architecture)
5. [Data Model](#data-model)
6. [Component Structure](#component-structure)
7. [State Management](#state-management)
8. [Key Features](#key-features)
9. [Service Worker & PWA](#service-worker--pwa)
10. [Application Flow](#application-flow)
11. [Contributing Guidelines](#contributing-guidelines)
12. [Development Setup](#development-setup)

---

## Overview

PlantTracker is a Progressive Web Application (PWA) designed to help users track their weekly consumption of different plant varieties. The application encourages users to consume at least 30 different plants per week for optimal gut health and nutrition diversity.

### Key Goals
- Track weekly plant consumption with a goal of 30 different plants
- Visualize progress through heatmaps and statistics
- Work offline without requiring a backend server
- Provide an installable mobile-friendly experience

### Design Philosophy
- **Single-file architecture**: All application code in one HTML file for simplicity
- **Client-side only**: No backend dependencies
- **Offline-first**: Full functionality without internet connection
- **Mobile-first**: Responsive design optimized for touch interfaces

---

## Technology Stack

### Frontend Libraries (via CDN)

| Library | Version | Purpose |
|---------|---------|---------|
| React | 18.x | UI component library |
| ReactDOM | 18.x | React DOM rendering |
| Babel Standalone | Latest | In-browser JSX transpilation |
| TailwindCSS | Latest | Utility-first CSS framework |

### Browser APIs

- **localStorage**: Data persistence
- **Service Worker API**: Offline caching and PWA capabilities
- **FileReader API**: CSV file import functionality
- **Web App Manifest**: PWA installation support

### No Backend Required

This application runs entirely in the browser with no server-side code, database server, or API endpoints needed.

---

## Project Structure

```
PlantTracker/
├── plant-tracker.html      # Main application (all code)
├── service-worker.js       # PWA service worker
├── plants-list.csv         # Default plant database (118 plants)
├── TECHNICAL_DESIGN.md     # This document
└── .git/                   # Git repository
```

### File Descriptions

#### `plant-tracker.html` (Main Application)
The single-file application containing:
- HTML structure and root element
- Embedded CSS styles
- React component code (JSX)
- Data constants (DEFAULT_PLANTS, CATEGORIES)
- Embedded base64 web manifest

#### `service-worker.js` (PWA Service Worker)
Handles:
- Asset caching for offline use
- Cache versioning and updates
- Network-first/cache-fallback strategy

#### `plants-list.csv` (Plant Database)
- Contains 118 default plants organized by category
- Format: `Category,Plant Name`
- Used as reference data and for user imports

---

## Architecture

### Single-Page Application (SPA)

The application uses React to manage a single-page interface with multiple views:

```
┌─────────────────────────────────────┐
│           PlantTracker              │
│         (Main Component)            │
├─────────────────────────────────────┤
│  ┌─────┐ ┌──────┐ ┌───────┐ ┌────┐ │
│  │Home │ │Select│ │History│ │Import│
│  │View │ │View  │ │View   │ │View │
│  └─────┘ └──────┘ └───────┘ └────┘ │
├─────────────────────────────────────┤
│        Navigation Bar               │
└─────────────────────────────────────┘
```

### Data Flow

```
User Action → State Update → localStorage Save → UI Re-render
     ↓
Service Worker → Cache Update (if applicable)
```

---

## Data Model

### localStorage Schema

The application uses two keys in localStorage for all data persistence:

#### 1. Plants List (`plantsList`)

```javascript
[
  {
    "name": "Arugula",
    "category": "Vegetables"
  },
  {
    "name": "Almonds",
    "category": "Nuts & Seeds"
  }
  // ... more plants
]
```

#### 2. Weekly Tracking Data (`weeklyData`)

```javascript
{
  "2025-W46": ["Arugula", "Almonds", "Apples"],
  "2025-W45": ["Spinach", "Kale", "Quinoa"],
  // ... week ID (YYYY-Wnn) → array of plant names
}
```

### Plant Categories

The application supports 7 categories:

| Category | Default Plants | Description |
|----------|---------------|-------------|
| Vegetables | 37 | Common vegetables |
| Fruits | 28 | Fresh and dried fruits |
| Legumes | 9 | Beans, lentils, peas |
| Grains | 12 | Whole grains and cereals |
| Nuts & Seeds | 15 | Tree nuts and seeds |
| Herbs & Spices | 18 | Culinary herbs and spices |
| Uncategorized | Variable | User-added plants without category |

### Week ID Format

Uses ISO 8601 week numbering:
- Format: `YYYY-Wnn` (e.g., `2025-W46`)
- Week starts on Monday
- Based on UTC time

---

## Component Structure

### Main Component: `PlantTracker`

The entire application is built as a single React functional component using hooks.

### Views

#### 1. Home View (`view === 'home'`)
- Weekly progress display with circular indicator
- Current plant count (X/30)
- Progress bar visualization
- All-time statistics panel
- Quick navigation to plant selection
- Preview of current week's selections

#### 2. Plant Selection View (`view === 'select'`)
- Search input for filtering plants
- Category dropdown filter
- Tab filters: All / Eaten / Not Yet
- Grid of plant cards with toggle selection
- Visual checkmark for selected plants

#### 3. History View (`view === 'history'`)
- Current streak counter (consecutive 30+ weeks)
- Year navigation selector
- 52-week heatmap grid (13 columns × 4 rows)
- Color-coded performance visualization
- Detailed week-by-week list with progress bars

#### 4. Import View (`view === 'import'`)
- Manual plant entry form (name + category)
- CSV file upload
- Current plant count display
- Format instructions
- App update checker
- Reset to defaults option

### Navigation

Fixed bottom navigation bar with four tabs:
- 🏠 Home
- 🌱 Select
- 📊 History
- ⚙️ Import

---

## State Management

### State Variables

```javascript
// Core Data
const [plants, setPlants] = useState([]);           // All available plants
const [weeklyData, setWeeklyData] = useState({});   // Weekly tracking data
const [currentWeek, setCurrentWeek] = useState(''); // Current week ID

// UI State
const [view, setView] = useState('home');           // Current view
const [searchTerm, setSearchTerm] = useState('');   // Search filter text
const [category, setCategory] = useState('all');    // eaten/uneaten filter
const [categoryFilter, setCategoryFilter] = useState('all'); // Category filter
const [selectedYear, setSelectedYear] = useState(new Date().getFullYear());

// PWA State
const [updateAvailable, setUpdateAvailable] = useState(false);
const [waitingWorker, setWaitingWorker] = useState(null);
const [checkingUpdate, setCheckingUpdate] = useState(false);

// Form State
const [newPlantName, setNewPlantName] = useState('');
const [newPlantCategory, setNewPlantCategory] = useState('');
```

### Computed Values (useMemo)

```javascript
// Filtered plants based on search and filters
const filteredPlants = useMemo(() => { ... }, [plants, searchTerm, category, categoryFilter, currentWeekPlants]);

// All-time statistics
const stats = useMemo(() => { ... }, [weeklyData, plants]);

// Current week's selected plants
const currentWeekPlants = useMemo(() => { ... }, [weeklyData, currentWeek]);
```

### Data Persistence

All state changes trigger automatic saves to localStorage:

```javascript
useEffect(() => {
  localStorage.setItem('plantsList', JSON.stringify(plants));
}, [plants]);

useEffect(() => {
  localStorage.setItem('weeklyData', JSON.stringify(weeklyData));
}, [weeklyData]);
```

---

## Key Features

### 1. Weekly Plant Tracking

**Goal**: 30 different plants per week

**How it works**:
1. User navigates to Select view
2. Searches or browses available plants
3. Taps plant cards to toggle selection
4. Selections are saved to current week
5. Progress updates in real-time

### 2. Progress Visualization

**Progress Bar**: Shows percentage toward 30-plant goal

**Color Coding**:
- Gray (`#e5e7eb`): No data
- Red (`#f87171`): 1-9 plants
- Orange (`#fb923c`): 10-19 plants
- Yellow (`#fbbf24`): 20-29 plants
- Green (`#10b981`): 30+ plants (goal achieved)

### 3. Statistics Tracking

- **Weeks Achieved**: Count of weeks with 30+ plants
- **Total Weeks**: All weeks with any data
- **Unique Plants**: Total different plants ever consumed
- **Success Rate**: Percentage of weeks hitting goal
- **Current Streak**: Consecutive weeks with 30+ plants

### 4. Heatmap History

- 52-week grid organized as 13×4
- Color-coded cells by performance
- Year navigation
- Detailed week breakdown below

### 5. Plant Management

**Add Manually**:
- Enter plant name
- Select category from dropdown
- Validates for duplicates and empty names

**CSV Import**:
- Supports two formats:
  - `Category,Plant Name`
  - `Plant Name` (single column)
- Skips header rows
- Trims whitespace
- Prevents duplicates

**Reset to Defaults**:
- Restores original 118 plants
- Clears all custom additions

### 6. Category Filtering

- Filter by any of the 7 categories
- Combined with search and eaten/uneaten filters
- Enables quick access to specific food groups

---

## Service Worker & PWA

### Configuration

```javascript
const CACHE_NAME = 'plant-tracker-v1.0.0';
```

### Cached Resources

- `/` (root path)
- `plant-tracker.html`
- React production build (CDN)
- ReactDOM production build (CDN)
- Babel standalone (CDN)
- TailwindCSS (CDN)

### Caching Strategy

**Install**: Pre-cache all essential assets
**Fetch**: Cache-first with network fallback
**Activate**: Clean up old cache versions

### Update Mechanism

1. Service worker detects new version
2. New worker installs but waits
3. Application shows update banner
4. User clicks "Update Now"
5. Message sent to skip waiting
6. Page reloads with new version

### Web App Manifest

Embedded as base64 in the HTML file:
- App name: "PlantTracker"
- Theme color: Green (`#10b981`)
- Display: Standalone
- Icons: Emoji-based (🌱)

---

## Application Flow

### Initialization

```
1. Browser loads plant-tracker.html
2. CDN libraries load (React, Babel, Tailwind)
3. Babel transpiles JSX
4. Service worker registers
5. React mounts PlantTracker component
6. useEffect loads data from localStorage
7. Data migration runs if needed
8. Current week calculated
9. UI renders with loaded data
```

### Data Migration

Automatically upgrades legacy data formats:
- Old string format → New object format with categories
- Assigns categories based on DEFAULT_PLANTS lookup
- Defaults to "Uncategorized" if not found

### User Workflows

**Track Plants for Current Week**:
```
Home → "Add Plants" → Select View → Search/Filter
→ Tap Plants → Back → See Updated Progress
```

**View Historical Performance**:
```
Home → History Tab → View Heatmap
→ Change Year → View Details → Identify Patterns
```

**Add Custom Plants**:
```
Home → Import Tab → Enter Name & Category
→ Submit → Plant Added to Database
```

**Import from CSV**:
```
Home → Import Tab → Choose File
→ Select CSV → Plants Imported
```

---

## Contributing Guidelines

### Code Style

- Use functional React components with hooks
- Follow existing naming conventions
- Use Tailwind CSS for styling
- Keep all code in single HTML file
- Use descriptive variable names

### Adding New Features

1. **Plan the feature**: Identify which view(s) it affects
2. **Update state**: Add necessary state variables
3. **Implement UI**: Add components using Tailwind
4. **Handle persistence**: Save to localStorage if needed
5. **Test thoroughly**: Check all edge cases

### Modifying Plant Data

To update default plants:
1. Edit `DEFAULT_PLANTS` constant in HTML file
2. Update `plants-list.csv` to match
3. Ensure categories are consistent

### Testing Checklist

- [ ] Works offline (disable network in DevTools)
- [ ] Data persists across page reloads
- [ ] Week calculations are correct
- [ ] All filters work correctly
- [ ] CSV import handles edge cases
- [ ] PWA update mechanism works
- [ ] Mobile responsive design

### Common Modifications

**Adding a new category**:
1. Add to `CATEGORIES` array
2. Add default plants to `DEFAULT_PLANTS`
3. Update `plants-list.csv`

**Adding a new view**:
1. Add view name to navigation
2. Create conditional render in component
3. Implement view content
4. Add state variables as needed

**Modifying statistics**:
1. Update `stats` useMemo calculation
2. Update UI to display new stats
3. Consider performance impact

---

## Development Setup

### Prerequisites

- Modern web browser (Chrome, Firefox, Safari, Edge)
- Local web server (optional, for PWA features)
- Text editor or IDE

### Running Locally

**Option 1: Direct file access**
```bash
# Simply open the file in a browser
open plant-tracker.html
# or
xdg-open plant-tracker.html  # Linux
```

**Option 2: Local server (recommended for PWA)**
```bash
# Using Python
python -m http.server 8000

# Using Node.js
npx serve .

# Then open http://localhost:8000/plant-tracker.html
```

### Development Tips

1. **Use browser DevTools**: React DevTools extension is helpful
2. **Test localStorage**: Use Application tab to inspect data
3. **Test Service Worker**: Use Application > Service Workers
4. **Test offline**: Use Network tab to simulate offline
5. **Test mobile**: Use device emulation in DevTools

### Debugging

**Clear application data**:
```javascript
localStorage.clear();
// Then reload page
```

**Reset service worker**:
1. Open DevTools > Application > Service Workers
2. Click "Unregister"
3. Reload page

**View current data**:
```javascript
console.log(JSON.parse(localStorage.getItem('plantsList')));
console.log(JSON.parse(localStorage.getItem('weeklyData')));
```

---

## Performance Considerations

### Optimizations Used

1. **useMemo**: Expensive calculations cached
2. **Minimal re-renders**: State updates are batched
3. **Efficient storage**: Only save on changes
4. **CDN libraries**: Fast, cached externally

### Potential Improvements

1. **Code splitting**: Separate large data constants
2. **Virtual scrolling**: For large plant lists
3. **IndexedDB**: For larger datasets
4. **Web Workers**: For heavy calculations

---

## Security Considerations

- No sensitive data transmitted (all local)
- No authentication required
- LocalStorage is per-origin isolated
- CSP headers recommended for production

---

## Browser Support

### Fully Supported
- Chrome 80+
- Firefox 75+
- Safari 13+
- Edge 80+

### Required Features
- ES6+ JavaScript
- Service Workers
- LocalStorage
- CSS Grid/Flexbox

---

## Future Enhancements

Potential features for future development:

1. **Data export**: Full backup to JSON
2. **Data sync**: Optional cloud sync
3. **Sharing**: Share week with friends
4. **Recipes**: Suggest meals using selected plants
5. **Reminders**: Notification for weekly check-in
6. **Themes**: Dark mode support
7. **Accessibility**: ARIA labels, keyboard navigation
8. **Localization**: Multi-language support

---

## License

This project is open source. See repository for license details.

---

## Contact & Support

For issues, questions, or contributions, please use the GitHub repository's issue tracker.
