# 📊 Telemetry & Analytics Guide

Guide for collecting game data for your data analyst.

---

## 🎯 What Data is Already Being Collected?

Your game already tracks these in the **PostgreSQL database**:

### 1. **Player Data** (`players` table)
- Player ID
- Username
- Email
- MMR (skill rating)
- Registration date
- Last login date
- Is guest (true/false)

### 2. **Match History** (`match_history` table)
- Match ID
- Player IDs (both players)
- Winner/Loser
- Final scores
- Match duration
- MMR changes
- Timestamp

### 3. **Player Stats** (`player_stats` table)
- Total matches played
- Matches won/lost
- Win rate
- Total swaps made
- Total tiles cleared
- Max combo achieved
- Power-ups used
- Average match score

---

## 📈 How to Access the Data

### Option 1: Direct Database Access (Recommended)

Your data analyst can connect directly to your Railway PostgreSQL database.

#### Get Database Credentials:

1. Go to **Railway dashboard**
2. Click on your **PostgreSQL** service
3. Go to **"Connect"** tab
4. Copy the connection details:

```
Host: your-host.railway.app
Port: 5432
Database: railway
Username: postgres
Password: [your-password]
```

#### Connect with Tools:

**Option A: PostgreSQL Client (pgAdmin, DBeaver)**
- Download pgAdmin: https://www.pgadmin.org/
- Use the credentials above to connect

**Option B: Python (for data analysis)**
```python
import psycopg2
import pandas as pd

# Connect to database
conn = psycopg2.connect(
    host="your-host.railway.app",
    port=5432,
    database="railway",
    user="postgres",
    password="your-password"
)

# Query player stats
df = pd.read_sql_query("SELECT * FROM player_stats", conn)
print(df.head())

# Query match history
matches = pd.read_sql_query("""
    SELECT * FROM match_history
    WHERE created_at >= NOW() - INTERVAL '7 days'
    ORDER BY created_at DESC
""", conn)
```

**Option C: SQL queries directly**
```sql
-- Get top 10 players by MMR
SELECT username, mmr, matches_played, win_rate
FROM players
JOIN player_stats ON players.id = player_stats.player_id
ORDER BY mmr DESC
LIMIT 10;

-- Get match statistics
SELECT
  COUNT(*) as total_matches,
  AVG(duration_seconds) as avg_match_duration,
  SUM(CASE WHEN winner_id = player1_id THEN 1 ELSE 0 END) as player1_wins,
  SUM(CASE WHEN winner_id = player2_id THEN 1 ELSE 0 END) as player2_wins
FROM match_history;

-- Get daily active users
SELECT
  DATE(last_login) as date,
  COUNT(*) as active_users
FROM players
WHERE last_login >= NOW() - INTERVAL '30 days'
GROUP BY DATE(last_login)
ORDER BY date DESC;
```

---

## 📊 Key Metrics to Track

### 1. Player Engagement
```sql
-- Daily Active Users (DAU)
SELECT DATE(last_login) as date, COUNT(*) as dau
FROM players
WHERE last_login >= NOW() - INTERVAL '30 days'
GROUP BY DATE(last_login);

-- Average session duration
SELECT AVG(duration_seconds / 60.0) as avg_minutes
FROM match_history;

-- Retention rate (7-day)
SELECT
  COUNT(DISTINCT CASE WHEN last_login >= NOW() - INTERVAL '7 days' THEN id END) * 100.0 /
  COUNT(DISTINCT id) as retention_7day
FROM players
WHERE created_at <= NOW() - INTERVAL '7 days';
```

### 2. Game Balance
```sql
-- Win rate distribution
SELECT
  CASE
    WHEN win_rate < 0.3 THEN 'Low (<30%)'
    WHEN win_rate < 0.5 THEN 'Medium (30-50%)'
    WHEN win_rate < 0.7 THEN 'High (50-70%)'
    ELSE 'Very High (>70%)'
  END as win_rate_bucket,
  COUNT(*) as player_count
FROM player_stats
GROUP BY win_rate_bucket;

-- MMR distribution
SELECT
  FLOOR(mmr / 100) * 100 as mmr_range,
  COUNT(*) as player_count
FROM players
GROUP BY mmr_range
ORDER BY mmr_range;

-- Average match score
SELECT
  AVG(winner_score) as avg_winner_score,
  AVG(loser_score) as avg_loser_score,
  AVG(winner_score - loser_score) as avg_score_difference
FROM match_history;
```

