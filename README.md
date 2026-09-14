# spotifyminiexe
Açık kaynak Kodu
(open source code)
//////
import io
import os
import sys
import json
import random
import threading
import time
import tkinter as tk
from datetime import datetime, timedelta
from tkinter import font as tkfont, simpledialog, messagebox
from colorthief import ColorThief
from PIL import Image, ImageDraw, ImageFilter, ImageTk, ImageFont
from pypresence import Presence
import requests
import spotipy
from spotipy.oauth2 import SpotifyOAuth
import syncedlyrics
import re
import pyperclip
import pystray
import keyboard
import webbrowser

try:
    import winsound
    HAS_WINSOUND = True
except ImportError:
    HAS_WINSOUND = False

try:
    from plyer import notification
    HAS_PLYER = True
except ImportError:
    HAS_PLYER = False

# ============================================================
# ★★★ EXE/SCRIPT BASE DIR ★★★
# ============================================================
def get_base_dir():
    """Exe olarak çalışıyorsa exe'nin yanını, script ise script klasörünü döner."""
    if getattr(sys, 'frozen', False):
        return os.path.dirname(sys.executable)
    return os.path.dirname(os.path.abspath(__file__))


BASE_DIR = get_base_dir()

# ============================================================
# SABİTLER
# ============================================================
REDIRECT_URI = "http://127.0.0.1:8888/callback"

SETTINGS_FILE = os.path.join(BASE_DIR, "settings.json")
HISTORY_FILE = os.path.join(BASE_DIR, "history.json")
LYRICS_CACHE_FILE = os.path.join(BASE_DIR, "lyrics_cache.json")
THEME_FILE = os.path.join(BASE_DIR, "theme.json")
LOG_FILE = os.path.join(BASE_DIR, "error.log")
LISTEN_TIME_FILE = os.path.join(BASE_DIR, "listen_time.json")
FAVORITES_FILE = os.path.join(BASE_DIR, "favorites.json")
CACHE_FILE = os.path.join(BASE_DIR, ".cache")

# ============================================================
# DİL
# ============================================================
LANG = {
    "tr": {
        "playing": "Oynatılıyor", "paused": "Duraklatıldı",
        "now_playing": "Şimdi çalıyor", "liked": "Beğenildi",
        "copied": "Kopyalandı", "added_queue": "Sıraya eklendi",
        "next_track": "Sıradaki", "no_lyrics": "Söz bulunamadı :(",
        "sleep_timer": "Uyku Zamanlayıcısı",
        "playlist": "Çalma Listesi", "history": "Geçmiş",
        "devices": "Cihazlar", "stats": "İstatistikler",
        "search": "Ara", "queue": "Sıradakiler",
        "lyrics": "Şarkı Sözleri", "settings": "Ayarlar",
        "unsynced_warning": "⚠ senkron yok",
        "volume": "Ses Seviyesi",
        "volume_prompt": "0-100 arası bir değer gir:",
        "menu": "Menü",
        "favorites": "Favoriler",
        "shortcuts": "Kısayollar",
    },
    "en": {
        "playing": "Playing", "paused": "Paused",
        "now_playing": "Now playing", "liked": "Liked",
        "copied": "Copied", "added_queue": "Added to queue",
        "next_track": "Next", "no_lyrics": "Lyrics not found :(",
        "sleep_timer": "Sleep Timer", "playlist": "Playlist",
        "history": "History", "devices": "Devices", "stats": "Stats",
        "search": "Search", "queue": "Queue",
        "lyrics": "Lyrics", "settings": "Settings",
        "unsynced_warning": "⚠ no sync",
        "volume": "Volume",
        "volume_prompt": "Enter a value between 0-100:",
        "menu": "Menu",
        "favorites": "Favorites",
        "shortcuts": "Shortcuts",
    },
}


def t(key):
    lang = SETTINGS.get("language", "tr")
    return LANG.get(lang, LANG["tr"]).get(key, key)


# ============================================================
# FONT
# ============================================================
def pick_font():
    try:
        families = set(tkfont.families())
    except:
        families = set()
    for f in ["Segoe UI Emoji", "Apple Color Emoji", "Noto Color Emoji"]:
        if f in families:
            return f
    return "Segoe UI" if "Segoe UI" in families else "TkDefaultFont"


# ============================================================
# AYARLAR
# ============================================================
DEFAULT_SETTINGS = {
    "alpha": 0.90, "font_size": 9,
    "discord_rpc": True, "blur_bg": True,
    "accent_mode": "auto", "custom_accent": "#1db954",
    "accent_locked": False, "start_minimized": False,
    "always_on_top": True, "show_mood": True, "mood_color": True,
    "window_x": None, "window_y": None, "mini_mode": False,
    "notify_new_track": True, "lyrics_cache_enabled": True,
    "auto_scroll_lyrics": True, "language": "tr",
    "snap_to_edge": True, "beep_on_track": False,
    "sleep_timer_minutes": 30, "show_next_preview": True,
    "lyrics_font_size": 10,
    "hotkeys_enabled": True,
    "always_show_volume_pct": True,
    "show_bitrate": True,
    "show_shortcuts_btn": True,
    # ★ YENİ: Kullanıcı bilgileri ★
    "client_id": "",
    "client_secret": "",
    "discord_client_id": "",
    "setup_completed": False,
}


def log_error(msg):
    try:
        with open(LOG_FILE, "a", encoding="utf-8") as f:
            f.write(f"[{datetime.now().isoformat()}] {msg}\n")
    except:
        pass


def _coerce_setting(key, value):
    if key not in DEFAULT_SETTINGS:
        return value
    default = DEFAULT_SETTINGS[key]
    if default is None:
        return value
    try:
        if isinstance(default, bool):
            if isinstance(value, str):
                return value.lower() in ("true", "1", "yes", "on")
            return bool(value)
        if isinstance(default, int) and not isinstance(default, bool):
            return int(value)
        if isinstance(default, float):
            return float(value)
        if isinstance(default, str):
            return str(value)
    except (ValueError, TypeError):
        return default
    return value


def load_settings():
    if os.path.exists(SETTINGS_FILE):
        try:
            with open(SETTINGS_FILE, "r", encoding="utf-8") as f:
                data = json.load(f)
            merged = DEFAULT_SETTINGS.copy()
            for k, v in data.items():
                merged[k] = _coerce_setting(k, v)
            return merged
        except Exception as e:
            log_error(f"Settings load: {e}")
    return DEFAULT_SETTINGS.copy()


def save_settings(s):
    try:
        clean = {k: v for k, v in s.items() if k in DEFAULT_SETTINGS}
        with open(SETTINGS_FILE, "w", encoding="utf-8") as f:
            json.dump(clean, f, indent=2, ensure_ascii=False)
    except Exception as e:
        log_error(f"Settings save: {e}")


SETTINGS = load_settings()


# ============================================================
# ★★★ İLK KURULUM WIZARD ★★★
# ============================================================
def needs_setup():
    """Kurulum tamamlanmış mı?"""
    if SETTINGS.get("setup_completed"):
        return False
    # Spotify bilgileri zorunlu
    return not (SETTINGS.get("client_id") and SETTINGS.get("client_secret"))


def run_setup_wizard():
    """İlk açılışta Spotify + Discord bilgilerini sor."""
    wizard = tk.Tk()
    wizard.title("Spotify Mini Pro - İlk Kurulum")
    wizard.geometry("540x600")
    wizard.config(bg="#121212")
    wizard.attributes("-topmost", True)

    # Pencereyi ekranın ortasına koy
    wizard.update_idletasks()
    sw = wizard.winfo_screenwidth()
    sh = wizard.winfo_screenheight()
    x = (sw - 540) // 2
    y = (sh - 600) // 2
    wizard.geometry(f"540x600+{x}+{y}")

    # --- BAŞLIK ---
    tk.Label(
        wizard, text="Spotify Mini Pro",
        fg="#1db954", bg="#121212",
        font=("Segoe UI", 20, "bold"),
    ).pack(pady=(25, 4))

    tk.Label(
        wizard, text="İlk kurulum - yaklaşık 2 dakika sürer",
        fg="#888888", bg="#121212",
        font=("Segoe UI", 9, "italic"),
    ).pack(pady=(0, 15))

    # --- ADIM GÖSTERGESİ ---
    step_label = tk.Label(
        wizard, text="ADIM 1 / 2 - Spotify",
        fg="#ffd93d", bg="#121212",
        font=("Segoe UI", 10, "bold"),
    )
    step_label.pack(pady=(0, 10))

    # --- İÇERİK ALANI ---
    content = tk.Frame(wizard, bg="#121212")
    content.pack(fill=tk.BOTH, expand=True, padx=35)

    # --- ADIM 1: SPOTIFY ---
    spotify_frame = tk.Frame(content, bg="#121212")

    tk.Label(
        spotify_frame,
        text="Spotify Developer bilgilerini gir:",
        fg="white", bg="#121212",
        font=("Segoe UI", 11, "bold"),
        anchor="w",
    ).pack(fill=tk.X, pady=(0, 8))

    tk.Label(
        spotify_frame,
        text="1. developer.spotify.com/dashboard adresine git\n"
             "2. Create App butonuna bas\n"
             "3. Redirect URI:  http://127.0.0.1:8888/callback\n"
             "4. Client ID ve Client Secret'ı kopyala",
        fg="#888888", bg="#121212",
        font=("Segoe UI", 8),
        justify="left", anchor="w",
    ).pack(fill=tk.X, pady=(0, 14))

    tk.Label(spotify_frame, text="Client ID:", fg="#d3d3d3", bg="#121212",
             font=("Segoe UI", 9), anchor="w").pack(fill=tk.X, pady=(6, 2))
    sp_id_entry = tk.Entry(
        spotify_frame, bg="#282828", fg="white",
        insertbackground="white", bd=0,
        font=("Consolas", 9),
    )
    sp_id_entry.pack(fill=tk.X, ipady=8)

    tk.Label(spotify_frame, text="Client Secret:", fg="#d3d3d3", bg="#121212",
             font=("Segoe UI", 9), anchor="w").pack(fill=tk.X, pady=(12, 2))
    sp_secret_entry = tk.Entry(
        spotify_frame, bg="#282828", fg="white",
        insertbackground="white", bd=0, show="•",
        font=("Consolas", 9),
    )
    sp_secret_entry.pack(fill=tk.X, ipady=8)

    # --- ADIM 2: DISCORD ---
    discord_frame = tk.Frame(content, bg="#121212")

    tk.Label(
        discord_frame,
        text="Discord Rich Presence (opsiyonel):",
        fg="white", bg="#121212",
        font=("Segoe UI", 11, "bold"),
        anchor="w",
    ).pack(fill=tk.X, pady=(0, 8))

    tk.Label(
        discord_frame,
        text="1. discord.com/developers/applications adresine git\n"
             "2. New Application → bir isim ver\n"
             "3. General Information → Application ID'yi kopyala\n"
             "4. (Opsiyonel) Rich Presence → Art Assets'e 'spotify_logo',\n"
             "   'play', 'pause' isimli görseller yükle\n\n"
             "Boş bırakırsan Discord RPC kapalı kalır.",
        fg="#888888", bg="#121212",
        font=("Segoe UI", 8),
        justify="left", anchor="w",
    ).pack(fill=tk.X, pady=(0, 14))

    tk.Label(discord_frame, text="Discord Application ID:", fg="#d3d3d3",
             bg="#121212", font=("Segoe UI", 9), anchor="w").pack(fill=tk.X, pady=(6, 2))
    dc_id_entry = tk.Entry(
        discord_frame, bg="#282828", fg="white",
        insertbackground="white", bd=0,
        font=("Consolas", 9),
    )
    dc_id_entry.pack(fill=tk.X, ipady=8)

    # --- HATA MESAJI ---
    error_label = tk.Label(
        wizard, text="", fg="#ff5555", bg="#121212",
        font=("Segoe UI", 9, "bold"),
    )
    error_label.pack(pady=(10, 0))

    # --- BUTONLAR ---
    button_frame = tk.Frame(wizard, bg="#121212")
    button_frame.pack(pady=20, fill=tk.X, padx=35)

    state = {"step": 1}

    def show_step(n):
        state["step"] = n
        if n == 1:
            discord_frame.pack_forget()
            spotify_frame.pack(fill=tk.BOTH, expand=True)
            step_label.config(text="ADIM 1 / 2 - Spotify")
            back_btn.pack_forget()
            next_btn.config(text="İleri →")
            sp_id_entry.focus_set()
        else:
            spotify_frame.pack_forget()
            discord_frame.pack(fill=tk.BOTH, expand=True)
            step_label.config(text="ADIM 2 / 2 - Discord (opsiyonel)")
            back_btn.pack(side=tk.LEFT)
            next_btn.config(text="Kaydet & Başla")
            dc_id_entry.focus_set()

    def go_next():
        error_label.config(text="")

        if state["step"] == 1:
            cid = sp_id_entry.get().strip()
            csec = sp_secret_entry.get().strip()
            if not cid or not csec:
                error_label.config(text="⚠ Client ID ve Secret zorunlu!")
                return
            if len(cid) != 32:
                error_label.config(text=f"⚠ Client ID 32 karakter olmalı (şu an: {len(cid)})")
                return
            if len(csec) != 32:
                error_label.config(text=f"⚠ Client Secret 32 karakter olmalı (şu an: {len(csec)})")
                return
            show_step(2)

        else:
            did = dc_id_entry.get().strip()
            if did and not did.isdigit():
                error_label.config(text="⚠ Discord ID sadece rakam olmalı!")
                return

            SETTINGS["client_id"] = sp_id_entry.get().strip()
            SETTINGS["client_secret"] = sp_secret_entry.get().strip()
            SETTINGS["discord_client_id"] = did
            SETTINGS["setup_completed"] = True
            # Discord ID boşsa RPC'yi kapat
            if not did:
                SETTINGS["discord_rpc"] = False

            save_settings(SETTINGS)
            wizard.destroy()

    def go_back():
        error_label.config(text="")
        show_step(1)

    back_btn = tk.Button(
        button_frame, text="← Geri", command=go_back,
        bg="#282828", fg="white", bd=0,
        activebackground="#383838", activeforeground="white",
        font=("Segoe UI", 10), cursor="hand2",
        padx=20, pady=8,
    )

    next_btn = tk.Button(
        button_frame, text="İleri →", command=go_next,
        bg="#1db954", fg="white", bd=0,
        activebackground="#1ed760", activeforeground="white",
        font=("Segoe UI", 10, "bold"), cursor="hand2",
        padx=20, pady=8,
    )
    next_btn.pack(side=tk.RIGHT)

    # Alt bilgi
    tk.Label(
        wizard,
        text="Bilgiler settings.json dosyasına kaydedilir.\n"
             "Bu dosyayı silmediğin sürece tekrar sorulmaz.",
        fg="#555555", bg="#121212",
        font=("Segoe UI", 7, "italic"),
        justify="center",
    ).pack(side=tk.BOTTOM, pady=10)

    # Enter tuşu ile ilerleme
    sp_secret_entry.bind("<Return>", lambda e: go_next())
    dc_id_entry.bind("<Return>", lambda e: go_next())

    show_step(1)
    wizard.mainloop()


