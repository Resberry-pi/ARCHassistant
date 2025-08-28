import os
import time, random
import asyncio
from concurrent.futures import ThreadPoolExecutor
import subprocess
import urllib.parse

import speech_recognition as sr
import pyttsx3
import datetime
import requests
import numpy as np
import librosa
import wikipedia
from wikipedia.exceptions import DisambiguationError, PageError

import spotipy
from spotipy.oauth2 import SpotifyOAuth

# ==========================
# 🔑 API KEYS (as provided)
# ==========================
GEMINI_API_KEY = "AIzaSyBffj6wrcO40eZuplSfMJPZzAGYowrXK8U"
import google.generativeai as genai
genai.configure(api_key=GEMINI_API_KEY)
gemini_models = {
    "1.5-flash": genai.GenerativeModel("gemini-1.5-flash"),
    "2.0-flash": genai.GenerativeModel("gemini-2.0-flash"),
    "2.5-pro": genai.GenerativeModel("gemini-2.5-pro"),
}

GROQ_API_KEY = "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"
OLLAMA_MODEL  = "llama3"
WOLFRAM_APP_ID = "R7JTHXH5GY"
NEWS_API_KEY   = "f0752422c6bc4b0fb0792f470490109a"

SPOTIFY_CLIENT_ID     = "2406a9eb3af548aba2dd33d5d7129345"
SPOTIFY_CLIENT_SECRET = "2cc3b3ee329643dcaad5430445081e1a"
SPOTIFY_REDIRECT_URI  = "http://127.0.0.1:8888/callback"
SPOTIFY_SCOPE = "user-read-playback-state user-modify-playback-state user-read-currently-playing streaming"

sp = spotipy.Spotify(auth_manager=SpotifyOAuth(
    client_id=SPOTIFY_CLIENT_ID,
    client_secret=SPOTIFY_CLIENT_SECRET,
    redirect_uri=SPOTIFY_REDIRECT_URI,
    scope=SPOTIFY_SCOPE
))

# ==========================
# Voice Engine
# ==========================
engine = pyttsx3.init()
engine.setProperty("rate", 180)
engine.setProperty("volume", 1)

def speak(text: str):
    print(f"ARCH: {text}")
    for sentence in text.split(". "):
        if not sentence:
            continue
        engine.say(sentence)
        engine.runAndWait()
        time.sleep(random.uniform(0.05, 0.2))

# ==========================
# Microphone
# ==========================
MIC_INDEX = 0
def list_mics():
    mics = sr.Microphone.list_microphone_names()
    for i, mic in enumerate(mics):
        print(f"{i}: {mic}")
    return mics

# ==========================
# Listen + MFCC/LPCC
# ==========================
def listen_blocking():
    r = sr.Recognizer()
    with sr.Microphone(device_index=MIC_INDEX) as source:
        print(" Listening...")
        r.adjust_for_ambient_noise(source, duration=1.0)
        try:
            audio = r.listen(source, phrase_time_limit=6)
            query = r.recognize_google(audio, language="en-in")
            print(f"You: {query}")

            # Optional acoustic features (guarded)
            features = None
            try:
                y = np.frombuffer(audio.get_raw_data(), dtype=np.int16).astype(np.float32)
                sr_rate = 16000
                mfccs = librosa.feature.mfcc(y=y, sr=sr_rate, n_mfcc=13)
                lpccs = librosa.lpc(y, order=12)
                features = np.concatenate([np.mean(mfccs, axis=1), np.log(np.abs(lpccs))])
            except Exception:
                features = None

            return query.lower().strip(), features
        except Exception:
            return "", None

# ==========================
# AI Queries (with safer fallbacks)
# ==========================
def ask_gemini(prompt, model_key):
    try:
        out = gemini_models[model_key].generate_content(prompt).text or ""
        return out.strip()
    except Exception as e:
        raise RuntimeError(f"Gemini {model_key} failed: {e}")

def ask_groq_llama(prompt):
    try:
        url = "https://api.groq.com/openai/v1/chat/completions"
        headers = {"Authorization": f"Bearer {GROQ_API_KEY}", "Content-Type": "application/json"}
        data = {
            "model": "llama3-70b-8192",
            "messages":[{"role":"user","content":prompt}],
            "max_tokens": 500
        }
        r = requests.post(url, headers=headers, json=data, timeout=12)
        r.raise_for_status()
        return r.json()["choices"][0]["message"]["content"].strip()
    except Exception as e:
        raise RuntimeError(f"Groq LLaMA-3 failed: {e}")

