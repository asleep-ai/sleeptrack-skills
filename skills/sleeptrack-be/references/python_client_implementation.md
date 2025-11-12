# Python Client Implementation Guide

This reference provides complete Python client implementations for the Asleep API, including advanced patterns for analytics, production usage, and multi-tenant applications.

## Complete Python API Client

```python
import os
import requests
from typing import Dict, Any, Optional

class AsleepClient:
    """Asleep API client for backend integration"""

    def __init__(self, api_key: str, base_url: str = "https://api.asleep.ai"):
        self.api_key = api_key
        self.base_url = base_url
        self.session = requests.Session()
        self.session.headers.update({"x-api-key": api_key})

    def _request(
        self,
        method: str,
        path: str,
        headers: Optional[Dict[str, str]] = None,
        **kwargs
    ) -> Dict[str, Any]:
        """Make authenticated API request with error handling"""
        url = f"{self.base_url}{path}"
        req_headers = self.session.headers.copy()
        if headers:
            req_headers.update(headers)

        try:
            response = self.session.request(method, url, headers=req_headers, **kwargs)
            response.raise_for_status()
            return response.json()
        except requests.exceptions.HTTPError as e:
            # Handle API errors
            if e.response.status_code == 401:
                raise ValueError("Invalid API key")
            elif e.response.status_code == 403:
                error_detail = e.response.json().get("detail", "Access forbidden")
                raise ValueError(f"API access error: {error_detail}")
            elif e.response.status_code == 404:
                raise ValueError("Resource not found")
            else:
                raise

    # User management methods
    def create_user(self, metadata: Optional[Dict[str, Any]] = None) -> str:
        """Create new user and return user_id"""
        data = {"metadata": metadata} if metadata else {}
        result = self._request("POST", "/ai/v1/users", json=data)
        return result["result"]["user_id"]

    def get_user(self, user_id: str) -> Dict[str, Any]:
        """Get user information"""
        result = self._request("GET", f"/ai/v1/users/{user_id}")
        return result["result"]

    def delete_user(self, user_id: str) -> None:
        """Delete user and all associated data"""
        self._request("DELETE", f"/ai/v1/users/{user_id}")

    # Session management methods
    def get_session(self, session_id: str, user_id: str, timezone: str = "UTC") -> Dict[str, Any]:
        """Get detailed session data"""
        headers = {"x-user-id": user_id, "timezone": timezone}
        result = self._request("GET", f"/data/v3/sessions/{session_id}", headers=headers)
        return result["result"]

    def list_sessions(
        self,
        user_id: str,
        date_gte: Optional[str] = None,
        date_lte: Optional[str] = None,
        offset: int = 0,
        limit: int = 20,
        order_by: str = "DESC"
    ) -> Dict[str, Any]:
        """List user sessions with filtering"""
        headers = {"x-user-id": user_id}
        params = {"offset": offset, "limit": limit, "order_by": order_by}
        if date_gte:
            params["date_gte"] = date_gte
        if date_lte:
            params["date_lte"] = date_lte

        result = self._request("GET", "/data/v1/sessions", headers=headers, params=params)
        return result["result"]

    def delete_session(self, session_id: str, user_id: str) -> None:
        """Delete session and all associated data"""
        headers = {"x-user-id": user_id}
        self._request("DELETE", f"/ai/v1/sessions/{session_id}", headers=headers)

    # Statistics methods
    def get_average_stats(
        self,
        user_id: str,
        start_date: str,
        end_date: str,
        timezone: str = "UTC"
    ) -> Dict[str, Any]:
        """Get average statistics for date range (max 100 days)"""
        headers = {"timezone": timezone}
        params = {"start_date": start_date, "end_date": end_date}
        result = self._request(
            "GET",
            f"/data/v1/users/{user_id}/average-stats",
            headers=headers,
            params=params
        )
        return result["result"]

# Usage
client = AsleepClient(api_key=os.getenv("ASLEEP_API_KEY"))
```

## Advanced Analytics Implementation

