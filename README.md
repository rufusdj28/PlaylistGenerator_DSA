# GenMix — DSA Music Playlist Manager

> A comprehensive music playlist management system demonstrating **Data Structures and Algorithms (DSA)** through a fully functional web prototype.

---

## 📋 Project Overview

GenMix is a working web application that allows users to:

| Feature | Status |
|---------|--------|
| Search and browse songs from a CSV-backed catalog | ✅ |
| Save named playlist copies in browser storage | ✅ |
| View saved playlists on Home and in Library | ✅ |
| Navigate a simulated queue in a circular manner | ✅ |
| Get random, distinct recommendations from the catalog | ✅ |
| Visualize DSA operations in real-time | ✅ |
| Generate seed-based or mood-based playlists | ✅ |

---

## 🎯 DSA Algorithms Implemented

### 1. B-Tree 🌳

**Purpose:** Balanced song catalog management indexed by `track_id`.

**Operations:**

| Method | Description |
|--------|-------------|
| `insert(trackId, song)` | Add a unique song to the B-Tree |
| `search(trackId)` | Find a song by its trimmed track ID |
| `inorderTraversal()` | Return songs in ascending track ID order |
| `height()` | Return the number of B-Tree levels |
| `searchByTitle(query)` | Title search using a traversal scan |

**Use Case:**
- Songs are stored by `track_id`; the B-Tree remains balanced as the catalog grows
- Partial-title search is retained as a compatibility scan
- The UI sorts displayed library and search results by title
- Catalog is loaded from `data/songs_deduped.csv` (4,428 unique songs; loader caps at 5,000 rows)

**Complexity:**

| Operation | Time Complexity | Space |
|-----------|----------------|-------|
| Insert / Track ID lookup | O(log n) | O(1) auxiliary |
| Title substring search / Full traversal | O(n) | O(n) |

---

### 2. Binary Search by Track ID 🔎

**Purpose:** Compare searching a sorted song array against a B-Tree lookup.

**Implementation:**
- `binarySearchIndexByTrackId()` and `binarySearchByTrackId()` live in `src/core/dsa.js`
- `SortedSongIndex` builds a verified snapshot from `BTree.inorderTraversal()`
- `compareRetrievalMethods(trackId)` in `src/app.js` returns both results for a console demo; the UI search remains B-Tree-based

**How It Works:**
- Binary search checks the middle track ID and discards the half that cannot contain the target
- The snapshot uses the B-Tree's in-order traversal, so it starts sorted with the same unique songs
- `SortedSongIndex.rebuild()` refreshes the snapshot after catalog changes; a search does not re-sort

**Complexity:**

| Operation | Data Structure | Time | Extra Space |
|-----------|----------------|------|-------------|
| Search track ID | Sorted array (binary search) | O(log n) | O(1) |
| Build from B-Tree traversal | Sorted array | O(n) | O(n) |
| Build from unsorted raw songs | Sorted array | O(n log n) | O(n) |
| Insert into sorted array | Sorted array | O(n) — shifting | O(1) additional |
| Search track ID | B-Tree | O(t · log_t n) | O(1) |
| Insert song | B-Tree | O(log n) for fixed t | O(1) auxiliary |

**Example:**

```javascript
const tree = new DSA.BTree();
tree.insert('track-b', { track_id: 'track-b', title: 'Second Song' });
tree.insert('track-a', { track_id: 'track-a', title: 'First Song' });

const songIndex = new DSA.SortedSongIndex(tree);
const song = songIndex.search('track-a');
console.log(song ? song.title : 'Not found');
```

Run B-Tree and binary search tests:

```bash
node tests/btree.test.js
```

> **Note:** No timing benchmark is included by default. Any optional timing comparison is approximate — results depend on the machine, runtime, and repetition count.

---

### 3. Song Similarity Graph 🎧

**Purpose:** Find songs with similar audio features using a directed, weighted adjacency-list graph.

**How It Works:**
- After the CSV catalog loads, `src/app.js` builds a `SongGraph` from the B-Tree's unique in-order traversal
- Songs with finite numeric `danceability`, `energy`, `acousticness`, `valence`, and `tempo` become graph nodes; others are recorded in `skippedSongs`
- Each feature is min-max normalized; pairwise Euclidean distance across five features becomes a similarity weight: `1 − distance / √5`
- Each node stores up to **10 outgoing edges** by default, ordered by descending similarity then ascending `track_id`
- `bfs(trackId, { limit })` and `dfs(trackId, { limit })` traverse reachable outgoing edges and return track IDs

