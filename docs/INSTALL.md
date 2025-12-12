# OBSEA3 - Quick Start Installation

## TL;DR - Get Running in 5 Minutes

```bash
git clone <repository>
cd obsea

# Optional: Check system requirements first
bash bin/check_requirements.sh

# Install everything automatically
bash bin/00_install_all.sh
```

That's it! The script will:
- Ask you a few configuration questions
- Request sudo when needed (for system setup)
- Install everything automatically
- Optionally start the system

## What Gets Installed

- ✅ System user and group (`obsea_admins`)
- ✅ Directory structure in `/opt/obsea`
- ✅ Python virtual environment with ROS2 dependencies
- ✅ ROS2 workspace built and ready
- ✅ Configuration files and secrets

## After Installation

**Start the system:**
```bash
bash bin/50_run_all.sh
```

**Access services:**
- API: http://localhost:8101/docs
- Web UI: http://localhost:8180
- Health check: `curl http://localhost:8101/health`

**Default login credentials:**
- Username: `test`
- Password: `test123`
- Role: `viewer` (read-only, cannot control ports)

**Create users (optional):**
```bash
sudo bash bin/03_create_users.sh
```

## Manual Installation

If you prefer step-by-step control, see [DEPLOYMENT_GUIDE.md](DEPLOYMENT_GUIDE.md)

## Troubleshooting

**Script fails with permission error?**
- Make sure you're not running as root
- The script will request sudo only when needed

**Configuration already exists?**
- The script will ask if you want to overwrite
- Existing configs are in `bin/.venv` and `bin/secrets`

**Build fails?**
- Check logs in `/opt/obsea/log/current/`
- Run quick tests: `bash bin/quick_test.sh`
- See full guide: [DEPLOYMENT_GUIDE.md](DEPLOYMENT_GUIDE.md)

## Need Help?

- 📖 Full documentation: [DEPLOYMENT_GUIDE.md](DEPLOYMENT_GUIDE.md)
- 🔧 Script reference: [bin/README.md](bin/README.md)
- 🤖 AI agent guide: [.github/ARCHITECTURE.md](.github/ARCHITECTURE.md)