# ★★★ KURULUM KONTROLÜ ★★★
if needs_setup():
    run_setup_wizard()
    # Wizard bittikten sonra ayarları tekrar yükle
    SETTINGS = load_settings()

# ★★★ Şimdi client bilgilerini oku ★★★
CLIENT_ID = SETTINGS.get("client_id", "")
CLIENT_SECRET = SETTINGS.get("client_secret", "")
DISCORD_CLIENT_ID = SETTINGS.get("discord_client_id", "")


# ============================================================
# TEMA
# ============================================================
DEFAULT_THEME = {
    "lyrics_active_color": "#ffd93d",
    "lyrics_past_color": "#4a4a4a",
    "lyrics_future_color": "#808080",
    "lyrics_bg_active": "#2a2a2a",
}


def load_theme():
    if os.path.exists(THEME_FILE):
        try:
            with open(THEME_FILE, "r", encoding="utf-8") as f:
                return {**DEFAULT_THEME, **json.load(f)}
        except:
            return DEFAULT_THEME.copy()
    return DEFAULT_THEME.copy()


THEME = load_theme()


# ============================================================
# GEÇMİŞ
# ============================================================
def load_history():
    if os.path.exists(HISTORY_FILE):
        try:
            with open(HISTORY_FILE, "r", encoding="utf-8") as f:
                return json.load(f)
        except:
            return []
    return []


def save_history(h):
    try:
        with open(HISTORY_FILE, "w", encoding="utf-8") as f:
            json.dump(h[-500:], f, indent=2, ensure_ascii=False)
    except:
        pass


HISTORY = load_history()


def load_favorites():
    if os.path.exists(FAVORITES_FILE):
        try:
            with open(FAVORITES_FILE, "r", encoding="utf-8") as f:
                return json.load(f)
        except:
            return []
    return []


def save_favorites(f):
    try:
        with open(FAVORITES_FILE, "w", encoding="utf-8") as f:
            json.dump(f, f, indent=2, ensure_ascii=False)
    except:
        pass


FAVORITES = load_favorites()


def load_listen_time():
    if os.path.exists(LISTEN_TIME_FILE):
        try:
            with open(LISTEN_TIME_FILE, "r", encoding="utf-8") as f:
                return json.load(f)
        except:
            return {}
    return {}


def save_listen_time(d):
    try:
        with open(LISTEN_TIME_FILE, "w", encoding="utf-8") as f:
            json.dump(d, f, indent=2)
    except:
        pass


LISTEN_TIME = load_listen_time()


# ============================================================
# LYRICS CACHE
# ============================================================
def load_lyrics_cache():
    if os.path.exists(LYRICS_CACHE_FILE):
        try:
            with open(LYRICS_CACHE_FILE, "r", encoding="utf-8") as f:
                return json.load(f)
        except:
            return {}
    return {}


def save_lyrics_cache(c):
    try:
        with open(LYRICS_CACHE_FILE, "w", encoding="utf-8") as f:
            json.dump(c, f, ensure_ascii=False)
    except:
        pass


LYRICS_CACHE = load_lyrics_cache()


# ============================================================
# SPOTIFY AUTH
# ============================================================
scope = (
    "user-read-playback-state user-modify-playback-state"
    " user-read-currently-playing user-library-read user-library-modify"
    " user-read-recently-played playlist-modify-public playlist-modify-private"
    " playlist-read-private"
)

try:
    sp = spotipy.Spotify(
        auth_manager=SpotifyOAuth(
            client_id=CLIENT_ID, client_secret=CLIENT_SECRET,
            redirect_uri=REDIRECT_URI, scope=scope, cache_path=CACHE_FILE,
        )
    )
except Exception as e:
    print("Spotify auth hatası:", e)
    messagebox.showerror("Hata", f"Spotify'a bağlanılamadı:\n{e}")
    sys.exit(1)

# ============================================================
# DISCORD RPC
# ============================================================
rpc = None
rpc_lock = threading.Lock()


def init_discord_rpc():
    global rpc
    if not SETTINGS.get("discord_rpc", True):
        return
    if not DISCORD_CLIENT_ID:
        print("Discord Client ID ayarlanmamış, RPC kapalı.")
        return
    try:
        rpc = Presence(DISCORD_CLIENT_ID)
        rpc.connect()
        print("Discord RPC bağlandı.")
    except Exception as e:
        print("Discord RPC hatası:", e)
        rpc = None


threading.Thread(target=init_discord_rpc, daemon=True).start()

last_drp_track_id = None
last_drp_state = None


def update_discord_presence(title, artist, img_url, is_playing, progress_ms, duration_ms, track_id):
    global last_drp_track_id, last_drp_state, rpc
    if not rpc:
        return
    with rpc_lock:
        if track_id == last_drp_track_id and is_playing == last_drp_state:
            return
        last_drp_track_id = track_id
        last_drp_state = is_playing
        try:
            buttons = [{"label": "Spotify'da Aç",
                        "url": f"https://open.spotify.com/track/{track_id}"}]
            if is_playing:
                now = time.time()
                start_time = int(now - (progress_ms / 1000))
                end_time = int(start_time + (duration_ms / 1000))
                rpc.update(
                    details=title, state=f"Sanatçı: {artist}",
                    start=start_time, end=end_time,
                    large_image="spotify_logo",
                    large_text=f"{title} - {artist}",
                    small_image="play", small_text=t("playing"),
                    buttons=buttons,
                )
            else:
                rpc.update(
                    details=title, state=f"{artist} ({t('paused')})",
                    large_image="spotify_logo",
                    large_text=f"{title} - {artist}",
                    small_image="pause", small_text=t("paused"),
                    buttons=buttons,
                )
        except Exception as e:
            log_error(f"DRP: {e}")


# ============================================================
# MOOD
# ============================================================
def analyze_mood(title, artist):
    tt = (title + " " + artist).lower()
    sad_kw = ["sad", "hüzün", "yagmur", "yağmur", "yar", "ayrılık", "acı", "pain", "cry", "melancholy", "broken"]
    happy_kw = ["happy", "mutlu", "dance", "party", "sun", "yaz", "summer", "love", "aşk", "neşe", "joy"]
    energy_kw = ["remix", "trap", "bass", "rock", "metal", "hard", "club", "edm", "phonk", "rage"]
    chill_kw = ["chill", "lofi", "relax", "sakin", "calm", "slow", "sleep", "study", "focus", "dream"]
    if any(k in tt for k in sad_kw): return "Melankolik"
    if any(k in tt for k in happy_kw): return "Neşeli"
    if any(k in tt for k in energy_kw): return "Enerjik"
    if any(k in tt for k in chill_kw): return "Sakin"
    return "Nötr"


MOOD_COLORS = {"Melankolik": "#4a7cff", "Neşeli": "#ffd93d",
               "Enerjik": "#ff4757", "Sakin": "#5f9ea0", "Nötr": "#1db954"}
MOOD_ICONS = {"Melankolik": "♪", "Neşeli": "★", "Enerjik": "⚡",
              "Sakin": "☾", "Nötr": "♫"}


def get_mood_color(mood): return MOOD_COLORS.get(mood, "#1db954")
def get_mood_icon(mood): return MOOD_ICONS.get(mood, "♫")


def add_to_history(track_id, title, artist, img_url):
    global HISTORY
    if HISTORY and HISTORY[-1].get("track_id") == track_id:
        return
    HISTORY.append({
        "track_id": track_id, "title": title, "artist": artist,
        "img_url": img_url, "time": datetime.now().isoformat(),
    })
    save_history(HISTORY)


def add_to_favorites(track_id, title, artist):
    global FAVORITES
    entry = {"track_id": track_id, "title": title, "artist": artist,
             "time": datetime.now().isoformat()}
    if not any(f["track_id"] == track_id for f in FAVORITES):
        FAVORITES.append(entry)
        save_favorites(FAVORITES)


# ============================================================
# BİLDİRİM / BIP
# ============================================================
def notify(title, message):
    if not HAS_PLYER:
        return
    try:
        notification.notify(title=title, message=message, timeout=3, app_name="Spotify Mini")
    except:
        pass


def beep():
    if not SETTINGS.get("beep_on_track", False):
        return
    try:
        if HAS_WINSOUND:
            winsound.Beep(880, 80)
        else:
            print("\a", end="", flush=True)
    except:
        pass


# ============================================================
# GÖRSEL
# ============================================================
def make_blurred_bg(image_bytes, size=(360, 130), blur=40):
    img = Image.open(io.BytesIO(image_bytes)).convert("RGB")
    img = img.resize(size, Image.Resampling.LANCZOS)
    img = img.filter(ImageFilter.GaussianBlur(blur))
    dark = Image.new("RGB", size, (0, 0, 0))
    return Image.blend(img, dark, alpha=0.55)


def make_rounded_cover(image_bytes, size=65, radius=8):
    img = Image.open(io.BytesIO(image_bytes)).convert("RGBA")
    img = img.resize((size, size), Image.Resampling.LANCZOS)
    mask = Image.new("L", (size, size), 0)
    ImageDraw.Draw(mask).rounded_rectangle((0, 0, size, size), radius=radius, fill=255)
    img.putalpha(mask)
    return ImageTk.PhotoImage(img)


def image_to_avg_hex(image_obj):
    avg = image_obj.resize((1, 1)).getpixel((0, 0))
    return f"#{avg[0]:02x}{avg[1]:02x}{avg[2]:02x}"


def get_palette(image_bytes, color_count=3):
    try:
        ct = ColorThief(io.BytesIO(image_bytes))
        palette = ct.get_palette(color_count=color_count, quality=5)
        return [f"#{r:02x}{g:02x}{b:02x}" for r, g, b in palette]
    except:
        return ["#1db954", "#1db954", "#1db954"]


# ============================================================
# YUVARLAK PENCERE
# ============================================================
def apply_rounded_window(win, w, h, radius=14):
    try:
        if os.name == "nt":
            import ctypes
            win.update_idletasks()
            hwnd = ctypes.windll.user32.GetParent(win.winfo_id())
            if hwnd == 0:
                hwnd = win.winfo_id()
            try:
                DWMWA_WINDOW_CORNER_PREFERENCE = 33
                DWMWCP_ROUND = 2
                pref = ctypes.c_int(DWMWCP_ROUND)
                ctypes.windll.dwmapi.DwmSetWindowAttribute(
                    hwnd, DWMWA_WINDOW_CORNER_PREFERENCE,
                    ctypes.byref(pref), ctypes.sizeof(pref))
            except:
                pass
        else:
            try:
                win.wm_attributes("-transparentcolor", "#000001")
            except:
                pass
    except Exception as e:
        log_error(f"Rounded window: {e}")


# ============================================================
# LYRICS
# ============================================================
def parse_lrc(lrc_text):
    lines = []
    pattern = re.compile(r"\[(\d+):(\d+)\.(\d+)\](.*)")
    for line in lrc_text.splitlines():
        m = pattern.match(line.strip())
        if m:
            minutes, seconds, centis, text = m.groups()
            ms = int(minutes) * 60000 + int(seconds) * 1000 + int(centis) * 10
            if text.strip():
                lines.append({"ms": ms, "text": text.strip()})
    return sorted(lines, key=lambda l: l["ms"])


def fetch_lyrics(title, artist):
    key = f"{title}|{artist}"
    if key in LYRICS_CACHE:
        c = LYRICS_CACHE[key]
        if isinstance(c, dict):
            return c.get("lines", []), c.get("synced", False)
        return c, False
    result = []
    synced = False
    try:
        lrc = syncedlyrics.search(f"{title} {artist}", synced_only=True)
        if lrc:
            result = parse_lrc(lrc)
            synced = True
        else:
            plain = syncedlyrics.search(f"{title} {artist}")
            if plain:
                result = [{"ms": i * 4000, "text": l} for i, l in enumerate(plain.splitlines()) if l.strip()]
                synced = False
    except Exception as e:
        log_error(f"Lyrics fetch: {e}")
    LYRICS_CACHE[key] = {"lines": result, "synced": synced}
    if SETTINGS.get("lyrics_cache_enabled", True):
        threading.Thread(target=save_lyrics_cache, args=(LYRICS_CACHE,), daemon=True).start()
    return result, synced


# ============================================================
# ROOT
# ============================================================
root = tk.Tk()
root.title("Spotify Mini Pro")
root.overrideredirect(True)
root.attributes("-topmost", SETTINGS.get("always_on_top", True))
root.attributes("-alpha", SETTINGS.get("alpha", 0.90))
root.config(bg="#121212")

screen_width = root.winfo_screenwidth()
screen_height = root.winfo_screenheight()
x_pos = SETTINGS.get("window_x") or (screen_width - 380)
y_pos = SETTINGS.get("window_y") or 20
root.geometry(f"360x130+{x_pos}+{y_pos}")

accent_color = SETTINGS.get("custom_accent", "#1db954")
palette = [accent_color, accent_color, accent_color]
is_playing_global = False
current_track_id_global = None
current_progress_ms = 0
current_duration_ms = 1
current_img_url = ""
current_title = ""
current_artist = ""
current_album = ""
current_mood = "Nötr"
img_cache = None
current_bg_photo = None
last_notify_time = 0
last_recommendation_time = 0
current_cover_bytes = None
next_track_name = ""
next_track_artist = ""
show_remaining_time = False
sleep_timer_id = None
sleep_timer_end = None
lyrics_synced = True
is_mini = False
is_dragging_volume = False
is_dragging_progress = False
current_popularity = 0

# BLUR
bg_canvas = tk.Canvas(root, width=360, height=130, highlightthickness=0, bg="#121212")
bg_canvas.place(x=0, y=0, relwidth=1, relheight=1)
bg_image_id = bg_canvas.create_image(0, 0, anchor="nw")

# ANA FRAME
main_frame = tk.Frame(root, bg="#121212", highlightthickness=1,
                      highlightbackground=accent_color)
