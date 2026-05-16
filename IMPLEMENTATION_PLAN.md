# Fire Suppression Compliance Checker - Supabase + Vercel Migration Plan

## Executive Summary

Migrating from a static HTML application to a full-stack Next.js application with:
- **Frontend**: Next.js 14+ (React framework)
- **Backend**: Supabase (PostgreSQL + Auth + Storage)
- **Deployment**: Vercel (CI/CD + Edge functions)
- **Language**: TypeScript for type safety and commercial readiness

## Requirements Summary

### ✅ User Authentication & Access
- Login/signup functionality (Supabase Auth)
- User-specific inspections (each user sees only their own)
- Inspection sharing between specific users (you and Ethan)

### ✅ Data Persistence
- Save inspection reports to PostgreSQL
- Store captured images in Supabase Storage
- View historical inspections dashboard

### ✅ Commercial-Ready Architecture
- TypeScript for type safety
- Next.js for scalability and SEO
- Server-side rendering for performance
- API routes for backend logic
- Environment-based configuration

---

## Phase 1: Project Setup & Configuration

### 1.1 Initialize Next.js Project

```bash
# Create new Next.js app with TypeScript
npx create-next-app@latest fire-suppression-checker --typescript --tailwind --app --src-dir

# Project structure:
fire-suppression-checker/
├── src/
│   ├── app/                 # App router (Next.js 14+)
│   │   ├── layout.tsx       # Root layout
│   │   ├── page.tsx         # Home page
│   │   ├── login/           # Auth pages
│   │   ├── dashboard/       # Protected dashboard
│   │   └── api/             # API routes
│   ├── components/          # React components
│   ├── lib/                 # Utilities & Supabase client
│   └── types/               # TypeScript definitions
├── public/                  # Static assets
├── .env.local              # Environment variables
├── next.config.js          # Next.js configuration
├── package.json            # Dependencies
└── tsconfig.json           # TypeScript config
```

### 1.2 Install Dependencies

```bash
npm install @supabase/supabase-js @supabase/auth-helpers-nextjs
npm install @supabase/auth-ui-react @supabase/auth-ui-shared
npm install date-fns uuid
npm install -D @types/uuid
```

### 1.3 Setup Supabase Project

1. Go to https://supabase.com
2. Create new project: "fire-suppression-checker"
3. Save credentials:
   - Project URL: `https://xxxxx.supabase.co`
   - Anon public key: `eyJhbGc...`
   - Service role key: `eyJhbGc...` (keep secret!)

### 1.4 Environment Variables

Create `.env.local`:
```env
NEXT_PUBLIC_SUPABASE_URL=https://xxxxx.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJhbGc...
SUPABASE_SERVICE_ROLE_KEY=eyJhbGc...
```

---

## Phase 2: Database Schema Design

### 2.1 Tables Overview

```
users (handled by Supabase Auth)
├── id (uuid, primary key)
├── email (text)
├── created_at (timestamp)
└── metadata (jsonb)

inspections
├── id (uuid, primary key)
├── user_id (uuid, foreign key → auth.users)
├── state (text)
├── system_type (text)
├── building_type (text)
├── image_url (text, → Supabase Storage)
├── status (text: draft, completed, shared)
├── created_at (timestamp)
├── updated_at (timestamp)
└── shared_with (uuid[], array of user IDs)

compliance_results
├── id (uuid, primary key)
├── inspection_id (uuid, foreign key → inspections)
├── rule_id (text)
├── check_name (text)
├── status (text: pass, fail, warning, not_applicable)
├── notes (text)
├── is_required (boolean)
└── created_at (timestamp)
```

### 2.2 SQL Migration Script

