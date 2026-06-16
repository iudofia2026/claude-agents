# How to Create Admin Accounts

If you're having trouble logging in with an admin account (like "brittany"), the account may not exist in the database yet. Here's how to create all admin accounts:

## Quick Fix: Create All Admin Accounts

### Option 1: Using the Setup Endpoint (Recommended)

1. **Make sure your development server is running:**
   ```bash
   npm run dev
   ```

2. **Call the setup endpoint** using one of these methods:

   **Using curl:**
   ```bash
   curl -X POST http://localhost:3000/api/admin/setup \
     -H "Content-Type: application/json"
   ```

   **Using the setup script:**
   ```bash
   ./setup-admin.sh
   ```

   **Or visit in browser (if GET is supported):**
   ```
   http://localhost:3000/api/admin/setup
   ```

3. **Check the response** - it should show:
   ```json
   {
     "message": "Admin users processed successfully",
     "results": [
       "Created admin user: brian",
       "Created admin user: isiah",
       "Created admin user: luis",
       "Created admin user: brittany"
     ],
     "credentials": [...]
   }
   ```

### Option 2: Using the Init Users Endpoint

```bash
curl -X POST http://localhost:3000/api/admin/init-users \
  -H "Content-Type: application/json"
```

### Option 3: Using Node Scripts

```bash
# JavaScript version
node scripts/create-admin.js

# TypeScript version (if you have ts-node)
npx ts-node scripts/create-admin.ts
```

## Verify Accounts Were Created

You can check which accounts exist by calling:

```bash
curl http://localhost:3000/api/admin/debug-users
```

This will show you all admin users in the database.

## Login Credentials

After creating the accounts, use these credentials:

| Username | Password |
|----------|----------|
| brian | password |
| isiah | password |
| luis | password |
| brittany | password |

**Important:** 
- Usernames are **case-sensitive** - use lowercase
- All accounts use the same password: `password`

## Troubleshooting

### "Invalid username or password" Error

1. **Verify the account exists:**
   ```bash
   curl http://localhost:3000/api/admin/debug-users
   ```

2. **If brittany doesn't appear**, run the setup endpoint again:
   ```bash
   curl -X POST http://localhost:3000/api/admin/setup
   ```

3. **Check for typos:**
   - Username must be exactly: `brittany` (lowercase)
   - Password must be exactly: `password` (lowercase)

### Account Exists But Still Can't Login

1. **Check the database connection** - make sure `POSTGRES_URL` is set in your `.env.local`
2. **Check server logs** for authentication errors
3. **Try resetting the password** by running the setup endpoint again (it will update existing accounts)

### For Production

If you're in production, make sure to:
1. Set environment variables in Vercel dashboard
2. Call the setup endpoint with your production URL
3. Or use the production script: `scripts/create-prod-admins.ts`




