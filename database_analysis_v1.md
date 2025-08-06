# Complete Database Schema Design cho Dự án Học Tiếng Việt

## 1. User Service - PostgreSQL Database

### Core Tables:
```sql
-- Users table - Main user entity
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE,
    full_name VARCHAR(255),
    avatar_url TEXT,
    phone VARCHAR(20),
    email_verified BOOLEAN DEFAULT FALSE,
    status VARCHAR(20) DEFAULT 'active', -- active, inactive, suspended
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- User profiles - Extended user information
CREATE TABLE user_profiles (
    user_id UUID PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
    gender VARCHAR(10),
    birthday DATE,
    bio TEXT,
    country_code VARCHAR(3),
    timezone VARCHAR(50),
    native_language VARCHAR(10),
    target_languages VARCHAR(10)[] DEFAULT ARRAY['vi'],
    proficiency_level INTEGER CHECK (proficiency_level BETWEEN 1 AND 10),
    learning_goals TEXT[],
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- User settings
CREATE TABLE user_settings (
    user_id UUID PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
    language_preference VARCHAR(10) DEFAULT 'en',
    theme_mode VARCHAR(10) DEFAULT 'light',
    learning_mode VARCHAR(20) DEFAULT 'standard', -- standard, intensive, casual
    notification_enabled BOOLEAN DEFAULT TRUE,
    email_notifications BOOLEAN DEFAULT TRUE,
    push_notifications BOOLEAN DEFAULT TRUE,
    sound_enabled BOOLEAN DEFAULT TRUE,
    auto_play_audio BOOLEAN DEFAULT FALSE,
    daily_goal_minutes INTEGER DEFAULT 30,
    reminder_time TIME DEFAULT '19:00:00',
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- User authentication & security
CREATE TABLE user_auth (
    user_id UUID PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
    password_hash TEXT NOT NULL,
    salt TEXT NOT NULL,
    last_password_change TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    failed_login_attempts INTEGER DEFAULT 0,
    locked_until TIMESTAMP,
    two_factor_enabled BOOLEAN DEFAULT FALSE,
    two_factor_secret TEXT,
    recovery_codes TEXT[],
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Social login providers
CREATE TABLE user_social_logins (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    provider VARCHAR(50) NOT NULL, -- google, facebook, apple
    provider_user_id TEXT NOT NULL,
    provider_email TEXT,
    access_token_hash TEXT,
    refresh_token_hash TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(provider, provider_user_id)
);

-- User sessions
CREATE TABLE user_sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    session_token TEXT UNIQUE NOT NULL,
    device_info JSONB,
    ip_address INET,
    user_agent TEXT,
    is_active BOOLEAN DEFAULT TRUE,
    expires_at TIMESTAMP NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- User learning streaks
CREATE TABLE user_streaks (
    user_id UUID PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
    current_streak INTEGER DEFAULT 0,
    longest_streak INTEGER DEFAULT 0,
    last_activity_date DATE,
    streak_freeze_count INTEGER DEFAULT 0, -- số lần được "đóng băng" streak
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- User achievements/badges
CREATE TABLE user_achievements (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    achievement_type VARCHAR(50) NOT NULL, -- first_lesson, 7_day_streak, 100_words, etc.
    achievement_data JSONB, -- metadata about the achievement
    earned_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(user_id, achievement_type)
);

-- Indexes
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_status ON users(status);
CREATE INDEX idx_user_sessions_token ON user_sessions(session_token);
CREATE INDEX idx_user_sessions_user_active ON user_sessions(user_id, is_active);
CREATE INDEX idx_user_social_provider ON user_social_logins(provider, provider_user_id);
```

## 2. Content Service - PostgreSQL Database

```sql
-- Words dictionary - Main vocabulary database
CREATE TABLE words (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    text TEXT NOT NULL,
    language_code VARCHAR(10) NOT NULL,
    part_of_speech VARCHAR(50), -- noun, verb, adjective, etc.
    phonetic TEXT, -- IPA notation
    difficulty_level INTEGER CHECK (difficulty_level BETWEEN 1 AND 10),
    frequency_rank INTEGER, -- từ phổ biến thứ mấy
    is_common BOOLEAN DEFAULT FALSE,
    etymology TEXT,
    related_words UUID[], -- array of word IDs
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(text, language_code)
);

-- Word definitions - Multiple definitions per word
CREATE TABLE word_definitions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    word_id UUID REFERENCES words(id) ON DELETE CASCADE,
    definition TEXT NOT NULL,
    translation TEXT,
    target_language VARCHAR(10), -- ngôn ngữ của translation
    context VARCHAR(100), -- formal, informal, slang, etc.
    usage_notes TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Word examples - Example sentences
CREATE TABLE word_examples (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    word_id UUID REFERENCES words(id) ON DELETE CASCADE,
    example_text TEXT NOT NULL,
    translation TEXT,
    target_language VARCHAR(10),
    audio_url TEXT,
    context VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Paragraphs - Reading content
CREATE TABLE paragraphs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title TEXT,
    content TEXT NOT NULL,
    content_vector tsvector GENERATED ALWAYS AS (to_tsvector('simple', content)) STORED,
    language_code VARCHAR(10) NOT NULL,
    difficulty_level INTEGER CHECK (difficulty_level BETWEEN 1 AND 10),
    topic_category VARCHAR(50), -- daily_life, business, travel, etc.
    estimated_read_time INTEGER, -- seconds
    word_count INTEGER,
    audio_url TEXT,
    author VARCHAR(255),
    source VARCHAR(255),
    is_published BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Paragraph tokens - Word annotations in paragraphs
CREATE TABLE paragraph_tokens (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    paragraph_id UUID REFERENCES paragraphs(id) ON DELETE CASCADE,
    start_index INTEGER NOT NULL,
    end_index INTEGER NOT NULL,
    text TEXT NOT NULL,
    word_id UUID REFERENCES words(id),
    token_type VARCHAR(20) DEFAULT 'word', -- word, punctuation, number
    is_clickable BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Content topics/categories
CREATE TABLE content_topics (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) NOT NULL,
    description TEXT,
    parent_topic_id UUID REFERENCES content_topics(id),
    color_code VARCHAR(7), -- hex color
    icon_name VARCHAR(50),
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(name)
);

-- Word-topic relationships
CREATE TABLE word_topics (
    word_id UUID REFERENCES words(id) ON DELETE CASCADE,
    topic_id UUID REFERENCES content_topics(id) ON DELETE CASCADE,
    relevance_score DECIMAL(3,2) DEFAULT 1.0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (word_id, topic_id)
);

-- Content tags
CREATE TABLE content_tags (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(50) UNIQUE NOT NULL,
    description TEXT,
    tag_type VARCHAR(20) DEFAULT 'general', -- general, difficulty, topic, format
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Content-tag relationships
CREATE TABLE content_tag_relations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    content_type VARCHAR(20) NOT NULL, -- word, paragraph, lesson
    content_id UUID NOT NULL,
    tag_id UUID REFERENCES content_tags(id) ON DELETE CASCADE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(content_type, content_id, tag_id)
);

-- Word synonyms and antonyms
CREATE TABLE word_relations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    word_id UUID REFERENCES words(id) ON DELETE CASCADE,
    related_word_id UUID REFERENCES words(id) ON DELETE CASCADE,
    relation_type VARCHAR(20) NOT NULL, -- synonym, antonym, similar, related
    strength DECIMAL(3,2) DEFAULT 1.0, -- độ mạnh của mối quan hệ
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(word_id, related_word_id, relation_type)
);

-- Indexes
CREATE INDEX idx_words_text_lang ON words(text, language_code);
CREATE INDEX idx_words_language ON words(language_code);
CREATE INDEX idx_words_difficulty ON words(difficulty_level);
CREATE INDEX idx_words_frequency ON words(frequency_rank);
CREATE INDEX idx_paragraphs_language ON paragraphs(language_code);
CREATE INDEX idx_paragraphs_difficulty ON paragraphs(difficulty_level);
CREATE INDEX idx_paragraphs_topic ON paragraphs(topic_category);
CREATE INDEX idx_paragraphs_content_vector ON paragraphs USING GIN(content_vector);
CREATE INDEX idx_paragraph_tokens_paragraph ON paragraph_tokens(paragraph_id);
CREATE INDEX idx_paragraph_tokens_word ON paragraph_tokens(word_id);
CREATE INDEX idx_word_definitions_word ON word_definitions(word_id);
CREATE INDEX idx_word_examples_word ON word_examples(word_id);
```

