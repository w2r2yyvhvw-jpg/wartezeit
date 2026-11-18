#!/usr/bin/env python3
"""
Berlin-Terminwächter: meldet sich, sobald beim Service-Portal Berlin freie Termine
für "Umschreibung eines zugelassenen Fahrzeuges mit Berliner Kennzeichen auf einen
anderen Halter" auftauchen.

Er bucht NICHTS selbst, er schickt dir nur eine Push-Nachricht mit dem Link.

Nur Standardbibliothek, Python 3.8+.

Start:
    python3 berlin_termin_watcher.py --ntfy-topic MEIN-GEHEIMES-TOPIC

Push aufs Handy: ntfy-App installieren (iOS/Android), das Topic abonnieren.
"""
import argparse
import random
import re
import sys
import time
import urllib.parse
import urllib.request
from datetime import datetime

# Dienstleistung 120906 = Umschreibung auf anderen Halter (Berliner Kennzeichen)
SERVICE_ID = "120906"
# Standorte: 121362 = Kfz-Zulassung Lichtenberg, 121364 = Kfz-Zulassung Friedrichshain-Kreuzberg
LOCATIONS = "121362,121364"

BASE = "https://service.berlin.de"
CAL_URL = (
    f"{BASE}/terminvereinbarung/termin/tag.php"
    f"?termin=1&anliegen[]={SERVICE_ID}&dienstleisterlist={LOCATIONS}"
    f"&herkunft=http%3A%2F%2Fservice.berlin.de%2Fdienstleistung%2F{SERVICE_ID}%2F"
)
HEADERS = {
    "User-Agent": "Mozilla/5.0 (X11; Linux x86_64) private-appointment-watcher/1.0",
    "Accept-Language": "de-DE,de;q=0.9",
}


def fetch(url: str) -> str:
    req = urllib.request.Request(url, headers=HEADERS)
    with urllib.request.urlopen(req, timeout=30) as r:
        return r.read().decode("utf-8", errors="replace")


def bookable_days(html: str):
    """Liefert Links auf buchbare Tage. Im Kalender haben freie Tage die Klasse 'buchbar'."""
    found = []
    for m in re.finditer(r'<td[^>]*class="[^"]*\bbuchbar\b[^"]*"[^>]*>(.*?)</td>', html, re.S):
        link = re.search(r'href="([^"]+)"', m.group(1))
        if link:
            found.append(urllib.parse.urljoin(BASE, link.group(1).replace("&amp;", "&")))
    return found


def notify(topic: str, message: str, url: str):
    print(f"[{datetime.now():%H:%M:%S}] ALARM: {message}")
    if not topic:
        return
    req = urllib.request.Request(
        f"https://ntfy.sh/{topic}",
        data=message.encode("utf-8"),
        headers={"Title": "Berlin Kfz-Termin frei!", "Priority": "urgent",
                 "Tags": "car,tada", "Click": url},
    )
    try:
        urllib.request.urlopen(req, timeout=15)
    except Exception as e:  # Benachrichtigung darf den Loop nicht killen
        print(f"ntfy fehlgeschlagen: {e}", file=sys.stderr)


def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("--ntfy-topic", default="", help="ntfy.sh Topic für Push-Nachrichten")
    ap.add_argument("--interval", type=int, default=180, help="Sekunden zwischen Abfragen (min. 120 empfohlen)")
    ap.add_argument("--once", action="store_true", help="Nur eine Abfrage, dann beenden (für GitHub Actions/Cron)")
    args = ap.parse_args()
    interval = max(args.interval, 60)

    print(f"Überwache: {CAL_URL}\nIntervall ~{interval}s. Strg+C zum Beenden.")
    last_seen = set()
    errors = 0
    while True:
        try:
            html = fetch(CAL_URL)
            errors = 0
            days = set(bookable_days(html))
            new = days - last_seen
            if new:
                notify(args.ntfy_topic,
                       f"{len(days)} freie(r) Tag(e) gefunden, {len(new)} neu. Schnell buchen!",
                       CAL_URL)
            else:
                state = "keine freien Termine" if not days else f"{len(days)} Tag(e) frei (schon gemeldet)"
                print(f"[{datetime.now():%H:%M:%S}] {state}")
            last_seen = days
        except Exception as e:
            errors += 1
            print(f"[{datetime.now():%H:%M:%S}] Fehler: {e}", file=sys.stderr)
            if errors == 10:
                notify(args.ntfy_topic, "Watcher hat 10 Fehler in Folge, bitte prüfen.", CAL_URL)
        if args.once:
            break
        # Jitter, damit es nicht wie ein starrer Bot-Takt aussieht, und Backoff bei Fehlern
        time.sleep(interval * (1 + min(errors, 5)) + random.uniform(0, interval * 0.3))


if __name__ == "__main__":
    try:
        main()
    except KeyboardInterrupt:
        pass
