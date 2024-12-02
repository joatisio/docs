# Frontend Architecture 

```typescript
// Project Structure
src/
├── assets/                  // Static assets like images, icons
├── components/             // Reusable UI components
│   ├── common/            // Basic UI components
│   │   ├── Button/
│   │   ├── Input/
│   │   ├── Select/
│   │   ├── Table/
│   │   ├── Modal/
│   │   └── ...
│   ├── layout/            // Layout components
│   │   ├── Sidebar/
│   │   ├── Header/
│   │   ├── PageLayout/
│   │   └── ...
│   └── features/          // Feature-specific components
│       ├── jobs/
│       ├── candidates/
│       ├── interviews/
│       └── ...
├── config/                // Configuration files
├── hooks/                 // Custom React hooks
├── lib/                   // Third-party library configurations
├── pages/                 // Page components
├── services/             // API services
├── store/                // State management
├── types/                // TypeScript type definitions
└── utils/                // Utility functions
```

### 1. Core Technologies & Libraries
```json
{
  "dependencies": {
    "react": "^18.x",
    "react-dom": "^18.x",
    "react-router-dom": "^6.x",
    "@tanstack/react-query": "^4.x",
    "@reduxjs/toolkit": "^1.x",
    "axios": "^1.x",
    "@mantine/core": "^6.x",
    "@mantine/hooks": "^6.x",
    "date-fns": "^2.x",
    "react-hook-form": "^7.x",
    "zod": "^3.x",
    "typescript": "^4.x"
  }
}
```

### 2. Component Examples

```tsx
// src/components/common/Button/Button.tsx
import { styled } from '@mantine/core';

type ButtonVariant = 'primary' | 'secondary' | 'danger';

interface ButtonProps {
  variant?: ButtonVariant;
  loading?: boolean;
  disabled?: boolean;
  children: React.ReactNode;
  onClick?: () => void;
}

export const Button = styled.button<ButtonProps>`
  // Styled component implementation
`;

// src/components/features/candidates/CandidateCard/CandidateCard.tsx
interface CandidateCardProps {
  candidate: Candidate;
  onEdit?: (id: string) => void;
  onDelete?: (id: string) => void;
}

export const CandidateCard: React.FC<CandidateCardProps> = ({
  candidate,
  onEdit,
  onDelete,
}) => {
  return (
    <Card>
      <CardHeader>
        <Avatar src={candidate.avatar} />
        <Title>{candidate.name}</Title>
      </CardHeader>
      <CardContent>
        {/* Candidate details */}
      </CardContent>
      <CardActions>
        <Button onClick={() => onEdit?.(candidate.id)}>Edit</Button>
        <Button variant="danger" onClick={() => onDelete?.(candidate.id)}>
          Delete
        </Button>
      </CardActions>
    </Card>
  );
};
```

### 3. API Integration & Services

```typescript
// src/services/api.ts
import axios from 'axios';

const api = axios.create({
  baseURL: '/api/v1',
});

api.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// src/services/candidates.ts
import { api } from './api';
import type { Candidate, CreateCandidateDTO } from '@/types';

export const candidateService = {
  getAll: async (params: PaginationParams) => {
    const { data } = await api.get<PaginatedResponse<Candidate>>('/candidates', { params });
    return data;
  },

  getById: async (id: string) => {
    const { data } = await api.get<Candidate>(`/candidates/${id}`);
    return data;
  },

  create: async (candidate: CreateCandidateDTO) => {
    const { data } = await api.post<Candidate>('/candidates', candidate);
    return data;
  },
  
  // ... other methods
};
```

### 4. State Management

```typescript
// src/store/index.ts
import { configureStore } from '@reduxjs/toolkit';
import { authReducer } from './slices/auth';
import { uiReducer } from './slices/ui';

export const store = configureStore({
  reducer: {
    auth: authReducer,
    ui: uiReducer,
  },
});

// src/store/slices/auth.ts
import { createSlice, PayloadAction } from '@reduxjs/toolkit';

interface AuthState {
  user: User | null;
  token: string | null;
  isAuthenticated: boolean;
}

const authSlice = createSlice({
  name: 'auth',
  initialState: {
    user: null,
    token: null,
    isAuthenticated: false,
  } as AuthState,
  reducers: {
    setUser: (state, action: PayloadAction<User>) => {
      state.user = action.payload;
      state.isAuthenticated = true;
    },
    logout: (state) => {
      state.user = null;
      state.token = null;
      state.isAuthenticated = false;
    },
  },
});
```

