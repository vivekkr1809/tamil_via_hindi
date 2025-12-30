# Bidirectional Language Learning - Required Changes Analysis

## Executive Summary

The Tamil Setu app is currently **hardcoded** to teach Tamil to Hindi speakers (Hindi → Tamil). To support bidirectional learning (Tamil → Hindi OR Hindi → Tamil), significant architectural changes are required across data models, UI, business logic, and configuration.

---

## Current State

### Language Direction
- **Fixed**: Hindi (Source) → Tamil (Target)
- **No user choice**: Direction cannot be changed
- **Hardcoded in**: Data models, UI text, quiz logic, TTS configuration, app title

### Technology Stack
- Flutter/Dart application
- State management: Provider pattern
- Data source: JSON file (`master_content.json`)
- Persistence: SharedPreferences

---

## Required Changes by Category

## 1. DATA MODEL CHANGES

### 1.1 WordPair Model Refactoring
**File**: `lib/models/word_pair.dart`

**Current Implementation**:
```dart
class WordPair {
  final String hindi;              // Hardcoded source
  final String tamil;              // Hardcoded target
  final String pronunciation;
  final String audioPath;
}
```

**Required Changes**:

**Option A: Add Direction-Aware Getters (Recommended)**
```dart
class WordPair {
  final String hindi;
  final String tamil;
  final String hindiPronunciation;  // NEW: Pronunciation for Hindi
  final String tamilPronunciation;   // Rename from 'pronunciation'
  final String hindiAudioPath;       // NEW: Audio for Hindi word
  final String tamilAudioPath;       // Rename from 'audioPath'

  // Direction-aware getters
  String getSource(LearningDirection direction) {
    return direction == LearningDirection.hindiToTamil ? hindi : tamil;
  }

  String getTarget(LearningDirection direction) {
    return direction == LearningDirection.hindiToTamil ? tamil : hindi;
  }

  String getPronunciation(LearningDirection direction) {
    return direction == LearningDirection.hindiToTamil
        ? tamilPronunciation
        : hindiPronunciation;
  }

  String getAudioPath(LearningDirection direction) {
    return direction == LearningDirection.hindiToTamil
        ? tamilAudioPath
        : hindiAudioPath;
  }
}
```

**Option B: Generic Source/Target Model** (More complex, requires full data migration)
```dart
class WordPair {
  final String sourceWord;
  final String targetWord;
  final String sourceLanguage;  // 'hi' or 'ta'
  final String targetLanguage;  // 'ta' or 'hi'
  final String pronunciation;
  final String audioPath;
}
```

**Recommendation**: Use Option A to preserve existing data structure while adding flexibility.

---

### 1.2 New Enum for Learning Direction
**New File**: `lib/models/learning_direction.dart`

```dart
enum LearningDirection {
  hindiToTamil,
  tamilToHindi;

  String get displayName {
    switch (this) {
      case hindiToTamil:
        return 'हिंदी → தமிழ் (Learn Tamil via Hindi)';
      case tamilToHindi:
        return 'தமிழ் → हिंदी (Learn Hindi via Tamil)';
    }
  }

  String get sourceLanguage {
    return this == hindiToTamil ? 'Hindi' : 'Tamil';
  }

  String get targetLanguage {
    return this == hindiToTamil ? 'Tamil' : 'Hindi';
  }

  String get ttsLanguageCode {
    // Language code for text-to-speech (target language)
    return this == hindiToTamil ? 'ta-IN' : 'hi-IN';
  }

  String get sourceLanguageCode {
    return this == hindiToTamil ? 'hi-IN' : 'ta-IN';
  }
}
```

---

## 2. CONFIGURATION & STATE MANAGEMENT

### 2.1 New Settings Provider
**New File**: `lib/providers/settings_provider.dart`

