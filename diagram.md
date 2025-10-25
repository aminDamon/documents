# MetaLearn Prompt Processing Flow Diagram

## Complete Flow: از ثبت پرامپت تا تولید خروجی Markdown

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                FRONTEND (React)                                │
└─────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        │ 1. User submits prompt
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              GO BACKEND (Gin)                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        │ 2. POST /prompts
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           PromptHandler.CreatePrompt()                         │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ Step 1: Create prompt in database (always insert first)                │   │
│  │ - Generate RandID (UUID v7)                                            │   │
│  │ - Set status = "pending"                                               │   │
│  │ - Store prompt_text, user info, etc.                                   │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                        │                                       │
│                                        │ 3. Check if course exists            │
│                                        ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ Step 1.5: Create/Update Course                                        │   │
│  │ - Generate course title from prompt text                               │   │
│  │ - Create course record with prompt_rand_id                             │   │
│  │ - Handle chat_rand_id (create new chat if needed)                      │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                        │                                       │
│                                        │ 4. Check IsGenerateAI flag           │
│                                        ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ Step 2: Caching Check                                                  │   │
│  │ - Find similar prompts using similarity threshold (default 0.3)        │   │
│  │ - If similar prompt found:                                             │   │
│  │   * Use cached topics from similar prompt                              │   │
│  │   * Update status to "completed_cache"                                 │   │
│  │   * Add topics to course                                               │   │
│  │   * Send WebSocket updates (cached result)                             │   │
│  │   * Return response immediately                                        │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                        │                                       │
│                                        │ 5. No cache found - AI processing    │
│                                        ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ Step 3: Start AI Processing (Background Goroutine)                     │   │
│  │ - Update status to "processing"                                        │   │
│  │ - Call aiService.SendPromptWithWebSocket()                             │   │
│  │ - Forward WebSocket updates to frontend                                │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                        │                                       │
│                                        │ 6. HTTP Request to FastAPI           │
│                                        ▼                                       │
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              FASTAPI AI SERVICE                                │
└─────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        │ 7. POST /api/v1/receive-prompt
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        receive_prompt_from_go()                                │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ Step 1: WebSocket Connection Setup                                    │   │
│  │ - Connect to /ws/{prompt_rand_id}                                      │   │
│  │ - Send "prompt_received" update                                        │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                        │                                       │
│                                        │ 8. AI Processing                     │
│                                        ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ Step 2: Send to Liara AI                                             │   │
│  │ - Build structured prompt template                                    │   │
│  │ - Send to Liara AI (OpenAI-compatible API)                           │   │
│  │ - Send "ai_processing" update                                         │   │
│  │ - Get YAML response with course structure                             │   │
│  │ - Send "ai_completed" update                                          │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                        │                                       │
│                                        │ 9. Parse YAML Response               │
│                                        ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ Step 3: Parse and Send Topics to Go Backend                          │   │
│  │ - Parse YAML response (remove markdown blocks)                        │   │
│  │ - Extract course_category and topics                                  │   │
│  │ - Send topics to Go backend via POST /topic-courses/bulk              │   │
│  │ - Send "topics_saved" update                                          │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                        │                                       │
│                                        │ 10. Save AI Response                 │
│                                        ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ Step 4: Save Response to File                                        │   │
│  │ - Save to ai_responses/response_{timestamp}.txt                       │   │
│  │ - Include user prompt, full prompt, and AI response                   │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                        │                                       │
│                                        │ 11. Return Response                  │
│                                        ▼                                       │
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              GO BACKEND (Gin)                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        │ 12. Receive Topics from FastAPI
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    TopicCourseService.BulkCreate()                             │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ Step 1: Create Topic Records                                          │   │
│  │ - Create topic_courses table entries for each topic                   │   │
│  │ - Store topic_title, keywords, search_terms, category_name, etc.      │   │
│  │ - Generate RandID for each topic                                      │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                        │                                       │
│                                        │ 13. Update Course                    │
│                                        ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ Step 2: Update Course with Topics                                     │   │
│  │ - Add topic relationships to course                                   │   │
│  │ - Update course in database                                           │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                        │                                       │
│                                        │ 14. Update Prompt                    │
│                                        ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ Step 3: Update Prompt Status                                          │   │
│  │ - Update topic_rand_ids in prompt                                     │   │
│  │ - Update status to "completed"                                        │   │
│  │ - Set is_generate_ai = true                                           │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                        │                                       │
│                                        │ 15. WebSocket Updates                │
│                                        ▼                                       │
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                FRONTEND (React)                                │
└─────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        │ 16. Real-time Updates via WebSocket
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              MARKDOWN GENERATION                               │
└─────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        │ 17. For each topic (separate process)
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              FASTAPI AI SERVICE                                │
└─────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        │ 18. POST /api/v1/collect-and-generate-markdown
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    collect_and_generate_markdown()                             │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ Step 1: Collect Topic Data                                            │   │
│  │ - Use TopicDataManager to collect data from external sources          │   │
│  │ - Save collected data to data_topic/ folder                           │   │
│  │ - Return file path and collected sources                              │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                        │                                       │
│                                        │ 19. Generate Markdown                │
│                                        ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ Step 2: AI Markdown Generation                                       │   │
│  │ - Read prompt template from md_ai_prompt.txt                         │   │
│  │ - Format prompt with collected data and topic info                   │   │
│  │ - Send to Liara AI for markdown generation                           │   │
│  │ - Save markdown to md_ai/markdown_{topic_rand_id}_{timestamp}.md     │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                        │                                       │
│                                        │ 20. Return Markdown Content           │
│                                        ▼                                       │
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                FRONTEND (React)                                │
└─────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        │ 21. Display Generated Content
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              FINAL RESULT                                      │
│  • Course with structured topics                                              │
│  • Each topic has generated markdown content                                  │
│  • Real-time progress updates via WebSocket                                   │
│  • Caching for similar prompts                                                │
└─────────────────────────────────────────────────────────────────────────────────┘

## Key Components:

### Go Backend (Gin):
- **PromptHandler**: Main handler for prompt creation and processing
- **AIService**: Communication with FastAPI AI service
- **WebSocketClient**: Real-time updates to frontend
- **TopicCourseService**: Management of course topics
- **Caching System**: Similarity-based prompt caching

### FastAPI AI Service:
- **Main App**: FastAPI application with multiple endpoints
- **AI Client**: Communication with Liara AI (OpenAI-compatible)
- **TopicDataManager**: Data collection from external sources
- **WebSocket Manager**: Real-time status updates
- **Markdown Generation**: AI-powered content creation

### Data Flow:
1. **Prompt Registration** → Database storage
2. **Caching Check** → Similar prompt lookup
3. **AI Processing** → Liara AI for course structure
4. **Topic Creation** → Database storage of topics
5. **Data Collection** → External sources (Wikipedia, etc.)
6. **Markdown Generation** → AI-powered content creation
7. **Real-time Updates** → WebSocket communication

### Caching Strategy:
- Similarity threshold: 0.3 (configurable)
- Cached results: topics from similar prompts
- Status: "completed_cache" for cached results
- Performance: Immediate response for cached prompts

### Error Handling:
- WebSocket connection failures (non-critical)
- AI service errors
- Database transaction failures
- YAML parsing errors
- File I/O errors

### File Storage:
- **AI Responses**: `ai_responses/response_{timestamp}.txt`
- **Topic Data**: `data_topic/topic_{id}_{rand_id}_{timestamp}.txt`
- **Markdown Files**: `md_ai/markdown_{topic_rand_id}_{timestamp}.md`
