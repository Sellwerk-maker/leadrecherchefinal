import streamlit as st
import pandas as pd
from serpapi import GoogleSearch
import requests
from bs4 import BeautifulSoup

# --- Einstellungen ---
st.set_page_config(page_title="Lead-Recherche-Agent", layout="wide")
st.title("🔍 Lead-Recherche-Agent für KMU")

# --- Eingabemaske ---
st.sidebar.header("Suchparameter")
branche = st.sidebar.text_input("Branche", "Bäckerei")
ort = st.sidebar.text_input("Stadt", "Düsseldorf")
api_key = st.sidebar.text_input("SerpAPI Key", type="password")

# --- Google Maps Suche ---
def suche_maps_firmen(branche, ort, api_key):
    params = {
        "engine": "google_maps",
        "q": f"{branche} {ort}",
        "type": "search",
        "api_key": api_key
    }
    search = GoogleSearch(params)
    results = search.get_dict()
    return results.get("local_results", [])

# --- SEO-Check ---
def seo_kurzcheck(url):
    try:
        r = requests.get(url, timeout=5)
        soup = BeautifulSoup(r.text, "html.parser")
        return {
            "SSL": url.startswith("https"),
            "Mobilfreundlich": "viewport" in r.text,
            "Meta Title": bool(soup.title and soup.title.text.strip()),
            "H1 vorhanden": bool(soup.find("h1")),
            "Ladezeit (s)": round(r.elapsed.total_seconds(), 2)
        }
    except:
        return {
            "SSL": False,
            "Mobilfreundlich": False,
            "Meta Title": False,
            "H1 vorhanden": False,
            "Ladezeit (s)": "Fehler"
        }

# --- Hauptfunktion ---
if st.button("🔍 Recherche starten"):
    if not api_key:
        st.error("Bitte einen SerpAPI Key eingeben.")
    else:
        with st.spinner("Suche läuft..."):
            firmen = suche_maps_firmen(branche, ort, api_key)
            daten = []

            for firma in firmen:
                website = firma.get("website")
                seo = seo_kurzcheck(website) if website else {}
                daten.append({
                    "Firmenname": firma.get("title"),
                    "Adresse": firma.get("address"),
                    "Telefon": firma.get("phone"),
                    "Website": website,
                    "Bewertung": firma.get("rating"),
                    **seo
                })

            df = pd.DataFrame(daten)
            st.success(f"{len(df)} Ergebnisse gefunden")
            st.dataframe(df, use_container_width=True)

            csv = df.to_csv(index=False).encode('utf-8')
            st.download_button("📥 Ergebnisse als CSV herunterladen", data=csv, file_name='leads.csv', mime='text/csv')
