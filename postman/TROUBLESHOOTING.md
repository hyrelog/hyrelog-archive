# Postman Troubleshooting Guide

## Common Issues and Solutions

### Issue: "Invalid API key" / "UNAUTHORIZED" Error

**Symptom:**
- Getting `401 UNAUTHORIZED` with `"error": "Invalid API key"`
- Logs show `plaintextPrefix: "{{company_key}}..."` (variable not replaced)

**Cause:**
The Postman environment variable `{{company_key}}` is not being replaced with the actual value.

**Solutions:**

#### 1. Check Environment is Selected

1. In Postman, look at the **top right corner**
2. Make sure **"HyreLog Local"** environment is selected from the dropdown
3. If it's not selected, click the dropdown and select it

#### 2. Verify Variable is Set

1. Click the **eye icon** (👁️) next to the environment dropdown
2. Or go to **Environments** → **HyreLog Local**
3. Check that `company_key` has a **value** (not empty)
4. The value should start with `hlk_co_` or `hlk_ws_` (the full key from seed output)

#### 3. Check Variable Name

Make sure the variable name in the request matches exactly:
- ✅ Correct: `{{company_key}}`
- ❌ Wrong: `{{companyKey}}`, `{{company-key}}`, `{{COMPANY_KEY}}`

#### 4. Verify You Copied the Full Key

The key should be the **complete key** from seed output, for example:
```
hlk_co_abc123def456ghi789jkl012mno345pqr678stu901vwx234yz
```

**Not just:**
- ❌ The prefix: `hlk_co_abc123`
- ❌ The variable name: `{{company_key}}`
- ❌ A partial key

#### 5. Check Authorization Header Format

In the request, the Authorization header should be:
```
Bearer {{company_key}}
```

**Not:**
- ❌ `Bearer company_key`
- ❌ `{{company_key}}`
- ❌ `Bearer hlk_co_abc123` (hardcoded)

#### 6. Re-import Environment (if needed)

If variables still aren't working:
1. Export your current environment (if you've made changes)
2. Re-import `HyreLog Local.postman_collection.json`
3. Set the values again

### Issue: "API key not found in any region"

**Symptom:**
- Logs show: `API key not found in any region`
- `regionsSearched: 4`

**Cause:**
The API key doesn't exist in the database, or the key hash doesn't match.

**Solutions:**

1. **Re-run seed script:**
   ```powershell
   npm run seed
   ```
   This creates fresh keys. Copy the new keys to Postman.

2. **Verify key is in database:**
   - Use Prisma Studio: `npm run prisma:studio`
   - Check `api_keys` table
   - Verify the `prefix` matches your key's prefix

3. **Check you're using the right key:**
   - `company_key` for company operations
   - `workspace_key` for workspace operations

### Issue: Variables Not Updating

**Symptom:**
- Changed environment variable but request still uses old value

**Solutions:**

1. **Save the environment:**
   - After changing a variable, click **Save** (or Ctrl+S)

2. **Refresh the request:**
   - Close and reopen the request tab
   - Or click the **refresh icon** next to the URL

3. **Check for multiple environments:**
   - Make sure you're editing the **correct** environment
   - Postman can have multiple environments

### Issue: "Workspace not found" or "Company not found"

**Symptom:**
- Getting `404 NOT_FOUND` errors

**Cause:**
The `workspace_id` or `company_id` in Postman doesn't match what's in the database.

**Solutions:**

1. **Re-run seed script:**
   ```powershell
   npm run seed
   ```

2. **Copy fresh IDs:**
   - Copy `Company ID`, `Workspace ID`, `Project ID` from seed output
   - Update Postman environment variables

3. **Verify IDs are correct:**
   - Should be UUIDs like: `123e4567-e89b-12d3-a456-426614174000`
   - Not empty strings

### Issue: "IP allowlist required"

**Symptom:**
- Getting `403 FORBIDDEN` with "IP allowlist required"

**Cause:**
Company key doesn't have IP allowlist configured (required for key management operations).

**Solutions:**

1. **Add your IP to the key:**
   - Use Prisma Studio: `npm run prisma:studio`
   - Find your company key in `api_keys` table
   - Edit `ipAllowlist` field: add `["127.0.0.1", "::1"]` for localhost

2. **Or update via seed script:**
   - The seed script should already add localhost IPs
   - Re-run: `npm run seed`

### Issue: "Plan restricted" / "Webhooks require Growth plan"

**Symptom:**
- Getting `403 FORBIDDEN` with "Webhooks require a Growth plan or higher"

**Cause:**
Company is on FREE or STARTER plan, but trying to create webhooks.

**Solutions:**

1. **Seed with GROWTH plan:**
   ```powershell
   $env:SEED_PLAN_TIER="GROWTH"
   npm run seed
   ```

2. **Update Postman:**
   - Set `plan_tier` = `GROWTH` in environment

3. **Or update company in database:**
   - Use Prisma Studio
   - Change `Company.planTier` to `GROWTH` or `ENTERPRISE`

## Quick Diagnostic Checklist

When troubleshooting, check:

- [ ] Environment is selected (top right dropdown)
- [ ] Variable has a value (not empty)
- [ ] Variable name matches exactly (case-sensitive)
- [ ] Full key is copied (not just prefix)
- [ ] Authorization header format: `Bearer {{company_key}}`
- [ ] Seed script was run recently
- [ ] Database is running (`docker ps`)
- [ ] API server is running (`npm run dev`)

## Still Not Working?

1. **Check server logs:**
   - Look at the terminal running `npm run dev`
   - Check for error messages

2. **Verify database connection:**
   ```powershell
   npm run prisma:studio
   ```
   - If Prisma Studio opens, database is connected
   - Check `api_keys` table for your keys

3. **Test with curl:**
   ```powershell
   $key = "hlk_co_your_actual_key_here"
   curl -H "Authorization: Bearer $key" http://localhost:3000/v1/keys/status
   ```
   - If curl works but Postman doesn't, it's a Postman configuration issue
   - If curl also fails, it's a key/database issue

4. **Check Postman console:**
   - View → Show Postman Console
   - Check what's actually being sent in the request
   - Verify the Authorization header value