```dart
class SettingsProvider extends ChangeNotifier {
  static const String _directionKey = 'learning_direction';

  LearningDirection _learningDirection = LearningDirection.hindiToTamil;

  LearningDirection get learningDirection => _learningDirection;

  Future<void> initialize() async {
    final prefs = await SharedPreferences.getInstance();
    final savedDirection = prefs.getString(_directionKey);

    if (savedDirection != null) {
      _learningDirection = LearningDirection.values.firstWhere(
        (d) => d.toString() == savedDirection,
        orElse: () => LearningDirection.hindiToTamil,
      );
    }
    notifyListeners();
  }

  Future<void> setLearningDirection(LearningDirection direction) async {
    if (_learningDirection == direction) return;

    _learningDirection = direction;
    final prefs = await SharedPreferences.getInstance();
    await prefs.setString(_directionKey, direction.toString());
    notifyListeners();
  }

  // Helper methods for widgets
  String getSourceWord(WordPair pair) => pair.getSource(_learningDirection);
  String getTargetWord(WordPair pair) => pair.getTarget(_learningDirection);
  String getPronunciation(WordPair pair) => pair.getPronunciation(_learningDirection);
  String getAudioPath(WordPair pair) => pair.getAudioPath(_learningDirection);
}
```

### 2.2 Update Main App Initialization
**File**: `lib/main.dart`

**Changes Needed**:
```dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();

  final progressProvider = ProgressProvider();
  final themeProvider = ThemeProvider();
  final contentProvider = ContentProvider();
  final reviewProvider = ReviewProvider();
  final settingsProvider = SettingsProvider();  // NEW

  await Future.wait([
    progressProvider.loadProgress(),
    themeProvider.initialize(),
    contentProvider.loadContent(),
    reviewProvider.loadReviewCards(),
    settingsProvider.initialize(),              // NEW
  ]);

  runApp(
    MultiProvider(
      providers: [
        ChangeNotifierProvider.value(value: progressProvider),
        ChangeNotifierProvider.value(value: themeProvider),
        ChangeNotifierProvider.value(value: contentProvider),
        ChangeNotifierProvider.value(value: reviewProvider),
        ChangeNotifierProvider.value(value: settingsProvider),  // NEW
      ],
      child: const MyApp(),
    ),
  );
}
```

---

## 3. UI CHANGES

### 3.1 Language Selection Screen
**New File**: `lib/screens/language_selection_screen.dart`

**Purpose**: Allow users to choose learning direction

**Features Needed**:
- Two large selection cards: "Learn Tamil via Hindi" and "Learn Hindi via Tamil"
- Visual representation (flag icons, language scripts)
- Peacock mascot with bilingual greeting
- "Continue" button to save selection
- Show on first launch or accessible from settings

**Integration Point**:
- Show before DashboardScreen on first launch
- Add menu item in DashboardScreen to change direction

### 3.2 Update Dashboard Screen
**File**: `lib/screens/dashboard_screen.dart`

**Changes Required**:

1. **Dynamic App Title** (Line 23):
```dart
// CURRENT:
title: const Text('Tamil Setu (हिंदी ➡️ தமிழ்)'),

// NEW:
title: Consumer<SettingsProvider>(
  builder: (context, settings, _) {
    return Text('Tamil Setu (${settings.learningDirection.displayName})');
  },
),
```

2. **Dynamic Mascot Greeting** (Line 46):
```dart
// CURRENT:
PeacockMascot(message: 'नमस्ते! आज तमिल सीखते हैं?'),

// NEW:
Consumer<SettingsProvider>(
  builder: (context, settings, _) {
    final message = settings.learningDirection == LearningDirection.hindiToTamil
        ? 'नमस्ते! आज तमिल सीखते हैं?'
        : 'வணக்கம்! இன்று இந்தி கற்போமா?';  // Tamil: "Hello! Shall we learn Hindi today?"
    return PeacockMascot(message: message);
  },
)
```

3. **Add Language Direction Toggle**:
```dart
// In AppBar actions (Line ~30)
IconButton(
  icon: Icon(Icons.language),
  tooltip: 'Change Language Direction',
  onPressed: () {
    // Show dialog or navigate to language selection screen
  },
),
```