## 3. Lesson Service - PostgreSQL Database

```sql
-- Lessons - Main lesson structure
CREATE TABLE lessons (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title VARCHAR(255) NOT NULL,
    description TEXT,
    language_code VARCHAR(10) NOT NULL DEFAULT 'vi',
    target_language VARCHAR(10) NOT NULL DEFAULT 'en', -- ngôn ngữ người học
    difficulty_level INTEGER CHECK (difficulty_level BETWEEN 1 AND 10),
    estimated_duration INTEGER, -- minutes
    lesson_type VARCHAR(30) DEFAULT 'reading', -- reading, listening, speaking, mixed
    topic_category VARCHAR(50),
    prerequisite_lessons UUID[],
    learning_objectives TEXT[],
    thumbnail_url TEXT,
    is_published BOOLEAN DEFAULT FALSE,
    is_premium BOOLEAN DEFAULT FALSE,
    view_count INTEGER DEFAULT 0,
    rating_average DECIMAL(3,2) DEFAULT 0,
    rating_count INTEGER DEFAULT 0,
    created_by UUID, -- instructor/admin user_id
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Lesson content - Paragraphs in lessons with ordering
CREATE TABLE lesson_paragraphs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    lesson_id UUID REFERENCES lessons(id) ON DELETE CASCADE,
    paragraph_id UUID NOT NULL, -- reference to content service
    position INTEGER NOT NULL,
    is_required BOOLEAN DEFAULT TRUE,
    instruction_text TEXT, -- hướng dẫn cho đoạn này
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(lesson_id, paragraph_id),
    UNIQUE(lesson_id, position)
);

-- Lesson exercises/activities
CREATE TABLE lesson_activities (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    lesson_id UUID REFERENCES lessons(id) ON DELETE CASCADE,
    activity_type VARCHAR(30) NOT NULL, -- quiz, fill_blank, match, speaking, etc.
    title VARCHAR(255),
    instruction TEXT,
    content JSONB NOT NULL, -- exercise content/questions
    position INTEGER,
    is_required BOOLEAN DEFAULT TRUE,
    points INTEGER DEFAULT 10,
    time_limit INTEGER, -- seconds, null = no limit
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- User lesson progress
CREATE TABLE user_lesson_progress (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL, -- from user service
    lesson_id UUID REFERENCES lessons(id) ON DELETE CASCADE,
    status VARCHAR(20) DEFAULT 'not_started', -- not_started, in_progress, completed, failed
    progress_percent INTEGER DEFAULT 0 CHECK (progress_percent >= 0 AND progress_percent <= 100),
    current_paragraph_id UUID,
    completed_paragraphs UUID[] DEFAULT '{}',
    completed_activities UUID[] DEFAULT '{}',
    total_time_spent INTEGER DEFAULT 0, -- seconds
    score INTEGER DEFAULT 0,
    max_score INTEGER DEFAULT 0,
    attempts_count INTEGER DEFAULT 0,
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    last_accessed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(user_id, lesson_id)
);

-- User activity results
CREATE TABLE user_activity_results (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL,
    lesson_id UUID REFERENCES lessons(id) ON DELETE CASCADE,
    activity_id UUID REFERENCES lesson_activities(id) ON DELETE CASCADE,
    attempt_number INTEGER DEFAULT 1,
    user_answers JSONB, -- user's responses
    correct_answers JSONB, -- correct responses for reference
    score INTEGER DEFAULT 0,
    max_score INTEGER DEFAULT 0,
    time_taken INTEGER, -- seconds
    is_passed BOOLEAN DEFAULT FALSE,
    feedback TEXT,
    completed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Lesson series/courses
CREATE TABLE lesson_series (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title VARCHAR(255) NOT NULL,
    description TEXT,
    language_code VARCHAR(10) NOT NULL,
    target_language VARCHAR(10) NOT NULL,
    difficulty_level INTEGER CHECK (difficulty_level BETWEEN 1 AND 10),
    total_lessons INTEGER DEFAULT 0,
    estimated_duration INTEGER, -- total minutes
    thumbnail_url TEXT,
    is_published BOOLEAN DEFAULT FALSE,
    is_premium BOOLEAN DEFAULT FALSE,
    created_by UUID,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Lessons in series
CREATE TABLE series_lessons (
    series_id UUID REFERENCES lesson_series(id) ON DELETE CASCADE,
    lesson_id UUID REFERENCES lessons(id) ON DELETE CASCADE,
    position INTEGER NOT NULL,
    is_optional BOOLEAN DEFAULT FALSE,
    unlock_criteria JSONB, -- conditions to unlock this lesson
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (series_id, lesson_id),
    UNIQUE(series_id, position)
);

-- User series progress
CREATE TABLE user_series_progress (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL,
    series_id UUID REFERENCES lesson_series(id) ON DELETE CASCADE,
    status VARCHAR(20) DEFAULT 'not_started',
    progress_percent INTEGER DEFAULT 0,
    current_lesson_id UUID,
    completed_lessons UUID[] DEFAULT '{}',
    total_time_spent INTEGER DEFAULT 0,
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    last_accessed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(user_id, series_id)
);

-- Lesson reviews/ratings
CREATE TABLE lesson_reviews (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL,
    lesson_id UUID REFERENCES lessons(id) ON DELETE CASCADE,
    rating INTEGER CHECK (rating BETWEEN 1 AND 5),
    review_text TEXT,
    is_public BOOLEAN DEFAULT TRUE,
    helpful_count INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(user_id, lesson_id)
);

-- User bookmarks
CREATE TABLE user_lesson_bookmarks (
    user_id UUID NOT NULL,
    lesson_id UUID REFERENCES lessons(id) ON DELETE CASCADE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (user_id, lesson_id)
);

CREATE TABLE user_word_bookmarks (
    user_id UUID NOT NULL,
    word_id UUID NOT NULL, -- from content service
    lesson_context_id UUID REFERENCES lessons(id), -- optional: where they bookmarked it
    notes TEXT, -- user's personal notes
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (user_id, word_id)
);

-- Indexes
CREATE INDEX idx_lessons_language ON lessons(language_code);
CREATE INDEX idx_lessons_difficulty ON lessons(difficulty_level);
CREATE INDEX idx_lessons_published ON lessons(is_published);
CREATE INDEX idx_lessons_premium ON lessons(is_premium);
CREATE INDEX idx_lesson_paragraphs_lesson ON lesson_paragraphs(lesson_id);
CREATE INDEX idx_lesson_paragraphs_position ON lesson_paragraphs(lesson_id, position);
CREATE INDEX idx_user_lesson_progress_user ON user_lesson_progress(user_id);
CREATE INDEX idx_user_lesson_progress_status ON user_lesson_progress(status);
CREATE INDEX idx_user_activity_results_user_lesson ON user_activity_results(user_id, lesson_id);
CREATE INDEX idx_series_lessons_series ON series_lessons(series_id, position);
CREATE INDEX idx_lesson_reviews_lesson ON lesson_reviews(lesson_id);
```

