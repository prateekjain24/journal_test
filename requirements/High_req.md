# Health Clarity POC: High-Level Implementation Plan

## 🏗️ System Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                         User Browser                             │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  Next.js 16 App (Shadcn UI Components)                     │ │
│  │  - Server Components (default)                              │ │
│  │  - Client Components (interactivity)                        │ │
│  └────────────┬────────────────────────────────────┬───────────┘ │
└───────────────┼────────────────────────────────────┼─────────────┘
                │                                    │
                ▼                                    ▼
    ┌──────────────────────┐           ┌──────────────────────┐
    │  Server Actions      │           │  Supabase            │
    │  (Next.js 16)        │◄─────────►│  - PostgreSQL DB     │
    │                      │           │  - Auth              │
    │  - createEntry()     │           │  - Real-time         │
    │  - transcribeAudio() │           └──────────────────────┘
    │  - analyzePatterns() │
    │  - getInsights()     │
    └──────────┬───────────┘
               │
               │ Calls External APIs
               │
    ┌──────────┴──────────────────────────────────────────┐
    │                                                      │
    ▼                                                      ▼
┌─────────────────────┐                      ┌────────────────────┐
│  Google Gemini 2.5  │                      │  Groq Whisper      │
│  Flash API          │                      │  API               │
│                     │                      │                    │
│  - Entity Extract   │                      │  - Audio → Text    │
│  - Pattern Insights │                      │  - Fast Transc.    │
└─────────────────────┘                      └────────────────────┘
```

---

## 🎯 Technology Integration Strategy

### Next.js 16 + App Router
- **Server Components by default** - All pages/components are Server Components unless marked with `'use client'`
- **Server Actions** - All database mutations and API calls happen through Server Actions
- **Turbopack** - Faster development builds (automatic in Next.js 16)
- **File-based routing** - Use app directory structure for intuitive routing

### Supabase Integration
- **Server-side client** - Create Supabase client with cookie handling for auth
- **Row Level Security** - Users can only access their own health data
- **Real-time subscriptions** - Optional: Live updates when patterns are detected
- **Auth** - Simple email/password for POC (can add OAuth later)

### Google Gemini 2.5 Flash
- **Structured output** - Use `responseSchema` for reliable JSON entity extraction
- **Prompt engineering** - Craft prompts for health-specific entity recognition
- **Cost optimization** - Flash model is cost-effective for high-volume requests

### Groq Whisper
- **Fast transcription** - Near-instant audio-to-text conversion
- **whisper-large-v3-turbo** - Optimal balance of speed and accuracy
- **Client-side audio recording** - Browser MediaRecorder API → send to server

### Shadcn UI
- **Copy-paste components** - Own and customize all UI code
- **Composable patterns** - Build complex UIs from simple primitives
- **Accessibility built-in** - ARIA-compliant components
- **Dark mode ready** - CSS variables for theme switching

---

## 📦 Component Structure

```
components/
├── ui/                          # Shadcn UI components
│   ├── button.tsx
│   ├── card.tsx
│   ├── input.tsx
│   ├── textarea.tsx
│   ├── badge.tsx
│   ├── dialog.tsx
│   ├── spinner.tsx
│   └── progress.tsx
│
├── entry/                       # Entry-related components
│   ├── entry-form.tsx          # Main entry form (text + audio)
│   ├── text-input.tsx          # Text entry subcomponent
│   ├── audio-recorder.tsx      # Voice recording UI (client)
│   ├── entity-display.tsx      # Show extracted entities
│   └── entry-card.tsx          # Single entry display
│
├── timeline/                    # Timeline view components
│   ├── timeline.tsx            # Main timeline container
│   ├── timeline-entry.tsx      # Individual entry in timeline
│   └── timeline-filters.tsx    # Filter/sort controls
│
├── insights/                    # Pattern insights components
│   ├── insights-grid.tsx       # Grid of pattern cards
│   ├── pattern-card.tsx        # Single pattern display
│   ├── confidence-badge.tsx    # Confidence score indicator
│   └── pattern-detail.tsx      # Expanded pattern view
│
├── progress/                    # Progress tracking
│   ├── progress-tracker.tsx   # "X of 10 entries" display
│   └── milestone-badge.tsx    # Achievement indicators
│
└── layout/                      # Layout components
    ├── nav-bar.tsx             # Main navigation
    ├── user-menu.tsx           # User profile dropdown
    └── theme-toggle.tsx        # Dark mode switch
