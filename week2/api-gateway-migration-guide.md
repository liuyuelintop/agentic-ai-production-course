# API Gateway Migration Guide: us-east-1 → ap-southeast-2

> **Related Tutorials:**
> - [Day 2: Deploy Your Digital Twin to AWS](./day2.md)
> - [S3 Bucket Migration Guide](./s3-bucket-migration-guide.md)

This guide walks you through migrating **only your API Gateway** from us-east-1 to ap-southeast-2. Your Lambda function is already in the correct region (ap-southeast-2), so this is a quick and simple migration!

## Your Current Setup

✅ **Already in ap-southeast-2:**
- Lambda function: `twin-api`
- S3 memory bucket (likely)
- S3 frontend bucket (migrated)

❌ **Wrong region (us-east-1):**
- API Gateway: `twin-api-gateway`
- Invoke URL: `https://8of5sre5v1.execute-api.us-east-1.amazonaws.com`

## Why This is Simple

Since your Lambda is already in ap-southeast-2, you only need to:
1. Create new API Gateway in ap-southeast-2
2. Point it to your existing Lambda function
3. Update frontend to use new API URL
4. Test and cleanup

**No Lambda redeployment, no S3 memory bucket migration needed!**

## Pre-Migration Checklist

**Document your current setup:**

1. **Current API Gateway (us-east-1):**
   - Name: `twin-api-gateway`
   - Invoke URL: `https://8of5sre5v1.execute-api.us-east-1.amazonaws.com`

2. **Current Lambda (ap-southeast-2):** ✅ Already correct!
   - Function name: `twin-api`
   - Region: ap-southeast-2

3. **Verify Lambda region:**
   - Go to AWS Console → **Switch to ap-southeast-2** region
   - Go to Lambda → Confirm `twin-api` exists here

## Step-by-Step Migration

### Phase 1: Create New API Gateway in ap-southeast-2

#### Step 1: Create HTTP API with Integration

1. In AWS Console, **switch to ap-southeast-2 region** (top-right dropdown) ← CRITICAL
2. Search for **API Gateway**
3. Click **Create API**
4. Choose **HTTP API** → **Build**

**Configure integrations:**
5. Click **Add integration**
6. Integration type: **Lambda**
7. **AWS Region: ap-southeast-2** ← Verify this matches!
8. Lambda function: Select **`twin-api`** from dropdown (your existing function)
9. API name: `twin-api-gateway-ap`
10. Click **Next**

#### Step 2: Configure Routes

**Add routes** by clicking **Add route** for each:

**Route 1 (default proxy - update existing):**
- Method: `ANY`
- Resource path: `/{proxy+}`
- Integration target: `twin-api`

**Route 2 (root):**
- Click **Add route**
- Method: `GET`
- Resource path: `/`
- Integration target: `twin-api`

**Route 3 (health check):**
- Click **Add route**
- Method: `GET`
- Resource path: `/health`
- Integration target: `twin-api`

**Route 4 (chat endpoint):**
- Click **Add route**
- Method: `POST`
- Resource path: `/chat`
- Integration target: `twin-api`

**Route 5 (CORS preflight):**
- Click **Add route**
- Method: `OPTIONS`
- Resource path: `/{proxy+}`
- Integration target: `twin-api`

Click **Next**

#### Step 3: Configure Stages

- Stage name: `$default` (leave as is)
- Auto-deploy: ✅ Enabled (leave checked)

Click **Next**

#### Step 4: Review and Create

- Review configuration
- Verify Lambda integration shows `twin-api` in **ap-southeast-2**
- Verify all 5 routes are listed

Click **Create**

#### Step 5: Configure CORS

1. In your new API, go to **CORS** in left menu
2. Click **Configure**
3. Add each setting (type the value and **click Add button**):
   - **Access-Control-Allow-Origin:**
     - Type: `*`
     - **Click Add** ← Must click!
   - **Access-Control-Allow-Headers:**
     - Type: `*`
     - **Click Add** ← Must click!
   - **Access-Control-Allow-Methods:**
     - Type: `*`
     - **Click Add** ← Must click!
   - **Access-Control-Max-Age:** `300`
4. Click **Save**

⚠️ **Important:** You MUST click the "Add" button after typing each value. Just typing without clicking Add won't save it!

#### Step 6: Get Your New API URL

1. Go to **Stages** → **$default** (or click **API details**)
2. Copy your **Invoke URL**
   - Should look like: `https://xxxxxxx.execute-api.ap-southeast-2.amazonaws.com`