**Example:**

```javascript
const graph = new DSA.SongGraph().buildFromSongs(songCatalog.inorderTraversal());
const neighbors = graph.getNeighbors('some-track-id');
console.log(neighbors);
// [{ to: 'another-track-id', weight: 0.98 }, ...]
console.log(graph.bfs('some-track-id', { limit: 20 }));
```

**Complexity:**

| Operation | Time | Space |
|-----------|------|-------|
| Build graph (pairwise comparison) | O(n² × 5) | O(nk) — k edges per node |
| BFS / DFS traversal | O(V + E) | O(V) auxiliary |

> **Limitation:** The O(n²) build cost is suitable for the 5,000-row CSV but not for the full ~80,000-row dataset.

Run graph tests:

```bash
node tests/graph.test.js
```

---

### 4. Smart Playlist Generator 🎼

**Purpose:** Generate a 20-song playlist from a seed track or an audio-feature mood profile.

**Implementation:** `PlaylistGenerator` in `src/playlist/playlistGenerator.js` receives the B-Tree catalog and SongGraph; it does not load CSV data itself.

**Seed Mode — `generateFromSeed(trackId, options)`:**
- Places the valid seed first
- Collects direct graph neighbors, then expands through neighbors-of-neighbors up to `expansionDepth` (default: 2)
- Re-ranks candidates by similarity to the seed using `DSA.computeSimilarity`, then by `track_id`
- Selects up to 19 similar songs and shuffles them with Fisher-Yates

**Mood Mode — `generateFromMood(moodName, options)`:**
- Filters graph songs by all rules in a mood profile
- Ranks matches by average normalized margin beyond rule thresholds, then by `track_id`
- Keeps the best 20 matches and shuffles them

**Return Format:** Both modes return `{ type, seed, mood, songs, requested, message }`.

**Mood Thresholds** *(empirical P25/P75 from the first 5,000 CSV songs after normalization)*:

| Mood | Filter Rules | Matches (first 5,000 songs) |
|------|--------------|--------------------------:|
| Happy / Upbeat | valence ≥ 0.677638 and danceability ≥ 0.681725 | 643 |
| Energetic | energy ≥ 0.822211 and tempo ≥ 0.635812 | 413 |
| Chill | energy ≤ 0.312949 and acousticness ≥ 0.791164 | 1,012 |
| Melancholic | valence ≤ 0.217085 and energy ≤ 0.312949 | 752 |
| Acoustic | acousticness ≥ 0.791164 | 1,255 |

**UI Wizard Flow:**
1. Name the playlist
2. Choose **Pick a Song** or **Pick a Mood**
3. Select a seed or mood
4. Review the generated playlist
5. **Save to Library** to keep it, or **Delete Playlist** to discard

> Results are temporary and do not appear on Home or in Library until **Save to Library** is chosen.

Run playlist tests:

```bash
node tests/playlistGenerator.test.js
node tests/playlistUI.test.js
node tests/playlistWizard.test.js
node tests/app.test.js
```

---

### 5. Circular Queue 🔄

**Purpose:** Playlist playback management with seamless circular navigation.

**Operations:**

| Method | Description |
|--------|-------------|
| `enqueue()` | Add a song to the queue |
| `dequeue()` | Remove a song from the queue |
| `next()` | Advance to the next song (wraps around) |
| `previous()` | Go back to the previous song (wraps around) |
| `getCurrent()` | Get the currently active song |
| `isEmpty()` | Check whether the queue is empty |
| `clear()` | Empty the entire queue |

**Complexity:**

| Operation | Time | Space |
|-----------|------|-------|
| All operations | O(1) | O(n) |

**Special Feature:** Automatically handles boundary wrapping; tracks current position independently of front/rear pointers.

---

### 6. Priority Queue — Max Heap ⭐

**Purpose:** Priority-based song retrieval ranked by rating (1–10 scale).

**Operations:**

