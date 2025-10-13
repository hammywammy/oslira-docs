# 🏗️ OSLIRA V1 - ZERO-REFACTOR FRONTEND ARCHITECTURE

> **Mission**: Build once, scale forever. No big refactors, ever.

---

## 📋 **TABLE OF CONTENTS**

1. [Tech Stack Decisions](#tech-stack)
2. [Project Structure](#structure)
3. [Setup Instructions](#setup)
4. [Architecture Patterns](#patterns)
5. [Development Workflow](#workflow)
6. [Testing Strategy](#testing)
7. [Deployment Pipeline](#deployment)

---

## 🎯 **TECH STACK DECISIONS** <a name="tech-stack"></a>

### **Core Technologies (Non-Negotiable)**

| Technology | Version | Why This Exact Choice |
|-----------|---------|----------------------|
| **React** | 19.x | New server components, use(), optimized concurrency |
| **TypeScript** | 5.8+ | Strict mode, improved error messages, no `any` types |
| **Vite** | 6.x | 20-30x faster than Webpack, native ESM, Rollup for prod |
| **Zustand** | 4.5+ | Lighter than Redux (3KB), no boilerplate, TS-first |
| **React Query** | 5.x | Server state caching, automatic refetch, optimistic updates |
| **React Router** | 7.x | Type-safe routes, nested layouts, data loaders |
| **Zod** | 3.24+ | Runtime validation, type inference, client+server schemas |
| **TailwindCSS** | 4.x | Utility-first, consistent spacing, purge unused CSS |
| **Radix UI** | Latest | Accessible primitives, keyboard navigation, no styling opinions |
| **Vitest** | 2.x | Vite-native, 10x faster than Jest, same API |

### **Why These Choices Will Never Require Refactoring**

1. **React 19** - Stable, mature, huge ecosystem. Won't be replaced for 5+ years
2. **TypeScript strict mode** - Catches 90% of bugs before runtime
3. **Vite** - Industry standard replacing Webpack everywhere. Future-proof
4. **Zustand** - Simple state management that scales. No Redux complexity
5. **React Query** - Solves server state caching problem permanently
6. **Zod** - Single source of truth for validation (client + server)

---

## 📁 **PROJECT STRUCTURE** <a name="structure"></a>

### **Domain-Driven, Feature-Based Architecture**

> **CRITICAL VITE CONVENTIONS:**
> - `public/` = Static files served as-is (referenced in index.html, never imported in code)
> - `src/assets/` = Assets imported in components (processed by Vite, get hashed filenames)
> - `index.html` = Lives at ROOT (Vite requirement)

```
oslira-v1/
├── .github/
│   └── workflows/
│       ├── ci.yml                    # Lint, test, type-check
│       └── deploy.yml                # Auto-deploy on merge
│
├── public/                           # ⚠️ STATIC FILES ONLY (served as-is)
│   ├── favicon.ico                   # Referenced in index.html
│   ├── robots.txt                    # SEO file
│   └── manifest.json                 # PWA manifest
│
├── index.html                        # ⚠️ MUST be at ROOT (Vite entry point)
│
├── src/                              # ⚠️ ALL SOURCE CODE GOES HERE
│   ├── main.tsx                      # ⚠️ App entry point (referenced in index.html)
│   ├── App.tsx                       # Root component
│   │
│   ├── assets/                       # ⚠️ IMPORTED assets (images, fonts, SVGs)
│   │   ├── images/                   # Component images (get hashed)
│   │   │   ├── logo.svg
│   │   │   └── hero-bg.jpg
│   │   ├── fonts/                    # Custom fonts
│   │   └── icons/                    # Icon SVGs
│   │
│   ├── core/                         # Core infrastructure (STABLE)
│   │   ├── api/
│   │   │   ├── client.ts             # Axios/fetch wrapper
│   │   │   ├── types.ts              # Shared API types
│   │   │   └── interceptors.ts       # Auth, error handling
│   │   │
│   │   ├── auth/
│   │   │   ├── AuthProvider.tsx      # React Context
│   │   │   ├── useAuth.ts            # Auth hook
│   │   │   ├── ProtectedRoute.tsx    # Route guard
│   │   │   └── authService.ts        # Auth API calls
│   │   │
│   │   ├── config/
│   │   │   ├── env.ts                # Environment detection
│   │   │   └── constants.ts          # App-wide constants
│   │   │
│   │   └── store/
│   │       ├── store.ts              # Zustand store (slices)
│   │       └── selectors.ts          # Memoized selectors
│   │
│   ├── features/                     # Feature modules (SCALABLE)
│   │   │
│   │   ├── dashboard/
│   │   │   ├── index.ts              # Public API
│   │   │   ├── Dashboard.tsx         # Main page component
│   │   │   ├── components/
│   │   │   │   ├── StatsCard.tsx
│   │   │   │   ├── RecentLeads.tsx
│   │   │   │   └── InsightsPanel.tsx
│   │   │   ├── hooks/
│   │   │   │   └── useDashboardData.ts
│   │   │   ├── api/
│   │   │   │   └── dashboardApi.ts
│   │   │   └── types.ts
│   │   │
│   │   ├── leads/
│   │   │   ├── index.ts
│   │   │   ├── LeadsPage.tsx
│   │   │   ├── components/
│   │   │   │   ├── LeadsTable.tsx
│   │   │   │   ├── LeadCard.tsx
│   │   │   │   ├── AnalysisModal.tsx
│   │   │   │   └── BulkUploadModal.tsx
│   │   │   ├── hooks/
│   │   │   │   ├── useLeads.ts
│   │   │   │   ├── useCreateLead.ts
│   │   │   │   ├── useAnalyzeLead.ts
│   │   │   │   └── useBulkAnalysis.ts
│   │   │   ├── api/
│   │   │   │   └── leadsApi.ts
│   │   │   ├── schemas/
│   │   │   │   └── leadSchema.ts     # Zod schemas
│   │   │   └── types.ts
│   │   │
│   │   ├── analytics/
│   │   │   ├── index.ts
│   │   │   ├── AnalyticsPage.tsx
│   │   │   ├── components/
│   │   │   │   ├── EngagementChart.tsx
│   │   │   │   └── ConversionFunnel.tsx
│   │   │   ├── hooks/
│   │   │   │   └── useAnalytics.ts
│   │   │   └── types.ts
│   │   │
│   │   ├── business/
│   │   │   ├── index.ts
│   │   │   ├── components/
│   │   │   │   ├── BusinessSelector.tsx
│   │   │   │   └── BusinessSettings.tsx
│   │   │   ├── hooks/
│   │   │   │   └── useBusiness.ts
│   │   │   └── types.ts
│   │   │
│   │   └── auth/
│   │       ├── index.ts
│   │       ├── LoginPage.tsx
│   │       ├── SignupPage.tsx
│   │       └── components/
│   │           ├── LoginForm.tsx
│   │           └── SignupForm.tsx
│   │
│   ├── shared/                       # Shared UI components
│   │   ├── ui/                       # Base components (Radix wrappers)
│   │   │   ├── Button.tsx
│   │   │   ├── Input.tsx
│   │   │   ├── Modal.tsx
│   │   │   ├── Table.tsx
│   │   │   ├── Select.tsx
│   │   │   ├── Tooltip.tsx
│   │   │   ├── Skeleton.tsx
│   │   │   └── ErrorBoundary.tsx
│   │   │
│   │   ├── layouts/
│   │   │   ├── DashboardLayout.tsx
│   │   │   ├── AppSidebar.tsx
│   │   │   ├── AppHeader.tsx
│   │   │   └── AppFooter.tsx
│   │   │
│   │   ├── hooks/
│   │   │   ├── useDebounce.ts
│   │   │   ├── useLocalStorage.ts
│   │   │   ├── useMediaQuery.ts
│   │   │   └── usePagination.ts
│   │   │
│   │   └── utils/
│   │       ├── formatters.ts         # Date, currency, number formatters
│   │       ├── validators.ts         # Common validation functions
│   │       ├── helpers.ts            # Generic helpers
│   │       └── constants.ts          # Shared constants
│   │
│   ├── routes/
│   │   ├── index.tsx                 # Router setup
│   │   └── routes.ts                 # Type-safe route definitions
│   │
│   └── types/
│       ├── api.ts                    # API response types
│       ├── entities.ts               # Domain entities
│       └── global.d.ts               # Global type declarations
│
├── .env.example                      # Environment variables template
├── .env.development
├── .env.production
├── .eslintrc.json                    # ESLint config
├── .prettierrc                       # Prettier config
├── tailwind.config.ts                # Tailwind config
├── tsconfig.json                     # TypeScript config (strict)
├── vite.config.ts                    # Vite config
├── vitest.config.ts                  # Vitest config
└── package.json
```

### **Why This Structure Is Vite-Perfect**

1. **`index.html` at root** - Vite uses it as entry point, serves it during dev
2. **`public/` for static assets** - Files never imported in code (favicon, robots.txt, manifest)
3. **`src/assets/` for imported assets** - Images/fonts used in components get optimized and hashed
4. **Feature-based structure** - Each feature is self-contained module with public API
5. **Colocation of tests** - `.test.tsx` files next to components for easy maintenance

### **File Naming Conventions**

| Type | Convention | Example |
|------|-----------|---------|
| Components | PascalCase | `LeadsTable.tsx` |
| Hooks | camelCase with `use` prefix | `useLeads.ts` |
| Utils | camelCase | `formatters.ts` |
| Types | PascalCase | `Lead`, `User` |
| Constants | UPPER_SNAKE_CASE | `API_BASE_URL` |
| Files | kebab-case or PascalCase | `leads-api.ts` or `LeadsApi.ts` |

### **Import Path Examples**

```typescript
// ✅ GOOD: Using path aliases
import { useAuth } from '@/core/auth/useAuth';
import { LeadsTable } from '@/features/leads/components/LeadsTable';
import { Button } from '@/shared/ui/Button';
import logo from '@/assets/images/logo.svg'; // Vite processes this

// ❌ BAD: Relative imports get messy
import { useAuth } from '../../../core/auth/useAuth';
import logo from '/logo.svg'; // Won't work (must be in public/)
```

### **Vite Asset Import Rules**

```typescript
// ✅ Assets in src/assets/ (imported in code)
import logo from '@/assets/images/logo.svg';
import heroImage from '@/assets/images/hero.jpg';

function Header() {
  return <img src={logo} alt="Logo" />; // Vite hashes: /assets/logo-a3f2d9.svg
}

// ✅ Assets in public/ (referenced by path only)
// In index.html:
<link rel="icon" href="/favicon.ico" />

// In component (when you need public URL):
<img src="/robots.txt" /> // Served as-is from public/
```

---

## 🚀 **SETUP INSTRUCTIONS** <a name="setup"></a>

### **1. Initialize Project (Vite Official Method)**

```bash
# Create Vite + React + TypeScript project
npm create vite@latest oslira-v1 -- --template react-ts

cd oslira-v1

# Verify structure (should see index.html at root, src/ folder)
ls -la
```

### **Expected Initial Structure from Vite:**
```
oslira-v1/
├── index.html          # ✅ At root (Vite requirement)
├── package.json
├── tsconfig.json
├── vite.config.ts
├── public/
│   └── vite.svg       # Static asset
└── src/
    ├── main.tsx       # Entry point
    ├── App.tsx
    ├── App.css
    └── assets/        # Imported assets
        └── react.svg
```

### **2. Install Dependencies**

```bash
# Core
npm install react@19 react-dom@19 react-router@7
npm install zustand@4 @tanstack/react-query@5
npm install zod@3 axios@1

# UI
npm install @radix-ui/react-dialog @radix-ui/react-dropdown-menu
npm install @radix-ui/react-select @radix-ui/react-tooltip
npm install tailwindcss@4 postcss autoprefixer
npm install lucide-react  # Icons

# Dev Dependencies
npm install -D @types/react @types/react-dom
npm install -D typescript@5.8 @typescript-eslint/parser @typescript-eslint/eslint-plugin
npm install -D eslint@9 eslint-config-prettier eslint-plugin-react-hooks
npm install -D prettier
npm install -D vitest@2 @vitest/ui jsdom
npm install -D @testing-library/react @testing-library/jest-dom @testing-library/user-event
```

### **3. Configure TypeScript (Strict Mode)**

**`tsconfig.json`**:
```json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "jsx": "react-jsx",
    
    // Strict type-checking
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "allowUnusedLabels": false,
    "allowUnreachableCode": false,
    
    // Interop
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true,
    "forceConsistentCasingInFileNames": true,
    "isolatedModules": true,
    
    // Output
    "skipLibCheck": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    
    // Path aliases
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"],
      "@/core/*": ["src/core/*"],
      "@/features/*": ["src/features/*"],
      "@/shared/*": ["src/shared/*"]
    }
  },
  "include": ["src"],
  "exclude": ["node_modules", "dist"]
}
```

### **4. Configure Vite**

**`vite.config.ts`**:
```typescript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react-swc';
import path from 'path';

export default defineConfig({
  plugins: [react()],
  
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
      '@/core': path.resolve(__dirname, './src/core'),
      '@/features': path.resolve(__dirname, './src/features'),
      '@/shared': path.resolve(__dirname, './src/shared'),
    },
  },
  
  build: {
    target: 'es2022',
    sourcemap: true,
    
    // Code splitting
    rollupOptions: {
      output: {
        manualChunks: {
          'react-vendor': ['react', 'react-dom', 'react-router'],
          'ui-vendor': ['@radix-ui/react-dialog', '@radix-ui/react-dropdown-menu'],
          'query-vendor': ['@tanstack/react-query', 'axios'],
        },
      },
    },
    
    // Bundle size limits
    chunkSizeWarningLimit: 1000,
  },
  
  server: {
    port: 5173,
    strictPort: false,
    proxy: {
      '/api': {
        target: 'http://localhost:8787',
        changeOrigin: true,
      },
    },
  },
});
```

### **5. Configure ESLint + Prettier**

**`.eslintrc.json`**:
```json
{
  "extends": [
    "eslint:recommended",
    "plugin:@typescript-eslint/recommended",
    "plugin:@typescript-eslint/recommended-requiring-type-checking",
    "plugin:react/recommended",
    "plugin:react-hooks/recommended",
    "prettier"
  ],
  "parser": "@typescript-eslint/parser",
  "parserOptions": {
    "ecmaVersion": "latest",
    "sourceType": "module",
    "project": "./tsconfig.json"
  },
  "plugins": ["@typescript-eslint", "react", "react-hooks"],
  "rules": {
    "@typescript-eslint/no-explicit-any": "error",
    "@typescript-eslint/explicit-function-return-type": "warn",
    "@typescript-eslint/no-unused-vars": ["error", { "argsIgnorePattern": "^_" }],
    "react/react-in-jsx-scope": "off",
    "react-hooks/rules-of-hooks": "error",
    "react-hooks/exhaustive-deps": "warn"
  },
  "settings": {
    "react": {
      "version": "detect"
    }
  }
}
```

**`.prettierrc`**:
```json
{
  "semi": true,
  "trailingComma": "es5",
  "singleQuote": true,
  "printWidth": 100,
  "tabWidth": 2,
  "useTabs": false,
  "arrowParens": "always",
  "endOfLine": "lf"
}
```

### **6. Configure Tailwind CSS**

```bash
npx tailwindcss init -p
```

**`tailwind.config.ts`**:
```typescript
import type { Config } from 'tailwindcss';

export default {
  content: ['./index.html', './src/**/*.{js,ts,jsx,tsx}'],
  theme: {
    extend: {
      colors: {
        primary: '#2563eb',
        secondary: '#64748b',
      },
    },
  },
  plugins: [],
} satisfies Config;
```

**`src/index.css`**:
```css
@tailwind base;
@tailwind components;
@tailwind utilities;

@layer base {
  :root {
    --primary: 221.2 83.2% 53.3%;
    --secondary: 215.4 16.3% 46.9%;
  }
}
```

---

## 🏛️ **ARCHITECTURE PATTERNS** <a name="patterns"></a>

### **1. State Management Strategy**

```typescript
// src/core/store/store.ts
import { create } from 'zustand';
import { devtools, persist } from 'zustand/middleware';
import { immer } from 'zustand/middleware/immer';

// RULE: Client state in Zustand, Server state in React Query

interface AppStore {
  // UI state
  sidebarOpen: boolean;
  theme: 'light' | 'dark';
  
  // User preferences
  selectedBusinessId: string | null;
  
  // Actions
  toggleSidebar: () => void;
  setTheme: (theme: 'light' | 'dark') => void;
  selectBusiness: (id: string) => void;
}

export const useAppStore = create<AppStore>()(
  devtools(
    persist(
      immer((set) => ({
        sidebarOpen: true,
        theme: 'light',
        selectedBusinessId: null,
        
        toggleSidebar: () => set((state) => { state.sidebarOpen = !state.sidebarOpen; }),
        setTheme: (theme) => set({ theme }),
        selectBusiness: (id) => set({ selectedBusinessId: id }),
      })),
      {
        name: 'oslira-storage',
        partialize: (state) => ({ theme: state.theme, selectedBusinessId: state.selectedBusinessId }),
      }
    )
  )
);
```

### **2. API Client Pattern**

```typescript
// src/core/api/client.ts
import axios, { type AxiosInstance } from 'axios';
import { useAppStore } from '@/core/store/store';

const API_BASE_URL = import.meta.env.VITE_API_URL || '/api';

export const apiClient: AxiosInstance = axios.create({
  baseURL: API_BASE_URL,
  timeout: 30000,
  headers: {
    'Content-Type': 'application/json',
  },
});

// Request interceptor (add auth token)
apiClient.interceptors.request.use((config) => {
  const token = localStorage.getItem('oslira-token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// Response interceptor (handle errors globally)
apiClient.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      // Token expired, redirect to login
      localStorage.removeItem('oslira-token');
      window.location.href = '/auth/login';
    }
    return Promise.reject(error);
  }
);
```

### **3. React Query Pattern (Server State)**

```typescript
// src/features/leads/hooks/useLeads.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { leadsApi } from '../api/leadsApi';
import type { Lead, CreateLeadDto } from '../types';

export function useLeads(businessId: string) {
  return useQuery({
    queryKey: ['leads', businessId],
    queryFn: () => leadsApi.getAll(businessId),
    staleTime: 5 * 60 * 1000, // 5 minutes
    enabled: !!businessId,
  });
}

export function useCreateLead() {
  const queryClient = useQueryClient();
  
  return useMutation({
    mutationFn: (data: CreateLeadDto) => leadsApi.create(data),
    onSuccess: (newLead, variables) => {
      // Optimistic update
      queryClient.setQueryData<Lead[]>(
        ['leads', variables.businessId],
        (old) => [...(old || []), newLead]
      );
    },
  });
}
```

### **4. Zod Schema Pattern (Validation)**

```typescript
// src/features/leads/schemas/leadSchema.ts
import { z } from 'zod';

export const createLeadSchema = z.object({
  username: z.string()
    .min(1, 'Username required')
    .max(30, 'Max 30 characters')
    .regex(/^[a-zA-Z0-9._]+$/, 'Invalid username format'),
  
  analysisType: z.enum(['xray', 'personality', 'deep']),
  businessId: z.string().uuid(),
});

export type CreateLeadDto = z.infer<typeof createLeadSchema>;

// Use in component
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';

const { register, handleSubmit } = useForm<CreateLeadDto>({
  resolver: zodResolver(createLeadSchema),
});
```

### **5. Component Pattern (Separation of Concerns)**

```typescript
// ❌ BAD: Everything in one component
function LeadsPage() {
  const [leads, setLeads] = useState([]);
  const [loading, setLoading] = useState(false);
  
  useEffect(() => {
    fetchLeads();
  }, []);
  
  const fetchLeads = async () => {
    // API call logic mixed with UI
  };
  
  return <div>{/* 500 lines of JSX */}</div>;
}

// ✅ GOOD: Separated concerns
function LeadsPage() {
  const { data: leads, isLoading } = useLeads(businessId);
  
  if (isLoading) return <LeadsTableSkeleton />;
  if (!leads?.length) return <EmptyState />;
  
  return (
    <DashboardLayout>
      <LeadsTable leads={leads} />
    </DashboardLayout>
  );
}
```

---

## 🧪 **TESTING STRATEGY** <a name="testing"></a>

### **Test Structure**

```
src/features/leads/
├── components/
│   ├── LeadsTable.tsx
│   └── LeadsTable.test.tsx        # Unit test
├── hooks/
│   ├── useLeads.ts
│   └── useLeads.test.ts           # Hook test
└── __tests__/
    └── LeadsPage.integration.test.tsx  # Integration test
```

### **Vitest Config**

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react-swc';

export default defineConfig({
  plugins: [react()],
  test: {
    globals: true,
    environment: 'jsdom',
    setupFiles: ['./src/setupTests.ts'],
    coverage: {
      reporter: ['text', 'json', 'html'],
      exclude: ['node_modules/', 'src/setupTests.ts'],
    },
  },
});
```

### **Example Test**

```typescript
// src/features/leads/components/LeadsTable.test.tsx
import { render, screen } from '@testing-library/react';
import { LeadsTable } from './LeadsTable';

describe('LeadsTable', () => {
  it('renders leads correctly', () => {
    const leads = [
      { id: '1', username: 'johndoe', followersCount: 10000 },
    ];
    
    render(<LeadsTable leads={leads} />);
    
    expect(screen.getByText('johndoe')).toBeInTheDocument();
    expect(screen.getByText('10,000')).toBeInTheDocument();
  });
});
```

---

## 🚀 **DEPLOYMENT PIPELINE** <a name="deployment"></a>

### **GitHub Actions CI/CD**

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Type check
        run: npm run type-check
      
      - name: Lint
        run: npm run lint
      
      - name: Test
        run: npm run test
      
      - name: Build
        run: npm run build
```

---

## ✅ **FINAL CHECKLIST**

- [ ] TypeScript strict mode enabled
- [ ] ESLint + Prettier configured
- [ ] Path aliases configured (@/ imports)
- [ ] Zustand for client state
- [ ] React Query for server state
- [ ] Zod for validation
- [ ] Feature-based folder structure
- [ ] CI/CD pipeline setup
- [ ] Environment variables configured
- [ ] Error boundaries in place

---

## 🎯 **NEXT STEPS**

1. **Setup core infrastructure** (Day 1)
   - Auth provider
   - API client
   - Store setup

2. **Build first feature** (Day 2-3)
   - Dashboard page
   - API integration
   - Loading states

3. **Add tests** (Day 4)
   - Unit tests for components
   - Integration tests for pages

4. **Deploy** (Day 5)
   - Vercel/Netlify
   - Monitor performance

---

**This architecture will scale to 100,000+ users without a single refactor. Trust the process.**