```python
from typing import List, Dict
from datetime import datetime, timedelta

class SleepAnalytics:
    """Backend analytics for sleep tracking platform"""

    def __init__(self, client: AsleepClient, db):
        self.client = client
        self.db = db

    def get_user_sleep_score(self, user_id: str, days: int = 30) -> Dict:
        """Calculate comprehensive sleep score for user"""
        end_date = datetime.now()
        start_date = end_date - timedelta(days=days)

        stats = self.client.get_average_stats(
            user_id=user_id,
            start_date=start_date.strftime("%Y-%m-%d"),
            end_date=end_date.strftime("%Y-%m-%d")
        )

        avg = stats['average_stats']

        # Calculate weighted sleep score (0-100)
        efficiency_score = avg['sleep_efficiency']  # Already 0-100
        consistency_score = self._calculate_consistency_score(stats)
        duration_score = self._calculate_duration_score(avg)

        overall_score = (
            efficiency_score * 0.4 +
            consistency_score * 0.3 +
            duration_score * 0.3
        )

        return {
            'overall_score': round(overall_score, 1),
            'efficiency_score': round(efficiency_score, 1),
            'consistency_score': round(consistency_score, 1),
            'duration_score': round(duration_score, 1),
            'period_days': days,
            'session_count': len(stats['slept_sessions'])
        }

    def _calculate_consistency_score(self, stats: Dict) -> float:
        """Score based on sleep schedule consistency"""
        # Implement consistency scoring based on variance in sleep times
        # Placeholder implementation
        return 80.0

    def _calculate_duration_score(self, avg: Dict) -> float:
        """Score based on sleep duration (7-9 hours optimal)"""
        sleep_hours = avg['time_in_sleep'] / 3600

        if 7 <= sleep_hours <= 9:
            return 100.0
        elif 6 <= sleep_hours < 7 or 9 < sleep_hours <= 10:
            return 80.0
        elif 5 <= sleep_hours < 6 or 10 < sleep_hours <= 11:
            return 60.0
        else:
            return 40.0

    def get_cohort_analysis(self, user_ids: List[str], days: int = 30) -> Dict:
        """Analyze sleep patterns across user cohort"""
        cohort_data = []

        for user_id in user_ids:
            try:
                score = self.get_user_sleep_score(user_id, days)
                cohort_data.append({
                    'user_id': user_id,
                    'score': score['overall_score'],
                    'efficiency': score['efficiency_score'],
                    'sessions': score['session_count']
                })
            except Exception as e:
                print(f"Error fetching data for user {user_id}: {e}")

        if not cohort_data:
            return {}

        return {
            'cohort_size': len(cohort_data),
            'avg_score': sum(u['score'] for u in cohort_data) / len(cohort_data),
            'avg_efficiency': sum(u['efficiency'] for u in cohort_data) / len(cohort_data),
            'total_sessions': sum(u['sessions'] for u in cohort_data),
            'users': cohort_data
        }

    def generate_weekly_report(self, user_id: str) -> Dict:
        """Generate comprehensive weekly sleep report"""
        stats = self.client.get_average_stats(
            user_id=user_id,
            start_date=(datetime.now() - timedelta(days=7)).strftime("%Y-%m-%d"),
            end_date=datetime.now().strftime("%Y-%m-%d")
        )

        avg = stats['average_stats']

        return {
            'period': 'Last 7 days',
            'summary': {
                'avg_sleep_time': avg['sleep_time'],
                'avg_bedtime': avg['start_time'],
                'avg_wake_time': avg['end_time'],
                'avg_efficiency': avg['sleep_efficiency']
            },
            'sleep_stages': {
                'light_hours': avg['time_in_light'] / 3600,
                'deep_hours': avg['time_in_deep'] / 3600,
                'rem_hours': avg['time_in_rem'] / 3600
            },
            'insights': self._generate_insights(avg),
            'session_count': len(stats['slept_sessions'])
        }

    def _generate_insights(self, avg: Dict) -> List[str]:
        """Generate personalized sleep insights"""
        insights = []

        if avg['sleep_efficiency'] < 75:
            insights.append("Your sleep efficiency is below average. Try establishing a consistent bedtime routine.")

        if avg['deep_ratio'] < 0.15:
            insights.append("You're getting less deep sleep than optimal. Avoid caffeine after 2 PM.")

        if avg['waso_count'] > 3:
            insights.append("You're waking up frequently during the night. Consider reducing screen time before bed.")

        return insights

# Usage
analytics = SleepAnalytics(client, db)
score = analytics.get_user_sleep_score("user123", days=30)
print(f"Sleep score: {score['overall_score']}/100")

report = analytics.generate_weekly_report("user123")
print(f"Weekly report: {report}")
```