| Method | Description |
|--------|-------------|
| `insert()` | Add a song with a priority rating |
| `extractMax()` | Remove and return the highest-rated song |
| `getTopRecommendations(n)` | Return a copy of the top N songs |
| `heapifyUp()` | Restore heap property after insertion |
| `heapifyDown()` | Restore heap property after extraction |

**Complexity:**

| Operation | Time | Space |
|-----------|------|-------|
| Insert | O(log n) | O(n) |
| Extract max | O(log n) | O(n) |
| Get top N | O(k log n) | O(n) |

---

### 7. Heap Sort 🔀

**Purpose:** Sort songs by rating using an in-place heap-based algorithm.

- `heapSort()` — Builds a max heap and extracts elements in descending rating order

**Complexity:**

| Time | Space |
|------|-------|
| O(n log n) | O(1) — in-place |

---

## 🚀 Getting Started

### Prerequisites

- **Node.js** v14 or higher
- **npm** or **yarn**

### Installation

1. **Navigate to the project directory**

   ```bash
   cd PlaylistGenerator_DSA
   ```

2. **Install dependencies**

   ```bash
   npm install
   ```

3. **Start the development server**

   ```bash
   npm run dev
   ```

4. **Open in browser**

   The app will automatically open at `http://localhost:3010`.

### Running Tests

Run the full test suite:

```bash
npm test
```

Or run individual suites:

```bash
node tests/btree.test.js
node tests/graph.test.js
node tests/playlistGenerator.test.js
node tests/app.test.js
```

---

## 📖 Usage Guide

### Adding Songs to the Playlist

1. Search the catalog by title using the search box
2. Browse results two rows at a time with the Previous / Next pagination
3. Click a result to add it to the current queue

### Saving and Viewing Playlists

1. Save generated playlists to Library from the generator results
2. Add recommendations from Home, or catalog songs from a playlist's Library detail page
3. Saved playlists are persisted under `genmixSavedPlaylists` in the browser's `localStorage`

### Generating Playlists

1. Open **Generate** from the navigation menu
2. Click **Create Playlist** and enter a name
3. Choose **Pick a Song** (seed-based) or **Pick a Mood** (mood-based)
4. Review the results, then **Save to Library** or **Delete Playlist**

### Searching Songs

1. Type in the search box
2. Matching titles from the B-Tree catalog appear instantly
3. Click any result to add it to the queue

### Using the Simulated Queue

1. Use the play/pause control to start or pause the simulation
2. Use **Next** or **Previous** to navigate circularly — no real audio is played
3. The queue is temporary; save playlists to Library to persist them

### Recommendations

- Home displays a carousel of up to **100 distinct songs** (4 columns × 2 rows on desktop)
- Click a song to pause the carousel and add it to a saved playlist
- Refresh Home for a fresh random sample

### Exporting to YouTube *(Optional)*

1. Complete the one-time setup in [`docs/YOUTUBE_EXPORT_SETUP.md`](docs/YOUTUBE_EXPORT_SETUP.md)
2. Use **Export to YouTube** on any saved-playlist card
3. Confirm the name, privacy setting, and estimated API quota
4. Authorize your Google account — OAuth tokens stay in memory only
5. GenMix preserves song order and links to the created YouTube playlist

---

## 📊 Algorithm Visualization

The application includes **real-time visualization** of DSA operations, accessible in the **Algorithm Visualization** panel:

- B-Tree insert and search events
- Circular queue navigation steps
- Heap operations during shuffle
- Graph traversal progress

---

## 🏗️ Project Structure

```
GenMix/
├── index.html                    # Main HTML entry point
├── css/
│   └── styles.css                # UI styling
├── src/
│   ├── app.js                    # Application logic and UI interactions
│   ├── core/
│   │   └── dsa.js                # All DSA implementations
│   ├── playlist/
│   │   ├── playlistGenerator.js  # Seed- and mood-based playlist logic
│   │   ├── playlistWizard.js     # Wizard state and saved-record helpers
│   │   ├── playlistEditor.js     # Saved-playlist edit helpers
│   │   └── playlistUI.js         # UI search and display helpers
│   └── integrations/
│       └── youtube/              # YouTube auth, API client, matcher, exporter
├── data/
│   ├── songs_deduped.csv         # Primary catalog — 4,428 unique songs
│   └── songs_cleaned.csv         # Original full dataset (reference only)
├── tests/
│   └── youtube/                  # YouTube integration tests
├── docs/                         # Setup guides and project report
├── scripts/
│   ├── generate-config.js        # Builds config.local.js from .env
│   └── convert_to_word.py        # Converts project report to Word format
├── config.local.js               # Generated, git-ignored browser config
├── package.json                  # Project metadata and npm scripts
└── README.md                     # This file
```