```sql
-- Enable UUID extension
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Inspections table
CREATE TABLE inspections (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE NOT NULL,
  state TEXT NOT NULL,
  system_type TEXT NOT NULL,
  building_type TEXT NOT NULL,
  image_url TEXT,
  status TEXT DEFAULT 'draft' CHECK (status IN ('draft', 'completed', 'shared')),
  shared_with UUID[] DEFAULT '{}',
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Compliance results table
CREATE TABLE compliance_results (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  inspection_id UUID REFERENCES inspections(id) ON DELETE CASCADE NOT NULL,
  rule_id TEXT NOT NULL,
  check_name TEXT NOT NULL,
  status TEXT NOT NULL CHECK (status IN ('pass', 'fail', 'warning', 'not_applicable')),
  notes TEXT,
  is_required BOOLEAN DEFAULT false,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Indexes for performance
CREATE INDEX idx_inspections_user_id ON inspections(user_id);
CREATE INDEX idx_inspections_created_at ON inspections(created_at DESC);
CREATE INDEX idx_compliance_inspection_id ON compliance_results(inspection_id);

-- Row Level Security (RLS) policies
ALTER TABLE inspections ENABLE ROW LEVEL SECURITY;
ALTER TABLE compliance_results ENABLE ROW LEVEL SECURITY;

-- Users can only see their own inspections or inspections shared with them
CREATE POLICY "Users can view own inspections"
  ON inspections FOR SELECT
  USING (
    auth.uid() = user_id
    OR auth.uid() = ANY(shared_with)
  );

-- Users can only insert their own inspections
CREATE POLICY "Users can insert own inspections"
  ON inspections FOR INSERT
  WITH CHECK (auth.uid() = user_id);

-- Users can only update their own inspections
CREATE POLICY "Users can update own inspections"
  ON inspections FOR UPDATE
  USING (auth.uid() = user_id);

-- Users can only delete their own inspections
CREATE POLICY "Users can delete own inspections"
  ON inspections FOR DELETE
  USING (auth.uid() = user_id);

-- Compliance results: users can see results for inspections they have access to
CREATE POLICY "Users can view compliance results"
  ON compliance_results FOR SELECT
  USING (
    EXISTS (
      SELECT 1 FROM inspections
      WHERE inspections.id = compliance_results.inspection_id
      AND (inspections.user_id = auth.uid() OR auth.uid() = ANY(inspections.shared_with))
    )
  );

-- Users can insert compliance results for their own inspections
CREATE POLICY "Users can insert compliance results"
  ON compliance_results FOR INSERT
  WITH CHECK (
    EXISTS (
      SELECT 1 FROM inspections
      WHERE inspections.id = compliance_results.inspection_id
      AND inspections.user_id = auth.uid()
    )
  );
```

### 2.3 Supabase Storage Bucket

```sql
-- Create storage bucket for inspection images
INSERT INTO storage.buckets (id, name, public)
VALUES ('inspection-images', 'inspection-images', false);

-- Storage policies
CREATE POLICY "Users can upload own images"
  ON storage.objects FOR INSERT
  WITH CHECK (
    bucket_id = 'inspection-images'
    AND auth.uid()::text = (storage.foldername(name))[1]
  );

CREATE POLICY "Users can view own images"
  ON storage.objects FOR SELECT
  USING (
    bucket_id = 'inspection-images'
    AND auth.uid()::text = (storage.foldername(name))[1]
  );
```

---

## Phase 3: Authentication Implementation

### 3.1 Supabase Client Setup

**File**: `src/lib/supabase.ts`
```typescript
import { createClientComponentClient } from '@supabase/auth-helpers-nextjs'
import { createServerComponentClient } from '@supabase/auth-helpers-nextjs'
import { cookies } from 'next/headers'

// Client component (browser)
export const createClient = () => createClientComponentClient()

// Server component (server-side)
export const createServerClient = () => createServerComponentClient({ cookies })
```

### 3.2 Auth Pages

**Login Page**: `src/app/login/page.tsx`
- Email/password login
- "Sign up" link
- Redirect to dashboard on success

**Signup Page**: `src/app/signup/page.tsx`
- Email/password registration
- Supabase Auth UI component
- Email verification flow

### 3.3 Protected Routes Middleware

**File**: `src/middleware.ts`
```typescript
import { createMiddlewareClient } from '@supabase/auth-helpers-nextjs'
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

export async function middleware(req: NextRequest) {
  const res = NextResponse.next()
  const supabase = createMiddlewareClient({ req, res })

  const {
    data: { session },
  } = await supabase.auth.getSession()

  // Redirect to login if not authenticated
  if (!session && req.nextUrl.pathname.startsWith('/dashboard')) {
    return NextResponse.redirect(new URL('/login', req.url))
  }

  return res
}

export const config = {
  matcher: ['/dashboard/:path*']
}
```

