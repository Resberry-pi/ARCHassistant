# Installation Instructions

1. Clone repo:
   ```bash
   git clone https://github.com/your-username/arch-assistant.git
   cd arch-assistant
Create virtual environment:

bash
Copy code
python -m venv venv
source venv/bin/activate  # Linux/Mac
venv\Scripts\activate     # Windows
Install dependencies:

bash
Copy code
pip install -r requirements.txt
Add API keys:

LLAMA

GEMINI_API_KEY

OLLAMA

SPOTIPY_CLIENT_ID / SPOTIPY_CLIENT_SECRET

Save them in .env file.

yaml
Copy code

---

### `06_user_guide.md`
```markdown
# User Guide

## Starting Arch
```bash
python arch_assistant.py

 Real-time voice recognition
- 🗣 Human-like speech synthesis
- 🔀 API router: Sarvam + Gemini + DeepSeek
- 🎶 Spotify voice automation
- 📚 Wikipedia & knowledge integration
- ⚡ Async + multithreading for speed
- 🔌 Hardware-ready (STM32/ESP32 integration)
- 🛠 Easy plugin system for new skills
