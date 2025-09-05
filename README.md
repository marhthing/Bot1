# MATDEV WhatsApp Bot - Host Anywhere

<div align="center">

[![Node.js](https://img.shields.io/badge/Node.js-18+-brightgreen.svg)](https://nodejs.org/)
[![Baileys](https://img.shields.io/badge/Baileys-Latest-blue.svg)](https://github.com/WhiskeySockets/Baileys)
[![Deploy](https://img.shields.io/badge/Deploy-One--Click-success.svg)]()

**⚡ High-Performance WhatsApp Bot with Auto-Update System**

*Host on any platform with zero configuration*

</div>

## 🚀 Quick Deploy

### Auto-Manager (Any Host)
Use our auto-manager system to automatically deploy and update your bot:

<div align="center">

**📋 Copy Auto-Manager Code**

<details>
<summary><strong>📋 Click to copy index.js content</strong></summary>

```javascript
const { spawn, spawnSync } = require('child_process');
const { existsSync } = require('fs');
const ManagerCommands = require('./lib/manager');

console.log('🎯 MATDEV Bot Auto-Manager');
console.log('📍 Working in:', __dirname);

// Your GitHub repository
const GITHUB_REPO = 'https://github.com/marhthing/MATDEV-BOT.git';

// Initialize manager commands
const managerCommands = new ManagerCommands(GITHUB_REPO);

// ... (truncated for brevity)
// The complete code is available below for copying
```

**To get the complete auto-manager code:**
1. Select and copy all the text below
2. Create a new `index.js` file on your hosting platform
3. Paste the complete code

```javascript
const { spawn, spawnSync } = require('child_process');
const { existsSync } = require('fs');
const ManagerCommands = require('./lib/manager');

console.log('🎯 MATDEV Bot Auto-Manager');
console.log('📍 Working in:', __dirname);

// Your GitHub repository - UPDATE THIS WITH YOUR ACTUAL REPO URL
const GITHUB_REPO = 'https://github.com/marhthing/MATDEV-BOT.git';

// Initialize manager commands
const managerCommands = new ManagerCommands(GITHUB_REPO);

// Expose essential manager commands globally  
console.log('🔧 Setting up manager commands...');
global.managerCommands = {
    restart: () => managerCommands.restart(),
    shutdown: () => managerCommands.shutdown(),
    checkUpdates: () => managerCommands.checkUpdates(),
    updateNow: () => managerCommands.updateNow()
};

console.log('✅ Manager commands ready and available globally');

// Check if this is an initial setup, restart, or forced update
const isInitialSetup = !existsSync('bot.js') || !existsSync('config.js') || !existsSync('package.json');
const isForcedUpdate = existsSync('.update_flag.json');

if (isInitialSetup || isForcedUpdate) {
    if (isForcedUpdate) {
        console.log('🔄 Forced update detected - recloning from GitHub...');
    } else {
        console.log('🔧 Initial setup detected - cloning from GitHub...');
    }
    cloneAndSetup();
} else {
    console.log('🚀 Starting MATDEV bot...');
    startBot();
}

function cloneAndSetup() {
    console.log('📥 Cloning bot from GitHub...');
    console.log('🔗 Repository:', GITHUB_REPO);

    // Clean workspace (preserve important files)
    console.log('🧹 Cleaning workspace (preserving session folder, .env, and config.js)...');
    spawnSync('bash', ['-c', 'find . -maxdepth 1 ! -name "." ! -name "index.js" ! -name "node_modules" ! -name "session" ! -name ".env" ! -name "config.js" -exec rm -rf {} +'], { stdio: 'inherit' });

    // Clone repository
    const cloneResult = spawnSync('git', ['clone', GITHUB_REPO, 'temp_clone'], {
        stdio: 'inherit'
    });

    if (cloneResult.error || cloneResult.status !== 0) {
        console.error('❌ Failed to clone repository!');
        console.error('Error:', cloneResult.error?.message || `Exit code: ${cloneResult.status}`);
        process.exit(1);
    }

    // Backup and move files
    console.log('📁 Moving bot files (preserving existing .env and config.js)...');
    spawnSync('bash', ['-c', 'cp .env .env.backup 2>/dev/null || true; cp config.js config.js.backup 2>/dev/null || true'], { stdio: 'inherit' });
    
    const moveResult = spawnSync('bash', ['-c', 'cp -r temp_clone/. . && rm -rf temp_clone'], {
        stdio: 'inherit'
    });
    
    spawnSync('bash', ['-c', 'mv .env.backup .env 2>/dev/null || true; mv config.js.backup config.js 2>/dev/null || true'], { stdio: 'inherit' });

    if (moveResult.error || moveResult.status !== 0) {
        console.error('❌ Failed to move bot files!');
        console.error('Error:', moveResult.error?.message || `Exit code: ${moveResult.status}`);
        process.exit(1);
    }

    console.log('✅ Bot files moved successfully!');

    // Find entry point
    let entryPoint = findEntryPoint();
    if (!entryPoint) {
        console.error('❌ No bot entry point found!');
        process.exit(1);
    }
    console.log(`✅ Found bot entry point: ${entryPoint}`);

    // Install dependencies
    if (existsSync('package.json')) {
        console.log('📦 Installing dependencies...');
        const installResult = spawnSync('npm', ['install', '--production'], {
            stdio: 'inherit'
        });

        if (installResult.error || installResult.status !== 0) {
            console.error('❌ Failed to install dependencies');
            process.exit(1);
        }
        console.log('✅ Dependencies installed!');
    }

    // Start the bot
    startBot(entryPoint);
    
    // Send update completion notification after successful setup
    setTimeout(() => {
        managerCommands.sendUpdateCompleteNotification();
    }, 10000);
}

function findEntryPoint() {
    const possibleEntryPoints = ['bot.js', 'app.js', 'main.js', 'src/index.js'];
    
    for (const file of possibleEntryPoints) {
        if (existsSync(file)) {
            return file;
        }
    }

    // Check package.json for main field
    if (existsSync('package.json')) {
        try {
            const packageJson = JSON.parse(require('fs').readFileSync('package.json', 'utf8'));
            if (packageJson.main && existsSync(packageJson.main)) {
                return packageJson.main;
            }
        } catch (err) {
            console.log('⚠️ Could not read package.json main field');
        }
    }

    return null;
}

function startBot(entryPoint = 'bot.js') {
    console.log(`🚀 Starting bot: ${entryPoint}`);

    const botProcess = spawn('node', [entryPoint], {
        stdio: 'inherit'
    });

    let restartCount = 0;
    const maxRestarts = 5;

    botProcess.on('exit', (code, signal) => {
        console.log(`🔄 Bot exited with code ${code}, signal ${signal}`);
        
        if (signal !== 'SIGTERM' && signal !== 'SIGINT') {
            if (code === 0) {
                console.log(`🔄 Restarting bot as requested...`);
                setTimeout(() => startBot(entryPoint), 2000);
            } else {
                // Check for update requests
                const isInitialSetup = !existsSync('bot.js') || !existsSync('config.js') || !existsSync('package.json');
                const isForcedUpdate = existsSync('.update_flag.json');
                
                if (isInitialSetup || isForcedUpdate) {
                    console.log('🔄 Update triggered - initiating recloning process...');
                    cloneAndSetup();
                    return;
                }
                
                restartCount++;
                if (restartCount <= maxRestarts) {
                    console.log(`🔄 Restarting bot after crash... (${restartCount}/${maxRestarts})`);
                    setTimeout(() => startBot(entryPoint), 2000);
                } else {
                    console.error('❌ Too many crash restarts, stopping');
                    process.exit(1);
                }
            }
        } else {
            console.log('🛑 Bot stopped by manager');
        }
    });

    botProcess.on('error', (error) => {
        console.error('❌ Bot start error:', error.message);
    });

    // Handle process signals
    process.on('SIGUSR1', () => {
        console.log('🔄 Received restart signal, restarting bot...');
        botProcess.kill('SIGTERM');
        setTimeout(() => startBot(entryPoint), 2000);
    });

    process.on('SIGTERM', () => {
        console.log('🛑 Received shutdown signal, stopping bot...');
        botProcess.kill('SIGTERM');
        process.exit(0);
    });

    process.on('SIGINT', () => {
        console.log('🛑 Received interrupt signal, stopping bot...');
        botProcess.kill('SIGINT');
        process.exit(0);
    });

    console.log('✅ Bot manager running!');
}

// Prevent manager from exiting unexpectedly  
process.on('uncaughtException', (error) => {
    console.error('❌ Manager uncaught exception:', error);
});

process.on('unhandledRejection', (reason, promise) => {
    console.error('❌ Manager unhandled rejection:', reason);
});
```

</details>

</div>

## 🌐 Hosting Platforms

### ☁️ Cloud Platforms
| Platform | Status | Notes |
|----------|---------|-------|
| **Heroku** | ✅ Supported | Use worker dyno |
| **Railway** | ✅ Supported | Auto-deploy from GitHub |
| **Render** | ✅ Supported | Free tier available |
| **Replit** | ✅ Supported | Perfect for development |
| **Koyeb** | ✅ Supported | European hosting |
| **Digital Ocean** | ✅ Supported | Use App Platform |

### 🖥️ VPS/Server
| Platform | Status | Notes |
|----------|---------|-------|
| **Ubuntu/Debian** | ✅ Supported | Install Node.js 18+ |
| **CentOS/RHEL** | ✅ Supported | Use NodeJS repository |
| **Windows Server** | ✅ Supported | Install Node.js & Git |
| **Docker** | ✅ Supported | Use official Node image |

## 📋 Setup Instructions

### 1. Using Auto-Manager (Recommended)
1. **Create new project** on your hosting platform
2. **Copy the auto-manager code** from the button above
3. **Create `index.js`** and paste the copied code
4. **Set start command**: `node index.js`
5. **Deploy** and watch it auto-install everything!

### 2. Direct Clone Method
```bash
# Clone the repository
git clone https://github.com/marhthing/MATDEV-BOT.git
cd MATDEV-BOT

# Install dependencies
npm install

# Start the bot
node bot.js
```

## 📱 Getting Started

1. **Deploy your bot** using any method above
2. **Open your hosting platform's console/logs**
3. **Look for QR code** in the console output
4. **Scan the QR code** with your WhatsApp
5. **Send `.ping`** to test if it's working

## ⚡ Features

- 🚀 **Auto-Deploy**: Clone and install everything automatically
- 🔄 **Auto-Update**: Update bot with `.update` command
- 🛡️ **Anti-Ban**: Smart rate limiting and security features
- 📦 **Plugin System**: Modular commands and features
- 💾 **Session Persistence**: Maintains WhatsApp connection
- 🔒 **Security**: Owner-only commands and permission system

## 🎮 Basic Commands

| Command | Description |
|---------|-------------|
| `.ping` | Test bot response |
| `.help` | Show all commands |
| `.status` | Show bot statistics |
| `.update` | Check for updates |
| `.restart` | Restart the bot |

## 🤖 Auto-Update System

Your bot includes an intelligent auto-update system:

- **`.update`** - Check for updates from GitHub
- **`.updatenow`** - Force update immediately  
- **Session Preservation** - Keeps your WhatsApp session during updates
- **Automatic Recovery** - Restarts after updates complete

## 🔧 Troubleshooting

### Bot Won't Start
- Check if Node.js 18+ is installed
- Verify all dependencies installed with `npm install`
- Check console logs for error messages

### QR Code Not Showing
- Refresh your hosting platform's console
- Wait 30 seconds after deployment
- Check if bot process is running

### Commands Not Working
- Make sure you're the bot owner (scan QR with your WhatsApp)
- Check if prefix is correct (default is `.`)
- Verify bot is responding with `.ping`

## 💡 Tips

- **Fork First**: Always fork this repository before deploying
- **Update Regularly**: Use `.update` to get latest features
- **Monitor Logs**: Keep an eye on console output
- **Backup Sessions**: Your session folder contains WhatsApp credentials

---

<div align="center">

**🚀 Ready to deploy? Fork this repository and start hosting!**

[Fork Now](https://github.com/marhthing/MATDEV-BOT/fork) • [Report Issues](https://github.com/marhthing/MATDEV-BOT/issues) • [Get Support](https://github.com/marhthing/MATDEV-BOT/discussions)

*Made with ❤️ for the community*

</div>