3. **Save this URL** - you'll need it for the frontend

Example: `https://abc123xyz.execute-api.ap-southeast-2.amazonaws.com`

#### Step 7: Test New API Gateway

Test with curl to verify it works:

```bash
# Replace with your actual Invoke URL from Step 6
curl https://YOUR-NEW-API-ID.execute-api.ap-southeast-2.amazonaws.com/health
```

**Expected response:**
```json
{"status":"healthy","use_s3":true}
```

If you get this response, your API Gateway is working! ✅

**If you get an error:**
- 403 "Missing Authentication Token" → Check you added `/health` to the URL
- 500 Internal Server Error → Check CloudWatch logs (Lambda → Monitor → Logs)
- CORS error → Review Step 5

### Phase 2: Update Frontend to Use New API

#### Step 8: Find Your Frontend Code

Locate your twin component file:

```bash
cd week2/twin/frontend
find . -name "*.tsx" -o -name "*.ts" | grep -i twin
```

Common locations:
- `components/twin.tsx`
- `app/components/twin.tsx`
- `src/components/twin.tsx`

#### Step 9: Update API URL

1. Open the twin component file
2. Search for: `execute-api` or `/chat` or `fetch(`
3. Find the line that looks like:

**Before:**
```typescript
const response = await fetch('https://8of5sre5v1.execute-api.us-east-1.amazonaws.com/chat', {
```

**After (replace with your Invoke URL from Step 6):**
```typescript
const response = await fetch('https://YOUR-NEW-API-ID.execute-api.ap-southeast-2.amazonaws.com/chat', {
```

4. Save the file

#### Step 10: Rebuild Frontend

```bash
cd week2/twin/frontend

# Rebuild for production
npm run build
```

This creates the `out/` directory with static files.

#### Step 11: Upload to S3

```bash
# Replace with your actual frontend bucket name
aws s3 sync out/ s3://twin-frontend-011528274445-ap/ --delete --region ap-southeast-2
```

The `--delete` flag removes old files that are no longer in your build.

#### Step 12: Invalidate CloudFront Cache

1. Go to CloudFront → Your distribution
2. Click **Invalidations** tab
3. Click **Create invalidation**
4. Object paths: `/*`
5. Click **Create invalidation**
6. Wait for status to change to "Completed" (1-3 minutes)

### Phase 3: Test Complete Setup

#### Step 13: Open Your Site

1. Visit your CloudFront URL: `https://dnrvm6q9a8ngv.cloudfront.net`
   (Use your actual CloudFront domain)

2. **Open Browser DevTools:**
   - Press F12 (Windows/Linux) or Cmd+Option+I (Mac)
   - Go to **Network** tab

#### Step 14: Test Chat Functionality

1. Send a message in the chat
2. Verify you get a response
3. In Network tab, click on the `/chat` request
4. Check the **Request URL** - should show:
   ```
   https://xxxxxxx.execute-api.ap-southeast-2.amazonaws.com/chat
   ```
   ✅ Should say **ap-southeast-2**, NOT us-east-1!

5. Check **Console** tab for errors
   - Should be no CORS errors
   - Should be no 500 errors

#### Step 15: Test Session Persistence

1. Send a few messages back and forth
2. Refresh the page (F5)
3. Send another message
4. Verify the twin remembers the previous conversation

#### Step 16: Complete Testing Checklist

Mark each item as you verify it:

- [ ] CloudFront loads the frontend successfully
- [ ] Chat interface appears correctly
- [ ] Can send messages and receive responses
- [ ] No CORS errors in Console tab
- [ ] Network tab shows requests to **ap-southeast-2** (not us-east-1)
- [ ] Response status is 200 (not 403, 500, etc.)
- [ ] Session memory persists after page refresh
- [ ] Browser works in incognito/private mode
- [ ] No errors in Lambda CloudWatch logs

### Phase 4: Update Lambda CORS (Optional Security)

#### Step 17: Restrict CORS to CloudFront Only

For better security, restrict API access to only your CloudFront domain:

1. Go to Lambda (in ap-southeast-2) → `twin-api`
2. Click **Configuration** tab → **Environment variables**
3. Click **Edit**
4. Find `CORS_ORIGINS` variable
5. Update value:
   - **From:** `*` (allows all origins)
   - **To:** `https://dnrvm6q9a8ngv.cloudfront.net` (your CloudFront domain)
6. Click **Save**

