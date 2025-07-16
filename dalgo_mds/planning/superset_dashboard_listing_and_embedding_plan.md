# Superset Dashboard Listing and Embedding Implementation Plan

## Feature Overview
This feature enables Dalgo users to view and interact with their Superset dashboards directly within the Dalgo platform. Users will see a list of their Superset dashboards with search/filter capabilities and can click to view embedded dashboards seamlessly.

## High-Level Architecture Flow

```
User → webapp_v2 → DDP_backend → Superset API
                        ↓
                  Redis Cache (Token Storage)
                        ↓
                  Secrets Manager (Admin Creds)
```

## Token Management Strategy

### Research Findings on Token Handling

Based on extensive research, here are the key findings:

1. **Guest Token Approach is Best Practice**: Guest tokens are specifically designed for embedding purposes and are the recommended approach by Apache Superset for production deployments.

2. **Automatic Token Refresh**: The Superset Embedded SDK automatically handles token refresh by calling the `fetchGuestToken` function when needed. Guest tokens have a default 5-minute expiry.

3. **Token Invalidation Detection**: The SDK doesn't provide explicit events for token expiry, but it automatically calls `fetchGuestToken` before expiry.

### Enhanced Token Management Design

```python
# Backend: Token generation with proper error handling
class SupersetService:
    def __init__(self, org: Org):
        self.org = org
        self.redis = RedisClient.get_instance()
        # Different TTLs for different token types
        self.access_token_ttl = 3600  # 1 hour
        self.csrf_token_ttl = 1800    # 30 minutes
        self.guest_token_ttl = 240    # 4 minutes (refresh before 5 min expiry)
        
    def get_guest_token(self, dashboard_uuid: str, force_refresh=False):
        """
        Generate guest token with automatic cache management
        Frontend will call this through API endpoint
        """
        cache_key = f"superset:token:{self.org.id}:guest:{dashboard_uuid}"
        
        if not force_refresh:
            cached_token = self.redis.get(cache_key)
            if cached_token:
                return json.loads(cached_token)
        
        # Generate new token
        access_token = self.get_access_token()
        csrf_token = self.get_csrf_token(access_token)
        
        # Generate guest token with RLS if needed
        guest_token_data = self._generate_guest_token(
            access_token, csrf_token, dashboard_uuid
        )
        
        # Cache with shorter TTL for security
        self.redis.set(
            cache_key, 
            json.dumps(guest_token_data),
            self.guest_token_ttl
        )
        
        return guest_token_data
```

### Frontend Token Handling

```typescript
// webapp_v2/components/dashboard/superset-embed.tsx
import { embedDashboard } from '@superset-ui/embedded-sdk';
import { apiPost } from '@/lib/api';

const SupersetEmbed = ({ dashboardId, dashboardUuid }) => {
  const mountRef = useRef(null);
  const [error, setError] = useState(null);
  const [isLoading, setIsLoading] = useState(true);
  
  useEffect(() => {
    let unmounted = false;
    
    const fetchGuestToken = async () => {
      try {
        // This function is called automatically by the SDK
        // whenever it needs a fresh token
        const response = await apiPost(
          `/api/superset/dashboards/${dashboardId}/guest_token/`,
          { dashboard_uuid: dashboardUuid }
        );
        
        if (unmounted) return null;
        
        // Return the token for the SDK to use
        return response.guest_token;
      } catch (error) {
        console.error('Failed to fetch guest token:', error);
        setError('Failed to load dashboard. Please try refreshing.');
        throw error; // Let SDK handle the error
      }
    };
    
    // Initialize embedding
    const embed = async () => {
      try {
        setIsLoading(true);
        
        await embedDashboard({
          id: dashboardUuid,
          supersetDomain: window.ENV.SUPERSET_URL,
          mountPoint: mountRef.current,
          fetchGuestToken, // SDK will call this automatically
          dashboardUiConfig: {
            hideTitle: false,
            filters: { expanded: true }
          }
        });
        
        if (!unmounted) {
          setIsLoading(false);
        }
      } catch (error) {
        if (!unmounted) {
          setError('Failed to embed dashboard');
          setIsLoading(false);
        }
      }
    };
    
    embed();
    
    return () => {
      unmounted = true;
      // Clean up embedded iframe
      if (mountRef.current) {
        mountRef.current.innerHTML = '';
      }
    };
  }, [dashboardId, dashboardUuid]);
  
  if (error) {
    return (
      <div className="flex items-center justify-center h-full">
        <Card>
          <CardContent>
            <p className="text-red-500">{error}</p>
            <Button onClick={() => window.location.reload()}>
              Refresh Page
            </Button>
          </CardContent>
        </Card>
      </div>
    );
  }
  
  return (
    <div className="relative w-full h-full">
      {isLoading && (
        <div className="absolute inset-0 flex items-center justify-center">
          <Spinner />
        </div>
      )}
      <div ref={mountRef} className="w-full h-full" />
    </div>
  );
};
```