## 4. Media Service - MongoDB Database

```javascript
// Collection: media_files - File metadata storage
{
  _id: ObjectId(),
  file_name: String, // "word_hello.mp3"
  original_name: String, // Original uploaded filename
  file_type: String, // "audio", "image", "video"
  mime_type: String, // "audio/mpeg", "image/png"
  file_size: Number, // bytes
  storage_provider: String, // "aws_s3", "cloudinary", "local"
  storage_path: String, // "/audio/words/hello.mp3"
  public_url: String, // Full accessible URL
  cdn_url: String, // CDN URL if available
  
  // Media-specific metadata
  metadata: {
    // Audio files
    duration: Number, // seconds
    bitrate: Number,
    sample_rate: Number,
    channels: Number, // mono=1, stereo=2
    
    // Image files
    width: Number,
    height: Number,
    format: String, // "JPEG", "PNG"
    color_space: String,
    has_alpha: Boolean,
    
    // Video files
    resolution: String, // "1920x1080"
    fps: Number,
    codec: String,
    
    // Processing status
    processing_status: String, // "pending", "completed", "failed"
    thumbnails: [
      {
        size: String, // "small", "medium", "large"
        url: String,
        width: Number,
        height: Number
      }
    ]
  },
  
  // Content references
  references: [
    {
      reference_type: String, // "word", "paragraph", "lesson", "user_avatar"
      reference_id: String, // UUID of the referenced entity
      usage_context: String, // "pronunciation", "illustration", "background"
      is_primary: Boolean // Main media for this reference
    }
  ],
  
  // Organization
  tags: [String], // ["pronunciation", "male_voice", "slow_speech"]
  category: String, // "word_audio", "lesson_image", "user_content"
  language_code: String, // if applicable
  
  // Access control
  visibility: String, // "public", "premium", "private"
  access_permissions: [String], // user IDs who can access
  
  // Upload info
  uploaded_by: String, // user UUID
  upload_session_id: String,
  ip_address: String,
  user_agent: String,
  
  // Timestamps
  created_at: Date,
  updated_at: Date,
  last_accessed_at: Date,
  
  // TTL for temporary files
  expires_at: Date, // null for permanent files
  
  // SEO and accessibility
  alt_text: String, // for images
  caption: String,
  transcript: String, // for audio/video
  
  // Quality and variants
  quality_variants: [
    {
      quality: String, // "low", "medium", "high"
      file_size: Number,
      url: String,
      bitrate: Number // for audio/video
    }
  ],
  
  // Download and usage stats
  download_count: Number,
  view_count: Number,
  
  // Content moderation
  moderation_status: String, // "pending", "approved", "rejected"
  moderation_flags: [String],
  moderated_at: Date,
  moderated_by: String
}

// Collection: media_processing_jobs
{
  _id: ObjectId(),
  media_file_id: ObjectId,
  job_type: String, // "thumbnail_generation", "audio_conversion", "transcription"
  status: String, // "pending", "processing", "completed", "failed"
  priority: Number, // 1-10, higher = more urgent
  
  input_params: {
    source_url: String,
    target_format: String,
    quality_settings: Object
  },
  
  output_results: {
    generated_files: [
      {
        variant_type: String,
        file_url: String,
        file_size: Number,
        metadata: Object
      }
    ],
    processing_time: Number, // milliseconds
    error_message: String
  },
  
  retry_count: Number,
  max_retries: Number,
  
  scheduled_at: Date,
  started_at: Date,
  completed_at: Date,
  created_at: Date
}

// Collection: media_usage_analytics
{
  _id: ObjectId(),
  media_file_id: ObjectId,
  user_id: String, // UUID
  
  event_type: String, // "view", "download", "play", "pause", "seek"
  event_data: {
    duration_watched: Number, // for videos
    seek_position: Number,
    volume_level: Number,
    playback_speed: Number,
    quality_selected: String
  },
  
  context: {
    reference_type: String, // where was this media accessed
    reference_id: String,
    page_url: String,
    user_agent: String,
    device_type: String, // "mobile", "tablet", "desktop"
    connection_type: String // "wifi", "cellular"
  },
  
  timestamp: Date,
  session_id: String,
  
  // Geographic data
  location: {
    country: String,
    city: String,
    timezone: String
  }
}

// Collection: media_cdn_cache
{
  _id: ObjectId(),
  original_media_id: ObjectId,
  cdn_provider: String, // "cloudflare", "aws_cloudfront"
  cdn_url: String,
  cache_key: String,
  
  cache_status: String, // "active", "purged", "expired"
  cache_regions: [String], // Geographic regions
  
  hit_count: Number,
  last_hit_at: Date,
  created_at: Date,
  expires_at: Date
}

// Indexes for MongoDB Media Service
db.media_files.createIndex({ "file_type": 1 })
db.media_files.createIndex({ "references.reference_type": 1, "references.reference_id": 1 })
db.media_files.createIndex({ "tags": 1 })
db.media_files.createIndex({ "uploaded_by": 1 })
db.media_files.createIndex({ "created_at": -1 })
db.media_files.createIndex({ "visibility": 1 })
db.media_files.createIndex({ "expires_at": 1 }, { expireAfterSeconds: 0 })

db.media_processing_jobs.createIndex({ "status": 1, "priority": -1 })
db.media_processing_jobs.createIndex({ "media_file_id": 1 })
db.media_processing_jobs.createIndex({ "scheduled_at": 1 })

db.media_usage_analytics.createIndex({ "media_file_id": 1, "timestamp": -1 })
db.media_usage_analytics.createIndex({ "user_id": 1, "timestamp": -1 })
db.media_usage_analytics.createIndex({ "timestamp": 1 }, { expireAfterSeconds: 7776000 }) // 90 days TTL
```

