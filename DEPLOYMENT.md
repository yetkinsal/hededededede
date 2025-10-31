# Brick City Wars - Deployment Guide

This guide covers deploying Brick City Wars to various production environments.

## Prerequisites

- Node.js 20+
- pnpm 10+
- Docker & Docker Compose (for containerized deployments)
- PostgreSQL 15+
- Redis 7+

## Environment Configuration

### Server Environment Variables

Copy `apps/server/.env.production.example` to `apps/server/.env.production` and configure:

```bash
# Required
DATABASE_URL=postgresql://user:pass@host:5432/dbname
REDIS_URL=redis://user:pass@host:6379
JWT_SECRET=your-super-secure-secret-min-32-chars
BUILD_VERSION=1.0.3
CORS_ORIGIN=https://yourdomain.com

# Optional
NODE_ENV=production
PORT=3000
COLYSEUS_PORT=2567
LOG_LEVEL=info
```

### Webclient Environment Variables

Update `apps/webclient/.env.production`:

```bash
VITE_API_BASE=https://api.yourdomain.com
VITE_WS_URL=wss://api.yourdomain.com/ws
VITE_BUILD_VERSION=1.0.3
```

## Deployment Methods

### Method 1: Docker Compose (Recommended for VPS)

1. **Build and start all services:**
```bash
docker-compose -f docker-compose.prod.yml up -d
```

2. **Check service health:**
```bash
docker-compose -f docker-compose.prod.yml ps
```

3. **View logs:**
```bash
docker-compose -f docker-compose.prod.yml logs -f server
```

4. **Run database migrations:**
```bash
docker-compose -f docker-compose.prod.yml exec server pnpm migrate
```

5. **Access the application:**
- Webclient: http://localhost:8080
- Server API: http://localhost:3000
- WebSocket: ws://localhost:2567

### Method 2: Railway.app (Current Production)

1. **Install Railway CLI:**
```bash
npm install -g @railway/cli
railway login
```

2. **Deploy Server:**
```bash
cd apps/server
railway link [project-id]
railway up
```

3. **Deploy Webclient:**
```bash
cd apps/webclient
pnpm build
# Upload dist/ to static hosting (Vercel, Netlify, etc.)
```

4. **Set environment variables in Railway dashboard:**
- `DATABASE_URL` - Postgres connection string
- `REDIS_URL` - Redis connection string
- `JWT_SECRET` - Secret key for JWT signing
- `BUILD_VERSION` - Must match webclient
- `CORS_ORIGIN` - Frontend URL

### Method 3: Manual VPS Deployment

1. **Install dependencies on server:**
```bash
curl -fsSL https://get.pnpm.io/install.sh | sh -
```

2. **Clone repository:**
```bash
git clone https://github.com/yourusername/brickcitywars.git
cd brickcitywars
pnpm install --frozen-lockfile
```

3. **Build packages:**
```bash
pnpm build
```

4. **Set up PostgreSQL and Redis:**
```bash
# Install PostgreSQL
sudo apt-get install postgresql-15

# Install Redis
sudo apt-get install redis-server

# Create database
sudo -u postgres psql -c "CREATE DATABASE brickcitywars;"
```

5. **Configure environment:**
```bash
cp apps/server/.env.production.example apps/server/.env.production
# Edit with your values
nano apps/server/.env.production
```

6. **Run migrations:**
```bash
pnpm -C apps/server migrate
```

7. **Start server with PM2:**
```bash
npm install -g pm2
pm2 start apps/server/dist/index.js --name bcw-server
pm2 save
pm2 startup
```