main_frame.place(x=1, y=1, width=358, height=128)

THEMED_WIDGETS = []


# ============================================================
# SÜRÜKLEME + HOVER
# ============================================================
last_x, last_y = 0, 0


def start_move(event):
    global last_x, last_y
    last_x, last_y = event.x, event.y


def do_move(event):
    x = root.winfo_x() + (event.x - last_x)
    y = root.winfo_y() + (event.y - last_y)
    if SETTINGS.get("snap_to_edge", True):
        if x < 10: x = 0
        elif x + 360 > screen_width - 10: x = screen_width - 360
        if y < 10: y = 0
        elif y + 130 > screen_height - 10: y = screen_height - 130
    root.geometry(f"+{x}+{y}")
    SETTINGS["window_x"] = x
    SETTINGS["window_y"] = y
    for w in [queue_win, lyrics_win, device_win, stats_win, history_win,
              search_win, playlist_win, favorites_win, shortcuts_win]:
        try:
            if w and w.winfo_exists():
                w.geometry(f"+{x}+{y + 132}")
        except:
            pass


def _is_pointer_inside_root():
    try:
        x, y = root.winfo_pointerxy()
        rx, ry = root.winfo_rootx(), root.winfo_rooty()
        rw, rh = root.winfo_width(), root.winfo_height()
        return rx <= x <= rx + rw and ry <= y <= ry + rh
    except:
        return False


def on_enter(e):
    try:
        root.attributes("-alpha", 1.0)
    except:
        pass


def on_leave(e):
    if _is_pointer_inside_root():
        return
    if any(w and w.winfo_exists() for w in
           [queue_win, lyrics_win, device_win, stats_win, history_win,
            search_win, playlist_win, favorites_win, shortcuts_win]):
        return
    try:
        root.attributes("-alpha", SETTINGS.get("alpha", 0.90))
    except:
        pass


def bind_hover_recursive(widget):
    try:
        widget.bind("<Enter>", on_enter, add="+")
        widget.bind("<Leave>", on_leave, add="+")
    except:
        pass
    try:
        for child in widget.winfo_children():
            bind_hover_recursive(child)
    except:
        pass


root.bind("<Escape>", lambda e: close_all_windows())


# ============================================================
# PENCERELER
# ============================================================
queue_win = None
queue_listbox = None
lyrics_win = None
lyrics_text = None
lyrics_lines = []
lyrics_synced_lbl = None
device_win = None
device_listbox = None
stats_win = None
stats_text = None
history_win = None
history_listbox = None
search_win = None
search_entry = None
search_results = None
recommendation_win = None
playlist_win = None
playlist_listbox = None
settings_win = None
sleep_win = None
favorites_win = None
favorites_listbox = None
shortcuts_win = None
menu_popup = None
DEVICE_CACHE = []
SEARCH_CACHE = []
PLAYLIST_CACHE = []


def close_all_windows():
    for name in ["queue_win", "lyrics_win", "device_win", "stats_win",
                 "history_win", "search_win", "recommendation_win",
                 "playlist_win", "settings_win", "sleep_win",
                 "favorites_win", "shortcuts_win", "menu_popup"]:
        w = globals().get(name)
        try:
            if w and w.winfo_exists():
                w.destroy()
        except:
            pass


# ============================================================
# KAPAT
# ============================================================
close_btn = tk.Button(
    main_frame, text="✕",
    command=lambda: (save_settings(SETTINGS), close_all_windows(), root.destroy()),
    bg="#121212", fg="#888888", bd=0,
    activebackground="#121212", activeforeground="#ff5555",
    font=("Segoe UI", 9, "bold"), cursor="hand2",
)
close_btn.place(x=338, y=4)
THEMED_WIDGETS.append(close_btn)


# ============================================================
# MENÜ POPUP
# ============================================================
def show_main_menu():
    global menu_popup
    if menu_popup and menu_popup.winfo_exists():
        menu_popup.destroy()
        menu_popup = None
        return

    menu_popup = tk.Toplevel(root)
    menu_popup.overrideredirect(True)
    menu_popup.attributes("-topmost", True)
    menu_popup.attributes("-alpha", 0.98)
    menu_popup.config(bg="#121212")

    mx = root.winfo_x() + 180
    my = root.winfo_y() + 30
    menu_popup.geometry(f"180x420+{mx}+{my}")
    apply_rounded_window(menu_popup, 180, 420, 14)

    mf = tk.Frame(menu_popup, bg="#181818", highlightthickness=1,
                  highlightbackground=accent_color)
    mf.pack(fill=tk.BOTH, expand=True)

    title_lbl = tk.Label(mf, text="  " + t("menu").upper(), fg="white", bg="#181818",
                         font=("Segoe UI", 8, "bold"), anchor="w")
    title_lbl.pack(fill=tk.X, padx=8, pady=(8, 4))

    def menu_item(icon, label, cmd):
        def on_click(event=None, _cmd=cmd):
            global menu_popup
            if menu_popup and menu_popup.winfo_exists():
                menu_popup.destroy()
                menu_popup = None
            root.after(50, _cmd)

        btn = tk.Button(mf, text=f"  {icon}  {label}",
                        command=on_click,
                        bg="#181818", fg="#d3d3d3", bd=0, anchor="w",
                        activebackground="#282828", activeforeground="white",
                        font=("Segoe UI", 9), cursor="hand2",
                        padx=8, pady=6)
        btn.pack(fill=tk.X, padx=4, pady=1)
        btn.bind("<Enter>", lambda e, b=btn: b.config(bg="#282828"))
        btn.bind("<Leave>", lambda e, b=btn: b.config(bg="#181818"))
        return btn

    menu_item("♪", t("lyrics"), toggle_lyrics)
    menu_item("☰", t("queue"), toggle_queue)
    menu_item("▤", t("stats"), toggle_stats)
    menu_item("◉", t("devices"), toggle_devices)
    menu_item("🕓", t("history"), toggle_history)
    menu_item("🔍", t("search"), toggle_search)
    menu_item("📁", t("playlist"), toggle_playlist_adder)
    menu_item("⭐", t("favorites"), toggle_favorites)
    menu_item("⌨", t("shortcuts"), toggle_shortcuts)
    menu_item("😴", t("sleep_timer"), toggle_sleep_timer)
    menu_item("⚙", t("settings"), toggle_settings)

    menu_popup.bind("<Escape>", lambda e: (menu_popup.destroy(),
                                            globals().__setitem__("menu_popup", None)))
    menu_popup.focus_set()

    def on_root_click(e):
        if menu_popup and menu_popup.winfo_exists():
            wx, wy = menu_popup.winfo_rootx(), menu_popup.winfo_rooty()
            ww, wh = menu_popup.winfo_width(), menu_popup.winfo_height()
            if not (wx <= e.x_root <= wx + ww and wy <= e.y_root <= wy + wh):
                menu_popup.destroy()
                globals().__setitem__("menu_popup", None)

    def enable_outside_click():
        try:
            root.bind("<Button-1>", on_root_click, add="+")
        except:
            pass

    root.after(200, enable_outside_click)


btn_menu = tk.Button(
    main_frame, text="⋮",
    command=show_main_menu,
    bg="#121212", fg="#888888", bd=0,
    activebackground="#121212", activeforeground=accent_color,
    font=("Segoe UI", 14, "bold"), cursor="hand2",
)
btn_menu.place(x=308, y=2, width=20, height=20)
THEMED_WIDGETS.append(btn_menu)


# ============================================================
# SIRADAKİLER
# ============================================================
def toggle_queue():
    global queue_win, queue_listbox
    if queue_win is None or not queue_win.winfo_exists():
        queue_win = tk.Toplevel(root)
        queue_win.overrideredirect(True)
        queue_win.attributes("-topmost", True)
        queue_win.attributes("-alpha", 0.95)
        queue_win.geometry(f"360x180+{root.winfo_x()}+{root.winfo_y() + 132}")
        apply_rounded_window(queue_win, 360, 180, 14)
        qf = tk.Frame(queue_win, bg="#181818", highlightthickness=1,
                      highlightbackground=accent_color)
        qf.pack(fill=tk.BOTH, expand=True)
        tk.Label(qf, text="  " + t("queue").upper(), fg="white", bg="#181818",
                 font=("Segoe UI", 8, "bold"), anchor="w").pack(fill=tk.X, padx=6, pady=(6, 2))
        queue_listbox = tk.Listbox(
            qf, bg="#121212", fg="#d3d3d3", selectbackground="#282828",
            selectforeground="white", font=("Segoe UI", 8), bd=0, highlightthickness=0)
        queue_listbox.pack(side=tk.LEFT, fill=tk.BOTH, expand=True, padx=6, pady=(0, 6))
        sc = tk.Scrollbar(qf, command=queue_listbox.yview)
        sc.pack(side=tk.RIGHT, fill=tk.Y, pady=(0, 6))
        queue_listbox.config(yscrollcommand=sc.set)
        queue_listbox.bind("<Double-Button-1>", lambda e: jump_to_queue_item())
        threading.Thread(target=fetch_queue, daemon=True).start()
        bind_hover_recursive(queue_win)
    else:
        queue_win.destroy()
        queue_win = None


def jump_to_queue_item():
    try:
        sel = queue_listbox.curselection()
        if not sel:
            return
        idx = sel[0]
        for _ in range(idx + 1):
            sp.next_track()
    except Exception as e:
        log_error(f"Queue jump: {e}")


def fetch_queue():
    try:
        q_data = sp.queue()
        items = q_data.get("queue", [])[:10]
        if queue_listbox and queue_listbox.winfo_exists():
            queue_listbox.delete(0, tk.END)
            if not items:
                queue_listbox.insert(tk.END, " Sıradaki şarkı yok.")
            for idx, track in enumerate(items, 1):
                t_name = track.get("name", "?")
                a_name = track["artists"][0]["name"] if track.get("artists") else ""
                queue_listbox.insert(tk.END, f" {idx}. {t_name} - {a_name}")
    except Exception as e:
        log_error(f"Queue fetch: {e}")


# ============================================================
# LYRICS
# ============================================================
def toggle_lyrics():
    global lyrics_win, lyrics_text, lyrics_synced_lbl
    if lyrics_win is None or not lyrics_win.winfo_exists():
        lyrics_win = tk.Toplevel(root)
        lyrics_win.overrideredirect(True)
        lyrics_win.attributes("-topmost", True)
        lyrics_win.attributes("-alpha", 0.95)
        lyrics_win.geometry(f"380x300+{root.winfo_x()}+{root.winfo_y() + 132}")
        apply_rounded_window(lyrics_win, 380, 300, 14)
        lf = tk.Frame(lyrics_win, bg="#181818", highlightthickness=1,
                      highlightbackground=accent_color)
        lf.pack(fill=tk.BOTH, expand=True)
        header = tk.Frame(lf, bg="#181818")
        header.pack(fill=tk.X, padx=6, pady=(6, 2))
        tk.Label(header, text="  " + t("lyrics").upper(), fg="white", bg="#181818",
                 font=("Segoe UI", 8, "bold"), anchor="w").pack(side=tk.LEFT)
        lyrics_synced_lbl = tk.Label(header, text="", fg="#ff9944", bg="#181818",
                                     font=("Segoe UI", 7, "italic"))
        lyrics_synced_lbl.pack(side=tk.LEFT, padx=6)
        tk.Button(header, text="Genius ↗", command=search_lyrics_on_genius,
                  bg="#181818", fg="#888888", bd=0, activebackground="#181818",
                  activeforeground=accent_color, font=("Segoe UI", 7),
                  cursor="hand2").pack(side=tk.RIGHT, padx=(0, 4))
        tk.Button(header, text="✕",
                  command=lambda: (lyrics_win.destroy(), globals().__setitem__("lyrics_win", None)),
                  bg="#181818", fg="#888888", bd=0, activebackground="#181818",
                  font=("Segoe UI", 8, "bold"), cursor="hand2").pack(side=tk.RIGHT)
        lyrics_text = tk.Text(
            lf, bg="#121212", fg="#b3b3b3",
            font=("Segoe UI", SETTINGS.get("lyrics_font_size", 11)),
            bd=0, highlightthickness=0, wrap=tk.WORD,
            padx=14, pady=8, spacing1=14, spacing3=14,
            cursor="arrow", insertwidth=0,
        )
        lyrics_text.pack(fill=tk.BOTH, expand=True, padx=6, pady=(0, 6))
        lyrics_text.tag_config("active",
                               foreground=THEME.get("lyrics_active_color", "#ffd93d"),
                               background=THEME.get("lyrics_bg_active", "#2a2a2a"),
                               font=("Segoe UI", SETTINGS.get("lyrics_font_size", 11) + 3, "bold"),
                               underline=True, justify="center")
        lyrics_text.tag_config("past", foreground=THEME.get("lyrics_past_color", "#4a4a4a"),
                               font=("Segoe UI", SETTINGS.get("lyrics_font_size", 11)),
                               justify="center")
        lyrics_text.tag_config("future", foreground=THEME.get("lyrics_future_color", "#a0a0a0"),
                               font=("Segoe UI", SETTINGS.get("lyrics_font_size", 11)),
                               justify="center")
        lyrics_text.config(state=tk.DISABLED)

        def _on_wheel(event):
            lyrics_text.yview_scroll(int(-1 * (event.delta / 120)), "units")
        lyrics_text.bind("<MouseWheel>", _on_wheel)
        threading.Thread(target=load_lyrics_async, daemon=True).start()
        bind_hover_recursive(lyrics_win)
    else:
        lyrics_win.destroy()
        lyrics_win = None


def search_lyrics_on_genius():
    if current_title:
        q = f"{current_title} {current_artist}".replace(" ", "%20")
        webbrowser.open(f"https://genius.com/search?q={q}")


def load_lyrics_async():
    global lyrics_lines, lyrics_synced
    if not current_title:
        return
    lines, synced = fetch_lyrics(current_title, current_artist)
    lyrics_lines = lines
    lyrics_synced = synced

    def update_ui():
        if not (lyrics_text and lyrics_text.winfo_exists()):
            return
        if lyrics_synced_lbl and lyrics_synced_lbl.winfo_exists():
            lyrics_synced_lbl.config(text="" if synced else t("unsynced_warning"))
        lyrics_text.config(state=tk.NORMAL)
        lyrics_text.delete("1.0", tk.END)
        if not lines:
            lyrics_text.insert(tk.END, "\n\n\n" + t("no_lyrics"), ("future",))
        else:
            lyrics_text.insert(tk.END, "\n\n\n")
            for i, line in enumerate(lines):
                lyrics_text.insert(tk.END, line["text"] + "\n", (f"l{i}", "future"))
            lyrics_text.insert(tk.END, "\n\n\n")
        lyrics_text.config(state=tk.DISABLED)
        highlight_lyrics(current_progress_ms, force=True)

    root.after(0, update_ui)


