
# voice_assistant.py
# A small, readable Python voice assistant.
# Dependencies: speechrecognition, pyaudio (or sounddevice+vosk alt), pyttsx3
# Optional: wikipedia
#
# How to run (Windows/macOS/Linux):
#   1) python -m venv .venv && .venv/bin/pip install -U pip
#   2) .venv/bin/pip install -r requirements.txt   (Windows: .venv\Scripts\pip install -r requirements.txt)
#   3) python voice_assistant.py
#
# Tips if microphone fails:
#   - On Windows, install PyAudio wheel from: https://www.lfd.uci.edu/~gohlke/pythonlibs/#pyaudio (pick your Python version)
#   - Or change INPUT_MODE below to "text" to type commands instead of speaking.

import os
import sys
import time
import webbrowser
from datetime import datetime
from pathlib import Path

# --- Settings you can tweak ---
ASSISTANT_NAME = "Anish"            # change the name if you like
INPUT_MODE = None               # "voice" or "text"
WAKE_WORDS = {"hey Anish", "ok Anish", "hello Anish"}
NOTES_FILE = Path("notes.txt")
LISTEN_TIMEOUT = 5                 # seconds to wait for you to start speaking
PHRASE_LIMIT = 8                   # max seconds per utterance

# --- Text to speech (pyttsx3) ---
try:
    import pyttsx3
    _engine = pyttsx3.init()
    _engine.setProperty("rate", 175)  # words per minute
    _engine.setProperty("volume", 1.0)
except Exception as e:
    _engine = None
    print(f"[warn] TTS init failed: {e}\n     (I'll still work, just without speaking)")

def say(text: str, also_print: bool = True):
    if also_print:
        print(f"{ASSISTANT_NAME}: {text}")
    if _engine:
        try:
            _engine.say(text)
            _engine.runAndWait()
        except Exception as e:
            print(f"[warn] TTS error: {e}")

# --- Speech to text (SpeechRecognition) ---
def listen() -> str | None:
    """Listen once and return lowercase transcript, or None on silence/error."""
    try:
        import speech_recognition as sr
    except Exception as e:
        print("[error] speech_recognition not installed. Switch INPUT_MODE='text' or install deps.")
        return None

    r = sr.Recognizer()
    r.dynamic_energy_threshold = True
    r.pause_threshold = 0.6

    with sr.Microphone() as source:
        # brief calibration to reduce background noise influence
        r.adjust_for_ambient_noise(source, duration=0.8)
        print("(listening...)")
        try:
            audio = r.listen(source, timeout=LISTEN_TIMEOUT, phrase_time_limit=PHRASE_LIMIT)
        except sr.WaitTimeoutError:
            print("(no speech detected)")
            return None

    # Try online Google recognizer first (works without API key for small usage)
    try:
        text = r.recognize_google(audio)
        return text.lower().strip()
    except Exception:
        # If that fails, try Sphinx (offline) if available
        try:
            text = r.recognize_sphinx(audio)
            return text.lower().strip()
        except Exception as e:
            print(f"(couldn't understand you: {e})")
            return None

# --- Helpers ---
def now_time():
    return datetime.now().strftime("%I:%M %p")

def now_date():
    return datetime.now().strftime("%A, %d %B %Y")

def write_note(text: str):
    timestamp = datetime.now().strftime("[%Y-%m-%d %H:%M]")
    NOTES_FILE.write_text((NOTES_FILE.read_text() if NOTES_FILE.exists() else "") + f"{timestamp} {text}\n", encoding="utf-8")
    return str(NOTES_FILE.resolve())

def open_site(url: str):
    webbrowser.open(url, new=2, autoraise=True)

def google_search(query: str):
    open_site(f"https://www.google.com/search?q={query.replace(' ', '+')}")

def youtube_search(query: str):
    open_site(f"https://www.youtube.com/results?search_query={query.replace(' ', '+')}")

# Optional wikipedia lookup (guarded)
def wiki_summary(topic: str, sentences: int = 2) -> str | None:
    try:
        import wikipedia
        wikipedia.set_lang("en")
        return wikipedia.summary(topic, sentences=sentences)
    except Exception:
        return None