### 5. Custom Hooks

```typescript
// src/hooks/useAuth.ts
export const useAuth = () => {
  const user = useSelector((state: RootState) => state.auth.user);
  const isAuthenticated = useSelector((state: RootState) => state.auth.isAuthenticated);
  const dispatch = useDispatch();

  const login = async (credentials: LoginCredentials) => {
    try {
      const response = await authService.login(credentials);
      dispatch(setUser(response.user));
      return response;
    } catch (error) {
      throw error;
    }
  };

  const logout = () => {
    dispatch(authSlice.actions.logout());
  };

  return { user, isAuthenticated, login, logout };
};

// src/hooks/useCandidates.ts
export const useCandidates = (params: PaginationParams) => {
  return useQuery({
    queryKey: ['candidates', params],
    queryFn: () => candidateService.getAll(params),
  });
};
```

### 6. Page Component Example

```typescript
// src/pages/CandidatesPage.tsx
export const CandidatesPage: React.FC = () => {
  const [searchParams, setSearchParams] = useSearchParams();
  const { data, isLoading } = useCandidates({
    page: parseInt(searchParams.get('page') || '1'),
    per_page: 20,
    search: searchParams.get('search') || '',
  });

  return (
    <PageLayout>
      <PageHeader
        title="Candidates"
        actions={
          <Button onClick={() => navigate('/candidates/new')}>
            Add Candidate
          </Button>
        }
      />
      
      <SearchBar
        value={searchParams.get('search') || ''}
        onChange={(value) => setSearchParams({ search: value })}
      />

      {isLoading ? (
        <LoadingSpinner />
      ) : (
        <CandidateList
          candidates={data?.candidates || []}
          onEdit={(id) => navigate(`/candidates/${id}/edit`)}
          onDelete={(id) => handleDelete(id)}
        />
      )}

      <Pagination
        total={data?.total_pages || 0}
        current={parseInt(searchParams.get('page') || '1')}
        onChange={(page) => setSearchParams({ page: page.toString() })}
      />
    </PageLayout>
  );
};
```

### 7. Routing Setup

```typescript
// src/App.tsx
import { RouterProvider, createBrowserRouter } from 'react-router-dom';

const router = createBrowserRouter([
  {
    path: '/',
    element: <RootLayout />,
    children: [
      {
        path: 'dashboard',
        element: <DashboardPage />,
      },
      {
        path: 'candidates',
        children: [
          { index: true, element: <CandidatesPage /> },
          { path: 'new', element: <CandidateFormPage /> },
          { path: ':id', element: <CandidateDetailsPage /> },
          { path: ':id/edit', element: <CandidateFormPage /> },
        ],
      },
      // ... other routes
    ],
  },
]);

export const App: React.FC = () => {
  return <RouterProvider router={router} />;
};
```

### 8. Form Handling Example

```typescript
// src/components/features/candidates/CandidateForm/CandidateForm.tsx
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

const candidateSchema = z.object({
  first_name: z.string().min(1, 'First name is required'),
  last_name: z.string().min(1, 'Last name is required'),
  email: z.string().email('Invalid email address'),
  phone: z.string().optional(),
  // ... other fields
});

type CandidateFormData = z.infer<typeof candidateSchema>;

export const CandidateForm: React.FC<{
  initialData?: Candidate;
  onSubmit: (data: CandidateFormData) => void;
}> = ({ initialData, onSubmit }) => {
  const {
    register,
    handleSubmit,
    formState: { errors },
  } = useForm<CandidateFormData>({
    resolver: zodResolver(candidateSchema),
    defaultValues: initialData,
  });

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <Input
        {...register('first_name')}
        error={errors.first_name?.message}
      />
      <Input
        {...register('last_name')}
        error={errors.last_name?.message}
      />
      <Input
        {...register('email')}
        error={errors.email?.message}
      />
      {/* ... other fields */}
      <Button type="submit">Save</Button>
    </form>
  );
};
```