def highlight_lyrics(progress_ms, force=False):
    if not lyrics_text or not lyrics_text.winfo_exists() or not lyrics_lines:
        return
    active_idx = 0
    for i, line in enumerate(lyrics_lines):
        if progress_ms >= line["ms"]:
            active_idx = i
        else:
            break
    try:
        for tag in ("active", "past", "future"):
            lyrics_text.tag_remove(tag, "1.0", tk.END)
        for i in range(len(lyrics_lines)):
            lyrics_text.tag_add("future", f"l{i}.0", f"l{i}.end")
        for i in range(active_idx):
            lyrics_text.tag_remove("future", f"l{i}.0", f"l{i}.end")
            lyrics_text.tag_add("past", f"l{i}.0", f"l{i}.end")
        lyrics_text.tag_remove("future", f"l{active_idx}.0", f"l{active_idx}.end")
        lyrics_text.tag_add("active", f"l{active_idx}.0", f"l{active_idx}.end")
        if SETTINGS.get("auto_scroll_lyrics", True):
            try:
                lyrics_text.see(f"l{active_idx}.0")
                lyrics_text.yview_scroll(-2, "units")
            except:
                pass
    except Exception as e:
        log_error(f"Lyrics highlight: {e}")


# ============================================================
# CİHAZLAR
# ============================================================
def toggle_devices():
    global device_win, device_listbox
    if device_win is None or not device_win.winfo_exists():
        device_win = tk.Toplevel(root)
        device_win.overrideredirect(True)
        device_win.attributes("-topmost", True)
        device_win.attributes("-alpha", 0.95)
        device_win.geometry(f"360x140+{root.winfo_x()}+{root.winfo_y() + 132}")
        apply_rounded_window(device_win, 360, 140, 14)
        df = tk.Frame(device_win, bg="#181818", highlightthickness=1,
                      highlightbackground=accent_color)
        df.pack(fill=tk.BOTH, expand=True)
        tk.Label(df, text="  " + t("devices").upper(), fg="white", bg="#181818",
                 font=("Segoe UI", 8, "bold"), anchor="w").pack(fill=tk.X, padx=6, pady=(6, 2))
        device_listbox = tk.Listbox(
            df, bg="#121212", fg="#d3d3d3", selectbackground="#282828",
            selectforeground="white", font=("Segoe UI", 8), bd=0, highlightthickness=0)
        device_listbox.pack(fill=tk.BOTH, expand=True, padx=6, pady=(0, 6))
        device_listbox.bind("<Double-Button-1>", lambda e: select_device())
        threading.Thread(target=fetch_devices, daemon=True).start()
        bind_hover_recursive(device_win)
    else:
        device_win.destroy()
        device_win = None


def fetch_devices():
    global DEVICE_CACHE
    try:
        data = sp.devices()
        DEVICE_CACHE = data.get("devices", [])
        if device_listbox and device_listbox.winfo_exists():
            device_listbox.delete(0, tk.END)
            if not DEVICE_CACHE:
                device_listbox.insert(tk.END, " Cihaz bulunamadı.")
            for d in DEVICE_CACHE:
                mark = "▶ " if d.get("is_active") else "  "
                device_listbox.insert(tk.END, f"{mark}{d['name']} ({d['type']})")
    except Exception as e:
        log_error(f"Device fetch: {e}")


def select_device():
    try:
        sel = device_listbox.curselection()
        if not sel:
            return
        d = DEVICE_CACHE[sel[0]]
        sp.transfer_playback(d["id"], force_play=True)
        notify("Cihaz", d["name"])
    except Exception as e:
        log_error(f"Transfer: {e}")


# ============================================================
# İSTATİSTİKLER
# ============================================================
def toggle_stats():
    global stats_win, stats_text
    if stats_win is None or not stats_win.winfo_exists():
        stats_win = tk.Toplevel(root)
        stats_win.overrideredirect(True)
        stats_win.attributes("-topmost", True)
        stats_win.attributes("-alpha", 0.95)
        stats_win.geometry(f"360x300+{root.winfo_x()}+{root.winfo_y() + 132}")
        apply_rounded_window(stats_win, 360, 300, 14)
        sf = tk.Frame(stats_win, bg="#181818", highlightthickness=1,
                      highlightbackground=accent_color)
        sf.pack(fill=tk.BOTH, expand=True)
        header = tk.Frame(sf, bg="#181818")
        header.pack(fill=tk.X, padx=6, pady=(6, 2))
        tk.Label(header, text="  " + t("stats").upper(), fg="white", bg="#181818",
                 font=("Segoe UI", 8, "bold"), anchor="w").pack(side=tk.LEFT)
        tk.Button(header, text="✕",
                  command=lambda: (stats_win.destroy(), globals().__setitem__("stats_win", None)),
                  bg="#181818", fg="#888888", bd=0, activebackground="#181818",
                  font=("Segoe UI", 8, "bold"), cursor="hand2").pack(side=tk.RIGHT)
        stats_text = tk.Text(sf, bg="#121212", fg="#d3d3d3",
                             font=("Consolas", 8), bd=0, highlightthickness=0,
                             wrap=tk.WORD, padx=8, pady=4)
        stats_text.pack(fill=tk.BOTH, expand=True, padx=6, pady=(0, 6))
        refresh_stats()
        bind_hover_recursive(stats_win)
    else:
        stats_win.destroy()
        stats_win = None


def format_duration(seconds):
    h = seconds // 3600
    m = (seconds % 3600) // 60
    s = seconds % 60
    if h > 0: return f"{h}sa {m}dk"
    if m > 0: return f"{m}dk {s}sn"
    return f"{s}sn"


def refresh_stats():
    if not stats_text or not stats_text.winfo_exists():
        return
    try:
        stats_text.config(state=tk.NORMAL)
        stats_text.delete("1.0", tk.END)
        today = datetime.now().strftime("%Y-%m-%d")
        week_ago = (datetime.now() - timedelta(days=7)).strftime("%Y-%m-%d")
        today_time = LISTEN_TIME.get(today, {}).get("total", 0)
        week_total = sum(v.get("total", 0) for k, v in LISTEN_TIME.items() if k >= week_ago)
        all_total = sum(v.get("total", 0) for v in LISTEN_TIME.values())
        stats_text.insert(tk.END, "⏱ Dinleme Süresi\n", ("h",))
        stats_text.insert(tk.END, f"  Bugün:   {format_duration(today_time)}\n")
        stats_text.insert(tk.END, f"  Bu hafta: {format_duration(week_total)}\n")
        stats_text.insert(tk.END, f"  Toplam:   {format_duration(all_total)}\n\n")
        if not HISTORY:
            stats_text.insert(tk.END, "Henüz geçmiş yok.")
            stats_text.tag_config("h", foreground=accent_color, font=("Segoe UI", 8, "bold"))
            stats_text.config(state=tk.DISABLED)
            return
        artist_count, track_count, hour_count = {}, {}, {}
        for h in HISTORY:
            artist_count[h["artist"]] = artist_count.get(h["artist"], 0) + 1
            track_count[h["title"]] = track_count.get(h["title"], 0) + 1
            try:
                hour = datetime.fromisoformat(h["time"]).hour
                hour_count[hour] = hour_count.get(hour, 0) + 1
            except:
                pass
        top_artists = sorted(artist_count.items(), key=lambda x: -x[1])[:5]
        top_tracks = sorted(track_count.items(), key=lambda x: -x[1])[:5]
        peak_hour = max(hour_count.items(), key=lambda x: x[1])[0] if hour_count else "-"
        stats_text.insert(tk.END, f"🎧 Toplam dinlenen: {len(HISTORY)} şarkı\n")
        stats_text.insert(tk.END, f"⏰ En aktif saat: {peak_hour}:00\n\n")
        stats_text.insert(tk.END, "🎤 Top sanatçılar:\n", ("h",))
        for i, (a, c) in enumerate(top_artists, 1):
            stats_text.insert(tk.END, f"  {i}. {a}  ({c}x)\n")
        stats_text.insert(tk.END, "\n🎵 Top şarkılar:\n", ("h",))
        for i, (tt, c) in enumerate(top_tracks, 1):
            stats_text.insert(tk.END, f"  {i}. {tt[:30]}  ({c}x)\n")
        stats_text.tag_config("h", foreground=accent_color, font=("Segoe UI", 8, "bold"))
        stats_text.config(state=tk.DISABLED)
    except Exception as e:
        log_error(f"Stats: {e}")


# ============================================================
# GEÇMİŞ
# ============================================================
def toggle_history():
    global history_win, history_listbox
    if history_win is None or not history_win.winfo_exists():
        history_win = tk.Toplevel(root)
        history_win.overrideredirect(True)
        history_win.attributes("-topmost", True)
        history_win.attributes("-alpha", 0.95)
        history_win.geometry(f"360x250+{root.winfo_x()}+{root.winfo_y() + 132}")
        apply_rounded_window(history_win, 360, 250, 14)
        hf = tk.Frame(history_win, bg="#181818", highlightthickness=1,
                      highlightbackground=accent_color)
        hf.pack(fill=tk.BOTH, expand=True)
        header = tk.Frame(hf, bg="#181818")
        header.pack(fill=tk.X, padx=6, pady=(6, 2))
        tk.Label(header, text="  " + t("history").upper(), fg="white", bg="#181818",
                 font=("Segoe UI", 8, "bold"), anchor="w").pack(side=tk.LEFT)
        tk.Button(header, text="+ Playlist", command=create_playlist_from_history,
                  bg="#181818", fg="#888888", bd=0, activebackground="#181818",
                  activeforeground=accent_color, font=("Segoe UI", 7),
                  cursor="hand2").pack(side=tk.RIGHT, padx=(0, 4))
        tk.Button(header, text="✕",
                  command=lambda: (history_win.destroy(), globals().__setitem__("history_win", None)),
                  bg="#181818", fg="#888888", bd=0, activebackground="#181818",
                  font=("Segoe UI", 8, "bold"), cursor="hand2").pack(side=tk.RIGHT)
        history_listbox = tk.Listbox(
            hf, bg="#121212", fg="#d3d3d3", selectbackground="#282828",
            selectforeground="white", font=("Segoe UI", 8), bd=0, highlightthickness=0)
        history_listbox.pack(side=tk.LEFT, fill=tk.BOTH, expand=True, padx=6, pady=(0, 6))
        sc = tk.Scrollbar(hf, command=history_listbox.yview)
        sc.pack(side=tk.RIGHT, fill=tk.Y, pady=(0, 6))
        history_listbox.config(yscrollcommand=sc.set)
        history_listbox.bind("<Double-Button-1>", lambda e: play_from_history())
        refresh_history()
        bind_hover_recursive(history_win)
    else:
        history_win.destroy()
        history_win = None


def refresh_history():
    if not history_listbox or not history_listbox.winfo_exists():
        return
    history_listbox.delete(0, tk.END)
    for h in reversed(HISTORY[-50:]):
        try:
            tm = datetime.fromisoformat(h["time"]).strftime("%H:%M")
        except:
            tm = "--:--"
        history_listbox.insert(tk.END, f" [{tm}] {h['title'][:25]} - {h['artist'][:18]}")


def play_from_history():
    try:
        sel = history_listbox.curselection()
        if not sel:
            return
        idx = sel[0]
        real_idx = len(HISTORY[-50:]) - 1 - idx
        item = HISTORY[-50:][real_idx]
        sp.start_playback(uris=[f"spotify:track:{item['track_id']}"])
    except Exception as e:
        log_error(f"History play: {e}")


def create_playlist_from_history():
    try:
        if not HISTORY:
            return
        unique_ids = []
        seen = set()
        for h in reversed(HISTORY[-50:]):
            if h["track_id"] not in seen:
                seen.add(h["track_id"])
                unique_ids.append(h["track_id"])
            if len(unique_ids) >= 20:
                break
        if not unique_ids:
            return
        name = f"Mini Pro - {datetime.now().strftime('%Y-%m-%d %H:%M')}"
        user_id = sp.current_user()["id"]
        playlist = sp.user_playlist_create(user_id, name, public=False,
                                           description="Spotify Mini Pro")
        sp.playlist_add_items(playlist["id"], [f"spotify:track:{i}" for i in unique_ids])
        notify("Playlist oluşturuldu", name)
        webbrowser.open(playlist["external_urls"]["spotify"])
    except Exception as e:
        log_error(f"Create playlist: {e}")


# ============================================================
# FAVORİLER
# ============================================================
def toggle_favorites():
    global favorites_win, favorites_listbox
    if favorites_win is None or not favorites_win.winfo_exists():
        favorites_win = tk.Toplevel(root)
        favorites_win.overrideredirect(True)
        favorites_win.attributes("-topmost", True)
        favorites_win.attributes("-alpha", 0.95)
        favorites_win.geometry(f"360x260+{root.winfo_x()}+{root.winfo_y() + 132}")
        apply_rounded_window(favorites_win, 360, 260, 14)
        ff = tk.Frame(favorites_win, bg="#181818", highlightthickness=1,
                      highlightbackground=accent_color)
        ff.pack(fill=tk.BOTH, expand=True)
        header = tk.Frame(ff, bg="#181818")
        header.pack(fill=tk.X, padx=6, pady=(6, 2))
        tk.Label(header, text="  " + t("favorites").upper(), fg="white", bg="#181818",
                 font=("Segoe UI", 8, "bold"), anchor="w").pack(side=tk.LEFT)
        tk.Button(header, text="Temizle", command=clear_favorites,
                  bg="#181818", fg="#888888", bd=0, activebackground="#181818",
                  activeforeground="#ff5555", font=("Segoe UI", 7),
                  cursor="hand2").pack(side=tk.RIGHT, padx=(0, 4))
        tk.Button(header, text="✕",
                  command=lambda: (favorites_win.destroy(), globals().__setitem__("favorites_win", None)),
                  bg="#181818", fg="#888888", bd=0, activebackground="#181818",
                  font=("Segoe UI", 8, "bold"), cursor="hand2").pack(side=tk.RIGHT)
        favorites_listbox = tk.Listbox(
            ff, bg="#121212", fg="#d3d3d3", selectbackground="#282828",
            selectforeground="white", font=("Segoe UI", 8), bd=0, highlightthickness=0)
        favorites_listbox.pack(side=tk.LEFT, fill=tk.BOTH, expand=True, padx=6, pady=(0, 6))
        sc = tk.Scrollbar(ff, command=favorites_listbox.yview)
        sc.pack(side=tk.RIGHT, fill=tk.Y, pady=(0, 6))
        favorites_listbox.config(yscrollcommand=sc.set)
        favorites_listbox.bind("<Double-Button-1>", lambda e: play_from_favorites())
        refresh_favorites()
        bind_hover_recursive(favorites_win)
    else:
        favorites_win.destroy()
        favorites_win = None