---

## 🔬 Technical Details

### Data Structures Comparison

| Structure | Primary Use Case | Insert | Search | Extract Max | Space |
|-----------|-----------------|--------|--------|-------------|-------|
| **B-Tree** | Song catalog | O(log n) | O(log n) by track ID | — | O(n) |
| **Circular Queue** | Playlist playback | O(1) | — | O(1) | O(n) |
| **Max Heap** | Song recommendations | O(log n) | — | O(log n) | O(n) |

### Catalog Data

- Loaded from `data/songs_deduped.csv` — 4,428 unique songs
- Loader caps input at 5,000 data rows
- If the CSV fails to load or yields no usable songs, the library displays an empty-state message; no demo songs are inserted

---

## 💡 Key Learning Outcomes

1. **Real-world DSA Applications** — How data structures solve practical problems and the trade-offs involved
2. **Time and Space Complexity** — Big-O notation applied to live operations
3. **Data Structure Integration** — Multiple structures working together within a single application
4. **Algorithm Visualization** — Visual feedback for understanding and debugging DSA behavior

---

## 📝 Future Enhancements

- [ ] B-Tree deletion as a standalone extension
- [ ] Hash table for O(1) artist lookup
- [ ] Trie for autocomplete search
- [ ] Merge sort comparison view
- [ ] Dijkstra's algorithm for graph-based playlist generation
- [ ] Connect playlist generator to additional UI controls

---

## 🎓 Educational Value

This project is a practical resource for:

- **Students** studying data structures and algorithms
- **Developers** exploring algorithm design patterns
- **Educators** demonstrating DSA concepts visually
- **Interview candidates** preparing for DSA-heavy interviews

---

## 📄 License

MIT License — Free to use for learning and educational purposes.

---

## 👨‍💻 Author

Created as a semester DSA project demonstrating practical implementations of fundamental data structures and algorithms.

---

## 🙏 Acknowledgments

- Algorithms based on standard computer science textbooks
- UI inspired by modern music streaming applications
- Educational resources from various DSA courses

---

*"The best way to learn algorithms is to implement them in a real project."*

## 📋 Project Overview

This project is a **working web application** that allows users to:
- Search and browse songs in a CSV-backed catalog
- ✅ Save named copies of the current queue in browser storage
- ✅ View saved playlists on Home and in Library
- ✅ Navigate a simulated playlist queue in a circular manner
- ✅ Get random, distinct recommendations from the loaded catalog
- ✅ Visualize DSA operations in real-time
- Generate a seed-based or mood-based playlist

## 🎯 DSA Algorithms Implemented

### 1. **B-Tree** 🌳
**Purpose:** Balanced song catalog management indexed by `track_id`

**Operations Implemented:**
- `insert(trackId, song)` - Add a unique song to the B-Tree
- `search(trackId)` - Find a song by its trimmed track ID
- `inorderTraversal()` - Return songs in ascending track ID order
- `height()` - Return the number of B-Tree levels
- `searchByTitle(query)` - Keep the existing title search using a simple traversal scan

**Use Case:** 
- Catalog songs are stored by `track_id`; the B-Tree remains balanced as it grows
- The existing partial-title search is retained as a compatibility scan
- Tree traversal is by track ID; the UI sorts displayed library and search results by title
- The catalog is loaded from `data/songs_deduped.csv` (4,428 unique songs; the loader caps input at 5,000 data rows)

**Complexity:**
- Insert and track ID lookup: O(log n) for fixed minimum degree
- Title substring search and full traversal: O(n)
- Space: O(n)

---

### Binary Search by Track ID 🔎
**Purpose:** Compare searching a sorted song array with looking up the same song in the B-Tree.