## 5. Analytics Service - MongoDB Database

```javascript
// Collection: user_interactions - Raw interaction events
{
  _id: ObjectId(),
  user_id: String, // UUID
  session_id: String, // To group related actions
  
  event_type: String, // "page_view", "word_click", "lesson_start", "lesson_complete", etc.
  event_category: String, // "learning", "navigation", "social", "system"
  
  content_info: {
    content_type: String, // "word", "paragraph", "lesson", "exercise"
    content_id: String, // UUID of the content
    content_title: String, // For easy querying/reporting
    language_code: String,
    difficulty_level: Number
  },
  
  interaction_data: {
    duration: Number, // How long user spent on this action (seconds)
    completion_rate: Number, // 0-1, how much of content was consumed
    score: Number, // For exercises/quizzes
    attempts: Number, // Number of tries
    
    // Specific to different content types
    word_lookup_count: Number, // For reading exercises
    audio_plays: Number,
    hint_usage: Number,
    
    // User behavior patterns
    scroll_depth: Number, // How far user scrolled (0-1)
    time_to_first_interaction: Number, // milliseconds
    idle_time: Number, // seconds of inactivity
    
    // Performance data
    loading_time: Number, // milliseconds
    error_count: Number,
    error_types: [String]
  },
  
  user_context: {
    device_type: String, // "mobile", "tablet", "desktop"
    browser: String,
    os: String,
    screen_resolution: String,
    connection_type: String, // "wifi", "cellular", "ethernet"
    is_returning_user: Boolean,
    days_since_registration: Number,
    current_streak: Number
  },
  
  location_data: {
    country_code: String,
    city: String,
    timezone: String,
    local_time: Date // User's local time when event occurred
  },
  
  timestamp: Date,
  server_timestamp: Date, // In case of time sync issues
  
  // Referrer information
  referrer: {
    source: String, // "direct", "google", "facebook", "email"
    medium: String, // "organic", "cpc", "email", "social"
    campaign: String, // Marketing campaign ID
    previous_page: String
  },
  
  // A/B testing
  experiments: [
    {
      experiment_name: String,
      variant: String,
      started_at: Date
    }
  ],
  
  // Data quality
  is_bot: Boolean,
  confidence_score: Number, // 0-1, how confident we are this is real user
  
  // For data retention
  expires_at: Date // TTL index, keep raw events for limited time
}

// Collection: daily_user_metrics - Aggregated daily statistics
{
  _id: ObjectId(),
  user_id: String,
  date: Date, // YYYY-MM-DD format for easy querying
  
  learning_metrics: {
    total_study_time: Number, // minutes
    lessons_started: Number,
    lessons_completed: Number,
    paragraphs_read: Number,
    words_encountered: Number,
    words_learned: Number, // Successfully completed exercises
    exercises_attempted: Number,
    exercises_completed: Number,
    
    avg_lesson_score: Number,
    avg_completion_rate: Number,
    
    // Streak and consistency
    maintained_streak: Boolean,
    study_sessions: Number, // How many separate study periods
    longest_session_minutes: Number
  },
  
  engagement_metrics: {
    page_views: Number,
    unique_content_pieces: Number, // Distinct lessons/words accessed
    social_interactions: Number, // Likes, shares, comments
    
    time_in_app: Number, // Total minutes in application
    bounce_rate: Number, // Sessions with only 1 page view
    return_visits: Number, // Times user came back same day
    
    feature_usage: {
      audio_plays: Number,
      word_bookmarks: Number,
      dictionary_lookups: Number,
      help_requests: Number
    }
  },
  
  performance_metrics: {
    avg_response_time: Number, // UI responsiveness
    error_count: Number,
    crash_count: Number,
    slow_queries: Number
  },
  
  created_at: Date,
  computed_at: Date // When this aggregation was calculated
}

// Collection: content_performance - How content pieces perform
{
  _id: ObjectId(),
  content_type: String, // "lesson", "paragraph", "word"
  content_id: String,
  date: Date,
  
  usage_stats: {
    unique_users: Number,
    total_views: Number,
    total_completions: Number,
    avg_time_spent: Number,
    avg_completion_rate: Number,
    
    bounce_rate: Number, // Users who left immediately
    return_rate: Number, // Users who came back to this content
    
    difficulty_feedback: {
      too_easy: Number,
      just_right: Number,
      too_hard: Number
    }
  },
  
  learning_outcomes: {
    avg_score: Number,
    pass_rate: Number,
    avg_attempts: Number,
    improvement_rate: Number, // How much users improve on retry
    
    common_mistakes: [
      {
        mistake_type: String,
        frequency: Number,
        user_count: Number
      }
    ]
  },
  
  user_feedback: {
    avg_rating: Number,
    rating_count: Number,
    positive_feedback_rate: Number,
    
    sentiment_scores: {
      positive: Number,
      neutral: Number,
      negative: Number
    }
  },
  
  technical_metrics: {
    avg_load_time: Number,
    error_rate: Number,
    mobile_usage_rate: Number
  },
  
  created_at: Date,
  expires_at: Date // Keep aggregated data longer than raw events
}

// Collection: learning_analytics - Advanced learning insights
{
  _id: ObjectId(),
  user_id: String,
  analysis_type: String, // "weekly_report", "monthly_report", "learning_path_analysis"
  period: {
    start_date: Date,
    end_date: Date
  },
  
  learning_patterns: {
    optimal_study_times: [
      {
        hour: Number, // 0-23
        day_of_week: Number, // 0-6
        performance_score: Number,
        frequency: Number
      }
    ],
    
    learning_velocity: {
      words_per_hour: Number,
      concepts_mastered_per_session: Number,
      difficulty_progression_rate: Number
    },
    
    retention_analysis: {
      short_term_retention: Number, // 0-1, within 24 hours
      medium_term_retention: Number, // within 1 week
      long_term_retention: Number, // within 1 month
      
      forgetting_curve_data: [
        {
          days_after_learning: Number,
          retention_rate: Number
        }
      ]
    },
    
    learning_style_indicators: {
      visual_preference: Number, // 0-1 score
      auditory_preference: Number,
      kinesthetic_preference: Number,
      reading_preference: Number,
      
      prefers_repetition: Boolean,
      prefers_variety: Boolean,
      responds_to_gamification: Boolean
    }
  },
  
  performance_trends: {
    accuracy_trend: [
      {
        date: Date,
        accuracy_rate: Number,
        confidence_interval: Number
      }
    ],
    
    speed_improvement: {
      initial_avg_time: Number,
      current_avg_time: Number,
      improvement_rate: Number
    },
    
    difficulty_adaptation: {
      comfort_level: Number, // 1-10
      challenge_preference: Number,
      recommended_difficulty: Number
    }
  },
  
  recommendations: [
    {
      type: String, // "content", "schedule", "method", "review"
      priority: Number, // 1-10
      title: String,
      description: String,
      suggested_actions: [String],
      expected_impact: String,
      confidence_score: Number
    }
  ],
  
  generated_at: Date,
  model_version: String, // AI model version used for analysis
  expires_at: Date
}

// Collection: system_analytics - Application performance and usage
{
  _id: ObjectId(),
  date: Date,
  metric_type: String, // "daily_summary", "hourly_peak", "feature_usage"
  
  user_metrics: {
    total_active_users: Number,
    new_registrations: Number,
    returning_users: Number,
    churned_users: Number, // Users who haven't been active recently
    
    user_segments: {
      beginners: Number, // Users with < 10 lessons
      intermediate: Number, // 10-50 lessons
      advanced: Number, // 50+ lessons
      premium_users: Number,
      free_users: Number
    },
    
    geographic_distribution: [
      {
        country_code: String,
        user_count: Number,
        active_percentage: Number
      }
    ]
  },
  
  content_metrics: {
    total_lessons_completed: Number,
    total_words_learned: Number,
    most_popular_content: [
      {
        content_type: String,
        content_id: String,
        usage_count: Number,
        title: String
      }
    ],
    
    content_effectiveness: {
      avg_completion_rate: Number,
      avg_user_rating: Number,
      problematic_content: [
        {
          content_id: String,
          issue_type: String,
          severity: String
        }
      ]
    }
  },
  
  technical_metrics: {
    avg_response_time: Number,
    error_rate: Number,
    uptime_percentage: Number,
    
    api_usage: [
      {
        endpoint: String,
        request_count: Number,
        avg_response_time: Number,
        error_count: Number
      }
    ],
    
    database_performance: {
      query_count: Number,
      slow_queries: Number,
      cache_hit_rate: Number
    },
    
    resource_usage: {
      cpu_avg: Number,
      memory_avg: Number,
      storage_used: Number,
      bandwidth_used: Number
    }
  },
  
  business_metrics: {
    revenue: Number,
    conversion_rate: Number, // Free to premium
    customer_acquisition_cost: Number,
    lifetime_value: Number,
    
    feature_adoption: [
      {
        feature_name: String,
        adoption_rate: Number,
        user_satisfaction: Number
      }
    ]
  },
  
  created_at: Date
}

// Collection: ab_test_results - A/B testing analytics
{
  _id: ObjectId(),
  experiment_name: String,
  variant_name: String,
  
  experiment_config: {
    start_date: Date,
    end_date: Date,
    target_metric: String,
    hypothesis: String,
    confidence_level: Number // 0.95, 0.99, etc.
  },
  
  participant_data: {
    total_users: Number,
    active_users: Number,
    user_segments: Object, // Breakdown by user characteristics
    
    sample_size: Number,
    statistical_power: Number,
    is_significant: Boolean
  },
  
  results: {
    primary_metric: {
      variant_value: Number,
      control_value: Number,
      lift: Number, // Percentage improvement
      p_value: Number,
      confidence_interval: {
        lower: Number,
        upper: Number
      }
    },
    
    secondary_metrics: [
      {
        metric_name: String,
        variant_value: Number,
        control_value: Number,
        lift: Number,
        significance: String // "significant", "not_significant", "trending"
      }
    ],
    
    user_segments_analysis: [
      {
        segment_name: String,
        segment_size: Number,
        lift: Number,
        significance: String
      }
    ]
  },
  
  recommendations: {
    decision: String, // "implement", "reject", "continue_testing"
    reasoning: String,
    expected_impact: String,
    rollout_plan: String
  },
  
  analyzed_at: Date,
  analyst: String // Who performed the analysis
}

// Indexes for Analytics Service
db.user_interactions.createIndex({ "user_id": 1, "timestamp": -1 })
db.user_interactions.createIndex({ "event_type": 1, "timestamp": -1 })
db.user_interactions.createIndex({ "content_info.content_id": 1, "timestamp": -1 })
db.user_interactions.createIndex({ "session_id": 1 })
db.user_interactions.createIndex({ "timestamp": 1 }, { expireAfterSeconds: 7776000 }) // 90 days TTL

db.daily_user_metrics.createIndex({ "user_id": 1, "date": -1 })
db.daily_user_metrics.createIndex({ "date": -1 })

db.content_performance.createIndex({ "content_id": 1, "date": -1 })
db.content_performance.createIndex({ "content_type": 1, "date": -1 })

db.learning_analytics.createIndex({ "user_id": 1, "generated_at": -1 })
db.learning_analytics.createIndex({ "analysis_type": 1 })

db.system_analytics.createIndex({ "date": -1, "metric_type": 1 })

db.ab_test_results.createIndex({ "experiment_name": 1, "variant_name": 1 })
```

