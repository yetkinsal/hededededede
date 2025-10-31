# 🚀 Easy Deployment Guide

Step-by-step guide to deploy Brick City Wars to Railway (backend) and Vercel (web frontend).

---

## 📋 What You Need Before Starting

- [ ] Railway account (sign up at railway.app)
- [ ] Vercel account (sign up at vercel.com)
- [ ] Your code pushed to GitHub
- [ ] 10 minutes of your time

---

## Part 1: Deploy Backend to Railway (15 minutes)

### Step 1: Create a New Railway Project

1. Go to https://railway.app
2. Click **"New Project"**
3. Select **"Deploy from GitHub repo"**
4. Choose your `brickcitywars` repository
5. Click **"Deploy Now"**

### Step 2: Add PostgreSQL Database

1. In your Railway project, click **"+ New"**
2. Select **"Database"**
3. Choose **"PostgreSQL"**
4. Railway will automatically create the database
5. Wait 30 seconds for it to be ready

### Step 3: Add Redis

1. Click **"+ New"** again
2. Select **"Database"**
3. Choose **"Redis"**
4. Wait 30 seconds for it to be ready

### Step 4: Configure Your Server

1. Click on your server deployment (the one from GitHub)
2. Go to **"Variables"** tab
3. Click **"+ New Variable"** and add these one by one:

```
NODE_ENV = production
PORT = 3000
COLYSEUS_PORT = 2567
BUILD_VERSION = 1.0.3
JWT_SECRET = your-super-secret-key-min-32-characters-long
LOG_LEVEL = info
```

**Important:** Change `your-super-secret-key-min-32-characters-long` to a random string!

### Step 5: Connect Database URLs

Railway automatically creates these variables when you add databases:
- `DATABASE_URL` - PostgreSQL connection
- `REDIS_URL` - Redis connection

**You don't need to add these manually!** Railway does it for you.

### Step 6: Set the Root Directory

1. Still in **"Variables"** tab, scroll down
2. Find **"Root Directory"** setting
3. Set it to: `apps/server`
4. Click **"Save"**

### Step 7: Configure Build Command

1. Go to **"Settings"** tab
2. Scroll to **"Build Command"**
3. Set it to:
```
cd ../.. && pnpm install && pnpm -C packages/shared build && pnpm -C apps/server build
```

### Step 8: Configure Start Command

1. Still in **"Settings"** tab
2. Find **"Start Command"**
3. Set it to:
```
node dist/index.js
```

### Step 9: Run Database Migration

1. Click on **"Deployments"** tab
2. Wait for the build to finish (green checkmark)
3. Once deployed, click on your service
4. Find the **"Connect"** button and copy your **Railway public URL**
5. Open your terminal and run:

```bash
# Set the DATABASE_URL temporarily
export DATABASE_URL="your-postgresql-url-from-railway"

# Run migrations
pnpm -C apps/server migrate
```

**To get the DATABASE_URL:**
- Click on the PostgreSQL database in Railway
- Go to "Connect" tab
- Copy the "Postgres Connection URL"

### Step 10: Get Your Server URL

1. Go back to your server deployment
2. Click **"Settings"** tab
3. Scroll to **"Domains"**
4. You'll see something like: `your-app-name.up.railway.app`
5. **Copy this URL** - you'll need it for the frontend!

### ✅ Backend is Done!

Your server should now be running at `https://your-app-name.up.railway.app`

Test it by visiting: `https://your-app-name.up.railway.app/healthz`

You should see: `{"status":"healthy"}`

---

## Part 2: Deploy Frontend to Vercel (5 minutes)

### Step 1: Update Environment Variables

1. Open `apps/webclient/.env.production` in your code editor
2. Update with your Railway URL:

```env
# Replace with YOUR Railway URL from Part 1, Step 10
VITE_API_BASE=https://your-app-name.up.railway.app
VITE_WS_URL=https://your-app-name.up.railway.app

VITE_BUILD_VERSION=1.0.3
VITE_ENABLE_ANALYTICS=false
VITE_ENABLE_ERROR_REPORTING=false
```

3. **Save the file**
4. Push to GitHub:

```bash
git add apps/webclient/.env.production
git commit -m "Update production URLs for Railway backend"
git push
```

### Step 2: Create Vercel Project

1. Go to https://vercel.com
2. Click **"Add New..."** → **"Project"**
3. Click **"Import"** next to your `brickcitywars` repository
4. Click **"Import"**

### Step 3: Configure Build Settings

Vercel will show you a configuration screen:

1. **Framework Preset**: Select **"Vite"**

2. **Root Directory**: Click **"Edit"** and set to: `apps/webclient`

3. **Build Command**: Set to:
```
pnpm install && pnpm --filter @brick-city-wars/shared build && pnpm build
```

4. **Output Directory**: Leave as `dist` (default for Vite)

5. **Install Command**: Set to:
```
pnpm install
```