def ask_ollama(prompt):
    try:
        result = subprocess.run(["ollama", "run", OLLAMA_MODEL],
                                input=prompt.encode("utf-8"),
                                stdout=subprocess.PIPE,
                                stderr=subprocess.PIPE,
                                timeout=20)
        txt = result.stdout.decode("utf-8").strip()
        if not txt:
            raise RuntimeError(result.stderr.decode("utf-8"))
        return txt
    except Exception as e:
        raise RuntimeError(f"Ollama Local failed: {e}")

def ask_arch(prompt):
    for model_key in ["2.0-flash", "1.5-flash", "2.5-pro"]:
        try:
            return f"[Gemini {model_key}] " + ask_gemini(prompt, model_key)
        except Exception:
            pass
    try:
        return "[Groq LLaMA-3] " + ask_groq_llama(prompt)
    except Exception:
        pass
    try:
        return "[Ollama Local] " + ask_ollama(prompt)
    except Exception:
        pass
    return " All AI backends failed."

# ==========================
# FIXED Wikipedia
# ==========================
def wiki_search(query: str) -> str:
    try:
        q = query
        for lead in ["wikipedia", "who is", "what is", "tell me about", "define"]:
            q = q.replace(lead, "").strip()
        if not q:
            return " Please say a topic to search on Wikipedia."
        return wikipedia.summary(q, sentences=2, auto_suggest=True, redirect=True)
    except DisambiguationError as e:
        options = ", ".join(e.options[:5])
        return f" That’s ambiguous. Try one of: {options}"
    except PageError:
        return " No Wikipedia page found for that."
    except Exception as e:
        return f" Wikipedia error: {str(e)}"

# ==========================
# FIXED Wolfram Alpha
# ==========================
def wolfram_query(query: str) -> str:
    try:
        q = query
        for lead in ["calculate", "solve", "math", "integrate", "derive"]:
            q = q.replace(lead, "").strip()
        if not q:
            q = query
        url = f"http://api.wolframalpha.com/v1/result?i={urllib.parse.quote(q)}&appid={WOLFRAM_APP_ID}"
        r = requests.get(url, timeout=10)
        if r.status_code == 200 and r.text:
            return f"{r.text.strip()}"
        return " Wolfram could not solve that."
    except Exception as e:
        return f" Wolfram error: {str(e)}"

# ==========================
# FIXED News API
# ==========================
def fetch_news(topic: str = "top"):
    try:
        url = "https://newsapi.org/v2/top-headlines"
        params = {"apiKey": NEWS_API_KEY, "language": "en", "pageSize": 5}
        topic = topic.strip().lower()
        if topic in {"tech","technology"}:
            params.update({"country":"in", "category":"technology"})
        elif topic in {"business","finance"}:
            params.update({"country":"in", "category":"business"})
        elif topic in {"sports"}:
            params.update({"country":"in", "category":"sports"})
        else:
            params.update({"country":"in"})
        r = requests.get(url, params=params, timeout=10)
        data = r.json()
        if data.get("status") != "ok":
            return f" News API error: {data.get('message','Unknown error')}"
        articles = data.get("articles", [])[:3]
        if not articles:
            return " No fresh headlines right now."
        headlines = [f"{i+1}. {a.get('title','(no title)')}" for i, a in enumerate(articles)]
        return " " + " | ".join(headlines)
    except Exception as e:
        return f" News fetch error: {str(e)}"

# ==========================
# Spotify Control (unchanged)
# ==========================
def _list_devices():
    try:
        d = sp.devices()
        return d.get("devices", [])
    except Exception:
        return []

def _pick_device():
    devices = _list_devices()
    if not devices:
        return None
    active = [d for d in devices if d.get("is_active")]
    if active:
        return active[0]["id"]
    return devices[0]["id"]

def spotify_play(song_name: str):
    if not song_name:
        return " Tell me a song name."
    try:
        results = sp.search(q=song_name, type="track", limit=1)
        items = results.get("tracks", {}).get("items", [])
        if not items:
            return " Song not found."
        track = items[0]
        device_id = _pick_device()
        if not device_id:
            return " No Spotify device available. Open Spotify on any device once."
        sp.start_playback(device_id=device_id, uris=[track["uri"]])
        name = track["name"]
        artist = track["artists"][0]["name"] if track.get("artists") else "Unknown"
        return f"Playing {name} by {artist}"
    except Exception as e:
        return f" Could not start playback: {str(e)[:140]}"

