# 11 - Testing & Monitoring

### 1. Test Full Deployment Flow

**Step 1: Trigger webhook via curl**

```bash
curl -X POST "https://yourdomain.com/deploy.php?token=YOUR_TOKEN" \
  -H "X-GitHub-Event: push" \
  -H "Content-Type: application/json" \
  -d '{"ref":"refs/heads/master","head_commit":{"id":"test","message":"Test deploy","author":{"name":"Test"}}}'
```

**Expected Response:**

```json
{
    "status": "success",
    "message": "Deployment queued successfully!",
    "data": {
        "flag_created": true,
        "note": "Deployment will be executed by cron job within 1 minute",
        ...
    }
}
```

**Step 2: Check flag created**

```bash
ls -la ~/public_html/deploy.flag
```

**Step 3: Wait 60 seconds for cron**

```bash
sleep 65
```

**Step 4: Verify deployment**

```bash
# Flag should be removed
ls -la ~/public_html/deploy.flag  # Should not exist

# Check cron log
tail -30 ~/public_html/cron-deploy.log

# Check deployment log
tail -30 ~/public_html/deployment.log
```

### 2. Test dengan GitHub Push

Di **local computer**:

```bash
# Make a change
echo "# CI/CD Test" >> README.md

# Commit and push
git add .
git commit -m "test: CI/CD auto deployment"
git push origin master  # or main
```

### 3. Monitor Logs (Real-time)

Via SSH:

```bash
# Watch all logs
tail -f ~/public_html/webhook.log ~/public_html/cron-deploy.log ~/public_html/deployment.log
```

Press `Ctrl+C` to stop.

### 4. Check GitHub Webhook Status

1. **GitHub** → **Settings** → **Webhooks**
2. Click your webhook
3. **Recent Deliveries** tab
4. Latest request should show:
   - ✅ Status 200
   - Response body: `"status": "success"`