**Where It Lives:**
- `binarySearchIndexByTrackId()` and `binarySearchByTrackId()` are implemented in `src/core/dsa.js`
- `SortedSongIndex` builds one verified snapshot from `BTree.inorderTraversal()`
- `compareRetrievalMethods(trackId)` in `src/app.js` returns both results for a console demo; the UI search remains B-Tree-based

**How It Works:**
- Binary search checks the middle track ID and discards the half that cannot contain the requested ID
- The array must be sorted by `track_id` first; the index uses the B-Tree's in-order traversal, so it starts sorted and contains the same unique songs
- The B-Tree stays balanced while songs are inserted; inserting into a sorted array can require shifting many entries
- `SortedSongIndex.rebuild()` refreshes the snapshot after catalog changes; a search does not sort the array again

**Complexity:**

| Operation | Data Structure | Time | Extra Space |
|-----------|----------------|------|-------------|
| Search track ID | Sorted array (iterative binary search) | O(log n) | O(1) |
| Build array from B-Tree traversal | Sorted array | O(n) | O(n) for the snapshot |
| Build array from unsorted raw songs | Sorted array | O(n log n) | O(n) for the copy |
| Insert into a sorted array | Sorted array | O(n) due to shifting | O(1) additional |
| Search track ID | B-Tree | O(t * log_t n) | O(1) |
| Insert song | B-Tree | O(t * log_t n), O(log n) for fixed t | O(1) auxiliary |

**Example:**
```javascript
const tree = new DSA.BTree();
tree.insert('track-b', { track_id: 'track-b', title: 'Second Song' });
tree.insert('track-a', { track_id: 'track-a', title: 'First Song' });

const songIndex = new DSA.SortedSongIndex(tree);
const song = songIndex.search('track-a');
console.log(song ? song.title : 'Not found');
```

Run the B-Tree and binary search tests with:
```bash
node tests/btree.test.js
```

No timing benchmark is included by default. Any optional timing comparison should be treated as approximate: results depend on the machine, runtime, and repetition count.

---

### Song Similarity Graph 🎧
**Purpose:** Find songs with similar audio features using a directed, weighted adjacency-list graph.

**How It Works:**
- After the CSV catalog loads, `src/app.js` builds a `SongGraph` from the B-Tree's unique in-order song traversal. The existing loader, retrieval, playlist, playback, and UI search behavior are unchanged.
- CSV songs with finite numeric `danceability`, `energy`, `acousticness`, `valence`, and `tempo` values become graph nodes. Songs without these values are recorded in `skippedSongs`.
- Each feature is min-max normalized across valid songs; constant-valued features normalize to zero. Pairwise Euclidean distance across the five normalized features becomes a similarity weight: `1 - distance / sqrt(5)`.
- Each node stores up to 10 outgoing edges by default. Edges are directed: a song may choose another song without being chosen in return. Neighbors are ordered by descending similarity, then ascending `track_id`.
- `bfs(trackId, { limit })` and `dfs(trackId, { limit })` iteratively traverse reachable outgoing edges and return track IDs. Unknown starts return an empty array; `getNeighbors()` also returns an empty array for unknown IDs, while `getSong()` returns `null`.

**Example:**
```javascript
const graph = new DSA.SongGraph().buildFromSongs(songCatalog.inorderTraversal());
const neighbors = graph.getNeighbors('some-track-id');
console.log(neighbors); // [{ to: 'another-track-id', weight: 0.98 }, ...]
console.log(graph.bfs('some-track-id', { limit: 20 }));
```

**Complexity and Limitations:**
- Building the graph currently compares every valid song pair: O(n² × 5) time. Only the best `k` outgoing edges per node are retained, so graph storage is O(nk); this is suitable for the existing 5,000-row CSV load but not the full ~80,000-row dataset.
- Traversals take O(V + E) time and O(V) auxiliary space.
- The existing CSV loader converts missing or malformed numeric values to `0`. Per the chosen behavior, graph validation treats those resulting zeros as valid values; it cannot distinguish them from true zero-valued source data without changing the loader.
- Run graph tests with `node tests/graph.test.js`; run the existing B-Tree, CSV, and binary-search tests with `node tests/btree.test.js`.

---

