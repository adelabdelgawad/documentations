# Next.js Page Architecture Plan Template

A general-purpose implementation guide for creating new pages in any Next.js App Router project following best practices.

---

## Quick Reference Checklist

Before implementing, verify each pattern:

- [ ] `page.tsx` is a Server Component (no `'use client'`)
- [ ] Initial data fetched in `page.tsx` via server functions
- [ ] Data passed to Client Component as `initialData` prop
- [ ] Context created for shared state (if 2+ components need same state)
- [ ] API routes created as proxy (client never calls backend directly)
- [ ] SWR configured with `fallbackData` and `revalidate: false` after mutations
- [ ] Custom hooks encapsulate all data fetching/mutation logic
- [ ] Backend mutations return the updated record (not just success/failure)

---

## 1. Folder Structure Template

```
app/(group-name)/feature-name/
├── page.tsx                    # Server Component - data fetching entry
├── _components/
│   ├── feature-body.tsx        # Main client component wrapper
│   ├── feature-form.tsx        # Form component (if applicable)
│   ├── feature-table.tsx       # Table/list component (if applicable)
│   └── feature-modal.tsx       # Modal component (if applicable)
├── context/
│   └── feature-context.tsx     # Shared state context
└── hooks/
    └── use-feature.ts          # Custom data hook (optional, can be in /hooks)

app/api/feature-name/
├── route.ts                    # GET (list), POST (create)
└── [id]/
    └── route.ts                # GET, PUT, DELETE (single item)

lib/
├── actions/
│   └── feature.actions.ts      # Server actions for data fetching
└── api/
    └── feature.ts              # Client-side API functions (optional)

types/
└── feature.types.ts            # TypeScript interfaces
```

---

## 2. Implementation Steps

### Step 1: Define Types

**File: `types/feature.types.ts`**

```typescript
// Use camelCase for frontend types
export interface Feature {
  id: number;
  name: string;
  description: string;
  isActive: boolean;
  createdAt: string;
  updatedAt: string;
}

export interface FeatureCreatePayload {
  name: string;
  description: string;
}

export interface FeatureUpdatePayload {
  name?: string;
  description?: string;
  isActive?: boolean;
}

export interface PaginatedFeatures {
  items: Feature[];
  total: number;
  page: number;
  pageSize: number;
}
```

---

### Step 2: Create Server Action for Data Fetching

**File: `lib/actions/feature.actions.ts`**

```typescript
'use server';

import { serverApi } from '@/lib/http/axios-server'; // Or your HTTP client

export async function getFeatures(filters?: {
  page?: number;
  pageSize?: number;
  search?: string;
}): Promise<PaginatedFeatures> {
  try {
    const response = await serverApi.get('/features', {
      params: {
        page: filters?.page ?? 1,
        page_size: filters?.pageSize ?? 10, // snake_case for backend
        search: filters?.search,
      },
    });

    if (!response.ok) {
      // Return empty fallback on error
      return { items: [], total: 0, page: 1, pageSize: 10 };
    }

    return response.data;
  } catch (error) {
    console.error('Failed to fetch features:', error);
    return { items: [], total: 0, page: 1, pageSize: 10 };
  }
}

export async function getFeatureById(id: number): Promise<Feature | null> {
  try {
    const response = await serverApi.get(`/features/${id}`);
    return response.ok ? response.data : null;
  } catch (error) {
    console.error('Failed to fetch feature:', error);
    return null;
  }
}
```

---

### Step 3: Create Page (Server Component)

**File: `app/(pages)/feature-name/page.tsx`**

```typescript
// NO 'use client' - this is a Server Component
import { getFeatures } from '@/lib/actions/feature.actions';
import { FeatureBody } from './_components/feature-body';

interface PageProps {
  searchParams: Promise<{
    page?: string;
    search?: string;
  }>;
}

export default async function FeaturePage({ searchParams }: PageProps) {
  // IMPORTANT: Always await searchParams in Next.js 14+
  const params = await searchParams;

  const page = params.page ? parseInt(params.page, 10) : 1;
  const search = params.search || '';

  // Fetch data on server
  const initialData = await getFeatures({
    page,
    pageSize: 10,
    search,
  });

  // Pass to client component
  return (
    <div className="container mx-auto p-4">
      <FeatureBody initialData={initialData} />
    </div>
  );
}
```