### Step 4: Add Environment Variables (in Vercel)

1. Scroll down to **"Environment Variables"** section
2. Add these variables:

```
VITE_API_BASE = https://your-app-name.up.railway.app
VITE_WS_URL = https://your-app-name.up.railway.app
VITE_BUILD_VERSION = 1.0.3
```

**Replace** `your-app-name.up.railway.app` with YOUR Railway URL!

### Step 5: Deploy!

1. Click **"Deploy"** button
2. Wait 2-3 minutes for the build to complete
3. You'll see a success screen with your URL!

### ✅ Frontend is Done!

Your game is now live at: `https://your-app-name.vercel.app`

---

## Part 3: Update CORS Settings on Railway

**Important:** You need to tell your backend to accept requests from your Vercel URL!

### Step 1: Add CORS Variable

1. Go back to Railway
2. Click on your server deployment
3. Go to **"Variables"** tab
4. Add a new variable:

```
CORS_ORIGIN = https://your-app-name.vercel.app
```

**Replace** `your-app-name.vercel.app` with YOUR Vercel URL!

### Step 2: Wait for Redeploy

Railway will automatically redeploy with the new variable. Wait 1-2 minutes.

### ✅ CORS is Done!

---

## 🎮 Part 4: Test Your Game!

### Step 1: Open Your Game

1. Go to your Vercel URL: `https://your-app-name.vercel.app`
2. You should see the Brick City Wars menu!

### Step 2: Test Authentication

1. Click **"Login / Sign Up"** button
2. Click **"Don't have an account? Sign up"**
3. Enter:
   - Email: `test@example.com`
   - Username: `testplayer`
   - Password: `testpass123`
4. Click **"Sign Up"**
5. You should be logged in and see your username!

### Step 3: Test Matchmaking

1. Click the **"Play"** button
2. Wait 15 seconds (you'll be matched with a bot)
3. Game should start!

### Step 4: Test Other Features

- Click **"🏆 Leaderboard"** - should load the leaderboard
- Click **"👤 My Profile"** - should show your stats
- Play a match and check if MMR updates

---

## 🐛 Troubleshooting

### Problem: "Cannot connect to server"

**Solution:**
1. Check if Railway server is running (go to Railway → Deployments → should be green)
2. Check `VITE_API_BASE` in Vercel matches your Railway URL exactly
3. Check `CORS_ORIGIN` in Railway matches your Vercel URL exactly

### Problem: "Build failed" on Vercel

**Solution:**
1. Check your build command includes `pnpm install`
2. Make sure Root Directory is set to `apps/webclient`
3. Check the error logs in Vercel deployment page

### Problem: "Database error" on Railway

**Solution:**
1. Make sure you ran the database migration (Part 1, Step 9)
2. Check that PostgreSQL and Redis are both running in Railway
3. Check Railway logs for errors

### Problem: Login doesn't work

**Solution:**
1. Check `JWT_SECRET` is set in Railway
2. Check that database migration ran successfully
3. Check Railway logs for authentication errors

### Problem: WebSocket connection fails

**Solution:**
1. Check `VITE_WS_URL` matches your Railway URL
2. Railway should automatically handle WebSocket connections
3. Make sure `COLYSEUS_PORT = 2567` is set in Railway

---

## 📱 Part 5: Deploy to TestFlight (LATER)

Once your web version works perfectly, you can deploy to TestFlight:

### Quick Steps:

1. **Update mobile environment**:
   - Edit `apps/mobile/capacitor.config.ts` (no changes needed, uses webclient build)

2. **Build webclient**:
```bash
pnpm -C apps/webclient build
```

3. **Sync with iOS**:
```bash
pnpm -C apps/mobile sync
pnpm -C apps/mobile ios
```

4. **In Xcode**:
   - Select your team
   - Archive the app
   - Upload to TestFlight

5. **Wait 24 hours** for Apple review

📖 **Detailed mobile deployment guide**: See `apps/mobile/DEPLOY_TO_TESTFLIGHT.md`

---

## 🎉 You're Done!

Your game is now live on the internet!

**URLs to share:**
- Web: `https://your-app-name.vercel.app`
- API: `https://your-app-name.up.railway.app`

**Next steps:**
- Share with friends to test
- Monitor Railway logs for errors
- Check Vercel analytics
- When everything works, deploy to TestFlight!

---

## 🆘 Need Help?

**Check logs:**
- Railway: Go to your deployment → "Logs" tab
- Vercel: Go to your deployment → "Logs" tab

**Common URLs to test:**
- Health check: `https://your-railway-url.up.railway.app/healthz`
- Version: `https://your-railway-url.up.railway.app/version`
- Queue stats: `https://your-railway-url.up.railway.app/api/queue/stats`

**Still stuck?**
- Check Railway logs for backend errors
- Check browser console (F12) for frontend errors
- Make sure all environment variables are set correctly