def refresh_favorites():
    if not favorites_listbox or not favorites_listbox.winfo_exists():
        return
    favorites_listbox.delete(0, tk.END)
    if not FAVORITES:
        favorites_listbox.insert(tk.END, " Henüz favori yok.")
        return
    for f in reversed(FAVORITES[-100:]):
        favorites_listbox.insert(tk.END, f" ♥ {f['title'][:28]} - {f['artist'][:20]}")


def play_from_favorites():
    try:
        sel = favorites_listbox.curselection()
        if not sel:
            return
        idx = sel[0]
        real_idx = len(FAVORITES[-100:]) - 1 - idx
        item = FAVORITES[-100:][real_idx]
        sp.start_playback(uris=[f"spotify:track:{item['track_id']}"])
    except Exception as e:
        log_error(f"Favorites play: {e}")


def clear_favorites():
    global FAVORITES
    if messagebox.askyesno("Temizle", "Tüm favoriler silinsin mi?"):
        FAVORITES = []
        save_favorites(FAVORITES)
        refresh_favorites()


# ============================================================
# KISAYOLLAR
# ============================================================
SHORTCUTS = [
    ("Ctrl+Alt+Space", "Oynat / Duraklat"),
    ("Ctrl+Alt+→", "Sonraki şarkı"),
    ("Ctrl+Alt+←", "Önceki şarkı"),
    ("Ctrl+Alt+↑", "10sn ileri"),
    ("Ctrl+Alt+↓", "10sn geri"),
    ("Ctrl+Alt+L", "Beğen"),
    ("Ctrl+Alt+C", "Link kopyala"),
    ("Ctrl+Alt+N", "Menü aç"),
    ("Ctrl+Alt+F", "Favoriler"),
    ("Ctrl+Alt+Q", "Sıradakiler"),
    ("Ctrl+Alt+Y", "Şarkı sözleri"),
    ("Ctrl+Alt+H", "Geçmiş"),
    ("Ctrl+Alt+S", "Ara"),
    ("Ctrl+Alt+P", "Playlist'e ekle"),
    ("Ctrl+Alt+M", "Mini mod"),
    ("Ctrl+Alt+T", "Uyku zamanlayıcı"),
    ("Ctrl+Alt+K", "Ayarlar"),
    ("Ctrl+Alt+V", "Ses değeri gir"),
]


def toggle_shortcuts():
    global shortcuts_win
    if shortcuts_win is None or not shortcuts_win.winfo_exists():
        shortcuts_win = tk.Toplevel(root)
        shortcuts_win.overrideredirect(True)
        shortcuts_win.attributes("-topmost", True)
        shortcuts_win.attributes("-alpha", 0.95)
        shortcuts_win.geometry(f"360x400+{root.winfo_x()}+{root.winfo_y() + 132}")
        apply_rounded_window(shortcuts_win, 360, 400, 14)
        wf = tk.Frame(shortcuts_win, bg="#181818", highlightthickness=1,
                      highlightbackground=accent_color)
        wf.pack(fill=tk.BOTH, expand=True)
        header = tk.Frame(wf, bg="#181818")
        header.pack(fill=tk.X, padx=6, pady=(6, 2))
        tk.Label(header, text="  " + t("shortcuts").upper(), fg="white", bg="#181818",
                 font=("Segoe UI", 8, "bold"), anchor="w").pack(side=tk.LEFT)
        tk.Button(header, text="✕",
                  command=lambda: (shortcuts_win.destroy(), globals().__setitem__("shortcuts_win", None)),
                  bg="#181818", fg="#888888", bd=0, activebackground="#181818",
                  font=("Segoe UI", 8, "bold"), cursor="hand2").pack(side=tk.RIGHT)
        body = tk.Frame(wf, bg="#121212")
        body.pack(fill=tk.BOTH, expand=True, padx=6, pady=(0, 6))
        for i, (key, desc) in enumerate(SHORTCUTS):
            row = tk.Frame(body, bg="#121212" if i % 2 == 0 else "#181818")
            row.pack(fill=tk.X, pady=1)
            tk.Label(row, text=f" {key}", fg=accent_color, bg=row["bg"],
                     font=("Consolas", 8, "bold"), anchor="w", width=18).pack(side=tk.LEFT)
            tk.Label(row, text=desc, fg="#d3d3d3", bg=row["bg"],
                     font=("Segoe UI", 8), anchor="w").pack(side=tk.LEFT, padx=6)
        bind_hover_recursive(shortcuts_win)
    else:
        shortcuts_win.destroy()
        shortcuts_win = None


# ============================================================
# ARAMA
# ============================================================
def toggle_search():
    global search_win, search_entry, search_results
    if search_win is None or not search_win.winfo_exists():
        search_win = tk.Toplevel(root)
        search_win.overrideredirect(True)
        search_win.attributes("-topmost", True)
        search_win.attributes("-alpha", 0.95)
        search_win.geometry(f"360x260+{root.winfo_x()}+{root.winfo_y() + 132}")
        apply_rounded_window(search_win, 360, 260, 14)
        sf = tk.Frame(search_win, bg="#181818", highlightthickness=1,
                      highlightbackground=accent_color)
        sf.pack(fill=tk.BOTH, expand=True)
        header = tk.Frame(sf, bg="#181818")
        header.pack(fill=tk.X, padx=6, pady=(6, 2))
        tk.Label(header, text="  " + t("search").upper(), fg="white", bg="#181818",
                 font=("Segoe UI", 8, "bold"), anchor="w").pack(side=tk.LEFT)
        tk.Button(header, text="✕",
                  command=lambda: (search_win.destroy(), globals().__setitem__("search_win", None)),
                  bg="#181818", fg="#888888", bd=0, activebackground="#181818",
                  font=("Segoe UI", 8, "bold"), cursor="hand2").pack(side=tk.RIGHT)
        search_entry = tk.Entry(sf, bg="#282828", fg="white", bd=0,
                                insertbackground="white", font=("Segoe UI", 9),
                                highlightthickness=0)
        search_entry.pack(fill=tk.X, padx=6, pady=(0, 4), ipady=4)
        search_entry.bind("<Return>", lambda e: do_search())
        search_results = tk.Listbox(
            sf, bg="#121212", fg="#d3d3d3", selectbackground="#282828",
            selectforeground="white", font=("Segoe UI", 8), bd=0, highlightthickness=0)
        search_results.pack(fill=tk.BOTH, expand=True, padx=6, pady=(0, 6))
        search_results.bind("<Double-Button-1>", lambda e: queue_from_search())
        search_entry.focus_set()
        bind_hover_recursive(search_win)
    else:
        search_win.destroy()
        search_win = None


def do_search():
    global SEARCH_CACHE
    q = search_entry.get().strip()
    if not q:
        return
    try:
        res = sp.search(q=q, type="track", limit=15)
        items = res.get("tracks", {}).get("items", [])
        SEARCH_CACHE = items
        if search_results and search_results.winfo_exists():
            search_results.delete(0, tk.END)
            for tt in items:
                search_results.insert(tk.END, f" {tt['name'][:30]} - {tt['artists'][0]['name'][:20]}")
    except Exception as e:
        log_error(f"Search: {e}")


def queue_from_search():
    try:
        sel = search_results.curselection()
        if not sel:
            return
        tt = SEARCH_CACHE[sel[0]]
        sp.add_to_queue(tt["uri"])
        notify(t("added_queue"), tt["name"])
    except Exception as e:
        log_error(f"Queue from search: {e}")


# ============================================================
# PLAYLIST
# ============================================================
def toggle_playlist_adder():
    global playlist_win, playlist_listbox
    if playlist_win is None or not playlist_win.winfo_exists():
        playlist_win = tk.Toplevel(root)
        playlist_win.overrideredirect(True)
        playlist_win.attributes("-topmost", True)
        playlist_win.attributes("-alpha", 0.95)
        playlist_win.geometry(f"360x260+{root.winfo_x()}+{root.winfo_y() + 132}")
        apply_rounded_window(playlist_win, 360, 260, 14)
        pf = tk.Frame(playlist_win, bg="#181818", highlightthickness=1,
                      highlightbackground=accent_color)
        pf.pack(fill=tk.BOTH, expand=True)
        header = tk.Frame(pf, bg="#181818")
        header.pack(fill=tk.X, padx=6, pady=(6, 2))
        tk.Label(header, text="  " + t("playlist").upper(), fg="white", bg="#181818",
                 font=("Segoe UI", 8, "bold"), anchor="w").pack(side=tk.LEFT)
        tk.Button(header, text="✕",
                  command=lambda: (playlist_win.destroy(), globals().__setitem__("playlist_win", None)),
                  bg="#181818", fg="#888888", bd=0, activebackground="#181818",
                  font=("Segoe UI", 8, "bold"), cursor="hand2").pack(side=tk.RIGHT)
        playlist_listbox = tk.Listbox(
            pf, bg="#121212", fg="#d3d3d3", selectbackground="#282828",
            selectforeground="white", font=("Segoe UI", 8), bd=0, highlightthickness=0)
        playlist_listbox.pack(side=tk.LEFT, fill=tk.BOTH, expand=True, padx=6, pady=(0, 6))
        sc = tk.Scrollbar(pf, command=playlist_listbox.yview)
        sc.pack(side=tk.RIGHT, fill=tk.Y, pady=(0, 6))
        playlist_listbox.config(yscrollcommand=sc.set)
        playlist_listbox.bind("<Double-Button-1>", lambda e: add_to_selected_playlist())
        threading.Thread(target=fetch_playlists, daemon=True).start()
        bind_hover_recursive(playlist_win)
    else:
        playlist_win.destroy()
        playlist_win = None


def fetch_playlists():
    global PLAYLIST_CACHE
    try:
        data = sp.current_user_playlists(limit=50)
        PLAYLIST_CACHE = data.get("items", [])
        if playlist_listbox and playlist_listbox.winfo_exists():
            playlist_listbox.delete(0, tk.END)
            if not PLAYLIST_CACHE:
                playlist_listbox.insert(tk.END, " Playlist yok.")
            for p in PLAYLIST_CACHE:
                name = p["name"][:35]
                count = p.get("tracks", {}).get("total", "?")
                playlist_listbox.insert(tk.END, f" {name} ({count})")
    except Exception as e:
        log_error(f"Playlist fetch: {e}")


def add_to_selected_playlist():
    try:
        if not current_track_id_global:
            return
        sel = playlist_listbox.curselection()
        if not sel:
            return
        p = PLAYLIST_CACHE[sel[0]]
        sp.playlist_add_items(p["id"], [f"spotify:track:{current_track_id_global}"])
        notify("Eklendi", f"{current_title} -> {p['name']}")
        playlist_win.destroy()
        globals().__setitem__("playlist_win", None)
    except Exception as e:
        log_error(f"Playlist add: {e}")


# ============================================================
# UYKU ZAMANLAYICI
# ============================================================
def toggle_sleep_timer():
    global sleep_win
    if sleep_win is None or not sleep_win.winfo_exists():
        sleep_win = tk.Toplevel(root)
        sleep_win.overrideredirect(True)
        sleep_win.attributes("-topmost", True)
        sleep_win.attributes("-alpha", 0.95)
        sleep_win.geometry(f"220x220+{root.winfo_x() - 230}+{root.winfo_y()}")
        apply_rounded_window(sleep_win, 220, 220, 14)
        sf = tk.Frame(sleep_win, bg="#181818", highlightthickness=1,
                      highlightbackground=accent_color)
        sf.pack(fill=tk.BOTH, expand=True)
        tk.Label(sf, text="  " + t("sleep_timer"), fg="white", bg="#181818",
                 font=("Segoe UI", 8, "bold"), anchor="w").pack(fill=tk.X, padx=6, pady=(6, 4))
        for mins in [15, 30, 45, 60, 90]:
            tk.Button(sf, text=f"{mins} dk",
                      command=lambda m=mins: set_sleep_timer(m),
                      bg="#282828", fg="white", bd=0,
                      activebackground=accent_color, activeforeground="white",
                      font=("Segoe UI", 8), cursor="hand2").pack(fill=tk.X, padx=6, pady=2)
        if sleep_timer_end:
            remaining = int((sleep_timer_end - datetime.now()).total_seconds() / 60)
            tk.Label(sf, text=f"Kalan: {remaining} dk", fg=accent_color,
                     bg="#181818", font=("Segoe UI", 7)).pack(pady=4)
        tk.Button(sf, text="İptal", command=cancel_sleep_timer,
                  bg="#3a1a1a", fg="#ff8888", bd=0,
                  font=("Segoe UI", 7), cursor="hand2").pack(fill=tk.X, padx=6, pady=(4, 6))
    else:
        sleep_win.destroy()
        sleep_win = None


def set_sleep_timer(minutes):
    global sleep_timer_id, sleep_timer_end
    if sleep_timer_id:
        try:
            root.after_cancel(sleep_timer_id)
        except:
            pass
    sleep_timer_end = datetime.now() + timedelta(minutes=minutes)
    sleep_timer_id = root.after(minutes * 60 * 1000, sleep_pause)
    notify(t("sleep_timer"), f"{minutes} dk sonra duraklat")


def cancel_sleep_timer():
    global sleep_timer_id, sleep_timer_end
    if sleep_timer_id:
        try:
            root.after_cancel(sleep_timer_id)
        except:
            pass
    sleep_timer_id = None
    sleep_timer_end = None


def sleep_pause():
    global sleep_timer_id, sleep_timer_end
    try:
        sp.pause_playback()
        notify(t("sleep_timer"), "Duraklatıldı")
    except:
        pass
    sleep_timer_id = None
    sleep_timer_end = None


# ============================================================
# VISUALIZER
# ============================================================
vis_canvas = tk.Canvas(main_frame, width=32, height=14, bg="#121212", highlightthickness=0)
vis_canvas.place(x=145, y=7)
THEMED_WIDGETS.append(vis_canvas)