---

### Step 4: Create API Routes (Proxy)

**File: `app/api/feature-name/route.ts`**

```typescript
import { NextRequest, NextResponse } from 'next/server';
import { serverApi } from '@/lib/http/axios-server';

// GET /api/feature-name - List features
export async function GET(request: NextRequest) {
  try {
    const { searchParams } = request.nextUrl;
    const params: Record<string, string> = {};

    // Forward all query params to backend
    searchParams.forEach((value, key) => {
      params[key] = value;
    });

    const result = await serverApi.get('/features', { params });

    if (!result.ok) {
      return NextResponse.json(
        { ok: false, error: result.error },
        { status: result.status }
      );
    }

    return NextResponse.json({ ok: true, data: result.data });
  } catch (error) {
    return NextResponse.json(
      { ok: false, error: 'Server error' },
      { status: 500 }
    );
  }
}

// POST /api/feature-name - Create feature
export async function POST(request: NextRequest) {
  try {
    const body = await request.json();

    const result = await serverApi.post('/features', body);

    if (!result.ok) {
      return NextResponse.json(
        { ok: false, error: result.error },
        { status: result.status }
      );
    }

    // CRITICAL: Return the created record from backend
    return NextResponse.json(
      { ok: true, data: result.data },
      { status: 201 }
    );
  } catch (error) {
    return NextResponse.json(
      { ok: false, error: 'Server error' },
      { status: 500 }
    );
  }
}
```

**File: `app/api/feature-name/[id]/route.ts`**

```typescript
import { NextRequest, NextResponse } from 'next/server';
import { serverApi } from '@/lib/http/axios-server';

interface RouteParams {
  params: Promise<{ id: string }>;
}

// GET /api/feature-name/[id]
export async function GET(request: NextRequest, { params }: RouteParams) {
  const { id } = await params;

  try {
    const result = await serverApi.get(`/features/${id}`);

    if (!result.ok) {
      return NextResponse.json(
        { ok: false, error: result.error },
        { status: result.status }
      );
    }

    return NextResponse.json({ ok: true, data: result.data });
  } catch (error) {
    return NextResponse.json(
      { ok: false, error: 'Server error' },
      { status: 500 }
    );
  }
}

// PUT /api/feature-name/[id]
export async function PUT(request: NextRequest, { params }: RouteParams) {
  const { id } = await params;

  try {
    const body = await request.json();
    const result = await serverApi.put(`/features/${id}`, body);

    if (!result.ok) {
      return NextResponse.json(
        { ok: false, error: result.error },
        { status: result.status }
      );
    }

    // CRITICAL: Return the updated record
    return NextResponse.json({ ok: true, data: result.data });
  } catch (error) {
    return NextResponse.json(
      { ok: false, error: 'Server error' },
      { status: 500 }
    );
  }
}

// DELETE /api/feature-name/[id]
export async function DELETE(request: NextRequest, { params }: RouteParams) {
  const { id } = await params;

  try {
    const result = await serverApi.delete(`/features/${id}`);

    if (!result.ok) {
      return NextResponse.json(
        { ok: false, error: result.error },
        { status: result.status }
      );
    }

    // Return deleted record or success confirmation
    return NextResponse.json({ ok: true, data: result.data });
  } catch (error) {
    return NextResponse.json(
      { ok: false, error: 'Server error' },
      { status: 500 }
    );
  }
}
```

---

### Step 5: Create Context (If Needed)

**File: `app/(pages)/feature-name/context/feature-context.tsx`**

