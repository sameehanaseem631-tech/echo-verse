# echo-verse
Echoverse-audiobook
#EchoVerse prototype for Google Colab
# Run this whole cell in Google Colab.
# Requirements: internet (for gTTS). If you later integrate IBM Granite,
# replace rewrite_with_local() with rewrite_with_granite() or call both.

# Install required packages
!pip install -q gTTS pydub nltk

# For audio playback in Colab:
from IPython.display import Audio, display, HTML
from gtts import gTTS
import os, uuid, textwrap
import nltk
nltk.download('punkt', quiet=True)
# Download the specific resource needed for sentence tokenization
nltk.download('punkt_tab', quiet=True)


# ---------- Helper functions ----------
def save_mp3_from_text(tts_text, lang='en', filename=None):
    """Create an mp3 from text using gTTS and return filepath."""
    if filename is None:
        filename = f"/content/echoverse_{uuid.uuid4().hex[:8]}.mp3"
    tts = gTTS(tts_text, lang=lang, slow=False)
    tts.save(filename)
    return filename

def play_audio_file(filepath):
    """Display an audio player for an mp3 file in Colab."""
    display(Audio(filepath, autoplay=False))

def download_link_html(filepath, link_text="Download MP3"):
    fname = os.path.basename(filepath)
    return HTML(f'<a href="files/{fname}" download>{link_text}</a>')

# ---------- Simple prompt-chaining local rewrite engine ----------
from nltk.tokenize import sent_tokenize

def enhance_sentence_neutral(s):
    return " ".join(s.split())

def enhance_sentence_suspense(s):
    s = " ".join(s.split())
    if len(s.split()) > 10:
        s = s.rstrip('.!?') + '...'
    prefixes = ["Then,", "Suddenly,", "Without warning,"]
    if s and s[0].isalpha():
        s = f"{prefixes[abs(hash(s)) % len(prefixes)]} {s}"
    return s

def enhance_sentence_inspiring(s):
    s = " ".join(s.split())
    suffixes = ["—and you can too.", "—this matters.", "—keep going."]
    if len(s.split()) > 3:
        s = s.rstrip('.!?') + f", {suffixes[abs(hash(s)) % len(suffixes)]}"
    repl = {
        " help ": " empower ",
        " use ": " leverage ",
        " learn ": " master ",
        " improve ": " transform ",
        " change ": " elevate "
    }
    for k, v in repl.items():
        s = s.replace(k, v)
    return s

def rewrite_with_local(text, tone="Neutral"):
    sentences = sent_tokenize(text)
    out = []
    for s in sentences:
        if tone == "Neutral":
            out.append(enhance_sentence_neutral(s))
        elif tone == "Suspenseful":
            out.append(enhance_sentence_suspense(s))
        elif tone == "Inspiring":
            out.append(enhance_sentence_inspiring(s))
        else:
            out.append(enhance_sentence_neutral(s))
    first_pass = " ".join(out)
    final = first_pass.replace("  ", " ").strip()
    return final

# ---------- Placeholder for IBM Watsonx Granite integration ----------
def rewrite_with_granite(text, tone, api_key=None, endpoint=None):
    raise NotImplementedError("Integrate IBM Watsonx Granite here. For now the notebook uses rewrite_with_local().")

# ---------- User interface (works in Colab) ----------
def echoverse_run_interactive(use_granite=False, granite_api_key=None, granite_endpoint=None):
    print("=== EchoVerse (Colab prototype) ===")
    print("Options to provide text:")
    print("  1) Upload a text file")
    print("  2) Paste text directly\n")
    choice = input("Choose 1 (upload) or 2 (paste) [default 2]: ").strip() or "2"
    if choice == "1":
        from google.colab import files
        uploaded = files.upload()
        if not uploaded:
            print("No file uploaded. Switching to paste mode.")
            text = input("Paste your text below (end with Enter):\n")
        else:
            fname = list(uploaded.keys())[0]
            with open(fname, 'r', encoding='utf-8', errors='ignore') as f:
                text = f.read()
            print(f"Loaded {fname} ({len(text)} chars).")
    else:
        print("Paste your text. Press Enter when done.")
        text = input("Enter text:\n")

    if not text.strip():
        print("No text provided. Exiting.")
        return

    print("\nChoose a tone:")
    print("  1) Neutral")
    print("  2) Suspenseful")
    print("  3) Inspiring")
    tone_choice = input("Enter 1 / 2 / 3 [default 1]: ").strip() or "1"
    tone_map = {"1":"Neutral", "2":"Suspenseful", "3":"Inspiring"}
    tone = tone_map.get(tone_choice, "Neutral")
    print(f"Selected tone: {tone}\n")

    rewritten = None
    if use_granite:
        try:
            print("Attempting to use IBM Watsonx Granite for rewrite...")
            rewritten = rewrite_with_granite(text, tone, api_key=granite_api_key, endpoint=granite_endpoint)
        except Exception as e:
            print("Granite unavailable -> using local rewrite.")
            rewritten = rewrite_with_local(text, tone)
    else:
        rewritten = rewrite_with_local(text, tone)

    preview = rewritten[:800] + ("..." if len(rewritten) > 800 else "")
    print("\n=== Rewritten preview ===\n")
    print(textwrap.fill(preview, width=90))
    print("\n")

    synth_choice = input("Generate audio? (y/n) [default y]: ").strip().lower() or "y"
    if synth_choice != 'y':
        print("Done. Copy the rewritten text from the preview above.")
        return

    print("Generating MP3 with gTTS...")
    mp3_path = save_mp3_from_text(rewritten, lang='en')
    print(f"Saved audio to: {mp3_path}")
    play_audio_file(mp3_path)
    display(download_link_html(mp3_path, link_text="Download EchoVerse MP3"))

# ---------- Demo run (uncomment to run immediately) ----------
# echoverse_run_interactive(use_granite=False)

# ---------- Example usage for scripted run ----------
if _name_ == "_main_":
    sample = (
        "EchoVerse transforms any text into expressive, natural-sounding audio. "
        "Users can upload or paste content and choose a tone to enhance style and clarity. "
        "It is designed to be accessible and helpful for students, professionals, and visually-impaired users."
    )
    print("Running scripted demo (sample text) with 'Inspiring' tone...\n")
    rewritten_demo = rewrite_with_local(sample, tone="Inspiring")
    print("Rewritten text:\n", rewritten_demo, "\n")
    demo_mp3 = save_mp3_from_text(rewritten_demo)
    print("Playing demo audio below:")
    play_audio_file(demo_mp3)
    print("Demo MP3 saved to:", demo_mp3)
