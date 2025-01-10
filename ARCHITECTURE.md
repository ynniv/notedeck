# NoteDeck Architecture

NoteDeck is a modern Nostr client GUI written in Rust using the egui/eframe framework. This document details the internal architecture and key design decisions.

## Core Components Overview

### Application Structure

The application is organized into four main crates:

1. **notedeck** - Core library providing:
   - Account management and persistence
   - Data caching and storage abstractions
   - Nostr protocol handling primitives
   - Theme and configuration management

2. **notedeck_chrome** - Main GUI application shell containing:
   - Application initialization and lifecycle management
   - Window/panel layout coordination
   - Cross-cutting concerns like logging

3. **notedeck_columns** - Column-based UI implementation:
   - Timeline views and navigation
   - Note display and interaction
   - Profile management
   - Thread visualization

4. **enostr** - Nostr protocol implementation:
   - Event handling and validation
   - Relay pool management
   - Key management and cryptography

### Key Data Flows

#### Event Pipeline and Timeline System

The timeline system is a core architectural component that manages the flow and display of Nostr events:

1. **Initialization Flow**
   - Each `Timeline` is created with a specific `TimelineKind` (e.g., contacts, hashtags)
   - Timelines contain multiple `TimelineTab` views (e.g., "Notes Only", "Notes & Replies")
   - Each timeline maintains its own `FilterStates` for different relay connections

2. **Data Loading Pipeline**
   ```
   RelayPool -> WebsocketRelay 
       -> FilterState Processing
           -> NoteCache 
               -> Timeline Views
   ```

3. **Key Components**:

   - `Timeline`: Core container managing filtered views and subscriptions
   - `TimelineTab`: Individual view instance with view-specific filtering
   - `FilterStates`: Per-relay filter management
   - `UnknownIds`: Tracks and resolves unknown references
   - `Subscription`: nostrdb subscription for local querying

4. **Filter Resolution Process**:
   
   a. Initial State:
      - Timeline starts with `FilterState::NeedsRemote`
      - System requests contact lists or other required data
   
   b. Data Fetching:
      - Transitions to `FilterState::FetchingRemote`
      - Manages async loading from multiple relays
   
   c. Ready State:
      - Reaches `FilterState::Ready` when data is available
      - Sets up nostrdb subscriptions
      - Begins normal event processing

5. **Event Processing**
   
   - Events enter through `RelayPool`
   - Filtered through subscription system
   - Processed by `poll_notes_into_view`:
     1. Poll for new note IDs
     2. Load full notes from database
     3. Update unknown references
     4. Apply view-specific filters
     5. Update timeline views

6. **Optimization Features**
   
   - Efficient merge sorting for new events
   - Smart view recycling for UI performance
   - "Since" optimization for incremental loading
   - Automatic limit enforcement for remote queries

#### Account Management

- `Accounts` struct manages multiple user identities
- Each account has associated `AccountData` containing:
  - Public/private keypairs via `Keypair`
  - Relay preferences (`AccountRelayData`) 
  - Muted users/content (`AccountMutedData`)
- Account state persists via OS keychain or encrypted file storage

### Storage Architecture

The system implements a sophisticated multi-layered storage architecture:

1. **Local Database** (`nostrdb`)
   - High-performance event storage and indexing
   - Query optimizations for timeline views
   - Location configurable via `--dbpath`
   - Primary store for note content and metadata

2. **File System Layer** (`DataPath`)
   ```
   notedeck/
   ├── logs/            # Application logs
   ├── settings/        # User configurations
   ├── storage/
   │   ├── accounts/    # Encrypted key storage
   │   └── selected_account/ 
   ├── db/             # nostrdb files
   └── cache/          # Cached resources
   ```
   
   Key features:
   - Cross-platform path management
   - Directory-based organization
   - Atomic file operations
   - Most-recent file tracking
   - Line-based file access

3. **Memory Cache System**
   
   a. **Note Cache** (`NoteCache`)
   - Fast lookup of note metadata
   - Cached reply relationships
   - Time-based string formatting
   - Optimized memory usage
   ```rust
   struct CachedNote {
       reltime: TimeCached<String>,  // Cached time display
       reply: NoteReplyBuf,          // Reply metadata
   }
   ```

   b. **Time Cache** (`TimeCached<T>`)
   - Generic TTL-based caching
   - Automatic value regeneration
   - Memory cleanup
   - Thread-safe access

   c. **Image Cache**
   - Profile picture caching
   - Media content buffering
   - Size-based eviction