**Note:** If `CORS_ORIGINS` doesn't exist, add it:
- Click **Add environment variable**
- Key: `CORS_ORIGINS`
- Value: `https://YOUR-CLOUDFRONT-DOMAIN.cloudfront.net`

This prevents unauthorized sites from calling your API.

### Phase 5: Cleanup Old API Gateway (After 24-48 Hours)

⚠️ **Wait 24-48 hours before deleting** to ensure everything works perfectly!

#### Step 18: Delete Old API Gateway

**Only proceed if Step 16 checklist is 100% complete:**

1. In AWS Console, **switch to us-east-1 region** ← Important!
2. Go to **API Gateway**
3. Find `twin-api-gateway` (the old one)
4. Select it (checkbox)
5. Click **Actions** → **Delete**
6. Type the API name to confirm
7. Click **Delete**

**Why wait?**
- Gives you time to discover any issues
- Allows testing across different devices/browsers
- Provides safety net for rollback if needed

## Rollback Plan (If Something Goes Wrong)

If the new API Gateway doesn't work, you can easily rollback:

### Quick Rollback Steps:

1. **Revert frontend code:**
   ```bash
   cd week2/twin/frontend
   # Edit twin.tsx - change API URL back to us-east-1
   # Update to: https://8of5sre5v1.execute-api.us-east-1.amazonaws.com/chat
   ```

2. **Rebuild and redeploy:**
   ```bash
   npm run build
   aws s3 sync out/ s3://twin-frontend-011528274445-ap/ --delete --region ap-southeast-2
   ```

3. **Invalidate CloudFront:**
   - Create invalidation for `/*`
   - Wait for completion

4. **Test:** Site should work with old API Gateway again

**Why rollback is safe:**
- Old API Gateway remains active until you delete it
- Lambda function unchanged (same in both setups)
- Just a URL change in frontend
- Can switch back and forth for testing

## Common Issues and Solutions

### Issue: CORS Errors in Browser Console

**Symptoms:**
```
Access to fetch at 'https://...' from origin 'https://...' has been blocked by CORS policy
```

**Solutions:**
1. ✅ Check API Gateway CORS configuration (Step 5)
   - Verify you clicked "Add" for each value
   - Go back and re-add if needed
2. ✅ Check Lambda `CORS_ORIGINS` environment variable
   - Should be `*` or include your CloudFront domain
3. ✅ Verify OPTIONS route exists in API Gateway
   - Routes → Should see `OPTIONS /{proxy+}`
4. ✅ Check FastAPI CORS middleware in server.py
   - Should have `allow_origins` set correctly

**Debug command:**
```bash
curl -H "Origin: https://dnrvm6q9a8ngv.cloudfront.net" \
     -H "Access-Control-Request-Method: POST" \
     -H "Access-Control-Request-Headers: Content-Type" \
     -X OPTIONS \
     https://YOUR-API.execute-api.ap-southeast-2.amazonaws.com/chat
```

Should return headers with `access-control-allow-origin`.

### Issue: Still Calling Old API (us-east-1)

**Symptoms:** Network tab shows requests to us-east-1 instead of ap-southeast-2

**Solutions:**
1. ✅ Verify you updated the correct file
   - Make sure you edited the right twin.tsx
2. ✅ Check you actually rebuilt
   - Run `npm run build` again
3. ✅ Verify upload to S3
   - Run `aws s3 sync out/ s3://...` again
4. ✅ Invalidate CloudFront cache
   - Create new invalidation for `/*`
5. ✅ Clear browser cache
   - Hard refresh: Ctrl+Shift+R (Windows) or Cmd+Shift+R (Mac)
   - Or use incognito mode

**Verify built code:**
```bash
cd week2/twin/frontend/out
grep -r "execute-api" .
# Should show ap-southeast-2, not us-east-1
```

### Issue: 500 Internal Server Error

**Symptoms:** API returns 500, chat doesn't work

**Solutions:**
1. Check CloudWatch Logs:
   - Lambda → Monitor → View logs in CloudWatch
   - Look for Python errors
2. Common causes:
   - Missing environment variable (e.g., `OPENAI_API_KEY`)
   - Wrong S3 bucket name in `S3_BUCKET` env var
   - Lambda timeout too short (should be 30 seconds)
   - Missing S3 permissions

### Issue: "Missing Authentication Token"

**Symptoms:** API returns 403 with this message