## Backend Implementation (DDP_backend)

### 1. Superset Service Module
**File to create**: `DDP_backend/ddpui/services/superset_service.py`

This will be the centralized gateway for all Superset communications with robust error handling and token management.

**Key Components**:
- Token management with Redis caching
- Automatic retry with exponential backoff
- 401 error detection and token refresh
- Raise HttpError for proper error handling

**Implementation Blueprint**:
```python
import time
import json
import requests
from typing import Optional, Dict, List
from ninja.errors import HttpError
from ddpui.utils.redis_client import RedisClient
from ddpui.utils.custom_logger import CustomLogger
from ddpui.models.org import Org

logger = CustomLogger("superset_service")

class SupersetService:
    def __init__(self, org: Org):
        self.org = org
        self.redis = RedisClient.get_instance()
        self.max_retries = 3
        self.retry_delay = 1  # seconds
        
    def _make_request_with_retry(self, method, url, **kwargs):
        """Make HTTP request with automatic retry on 401"""
        last_error = None
        
        for attempt in range(self.max_retries):
            try:
                response = requests.request(method, url, **kwargs)
                
                if response.status_code == 401:
                    # Clear cached tokens and retry
                    self._clear_cached_tokens()
                    if attempt < self.max_retries - 1:
                        time.sleep(self.retry_delay * (attempt + 1))
                        # Get fresh token for next attempt
                        kwargs['headers']['Authorization'] = f"Bearer {self.get_access_token(force_refresh=True)}"
                        continue
                    else:
                        # Final attempt failed, raise HttpError
                        raise HttpError(401, "Authentication expired with Superset")
                
                response.raise_for_status()
                return response
                
            except requests.exceptions.RequestException as e:
                last_error = e
                if attempt < self.max_retries - 1:
                    time.sleep(self.retry_delay * (attempt + 1))
                    continue
        
        # All retries failed
        logger.error(f"Failed to connect to Superset after {self.max_retries} attempts: {str(last_error)}")
        raise HttpError(503, "Failed to connect to Superset")
    
    def _clear_cached_tokens(self):
        """Clear all cached tokens for the org"""
        pattern = f"superset:token:{self.org.id}:*"
        for key in self.redis.scan_iter(match=pattern):
            self.redis.delete(key)
            
    def get_access_token(self, force_refresh=False):
        """Get access token with caching"""
        cache_key = f"superset:token:{self.org.id}:access"
        
        if not force_refresh:
            cached_token = self.redis.get(cache_key)
            if cached_token:
                return cached_token.decode('utf-8')
        
        # Get credentials from secrets manager
        if not self.org.dalgouser_superset_creds_key:
            raise HttpError(400, "Superset admin credentials not configured")
            
        credentials = secretsmanager.retrieve_dalgo_user_superset_credentials(
            self.org.dalgouser_superset_creds_key
        )
        if not credentials:
            raise HttpError(400, "Superset admin credentials not found")
        
        # Login to get access token
        try:
            response = requests.post(
                f"{self.org.viz_url}api/v1/security/login",
                json={
                    "password": credentials["password"],
                    "username": credentials["username"],
                    "refresh": True,
                    "provider": "db",
                },
                timeout=10,
            )
            response.raise_for_status()
            access_token = response.json()["access_token"]
            
            # Cache the token
            self.redis.set(cache_key, access_token, self.access_token_ttl)
            return access_token
            
        except requests.exceptions.RequestException as e:
            logger.error(f"Failed to authenticate with Superset: {str(e)}")
            raise HttpError(503, "Failed to authenticate with Superset")
            
    def get_dashboards(self, page=0, page_size=20, search=None, status=None):
        """Get dashboards with automatic token refresh on 401"""
        access_token = self.get_access_token()
        
        # Build query parameters
        filters = []
        if search:
            filters.append({
                "col": "dashboard_title",
                "opr": "ct",  # contains
                "value": search
            })
        if status:
            filters.append({
                "col": "published",
                "opr": "eq",
                "value": status == "published"
            })
            
        query_params = {
            "page": page,
            "page_size": page_size,
        }
        if filters:
            query_params["filters"] = filters
            
        url = f"{self.org.viz_url}api/v1/dashboard/"
        
        response = self._make_request_with_retry(
            "GET",
            url,
            headers={
                "Authorization": f"Bearer {access_token}",
                "Content-Type": "application/json",
            },
            params={"q": json.dumps(query_params)},
            timeout=30
        )
        
        return response.json()
    
    def get_dashboard_thumbnail(self, dashboard_id: str):
        """Get dashboard thumbnail"""
        access_token = self.get_access_token()
        
        url = f"{self.org.viz_url}api/v1/dashboard/{dashboard_id}/thumbnail/"
        
        try:
            response = self._make_request_with_retry(
                "GET",
                url,
                headers={
                    "Authorization": f"Bearer {access_token}",
                },
                timeout=30
            )
            return response.content
        except HttpError as e:
            if e.status_code == 404:
                return None  # Dashboard doesn't have thumbnail
            raise
```

