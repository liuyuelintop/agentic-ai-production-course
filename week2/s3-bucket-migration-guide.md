# S3 Frontend Bucket Migration Guide: us-east-1 → ap-southeast-2

> **Related Tutorial:** [Day 2: Deploy Your Digital Twin to AWS](./day2.md)

This guide walks you through safely migrating your S3 frontend bucket from the wrong region (us-east-1) to the correct region (ap-southeast-2) without any data loss or downtime.

## Pre-Migration Safety Checklist

**Before starting, document your current setup:**
1. Note your current CloudFront distribution domain name
2. Note your current S3 bucket name: `twin-frontend-011528274445`
3. Save your API Gateway URL (in case you need to reference it)
4. Take screenshots of your working deployment

## Step-by-Step Migration Plan

### Phase 1: Create New S3 Bucket in ap-southeast-2

#### Step 1: Create the new bucket

1. Go to AWS Console → S3
2. Click "Create bucket"
3. Configuration:
   - Bucket name: `twin-frontend-ap-011528274445` (or similar unique name)
   - **Region: ap-southeast-2 (Sydney)** ← CRITICAL
   - **Uncheck "Block all public access"**
   - Check the acknowledgment box
4. Click "Create bucket"

#### Step 2: Enable Static Website Hosting

1. Click on new bucket → Properties tab
2. Scroll to "Static website hosting" → Edit
3. Enable: "Host a static website"
4. Configuration:
   - Index document: `index.html`
   - Error document: `404.html`
5. Save changes
6. **COPY the Bucket website endpoint URL**
   - Example: `http://twin-frontend-ap-011528274445.s3-website-ap-southeast-2.amazonaws.com`
   - You'll need this for CloudFront

#### Step 3: Configure Bucket Policy