```typescript
'use client';

import { createContext, useContext, useState, ReactNode } from 'react';
import { Feature } from '@/types/feature.types';

interface FeatureContextType {
  // Shared UI state
  selectedFeature: Feature | null;
  setSelectedFeature: (feature: Feature | null) => void;
  isModalOpen: boolean;
  setIsModalOpen: (open: boolean) => void;

  // Shared filters
  searchQuery: string;
  setSearchQuery: (query: string) => void;
}

const FeatureContext = createContext<FeatureContextType | null>(null);

interface FeatureProviderProps {
  children: ReactNode;
}

export function FeatureProvider({ children }: FeatureProviderProps) {
  const [selectedFeature, setSelectedFeature] = useState<Feature | null>(null);
  const [isModalOpen, setIsModalOpen] = useState(false);
  const [searchQuery, setSearchQuery] = useState('');

  const value: FeatureContextType = {
    selectedFeature,
    setSelectedFeature,
    isModalOpen,
    setIsModalOpen,
    searchQuery,
    setSearchQuery,
  };

  return (
    <FeatureContext.Provider value={value}>
      {children}
    </FeatureContext.Provider>
  );
}

export function useFeatureContext() {
  const context = useContext(FeatureContext);
  if (!context) {
    throw new Error('useFeatureContext must be used within FeatureProvider');
  }
  return context;
}
```

**When to use Context:**
- UI state shared between siblings (selected item, modal open/close)
- Filter state that affects multiple components
- Temporary form state across steps

**When NOT to use Context:**
- Server data (use SWR instead)
- Data only one component needs (use local state)
- Static configuration

---

### Step 6: Create Custom Hook with SWR

**File: `hooks/use-features.ts`**

```typescript
'use client';

import useSWR from 'swr';
import { clientApi } from '@/lib/http/axios-client';
import { Feature, PaginatedFeatures, FeatureCreatePayload, FeatureUpdatePayload } from '@/types/feature.types';

interface UseFeaturesOptions {
  initialData?: PaginatedFeatures;
  page?: number;
  pageSize?: number;
  search?: string;
}

// Fetcher function
const fetcher = async (url: string): Promise<PaginatedFeatures> => {
  const response = await clientApi.get<PaginatedFeatures>(url);
  if (!response.ok) {
    throw new Error(response.error || 'Failed to fetch');
  }
  return response.data;
};

export function useFeatures(options: UseFeaturesOptions = {}) {
  const { initialData, page = 1, pageSize = 10, search = '' } = options;

  // Build API URL
  const params = new URLSearchParams();
  params.append('page', page.toString());
  params.append('page_size', pageSize.toString());
  if (search) params.append('search', search);

  const apiUrl = `/feature-name?${params.toString()}`;

  // SWR hook
  const { data, error, isLoading, isValidating, mutate } = useSWR<PaginatedFeatures>(
    apiUrl,
    fetcher,
    {
      fallbackData: initialData,        // Use server data as initial
      revalidateOnMount: false,         // Don't refetch on mount (we have initialData)
      revalidateOnFocus: false,         // Optional: don't refetch on window focus
      keepPreviousData: true,           // Smooth pagination transitions
    }
  );

  // CREATE - Add new feature
  const createFeature = async (payload: FeatureCreatePayload): Promise<Feature> => {
    const response = await clientApi.post<Feature>('/feature-name', payload);

    if (!response.ok) {
      throw new Error(response.error || 'Failed to create');
    }

    // Update cache with new item (no refetch)
    await mutate(
      (currentData) => {
        if (!currentData) return currentData;
        return {
          ...currentData,
          items: [response.data, ...currentData.items],
          total: currentData.total + 1,
        };
      },
      { revalidate: false }  // CRITICAL: Don't refetch
    );

    return response.data;
  };

  // UPDATE - Modify existing feature
  const updateFeature = async (id: number, payload: FeatureUpdatePayload): Promise<Feature> => {
    const response = await clientApi.put<Feature>(`/feature-name/${id}`, payload);

    if (!response.ok) {
      throw new Error(response.error || 'Failed to update');
    }

    // Update cache with modified item (no refetch)
    await mutate(
      (currentData) => {
        if (!currentData) return currentData;
        return {
          ...currentData,
          items: currentData.items.map((item) =>
            item.id === id ? response.data : item
          ),
        };
      },
      { revalidate: false }  // CRITICAL: Don't refetch
    );

    return response.data;
  };

  // DELETE - Remove feature
  const deleteFeature = async (id: number): Promise<void> => {
    const response = await clientApi.delete(`/feature-name/${id}`);

    if (!response.ok) {
      throw new Error(response.error || 'Failed to delete');
    }

    // Remove from cache (no refetch)
    await mutate(
      (currentData) => {
        if (!currentData) return currentData;
        return {
          ...currentData,
          items: currentData.items.filter((item) => item.id !== id),
          total: currentData.total - 1,
        };
      },
      { revalidate: false }  // CRITICAL: Don't refetch
    );
  };

  return {
    // Data
    features: data?.items ?? [],
    total: data?.total ?? 0,

    // States
    isLoading,
    isValidating,
    error,

    // Mutations
    createFeature,
    updateFeature,
    deleteFeature,

    // Manual revalidation (if needed)
    refresh: () => mutate(),
  };
}
```