## 6. Notification Service - MongoDB Database

```javascript
// Collection: notification_templates - Reusable notification templates
{
  _id: ObjectId(),
  template_code: String, // "lesson_reminder", "streak_milestone", "new_word_daily"
  template_name: String,
  description: String,
  
  // Multi-language support
  localizations: {
    "vi": {
      title_template: String, // "Chúc mừng {{username}}! Bạn đã hoàn thành bài học!"
      body_template: String,
      action_text: String, // "Tiếp tục học"
      rich_content: Object // HTML, images, etc.
    },
    "en": {
      title_template: String, // "Congratulations {{username}}! You completed the lesson!"
      body_template: String,
      action_text: String, // "Continue Learning"
      rich_content: Object
    }
    // Add more languages as needed
  },
  
  // Template variables/placeholders
  variables: [
    {
      name: String, // "username", "lesson_title", "streak_count"
      type: String, // "string", "number", "date", "url"
      required: Boolean,
      default_value: String,
      description: String
    }
  ],
  
  // Delivery configuration
  delivery_settings: {
    channels: [String], // ["push", "email", "in_app", "sms"]
    priority: String, // "low", "normal", "high", "urgent"
    
    // Timing rules
    send_immediately: Boolean,
    delay_minutes: Number, // Optional delay before sending
    optimal_time_delivery: Boolean, // Send at user's optimal time
    
    // Frequency limits
    max_per_day: Number,
    max_per_week: Number,
    cooldown_hours: Number, // Minimum time between similar notifications
    
    // Personalization
    personalization_level: String, // "basic", "advanced", "ai_powered"
    dynamic_content: Boolean // Use AI to customize content
  },
  
  // Targeting criteria
  target_criteria: {
    user_segments: [String], // "beginners", "premium", "inactive_users"
    user_properties: Object, // Flexible criteria matching
    behavioral_triggers: [
      {
        event_type: String,
        conditions: Object,
        time_window: String // "immediate", "1_hour", "1_day"
      }
    ],
    
    // Geographic and temporal targeting
    timezones: [String],
    countries: [String],
    send_days: [Number], // 0-6, days of week
    send_hours: {
      start: Number, // 0-23
      end: Number
    }
  },
  
  // A/B testing
  variants: [
    {
      variant_name: String,
      traffic_percentage: Number, // 0-100
      title_override: Object, // Override titles for this variant
      body_override: Object,
      is_active: Boolean
    }
  ],
  
  // Template status and metadata
  status: String, // "draft", "active", "paused", "archived"
  category: String, // "learning", "engagement", "system", "marketing"
  tags: [String],
  
  // Performance tracking
  stats: {
    total_sent: Number,
    total_delivered: Number,
    total_opened: Number,
    total_clicked: Number,
    unsubscribe_count: Number,
    
    avg_open_rate: Number,
    avg_click_rate: Number,
    avg_conversion_rate: Number
  },
  
  created_by: String, // User ID
  created_at: Date,
  updated_at: Date,
  last_used_at: Date
}

// Collection: notification_queue - Scheduled notifications
{
  _id: ObjectId(),
  user_id: String,
  template_code: String,
  
  // Notification content (resolved from template)
  content: {
    title: String,
    body: String,
    action_text: String,
    action_url: String,
    image_url: String,
    rich_content: Object,
    
    // Deep linking
    deep_link: String,
    web_url: String,
    
    // Customization data
    personalization_data: Object,
    template_variables: Object
  },
  
  // Delivery configuration
  delivery: {
    channels: [String], // Which channels to send through
    priority: String,
    
    // Scheduling
    scheduled_at: Date,
    send_after: Date, // Don't send before this time
    expires_at: Date, // Don't send after this time
    
    // User preferences override
    respect_quiet_hours: Boolean,
    respect_frequency_limits: Boolean,
    
    // Retry configuration
    max_retries: Number,
    retry_delay: Number, // Minutes between retries
    current_retry: Number
  },
  
  // Processing status
  status: String, // "scheduled", "processing", "sent", "delivered", "failed", "cancelled"
  processing_history: [
    {
      status: String,
      timestamp: Date,
      channel: String,
      error_message: String,
      external_id: String, // Provider's message ID
      metadata: Object
    }
  ],
  
  // User targeting verification
  user_context: {
    user_timezone: String,
    user_language: String,
    user_segments: [String],
    subscription_status: String,
    notification_preferences: Object,
    
    // Context when notification was created
    trigger_event: String,
    trigger_context: Object,
    session_id: String
  },
  
  // A/B testing
  experiment_data: {
    experiment_name: String,
    variant_name: String,
    control_group: Boolean
  },
  
  // Tracking
  tracking: {
    sent_at: Date,
    delivered_at: Date,
    opened_at: Date,
    clicked_at: Date,
    
    // Engagement metrics
    time_to_open: Number, // Minutes from sent to opened
    time_to_click: Number,
    
    // Technical details
    user_agent: String,
    device_type: String,
    platform: String
  },
  
  created_at: Date,
  updated_at: Date
}

// Collection: user_notification_preferences - User's notification settings
{
  _id: ObjectId(),
  user_id: String, // Unique per user
  
  // Global preferences
  global_settings: {
    notifications_enabled: Boolean,
    email_notifications: Boolean,
    push_notifications: Boolean,
    sms_notifications: Boolean,
    in_app_notifications: Boolean,
    
    // Timing preferences
    quiet_hours: {
      enabled: Boolean,
      start_time: String, // "22:00"
      end_time: String, // "08:00"
      timezone: String
    },
    
    // Frequency limits
    max_notifications_per_day: Number,
    digest_mode: Boolean, // Batch notifications into digest
    digest_frequency: String // "daily", "weekly"
  },
  
  // Category-specific preferences
  category_preferences: {
    learning_reminders: {
      enabled: Boolean,
      channels: [String],
      frequency: String // "daily", "every_other_day", "weekly"
    },
    
    achievement_notifications: {
      enabled: Boolean,
      channels: [String],
      milestone_only: Boolean // Only major achievements
    },
    
    social_notifications: {
      enabled: Boolean,
      channels: [String],
      friend_activities: Boolean,
      comments_replies: Boolean
    },
    
    system_notifications: {
      enabled: Boolean,
      channels: [String],
      maintenance_alerts: Boolean,
      security_alerts: Boolean
    },
    
    marketing_notifications: {
      enabled: Boolean,
      channels: [String],
      promotional_offers: Boolean,
      product_updates: Boolean,
      newsletter: Boolean
    }
  },
  
  // Content preferences
  content_preferences: {
    language: String, // Preferred language for notifications
    tone: String, // "casual", "formal", "encouraging"
    personalization_level: String, // "minimal", "moderate", "high"
    
    // AI-powered customization
    learning_focus_areas: [String],
    motivation_style: String, // "achievement", "progress", "social"
    content_difficulty: String // "challenging", "supportive", "adaptive"
  },
  
  // Subscription management
  subscriptions: [
    {
      template_code: String,
      subscribed: Boolean,
      subscribed_at: Date,
      unsubscribed_at: Date,
      unsubscribe_reason: String
    }
  ],
  
  // Delivery tracking
  delivery_stats: {
    total_sent: Number,
    total_delivered: Number,
    total_opened: Number,
    total_clicked: Number,
    
    last_notification_at: Date,
    last_opened_at: Date,
    last_clicked_at: Date,
    
    // Engagement patterns
    best_open_times: [
      {
        hour: Number,
        day_of_week: Number,
        open_rate: Number
      }
    ]
  },
  
  updated_at: Date,
  created_at: Date
}

// Collection: notification_campaigns - Marketing and bulk notifications
{
  _id: ObjectId(),
  campaign_name: String,
  campaign_type: String, // "marketing", "announcement", "educational_series"
  
  // Campaign configuration
  config: {
    template_code: String,
    target_audience: {
      user_segments: [String],
      user_filters: Object,
      exclude_filters: Object,
      estimated_reach: Number
    },
    
    // Delivery schedule
    send_strategy: String, // "immediate", "scheduled", "drip", "trigger_based"
    scheduled_at: Date,
    
    // Drip campaign settings
    drip_settings: {
      interval_days: Number,
      max_messages: Number,
      stop_on_goal: Boolean // Stop when user completes goal
    },
    
    // Performance goals
    success_metrics: {
      target_open_rate: Number,
      target_click_rate: Number,
      target_conversion_rate: Number,
      conversion_goal: String // What action constitutes success
    }
  },
  
  // A/B testing for campaigns
  variants: [
    {
      variant_name: String,
      traffic_split: Number, // Percentage of audience
      template_overrides: Object,
      send_time_offset: Number // Hours from base send time
    }
  ],
  
  // Campaign status and results
  status: String, // "draft", "scheduled", "running", "paused", "completed", "cancelled"
  
  results: {
    total_targeted: Number,
    total_sent: Number,
    total_delivered: Number,
    total_opened: Number,
    total_clicked: Number,
    total_converted: Number,
    total_unsubscribed: Number,
    
    // Calculated rates
    delivery_rate: Number,
    open_rate: Number,
    click_rate: Number,
    conversion_rate: Number,
    unsubscribe_rate: Number,
    
    // Time-based breakdown
    performance_by_hour: [
      {
        hour: Number,
        sent: Number,
        opened: Number,
        clicked: Number
      }
    ],
    
    // Segment performance
    segment_performance: [
      {
        segment_name: String,
        user_count: Number,
        open_rate: Number,
        click_rate: Number,
        conversion_rate: Number
      }
    ]
  },
  
  created_by: String,
  created_at: Date,
  updated_at: Date,
  started_at: Date,
  completed_at: Date
}

// Collection: notification_analytics - Detailed analytics
{
  _id: ObjectId(),
  date: Date,
  metric_type: String, // "daily_summary", "template_performance", "user_engagement"
  
  // Overall notification metrics
  global_metrics: {
    total_notifications_sent: Number,
    total_notifications_delivered: Number,
    total_notifications_opened: Number,
    total_notifications_clicked: Number,
    
    avg_delivery_rate: Number,
    avg_open_rate: Number,
    avg_click_rate: Number,
    avg_time_to_open: Number, // Minutes
    
    // Channel performance
    channel_breakdown: {
      email: { sent: Number, delivered: Number, opened: Number, clicked: Number },
      push: { sent: Number, delivered: Number, opened: Number, clicked: Number },
      in_app: { sent: Number, delivered: Number, opened: Number, clicked: Number },
      sms: { sent: Number, delivered: Number, opened: Number, clicked: Number }
    }
  },
  
  // Template-specific performance
  template_performance: [
    {
      template_code: String,
      template_name: String,
      sent_count: Number,
      delivery_rate: Number,
      open_rate: Number,
      click_rate: Number,
      conversion_rate: Number,
      unsubscribe_rate: Number,
      
      // Engagement patterns
      best_send_times: [
        {
          hour: Number,
          day_of_week: Number,
          performance_score: Number
        }
      ],
      
      // User feedback
      user_ratings: {
        helpful: Number,
        not_helpful: Number,
        spam_reports: Number
      }
    }
  ],
  
  // User engagement insights
  user_engagement: {
    highly_engaged_users: Number, // Users with >80% open rate
    moderately_engaged_users: Number, // 40-80% open rate
    low_engaged_users: Number, // <40% open rate
    unengaged_users: Number, // 0% open rate
    
    // Churn risk indicators
    declining_engagement_users: Number,
    at_risk_users: Number, // Haven't opened notifications in X days
    
    // Engagement trends
    engagement_trend: String, // "improving", "declining", "stable"
    trend_change_percentage: Number
  },
  
  // Technical metrics
  technical_metrics: {
    avg_processing_time: Number, // Milliseconds
    failed_deliveries: Number,
    retry_count: Number,
    
    // Provider performance
    provider_stats: [
      {
        provider_name: String, // "sendgrid", "firebase", "twilio"
        delivery_rate: Number,
        avg_delivery_time: Number,
        error_rate: Number,
        cost_per_notification: Number
      }
    ]
  },
  
  created_at: Date
}

// Indexes for Notification Service
db.notification_templates.createIndex({ "template_code": 1 })
db.notification_templates.createIndex({ "status": 1 })
db.notification_templates.createIndex({ "category": 1 })

db.notification_queue.createIndex({ "user_id": 1, "scheduled_at": 1 })
db.notification_queue.createIndex({ "status": 1, "scheduled_at": 1 })
db.notification_queue.createIndex({ "template_code": 1 })
db.notification_queue.createIndex({ "created_at": 1 }, { expireAfterSeconds: 2592000 }) // 30 days TTL

db.user_notification_preferences.createIndex({ "user_id": 1 })

db.notification_campaigns.createIndex({ "status": 1 })
db.notification_campaigns.createIndex({ "created_by": 1 })
db.notification_campaigns.createIndex({ "scheduled_at": 1 })

db.notification_analytics.createIndex({ "date": -1, "metric_type": 1 })
```