---

## Phase 4: Core Application Features

### 4.1 Application Structure

```
src/app/
├── layout.tsx              # Root layout with auth provider
├── page.tsx                # Public landing page
├── login/page.tsx          # Login page
├── signup/page.tsx         # Signup page
├── dashboard/
│   ├── layout.tsx          # Dashboard layout (protected)
│   ├── page.tsx            # Inspection list/history
│   ├── new/page.tsx        # New inspection form
│   └── [id]/page.tsx       # View inspection details
└── api/
    ├── inspections/
    │   ├── route.ts        # GET, POST inspections
    │   └── [id]/route.ts   # GET, PUT, DELETE by ID
    └── upload/route.ts     # Image upload endpoint
```

### 4.2 Key Components

**1. InspectionForm Component**
- State dropdown (11 states)
- System type dropdown (6 types)
- Building type dropdown (5 types)
- Camera access button
- Capture image button
- Preview captured image
- Submit button

**2. CameraCapture Component**
- Request camera permission
- Live video stream preview
- Capture button
- Retake functionality
- Upload to Supabase Storage

**3. ComplianceResults Component**
- Display compliance checks in table
- Color-coded status (pass/fail/warning)
- Expandable notes
- Print/export functionality

**4. InspectionHistory Component**
- List of past inspections
- Filter by date, status, system type
- Search functionality
- Click to view details

### 4.3 Compliance Rules Management

**File**: `src/lib/complianceRules.ts`
```typescript
export const complianceRules = {
  CA: {
    sprinkler: {
      name: "Sprinkler System",
      rules: [
        {
          id: "CA-SPR-001",
          check: "Visual inspection completed",
          required: true
        },
        // ... more rules
      ]
    },
    // ... other system types
  },
  // ... other states
}
```

### 4.4 API Routes

**POST /api/inspections** - Create new inspection
```typescript
// 1. Verify user authentication
// 2. Validate input data
// 3. Upload image to Supabase Storage
// 4. Insert inspection record
// 5. Insert compliance results
// 6. Return inspection ID
```

**GET /api/inspections** - List user's inspections
```typescript
// 1. Verify authentication
// 2. Query inspections WHERE user_id = current_user OR current_user IN shared_with
// 3. Return paginated results
```

**GET /api/inspections/[id]** - Get inspection details
```typescript
// 1. Verify user has access (RLS handles this)
// 2. Fetch inspection + compliance results (JOIN)
// 3. Get signed URL for image
// 4. Return full inspection data
```

**PUT /api/inspections/[id]** - Update inspection (e.g., share with Ethan)
```typescript
// 1. Verify user owns inspection
// 2. Update shared_with array
// 3. Return updated inspection
```

---

## Phase 5: Deployment Strategy

### 5.1 Vercel Setup

1. **Connect GitHub Repository**
   - Go to https://vercel.com
   - Import `jumpin69.github.io` repository
   - Select root directory (or create separate repo for Next.js app)

2. **Configure Build Settings**
   ```
   Framework Preset: Next.js
   Build Command: npm run build
   Output Directory: .next
   Install Command: npm install
   ```

3. **Environment Variables**
   Add in Vercel dashboard:
   - `NEXT_PUBLIC_SUPABASE_URL`
   - `NEXT_PUBLIC_SUPABASE_ANON_KEY`
   - `SUPABASE_SERVICE_ROLE_KEY`

4. **Deploy**
   - Automatic deployment on push to `main`
   - Preview deployments for PRs
   - Custom domain: `fire-suppression.yourdomain.com`

### 5.2 CI/CD Pipeline

Vercel automatically:
- Builds on every git push
- Runs Next.js build checks
- Deploys to preview URL (PRs)
- Deploys to production (main branch)
- Rolls back on errors

### 5.3 Domain Configuration

**Option 1**: Subdomain
- `fire-compliance.jumpin69.com`

**Option 2**: Root domain
- `fire-compliance-checker.com`

**Option 3**: Vercel subdomain
- `fire-suppression-checker.vercel.app`

---

## Phase 6: Migration Checklist