vis_bars = []
bar_w = 4
bar_gap = 3
for i in range(4):
    x0 = i * (bar_w + bar_gap)
    bar = vis_canvas.create_rectangle(x0, 12, x0 + bar_w, 14, fill=accent_color, outline="")
    vis_bars.append(bar)


def animate_visualizer():
    if is_playing_global:
        for bar in vis_bars:
            h = random.randint(3, 13)
            coords = vis_canvas.coords(bar)
            vis_canvas.coords(bar, coords[0], 14 - h, coords[2], 14)
    else:
        for bar in vis_bars:
            coords = vis_canvas.coords(bar)
            vis_canvas.coords(bar, coords[0], 12, coords[2], 14)
    root.after(110, animate_visualizer)


# ============================================================
# ALBÜM KAPAĞI + BİLGİ + PALET
# ============================================================
img_label = tk.Label(main_frame, bg="#121212", cursor="hand2")
img_label.place(x=12, y=12, width=65, height=65)
THEMED_WIDGETS.append(img_label)
img_label.bind("<Double-Button-1>", lambda e: open_in_spotify())

lbl_title = tk.Label(main_frame, text="Şarkı Aranıyor...", fg="white", bg="#121212",
                     font=("Segoe UI", SETTINGS["font_size"], "bold"),
                     anchor="w", cursor="hand2")
lbl_title.place(x=87, y=10, width=115)
THEMED_WIDGETS.append(lbl_title)
lbl_title.bind("<Double-Button-1>", lambda e: copy_track_title())

lbl_artist = tk.Label(main_frame, text="-", fg="#b3b3b3", bg="#121212",
                      font=("Segoe UI", SETTINGS["font_size"] - 1), anchor="w")
lbl_artist.place(x=87, y=26, width=150)
THEMED_WIDGETS.append(lbl_artist)

lbl_mood = tk.Label(main_frame, text="", fg=accent_color, bg="#121212",
                    font=("Segoe UI", 7, "italic"), anchor="w")
lbl_mood.place(x=87, y=40, width=150)
THEMED_WIDGETS.append(lbl_mood)

lbl_next = tk.Label(main_frame, text="", fg="#666666", bg="#121212",
                    font=("Segoe UI", 6, "italic"), anchor="w")
lbl_next.place(x=87, y=52, width=150)
THEMED_WIDGETS.append(lbl_next)

palette_frame = tk.Frame(main_frame, bg="#121212", height=10)
palette_frame.place(x=12, y=80, width=65, height=10)
THEMED_WIDGETS.append(palette_frame)

palette_dots = []
for i in range(3):
    dot_frame = tk.Frame(palette_frame, bg="#333", width=18, height=10,
                         cursor="hand2", highlightthickness=1,
                         highlightbackground="#444")
    dot_frame.pack(side=tk.LEFT, padx=1, pady=0)
    dot_frame.pack_propagate(False)
    dot_frame.bind("<Button-1>", lambda e, idx=i: apply_palette_color(idx))
    palette_dots.append(dot_frame)


def copy_track_title():
    if current_title:
        pyperclip.copy(current_title)
        notify(t("copied"), current_title)


def apply_palette_color(idx):
    if idx < len(palette):
        SETTINGS["accent_locked"] = False
        apply_accent_theme(palette[idx])
        notify("Renk değişti", palette[idx])


# ============================================================
# SAĞ TIK MENÜ
# ============================================================
def show_context_menu(event):
    menu = tk.Menu(root, tearoff=0, bg="#282828", fg="white",
                   activebackground=accent_color, activeforeground="white", bd=0)
    menu.add_command(label="🔗 Linki Kopyala", command=copy_current_link)
    menu.add_command(label="📋 Bilgi Kopyala", command=copy_track_info)
    menu.add_command(label="➕ Sıraya Ekle", command=lambda: queue_track(
        f"spotify:track:{current_track_id_global}") if current_track_id_global else None)
    menu.add_command(label="⭐ Favorilere Ekle", command=add_current_to_favorites)
    menu.add_command(label="📁 Playlist'e Ekle", command=toggle_playlist_adder)
    menu.add_separator()
    menu.add_command(label="🎨 Sanatçıyı Aç", command=open_artist_in_spotify)
    menu.add_command(label="💿 Albümü Aç", command=open_album_in_spotify)
    menu.add_separator()
    lock_label = "🔓 Rengi Kilitle" if not SETTINGS.get("accent_locked") else "🔒 Rengi Çöz"
    menu.add_command(label=lock_label, command=toggle_accent_lock)
    menu.add_command(label="⏱ Kalan Süre", command=toggle_remaining_time)
    menu.add_command(label="😴 Uyku Zamanlayıcı", command=toggle_sleep_timer)
    menu.add_command(label="🌐 Dil (TR/EN)", command=toggle_language)
    menu.add_separator()
    menu.add_command(label="📷 Story PNG", command=create_story_png)
    menu.add_command(label="⌨ Kısayollar", command=toggle_shortcuts)
    menu.add_command(label="⚙ Ayarlar", command=toggle_settings)
    menu.add_separator()
    menu.add_command(label="🔄 Kurulumu Sıfırla", command=reset_setup)
    menu.add_separator()
    menu.add_command(label="❌ Kapat", command=lambda: (save_settings(SETTINGS),
                                                        close_all_windows(), root.destroy()))
    try:
        menu.tk_popup(event.x_root, event.y_root)
    finally:
        menu.grab_release()


main_frame.bind("<Button-3>", show_context_menu)


# ★★★ YENİDEN KURULUM ★★★
def reset_setup():
    if messagebox.askyesno("Yeniden Kurulum",
                           "Kurulumu tekrar yapmak istiyor musun?\n"
                           "Mevcut Spotify/Discord bilgileri silinecek.\n"
                           "Uygulama yeniden başlatılmalı."):
        SETTINGS["client_id"] = ""
        SETTINGS["client_secret"] = ""
        SETTINGS["discord_client_id"] = ""
        SETTINGS["setup_completed"] = False
        save_settings(SETTINGS)
        # .cache dosyasını da sil
        try:
            if os.path.exists(CACHE_FILE):
                os.remove(CACHE_FILE)
        except:
            pass
        messagebox.showinfo("Tamam", "Uygulamayı yeniden başlat.")
        root.after(100, lambda: (close_all_windows(), root.destroy()))


def add_current_to_favorites():
    if current_track_id_global and current_title and current_artist:
        add_to_favorites(current_track_id_global, current_title, current_artist)
        notify(t("favorites"), f"{current_title} eklendi")


def toggle_accent_lock():
    SETTINGS["accent_locked"] = not SETTINGS.get("accent_locked", False)
    notify("Renk", "Kilitli" if SETTINGS["accent_locked"] else "Serbest")


def toggle_remaining_time():
    global show_remaining_time
    show_remaining_time = not show_remaining_time


def toggle_language():
    SETTINGS["language"] = "en" if SETTINGS.get("language", "tr") == "tr" else "tr"
    notify("Dil / Language", SETTINGS["language"].upper())


def open_in_spotify():
    if current_track_id_global and os.name == "nt":
        try:
            os.startfile(f"spotify:track:{current_track_id_global}")
        except:
            pass


def open_artist_in_spotify():
    try:
        current = sp.current_playback()
        if current and current["item"] and os.name == "nt":
            os.startfile(f"spotify:artist:{current['item']['artists'][0]['id']}")
    except:
        pass


def open_album_in_spotify():
    try:
        current = sp.current_playback()
        if current and current["item"] and os.name == "nt":
            os.startfile(f"spotify:album:{current['item']['album']['id']}")
    except:
        pass