**Solutions:**
1. ✅ Check you're using correct path
   - Use `/health` not just base URL
   - Example: `https://xxx.execute-api.ap-southeast-2.amazonaws.com/health`
2. ✅ Verify routes are configured
   - API Gateway → Routes → Should see all 5 routes
3. ✅ Check Lambda integration
   - Each route should have integration attached

### Issue: Chat Works But Doesn't Remember

**Symptoms:** Each message is treated as new conversation

**Solutions:**
1. Check Lambda environment variables:
   - `USE_S3` should be `true`
   - `S3_BUCKET` should have correct bucket name
2. Verify Lambda has S3 permissions:
   - Configuration → Permissions → Check execution role
   - Should have `AmazonS3FullAccess` or similar
3. Check S3 bucket:
   ```bash
   aws s3 ls s3://YOUR-MEMORY-BUCKET/ --region ap-southeast-2
   ```
   Should see JSON files appearing after conversations

## Architecture After Migration

```
User Browser
    ↓ HTTPS
CloudFront (Global CDN)
    ↓ HTTP
S3 Frontend Bucket (ap-southeast-2) ✅
    ↓ [Frontend JavaScript makes HTTPS API calls to]
API Gateway (ap-southeast-2) ✅ MIGRATED
    ↓
Lambda Function (ap-southeast-2) ✅ Was already here
    ↓
    ├── OpenAI API (Global)
    └── S3 Memory Bucket (ap-southeast-2) ✅
```

**All resources now in ap-southeast-2 (Sydney)!** 🎉

## Verification Commands

```bash
# Test new API directly
curl https://YOUR-API-ID.execute-api.ap-southeast-2.amazonaws.com/health

# Check S3 frontend bucket region
aws s3api get-bucket-location --bucket twin-frontend-011528274445-ap

# List memory bucket files
aws s3 ls s3://YOUR-MEMORY-BUCKET/ --region ap-southeast-2

# Check built frontend code
cd week2/twin/frontend/out
grep -r "execute-api" . | head -1
```

## What You Changed

✅ **Created:**
- New API Gateway in ap-southeast-2
- New routes pointing to existing Lambda

✅ **Modified:**
- Frontend API URL (twin.tsx)
- Rebuilt and reuploaded frontend to S3

✅ **Unchanged:**
- Lambda function (was already correct)
- S3 memory bucket (was already correct)
- CloudFront distribution
- S3 frontend bucket

## Cost Implications

**During migration (dual API Gateways):**
- API Gateway in us-east-1: ~$1/million requests (minimal)
- API Gateway in ap-southeast-2: ~$1/million requests (minimal)
- Estimated extra cost: **$0-1** (very minimal traffic during migration)

**After cleanup:**
- Same cost as before (just single API Gateway in different region)
- No cost increase from migration

## Learning Outcomes

You've successfully learned:

- ✅ How to create API Gateway HTTP APIs
- ✅ Lambda integration with API Gateway
- ✅ Route configuration (GET, POST, OPTIONS)
- ✅ CORS configuration in API Gateway
- ✅ Testing APIs with curl
- ✅ Frontend-backend integration
- ✅ CloudFront cache invalidation
- ✅ Safe migration with rollback strategy
- ✅ Multi-region AWS architecture principles
- ✅ Debugging with CloudWatch Logs

## Estimated Time

| Task | Duration |
|------|----------|
| Create API Gateway | 10 minutes |
| Update frontend code | 5 minutes |
| Build and deploy | 5 minutes |
| Testing | 5 minutes |
| **Total** | **~25 minutes** |

Much faster than full backend migration! 🚀

## Next Steps

After successful migration:

1. ✅ **Monitor for 24-48 hours** before deleting old API Gateway
2. ✅ **Test from different browsers/devices**
3. ✅ **Update any documentation** with new API URL
4. ✅ **Continue with Day 2 tutorial** - fully in ap-southeast-2!
5. ✅ **Set up CloudWatch alarms** for monitoring (optional)

## Additional Resources

- [API Gateway HTTP APIs Documentation](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api.html)
- [API Gateway CORS Configuration](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-cors.html)
- [Lambda Integration with API Gateway](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-develop-integrations-lambda.html)
- [Troubleshooting API Gateway](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-troubleshooting.html)

---

**Related Guides:**
- [Main Tutorial: Day 2](./day2.md)
- [S3 Bucket Migration Guide](./s3-bucket-migration-guide.md)

**Great job simplifying your migration by already having Lambda in the right region!** 🎯