```

---

## 🗄️ Database Schema (Supabase)

```sql
-- Users table (managed by Supabase Auth)
-- No need to create manually

-- Entries table
CREATE TABLE entries (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
    entry_type TEXT NOT NULL CHECK (entry_type IN ('text', 'audio')),
    raw_text TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Extracted entities table
CREATE TABLE extracted_entities (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    entry_id UUID NOT NULL REFERENCES entries(id) ON DELETE CASCADE,
    symptoms JSONB DEFAULT '[]'::jsonb,
    mood TEXT,
    activities JSONB DEFAULT '[]'::jsonb,
    food JSONB DEFAULT '[]'::jsonb,
    time_of_day TEXT,
    confidence_score REAL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(entry_id)
);

-- Patterns table
CREATE TABLE patterns (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
    pattern_type TEXT NOT NULL CHECK (pattern_type IN ('frequency', 'temporal', 'correlation', 'trigger')),
    title TEXT NOT NULL,
    description TEXT NOT NULL,
    confidence TEXT NOT NULL CHECK (confidence IN ('low', 'medium', 'high')),
    supporting_entry_ids JSONB NOT NULL,
    metadata JSONB DEFAULT '{}'::jsonb,
    dismissed BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_updated TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Row Level Security policies
ALTER TABLE entries ENABLE ROW LEVEL SECURITY;
ALTER TABLE extracted_entities ENABLE ROW LEVEL SECURITY;
ALTER TABLE patterns ENABLE ROW LEVEL SECURITY;

-- Users can only see their own data
CREATE POLICY "Users can view own entries" ON entries
    FOR SELECT USING (auth.uid() = user_id);

CREATE POLICY "Users can insert own entries" ON entries
    FOR INSERT WITH CHECK (auth.uid() = user_id);

CREATE POLICY "Users can view own entities" ON extracted_entities
    FOR SELECT USING (
        entry_id IN (SELECT id FROM entries WHERE user_id = auth.uid())
    );

CREATE POLICY "Users can insert own entities" ON extracted_entities
    FOR INSERT WITH CHECK (
        entry_id IN (SELECT id FROM entries WHERE user_id = auth.uid())
    );

CREATE POLICY "Users can view own patterns" ON patterns
    FOR SELECT USING (auth.uid() = user_id);

CREATE POLICY "Users can update own patterns" ON patterns
    FOR UPDATE USING (auth.uid() = user_id);

-- Indexes for performance
CREATE INDEX idx_entries_user_created ON entries(user_id, created_at DESC);
CREATE INDEX idx_patterns_user_created ON patterns(user_id, created_at DESC);
CREATE INDEX idx_extracted_entities_entry ON extracted_entities(entry_id);
```

---

## 🔌 API Integration Details

### 1. Google Gemini 2.5 Flash - Entity Extraction

```typescript
// lib/gemini.ts
import { GoogleGenerativeAI } from '@google/generative-ai';

const genAI = new GoogleGenerativeAI(process.env.GEMINI_API_KEY!);

export async function extractHealthEntities(text: string) {
  const model = genAI.getGenerativeModel({ model: 'gemini-2.5-flash' });
  
  const schema = {
    type: 'object',
    properties: {
      symptoms: {
        type: 'array',
        items: { type: 'string' },
        description: 'Physical or mental symptoms mentioned (e.g., headache, fatigue, anxiety)',
      },
      mood: {
        type: 'string',
        description: 'Overall emotional state (e.g., happy, stressed, anxious, calm)',
      },
      activities: {
        type: 'array',
        items: { type: 'string' },
        description: 'Activities mentioned (e.g., exercise, work, sleep, screen time)',
      },
      food: {
        type: 'array',
        items: { type: 'string' },
        description: 'Foods, drinks, or substances mentioned (e.g., coffee, alcohol, dairy)',
      },
      time_of_day: {
        type: 'string',
        enum: ['morning', 'afternoon', 'evening', 'night', 'unknown'],
        description: 'Time context from the entry',
      },
      confidence: {
        type: 'number',
        description: 'Confidence score 0-1 for the extraction quality',
      },
    },
    required: ['symptoms', 'mood', 'activities', 'food', 'time_of_day', 'confidence'],
  };

  const result = await model.generateContent({
    contents: [{
      role: 'user',
      parts: [{
        text: `Extract health-related entities from this journal entry. Be conservative and only extract what is explicitly mentioned.

Entry: "${text}"

Return a JSON object with the specified schema. If nothing is mentioned for a field, return empty array or "unknown".`,
      }],
    }],
    generationConfig: {
      responseSchema: schema,
      responseMimeType: 'application/json',
      temperature: 0.3, // Lower temperature for consistency
    },
  });

  const response = await result.response;
  return JSON.parse(response.text());
}
```

### 2. Google Gemini 2.5 Flash - Pattern Insights

```typescript
// lib/gemini.ts (continued)
export async function generatePatternInsights(entries: any[]) {
  const model = genAI.getGenerativeModel({ model: 'gemini-2.5-flash' });
  
  const entriesSummary = entries.map(e => ({
    date: e.created_at,
    symptoms: e.extracted_entities?.symptoms || [],
    mood: e.extracted_entities?.mood,
    activities: e.extracted_entities?.activities || [],
    food: e.extracted_entities?.food || [],
  }));

  const prompt = `Analyze these health journal entries and identify 2-3 notable patterns or correlations.

Entries: ${JSON.stringify(entriesSummary, null, 2)}

Guidelines:
- Focus on symptom-trigger correlations and temporal patterns
- Be specific but acknowledge uncertainty
- Always include disclaimer about other factors
- Keep insights under 50 words each
- Assign confidence level: low, medium, or high

Return JSON array of insights:
[
  {
    "pattern_type": "frequency|temporal|correlation|trigger",
    "title": "Brief title",
    "description": "Clear description with specifics",
    "confidence": "low|medium|high",
    "supporting_entry_ids": [array of entry IDs]
  }
]`;

  const result = await model.generateContent({
    contents: [{ role: 'user', parts: [{ text: prompt }] }],
    generationConfig: {
      responseMimeType: 'application/json',
      temperature: 0.5,
    },
  });

  return JSON.parse((await result.response).text());
}
```

### 3. Groq Whisper - Audio Transcription

```typescript
// lib/groq.ts
import Groq from 'groq-sdk';
import { File } from 'buffer';

const groq = new Groq({ apiKey: process.env.GROQ_API_KEY });

export async function transcribeAudio(audioBuffer: Buffer, filename: string) {
  const transcription = await groq.audio.transcriptions.create({
    file: new File([audioBuffer], filename, { type: 'audio/webm' }),
    model: 'whisper-large-v3-turbo',
    response_format: 'json',
    language: 'en', // or detect automatically
  });

  return transcription.text;
}
```

### 4. Supabase Server Client

```typescript
// lib/supabase/server.ts
import { createServerClient } from '@supabase/ssr';
import { cookies } from 'next/headers';

export async function createClient() {
  const cookieStore = await cookies();

  return createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() {
          return cookieStore.getAll();
        },
        setAll(cookiesToSet) {
          try {
            cookiesToSet.forEach(({ name, value, options }) =>
              cookieStore.set(name, value, options)
            );
          } catch {}
        },
      },
    }
  );
}
```

---

## 📅 Implementation Phases (4 Weeks)

### **Phase 1: Foundation Setup** (Week 1)

**Objectives:** Project initialization, authentication, basic layout

**Tasks:**
1. Create Next.js 16 project with TypeScript and Tailwind
   ```bash
   npx create-next-app@latest health-clarity --typescript --tailwind --app
   ```

2. Initialize Shadcn UI
   ```bash
   npx shadcn-ui@latest init
   npx shadcn-ui@latest add button card input textarea badge dialog spinner
   ```

3. Set up Supabase project
   - Create new project on supabase.com
   - Run database migrations (schema from above)
   - Configure authentication (email/password)
   - Get API keys

4. Configure environment variables
   ```env
   NEXT_PUBLIC_SUPABASE_URL=
   NEXT_PUBLIC_SUPABASE_ANON_KEY=
   GEMINI_API_KEY=
   GROQ_API_KEY=
   ```

5. Create basic layout structure
   - `app/layout.tsx` - Root layout with providers
   - `app/(auth)/login/page.tsx` - Login page
   - `app/(dashboard)/layout.tsx` - Dashboard layout with nav
   - `app/(dashboard)/page.tsx` - Home page

6. Implement authentication
   - Server Actions for login/signup
   - Protected routes middleware
   - User context provider

**Deliverable:** Working authentication system with basic layout

---

### **Phase 2: Entry System** (Week 2)

**Objectives:** Text and audio entry creation with entity extraction

**Tasks:**
1. Build text entry form component
   - `components/entry/text-input.tsx`
   - Textarea with character count
   - Submit button

2. Integrate Groq Whisper for audio transcription
   - `components/entry/audio-recorder.tsx` (client component)
   - Browser MediaRecorder API
   - Upload audio to server
   - Server Action calls Groq API
   - Display editable transcript

3. Create entry submission Server Action
   ```typescript
   // app/actions/entries.ts
   'use server'
   
   export async function createEntry(formData: FormData) {
     const supabase = await createClient();
     const text = formData.get('text');
     
     // Save entry
     const { data: entry } = await supabase
       .from('entries')
       .insert({ raw_text: text, entry_type: 'text' })
       .select()
       .single();
     
     // Extract entities using Gemini
     const entities = await extractHealthEntities(text);
     
     // Save extracted entities
     await supabase
       .from('extracted_entities')
       .insert({ entry_id: entry.id, ...entities });
     
     return { success: true, entry, entities };
   }
   ```

4. Build entity display component
   - `components/entry/entity-display.tsx`
   - Show extracted symptoms, mood, activities as badges
   - Confirmation message after entry

5. Create timeline view
   - `app/(dashboard)/timeline/page.tsx`
   - Fetch entries as Server Component
   - `components/timeline/timeline-entry.tsx` for each entry
   - Reverse chronological order
   - Expand/collapse full text

6. Add progress tracker
   - `components/progress/progress-tracker.tsx`
   - "X of 10 entries to unlock insights"
   - Progress bar

**Deliverable:** Full entry creation flow (text + audio) with timeline view

---

### **Phase 3: Pattern Detection & Insights** (Week 3)

**Objectives:** Detect patterns and display insights

**Tasks:**
1. Implement pattern detection algorithms
   ```typescript
   // lib/patterns.ts
   
   export function detectFrequencyPatterns(entries: any[]) {
     const symptomCounts = new Map();
     const window = 14; // days
     
     entries.forEach(entry => {
       entry.extracted_entities?.symptoms?.forEach((symptom: string) => {
         symptomCounts.set(
           symptom,
           (symptomCounts.get(symptom) || 0) + 1
         );
       });
     });
     
     return Array.from(symptomCounts.entries())
       .filter(([_, count]) => count >= 3)
       .map(([symptom, count]) => ({
         type: 'frequency',
         title: `Recurring ${symptom}`,
         description: `You've mentioned "${symptom}" ${count} times in the last ${window} days`,
         confidence: count >= 5 ? 'high' : 'medium',
       }));
   }
   
   export function detectTemporalPatterns(entries: any[]) {
     // Group by day of week and time of day
     const patterns = new Map();
     
     entries.forEach(entry => {
       const date = new Date(entry.created_at);
       const dayOfWeek = date.toLocaleDateString('en-US', { weekday: 'long' });
       const timeOfDay = entry.extracted_entities?.time_of_day;
       
       entry.extracted_entities?.symptoms?.forEach((symptom: string) => {
         const key = `${symptom}-${dayOfWeek}-${timeOfDay}`;
         patterns.set(key, (patterns.get(key) || 0) + 1);
       });
     });
     
     // Return patterns with 3+ occurrences
     return Array.from(patterns.entries())
       .filter(([_, count]) => count >= 3)
       .map(([key, count]) => {
         const [symptom, day, time] = key.split('-');
         return {
           type: 'temporal',
           title: `${symptom} pattern`,
           description: `Your ${symptom} tends to occur on ${day}s during the ${time}`,
           confidence: count >= 4 ? 'high' : 'medium',
         };
       });
   }
   ```

2. Create pattern analysis Server Action
   ```typescript
   // app/actions/patterns.ts
   'use server'
   
   export async function analyzePatterns() {
     const supabase = await createClient();
     
     // Fetch last 30 days of entries with entities
     const { data: entries } = await supabase
       .from('entries')
       .select('*, extracted_entities(*)')
       .gte('created_at', new Date(Date.now() - 30*24*60*60*1000).toISOString())
       .order('created_at', { ascending: false });
     
     // Run basic pattern detection
     const frequencyPatterns = detectFrequencyPatterns(entries);
     const temporalPatterns = detectTemporalPatterns(entries);
     
     // Use Gemini for additional insights
     const aiInsights = await generatePatternInsights(entries);
     
     // Combine and save patterns
     const allPatterns = [...frequencyPatterns, ...temporalPatterns, ...aiInsights];
     
     await supabase.from('patterns').insert(allPatterns);
     
     return allPatterns;
   }
   ```

3. Build insights components
   - `app/(dashboard)/insights/page.tsx` - Insights page
   - `components/insights/insights-grid.tsx` - Grid layout
   - `components/insights/pattern-card.tsx` - Individual pattern card with:
     * Pattern type badge
     * Title and description
     * Confidence indicator
     * "Dismiss" button
     * "Show related entries" link

4. Add pattern dismissal
   - Server Action to mark pattern as dismissed
   - UI to hide dismissed patterns

5. Create "Analyze Patterns" button
   - Trigger pattern analysis on demand
   - Loading state during analysis
   - Success notification

**Deliverable:** Working pattern detection and insights display

---

### **Phase 4: Polish & Deploy** (Week 4)

**Objectives:** Improve UX, handle edge cases, deploy to Railway

**Tasks:**
1. UI/UX improvements
   - Loading states for all async operations
   - Error boundaries and error messages
   - Empty states ("No entries yet")
   - Skeleton loaders
   - Toast notifications for success/error

2. Add micro-feedback after each entry
   - "Entry saved!" confirmation
   - Show what was understood
   - Progress update

3. Edge case handling
   - Very long entries (truncate or paginate)
   - Missing audio permissions
   - API rate limiting
   - Network errors

4. Performance optimization
   - Optimize Supabase queries
   - Add caching where appropriate
   - Lazy load timeline entries

5. Dark mode setup
   - Configure Tailwind dark mode
   - Add theme toggle component
   - Test all components in both modes

6. Railway deployment
   - Connect GitHub repository
   - Add environment variables in Railway dashboard
   - Configure build settings (automatic for Next.js)
   - Deploy and test

7. Testing
   - Manual testing of all user flows
   - Test on mobile browsers
   - Verify audio recording works
   - Check pattern detection with real data

**Deliverable:** Fully functional POC deployed on Railway

---

## 📁 File Structure

```
health-clarity/
├── app/
│   ├── layout.tsx                    # Root layout with providers
│   ├── page.tsx                      # Landing page (redirect to dashboard if auth)
│   ├── globals.css                   # Global styles
│   │
│   ├── (auth)/                       # Auth routes (grouped)
│   │   ├── layout.tsx               # Auth layout (centered, no nav)
│   │   ├── login/
│   │   │   └── page.tsx             # Login page
│   │   └── signup/
│   │       └── page.tsx             # Signup page
│   │
│   ├── (dashboard)/                  # Dashboard routes (grouped)
│   │   ├── layout.tsx               # Dashboard layout (with nav)
│   │   ├── page.tsx                 # Home/Entry creation page
│   │   ├── timeline/
│   │   │   └── page.tsx             # Timeline view
│   │   └── insights/
│   │       └── page.tsx             # Insights/patterns page
│   │
│   └── actions/                      # Server Actions
│       ├── auth.ts                  # Login/signup actions
│       ├── entries.ts               # Entry CRUD actions
│       └── patterns.ts              # Pattern analysis actions
│
├── components/
│   ├── ui/                          # Shadcn UI components
│   │   ├── button.tsx
│   │   ├── card.tsx
│   │   ├── input.tsx
│   │   ├── textarea.tsx
│   │   ├── badge.tsx
│   │   ├── dialog.tsx
│   │   ├── spinner.tsx
│   │   └── ... (other Shadcn components)
│   │
│   ├── entry/
│   │   ├── entry-form.tsx           # Main entry form container
│   │   ├── text-input.tsx           # Text entry component
│   │   ├── audio-recorder.tsx       # Audio recording component
│   │   ├── entity-display.tsx       # Display extracted entities
│   │   └── entry-card.tsx           # Single entry card
│   │
│   ├── timeline/
│   │   ├── timeline.tsx             # Timeline container
│   │   ├── timeline-entry.tsx       # Timeline entry item
│   │   └── timeline-filters.tsx     # Filter controls
│   │
│   ├── insights/
│   │   ├── insights-grid.tsx        # Insights grid container
│   │   ├── pattern-card.tsx         # Pattern insight card
│   │   ├── confidence-badge.tsx     # Confidence indicator
│   │   └── pattern-detail.tsx       # Expanded pattern view
│   │
│   ├── progress/
│   │   ├── progress-tracker.tsx     # Progress toward insights
│   │   └── milestone-badge.tsx      # Achievement badges
│   │
│   └── layout/
│       ├── nav-bar.tsx              # Main navigation
│       ├── user-menu.tsx            # User dropdown menu
│       └── theme-toggle.tsx         # Dark mode toggle
│
├── lib/
│   ├── supabase/
│   │   ├── client.ts                # Client-side Supabase client
│   │   └── server.ts                # Server-side Supabase client
│   │
│   ├── gemini.ts                    # Gemini API integration
│   ├── groq.ts                      # Groq Whisper integration
│   ├── patterns.ts                  # Pattern detection algorithms
│   └── utils.ts                     # Utility functions (cn, etc.)
│
├── types/
│   ├── database.ts                  # Supabase database types
│   ├── entry.ts                     # Entry types
│   └── pattern.ts                   # Pattern types
│
├── public/                          # Static assets
├── .env.local                       # Environment variables (gitignored)
├── .gitignore
├── next.config.js
├── package.json
├── tailwind.config.js
├── tsconfig.json
└── README.md
```

---

## 🚀 Deployment Strategy (Railway)

### Setup Steps

1. **Connect Repository**
   - Push code to GitHub
   - Connect Railway to your GitHub account
   - Select repository

2. **Configure Environment Variables**
   In Railway dashboard, add:
   ```
   NEXT_PUBLIC_SUPABASE_URL=your_supabase_url
   NEXT_PUBLIC_SUPABASE_ANON_KEY=your_supabase_anon_key
   GEMINI_API_KEY=your_gemini_key
   GROQ_API_KEY=your_groq_key
   ```

3. **Deploy**
   - Railway auto-detects Next.js
   - Builds with Turbopack automatically
   - Assigns a public URL
   - No additional configuration needed

4. **Verify Deployment**
   - Check build logs
   - Test authentication
   - Create test entry
   - Verify API integrations work

### Ongoing Deployment
- Push to main branch → automatic deployment
- Railway handles SSL certificates
- Monitor usage in Railway dashboard

---

## 🎯 POC Success Criteria

**Must Have:**
- ✅ User can sign up and log in
- ✅ User can create text entries in < 30 seconds
- ✅ User can create audio entries with transcription
- ✅ Entities are extracted and displayed correctly (80%+ accuracy)
- ✅ Timeline displays all entries
- ✅ After 10+ entries, patterns are detected
- ✅ Insights are displayed with confidence scores
- ✅ App is deployed and accessible via URL

**Nice to Have (if time permits):**
- Entry editing
- Pattern dismissal
- Dark mode
- Mobile-responsive design
- Error recovery mechanisms

---

## 🔧 Development Workflow with Claude Code

### Recommended Approach

1. **Start with Phase 1**
   - Use Claude Code to scaffold the Next.js project
   - Set up Supabase and run migrations
   - Implement authentication first

2. **Iterate Component by Component**
   - Build one component at a time
   - Test immediately in browser
   - Commit working code frequently

3. **Integration Testing**
   - Test APIs with sample data
   - Verify entity extraction quality
   - Tune prompts if needed

4. **User Testing**
   - Use the app yourself for 2 weeks
   - Gather real pattern insights
   - Refine based on experience

### Claude Code Commands

```bash
# Initialize project
npx create-next-app@latest health-clarity --typescript --tailwind --app
cd health-clarity

# Install dependencies
npm install @supabase/ssr @supabase/supabase-js
npm install @google/generative-ai groq-sdk
npm install date-fns # for date handling

# Shadcn UI setup
npx shadcn-ui@latest init
npx shadcn-ui@latest add button card input textarea badge dialog spinner progress

# Run development server
npm run dev
```

---

## 📝 Final Notes

- **Keep it simple**: Focus on core POC features, resist scope creep
- **Test with real data**: Use the app yourself to validate value
- **Iterate on prompts**: Gemini and Groq prompts may need tuning
- **Monitor costs**: Track API usage (Gemini Flash and Groq are cheap but monitor)
- **Security**: Supabase RLS ensures data privacy
- **Performance**: Next.js 16 + Turbopack makes development fast
- **Scalability**: This architecture scales beyond POC with minimal changes

**You now have everything needed to build Health Clarity POC in 4 weeks. Start with Phase 1 and use Claude Code to implement each component systematically.**