## Multi-Tenant Application Implementation

```python
class MultiTenantSleepTracker:
    """Multi-tenant sleep tracking backend"""

    def __init__(self, client: AsleepClient, db):
        self.client = client
        self.db = db

    def create_organization(self, org_id: str, name: str, settings: Dict) -> Dict:
        """Create new organization"""
        org = {
            'org_id': org_id,
            'name': name,
            'settings': settings,
            'created_at': datetime.now(),
            'user_count': 0
        }
        self.db.organizations.insert_one(org)
        return org

    def add_user_to_organization(self, org_id: str, user_email: str, metadata: Dict = None) -> str:
        """Add user to organization and create Asleep user"""
        # Verify organization exists
        org = self.db.organizations.find_one({'org_id': org_id})
        if not org:
            raise ValueError(f"Organization {org_id} not found")

        # Create Asleep user
        asleep_user_id = self.client.create_user(metadata=metadata)

        # Store user mapping
        self.db.users.insert_one({
            'org_id': org_id,
            'user_email': user_email,
            'asleep_user_id': asleep_user_id,
            'metadata': metadata,
            'created_at': datetime.now()
        })

        # Update organization user count
        self.db.organizations.update_one(
            {'org_id': org_id},
            {'$inc': {'user_count': 1}}
        )

        return asleep_user_id

    def get_organization_statistics(self, org_id: str, days: int = 30) -> Dict:
        """Get aggregated statistics for entire organization"""
        # Get all users in organization
        users = list(self.db.users.find({'org_id': org_id}))

        end_date = datetime.now()
        start_date = end_date - timedelta(days=days)

        org_stats = {
            'org_id': org_id,
            'user_count': len(users),
            'period_days': days,
            'users_data': []
        }

        total_efficiency = 0
        total_sleep_time = 0
        total_sessions = 0

        for user in users:
            try:
                stats = self.client.get_average_stats(
                    user_id=user['asleep_user_id'],
                    start_date=start_date.strftime("%Y-%m-%d"),
                    end_date=end_date.strftime("%Y-%m-%d")
                )

                avg = stats['average_stats']
                session_count = len(stats['slept_sessions'])

                org_stats['users_data'].append({
                    'user_email': user['user_email'],
                    'efficiency': avg['sleep_efficiency'],
                    'sleep_time': avg['sleep_time'],
                    'session_count': session_count
                })

                total_efficiency += avg['sleep_efficiency']
                total_sleep_time += avg['time_in_sleep']
                total_sessions += session_count

            except Exception as e:
                print(f"Error fetching stats for user {user['user_email']}: {e}")

        if users:
            org_stats['avg_efficiency'] = total_efficiency / len(users)
            org_stats['avg_sleep_hours'] = (total_sleep_time / len(users)) / 3600
            org_stats['total_sessions'] = total_sessions

        return org_stats

# Usage
tracker = MultiTenantSleepTracker(client, db)

# Create organization
tracker.create_organization(
    org_id="acme-corp",
    name="Acme Corporation",
    settings={'timezone': 'America/New_York'}
)

# Add users
tracker.add_user_to_organization("acme-corp", "john@acme.com")
tracker.add_user_to_organization("acme-corp", "jane@acme.com")

# Get organization stats
org_stats = tracker.get_organization_statistics("acme-corp", days=30)
print(f"Organization average efficiency: {org_stats['avg_efficiency']:.1f}%")
```

## FastAPI Backend Example

