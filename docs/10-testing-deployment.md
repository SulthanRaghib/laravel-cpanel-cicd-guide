## Testing & Monitoring

### 1. Test Deployment

Di **local computer** Anda:

```bash
# Make a change
echo "# Test CI/CD" >> README.md

# Commit and push
git add .
git commit -m "test: CI/CD auto deployment"
git push origin main  # or master
```

### 2. Monitor Logs (Real-time)

Via SSH di cPanel:

```bash
# Watch webhook log
tail -f ~/public_html/webhook.log

# Watch deployment log
tail -f ~/public_html/deployment.log
```

Press `Ctrl+C` to stop monitoring.

### 3. Check Webhook Deliveries

Di GitHub:

1. **Settings** → **Webhooks** → Click your webhook
2. **Recent Deliveries** tab
3. Click latest request
4. Check **Response** tab (should be 200 with "success")

### 4. View Log History

```bash
# Last 50 lines
tail -50 ~/public_html/webhook.log

# Search for errors
grep -i "error" ~/public_html/deployment.log

# Search for success
grep -i "success" ~/public_html/webhook.log
```
