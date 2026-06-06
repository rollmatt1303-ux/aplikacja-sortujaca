import streamlit as st
import os
import json
import shutil
import time
from pathlib import Path
from collections import deque
from watchdog.observers import Observer
from watchdog.events import FileSystemEventHandler

# --- 1. OBSŁUGA USTAWIEŃ (JSON) ---
PLIK_USTAWIEN = "ustawienia.json"

DOMYSLNE_USTAWIENIA = {
    "monitorowany_folder": "",
    "kategorie": {
        "Dokumenty": [".pdf", ".docx", ".doc", ".txt", ".xlsx", ".pptx", ".csv"],
        "Obrazy": [".jpg", ".jpeg", ".png", ".gif", ".bmp", ".svg"],
        "Wideo": [".mp4", ".mkv", ".avi", ".mov"],
        "Muzyka": [".mp3", ".wav", ".flac"],
        "Archiwa": [".zip", ".rar", ".7z", ".tar"],
        "Programy": [".exe", ".msi", ".iso"]
    }
}

def wczytaj_ustawienia():
    if os.path.exists(PLIK_USTAWIEN):
        try:
            with open(PLIK_USTAWIEN, 'r', encoding='utf-8') as f:
                return json.load(f)
        except json.JSONDecodeError:
            pass
    return DOMYSLNE_USTAWIENIA.copy()

def zapisz_ustawienia(ustawienia):
    with open(PLIK_USTAWIEN, 'w', encoding='utf-8') as f:
        json.dump(ustawienia, f, indent=4, ensure_ascii=False)

# --- 2. ZARZĄDZANIE STANEM W TLE (CACHE) ---
# Używamy cache_resource, aby wątek Watchdog oraz logi przetrwały odświeżanie UI Streamlit
@st.cache_resource
def pobierz_wspoldzielone_dane():
    return {
        "observer": None,
        "logi": deque(maxlen=50) # Przechowuje tylko 50 ostatnich logów
    }

wspoldzielone = pobierz_wspoldzielone_dane()

# --- 3. LOGIKA MONITOROWANIA (WATCHDOG) ---
class OrganizatorHandler(FileSystemEventHandler):
    def __init__(self, folder, kategorie):
        super().__init__()
        self.folder = Path(folder)
        self.kategorie = kategorie
        self.ignorowane = [".crdownload", ".part", ".tmp"]

    def loguj(self, wiadomosc):
        czas = time.strftime("%H:%M:%S")
        wspoldzielone["logi"].appendleft(f"[{czas}] {wiadomosc}")

    def on_created(self, event):
        if not event.is_directory:
            self.przetworz_plik(Path(event.src_path))

    def on_moved(self, event):
        if not event.is_directory:
            self.przetworz_plik(Path(event.dest_path))

    def przetworz_plik(self, sciezka_pliku):
        rozszerzenie = sciezka_pliku.suffix.lower()
        if rozszerzenie in self.ignorowane:
            return 

        time.sleep(1) # Czekamy aż przeglądarka zwolni plik

        znaleziono = False
        for kategoria, rozsz_list in self.kategorie.items():
            if rozszerzenie in rozsz_list:
                self.przenies_plik(sciezka_pliku, kategoria)
                znaleziono = True
                break

        if not znaleziono and rozszerzenie:
            self.przenies_plik(sciezka_pliku, "Inne")

    def przenies_plik(self, sciezka_pliku, kategoria):
        folder_docelowy = self.folder / kategoria
        folder_docelowy.mkdir(exist_ok=True)
        docelowa_sciezka = folder_docelowy / sciezka_pliku.name

        try:
            if docelowa_sciezka.exists():
                baza = sciezka_pliku.stem
                rozsz = sciezka_pliku.suffix
                licznik = 1
                while docelowa_sciezka.exists():
                    docelowa_sciezka = folder_docelowy / f"{baza}_({licznik}){rozsz}"
                    licznik += 1

            shutil.move(str(sciezka_pliku), str(docelowa_sciezka))
            self.loguj(f"✅ Przeniesiono: {sciezka_pliku.name} ➡️ {kategoria}")
        except Exception:
            self.loguj(f"❌ Błąd (zablokowany plik): {sciezka_pliku.name}")

# --- 4. FUNKCJE KONTROLNE ---
def start_monitorowania(folder, kategorie):
    if not os.path.exists(folder):
        st.error("Podana ścieżka nie istnieje!")
        return

    if wspoldzielone["observer"] is None:
        handler = OrganizatorHandler(folder, kategorie)
        observer = Observer()
        observer.schedule(handler, folder, recursive=False)
        observer.start()
        wspoldzielone["observer"] = observer
        czas = time.strftime("%H:%M:%S")
        wspoldzielone["logi"].appendleft(f"[{czas}] 🚀 Uruchomiono monitorowanie.")

def stop_monitorowania():
    if wspoldzielone["observer"] is not None:
        wspoldzielone["observer"].stop()
        wspoldzielone["observer"].join()
        wspoldzielone["observer"] = None
        czas = time.strftime("%H:%M:%S")
        wspoldzielone["logi"].appendleft(f"[{czas}] 🛑 Zatrzymano monitorowanie.")

# --- 5. INTERFEJS STREAMLIT ---
st.set_page_config(page_title="Organizator Plików", page_icon="📂", layout="centered")

st.title("📂 Automatyczny Organizator")
st.write("Skonfiguruj folder i pozwól aplikacji automatycznie sortować pliki pobierane z internetu.")

# Wczytanie ustawień
ustawienia = wczytaj_ustawienia()

# Wprowadzanie ścieżki (w Streamlit musimy wpisać/wkleić tekst)
nowy_folder = st.text_input("Ścieżka do folderu Pobrane:", value=ustawienia.get("monitorowany_folder", ""))
if nowy_folder != ustawienia["monitorowany_folder"]:
    ustawienia["monitorowany_folder"] = nowy_folder
    zapisz_ustawienia(ustawienia)

st.divider()

# Przyciski sterujące
col1, col2, col3 = st.columns(3)
czy_dziala = wspoldzielone["observer"] is not None

with col1:
    if st.button("▶️ Start", disabled=czy_dziala, use_container_width=True):
        start_monitorowania(ustawienia["monitorowany_folder"], ustawienia["kategorie"])
        st.rerun()

with col2:
    if st.button("⏹️ Stop", disabled=not czy_dziala, use_container_width=True):
        stop_monitorowania()
        st.rerun()

with col3:
    # Używamy przycisku do odświeżenia strony, aby zobaczyć nowe logi
    if st.button("🔄 Odśwież logi", use_container_width=True):
        st.rerun()

# Status
if czy_dziala:
    st.success("Status: Działa w tle (Monitorowanie aktywne)")
else:
    st.info("Status: Oczekuje (Zatrzymany)")

st.divider()

# Wyświetlanie logów
st.subheader("Dziennik zdarzeń")
logi_tekst = "\n".join(wspoldzielone["logi"]) if wspoldzielone["logi"] else "Brak zdarzeń. Uruchom program i pobierz jakiś plik."
st.text_area("Ostatnie akcje", value=logi_tekst, height=250, disabled=True)