### Smart Playlist Generator 🎼
**Purpose:** Generate a 20-song playlist from a seed track or an audio-feature mood profile.

**Logic and API:**
- `PlaylistGenerator` is implemented in `src/playlist/playlistGenerator.js`, receives the existing B-Tree catalog and SongGraph, and does not load CSV data itself.
- `generateFromSeed(trackId, options)` puts the valid seed first, collects direct graph neighbors and then expands through outgoing neighbors-of-neighbors up to `expansionDepth` (default 2). Similarity is re-calculated against the seed using `DSA.computeSimilarity`; candidates are ranked by similarity then `track_id`. It takes up to 19 and shuffles only those similar songs with Fisher-Yates.
- The local expansion is an approximation: an expanded song may not be among the seed's globally closest songs, and the bounded pool may miss better candidates. A full-catalog scan would be required for an exact global top-19.
- `generateFromMood(moodName, options)` filters graph songs using all rules in a mood profile, ranks matches by average normalized margin beyond the rule thresholds (then `track_id`), keeps the best 20, and shuffles them.
- `getMatchingSongs(moodName)` returns every valid match without shuffling; `matchesMood(song, moodName)` checks the profile. Mood keys are exported in `MOOD_NAMES`; matching accepts keys or display names case-insensitively.
- Both modes return `{ type, seed, mood, songs, requested, message }`. An optional `random` function enables repeatable shuffle tests. The generator is exposed as `window.playlistGenerator` after the CSV catalog and graph finish loading.
- The Generate page uses a multi-step wizard: name the playlist, choose Pick a Song or Pick a Mood, select a catalog seed or exported mood, then review the returned songs in their generated order.
- The wizard calls the existing `generateFromSeed(trackId)` or `generateFromMood(moodName)` method once per run. A UI-only 2.5-second loading screen shows progress messages without delaying or changing generator or DSA logic.
- Results stay temporary and do not appear on Home or in Library until **Save to Library** is chosen. Saving writes a copied song snapshot to `genmixSavedPlaylists` in the existing `{ id, name, createdAt, songs }` format. **Delete Playlist** discards only the current unsaved result.
- Leaving Generate resets the wizard. Duplicate playlist names are allowed, but the same generated result can only be saved once.

**Mood thresholds:** Rules use empirical quartiles from the first 5,000 CSV songs after the graph's min-max normalization. High thresholds are normalized P75; low thresholds are normalized P25: valence 0.677638, danceability 0.681725, energy 0.822211 (high) / 0.312949 (low), tempo 0.635812, acousticness 0.791164 (high). A song can match multiple profiles.

| Mood | Rules | Matches in first 5,000 CSV songs |
|------|-------|---------------------------------:|
| Happy / Upbeat | valence ≥ 0.677638 and danceability ≥ 0.681725 | 643 |
| Energetic | energy ≥ 0.822211 and normalized tempo ≥ 0.635812 | 413 |
| Chill | energy ≤ 0.312949 and acousticness ≥ 0.791164 | 1,012 |
| Melancholic | valence ≤ 0.217085 and energy ≤ 0.312949 | 752 |
| Acoustic | acousticness ≥ 0.791164 | 1,255 |

Run generator, UI helper, and wizard helper tests with `node tests/playlistGenerator.test.js`, `node tests/playlistUI.test.js`, and `node tests/playlistWizard.test.js`; app startup and generator integration checks use `node tests/app.test.js`. `npm test` runs every suite.

---

### 2. **Circular Queue** 🔄
**Purpose:** Playlist playback management

**Operations Implemented:**
- `enqueue()` - Add songs to playlist
- `dequeue()` - Remove songs from playlist
- `next()` - Move to next song (circular)
- `previous()` - Move to previous song (circular)
- `getCurrent()` - Get currently playing song
- `isEmpty()` - Check if queue is empty
- `clear()` - Empty the entire queue

**Use Case:**
- Playlist uses circular queue for seamless looping
- Next/Previous buttons navigate circularly through songs
- No need to restart when reaching the end of playlist

**Complexity:**
- All operations: O(1)
- Space: O(n) where n is capacity

**Special Feature:**
- Automatically handles wrapping around when reaching queue boundaries
- Tracks current song position separately from front/rear pointers

---