### 3.3 Update WordCard Widget
**File**: `lib/widgets/word_card.dart`

**Changes Required**:

```dart
// CURRENT CONSTRUCTOR:
class WordCard extends StatelessWidget {
  final String hindi;
  final String tamil;
  final String pronunciation;
  final VoidCallback onPlayAudio;

// NEW CONSTRUCTOR (Option 1 - Pass direction):
class WordCard extends StatelessWidget {
  final WordPair wordPair;
  final LearningDirection direction;
  final VoidCallback onPlayAudio;

  // In build method, use direction to determine which is source/target
  String get sourceWord => wordPair.getSource(direction);
  String get targetWord => wordPair.getTarget(direction);
  String get pronunciation => wordPair.getPronunciation(direction);

// NEW CONSTRUCTOR (Option 2 - Use provider):
class WordCard extends StatelessWidget {
  final WordPair wordPair;
  final VoidCallback onPlayAudio;

  // In build method
  @override
  Widget build(BuildContext context) {
    final settings = Provider.of<SettingsProvider>(context);
    final sourceWord = settings.getSourceWord(wordPair);
    final targetWord = settings.getTargetWord(wordPair);
    // ...
  }
```

**UI Layout Updates**:
- Keep color scheme (blue for source, orange for target)
- Update labels to be dynamic: "Source Word" / "Target Word" instead of hardcoded "Hindi" / "Tamil"
- Or use actual language names from settings

### 3.4 Update Lesson Screen
**File**: `lib/screens/lesson_screen.dart`

**Changes Required**:
- Pass `SettingsProvider` to `WordCard` widgets
- Update any hardcoded Hindi/Tamil references to use dynamic source/target

### 3.5 Update Quiz Screens

#### Multiple Choice Quiz
**File**: `lib/screens/multiple_choice_quiz.dart` (Line 79-91)

**Current Logic**:
```dart
final correctAnswer = shuffledWords[currentIndex].tamil;  // Always Tamil
final wrongAnswers = otherWords.map((w) => w.tamil).toList();
```

**New Logic**:
```dart
final settings = Provider.of<SettingsProvider>(context, listen: false);
final currentWord = shuffledWords[currentIndex];
final correctAnswer = settings.getTargetWord(currentWord);

final wrongAnswers = otherWords
    .map((w) => settings.getTargetWord(w))
    .toList();

// Question text should show source word
final questionWord = settings.getSourceWord(currentWord);
```

#### Flashcard Quiz
**File**: `lib/screens/quiz_view.dart`

**Similar changes**: Use `SettingsProvider` to determine which side is question/answer

#### Checkpoint Quiz
**File**: `lib/screens/checkpoint_quiz_screen.dart`

**Similar changes**: Apply direction-aware logic

---

## 4. SERVICE CHANGES

### 4.1 TTS Service
**File**: `lib/services/tts_service.dart`

**Current Implementation** (Line 24):
```dart
await _flutterTts.setLanguage('ta-IN');  // Hardcoded Tamil
```

**New Implementation**:
```dart
class TtsService {
  LearningDirection? _currentDirection;

  Future<void> setLearningDirection(LearningDirection direction) async {
    if (_currentDirection == direction) return;

    _currentDirection = direction;
    await _flutterTts.setLanguage(direction.ttsLanguageCode);
  }

  Future<void> speak(String text) async {
    if (_currentDirection == null) {
      // Get from provider or use default
      await _flutterTts.setLanguage('ta-IN');
    }
    await _flutterTts.speak(text);
  }
}
```

**Integration**:
- Listen to `SettingsProvider` changes
- Update TTS language when direction changes
- Ensure TTS speaks target language words

### 4.2 Review Service
**File**: `lib/services/review_storage_service.dart`

**Considerations**:
- Review cards should be direction-aware
- Option 1: Separate review decks for each direction
- Option 2: Store direction with each card
- When direction changes, decide whether to:
  - Reset all review progress (harsh but simple)
  - Maintain separate progress per direction (complex)
  - Migrate existing cards to new direction (confusing)