### 2. API Endpoints Enhancement
**File to modify**: `DDP_backend/ddpui/api/superset_api.py`

**New/Modified Endpoints** - Clean implementation without try-catch blocks:

```python
from ddpui.services.superset_service import SupersetService

@superset_router.post("dashboards/{dashboard_id}/guest_token/")
def post_dashboard_guest_token(request, dashboard_id: str):
    """Generate guest token for dashboard embedding"""
    orguser: OrgUser = request.orguser
    
    if not orguser.org or not orguser.org.viz_url:
        raise HttpError(400, "Superset not configured for this organization")
    
    # SupersetService will raise HttpError if something goes wrong
    service = SupersetService(orguser.org)
    dashboard = service.get_dashboard_by_id(dashboard_id)
    dashboard_uuid = dashboard.get("uuid")
    
    if not dashboard_uuid:
        raise HttpError(404, "Dashboard not found")
    
    # Generate guest token - service will raise HttpError on failure
    guest_token_data = service.get_guest_token(dashboard_uuid)
    
    return {
        "guest_token": guest_token_data["token"],
        "expires_in": 300,  # 5 minutes
        "dashboard_uuid": dashboard_uuid,
        "superset_domain": orguser.org.viz_url.rstrip("/")
    }

@superset_router.get("dashboards/")
def get_dashboards(request, page: int = 0, page_size: int = 20, 
                  search: str = None, status: str = None):
    """List dashboards with enhanced features"""
    orguser: OrgUser = request.orguser
    
    if not orguser.org or not orguser.org.viz_url:
        return {"result": [], "count": 0}
    
    # SupersetService will raise HttpError if something goes wrong
    service = SupersetService(orguser.org)
    data = service.get_dashboards(page, page_size, search, status)
    
    # Enhance with thumbnail URLs
    for dashboard in data.get("result", []):
        dashboard["thumbnail_url"] = (
            f"/api/superset/dashboards/{dashboard['id']}/thumbnail/"
        )
        
    return data

@superset_router.get("dashboards/{dashboard_id}/")
def get_dashboard_by_id(request, dashboard_id: str):
    """Get single dashboard details"""
    orguser: OrgUser = request.orguser
    
    if not orguser.org or not orguser.org.viz_url:
        raise HttpError(400, "Superset not configured for this organization")
    
    # SupersetService will raise HttpError if something goes wrong
    service = SupersetService(orguser.org)
    return service.get_dashboard_by_id(dashboard_id)

@superset_router.get("dashboards/{dashboard_id}/thumbnail/")
def get_dashboard_thumbnail(request, dashboard_id: str):
    """Get dashboard thumbnail with caching"""
    orguser: OrgUser = request.orguser
    
    if not orguser.org or not orguser.org.viz_url:
        raise HttpError(400, "Superset not configured for this organization")
    
    service = SupersetService(orguser.org)
    thumbnail = service.get_dashboard_thumbnail(dashboard_id)
    
    if not thumbnail:
        # Return placeholder image
        return FileResponse("static/dashboard-placeholder.png")
    
    return HttpResponse(thumbnail, content_type="image/png")
```