### 3. **Priority Queue (Max Heap)** ⭐
**Purpose:** Demonstrate priority-based song retrieval from a max heap

**Operations Implemented:**
- `insert()` - Add songs with priority based on rating
- `extractMax()` - Get highest rated song
- `getTopRecommendations(n)` - Get top N recommendations
- `heapifyUp()` - Maintain heap property when inserting
- `heapifyDown()` - Maintain heap property when extracting

**Use Case:**
- Songs are prioritized by their rating (1-10 scale)
- `getTopRecommendations(n)` returns a copy of the highest-rated songs
- Home recommendations are sampled randomly from the catalog, independently of this heap
- Efficient prioritization

**Complexity:**
- Insert: O(log n)
- ExtractMax: O(log n)
- getTopRecommendations: O(k log n) where k is number of recommendations
- Space: O(n)

---

### 4. **Heap Sort** 🔀
**Purpose:** Sort songs by rating

**Operations:**
- `heapSort()` - Sort songs by rating using heap
- Builds max heap and extracts elements in sorted order

**Use Case:**
- Demonstrates sorting songs from highest to lowest rating
- Demonstrates in-place sorting algorithm

**Complexity:**
- Time: O(n log n)
- Space: O(1) (in-place sorting)

---

## 🚀 Getting Started

### Prerequisites
- Node.js (v14 or higher)
- npm or yarn package manager

### Installation

1. **Clone or download the project**
   ```bash
   cd Music\ Playlist\ Organizer
   ```

2. **Install dependencies**
   ```bash
   npm install
   ```

3. **Run the application**
   ```bash
   npm run dev
   ```

4. **Open in browser**
   - The application will automatically open at `http://localhost:3010`
   - Or manually navigate to the provided URL

   Run all standalone suites with:
   ```bash
   npm test
   ```

### Usage

#### Adding Songs to the Playlist
1. Search the existing catalog by title
2. Browse matching songs two rows at a time with the smooth Previous/Next pagination
3. Click a matching result to add it to the current playlist

#### Saving and Viewing Playlists
1. Save generated playlists to Library from the generator results
2. Add recommendations to an existing playlist from Home, or add catalog songs from a playlist's Library detail page
3. Saved playlists are stored under `genmixSavedPlaylists` in this browser's localStorage
4. Home and Library display saved playlists and their song counts

#### Generating Playlists
1. Open **Generate** from the navigation menu
2. Choose **Create Playlist**, enter a name, then select **Pick a Song** or **Pick a Mood**
3. Select a catalog seed or one of the exported moods and generate the playlist
4. Choose **Save to Library** to keep it, or **Delete Playlist** to discard it without saving

#### Exporting to YouTube
1. Complete the one-time Google Cloud and local configuration in [docs/YOUTUBE_EXPORT_SETUP.md](docs/YOUTUBE_EXPORT_SETUP.md)
2. Generate a playlist or use **Export to YouTube** on a saved-playlist card
3. Confirm the name, privacy, and estimated quota use, then authorize your own Google account
4. GenMix preserves song order, reports unmatched or failed songs, and links to the created YouTube playlist

OAuth tokens stay in memory only. Accepted song-to-video matches are cached separately in localStorage to reduce later YouTube searches.

#### Searching Songs
1. Type in the search box
2. Matching song titles from the B-Tree catalog will appear
3. Click any result to add it to the playlist

#### Using the Simulated Queue
1. Use the existing play/pause control to start or pause the queue simulation
2. Use Next or Previous to move through the circular queue
3. No real audio is played
4. The queue is temporary; save generated playlists to Library to keep them in this browser

#### Recommendations
- Home pairs cycling music quotes with a carousel of up to 100 distinct songs, showing four columns and two rows at a time on desktop
- Click a song to pause the carousel and add it to an existing saved playlist
- Refresh or revisit Home for a new sample; recommendations do not alter the catalog or queue

---

## 📊 Algorithm Visualization

The application includes **real-time visualization** of DSA operations:
- See when songs are queued and when DSA operations run
- Track search operations in the B-Tree catalog
- Monitor circular queue navigation
- Watch heap operations during shuffle

Visualizations appear in the center panel's "Algorithm Visualization" section.

---

## 🏗️ Project Structure