### 3. Progression Metrics
```sql
-- Player progression over time
SELECT
  username,
  created_at,
  mmr,
  matches_played,
  win_rate,
  EXTRACT(DAYS FROM NOW() - created_at) as days_since_signup
FROM players
JOIN player_stats ON players.id = player_stats.player_id
ORDER BY created_at DESC;

-- Skill improvement
SELECT
  player_id,
  MIN(mmr) as starting_mmr,
  MAX(mmr) as peak_mmr,
  MAX(mmr) - MIN(mmr) as mmr_gain
FROM (
  SELECT player1_id as player_id, player1_mmr_after as mmr FROM match_history
  UNION ALL
  SELECT player2_id as player_id, player2_mmr_after as mmr FROM match_history
) t
GROUP BY player_id
HAVING COUNT(*) >= 10
ORDER BY mmr_gain DESC;
```

---

## 🔧 Setting Up Telemetry Service (Optional)

If you want **real-time analytics** or **custom events**, you can enable the telemetry service.

### What is Telemetry?

The game has a built-in telemetry system (`apps/server/src/telemetry.ts`) that can send events to external analytics services.

### How to Enable:

1. **Choose an analytics service:**
   - **PostHog** (recommended, free tier) - https://posthog.com
   - **Mixpanel** (popular)
   - **Amplitude** (enterprise)
   - **Custom API** (your own endpoint)

2. **Get your API key:**
   - Sign up at PostHog (or your chosen service)
   - Copy your API key

3. **Add to Railway environment:**
   ```
   ENABLE_TELEMETRY = true
   TELEMETRY_ENDPOINT = https://app.posthog.com/capture/
   TELEMETRY_API_KEY = your-api-key-here
   ```

4. **Events automatically tracked:**
   - Player signup
   - Player login
   - Match start
   - Match end
   - Swap made
   - Power-up used
   - Player disconnect

### Example PostHog Setup:

```typescript
// In apps/server/src/telemetry.ts (already exists)
import { PostHog } from 'posthog-node';

const posthog = new PostHog(
  process.env.TELEMETRY_API_KEY || '',
  { host: 'https://app.posthog.com' }
);

// Track custom event
posthog.capture({
  distinctId: playerId,
  event: 'match_completed',
  properties: {
    duration: durationSeconds,
    winner: winnerId,
    score: finalScore,
    mmr_change: mmrChange
  }
});
```

---

## 📋 Common Analysis Queries

### 1. Player Cohort Analysis
```sql
-- Players by signup week
SELECT
  DATE_TRUNC('week', created_at) as cohort_week,
  COUNT(*) as new_players,
  AVG(matches_played) as avg_matches,
  AVG(win_rate) as avg_win_rate
FROM players
JOIN player_stats ON players.id = player_stats.player_id
GROUP BY cohort_week
ORDER BY cohort_week DESC;
```

### 2. Match Frequency
```sql
-- How often players play
SELECT
  CASE
    WHEN matches_played = 1 THEN 'One-time player'
    WHEN matches_played < 5 THEN 'Casual (2-4 matches)'
    WHEN matches_played < 20 THEN 'Regular (5-19 matches)'
    WHEN matches_played < 50 THEN 'Active (20-49 matches)'
    ELSE 'Core player (50+ matches)'
  END as player_type,
  COUNT(*) as count,
  AVG(win_rate) as avg_win_rate
FROM player_stats
GROUP BY player_type;
```

### 3. Power-up Usage
```sql
-- Power-up effectiveness
SELECT
  AVG(CASE WHEN power_ups_used > 0 THEN win_rate ELSE NULL END) as win_rate_with_powerups,
  AVG(CASE WHEN power_ups_used = 0 THEN win_rate ELSE NULL END) as win_rate_without_powerups
FROM player_stats
WHERE matches_played >= 10;
```