def spotify_pause():
    try:
        device_id = _pick_device()
        if not device_id:
            return " No Spotify device available."
        sp.pause_playback(device_id=device_id)
        return " Paused Spotify"
    except Exception:
        return " Pause failed."

def spotify_resume():
    try:
        device_id = _pick_device()
        if not device_id:
            return " No Spotify device available."
        sp.start_playback(device_id=device_id)
        return " Resumed Spotify"
    except Exception:
        return " Resume failed."

def spotify_next():
    try:
        device_id = _pick_device()
        if not device_id:
            return "No Spotify device available."
        sp.next_track(device_id=device_id)
        return "Skipped to next track"
    except Exception:
        return "Skip failed."

def spotify_previous():
    try:
        device_id = _pick_device()
        if not device_id:
            return "No Spotify device available."
        sp.previous_track(device_id=device_id)
        return "Playing previous track"
    except Exception:
        return "Previous failed."

def spotify_current():
    try:
        pb = sp.current_playback()
        if pb and pb.get("item"):
            name = pb["item"].get("name", "Unknown")
            artists = pb["item"].get("artists", [])
            artist = artists[0]["name"] if artists else "Unknown"
            return f"Currently playing {name} by {artist}"
        return "Nothing is playing."
    except Exception:
        return "Could not read current playback."

# ==========================
# Intent helpers
# ==========================
def is_spotify_intent(cmd: str) -> bool:
    music_keywords = ["play ", "song", "track", "music", "on spotify", "playlist", "album", "artist"]
    return any(k in cmd for k in music_keywords)

def extract_song_from(cmd: str) -> str:
    if "play " in cmd:
        return cmd.split("play ", 1)[1].strip()
    return cmd

# ==========================
# Command Processor
# ==========================
def process_command(cmd):
    cmd_lower = cmd.lower()

    if "date" in cmd_lower:
        speak(datetime.date.today().strftime("Today is %B %d, %Y")); return
    if "time" in cmd_lower:
        speak(datetime.datetime.now().strftime("Current time is %I:%M %p")); return
    if "your name" in cmd_lower:
        speak("I am ARCH, your AI assistant!"); return

    if any(k in cmd_lower for k in ["wikipedia","who is","what is","tell me about","define"]):
        speak(wiki_search(cmd)); return

    if any(k in cmd_lower for k in ["calculate","solve","math","integrate","derive"]):
        speak(wolfram_query(cmd)); return

    if any(k in cmd_lower for k in ["news","headlines"]):
        topic = "top"
        for t in ["tech","technology","sports","business","finance"]:
            if t in cmd_lower:
                topic = t; break
        speak(fetch_news(topic)); return

    if is_spotify_intent(cmd_lower):
        if "pause" in cmd_lower or "stop" in cmd_lower:
            speak(spotify_pause()); return
        if "resume" in cmd_lower or "continue" in cmd_lower:
            speak(spotify_resume()); return
        if "next" in cmd_lower or "skip" in cmd_lower:
            speak(spotify_next()); return
        if "previous" in cmd_lower or "back" in cmd_lower:
            speak(spotify_previous()); return
        if any(k in cmd_lower for k in ["current song","what is playing","what's playing"]):
            speak(spotify_current()); return
        song = extract_song_from(cmd_lower)
        speak(spotify_play(song)); return

    if "exit" in cmd_lower or "quit" in cmd_lower:
        speak("Goodbye brother"); os._exit(0)

    speak(ask_arch(cmd))

# ==========================
# Async main
# ==========================
async def main():
    speak("Hello brother, I am ARCH, let’s roll!")
    list_mics()
    loop = asyncio.get_event_loop()
    executor = ThreadPoolExecutor(max_workers=3)
    try:
        while True:
            cmd_text, _features = await loop.run_in_executor(executor, listen_blocking)
            if cmd_text:
                loop.run_in_executor(executor, process_command, cmd_text)
    except (KeyboardInterrupt, asyncio.CancelledError):
        print("\nShutting down ARCH...")
    finally:
        executor.shutdown(wait=True)

if __name__ == "__main__":
    asyncio.run(main())