8. **Serve webclient with nginx:**
```bash
sudo cp apps/webclient/nginx.conf /etc/nginx/sites-available/brickcitywars
sudo ln -s /etc/nginx/sites-available/brickcitywars /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

### Method 4: Kubernetes (Enterprise)

1. **Build Docker images:**
```bash
docker build -f apps/server/Dockerfile -t bcw-server:1.0.3 .
docker build -f apps/webclient/Dockerfile -t bcw-webclient:1.0.3 .
```

2. **Push to registry:**
```bash
docker tag bcw-server:1.0.3 yourregistry.com/bcw-server:1.0.3
docker push yourregistry.com/bcw-server:1.0.3
```

3. **Apply Kubernetes manifests:**
```bash
kubectl apply -f k8s/postgres.yaml
kubectl apply -f k8s/redis.yaml
kubectl apply -f k8s/server.yaml
kubectl apply -f k8s/webclient.yaml
```

## Database Migrations

### Run migrations:
```bash
pnpm -C apps/server migrate
```

### Rollback:
```bash
# Manually connect and restore from backup
psql $DATABASE_URL -f backup.sql
```

## Monitoring & Health Checks

### Health Check Endpoints

- **Server API:** `GET /healthz` (should return 200)
- **Server Version:** `GET /version` (returns build info)
- **Webclient:** `GET /health` (nginx health)

### Logs

**Docker:**
```bash
docker-compose -f docker-compose.prod.yml logs -f --tail=100
```

**PM2:**
```bash
pm2 logs bcw-server
```

### Metrics

Monitor these key metrics:
- Request rate (API endpoint hits)
- WebSocket connection count
- Matchmaking queue size
- Average match duration
- Error rate
- Database connection pool usage
- Redis memory usage

## Backup & Recovery

### Database Backup

**Create backup:**
```bash
pg_dump $DATABASE_URL > backup_$(date +%Y%m%d_%H%M%S).sql
```

**Restore backup:**
```bash
psql $DATABASE_URL < backup_20250128_120000.sql
```

### Redis Backup

Redis automatically saves snapshots to `/data/dump.rdb`

## Scaling

### Horizontal Scaling (Multiple Server Instances)

1. **Enable cluster mode in server config:**
```env
CLUSTER_MODE=true
WORKER_PROCESSES=4
```

2. **Use sticky sessions for WebSocket connections:**
```nginx
upstream bcw_backend {
    ip_hash;  # Sticky sessions
    server server1:2567;
    server server2:2567;
    server server3:2567;
}
```

3. **Shared Redis for matchmaking state**

### Vertical Scaling

- Increase PostgreSQL max_connections
- Add more Redis memory
- Increase Node.js heap size: `NODE_OPTIONS=--max-old-space-size=4096`

## Troubleshooting

### Server won't start

1. Check environment variables are set
2. Verify PostgreSQL and Redis are accessible
3. Check logs for errors: `docker-compose logs server`

### WebSocket connection fails

1. Verify CORS_ORIGIN matches your frontend URL
2. Check firewall allows port 2567
3. Ensure WebSocket upgrade headers are passed through proxy

### Build version mismatch

1. Update BUILD_VERSION in server .env
2. Update VITE_BUILD_VERSION in webclient .env
3. Rebuild both server and webclient
4. Clients will be rejected until versions match

### Database connection pool exhausted

1. Increase DATABASE_MAX_CONNECTIONS
2. Check for connection leaks in code
3. Monitor active connections:
```sql
SELECT count(*) FROM pg_stat_activity;
```

## Security Checklist

- [ ] JWT_SECRET is strong (min 32 characters)
- [ ] DATABASE_URL uses SSL in production
- [ ] REDIS_URL uses TLS in production
- [ ] CORS_ORIGIN is set to your domain
- [ ] Rate limiting is enabled
- [ ] HTTPS is enforced
- [ ] Database credentials are rotated regularly
- [ ] Server logs don't contain sensitive data
- [ ] Environment variables are not committed to git

## CI/CD

The GitHub Actions workflow (`.github/workflows/ci.yml`) automatically:
- Runs all tests (unit, integration, E2E)
- Type-checks all packages
- Builds all artifacts
- Uploads build artifacts

To deploy on push to main:
1. Configure deployment secrets in GitHub
2. Add deployment step to workflow
3. Push to main branch

## Performance Tuning

### Database Optimization

```sql
-- Add indexes for common queries
CREATE INDEX idx_players_mmr ON players(mmr);
CREATE INDEX idx_matches_created_at ON matches(created_at DESC);
```

### Redis Optimization

```bash
# Increase max memory
redis-cli CONFIG SET maxmemory 2gb
redis-cli CONFIG SET maxmemory-policy allkeys-lru
```

### Node.js Optimization

```bash
# Use production mode
NODE_ENV=production

# Increase heap size
NODE_OPTIONS=--max-old-space-size=4096
```

## Support

For deployment issues:
- Check logs first
- Review this documentation
- Open an issue on GitHub
- Contact devops team