---

### Step 7: Create Client Component (Page Body)

**File: `app/(pages)/feature-name/_components/feature-body.tsx`**

```typescript
'use client';

import { useFeatures } from '@/hooks/use-features';
import { FeatureProvider, useFeatureContext } from '../context/feature-context';
import { FeatureTable } from './feature-table';
import { FeatureModal } from './feature-modal';
import { PaginatedFeatures } from '@/types/feature.types';

interface FeatureBodyProps {
  initialData: PaginatedFeatures;
}

export function FeatureBody({ initialData }: FeatureBodyProps) {
  return (
    <FeatureProvider>
      <FeatureBodyContent initialData={initialData} />
    </FeatureProvider>
  );
}

function FeatureBodyContent({ initialData }: FeatureBodyProps) {
  const { isModalOpen, setIsModalOpen, selectedFeature } = useFeatureContext();

  const {
    features,
    total,
    isLoading,
    createFeature,
    updateFeature,
    deleteFeature,
  } = useFeatures({ initialData });

  const handleCreate = async (data: FeatureCreatePayload) => {
    try {
      await createFeature(data);
      setIsModalOpen(false);
      // Show success toast
    } catch (error) {
      // Show error toast
    }
  };

  const handleUpdate = async (id: number, data: FeatureUpdatePayload) => {
    try {
      await updateFeature(id, data);
      setIsModalOpen(false);
      // Show success toast
    } catch (error) {
      // Show error toast
    }
  };

  const handleDelete = async (id: number) => {
    try {
      await deleteFeature(id);
      // Show success toast
    } catch (error) {
      // Show error toast
    }
  };

  return (
    <div className="space-y-4">
      <header className="flex justify-between items-center">
        <h1 className="text-2xl font-bold">Features</h1>
        <button onClick={() => setIsModalOpen(true)}>
          Add Feature
        </button>
      </header>

      <FeatureTable
        features={features}
        isLoading={isLoading}
        onEdit={(feature) => {
          // Set selected and open modal
        }}
        onDelete={handleDelete}
      />

      <FeatureModal
        open={isModalOpen}
        onClose={() => setIsModalOpen(false)}
        feature={selectedFeature}
        onSubmit={selectedFeature ? handleUpdate : handleCreate}
      />
    </div>
  );
}
```

---

## 3. Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        USER REQUEST                              │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                    page.tsx (Server Component)                   │
│  • Await searchParams                                           │
│  • Call server action to fetch initial data                     │
│  • Pass initialData to client component                         │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                  feature-body.tsx (Client Component)             │
│  • Wrap with Context Provider                                   │
│  • Use custom hook with initialData as fallback                 │
│  • Render UI components                                         │
└─────────────────────────────────────────────────────────────────┘
                                │
              ┌─────────────────┴─────────────────┐
              ▼                                   ▼
┌──────────────────────────┐       ┌──────────────────────────────┐
│    Context Provider      │       │    useFeatures Hook (SWR)    │
│  • UI state (selected)   │       │  • Data fetching             │
│  • Modal state           │       │  • Cache management          │
│  • Filter state          │       │  • Mutations                 │
└──────────────────────────┘       └──────────────────────────────┘
                                                  │
                                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                    /api/feature-name (API Route)                 │
