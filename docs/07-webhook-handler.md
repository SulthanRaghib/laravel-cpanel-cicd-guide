# 07 - Webhook Handler

## Buat Webhook Handler

### 1. Generate Secret Token

```bash
openssl rand -hex 32
```

**Copy output token** - Anda akan membutuhkannya!

Contoh output:

```
a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6q7r8s9t0u1v2w3x4y5z6a7b8c9d0e1f2
```

### 2. Buat File deploy.php

```bash
cd ~/public_html/public
nano deploy.php
```

⚠️ **PENTING:** File harus di folder `public/` bukan root project!

### 3. Paste Script Berikut

```php
<?php
/**
 * GitHub Webhook Auto Deploy Handler
 * For Laravel on cPanel
 *
 * Security: Only GitHub IPs can access this endpoint
 * Token authentication required
 */

// =========================================
// CONFIGURATION - EDIT THESE VALUES
// =========================================

// Paste your token from: openssl rand -hex 32
define('SECRET_TOKEN', 'PASTE_YOUR_TOKEN_HERE');

define('PROJECT_PATH', '/home/YOUR_USERNAME/public_html');
define('BRANCH_TO_DEPLOY', 'main'); // Change to 'master' if needed

// =========================================
// DO NOT EDIT BELOW THIS LINE
// =========================================

define('DEPLOY_SCRIPT', PROJECT_PATH . '/deploy.sh');
define('LOG_FILE', PROJECT_PATH . '/webhook.log');
define('MAX_LOG_SIZE', 5 * 1024 * 1024); // 5MB

// =========================================
// FUNCTIONS
// =========================================

function writeLog($message, $data = null) {
    $timestamp = date('Y-m-d H:i:s');
    $logMessage = "[$timestamp] $message";

    if ($data !== null) {
        $logMessage .= "\n" . json_encode($data, JSON_PRETTY_PRINT | JSON_UNESCAPED_SLASHES);
    }

    $logMessage .= "\n" . str_repeat('-', 80) . "\n";

    // Rotate log if too large
    if (file_exists(LOG_FILE) && filesize(LOG_FILE) > MAX_LOG_SIZE) {
        rename(LOG_FILE, LOG_FILE . '.' . date('Ymd-His') . '.old');
    }

    file_put_contents(LOG_FILE, $logMessage, FILE_APPEND);
}

function sendResponse($status, $message, $data = null, $code = 200) {
    http_response_code($code);
    header('Content-Type: application/json');

    $response = [
        'status' => $status,
        'message' => $message,
        'timestamp' => date('Y-m-d H:i:s'),
    ];

    if ($data !== null) {
        $response['data'] = $data;
    }

    echo json_encode($response, JSON_PRETTY_PRINT);
    exit;
}

// =========================================
// SECURITY CHECK
// =========================================

// Check token exists
if (!isset($_GET['token'])) {
    writeLog('UNAUTHORIZED: No token provided', [
        'ip' => $_SERVER['REMOTE_ADDR'] ?? 'Unknown',
        'user_agent' => $_SERVER['HTTP_USER_AGENT'] ?? 'Unknown'
    ]);
    sendResponse('error', 'Unauthorized: Token required', null, 403);
}

// Verify token
if ($_GET['token'] !== SECRET_TOKEN) {
    writeLog('UNAUTHORIZED: Invalid token', [
        'ip' => $_SERVER['REMOTE_ADDR'] ?? 'Unknown',
        'provided' => substr($_GET['token'], 0, 10) . '...'
    ]);
    sendResponse('error', 'Unauthorized: Invalid token', null, 403);
}

// =========================================
// WEBHOOK PROCESSING
// =========================================

writeLog('✅ Webhook received - Token verified');

// Get payload
$payload = file_get_contents('php://input');
$data = json_decode($payload, true);

// Get GitHub headers
$event = $_SERVER['HTTP_X_GITHUB_EVENT'] ?? 'unknown';
$delivery = $_SERVER['HTTP_X_GITHUB_DELIVERY'] ?? 'unknown';

writeLog('📨 GitHub Event Received', [
    'event' => $event,
    'delivery_id' => $delivery
]);

// Only process push events
if ($event !== 'push') {
    writeLog("⏭️  Skipped: Not a push event", ['event' => $event]);
    sendResponse('skipped', "Event '$event' ignored. Only 'push' events trigger deployment.");
}

// Check branch
$ref = $data['ref'] ?? '';
$branch = str_replace('refs/heads/', '', $ref);

if ($branch !== BRANCH_TO_DEPLOY) {
    writeLog("⏭️  Skipped: Wrong branch", [
        'expected' => BRANCH_TO_DEPLOY,
        'received' => $branch
    ]);
    sendResponse('skipped', "Push to '$branch' ignored. Only '" . BRANCH_TO_DEPLOY . "' triggers deployment.");
}

// Get commit info
$commit = $data['head_commit'] ?? [];
$commitInfo = [
    'id' => substr($commit['id'] ?? 'unknown', 0, 7),
    'message' => $commit['message'] ?? 'No message',
    'author' => $commit['author']['name'] ?? 'Unknown',
    'timestamp' => $commit['timestamp'] ?? date('c'),
    'url' => $commit['url'] ?? ''
];

writeLog('🚀 Starting deployment', [
    'branch' => $branch,
    'commit' => $commitInfo
]);

// =========================================
// DEPLOYMENT
// =========================================

// Check deploy script exists
if (!file_exists(DEPLOY_SCRIPT)) {
    writeLog('❌ ERROR: Deploy script not found', ['path' => DEPLOY_SCRIPT]);
    sendResponse('error', 'Deploy script not found', ['path' => DEPLOY_SCRIPT], 500);
}

// Execute deployment
$startTime = microtime(true);
$output = shell_exec(DEPLOY_SCRIPT . ' 2>&1');
$executionTime = round(microtime(true) - $startTime, 2);

// Check success
$success = (strpos($output, '✅ Deployment completed successfully!') !== false);

if ($success) {
    writeLog('✅ Deployment SUCCESS', [
        'execution_time' => $executionTime . 's',
        'commit_id' => $commitInfo['id'],
        'author' => $commitInfo['author']
    ]);

    sendResponse('success', 'Deployment completed successfully!', [
        'execution_time' => $executionTime . 's',
        'commit' => $commitInfo,
        'branch' => $branch,
        'deployed_at' => date('Y-m-d H:i:s')
    ]);
} else {
    writeLog('⚠️  Deployment completed with warnings', [
        'execution_time' => $executionTime . 's',
        'output_preview' => substr($output, -500)
    ]);

    sendResponse('warning', 'Deployment executed, check logs for details', [
        'execution_time' => $executionTime . 's'
    ], 200);
}
```