## 7. Cross-Service Data Consistency & Event Streaming

### Event Schema for Inter-Service Communication:
```javascript
// Standard event structure
{
  event_id: String, // UUID
  event_type: String, // "user.created", "lesson.completed", "word.bookmarked"
  service_name: String, // Source service
  version: String, // Event schema version
  
  timestamp: Date,
  correlation_id: String, // For tracing related events
  
  // Event payload
  data: {
    // Specific to event type
    user_id: String,
    entity_id: String,
    entity_type: String,
    
    // Before/after states for update events
    previous_state: Object,
    current_state: Object,
    
    // Metadata
    triggered_by: String, // user_action, system, scheduled_task
    source_ip: String,
    user_agent: String
  },
  
  // Event metadata
  metadata: {
    retry_count: Number,
    max_retries: Number,
    dead_letter: Boolean,
    
    // Processing info
    processed_by: [String], // Services that processed this event
    failed_services: [String],
    
    // Routing info
    target_services: [String], // Which services should process this
    routing_key: String
  }
}
```

## 8. Database Migration Scripts & Deployment Strategy

### PostgreSQL Migration Example:
```sql
-- Migration: 001_initial_schema.sql
BEGIN;

-- Enable UUID generation
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";

-- User Service Tables
CREATE TABLE users (
    -- Full schema as defined above
);

-- Add triggers for updated_at
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$ language 'plpgsql';

CREATE TRIGGER update_users_updated_at 
    BEFORE UPDATE ON users 
    FOR EACH ROW 
    EXECUTE FUNCTION update_updated_at_column();

COMMIT;
```

