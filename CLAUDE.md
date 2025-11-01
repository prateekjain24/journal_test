# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Health Clarity** is a personal health companion that reveals patterns in wellbeing through effortless journaling. Users speak or type naturally about how they feel, and AI automatically detects symptom-trigger correlations and predictive warnings they'd never notice on their own.

### Core Value Proposition
- Passive wisdom from active living
- 30-second daily entries (voice or text)
- AI-powered pattern detection after 10+ entries
- Privacy-first with Supabase Row Level Security

## Tech Stack

- **Framework:** Next.js 16.0.1 (App Router with src/ directory)
- **React:** 19.2.0
- **TypeScript:** v5 (strict mode enabled)
- **Styling:** Tailwind CSS v4 (using @import syntax)
- **Database:** Supabase (PostgreSQL + Auth + Real-time)
- **AI Services:**
  - Google Gemini 2.5 Flash - Entity extraction and pattern insights
  - Groq Whisper (whisper-large-v3-turbo) - Audio transcription
- **UI Components:** Shadcn UI (copy-paste components, owned by project)
- **Package Manager:** **bun** (preferred over npm/yarn/pnpm)

## Development Commands

```bash
# Development server (with hot reload)
bun dev

# Production build
bun run build

# Start production server
bun start

# Run linter
bun run lint

# Install dependencies
bun install

# Add Shadcn UI components
npx shadcn-ui@latest add [component-name]
```

## Project Structure

```
my-app/
├── src/
│   └── app/
│       ├── layout.tsx           # Root layout with Geist fonts
│       ├── page.tsx             # Home page
│       ├── globals.css          # Tailwind CSS v4 imports
│       ├── (auth)/              # Auth routes (grouped, no layout nav)
│       │   ├── layout.tsx      # Centered auth layout
│       │   ├── login/
│       │   └── signup/
│       ├── (dashboard)/         # Dashboard routes (grouped, with nav)
│       │   ├── layout.tsx      # Dashboard layout with navigation
│       │   ├── page.tsx        # Entry creation page
│       │   ├── timeline/       # Timeline view
│       │   └── insights/       # Pattern insights view
│       └── actions/             # Server Actions (mutations/API calls)
│           ├── auth.ts
│           ├── entries.ts
│           └── patterns.ts
│
├── components/
│   ├── ui/                      # Shadcn UI components
│   ├── entry/                   # Entry-related components
│   │   ├── entry-form.tsx
│   │   ├── text-input.tsx
│   │   ├── audio-recorder.tsx   # 'use client' - uses MediaRecorder API
│   │   ├── entity-display.tsx
│   │   └── entry-card.tsx
│   ├── timeline/
│   │   ├── timeline.tsx
│   │   ├── timeline-entry.tsx
│   │   └── timeline-filters.tsx
│   ├── insights/
│   │   ├── insights-grid.tsx
│   │   ├── pattern-card.tsx
│   │   ├── confidence-badge.tsx
│   │   └── pattern-detail.tsx
│   ├── progress/
│   │   ├── progress-tracker.tsx
│   │   └── milestone-badge.tsx
│   └── layout/
│       ├── nav-bar.tsx
│       ├── user-menu.tsx
│       └── theme-toggle.tsx
│
├── lib/
│   ├── supabase/
│   │   ├── client.ts            # Client-side Supabase client
│   │   └── server.ts            # Server-side Supabase client (with cookies)
│   ├── gemini.ts                # Gemini API integration
│   ├── groq.ts                  # Groq Whisper integration
│   ├── patterns.ts              # Pattern detection algorithms
│   └── utils.ts                 # Utility functions (cn, etc.)
│
├── types/
│   ├── database.ts              # Supabase database types
│   ├── entry.ts
│   └── pattern.ts
│
├── requirements/                # Product requirements
│   ├── prd.md                   # Full product concept & user stories
│   └── High_req.md              # Implementation plan & architecture
│
├── next.config.ts
├── tsconfig.json                # Path alias: @/* → ./src/*
├── tailwind.config.ts
└── package.json
```

## Architecture Patterns

### Server vs Client Components

**Default to Server Components** - Next.js 16 uses Server Components by default. Only use `'use client'` when:
- Using browser APIs (MediaRecorder, localStorage, etc.)
- Using React hooks (useState, useEffect, useContext)
- Handling user interactions (onClick, onChange)
- Using client-side libraries

**Examples:**
- ✅ Server: Timeline, entry list, pattern cards (data fetching)
- ❌ Client: Audio recorder, form inputs with state, theme toggle

### Server Actions Pattern

All database mutations and API calls go through Server Actions:

```typescript
// app/actions/entries.ts
'use server'

export async function createEntry(formData: FormData) {
  const supabase = await createClient();
  const text = formData.get('text');

  // 1. Save entry to Supabase
  const { data: entry } = await supabase
    .from('entries')
    .insert({ raw_text: text, entry_type: 'text' })
    .select()
    .single();

  // 2. Extract entities using Gemini
  const entities = await extractHealthEntities(text);

  // 3. Save extracted entities
  await supabase
    .from('extracted_entities')
    .insert({ entry_id: entry.id, ...entities });

  return { success: true, entry, entities };
}
```