4. **Cache Coordination**
   - Lazy loading strategies
   - Cross-cache consistency
   - Memory pressure management
   - Background cleanup

### UI Architecture

The UI follows a Model-View pattern with unidirectional data flow:

1. **Models** - Core data structures:
   - `Timeline` - Manages filtered event streams
   - `Profile` - User profile data
   - `Thread` - Conversation threading
   - `Draft` - Post composition state

2. **Post System**
   
   The posting system handles new notes, replies, and quotes through a unified architecture:

   a. **Key Components**:
      - `PostView`: Main UI component for post composition
      - `PostAction`: Execution logic for different post types
      - `Draft`: Temporary storage for in-progress posts
      - `NewPost`: Final post data ready for submission

   b. **Post Types**:
      ```rust
      enum PostType {
          New,              // Fresh post
          Quote(NoteId),    // Quote with reference
          Reply(NoteId)     // Direct reply
      }
      ```

   c. **Post Flow**:
      1. User enters content through `PostView`
      2. System creates appropriate `PostAction`
      3. Action executes:
         - Generates note with correct references
         - Signs with user's key
         - Broadcasts to relay pool
         - Clears draft

   d. **UI Features**:
      - Integrated profile picture display
      - Draft preview system
      - Interactive post button state
      - Focus-based visual feedback

3. **Views** - UI Components:
   - `TimelineView` - Main column display
   - `ProfileView` - User profile display
   - `ThreadView` - Threaded conversation
   - `NoteView` - Individual note rendering

3. **State Management**:
   - `ViewState` tracks UI state per component
   - `DeckState` manages column layouts
   - `Router<R>` handles navigation
   - `AnimationHelper` coordinates transitions

### Extension Points

1. **Column Types**
   - New timeline views can be added by implementing the `Timeline` trait
   - Custom filters via `FilterStates` and `UnifiedSubscription`

2. **Storage Backends**
   - Abstract `KeyStorage` trait allows different key storage implementations
   - Modular cache implementations via traits

3. **Theme System**
   - `ColorTheme` customization
   - Style variants via `ThemeHandler`

## Performance Considerations

1. **Event Processing**
   - Efficient event filtering using nostrdb indexes
   - Background processing of relay messages
   - Rate limiting and backpressure via relay pool

2. **UI Optimization**
   - `FrameHistory` tracking for performance monitoring
   - Lazy loading of images and content
   - View recycling in timeline scrolling

3. **Resource Management**
   - `TimeCached<T>` automatic cleanup of temporary data
   - Image cache size limits
   - Connection pooling for relays

## Architectural Decisions

1. **Why Egui/Eframe?**
   - Immediate mode GUI enables simple state management
   - Cross-platform support including web (future)
   - Good performance for timeline scrolling
   - Native integration with Rust async patterns

2. **Column-Based Design**
   The column-based architecture was chosen for several reasons:
   - Natural parallel streams of content
   - Independent view state management
   - Flexible layout adaptation
   - Efficient subscription management per column
   - Easy addition of new view types

3. **Layered Storage Design**
   The multi-layered storage approach provides:
   - Separation of concerns between data types
   - Performance optimization per storage type
   - Flexible caching strategies
   - Simple backup and sync options
   - Graceful degradation options

4. **State Management Patterns**
   Several key patterns improve maintainability:
   - Immutable data structures where possible
   - Explicit state transitions
   - Central subscription coordination
   - Atomic file operations
   - Clear ownership boundaries

5. **Performance Optimizations**
   Strategic choices for performance include:
   - nostrdb for efficient event storage
   - Smart caching at multiple levels
   - Lazy loading of resources
   - Background processing where appropriate
   - Memory-conscious data structures

6. **Security Considerations**
   Security is addressed through:
   - Encrypted key storage options
   - Secure default paths
   - Memory zeroing for sensitive data
   - Validation of all nostr events
   - Safe file system operations

7. **Error Handling Strategy**
   The error handling approach:
   - Type-safe error propagation
   - Detailed error contexts
   - Graceful degradation
   - Clear error boundaries
   - User-friendly error reporting

8. **Testing Architecture**
   Testing is facilitated by:
   - Mock relay support
   - Test-specific file paths
   - Simulation capabilities
   - Property-based testing options
   - Integration test scaffolding