# ============================================================
# STORY PNG
# ============================================================
def create_story_png():
    if not current_cover_bytes:
        return
    def worker():
        try:
            W, H = 1080, 1920
            canvas = Image.new("RGB", (W, H), "#121212")
            draw = ImageDraw.Draw(canvas)
            bg = Image.open(io.BytesIO(current_cover_bytes)).convert("RGB")
            bg = bg.resize((W, H), Image.Resampling.LANCZOS)
            bg = bg.filter(ImageFilter.GaussianBlur(60))
            dark = Image.new("RGB", (W, H), (0, 0, 0))
            bg = Image.blend(bg, dark, 0.6)
            canvas.paste(bg, (0, 0))
            cover = Image.open(io.BytesIO(current_cover_bytes)).convert("RGBA")
            cover = cover.resize((700, 700), Image.Resampling.LANCZOS)
            mask = Image.new("L", (700, 700), 0)
            ImageDraw.Draw(mask).rounded_rectangle((0, 0, 700, 700), radius=30, fill=255)
            cover.putalpha(mask)
            canvas.paste(cover, ((W - 700) // 2, 500), cover)
            try:
                font_big = ImageFont.truetype("C:/Windows/Fonts/segoeuib.ttf", 60)
                font_small = ImageFont.truetype("C:/Windows/Fonts/segoeui.ttf", 40)
            except:
                font_big = font_small = None
            if font_big:
                tw = draw.textlength(current_title[:35], font=font_big)
                draw.text(((W - tw) // 2, 1280), current_title[:35], fill="white", font=font_big)
            if font_small:
                aw = draw.textlength(current_artist[:40], font=font_small)
                draw.text(((W - aw) // 2, 1370), current_artist[:40], fill="#b3b3b3", font=font_small)
                bw = draw.textlength("Spotify Mini Pro", font=font_small)
                draw.text(((W - bw) // 2, 1780), "Spotify Mini Pro", fill=accent_color, font=font_small)
            out = os.path.join(BASE_DIR, f"story_{int(time.time())}.png")
            canvas.save(out)
            notify("Story kaydedildi", os.path.basename(out))
            if os.name == "nt":
                os.startfile(out)
        except Exception as e:
            log_error(f"Story: {e}")
    threading.Thread(target=worker, daemon=True).start()


# ============================================================
# AYARLAR
# ============================================================
def toggle_settings():
    global settings_win
    if settings_win is None or not settings_win.winfo_exists():
        settings_win = tk.Toplevel(root)
        settings_win.overrideredirect(True)
        settings_win.attributes("-topmost", True)
        settings_win.attributes("-alpha", 0.98)
        settings_win.geometry(f"360x460+{root.winfo_x()}+{root.winfo_y() + 132}")
        apply_rounded_window(settings_win, 360, 460, 14)
        sf = tk.Frame(settings_win, bg="#181818", highlightthickness=1,
                      highlightbackground=accent_color)
        sf.pack(fill=tk.BOTH, expand=True)
        header = tk.Frame(sf, bg="#181818")
        header.pack(fill=tk.X, padx=6, pady=(6, 2))
        tk.Label(header, text="  " + t("settings").upper(), fg="white", bg="#181818",
                 font=("Segoe UI", 8, "bold"), anchor="w").pack(side=tk.LEFT)
        tk.Button(header, text="✕",
                  command=lambda: (settings_win.destroy(), globals().__setitem__("settings_win", None)),
                  bg="#181818", fg="#888888", bd=0, activebackground="#181818",
                  font=("Segoe UI", 8, "bold"), cursor="hand2").pack(side=tk.RIGHT)

        body = tk.Frame(sf, bg="#181818")
        body.pack(fill=tk.BOTH, expand=True, padx=8, pady=6)

        def make_check(parent, label, key):
            var = tk.BooleanVar(value=SETTINGS.get(key, False))
            def cmd():
                SETTINGS[key] = var.get()
                save_settings(SETTINGS)
            tk.Checkbutton(parent, text=label, variable=var, command=cmd,
                           bg="#181818", fg="#d3d3d3", selectcolor="#282828",
                           activebackground="#181818", activeforeground="white",
                           font=("Segoe UI", 8), anchor="w", bd=0,
                           highlightthickness=0).pack(fill=tk.X, pady=1)

        make_check(body, "Blur arka plan", "blur_bg")
        make_check(body, "Discord RPC", "discord_rpc")
        make_check(body, "Bildirim göster", "notify_new_track")
        make_check(body, "Şarkı değişince bip", "beep_on_track")
        make_check(body, "Kenara yapıştır", "snap_to_edge")
        make_check(body, "Sonraki şarkı önizleme", "show_next_preview")
        make_check(body, "Sözleri otomatik kaydır", "auto_scroll_lyrics")
        make_check(body, "Mood rengi", "mood_color")
        make_check(body, "Her zaman üstte", "always_on_top")
        make_check(body, "Popülerlik göstergesi", "show_bitrate")

        tk.Label(body, text="Şeffaflık", fg="#888", bg="#181818",
                 font=("Segoe UI", 7)).pack(anchor="w", pady=(8, 0))
        alpha_var = tk.DoubleVar(value=SETTINGS.get("alpha", 0.9))
        def on_alpha(v):
            SETTINGS["alpha"] = float(v)
            root.attributes("-alpha", float(v))
        tk.Scale(body, from_=0.4, to=1.0, resolution=0.05, orient=tk.HORIZONTAL,
                 variable=alpha_var, command=on_alpha, bg="#181818", fg="white",
                 troughcolor="#282828", highlightthickness=0, bd=0,
                 font=("Segoe UI", 7)).pack(fill=tk.X)

        tk.Label(body, text="Şarkı sözü boyutu", fg="#888", bg="#181818",
                 font=("Segoe UI", 7)).pack(anchor="w", pady=(8, 0))
        lsize_var = tk.IntVar(value=SETTINGS.get("lyrics_font_size", 11))
        def on_lsize(v):
            SETTINGS["lyrics_font_size"] = int(v)
            if lyrics_text and lyrics_text.winfo_exists():
                sz = int(v)
                lyrics_text.config(font=("Segoe UI", sz))
                lyrics_text.tag_config("active", font=("Segoe UI", sz + 3, "bold"))
                lyrics_text.tag_config("past", font=("Segoe UI", sz))
                lyrics_text.tag_config("future", font=("Segoe UI", sz))
        tk.Scale(body, from_=8, to=20, orient=tk.HORIZONTAL,
                 variable=lsize_var, command=on_lsize, bg="#181818", fg="white",
                 troughcolor="#282828", highlightthickness=0, bd=0,
                 font=("Segoe UI", 7)).pack(fill=tk.X)

        tk.Button(body, text="Kaydet & Kapat",
                  command=lambda: (save_settings(SETTINGS),
                                   settings_win.destroy(),
                                   globals().__setitem__("settings_win", None)),
                  bg=accent_color, fg="white", bd=0, activebackground=accent_color,
                  font=("Segoe UI", 8, "bold"), cursor="hand2").pack(fill=tk.X, pady=(12, 0), ipady=6)
        bind_hover_recursive(settings_win)
    else:
        settings_win.destroy()
        settings_win = None


# ============================================================
# KALP
# ============================================================
is_saved = False
heart_animating = False


def toggle_like():
    global is_saved, heart_animating
    if heart_animating:
        return
    try:
        current = sp.current_playback()
        if current and current["item"]:
            track_id = [current["item"]["id"]]
            if is_saved:
                sp.current_user_saved_tracks_delete(track_id)
                is_saved = False
                btn_like.config(text="♡", fg="#b3b3b3")
            else:
                sp.current_user_saved_tracks_add(track_id)
                is_saved = True
                btn_like.config(text="♥", fg="#ffd93d")
                notify(t("liked"), current["item"]["name"])
                add_current_to_favorites()
                threading.Thread(target=animate_heart, daemon=True).start()
    except Exception as e:
        log_error(f"Like: {e}")


def animate_heart():
    global heart_animating
    heart_animating = True
    try:
        for size in [11, 13, 15, 17, 15, 13, 11]:
            btn_like.config(font=("Segoe UI", size), fg="#ffd93d")
            root.update_idletasks()
            time.sleep(0.04)
        for col in ["#ffd93d", "#e0c230", accent_color]:
            btn_like.config(fg=col)
            root.update_idletasks()
            time.sleep(0.05)
        btn_like.config(fg=accent_color, font=("Segoe UI", 11))
    except:
        pass
    finally:
        heart_animating = False


btn_like = tk.Button(main_frame, text="♡", command=toggle_like,
                     bg="#121212", fg="#b3b3b3", bd=0,
                     activebackground="#121212",
                     font=("Segoe UI", 11),
                     cursor="hand2")
btn_like.place(x=240, y=22)
THEMED_WIDGETS.append(btn_like)


# ============================================================
# MEDYA KONTROLLERİ
# ============================================================
def prev_song():
    try: sp.previous_track()
    except: pass


def play_pause_song():
    try:
        playback = sp.current_playback()
        if playback and playback["is_playing"]:
            sp.pause_playback()
        else:
            sp.start_playback()
    except: pass


def next_song():
    try: sp.next_track()
    except: pass


def seek_relative(delta_ms):
    try:
        current = sp.current_playback()
        if current and current["item"]:
            new_pos = max(0, min(current["progress_ms"] + delta_ms,
                                 current["item"]["duration_ms"] - 1000))
            sp.seek_track(new_pos)
    except: pass


is_shuffle = False
repeat_state = "off"


def toggle_shuffle():
    global is_shuffle
    try:
        sp.shuffle(not is_shuffle)
        is_shuffle = not is_shuffle
        btn_shuffle.config(fg=accent_color if is_shuffle else "#b3b3b3")
    except: pass


def toggle_repeat():
    global repeat_state
    try:
        states = ["off", "context", "track"]
        next_index = (states.index(repeat_state) + 1) % 3
        repeat_state = states[next_index]
        sp.repeat(repeat_state)
        btn_repeat.config(
            fg=accent_color if repeat_state != "off" else "#b3b3b3",
            text="↻" if repeat_state == "track" else "⟳")
    except: pass


btn_shuffle = tk.Button(main_frame, text="⤨", command=toggle_shuffle,
                        bg="#121212", fg="#b3b3b3", bd=0,
                        font=("Segoe UI", 10), cursor="hand2")
btn_shuffle.place(x=87, y=72)
THEMED_WIDGETS.append(btn_shuffle)

btn_prev = tk.Button(main_frame, text="◀◀", command=prev_song,
                     bg="#121212", fg="white", bd=0,
                     font=("Segoe UI", 9), cursor="hand2")
btn_prev.place(x=110, y=70)
THEMED_WIDGETS.append(btn_prev)

btn_play = tk.Button(main_frame, text="❚❚", command=play_pause_song,
                     bg="#121212", fg="white", bd=0,
                     font=("Segoe UI", 11, "bold"), cursor="hand2")
btn_play.place(x=137, y=68)
THEMED_WIDGETS.append(btn_play)

btn_next = tk.Button(main_frame, text="▶▶", command=next_song,
                     bg="#121212", fg="white", bd=0,
                     font=("Segoe UI", 9), cursor="hand2")
btn_next.place(x=162, y=70)
THEMED_WIDGETS.append(btn_next)

btn_repeat = tk.Button(main_frame, text="⟳", command=toggle_repeat,
                       bg="#121212", fg="#b3b3b3", bd=0,
                       font=("Segoe UI", 10), cursor="hand2")
btn_repeat.place(x=187, y=72)
THEMED_WIDGETS.append(btn_repeat)


# ============================================================
# SES SEVİYESİ
# ============================================================
current_volume = 50
volume_debounce = None
VOL_BAR_WIDTH = 60


def apply_volume():
    try:
        sp.volume(int(current_volume))
    except:
        pass


def _set_volume_visual(rel_x):
    rel_x = max(0, min(rel_x, VOL_BAR_WIDTH))
    try:
        vol_fill.place(x=0, y=0, width=rel_x, height=10)
        knob_x = max(0, min(rel_x - 5, VOL_BAR_WIDTH - 10))
        vol_knob.place(x=knob_x, y=0)
        lbl_vol_percent.config(text=f"{current_volume}%")
    except Exception as e:
        log_error(f"Vol visual: {e}")


def _update_volume_from_root_x(x_root):
    global current_volume, volume_debounce
    try:
        vol_x = vol_bg.winfo_rootx()
        rel = x_root - vol_x
        rel = max(0, min(rel, VOL_BAR_WIDTH))
    except:
        return
    current_volume = int((rel / VOL_BAR_WIDTH) * 100)
    _set_volume_visual(rel)
    if volume_debounce:
        try:
            root.after_cancel(volume_debounce)
        except:
            pass
    volume_debounce = root.after(80, apply_volume)


def on_vol_press(event):
    global is_dragging_volume
    is_dragging_volume = True
    _update_volume_from_root_x(event.x_root)


def on_vol_drag(event):
    if is_dragging_volume:
        _update_volume_from_root_x(event.x_root)


def on_vol_release(event):
    global is_dragging_volume
    is_dragging_volume = False
    apply_volume()


def ask_volume_value():
    global current_volume
    try:
        v = simpledialog.askinteger(t("volume"), t("volume_prompt"),
                                    initialvalue=current_volume,
                                    minvalue=0, maxvalue=100, parent=root)
        if v is not None:
            current_volume = int(v)
            rel = int((current_volume / 100) * VOL_BAR_WIDTH)
            _set_volume_visual(rel)
            apply_volume()
            notify(t("volume"), f"{current_volume}%")
    except Exception as e:
        log_error(f"Volume dialog: {e}")


vol_bg = tk.Frame(main_frame, bg="#3a3a3a", width=VOL_BAR_WIDTH, height=10,
                  cursor="hand2")
vol_bg.place(x=265, y=56)
vol_bg.pack_propagate(False)

vol_fill = tk.Frame(vol_bg, bg=accent_color, width=30, height=10)
vol_fill.place(x=0, y=0, width=30, height=10)

vol_knob = tk.Frame(vol_bg, bg="white", width=10, height=10)
vol_knob.place(x=25, y=0)

for w in [vol_bg, vol_fill, vol_knob]:
    w.bind("<Button-1>", on_vol_press)
    w.bind("<B1-Motion>", on_vol_drag)
    w.bind("<ButtonRelease-1>", on_vol_release)
    w.bind("<Button-3>", lambda e: ask_volume_value())
    w.bind("<Double-Button-1>", lambda e: ask_volume_value())

lbl_vol_icon = tk.Label(main_frame, text="♪", fg="#b3b3b3", bg="#121212",
                        font=("Segoe UI", 9), cursor="hand2")
lbl_vol_icon.place(x=242, y=52)
lbl_vol_icon.bind("<Double-Button-1>", lambda e: ask_volume_value())
lbl_vol_icon.bind("<Button-3>", lambda e: ask_volume_value())
THEMED_WIDGETS.append(lbl_vol_icon)

lbl_vol_percent = tk.Label(main_frame, text="50%", fg="#d3d3d3", bg="#121212",
                           font=("Segoe UI", 7, "bold"), cursor="hand2")
lbl_vol_percent.place(x=326, y=52)
lbl_vol_percent.bind("<Double-Button-1>", lambda e: ask_volume_value())
lbl_vol_percent.bind("<Button-3>", lambda e: ask_volume_value())
THEMED_WIDGETS.append(lbl_vol_percent)


def on_mouse_wheel(event):
    global current_volume, volume_debounce
    if event.state & 0x1:
        seek_relative(10000 if event.delta > 0 else -10000)
        return
    if event.delta > 0:
        current_volume = min(100, current_volume + 5)
    else:
        current_volume = max(0, current_volume - 5)
    rel = int((current_volume / 100) * VOL_BAR_WIDTH)
    _set_volume_visual(rel)
    if volume_debounce:
        try:
            root.after_cancel(volume_debounce)
        except:
            pass
    volume_debounce = root.after(150, apply_volume)


main_frame.bind("<MouseWheel>", on_mouse_wheel)


# ============================================================
# İLERLEME ÇUBUĞU
# ============================================================
lbl_time = tk.Label(main_frame, text="0:00 / 0:00", fg="#b3b3b3", bg="#121212",
                    font=("Segoe UI", 7))
lbl_time.place(x=290, y=85)
THEMED_WIDGETS.append(lbl_time)

progress_canvas = tk.Canvas(main_frame, width=336, height=16,
                            bg="#121212", highlightthickness=0, cursor="hand2")
progress_canvas.place(x=12, y=98)
THEMED_WIDGETS.append(progress_canvas)

progress_bg_rect = progress_canvas.create_rectangle(
    0, 6, 336, 10, fill="#404040", outline="")
progress_fill_rect = progress_canvas.create_rectangle(
    0, 6, 0, 10, fill=accent_color, outline="")
progress_knob = progress_canvas.create_oval(
    0, 2, 16, 14, fill="#ff2d55", outline="white", width=2)

PROGRESS_W = 336


def _update_progress_visual(ratio):
    ratio = max(0, min(ratio, 1))
    x = ratio * PROGRESS_W
    try:
        progress_canvas.coords(progress_fill_rect, 0, 6, x, 10)
        progress_canvas.coords(progress_knob, x - 8, 2, x + 8, 14)
    except:
        pass


def seek_from_progress(event):
    global is_dragging_progress
    try:
        x = event.x_root - progress_canvas.winfo_rootx()
        x = max(0, min(x, PROGRESS_W))
        ratio = x / PROGRESS_W
        _update_progress_visual(ratio)
        current = sp.current_playback()
        if current and current["item"]:
            target = int(current["item"]["duration_ms"] * ratio)
            sp.seek_track(target)
    except Exception as e:
        log_error(f"Seek: {e}")


def on_progress_press(event):
    global is_dragging_progress
    is_dragging_progress = True
    seek_from_progress(event)


def on_progress_drag(event):
    if is_dragging_progress:
        try:
            x = event.x_root - progress_canvas.winfo_rootx()
            x = max(0, min(x, PROGRESS_W))
            ratio = x / PROGRESS_W
            _update_progress_visual(ratio)
        except:
            pass


def on_progress_release(event):
    global is_dragging_progress
    is_dragging_progress = False
    seek_from_progress(event)


progress_canvas.bind("<Button-1>", on_progress_press)
progress_canvas.bind("<B1-Motion>", on_progress_drag)
progress_canvas.bind("<ButtonRelease-1>", on_progress_release)


# ============================================================
# TEMA
# ============================================================
def apply_accent_theme(color_hex):
    global accent_color
    accent_color = color_hex
    try:
        main_frame.config(highlightbackground=accent_color)
        progress_canvas.itemconfig(progress_fill_rect, fill=accent_color)
        vol_fill.config(bg=accent_color)
        lbl_mood.config(fg=accent_color)
        if is_saved:
            if not heart_animating:
                btn_like.config(fg=accent_color)
        if is_shuffle: btn_shuffle.config(fg=accent_color)
        if repeat_state != "off": btn_repeat.config(fg=accent_color)
        for bar in vis_bars:
            vis_canvas.itemconfig(bar, fill=accent_color)
        if lyrics_text and lyrics_text.winfo_exists():
            lyrics_text.tag_config("active",
                                   foreground=THEME.get("lyrics_active_color", "#ffd93d"),
                                   background=THEME.get("lyrics_bg_active", "#2a2a2a"),
                                   font=("Segoe UI", SETTINGS.get("lyrics_font_size", 11) + 3, "bold"),
                                   underline=True, justify="center")
    except Exception as e:
        log_error(f"Theme: {e}")


def apply_dynamic_bg(avg_hex):
    for w in THEMED_WIDGETS:
        try: w.config(bg=avg_hex)
        except: pass
    try:
        main_frame.config(bg=avg_hex)
        bg_canvas.config(bg=avg_hex)
    except: pass


def update_palette_dots(new_palette):
    global palette
    palette = new_palette
    for i, dot in enumerate(palette_dots):
        try:
            if i < len(palette):
                dot.config(bg=palette[i])
            else:
                dot.config(bg="#333")
        except: pass


# ============================================================
# PULSE / ZOOM
# ============================================================
def pulse_animation():
    try:
        base = SETTINGS.get("alpha", 0.90)
        for a in [base, min(base + 0.05, 1.0), 1.0, min(base + 0.05, 1.0), base]:
            root.attributes("-alpha", a)
            root.update()
            time.sleep(0.04)
        root.attributes("-alpha", base)
    except: pass


def animate_cover_zoom(image_bytes):
    try:
        for size in range(35, 66, 5):
            img = make_rounded_cover(image_bytes, size=size)
            img_label.config(image=img)
            img_label.image = img
            root.update_idletasks()
            time.sleep(0.015)
        final = make_rounded_cover(image_bytes, size=65)
        img_label.config(image=final)
        img_label.image = final
    except: pass


# ============================================================
# SPOTIFY VERİ DÖNGÜSÜ
# ============================================================
def update_spotify_data():
    global current_img_url, img_cache, is_saved, is_playing_global
    global current_track_id_global, current_progress_ms, current_duration_ms
    global current_title, current_artist, current_mood, current_bg_photo
    global current_album, last_notify_time, is_shuffle, repeat_state
    global current_cover_bytes, next_track_name, next_track_artist, current_volume
    global current_popularity

    try:
        current = sp.current_playback()
        if current and current["item"]:
            is_playing_global = current["is_playing"]
            track = current["item"]
            title = track["name"]
            artist = track["artists"][0]["name"]
            album = track["album"]["name"]
            track_id = track["id"]
            progress_ms = current["progress_ms"]
            duration_ms = track["duration_ms"]
            current_popularity = track.get("popularity", 0)

            current_progress_ms = progress_ms
            current_duration_ms = duration_ms
            current_title = title
            current_artist = artist
            current_album = album

            if not is_dragging_volume and "device" in current:
                try:
                    vol = current["device"].get("volume_percent", None)
                    if vol is not None and abs(vol - current_volume) >= 2:
                        current_volume = int(vol)
                        rel = int((current_volume / 100) * VOL_BAR_WIDTH)
                        _set_volume_visual(rel)
                except:
                    pass

            if "shuffle_state" in current:
                is_shuffle = current.get("shuffle_state", False)
                btn_shuffle.config(fg=accent_color if is_shuffle else "#b3b3b3")
            if "repeat_state" in current:
                repeat_state = current.get("repeat_state", "off")
                btn_repeat.config(
                    fg=accent_color if repeat_state != "off" else "#b3b3b3",
                    text="↻" if repeat_state == "track" else "⟳")

            lbl_title.config(text=title if len(title) <= 16 else title[:14] + "...")
            lbl_artist.config(text=artist if len(artist) <= 22 else artist[:19] + "...")

            if SETTINGS.get("show_mood", True):
                current_mood = analyze_mood(title, artist)
                icon = get_mood_icon(current_mood)
                lbl_mood.config(text=f"{icon} {current_mood}")
                if SETTINGS.get("mood_color"):
                    try: lbl_mood.config(fg=get_mood_color(current_mood))
                    except: pass

            p_min, p_sec = divmod(int(progress_ms / 1000), 60)
            d_min, d_sec = divmod(int(duration_ms / 1000), 60)
            if show_remaining_time:
                rem = int((duration_ms - progress_ms) / 1000)
                r_min, r_sec = divmod(rem, 60)
                lbl_time.config(text=f"-{r_min}:{r_sec:02d}")
            else:
                lbl_time.config(text=f"{p_min}:{p_sec:02d} / {d_min}:{d_sec:02d}")

            if not is_dragging_progress:
                ratio = progress_ms / duration_ms if duration_ms > 0 else 0
                _update_progress_visual(ratio)

            btn_play.config(text="❚❚" if is_playing_global else "▶")

            images = track["album"]["images"]
            img_url = images[0]["url"] if images else ""

            if track_id != current_track_id_global:
                current_track_id_global = track_id
                add_to_history(track_id, title, artist, img_url)
                threading.Thread(target=pulse_animation, daemon=True).start()
                beep()
                now = time.time()
                if (SETTINGS.get("notify_new_track", True)
                        and not root.focus_get()
                        and (now - last_notify_time) > 3):
                    last_notify_time = now
                    notify(t("now_playing"), f"{title} - {artist}")
                if lyrics_win and lyrics_win.winfo_exists():
                    threading.Thread(target=load_lyrics_async, daemon=True).start()
                if queue_win and queue_win.winfo_exists():
                    threading.Thread(target=fetch_queue, daemon=True).start()
                if history_win and history_win.winfo_exists():
                    refresh_history()
                threading.Thread(target=fetch_next_preview, daemon=True).start()
                if now - last_recommendation_time > 15:
                    last_recommendation_time = now
                    threading.Thread(target=show_recommendation, daemon=True).start()

            if images and img_url != current_img_url:
                current_img_url = img_url
                try:
                    res = requests.get(images[-1]["url"], timeout=5)
                    current_cover_bytes = res.content
                    threading.Thread(target=animate_cover_zoom,
                                     args=(res.content,), daemon=True).start()
                    if SETTINGS.get("blur_bg", True):
                        blur = make_blurred_bg(res.content, size=(360, 130))
                        current_bg_photo = ImageTk.PhotoImage(blur)
                        bg_canvas.itemconfig(bg_image_id, image=current_bg_photo)
                        bg_canvas.image = current_bg_photo
                        avg_hex = image_to_avg_hex(blur)
                        apply_dynamic_bg(avg_hex)
                    if SETTINGS.get("accent_mode") == "auto" and not SETTINGS.get("accent_locked", False):
                        new_palette = get_palette(res.content, color_count=3)
                        update_palette_dots(new_palette)
                        apply_accent_theme(new_palette[0])
                except Exception as e:
                    log_error(f"Cover: {e}")
                try:
                    is_saved = sp.current_user_saved_tracks_contains([track_id])[0]
                    if not heart_animating:
                        btn_like.config(text="♥" if is_saved else "♡",
                                        fg=accent_color if is_saved else "#b3b3b3")
                except: pass

            if lyrics_win and lyrics_win.winfo_exists():
                highlight_lyrics(progress_ms)

            update_discord_presence(title, artist, img_url, is_playing_global,
                                    progress_ms, duration_ms, track_id)
            if is_playing_global:
                today = datetime.now().strftime("%Y-%m-%d")
                if today not in LISTEN_TIME:
                    LISTEN_TIME[today] = {"total": 0, "tracks": {}}
                LISTEN_TIME[today]["total"] += 1
                LISTEN_TIME[today]["tracks"][track_id] = \
                    LISTEN_TIME[today]["tracks"].get(track_id, 0) + 1
                if LISTEN_TIME[today]["total"] % 30 == 0:
                    threading.Thread(target=save_listen_time, args=(LISTEN_TIME,), daemon=True).start()

        else:
            is_playing_global = False
            lbl_title.config(text="Spotify Kapalı")
            lbl_artist.config(text="Müzik aç kanka")
            lbl_mood.config(text="")
            lbl_next.config(text="")
            lbl_time.config(text="0:00 / 0:00")
            _update_progress_visual(0)
            if rpc:
                try: rpc.clear()
                except: pass
    except Exception as e:
        log_error(f"Update: {e}")

    root.after(1000, update_spotify_data)


def fetch_next_preview():
    global next_track_name, next_track_artist
    try:
        q = sp.queue()
        items = q.get("queue", [])
        if items:
            next_track_name = items[0]["name"]
            next_track_artist = items[0]["artists"][0]["name"]
            def update():
                if SETTINGS.get("show_next_preview", True):
                    lbl_next.config(text=f"→ {next_track_name[:18]} - {next_track_artist[:14]}")
            root.after(0, update)
    except: pass


# ============================================================
# ÖNERİ
# ============================================================
def show_recommendation():
    global recommendation_win
    try:
        if not current_track_id_global:
            return
        recs = sp.recommendations(seed_tracks=[current_track_id_global], limit=3)
        tracks = recs.get("tracks", [])
        if not tracks:
            return
        def build():
            global recommendation_win
            if recommendation_win and recommendation_win.winfo_exists():
                recommendation_win.destroy()
            recommendation_win = tk.Toplevel(root)
            recommendation_win.overrideredirect(True)
            recommendation_win.attributes("-topmost", True)
            recommendation_win.attributes("-alpha", 0.95)
            recommendation_win.geometry(f"260x100+{root.winfo_x() - 270}+{root.winfo_y()}")
            apply_rounded_window(recommendation_win, 260, 100, 14)
            rf = tk.Frame(recommendation_win, bg="#181818", highlightthickness=1,
                          highlightbackground=accent_color)
            rf.pack(fill=tk.BOTH, expand=True)
            tk.Label(rf, text="  Bunları da sevebilirsin", fg="white", bg="#181818",
                     font=("Segoe UI", 8, "bold"), anchor="w").pack(fill=tk.X, padx=6, pady=(6, 2))
            for tt in tracks[:3]:
                name = tt["name"][:22]
                art = tt["artists"][0]["name"][:15]
                lbl = tk.Label(rf, text=f"  {name} - {art}", fg="#d3d3d3", bg="#181818",
                               font=("Segoe UI", 8), anchor="w", cursor="hand2")
                lbl.pack(fill=tk.X, padx=4)
                lbl.bind("<Button-1>", lambda e, uri=tt["uri"]: queue_track(uri))
            recommendation_win.after(8000, lambda: recommendation_win.destroy()
                                     if recommendation_win and recommendation_win.winfo_exists() else None)
        root.after(0, build)
    except Exception as e:
        log_error(f"Rec: {e}")


def queue_track(uri):
    try:
        sp.add_to_queue(uri)
        notify(t("added_queue"), "✓")
    except Exception as e:
        log_error(f"Queue track: {e}")


def copy_current_link():
    if current_track_id_global:
        pyperclip.copy(f"https://open.spotify.com/track/{current_track_id_global}")
        notify(t("copied"), "Link")


def copy_track_info():
    if current_title and current_artist:
        pyperclip.copy(f"{current_title} - {current_artist} ({current_album})")
        notify(t("copied"), "Bilgi")


# ============================================================
# GLOBAL KISAYOLLAR
# ============================================================
def setup_hotkeys():
    if not SETTINGS.get("hotkeys_enabled", True):
        return
    try:
        keyboard.add_hotkey("ctrl+alt+space", lambda: root.after(0, play_pause_song))
        keyboard.add_hotkey("ctrl+alt+right", lambda: root.after(0, next_song))
        keyboard.add_hotkey("ctrl+alt+left", lambda: root.after(0, prev_song))
        keyboard.add_hotkey("ctrl+alt+l", lambda: root.after(0, toggle_like))
        keyboard.add_hotkey("ctrl+alt+c", lambda: root.after(0, copy_current_link))
        keyboard.add_hotkey("ctrl+alt+q", lambda: root.after(0, toggle_queue))
        keyboard.add_hotkey("ctrl+alt+y", lambda: root.after(0, toggle_lyrics))
        keyboard.add_hotkey("ctrl+alt+h", lambda: root.after(0, toggle_history))
        keyboard.add_hotkey("ctrl+alt+s", lambda: root.after(0, toggle_search))
        keyboard.add_hotkey("ctrl+alt+p", lambda: root.after(0, toggle_playlist_adder))
        keyboard.add_hotkey("ctrl+alt+m", lambda: root.after(0, toggle_mini_mode))
        keyboard.add_hotkey("ctrl+alt+t", lambda: root.after(0, toggle_sleep_timer))
        keyboard.add_hotkey("ctrl+alt+k", lambda: root.after(0, toggle_settings))
        keyboard.add_hotkey("ctrl+alt+up", lambda: seek_relative(10000))
        keyboard.add_hotkey("ctrl+alt+down", lambda: seek_relative(-10000))
        keyboard.add_hotkey("ctrl+alt+v", lambda: root.after(0, ask_volume_value))
        keyboard.add_hotkey("ctrl+alt+n", lambda: root.after(0, show_main_menu))
        keyboard.add_hotkey("ctrl+alt+f", lambda: root.after(0, toggle_favorites))
    except Exception as e:
        log_error(f"Hotkeys: {e}")


# ============================================================
# MİNİ MOD
# ============================================================
def toggle_mini_mode():
    global is_mini
    is_mini = not SETTINGS.get("mini_mode", False)
    SETTINGS["mini_mode"] = is_mini
    if is_mini:
        root.geometry(f"220x60+{root.winfo_x()}+{root.winfo_y()}")
        main_frame.place(x=1, y=1, width=218, height=58)
        for w in [btn_shuffle, btn_repeat, vis_canvas, lbl_mood, lbl_next, lbl_time,
                  progress_canvas, vol_bg, lbl_vol_icon, lbl_vol_percent,
                  palette_frame, lbl_artist, btn_menu]:
            try: w.place_forget()
            except: pass
        lbl_title.place(x=87, y=20, width=100)
        img_label.place(x=12, y=8, width=45, height=45)
        img_label.config(width=45, height=45)
        btn_like.place(x=190, y=20)
    else:
        root.geometry(f"360x130+{root.winfo_x()}+{root.winfo_y()}")
        main_frame.place(x=1, y=1, width=358, height=128)
        img_label.place(x=12, y=12, width=65, height=65)
        img_label.config(width=65, height=65)
        lbl_title.place(x=87, y=10, width=115)
        lbl_artist.place(x=87, y=26, width=150)
        lbl_mood.place(x=87, y=40, width=150)
        lbl_next.place(x=87, y=52, width=150)
        btn_like.place(x=240, y=22)
        btn_shuffle.place(x=87, y=72)
        btn_prev.place(x=110, y=70)
        btn_play.place(x=137, y=68)
        btn_next.place(x=162, y=70)
        btn_repeat.place(x=187, y=72)
        vis_canvas.place(x=145, y=7)
        lbl_time.place(x=290, y=85)
        progress_canvas.place(x=12, y=98)
        vol_bg.place(x=265, y=56)
        lbl_vol_icon.place(x=242, y=52)
        lbl_vol_percent.place(x=326, y=52)
        palette_frame.place(x=12, y=80, width=65, height=10)
        btn_menu.place(x=308, y=2, width=20, height=20)


# ============================================================
# SİSTEM TEPSİSİ
# ============================================================
def create_tray_icon():
    def show_window(icon, item): root.after(0, root.deiconify)
    def hide_window(icon, item):
        root.after(0, lambda: (root.withdraw(), notify("Spotify Mini", "Arka planda")))
    def quit_app(icon, item):
        icon.stop()
        root.after(0, lambda: (save_settings(SETTINGS), close_all_windows(), root.destroy()))
    try:
        image = Image.new("RGB", (64, 64), "#1db954")
        d = ImageDraw.Draw(image)
        d.ellipse((12, 12, 52, 52), fill="white")
        menu = pystray.Menu(
            pystray.MenuItem("Göster", show_window, default=True),
            pystray.MenuItem("Gizle", hide_window),
            pystray.MenuItem("Çıkış", quit_app),
        )
        icon = pystray.Icon("SpotifyMini", image, "Spotify Mini Pro", menu)
        threading.Thread(target=icon.run, daemon=True).start()
    except Exception as e:
        log_error(f"Tray: {e}")


# ============================================================
# BAŞLAT
# ============================================================
if SETTINGS.get("accent_mode") == "custom":
    apply_accent_theme(SETTINGS.get("custom_accent", "#1db954"))

root.after(100, lambda: apply_rounded_window(root, 360, 130, 14))
root.after(200, lambda: bind_hover_recursive(root))
root.after(500, update_spotify_data)
root.after(100, animate_visualizer)
setup_hotkeys()
create_tray_icon()

if SETTINGS.get("start_minimized", False):
    root.withdraw()
if SETTINGS.get("mini_mode", False):
    root.after(600, toggle_mini_mode)


def on_close():
    save_settings(SETTINGS)
    close_all_windows()
    root.destroy()


root.protocol("WM_DELETE_WINDOW", on_close)
root.mainloop()