# --- Command handling ---
def handle(command: str) -> bool:
    """Return True to continue, False to exit."""
    c = " ".join(command.split())  # normalize spaces
    global INPUT_MODE
    if c in {"mode voice", "switch voice"}:
        INPUT_MODE = "voice"
        say("Switched to voice mode.")
        return True
    if c in {"mode text", "switch text"}:
        INPUT_MODE = "text"
        say("Switched to text mode.")
        return True


    # exit commands
    if any(x in c for x in ["quit", "exit", "stop", "goodbye", "bye"]):
        say("See you later!")
        return False

    # small talk
    if c in {"hi", "hello", "hey", f"hey {ASSISTANT_NAME.lower()}"}:
        say("Hello! How can I help?")
        return True

    # time/date
    if "time" in c:
        say(f"It's {now_time()}.")
        return True
    if any(x in c for x in ["date", "day today"]):
        say(f"Today is {now_date()}.")
        return True

    # open common sites
    if "open youtube" in c:
        say("Opening YouTube."); open_site("https://youtube.com"); return True
    if "open google" in c:
        say("Opening Google."); open_site("https://google.com"); return True
    if "open github" in c:
        say("Opening GitHub."); open_site("https://github.com"); return True

    # search
    if c.startswith("search for ") or c.startswith("google "):
        q = c.split(" ", 2)[-1] if c.startswith("google ") else c.removeprefix("search for ")
        say(f"Searching for {q}."); google_search(q); return True

    # youtube play/search
    if c.startswith("play ") or c.startswith("youtube "):
        q = c.split(" ", 1)[1]
        say(f"Looking for {q} on YouTube."); youtube_search(q); return True

    # notes
    if c.startswith("note ") or c.startswith("take note ") or c.startswith("remember "):
        text = c.split(" ", 1)[1]
        path = write_note(text)
        say(f"Noted. Saved to {path}.")
        return True

    # wikipedia (optional)
    if c.startswith("what is ") or c.startswith("who is ") or c.startswith("tell me about "):
        topic = c.replace("what is ", "").replace("who is ", "").replace("tell me about ", "").strip()
        summary = wiki_summary(topic)
        if summary:
            say(summary)
        else:
            say("Couldn't fetch a summary right now. Opening search for you.")
            google_search(topic)
        return True

    # fallback: open a search
    say("I'll open a quick search for that.")
    google_search(c)
    return True

# --- Main loop ---
def main():
    global INPUT_MODE
    # ask mode at startup
    while INPUT_MODE not in {"voice", "text"}:
        choice = input("Start in 'voice' or 'text' mode? ").strip().lower()
        if choice in {"voice", "text"}:
            INPUT_MODE = choice
        else:
            print("Please type 'voice' or 'text'.")
    
    say(f"{ASSISTANT_NAME} ready. Current mode: {INPUT_MODE}. Say a wake word or type 'exit' to quit.")
    while True:

        if INPUT_MODE == "text":
            user = input("You: ").strip().lower()
            if not user:
                continue
            if user in WAKE_WORDS:
                say("I'm listening.")
                continue
            cont = handle(user)
            if not cont:
                break
            continue

        # voice mode
        heard = listen()
        if not heard:
            continue

        # wake word logic
        if any(heard.startswith(w) for w in WAKE_WORDS):
            # strip wake word
            for w in WAKE_WORDS:
                if heard.startswith(w):
                    heard = heard[len(w):].strip(" ,.-")
                    break
            if not heard:
                say("I'm here.")
                continue
        elif WAKE_WORDS:
            # ignore random room noise (if no wake word, comment this block out)
            # remove this 'continue' if you want hands-free, no-wake-word mode.
            print("(no wake word detected)")
            continue

        print(f"You: {heard}")
        if not handle(heard):
            break

if __name__ == "__main__":
    try:
        main()
    except KeyboardInterrupt:
        print("\n[quit]")
