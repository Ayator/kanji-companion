# RTK Companion App - Implementation Plan

## Overview
A React Native (Expo) mobile application designed as a companion to James Heisig's *Remembering the Kanji* (RTK). The app relies entirely on local storage (SQLite) to remain functional completely offline, ensuring no cloud database costs. It integrates dictionary features via public APIs and handwriting animations to enforce character recognition and retention.

## Technology Stack
*   **Framework:** React Native via Expo [docs.expo](https://docs.expo.dev/versions/latest/sdk/sqlite/)
*   **Local Database:** `expo-sqlite` (for offline data storage and SRS tracking)
*   **Kanji Animations/Quizzes:** `@jamsch/react-native-hanzi-writer` (wrapper around Hanzi Writer utilizing SVG for stroke order animation) [github](https://github.com/jamsch/react-native-hanzi-writer)
*   **Dictionary API:** `kanjiapi.dev` (Free REST API for external meanings, readings, and example vocabulary)
*   **Gestures & UI:** `react-native-gesture-handler` (required for drawing gestures) and standard React Native components.

## Database Architecture (SQLite)
The application will use `expo-sqlite` initialized with a prepopulated `.db` file (or seeded on first launch via JSON/CSV).

### Schema
1.  **`Kanji` Table**
    *   `id` (INTEGER PRIMARY KEY) - Heisig Index (1 to 2200).
    *   `character` (TEXT) - The Japanese Kanji.
    *   `keyword` (TEXT) - Heisig's core English meaning.
2.  **`Component` Table (Primitives)**
    *   `id` (INTEGER PRIMARY KEY)
    *   `character` (TEXT) - The visual radical/primitive.
    *   `keyword` (TEXT) - Heisig's name for the primitive (e.g., "sun", "animal legs").
3.  **`KanjiComponents` Table (Many-to-Many)**
    *   `kanji_id` (INTEGER) - Foreign key to `Kanji`.
    *   `component_id` (INTEGER) - Foreign key to `Component`.
    *   *Purpose:* Enables querying study lists by component (e.g., "Find all Kanji containing the 'sun' primitive").
4.  **`StudyData` Table**
    *   `kanji_id` (INTEGER PRIMARY KEY) - Foreign key to `Kanji`.
    *   `custom_story` (TEXT) - User's personalized Heisig mnemonic.
    *   `next_review` (DATETIME) - Next scheduled SRS review date.
    *   `srs_interval` (INTEGER) - Current interval in days.
    *   `ease_factor` (REAL) - SM-2 ease multiplier.

## Feature 1: Learning Mode
**Goal:** To be used alongside reading the RTK book to cement new kanji.

1.  **Kanji Display & Animation:** Uses `writer.animator.start()` from the Hanzi Writer library to automatically draw the correct stroke order. [github](https://github.com/jamsch/react-native-hanzi-writer)
2.  **Writing Practice:** Users can switch to a practice canvas where `writer.quiz.start()` lets them trace the kanji at least once to build muscle memory. [github](https://github.com/jamsch/react-native-hanzi-writer)
3.  **Mnemonic Entry:** A `TextInput` field allowing the user to type and save their custom Heisig story into the `StudyData` table.
4.  **Dictionary Lookup (Motivation):** When loading a kanji, `fetch()` from `https://kanjiapi.dev/v1/kanji/{character}` and `https://kanjiapi.dev/v1/words/{character}`. This displays Onyomi/Kunyomi readings and example words below the Heisig content for extra context.

## Feature 2: Study Mode (SRS)
**Goal:** Spaced Repetition testing (Keyword → Kanji) with autonomous grading and visual stroke feedback.

1.  **The Prompt:** Display the Heisig Keyword text. Do not show the Kanji.
2.  **The Canvas:** Render the `<HanziWriter>` component inside a `<GestureHandlerRootView>` with `quizMode` active.
3.  **Stroke Feedback (Visual Only):** 
    *   As the user draws with their finger, Hanzi Writer highlights correct strokes in green.
    *   Incorrect strokes get a red highlight/fade out (`<HanziWriter.QuizMistakeHighlighter />`). 
    *   Use the `leniency: 0.8` (or similar) prop inside `writer.quiz.start({ leniency: 0.8 })` to forgive minor inaccuracies on small phone screens.
    *   *Crucial:* This stroke validation does *not* automatically calculate the user's score.
4.  **Show Answer & Grading:**
    *   A "Show Answer" button reveals the fully drawn character.
    *   Display four Anki-style buttons: **Again (1), Hard (2), Good (3), Easy (4)**.
    *   The user autonomously decides their grade based on how well they recalled the character and stroke order.
5.  **SRS Algorithm:** Implements the SuperMemo-2 (SM-2) algorithm. [forums.ankiweb](https://forums.ankiweb.net/t/sm-2-algorithm-pseudo-code/8350)
    *   If "Again" (1): `srs_interval = 1`, `ease_factor -= 0.20`.
    *   If "Good/Easy" (3/4): `srs_interval = (srs_interval * ease_factor)`, `ease_factor += 0.15`.
    *   Update `next_review` in the `StudyData` SQLite table.

## Feature 3: Component Tree Explorer
**Goal:** Query Kanji families based on shared Heisig primitives.

1.  **UI:** A list or grid of Heisig primitives (Components).
2.  **Query Engine:** Tapping a primitive (e.g., "tree") executes an SQLite `JOIN` query via `db.transaction()`:
    ```sql
    SELECT K.id, K.character, K.keyword 
    FROM Kanji K 
    JOIN KanjiComponents KC ON K.id = KC.kanji_id 
    JOIN Component C ON KC.component_id = C.id 
    WHERE C.keyword = 'tree'
    ```
3.  **Action:** The resulting list of Kanji can be sent directly to the "Study Mode" queue for a targeted drill session.

## Phased Development Plan

### Phase 1: Setup & Data Prep
*   Initialize Expo app: `npx create-expo-app rtk-companion`
*   Install dependencies: `expo-sqlite`, `react-native-svg`, `react-native-gesture-handler`, `@jamsch/react-native-hanzi-writer`.
*   Download an open-source RTK Kanji CSV.
*   Write a one-time setup script (`useEffect` in `App.js`) to parse the CSV and seed the `expo-sqlite` tables if they are empty. [w3resource](https://www.w3resource.com/sqlite/snippets/working-with-expo-sqlite.php)

### Phase 2: Core UI & Learning Mode
*   Create the Bottom Tab Navigator (Learning, Study, Components).
*   Build the Learning Screen UI (Keyword, Custom Story Input).
*   Implement `useHanziWriter` to animate the character.}
*   Integrate the `kanjiapi.dev` fetch calls for external dictionary data.

### Phase 3: Study Mode & Gestures
*   Wrap the app in `<GestureHandlerRootView>`.
*   Implement the `<HanziWriter.Svg>` canvas for user drawing input.
*   Build the SM-2 SRS logic function.
*   Wire the 1-4 grading buttons to update SQLite's `next_review` timestamps.

### Phase 4: Component Tree & Polish
*   Build the Component Explorer list.
*   Implement the SQLite M2M `JOIN` query to filter kanji.
*   Add offline queue management (e.g., "You have 45 reviews due today").
*   Finalize styling and test small-screen touch leniency on physical devices.