│  • Receives request from client                                 │
│  • Forwards to backend                                          │
│  • Returns response to client                                   │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                         BACKEND API                              │
│  • Process request                                              │
│  • Return updated record (CRITICAL for mutations)               │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4. Mutation Flow (No Full Refetch)

```
User Action (Create/Update/Delete)
         │
         ▼
┌─────────────────────┐
│   Call API Route    │
│   via clientApi     │
└─────────────────────┘
         │
         ▼
┌─────────────────────┐
│   API Route Proxy   │
│   → Backend API     │
└─────────────────────┘
         │
         ▼
┌─────────────────────┐
│  Backend Returns    │
│  Updated Record     │  ← CRITICAL: Not just success/failure
└─────────────────────┘
         │
         ▼
┌─────────────────────┐
│   SWR mutate()      │
│   with response     │
│   revalidate: false │  ← CRITICAL: Don't refetch
└─────────────────────┘
         │
         ▼
┌─────────────────────┐
│   UI Updates        │
│   Instantly         │
└─────────────────────┘
```

---

## 5. Decision Tree

| Scenario | Solution |
|----------|----------|
| Fetching initial page data | Do it in `page.tsx` (Server Component) |
| Component needs state that sibling also needs | Put it in Context |
| Calling backend from client component | Create/use API route as proxy |
| After successful create/update/delete | Update SWR cache with response, `revalidate: false` |
| Multiple components need same server data | Use SWR with shared key (auto-deduplicates) |
| Data only one component needs | Local `useState` |
| Loading states between pages | Use `keepPreviousData: true` in SWR |

---

## 6. Anti-Patterns to Avoid

| Don't | Do Instead |
|-------|------------|
| Add `'use client'` to `page.tsx` | Keep page as Server Component |
| Fetch data in `useEffect` on mount | Fetch in page, pass as `initialData` |
| Call backend URL from client | Use `/api/*` routes as proxy |
| Call `mutate()` without data | Pass updated data to `mutate()` |
| Set `revalidate: true` after mutations | Set `revalidate: false` |
| Use `router.refresh()` after mutations | Update SWR cache directly |
| Prop drill through 3+ levels | Use Context for shared state |
| Return only `{ success: true }` from backend | Return the updated record |

---

## 7. Backend Requirements

For this pattern to work, the backend MUST:

1. **CREATE endpoints**: Return the newly created record with generated ID
2. **UPDATE endpoints**: Return the complete updated record
3. **DELETE endpoints**: Return the deleted record or confirmation

Example backend response for POST `/features`:
```json
{
  "id": 123,
  "name": "New Feature",
  "description": "Description",
  "isActive": true,
  "createdAt": "2024-01-15T10:30:00Z",
  "updatedAt": "2024-01-15T10:30:00Z"
}
```

---

## 8. Files to Create (Summary)

For a new page called `feature-name`:

1. `types/feature.types.ts` - TypeScript interfaces
2. `lib/actions/feature.actions.ts` - Server actions
3. `app/(pages)/feature-name/page.tsx` - Server Component page
4. `app/api/feature-name/route.ts` - API route (list/create)
5. `app/api/feature-name/[id]/route.ts` - API route (single item)
6. `app/(pages)/feature-name/context/feature-context.tsx` - Context (if needed)
7. `hooks/use-features.ts` - Custom SWR hook
8. `app/(pages)/feature-name/_components/feature-body.tsx` - Main client component
9. `app/(pages)/feature-name/_components/*.tsx` - Sub-components

---

## 9. Adaptation Notes

When applying to your project:

1. **HTTP Client**: Replace `serverApi`/`clientApi` with your HTTP client (fetch, axios, etc.)
2. **Folder Structure**: Adapt paths to match your project's conventions
3. **Route Groups**: Use your project's route group naming (e.g., `(dashboard)`, `(admin)`)
4. **Naming Conventions**: Follow your project's file/component naming patterns
5. **Authentication**: Add auth headers in API routes if needed
6. **Error Handling**: Use your project's error/toast system
7. **Styling**: Use your project's styling approach (Tailwind, CSS modules, etc.)