**Recommended Approach**: Maintain separate review progress per direction

```dart
class ReviewCard {
  final String id;
  final int lessonIndex;
  final int wordIndex;
  final LearningDirection direction;  // NEW FIELD
  // ... rest of fields
}
```

---

## 5. DATA CONTENT CHANGES

### 5.1 Update master_content.json
**File**: `assets/data/master_content.json`

**Current Structure**:
```json
{
  "hindi": "नमस्ते / हेलो",
  "tamil": "வணக்கம்",
  "pronunciation": "वणक्कम्",
  "audio_path": "assets/audio/l1_hello.mp3"
}
```

**New Structure Required**:
```json
{
  "hindi": "नमस्ते / हेलो",
  "tamil": "வணக்கம்",
  "tamil_pronunciation": "वणक्कम्",     // Current: for Hindi speakers
  "hindi_pronunciation": "namaste",     // NEW: for Tamil speakers (Tamil script or romanized)
  "tamil_audio_path": "assets/audio/l1_hello_ta.mp3",
  "hindi_audio_path": "assets/audio/l1_hello_hi.mp3"  // NEW
}
```

**Migration Task**:
- Add `hindi_pronunciation` field for each word (romanized or Tamil script)
- Add `hindi_audio_path` for each word (if audio feature needed for both directions)
- Rename `pronunciation` → `tamil_pronunciation`
- Rename `audio_path` → `tamil_audio_path`
- Update all 60+ word pairs across 12 lessons

**Alternative**: Keep pronunciation as-is initially, only show for Tamil target:
```dart
String? getPronunciation(LearningDirection direction) {
  if (direction == LearningDirection.hindiToTamil) {
    return tamilPronunciation;  // Devanagari pronunciation
  }
  // For Tamil to Hindi, could show romanization or skip
  return hindiPronunciation;  // Could be null initially
}
```

### 5.2 Audio Assets
**Directory**: `assets/audio/`

**Current State**: Audio files for Tamil words only

**Required**:
- **Option 1**: Record Hindi audio for all words (60+ files)
- **Option 2**: Rely on TTS for Hindi (simpler initially)
- **Option 3**: Prioritize and only add audio for key words

**Recommendation**: Use TTS for Hindi words initially, add curated audio later

---

## 6. PROGRESS & PERSISTENCE CHANGES

### 6.1 Progress Provider
**File**: `lib/providers/progress_provider.dart`

**Required Changes**:

**Option A: Separate Progress per Direction**
```dart
class ProgressProvider extends ChangeNotifier {
  // Current keys: 'unlockedLevel', 'completedLessons', 'completedCheckpoints'

  // New structure:
  // 'unlockedLevel_hindiToTamil', 'unlockedLevel_tamilToHindi'
  // 'completedLessons_hindiToTamil', 'completedLessons_tamilToHindi'
  // 'completedCheckpoints_hindiToTamil', 'completedCheckpoints_tamilToHindi'

  String _getKey(String baseKey, LearningDirection direction) {
    return '${baseKey}_${direction.name}';
  }

  Future<void> loadProgress(LearningDirection direction) async {
    final prefs = await SharedPreferences.getInstance();
    _currentUnlockedLevel = prefs.getInt(_getKey('unlockedLevel', direction)) ?? 1;
    // ... load other progress
  }
}
```

**Option B: Shared Progress**
- Same lessons unlocked regardless of direction
- Might confuse users if Tamil→Hindi lessons are marked complete when they only did Hindi→Tamil
- Not recommended

**Recommendation**: Use Option A (separate progress per direction)

### 6.2 Migration Strategy
When users update to bidirectional version:
1. Detect existing progress (old keys without direction suffix)
2. Migrate to `hindiToTamil` keys (preserve existing progress)
3. Initialize `tamilToHindi` keys with default values
4. Set initial direction preference to `hindiToTamil` for existing users

---

## 7. TESTING CHANGES

### 7.1 Unit Tests
**Files to Update**: All files in `test/` directory

