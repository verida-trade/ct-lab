# 01 — CT Lab Desktop

**CT Lab Desktop** is the main application of the CT Lab ecosystem. Built on
Electron, it runs natively on macOS, Linux, and Windows and provides a complete
interface to chat with AI, visualize charts, and manage quantitative analysis
projects.

---

## System Requirements

| Operating System | Minimum Version | Notes |
|-------------------|-----------------|-------|
| macOS | 11 — Big Sur | Intel and Apple Silicon |
| Ubuntu | 20.04 (Focal Fossa) | Also compatible with derivative distributions |
| Windows | 10 | Windows 10 build 19041+ |

**Recommended resources:**

- 4 GB RAM (minimum 2 GB)
- 500 MB disk space
- Active internet connection for market-data queries

---

## Download

Visit the official website to download the installer:

> **[verida.trade/download](https://verida.trade)**

Select the version matching your system:

| Platform | File |
|----------|------|
| macOS | `CT-Lab-x.x.x.dmg` (macOS universal) |
| Linux | `CT-Lab-x.x.x.AppImage` |
| Windows | `CT-Lab-x.x.x.Setup.exe` |

> ⚠️ **Always download from the official website**. CT Lab does not distribute
> the application through third-party channels.

---

## Installation

### macOS

1. Open the downloaded `.dmg` file.
2. Drag the **CT Lab** icon into the **Applications** folder.
3. Open Launchpad and click **CT Lab**.

> On first launch, macOS may show a security warning. Go to
> **System Preferences → Security & Privacy → Open Anyway** to authorize.

### Ubuntu / Linux

```bash
# Make the AppImage executable
chmod +x CT-Lab-x.x.x.AppImage

# Run
./CT-Lab-x.x.x.AppImage
```

> If Ubuntu blocks the AppImage for security reasons, install FUSE:
> `sudo apt install libfuse2`

### Windows

1. Double-click the `CT-Lab-x.x.x.Setup.exe` file.
2. The installer will request permission — click **Yes**.
3. Follow the wizard: choose your installation destination and click **Install**.
4. When done, click **Finish** — CT Lab will open automatically.

> If Windows Defender blocks the installation, click **More info → Run anyway**.
> The installer is signed and safe.

---

## First Boot

When you open CT Lab for the first time, you'll see the following screen:

```
┌───────────────────────────────────────────────────────────────────────────────┐
│                           CT LAB — Welcome!                                  │
│                                                                               │
│   Before we begin, let's set up a few things:                                 │
│                                                                               │
│   1. [●]  Configure AI provider                       [Configure →]          │
│   2. [○]  Connect MCP server to CT Lab               [Connect →]             │
│   3. [○]  Create your first project                    [Start →]              │
│                                                                               │
│   ───────────────────────────────────────────────────────────────────────    │
│   Skip wizard · Open Settings                                                 │
└───────────────────────────────────────────────────────────────────────────────┘
```

The welcome wizard guides you through three steps:

1. **Configure AI provider** — choose OpenAI, Anthropic, Google, or Ollama (see doc 02).
2. **Connect MCP** — link the server to the app (see doc 03).
3. **Create your first project** — run a practical analysis (see doc 04).

> 💡 You can skip the wizard and configure everything manually in **Settings**.

---

## Settings Overview

Access via **CT Lab → Settings** (or `Cmd/Ctrl + ,`):

| Section | Description |
|---------|-------------|
| **General** | Language, theme (light/dark), data directory |
| **AI Provider** | Choose and configure provider (OpenAI, Anthropic, Google, Ollama) |
| **Extensions** | Add and manage MCP servers |
| **Data Sources** | Connect exchanges and data providers (Binance, Yahoo) |
| **License** | Check license status, purchase Premium plan |
| **Advanced** | Logs, series cache, diagnostics |

---

## Data Directory

CT Lab stores configurations and series cache in a local directory:

| System | Path |
|--------|------|
| macOS | `~/Library/Application Support/ct-lab` |
| Linux | `~/.config/ct-lab` |
| Windows | `%APPDATA%\ct-lab` |

> ⚠️ Do not manually edit files in this directory. Use the CT Lab interface for
> all configuration changes.

---

## License

On first launch, CT Lab runs in **Free** mode:

- ✅ 1 time series in cache
- ✅ 36 public technical indicators
- ✅ Backtesting

To unlock premium features (100 series, CT indicators, ML, microstructure),
check the **License** section in settings — visit [verida.trade](https://verida.trade)
for details.

---

## Verification

After installation, verify the app is working:

1. Open CT Lab.
2. You should see the main dashboard with the chat disabled and the welcome wizard.
3. Go to **Settings → General** and confirm the version number displays correctly.

> If nothing appears or there are startup errors, check the logs at
> **Settings → Advanced → Open Logs**.

---

## Next Steps

- ➡️ **[02 — AI Provider](./02-provider-ia)** — Configure your AI
- ⬅️ **[Back to Index](./README)**

---

_Last updated: 2026-08-11_