### 3. Error Handling Strategy

The error handling is now simplified:

1. **SupersetService** raises `HttpError` with appropriate status codes
2. **API endpoints** don't need try-catch blocks
3. **Global exception handlers** in `routes.py` handle all errors
4. **Frontend** receives consistent error responses

Error types:
- `400`: Bad request (missing credentials, invalid org)
- `401`: Authentication expired
- `404`: Resource not found
- `503`: Service unavailable (Superset down)

## Frontend Implementation (webapp_v2)

### 1. Dashboard Listing Page Enhancement
**File to modify**: `webapp_v2/components/dashboard/dashboard-list.tsx`

Add proper error handling and retry logic:

```typescript
const useSupersetDashboards = (params: DashboardParams) => {
  const { data, error, mutate } = useSWR(
    `/api/superset/dashboards?${new URLSearchParams(params)}`,
    apiGet,
    {
      refreshInterval: 60000,
      revalidateOnFocus: true,
      onError: (error) => {
        // SWR will automatically retry based on configuration
        console.error('Dashboard fetch error:', error);
      }
    }
  );
  
  return { data, error, mutate, isLoading: !data && !error };
};
```

### 2. Dashboard Embedding View
**File to create**: `webapp_v2/components/dashboard/superset-embed.tsx`

(Implementation shown above in Token Handling section)

### 3. Routing Updates
**File to modify**: `webapp_v2/app/dashboards/[id]/page.tsx`

```typescript
export default function DashboardPage({ params }: { params: { id: string } }) {
  const { data: dashboard, error } = useSWR(
    `/api/superset/dashboards/${params.id}/`,
    apiGet
  );
  
  if (error) return <ErrorPage error={error} />;
  if (!dashboard) return <LoadingPage />;
  
  return (
    <div className="h-screen flex flex-col">
      <DashboardHeader dashboard={dashboard} />
      <div className="flex-1">
        <SupersetEmbed 
          dashboardId={dashboard.id}
          dashboardUuid={dashboard.uuid}
        />
      </div>
    </div>
  );
}
```

## Testing Strategy

### Unit Tests

#### Backend Tests
**File to create**: `DDP_backend/ddpui/tests/services/test_superset_service.py`
```python
def test_token_caching():
    # Test that tokens are cached and retrieved correctly
    
def test_token_refresh_on_401():
    # Test automatic token refresh on 401 response
    
def test_retry_logic():
    # Test exponential backoff retry
    
def test_guest_token_generation():
    # Test guest token generation and caching
    
def test_http_error_raising():
    # Test that appropriate HttpErrors are raised
```