**New Tests Needed**:
- WordPair model with direction-aware getters
- SettingsProvider direction switching
- Progress separation by direction
- Quiz logic for both directions

### 7.2 Widget Tests
- Language selection screen
- WordCard display for both directions
- Dashboard with dynamic title
- Quiz screens with reversed roles

### 7.3 Integration Tests
- Complete flow: Select direction → Complete lesson → Take quiz
- Switch direction and verify progress isolation
- TTS configuration for both languages

---

## 8. DOCUMENTATION CHANGES

### 8.1 README.md
**File**: `README.md`

**Updates Needed**:
- Change title from "Tamil via Hindi" to "Tamil Setu: Bidirectional Language Learning"
- Update description to mention both directions
- Add screenshots showing language selection
- Update feature list

### 8.2 App Store Descriptions
- Update Google Play / App Store descriptions
- Mention bidirectional learning capability
- Update screenshots

---

## IMPLEMENTATION PLAN

### Phase 1: Core Infrastructure (Breaking Changes)
**Priority**: High | **Estimated Complexity**: Medium

1. Create `LearningDirection` enum
2. Update `WordPair` model with direction-aware getters
3. Create `SettingsProvider` for direction management
4. Update `main.dart` to initialize settings provider
5. **Data Migration**: Update `master_content.json` with new field names
6. Update `fromJson` methods in models

### Phase 2: UI Updates
**Priority**: High | **Estimated Complexity**: Medium

1. Create `LanguageSelectionScreen`
2. Update `DashboardScreen` with dynamic title and greeting
3. Add language toggle button/menu item
4. Update `WordCard` widget to use provider
5. Update `LessonScreen` to pass direction context

### Phase 3: Quiz Logic Updates
**Priority**: High | **Estimated Complexity**: Medium

1. Update `MultipleChoiceQuiz` to use direction-aware logic
2. Update `QuizView` (flashcards) for both directions
3. Update `CheckpointQuizScreen` for direction awareness
4. Test all quiz types thoroughly

### Phase 4: Services & Storage
**Priority**: Medium | **Estimated Complexity**: High

1. Update `TtsService` for bidirectional TTS
2. Update `ReviewProvider` to separate cards by direction
3. Update `ProgressProvider` to separate progress by direction
4. Implement migration logic for existing users
5. Update review scheduling to be direction-aware

### Phase 5: Content & Assets
**Priority**: Low | **Estimated Complexity**: High

1. Add `hindi_pronunciation` for all words (romanized or Tamil script)
2. Decide on audio strategy (TTS vs recorded)
3. If recorded: Create Hindi audio files
4. Update asset bundle

### Phase 6: Testing & Polish
**Priority**: High | **Estimated Complexity**: Medium

1. Write unit tests for all new functionality
2. Write widget tests for new screens
3. Manual testing of both directions
4. Fix edge cases and bugs
5. Update documentation

---

## RISK ASSESSMENT

### High Risk
1. **Data Migration**: Existing users losing progress
   - **Mitigation**: Careful migration logic, backup before migration

2. **Breaking Changes**: `WordPair` model changes affect entire app
   - **Mitigation**: Thorough testing, staged rollout

3. **Audio Assets**: 60+ new audio files if going with recorded approach
   - **Mitigation**: Use TTS initially, add curated audio later

### Medium Risk
1. **UI Complexity**: Managing state across multiple providers
   - **Mitigation**: Clear provider hierarchy, good documentation

2. **Testing Coverage**: Many new code paths to test
   - **Mitigation**: Comprehensive test suite before release

### Low Risk
1. **TTS Configuration**: Hindi TTS may have quality issues
   - **Mitigation**: Test with multiple TTS voices, allow user configuration

---

## ALTERNATIVE APPROACHES

### Approach 1: Duplicate App (Not Recommended)
- Create separate "Hindi via Tamil" app
- **Pros**: No code complexity, simple
- **Cons**: Maintenance burden, user confusion, violates DRY principle