### Database Schema (Supabase)

**Key tables:**
- `entries` - Raw journal entries (text/audio transcripts)
- `extracted_entities` - AI-extracted symptoms, mood, activities, food, time_of_day
- `patterns` - Detected patterns (frequency, temporal, correlation, trigger)

**Security:** Row Level Security (RLS) ensures users can only access their own data.

See `requirements/High_req.md` for full SQL schema.

### AI Integration Patterns

**Gemini 2.5 Flash (Entity Extraction):**
- Structured output using `responseSchema` for reliable JSON
- Conservative extraction (only extract what's explicitly mentioned)
- Confidence scores (0-1) to track accuracy

**Gemini 2.5 Flash (Pattern Insights):**
- Analyze 30 days of entries with extracted entities
- Generate 2-3 insights with confidence levels (low/medium/high)
- Include disclaimers about correlation vs causation

**Groq Whisper (Transcription):**
- Fast audio-to-text conversion
- Client records via MediaRecorder → uploads to server
- Server sends to Groq API → returns editable transcript

## Implementation Priorities

### Must Have (P0) - POC Core
1. Text entry with entity extraction
2. Voice entry with transcription
3. Timeline view of all entries
4. Frequency pattern detection
5. Temporal pattern detection
6. Basic insights display
7. Progress tracking (X of 10 entries)

### Should Have (P1)
1. Correlation pattern detection
2. Trigger hypothesis detection
3. Pattern confidence scores
4. Ability to dismiss patterns
5. Micro-insights after each entry

### Nice to Have (P2)
1. Smart follow-up questions
2. Conversational intelligence
3. Export/sharing functionality
4. Entry editing
5. Advanced filtering

## Critical Success Metrics

- **Entry Time:** < 45 seconds per entry
- **Entity Extraction Accuracy:** 80%+ correct identification
- **Pattern Accuracy:** User agrees 80%+ of time
- **Time to First Entry:** < 2 minutes from app open
- **Voice Transcription Accuracy:** > 90%

## Key Development Guidelines

### TypeScript Configuration
- Strict mode enabled
- Path alias: `@/*` maps to `./src/*`
- Target: ES2017
- Module resolution: bundler

### Styling with Tailwind v4
- Use `@import "tailwindcss"` syntax in globals.css
- Custom properties in `@theme inline` block
- Dark mode: prefers-color-scheme (automatic)
- Fonts: Geist Sans and Geist Mono (loaded via next/font/google)

### Environment Variables (Required)
```env
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
GEMINI_API_KEY=
GROQ_API_KEY=
```

### Component Development
- Use Shadcn UI for base components (button, card, input, textarea, badge, dialog, spinner, progress)
- Build custom components in domain-specific folders (entry/, timeline/, insights/)
- Components are owned by the project (copy-paste, not npm packages)

### Error Handling
- Show user-friendly error messages
- Handle missing audio permissions gracefully
- Account for API rate limiting
- Provide loading states for all async operations

### User Experience Principles
- **Effortless:** Sub-45 second entry time
- **Trust-building:** Show what was understood after each entry
- **Progressive value:** Provide micro-insights before pattern threshold
- **Non-judgmental:** No guilt-inducing messages for gaps
- **Private:** User data stays secure with RLS

## Common Workflows

### Adding a New Server Action
1. Create function in `app/actions/[domain].ts`
2. Add `'use server'` directive at top
3. Use `createClient()` from `lib/supabase/server.ts` for auth-aware DB access
4. Return serializable data (no functions, class instances)

### Adding a New Component
1. Determine if it needs 'use client' (interactivity/hooks/browser APIs)
2. Place in appropriate domain folder (entry/, timeline/, insights/, etc.)
3. Use Shadcn UI primitives where possible
4. Import via path alias: `import { Button } from '@/components/ui/button'`

### Testing AI Integrations
- Test entity extraction with varied input (symptoms, mood, activities, food)
- Verify structured output matches schema
- Check confidence scores are reasonable
- Test pattern detection with 10+ realistic entries

## Common Errors to Avoid

1. **Don't** add `'use client'` to Server Components unnecessarily
2. **Don't** use Supabase client-side client for authenticated operations
3. **Don't** forget to handle auth state (users must be logged in)
4. **Don't** commit .env files (already in .gitignore)
5. **Don't** use `'` in commit messages or code - use `&apos;` in JSX (common linting error)

## Deployment (Railway)

1. Connect GitHub repository to Railway
2. Add environment variables in Railway dashboard
3. Railway auto-detects Next.js and handles build
4. Push to main branch → automatic deployment

## Reference Documentation

- **Product Vision:** `requirements/prd.md` - User stories, personas, success criteria
- **Implementation Plan:** `requirements/High_req.md` - Architecture, database schema, API integration code, 4-week phased plan
- **Next.js 16:** https://nextjs.org/docs
- **Supabase:** https://supabase.com/docs
- **Shadcn UI:** https://ui.shadcn.com/docs