#### Frontend Tests
**File to create**: `webapp_v2/__tests__/components/dashboard/superset-embed.test.tsx`
```typescript
describe('SupersetEmbed', () => {
  it('should call fetchGuestToken on mount');
  it('should handle token refresh automatically');
  it('should display error state on embedding failure');
  it('should cleanup iframe on unmount');
});
```

### Integration Tests
1. Test full flow: Login → List Dashboards → View Dashboard
2. Test token expiry scenarios (wait 5+ minutes)
3. Test 401 error recovery
4. Test concurrent requests with token refresh
5. Test dashboard switching

### Manual Testing Checklist
- [ ] Dalgo server starts without errors
- [ ] Dashboard listing loads with thumbnails
- [ ] Search and filter work correctly
- [ ] Dashboard embedding works on first load
- [ ] Dashboard continues working after 5+ minutes (token refresh)
- [ ] 401 errors are handled gracefully
- [ ] Error states show proper messages
- [ ] Mobile responsive design works
- [ ] Performance is acceptable with caching

## Implementation Steps

### Phase 1: Backend Foundation (2-3 days)
1. Create SupersetService with retry logic and HttpError raising
2. Implement Redis token caching with TTLs
3. Update API endpoints (no try-catch needed)
4. Add thumbnail endpoint
5. Write comprehensive tests

### Phase 2: Frontend Dashboard List (2 days)
1. Update dashboard-list with real API
2. Add thumbnail rendering with placeholders
3. Implement search/filter with debouncing
4. Add error handling
5. Write component tests

### Phase 3: Dashboard Embedding (2 days)
1. Create superset-embed component with SDK
2. Implement automatic token refresh
3. Add error boundaries
4. Test 5+ minute sessions
5. Verify mobile responsiveness

### Phase 4: Testing & Polish (2 days)
1. Run all automated tests
2. Perform extended session testing
3. Test edge cases (network issues, Superset down)
4. Performance optimization
5. Documentation

## Gotchas and Considerations

1. **Token Lifecycle**: Guest tokens expire in 5 minutes, SDK handles refresh automatically
2. **CORS Configuration**: Ensure Superset allows embedding from Dalgo domain
3. **Buffer Polyfill**: May need Buffer polyfill for some browsers
4. **Rate Limiting**: Implement request throttling to avoid Superset limits
5. **Large Dashboards**: Show loading progress for slow dashboards
6. **Security**: Never expose admin credentials to frontend

## External Documentation References
- Superset API: https://superset.apache.org/docs/api/
- Embedded SDK: https://www.npmjs.com/package/@superset-ui/embedded-sdk
- Guest Token Guide: https://www.restack.io/docs/superset-knowledge-superset-guest-token-overview
- Embedding Best Practices: https://www.tetranyde.com/blog/embedding-superset/
- Token Handling: https://stackoverflow.com/questions/78385142/how-to-integrate-refresh-token-functionality-for-embedding-superset-dashboards

## Success Criteria
1. All unit tests pass
2. Dashboard listing shows real dashboards with thumbnails
3. Search and filter work correctly
4. Dashboard embedding works initially
5. Dashboard continues working after 5+ minutes (automatic token refresh)
6. 401 errors are recovered automatically
7. Token caching reduces API calls by 80%+
8. Mobile responsive design works
9. Page load time < 2 seconds

## Confidence Score: 9.5/10

The plan is now comprehensive with proper error handling:
1. SupersetService raises HttpError - clean and consistent
2. No try-catch blocks needed in API endpoints
3. Global error handlers manage all exceptions
4. Guest tokens with automatic refresh by SDK
5. Comprehensive caching strategy

The confidence is very high because:
- Follows existing codebase patterns
- Clean error handling without middleware
- Automatic token refresh handled by SDK
- Well-documented external resources
- Clear implementation path