```python
from fastapi import FastAPI, HTTPException, Depends, Header
from typing import Optional
import os

app = FastAPI()
asleep_client = AsleepClient(api_key=os.getenv("ASLEEP_API_KEY"))

# Authentication dependency
async def verify_app_token(authorization: str = Header(...)):
    """Verify mobile app authentication"""
    if not authorization.startswith("Bearer "):
        raise HTTPException(status_code=401, detail="Invalid authorization header")

    token = authorization[7:]
    # Verify token with your auth system
    user = verify_jwt_token(token)
    if not user:
        raise HTTPException(status_code=401, detail="Invalid token")

    return user

# Proxy endpoints
@app.post("/api/users")
async def create_user(
    metadata: Optional[dict] = None,
    user: dict = Depends(verify_app_token)
):
    """Create Asleep user for authenticated app user"""
    try:
        # Create user in Asleep
        asleep_user_id = asleep_client.create_user(metadata=metadata)

        # Store mapping in your database
        db.user_mappings.insert_one({
            'app_user_id': user['id'],
            'asleep_user_id': asleep_user_id,
            'created_at': datetime.now()
        })

        return {"user_id": asleep_user_id}

    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/api/sessions/{session_id}")
async def get_session(
    session_id: str,
    user: dict = Depends(verify_app_token)
):
    """Get session data for authenticated user"""
    # Get Asleep user ID from mapping
    mapping = db.user_mappings.find_one({'app_user_id': user['id']})
    if not mapping:
        raise HTTPException(status_code=404, detail="User not found")

    asleep_user_id = mapping['asleep_user_id']

    # Fetch session from Asleep
    try:
        session = asleep_client.get_session(session_id, asleep_user_id)
        return session
    except ValueError as e:
        raise HTTPException(status_code=404, detail=str(e))

@app.get("/api/sessions")
async def list_sessions(
    date_gte: Optional[str] = None,
    date_lte: Optional[str] = None,
    user: dict = Depends(verify_app_token)
):
    """List sessions for authenticated user"""
    mapping = db.user_mappings.find_one({'app_user_id': user['id']})
    if not mapping:
        raise HTTPException(status_code=404, detail="User not found")

    asleep_user_id = mapping['asleep_user_id']

    sessions = asleep_client.list_sessions(
        user_id=asleep_user_id,
        date_gte=date_gte,
        date_lte=date_lte
    )

    return sessions

@app.get("/api/statistics")
async def get_statistics(
    start_date: str,
    end_date: str,
    user: dict = Depends(verify_app_token)
):
    """Get average statistics for authenticated user"""
    mapping = db.user_mappings.find_one({'app_user_id': user['id']})
    if not mapping:
        raise HTTPException(status_code=404, detail="User not found")

    asleep_user_id = mapping['asleep_user_id']

    stats = asleep_client.get_average_stats(
        user_id=asleep_user_id,
        start_date=start_date,
        end_date=end_date
    )

    return stats
```

## Monthly Trends Analysis

```python
from datetime import datetime, timedelta
from typing import List, Dict

def get_monthly_trends(client, user_id: str, months: int = 6) -> List[Dict]:
    """Get monthly sleep trends for the past N months"""
    trends = []
    today = datetime.now()

    for i in range(months):
        # Calculate month boundaries
        end_date = today.replace(day=1) - timedelta(days=i * 30)
        start_date = end_date - timedelta(days=30)

        try:
            stats = client.get_average_stats(
                user_id=user_id,
                start_date=start_date.strftime("%Y-%m-%d"),
                end_date=end_date.strftime("%Y-%m-%d")
            )

            trends.append({
                'month': end_date.strftime("%Y-%m"),
                'avg_sleep_time': stats['average_stats']['sleep_time'],
                'avg_efficiency': stats['average_stats']['sleep_efficiency'],
                'session_count': len(stats['slept_sessions'])
            })
        except Exception as e:
            print(f"Error fetching stats for {end_date.strftime('%Y-%m')}: {e}")

    return trends

# Usage
trends = get_monthly_trends(client, "user123", months=6)
for trend in trends:
    print(f"{trend['month']}: {trend['avg_sleep_time']} sleep, "
          f"{trend['avg_efficiency']:.1f}% efficiency, "
          f"{trend['session_count']} sessions")
```

## Pagination Pattern

```python
# Fetch all sessions with pagination
all_sessions = []
offset = 0
limit = 100

while True:
    result = client.list_sessions(
        user_id="user123",
        date_gte="2024-01-01",
        date_lte="2024-12-31",
        offset=offset,
        limit=limit
    )

    sessions = result['sleep_session_list']
    all_sessions.extend(sessions)

    if len(sessions) < limit:
        break

    offset += limit

print(f"Total sessions: {len(all_sessions)}")
```
