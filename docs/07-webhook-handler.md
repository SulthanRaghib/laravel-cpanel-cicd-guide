# 07 - Buat Webhook Handler

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
 * For Laravel on cPanel with Disabled Execution Functions
 *
 * Uses Flag System + Cron Job for deployment
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
// DEPLOYMENT VIA FLAG SYSTEM
// =========================================

// Check deploy script exists
if (!file_exists(DEPLOY_SCRIPT)) {
    writeLog('❌ ERROR: Deploy script not found', ['path' => DEPLOY_SCRIPT]);
    sendResponse('error', 'Deploy script not found', ['path' => DEPLOY_SCRIPT], 500);
}

// Create deployment flag file (since exec/shell_exec disabled)
$flagFile = PROJECT_PATH . '/deploy.flag';
$flagData = [
    'timestamp' => time(),
    'branch' => $branch,
    'commit' => $commitInfo,
    'triggered_at' => date('Y-m-d H:i:s'),
    'delivery_id' => $delivery
];

// Write flag file
$flagWritten = @file_put_contents($flagFile, json_encode($flagData, JSON_PRETTY_PRINT));

if ($flagWritten === false) {
    writeLog('❌ ERROR: Cannot create deploy flag', [
        'flag_path' => $flagFile,
        'directory_writable' => is_writable(PROJECT_PATH)
    ]);
    sendResponse('error', 'Cannot create deployment flag', null, 500);
}

writeLog('✅ Deployment flag created', [
    'flag_file' => $flagFile,
    'commit' => $commitInfo['id'],
    'branch' => $branch,
    'note' => 'Waiting for cron job to execute deployment'
]);

sendResponse('success', 'Deployment queued successfully!', [
    'flag_created' => true,
    'commit' => $commitInfo,
    'branch' => $branch,
    'note' => 'Deployment will be executed by cron job within 1 minute',
    'queued_at' => date('Y-m-d H:i:s')
]);
?>
```

⚠️ **PENTING: Edit nilai berikut:**

- `SECRET_TOKEN` - Paste token dari `openssl rand -hex 32`
- `PROJECT_PATH` - Path ke project Anda (contoh: `/home/jasz5267/public_html`)
- `BRANCH_TO_DEPLOY` - `'main'` atau `'master'`

**Save:** Ctrl+X, Y, Enter

---