### Pre-Migration
- [ ] Create Supabase account and project
- [ ] Create Vercel account
- [ ] Decide on repository structure (new repo vs. subdirectory)

### Development Phase
- [ ] Initialize Next.js project
- [ ] Set up Supabase client
- [ ] Run database migrations
- [ ] Create storage bucket
- [ ] Implement authentication pages
- [ ] Build inspection form component
- [ ] Integrate camera capture
- [ ] Migrate compliance rules to TypeScript
- [ ] Create API routes
- [ ] Build dashboard and history views
- [ ] Implement sharing functionality
- [ ] Test locally with Supabase dev environment

### Testing Phase
- [ ] Test user registration/login
- [ ] Test inspection creation
- [ ] Test image upload/retrieval
- [ ] Test compliance results saving
- [ ] Test inspection history view
- [ ] Test sharing between users
- [ ] Test mobile responsiveness
- [ ] Test camera on different devices

### Deployment Phase
- [ ] Push code to GitHub
- [ ] Connect Vercel to repository
- [ ] Configure environment variables
- [ ] Deploy to production
- [ ] Test production deployment
- [ ] Configure custom domain (optional)

### Post-Deployment
- [ ] Monitor Supabase usage
- [ ] Set up error tracking (Sentry, optional)
- [ ] Create user documentation
- [ ] Train Ethan on system usage

---

## Phase 7: Future Enhancements (Commercial Features)

### Tier 1: Basic Commercial Features
- [ ] Export reports to PDF
- [ ] Email notifications for shared inspections
- [ ] Inspection templates
- [ ] Bulk image upload

### Tier 2: Advanced Features
- [ ] AI-powered image analysis (OpenAI Vision API)
- [ ] Team accounts (multiple users per organization)
- [ ] Role-based permissions (admin, inspector, viewer)
- [ ] Scheduled inspections and reminders
- [ ] Mobile app (React Native)

### Tier 3: Enterprise Features
- [ ] Multi-tenant architecture
- [ ] Custom compliance rules per organization
- [ ] API access for third-party integrations
- [ ] Advanced analytics and reporting
- [ ] SSO/SAML authentication
- [ ] Audit logs

---

## Cost Estimates

### Development Phase (Free Tier)
- **Supabase**: Free up to 500MB database, 1GB storage, 50,000 monthly active users
- **Vercel**: Free for personal projects, unlimited deployments
- **GitHub**: Free public/private repositories

### Production Phase (If Scaling)
- **Supabase Pro**: $25/month (8GB database, 100GB storage)
- **Vercel Pro**: $20/month (unlimited bandwidth, better support)

### Commercial Phase
- **Supabase Pro/Team**: $25-599/month (based on usage)
- **Vercel Team/Enterprise**: $20+/month per seat

---

## Repository Structure Recommendation

### Option 1: Monorepo (Recommended)
Keep everything in `jumpin69.github.io`:
```
jumpin69.github.io/
├── fire-compliance-checker/    # Next.js app (new)
├── Index.html                   # Keep for portfolio
└── fire-compliance-checker.html # Archive (legacy)
```

### Option 2: Separate Repository
Create new repo: `fire-suppression-checker`
- Cleaner separation
- Dedicated CI/CD
- Easier to open-source or sell later

**Recommendation**: Use Option 2 for commercial readiness.

---

## Next Steps

1. **Confirm Approach**: Review this plan and confirm the technical stack
2. **Create Supabase Project**: Set up database and storage
3. **Initialize Next.js App**: Start coding the new application
4. **Migrate Features**: Port existing functionality to new stack
5. **Deploy to Vercel**: Go live!

**Estimated Timeline**:
- Setup & Auth: 2-3 hours
- Core Features: 4-6 hours
- Testing & Deployment: 2-3 hours
- **Total**: 8-12 hours of development

---

## Questions to Resolve

1. **Repository**: New repo or keep in `jumpin69.github.io`?
2. **Domain**: Custom domain or Vercel subdomain?
3. **AI Analysis**: Keep mock analysis or integrate real AI (adds cost)?
4. **User Onboarding**: Should Ethan's account be pre-created or self-signup?

Let me know your preferences and we'll start building! 🚀