1. Go to Permissions tab → Bucket policy → Edit
2. Paste this policy (replace `YOUR-NEW-BUCKET-NAME` with your actual bucket name):

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::YOUR-NEW-BUCKET-NAME/*"
        }
    ]
}
```

3. Click "Save changes"

### Phase 2: Copy Files from Old Bucket to New Bucket

Choose one of these options:

#### Option A: Using AWS CLI (Fastest)

```bash
# Sync all files from old bucket to new bucket
aws s3 sync s3://twin-frontend-011528274445/ s3://twin-frontend-ap-011528274445/ --region ap-southeast-2
```

#### Option B: Re-upload from Local (Safest if you have local build)

```bash
cd week2/twin/frontend
npm run build
aws s3 sync out/ s3://twin-frontend-ap-011528274445/ --delete --region ap-southeast-2
```

#### Step 4: Verify Files Were Copied

1. Go to new bucket in AWS Console
2. Check that all files are present (should see index.html, _next/, etc.)
3. Verify file counts match between old and new bucket

### Phase 3: Update CloudFront Distribution

#### Step 5: Update CloudFront Origin

1. Go to CloudFront → Your distribution
2. Go to "Origins" tab
3. Select your current origin → Click "Edit"
4. Update the following:
   - **Origin domain:** Change to your new bucket's website endpoint (WITHOUT `http://`)
     - Example: `twin-frontend-ap-011528274445.s3-website-ap-southeast-2.amazonaws.com`
   - **Origin protocol policy:** Keep as "HTTP only" (CRITICAL - not HTTPS!)
5. Save changes
6. **Wait 5-15 minutes** for CloudFront to redeploy
   - Status will show "Deploying" then change to "Enabled"

#### Step 6: Create CloudFront Invalidation

1. Go to "Invalidations" tab
2. Click "Create invalidation"
3. Object paths: `/*`
4. Click "Create invalidation"
5. Wait for completion (usually 1-3 minutes)

### Phase 4: Test the Migration

#### Step 7: Test Your CloudFront URL

1. Visit your CloudFront domain: `https://YOUR-DISTRIBUTION.cloudfront.net`
2. Test chat functionality thoroughly:
   - Send messages
   - Verify responses come back
   - Check session persistence (refresh page, continue conversation)
3. Open browser developer console (F12)
   - Check for any errors in Console tab
   - Check Network tab for failed requests
4. Test in incognito/private mode to avoid cache issues

#### Step 8: Verify Functionality

Complete this checklist:
- [ ] Chat loads and displays correctly
- [ ] Can send messages successfully
- [ ] Receive responses from your digital twin
- [ ] No CORS errors in browser console
- [ ] Session memory persists across page refreshes
- [ ] All pages/routes load correctly
- [ ] No 404 or 403 errors

### Phase 5: Cleanup (ONLY AFTER CONFIRMING EVERYTHING WORKS)

#### Step 9: Wait Period

**Wait 24-48 hours** to ensure stability before deleting old bucket. This gives you time to:
- Monitor for any issues
- Get user feedback (if applicable)
- Verify everything works across different browsers/devices

#### Step 10: Delete Old Bucket

**Only proceed if Step 8 checklist is 100% complete:**

1. Go to S3 → `twin-frontend-011528274445`
2. Click "Empty" button first
   - Type confirmation text
   - Click "Empty"
   - Wait for all objects to be deleted
3. Then click "Delete" button on the bucket itself
   - Type bucket name to confirm
   - Click "Delete bucket"

## Rollback Plan (If Something Goes Wrong)

**If the new setup doesn't work, you can easily rollback:**

1. Go to CloudFront → Your distribution → Origins tab
2. Select origin → Click "Edit"
3. Change origin domain back to old bucket website endpoint:
   - `twin-frontend-011528274445.s3-website-us-east-1.amazonaws.com`
4. Save changes
5. Create new invalidation: `/*`
6. Wait for CloudFront to redeploy (5-15 minutes)
7. Your site will be back to working state

**Why this is safe:**
- Old bucket remains intact until you manually delete it
- CloudFront change is just updating a pointer
- No data is lost during the process

## Important Notes

### Safety Features
- ✅ **No downtime risk:** CloudFront continues serving cached content during migration
- ✅ **Old bucket stays intact:** We only update CloudFront pointer, old bucket remains until you manually delete it
- ✅ **Reversible:** Can rollback by pointing CloudFront back to old bucket
- ✅ **Cost:** Running two buckets temporarily costs only cents per day

### Why ap-southeast-2?
While CloudFront makes the bucket region less critical for frontend hosting, having resources in the correct region is important for:
- Learning best practices
- Future optimizations
- Consistency with other AWS resources you may add later
- Compliance requirements (if any)

## Common Pitfalls to Avoid

| ❌ DON'T | ✅ DO |
|---------|-------|
| Delete old bucket before testing new setup | Keep old bucket until 100% confident |
| Forget to set bucket to public | Uncheck "Block all public access" |
| Use S3 bucket URL directly | Use S3 **website endpoint** URL |
| Select HTTPS for origin protocol | Use HTTP only (S3 static hosting limitation) |
| Skip CloudFront invalidation | Always create invalidation after changes |
| Test immediately after changes | Wait for CloudFront "Enabled" status |
| Test with browser cache | Use incognito/private mode |

## Estimated Time

| Phase | Duration |
|-------|----------|
| Creating bucket | 5 minutes |
| Copying files | 2-5 minutes |
| CloudFront update | 10-15 minutes |
| Testing | 5 minutes |
| **Total** | **~30 minutes** |

## Troubleshooting

### Issue: CloudFront shows 504 Gateway Timeout

**Cause:** Origin protocol is set to HTTPS instead of HTTP

**Solution:**
1. Go to CloudFront → Origins → Edit
2. Change "Origin protocol policy" to "HTTP only"
3. Save and wait for redeployment

### Issue: 403 Forbidden errors

**Cause:** Bucket policy not configured or bucket is not public

**Solution:**
1. Check bucket policy is correctly set (Step 3)
2. Verify "Block all public access" is OFF
3. Check bucket website endpoint works directly in browser

### Issue: Files not found (404)

**Cause:** Files weren't copied correctly or static website hosting not enabled

**Solution:**
1. Verify static website hosting is enabled
2. Check index.html exists in bucket root
3. Re-sync files from old bucket or rebuild and re-upload

### Issue: Changes not appearing

**Cause:** CloudFront cache or browser cache

**Solution:**
1. Create CloudFront invalidation for `/*`
2. Clear browser cache or use incognito mode
3. Wait 5-10 minutes for CloudFront propagation

## Learning Outcomes

By completing this migration, you've learned:
- ✅ How to create and configure S3 buckets in specific regions
- ✅ S3 static website hosting setup
- ✅ How to update CloudFront distributions
- ✅ CloudFront invalidation and cache management
- ✅ Safe migration practices with rollback plans
- ✅ AWS CLI commands for S3 operations
- ✅ Troubleshooting CloudFront and S3 integration

## Next Steps

After successful migration:
1. Continue with [Day 2 tutorial](./day2.md#part-9-test-everything) from Part 9
2. Update any documentation with new bucket name
3. Consider setting up S3 lifecycle policies for cost optimization
4. Monitor CloudWatch metrics for your distribution

---

**References:**
- [Main Tutorial: Day 2](./day2.md)
- [AWS S3 Static Website Hosting](https://docs.aws.amazon.com/AmazonS3/latest/userguide/WebsiteHosting.html)
- [CloudFront Documentation](https://docs.aws.amazon.com/cloudfront/)