```
GenMix/
├── index.html                  # Main HTML structure (served from the project root)
├── css/
│   └── styles.css              # UI styling
├── src/
│   ├── app.js                  # Application logic and UI interactions
│   ├── core/
│   │   └── dsa.js              # DSA implementations (B-Tree, binary search, Circular Queue, Max Heap, SongGraph)
│   ├── playlist/
│   │   ├── playlistGenerator.js  # Seed- and mood-based playlist logic
│   │   ├── playlistWizard.js     # Pure wizard state and saved-record helpers
│   │   ├── playlistEditor.js     # Pure saved-playlist edit helpers (add, remove, rename, move, delete)
│   │   └── playlistUI.js         # Pure UI search and display helpers
│   └── integrations/
│       └── youtube/            # YouTube auth, API client, matcher, cache, exporter, and export UI
├── data/
│   ├── songs_deduped.csv       # Song catalog loaded by the app (4,428 unique songs)
│   └── songs_cleaned.csv       # Original full dataset (kept for reference)
├── tests/                      # Standalone Node.js test suites (run with npm test)
│   └── youtube/                # YouTube integration tests
├── docs/                       # Quick start, project summary, YouTube setup, and project report
├── scripts/
│   ├── generate-config.js      # Builds config.local.js from .env
│   └── convert_to_word.py      # Converts docs/PROJECT_REPORT.md to Word
├── config.local.js             # Generated, git-ignored browser configuration
├── package.json                # Project configuration and npm scripts
└── README.md                   # This file
```

---

## 💡 Key Learning Outcomes

This project demonstrates:

1. **Real-world DSA applications**
   - How data structures solve practical problems
   - Trade-offs between different algorithms

2. **Time and Space Complexity**
   - Understanding Big-O notation in action
   - Optimal algorithm selection

3. **Data Structure Integration**
   - Multiple structures working together
   - Efficient data management strategies

4. **Algorithm Visualization**
   - Visual understanding of operations
   - Debugging and comprehension aid

---

## 🎨 Features

### User Interface
- Responsive blue theme with a sliding navigation menu
- Keyboard-accessible navigation with focus states
- Subtle blue gradients and restrained glass surfaces
- Seed- and mood-based playlist generation

### Functionality
- CSV-backed catalog search and browsing
- Saved-playlist collections on Home and Library
- 🔍 Track ID lookup with B-Tree and title search compatibility scan
- 🔄 Circular queue navigation simulation (no audio playback)
- ⭐ Random catalog recommendations
- 🎬 Visual algorithm execution

---

## 🔬 Technical Details

### Data Structures Comparison

| Structure | Use Case | Insert | Search | Extract Max | Space |
|-----------|----------|--------|--------|-------------|-------|
| **B-Tree** | Song catalog | O(log n) | O(log n) by track ID | - | O(n) |
| **Circular Queue** | Playlist | O(1) | - | O(1) | O(n) |
| **Max Heap** | Recommendations | O(log n) | - | O(log n) | O(n) |

### Catalog Data
- The startup library is loaded from `data/songs_deduped.csv` (4,428 unique songs).
- If the CSV cannot be loaded or yields no usable songs, the library displays an empty-state message; no demo songs are inserted.

---

## 📝 Future Enhancements

Potential additions for extended learning:
- [ ] B-Tree deletion as a separate extension
- [ ] Connect playlist generator to future UI controls
- [ ] Hash table for O(1) artist lookup
- [ ] Trie for autocomplete search
- [ ] Merge sort comparison
- [ ] Dijkstra's algorithm for playlist generation

---

## 🎓 Educational Value

This project serves as an excellent resource for:
- **Students** learning DSA
- **Developers** understanding algorithm applications
- **Educators** demonstrating concepts visually
- **Interview preparation** for DSA questions

---

## 📄 License

MIT License - Feel free to use this project for learning and education.

---

## 👨‍💻 Author

Created as a DSA project demonstrating practical implementations of fundamental data structures and algorithms.

---

## 🙏 Acknowledgments

- Algorithms and data structures based on standard computer science textbooks
- UI inspired by modern music streaming applications
- Educational resources from various DSA courses

---

**Built with ❤️ for DSA Learning**

*"The best way to learn algorithms is to implement them in a real project!"*