### MongoDB Migration Example:
```javascript
// Migration: 001_setup_indexes.js
db.media_files.createIndex({ "file_type": 1 });
db.media_files.createIndex({ "references.reference_type": 1, "references.reference_id": 1 });
db.media_files.createIndex({ "tags": 1 });
db.media_files.createIndex({ "uploaded_by": 1 });
db.media_files.createIndex({ "created_at": -1 });
db.media_files.createIndex({ "expires_at": 1 }, { expireAfterSeconds: 0 });

// Set up TTL indexes for cleanup
db.user_interactions.createIndex(
  { "timestamp": 1 }, 
  { expireAfterSeconds: 7776000 } // 90 days
);
```

## 9. Performance Optimization Recommendations

### Connection Pooling Configuration:
```yaml
# PostgreSQL (per service)
max_connections: 20
min_connections: 5
connection_timeout: 30s
idle_timeout: 600s
max_lifetime: 1800s

# MongoDB
max_pool_size: 100
min_pool_size: 10
max_idle_time: 600s
```

### Caching Strategy:
```yaml
Redis_Layers:
  L1_Application_Cache:
    - User sessions (TTL: 30min)
    - User preferences (TTL: 1hour)
    - Content metadata (TTL: 6hours)
    
  L2_Database_Query_Cache:
    - Frequent word lookups (TTL: 24hours)
    - Popular lesson content (TTL: 12hours)
    - User analytics summaries (TTL: 1hour)
    
  L3_CDN_Cache:
    - Media files (TTL: 7days)
    - Static content (TTL: 30days)
```

## 10. Monitoring & Alerting Setup

### Key Metrics to Monitor:
```yaml
Database_Health:
  PostgreSQL:
    - Connection pool utilization
    - Query response times (95th percentile)
    - Slow query count
    - Dead locks count
    - Buffer hit ratio
    
  MongoDB:
    - Operation latency
    - Queue depth
    - Replication lag
    - Index hit ratio
    - Working set size

Application_Metrics:
  - Cross-service API latency
  - Event processing lag
  - Cache hit rates
  - Error rates by service
  - User engagement metrics
```

Đây là kiến trúc database hoàn chỉnh cho dự án của bạn. Mỗi service có database phù hợp với workload và yêu cầu riêng, đảm bảo scalability và performance tốt nhất.