### 4. Churn Analysis
```sql
-- Players who stopped playing
SELECT
  EXTRACT(DAYS FROM NOW() - last_login) as days_inactive,
  COUNT(*) as player_count
FROM players
GROUP BY days_inactive
ORDER BY days_inactive;
```

---

## 📤 Exporting Data

### Option 1: CSV Export (Simple)

```bash
# Connect to Railway PostgreSQL
railway connect postgres

# Export to CSV
\copy (SELECT * FROM player_stats) TO 'player_stats.csv' WITH CSV HEADER;
\copy (SELECT * FROM match_history) TO 'match_history.csv' WITH CSV HEADER;
```

### Option 2: Automated Daily Export (Python Script)

```python
import psycopg2
import pandas as pd
from datetime import datetime

# Connect to database
conn = psycopg2.connect("your-database-url")

# Export player stats
df_players = pd.read_sql_query("SELECT * FROM player_stats", conn)
df_players.to_csv(f"exports/players_{datetime.now():%Y%m%d}.csv", index=False)

# Export match history
df_matches = pd.read_sql_query("""
    SELECT * FROM match_history
    WHERE created_at >= NOW() - INTERVAL '1 day'
""", conn)
df_matches.to_csv(f"exports/matches_{datetime.now():%Y%m%d}.csv", index=False)

print(f"Exported {len(df_players)} players and {len(df_matches)} matches")
```

### Option 3: API Endpoint (For Real-time Dashboards)

You can create a read-only API endpoint for your analyst:

```typescript
// Add to apps/server/src/index.ts
app.get('/api/analytics/stats', async (request, reply) => {
  // Only allow with special API key
  const apiKey = request.headers['x-api-key'];
  if (apiKey !== process.env.ANALYTICS_API_KEY) {
    return reply.code(401).send({ error: 'Unauthorized' });
  }

  const stats = await db.query(`
    SELECT
      (SELECT COUNT(*) FROM players) as total_players,
      (SELECT COUNT(*) FROM match_history) as total_matches,
      (SELECT AVG(mmr) FROM players) as avg_mmr,
      (SELECT COUNT(*) FROM players WHERE last_login >= NOW() - INTERVAL '1 day') as dau
  `);

  return stats.rows[0];
});
```

---

## 🎯 Recommended Setup for Your Analyst

**Simple Setup (Start Here):**
1. Give them database credentials
2. They use **pgAdmin** or **DBeaver** to connect
3. They run SQL queries directly
4. Export to CSV for analysis in Excel/Python

**Advanced Setup (Later):**
1. Set up **PostHog** for real-time events
2. Create **Metabase** or **Grafana** dashboard
3. Automated daily exports to Google Sheets
4. Custom API endpoints for live dashboards

---

## 📊 Sample Dashboard Queries

### Daily Dashboard:
```sql
-- Today's stats
SELECT
  (SELECT COUNT(*) FROM players WHERE DATE(created_at) = CURRENT_DATE) as new_players_today,
  (SELECT COUNT(*) FROM players WHERE DATE(last_login) = CURRENT_DATE) as active_players_today,
  (SELECT COUNT(*) FROM match_history WHERE DATE(created_at) = CURRENT_DATE) as matches_today,
  (SELECT AVG(duration_seconds / 60.0) FROM match_history WHERE DATE(created_at) = CURRENT_DATE) as avg_match_duration_minutes;
```

### Weekly Trends:
```sql
-- Last 7 days
SELECT
  DATE(created_at) as date,
  COUNT(*) as matches,
  COUNT(DISTINCT player1_id) + COUNT(DISTINCT player2_id) as unique_players,
  AVG(winner_score) as avg_score
FROM match_history
WHERE created_at >= NOW() - INTERVAL '7 days'
GROUP BY DATE(created_at)
ORDER BY date;
```

---

## 🆘 Support

**For your data analyst:**
- Database schema: See `apps/server/src/db/schema.sql`
- Sample queries: See this document
- Connection issues: Check Railway dashboard

**Questions?**
- Telemetry setup: Check `apps/server/src/telemetry.ts`
- Database queries: PostgreSQL docs
- Analytics tools: PostHog, Mixpanel documentation