### Approach 2: Runtime Data Transformation (Current Recommendation)
- Keep data in current format, transform based on direction
- **Pros**: Minimal data changes, existing content reused
- **Cons**: Some runtime overhead (negligible)

### Approach 3: Fully Generic Data Model
- Restructure data to be completely language-agnostic
- **Pros**: Extensible to other language pairs (Tamil-Telugu, etc.)
- **Cons**: High complexity, overkill for current requirement

**Recommendation**: Use Approach 2 with option to evolve to Approach 3 later if needed

---

## FILES THAT NEED MODIFICATION

### Must Modify (23 files)

1. **Models** (4 files)
   - `lib/models/word_pair.dart` - Add direction-aware getters
   - `lib/models/lesson.dart` - May need updates if lesson titles are language-specific
   - `lib/models/review_card.dart` - Add direction field
   - `lib/models/checkpoint.dart` - Review for language-specific content

2. **Providers** (4 files)
   - `lib/providers/content_provider.dart` - Handle direction context
   - `lib/providers/progress_provider.dart` - Separate progress by direction
   - `lib/providers/review_provider.dart` - Direction-aware reviews
   - `lib/providers/theme_provider.dart` - May coordinate with settings

3. **Screens** (7 files)
   - `lib/screens/dashboard_screen.dart` - Dynamic title, greeting, language toggle
   - `lib/screens/lesson_screen.dart` - Pass direction to word cards
   - `lib/screens/quiz_view.dart` - Reverse question/answer
   - `lib/screens/multiple_choice_quiz.dart` - Direction-aware options
   - `lib/screens/checkpoint_quiz_screen.dart` - Direction-aware quiz
   - `lib/screens/review_screen.dart` - Direction-aware review cards
   - `lib/screens/stats_screen.dart` - May show direction-specific stats

4. **Widgets** (2 files)
   - `lib/widgets/word_card.dart` - Use direction for display
   - `lib/widgets/peacock_mascot.dart` - Potential bilingual messages

5. **Services** (3 files)
   - `lib/services/tts_service.dart` - Dynamic language selection
   - `lib/services/review_storage_service.dart` - Store direction with cards
   - `lib/services/progress_service.dart` - Direction-aware progress

6. **Core** (2 files)
   - `lib/main.dart` - Add SettingsProvider
   - `lib/data/curriculum.dart` - May need direction-aware loading

7. **Data** (1 file)
   - `assets/data/master_content.json` - Add new fields

### Must Create (2 files)

1. `lib/models/learning_direction.dart` - Enum and helpers
2. `lib/screens/language_selection_screen.dart` - UI for direction choice
3. `lib/providers/settings_provider.dart` - Settings management

---

## ESTIMATED EFFORT

Based on complexity and scope:

- **Phase 1 (Core)**: 2-3 days
- **Phase 2 (UI)**: 2-3 days
- **Phase 3 (Quiz)**: 1-2 days
- **Phase 4 (Services)**: 3-4 days
- **Phase 5 (Content)**: 1-2 days (without audio recording)
- **Phase 6 (Testing)**: 2-3 days

**Total**: 11-17 development days for full bidirectional support

**Minimum Viable Implementation** (Phases 1-3 only): 5-8 days

---

## CONCLUSION

Converting Tamil Setu from unidirectional (Hindi→Tamil) to bidirectional (Hindi↔Tamil) is a **medium-to-large refactoring effort** that touches most layers of the application. The core challenge is introducing direction awareness while maintaining backward compatibility for existing users.

**Recommended Strategy**:
1. Start with core infrastructure (models, providers)
2. Update UI to support direction selection
3. Modify quiz logic for bidirectional testing
4. Use TTS for Hindi audio initially (defer recorded audio)
5. Implement careful migration for existing users
6. Comprehensive testing before release

**Key Success Factors**:
- Thorough testing of both directions
- Smooth migration for existing users
- Clear UI for direction selection
- Maintaining separate progress per direction
- Quality TTS for both languages

This analysis provides a complete roadmap for implementing bidirectional language learning in Tamil Setu.