⚠️ **PENTING: Edit nilai berikut:**

- `SECRET_TOKEN` - Paste token dari `openssl rand -hex 32`
- `PROJECT_PATH` - Path ke project Anda
- `BRANCH_TO_DEPLOY` - `'main'` atau `'master'`

**Save:** Ctrl+X, Y, Enter

### 4. Test Webhook Handler

```bash
# Ganti YOUR_TOKEN dengan token Anda
curl "https://yourdomain.com/deploy.php?token=YOUR_TOKEN" \
  -H "X-GitHub-Event: push" \
  -H "Content-Type: application/json" \
  -d '{"ref":"refs/heads/main","head_commit":{"id":"test","message":"Test","author":{"name":"Test"}}}'
```

**Expected response:**

```json
{
    "status": "success",
    "message": "Deployment completed successfully!",
    ...
}
```

---

## Setup GitHub Webhook

### 1. Buka Repository GitHub

Navigate ke repository Anda di GitHub.

### 2. Pergi ke Settings

Klik tab **Settings** di bagian atas repository.

### 3. Pilih Webhooks

Di sidebar kiri, klik **Webhooks** → **Add webhook**

### 4. Konfigurasi Webhook

**Payload URL:**

```
https://yourdomain.com/deploy.php?token=YOUR_SECRET_TOKEN
```

⚠️ Ganti `YOUR_SECRET_TOKEN` dengan token dari `openssl rand -hex 32`

**Content type:**

```
application/json
```

**Secret:**

```
(kosongkan)
```

**SSL verification:**

```
☑️ Enable SSL verification
```

**Which events would you like to trigger this webhook?**

```
☑️ Just the push event
```

**Active:**

```
☑️ Active
```

### 5. Add Webhook

Klik **Add webhook**

### 6. Verify Webhook

GitHub akan otomatis mengirim test ping.

1. Scroll ke bawah ke **Recent Deliveries**
2. Klik request pertama (ping)
3. Check **Response** tab
4. Harus ada response 200 (boleh "skipped" karena bukan push event)
