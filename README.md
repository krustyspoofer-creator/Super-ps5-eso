# Super-ps5-eso
#!/usr/bin/env bash
set -euo pipefail
# ps5_monitor installer (improved "full stack" block)
# - Creates promo_watcher.py (with --once)
# - Creates presence_watch.sh (fixed -z bug, faster scan options)
# - Creates run_all.sh (tmux/nohup)
# - Uses $HOME/ps5_monitor and a .env for credentials
# - Adds logging with timestamps and caches discovered PS5 IP
BASE="$HOME/ps5_monitor"
PY="$BASE/promo_watcher.py"
PRESENCE="$BASE/presence_watch.sh"
RUNNER="$BASE/run_all.sh"
ENVFILE="$BASE/.env"
PROMO_TRIGGER_SCRIPT="$BASE/promo_trigger.sh"
STATE_FILE="$HOME/.ps5_promo_state.json"

mkdir -p "$BASE"
cd "$BASE"

# 0) .env example
cat > "$ENVFILE" <<'ENV'
# PS5 monitor environment file
# Fill these in or export as environment variables before running run_all.sh
# Example:
# TG_BOT_TOKEN="123456:ABC-DEF..."
# TG_CHAT_ID="-987654321"

TG_BOT_TOKEN="<YOUR_TELEGRAM_BOT_TOKEN>"
TG_CHAT_ID="<YOUR_CHAT_ID>"
CHECK_INTERVAL=30       # seconds between presence checks
PROMO_INTERVAL=600      # seconds between promo page checks
ENV
chmod 600 "$ENVFILE"

# 1) PS5 Wi-Fi connect instructions
cat > "$BASE/PS5_WIFI_INSTRUCTIONS.txt" <<'EOF'
PS5 Wi-Fi connect quick steps:
1) On PS5: Settings -> Network -> Set Up Internet Connection -> Set Up via Wi-Fi.
2) Select your SSID (must be same Wi-Fi as this device for auto-detect).
3) Enter the Wi-Fi password (keep it private).
4) Note the PS5 IP in Settings -> Network -> View Connection Status (optional).
If you prefer, set PS5_IP in $HOME/ps5_monitor/.env or export it before running.
EOF

# 2) promo_watcher.py (improved, supports --once)
cat > "$PY" <<'PYEOF'
#!/usr/bin/env python3
# promo_watcher.py
# Monitors specified public pages and sends Telegram alerts when text snapshots change.
import os, time, hashlib, json, argparse, sys
from datetime import datetime
try:
    import requests
    from bs4 import BeautifulSoup
except Exception as e:
    print("Missing dependencies. Install: pip3 install requests beautifulsoup4", file=sys.stderr)
    raise

TG_BOT = os.getenv("TG_BOT_TOKEN") or "<YOUR_TELEGRAM_BOT_TOKEN>"
TG_CHAT = os.getenv("TG_CHAT_ID") or "<YOUR_CHAT_ID>"
INTERVAL = int(os.getenv("PROMO_INTERVAL", os.getenv("CHECK_INTERVAL", "600")))

WATCH = {
    "ps_plus_whats_new": "https://www.playstation.com/en-us/ps-plus/whats-new/",
    "ps_plus": "https://www.playstation.com/en-us/ps-plus/",
    "eso_plus": "https://www.elderscrollsonline.com/en-us/esoplus",
    "eso_store": "https://account.elderscrollsonline.com/en-us/store"
}

STATE_FILE = os.path.expanduser(os.getenv("PS5_PROMO_STATE", "{}").format())

# If no explicit file, fallback to default ~/.ps5_promo_state.json
if not STATE_FILE or STATE_FILE == "{}":
    STATE_FILE = os.path.expanduser("~/.ps5_promo_state.json")

def now_ts():
    return datetime.utcnow().strftime("%Y-%m-%d %H:%M:%S UTC")

def send_telegram(text):
    if "YOUR_TELEGRAM" in TG_BOT or TG_BOT.strip() == "" or TG_CHAT.strip() == "":
        print(f"[tg disabled] {text[:200]}")
        return False
    url = f"https://api.telegram.org/bot{TG_BOT}/sendMessage"
    payload = {"chat_id": TG_CHAT, "text": text[:4000]}
    try:
        r = requests.post(url, json=payload, timeout=10)
        if r.ok:
            return True
        else:
            print("Telegram API error:", r.status_code, r.text)
            return False
    except Exception as e:
        print("Telegram send error:", e)
        return False

def snapshot_text(html):
    soup = BeautifulSoup(html, "html.parser")
    # remove scripts/styles
    for s in soup(["script", "style", "noscript"]):
        s.decompose()
    txt = "\n".join(line.strip() for line in soup.get_text().splitlines() if line.strip())
    return txt[:10000]

def load_state():
    try:
        with open(STATE_FILE, "r") as f:
            return json.load(f)
    except Exception:
        return {}

def save_state(s):
    try:
        with open(STATE_FILE, "w") as f:
            json.dump(s, f)
    except Exception as e:
        print("Could not save state:", e)

def check_once():
    state = load_state()
    changes = []
    for key, url in WATCH.items():
        try:
            r = requests.get(url, timeout=20, headers={"User-Agent":"ps5-monitor/1.0"})
            r.raise_for_status()
            snap = snapshot_text(r.text)
            h = hashlib.sha256(snap.encode("utf-8")).hexdigest()
            if state.get(key) != h:
                excerpt = snap[:800].replace("\n", " ")
                changes.append((key, url, excerpt))
                state[key] = h
        except Exception as e:
            print(f"[{now_ts()}] fetch error {url}: {e}")
    if changes:
        for k, u, ex in changes:
            msg = f"[CHANGE] {k} -> {u}\nExcerpt: {ex}\nTime: {now_ts()}"
            send_telegram(msg)
    save_state(state)
    return len(changes) > 0

def main_loop():
    # initial silent populate
    try:
        check_once()
    except Exception as e:
        print("initial check failed", e)
    while True:
        try:
            changed = check_once()
            if not changed:
                print(now_ts(), "no changes")
        except Exception as e:
            print("check error", e)
        time.sleep(INTERVAL)

if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("--once", action="store_true", help="Run a single check and exit")
    args = parser.parse_args()
    if args.once:
        sys.exit(0 if check_once() else 0)
    else:
        main_loop()
PYEOF
chmod +x "$PY"

# 3) presence_watch.sh (fixed and improved)
cat > "$PRESENCE" <<'SH'
#!/usr/bin/env bash
# presence_watch.sh - detect PS5 on LAN and trigger a promo check (calls promo_watcher.py --once)
set -euo pipefail
# Load env if present
ENVFILE="${HOME:-.}/ps5_monitor/.env"
if [[ -f "$ENVFILE" ]]; then
  # shellcheck disable=SC1090
  source "$ENVFILE"
fi

TG_BOT="${TG_BOT_TOKEN:-<YOUR_TELEGRAM_BOT_TOKEN>}"
TG_CHAT="${TG_CHAT_ID:-<YOUR_CHAT_ID>}"
PS5_IP="${PS5_IP:-}"
CHECK_INTERVAL="${CHECK_INTERVAL:-30}"
BASE_DIR="${HOME:-.}/ps5_monitor"
PROMO_TRIGGER_SCRIPT="${BASE_DIR}/promo_trigger.sh"
PS5_IP_CACHE="${BASE_DIR}/ps5_ip.cache"

log() {
  echo "$(date '+%Y-%m-%d %H:%M:%S') $*"
}

tg_send() {
  local txt="$1"
  if [[ -z "$TG_BOT" || "$TG_BOT" == *"YOUR_TELEGRAM"* ]]; then
    log "[tg disabled] $txt"
    return
  fi
  curl -s -X POST "https://api.telegram.org/bot${TG_BOT}/sendMessage" \
    -H "Content-Type: application/json" \
    -d "{\"chat_id\":\"${TG_CHAT}\",\"text\":\"${txt}\"}" >/dev/null || true
}

get_local_subnet() {
  # prefer ip route method
  IFACE=$(ip route get 1.1.1.1 2>/dev/null | awk '{for(i=1;i<=NF;i++) if($i=="dev"){print $(i+1); exit}}' || true)
  if [[ -z "$IFACE" ]]; then
    IFACE=$(ip -o -f inet addr show | awk '/scope global/ {print $2; exit}' || true)
  fi
  CIDR=$(ip -o -f inet addr show "$IFACE" 2>/dev/null | awk '{print $4; exit}' || true)
  echo "$CIDR"
}

fast_scan_ips() {
  # prefer fping or nmap if available
  local cidr="$1"
  if command -v fping >/dev/null 2>&1; then
    fping -a -g "$cidr" 2>/dev/null || true
  elif command -v nmap >/dev/null 2>&1; then
    nmap -sn "$cidr" -oG - | awk '/Up$/{print $2}'
  else
    # fallback to ping loop (slower)
    local base3 prefix ip
    IFS=/ read -r base3 prefix <<< "$cidr"
    prefix=${prefix:-24}
    base3=$(echo "$base3" | awk -F. '{print $1"."$2"."$3}')
    for i in $(seq 1 254); do
      ip="${base3}.$i"
      if ping -c1 -W1 "$ip" >/dev/null 2>&1; then
        echo "$ip"
      fi
    done
  fi
}

auto_find_ps5() {
  log "[*] Attempting to auto-scan subnet for live hosts..."
  CIDR=$(get_local_subnet)
  if [[ -z "$CIDR" ]]; then
    log "Could not determine subnet. Please set PS5_IP in .env or env var."
    return 1
  fi
  # scan live ips
  mapfile -t IPS < <(fast_scan_ips "$CIDR")
  for ip in "${IPS[@]}"; do
    # try to get mac via arp
    mac=$(arp -n "$ip" 2>/dev/null | awk '/ether/ {print $3; exit}' || true)
    if [[ -n "$mac" ]]; then
      macp=$(echo "$mac" | tr '[:lower:]' '[:upper:]' | sed 's/://g' | cut -c1-6)
      # crude common Sony prefixes (not exhaustive). Accept candidate if matches or present.
      case "$macp" in
        0014A0|001A11|000F66|0022E0|2C3E3A|3C5A37|086B0B|F0D1*)
          echo "$ip"
          return 0
          ;;
      esac
    else
      # still consider the live host if we couldn't read MAC
      echo "$ip"
      return 0
    fi
  done
  return 1
}

# create promo trigger script (one-shot run)
cat > "$PROMO_TRIGGER_SCRIPT" <<'TR'
#!/usr/bin/env bash
# one-shot trigger to run promo watcher once (non-daemon)
ENVFILE="${HOME}/ps5_monitor/.env"
if [[ -f "$ENVFILE" ]]; then
  # shellcheck disable=SC1090
  source "$ENVFILE"
fi
python3 "${HOME}/ps5_monitor/promo_watcher.py" --once 2>/dev/null || true
TR
chmod +x "$PROMO_TRIGGER_SCRIPT"

# Interactive selection / caching logic
if [[ -z "${PS5_IP:-}" ]]; then
  # try cached IP first
  if [[ -f "$PS5_IP_CACHE" ]]; then
    cached=$(cat "$PS5_IP_CACHE")
    read -r -p "Use cached PS5 IP ${cached}? [Y/n] " yn
    yn=${yn:-Y}
    if [[ "$yn" =~ ^[Yy] ]]; then
      PS5_IP="$cached"
    fi
  fi
fi

if [[ -z "${PS5_IP:-}" ]]; then
  log "PS5_IP not set. Attempting auto-detect..."
  CAND=$(auto_find_ps5 2>/dev/null | head -n1 || true)
  if [[ -n "$CAND" ]]; then
    log "Candidate PS5 IP found: $CAND"
    read -r -p "Use this IP? [Y/n] " yn
    yn=${yn:-Y}
    if [[ "$yn" =~ ^[Yy] ]]; then
      PS5_IP="$CAND"
      echo "$PS5_IP" > "$PS5_IP_CACHE"
    else
      read -r -p "Enter PS5 IP manually: " PS5_IP
      echo "$PS5_IP" > "$PS5_IP_CACHE"
    fi
  else
    read -r -p "Could not auto-detect. Enter PS5 IP (e.g.10.0.0.10): " PS5_IP
    echo "$PS5_IP" > "$PS5_IP_CACHE"
  fi
fi

log "[*] Monitoring PS5 at $PS5_IP every ${CHECK_INTERVAL}s"
ONLINE=0
while true; do
  if ping -c1 -W1 "$PS5_IP" >/dev/null 2>&1; then
    if [[ "$ONLINE" -eq 0 ]]; then
      ONLINE=1
      txt="PS5 is ONLINE at $PS5_IP"
      log "$txt"
      tg_send "$txt"
      # trigger promo check immediately in background
      "$PROMO_TRIGGER_SCRIPT" &
      # cache IP proactively
      echo "$PS5_IP" > "$PS5_IP_CACHE" || true
    fi
  else
    if [[ "$ONLINE" -eq 1 ]]; then
      ONLINE=0
      txt="PS5 is OFFLINE at $(date '+%Y-%m-%d %H:%M:%S')"
      log "$txt"
      tg_send "$txt"
    fi
  fi
  sleep "$CHECK_INTERVAL"
done
SH
chmod +x "$PRESENCE"

# 4) runner script (tmux preferable)
cat > "$RUNNER" <<'RUN'
#!/usr/bin/env bash
set -euo pipefail
BASE="$HOME/ps5_monitor"
ENVFILE="$BASE/.env"
if [[ -f "$ENVFILE" ]]; then
  # shellcheck disable=SC1090
  source "$ENVFILE"
fi
cd "$BASE"
PROMO_LOG="$BASE/promo.log"
PRESENCE_LOG="$BASE/presence.log"

# start promo watcher (python) in background with stdout->log and timestamps
start_promo() {
  if command -v stdbuf >/dev/null 2>&1; then
    stdbuf -oL -eL python3 "$BASE/promo_watcher.py" >> "$PROMO_LOG" 2>&1 &
  else
    python3 "$BASE/promo_watcher.py" >> "$PROMO_LOG" 2>&1 &
  fi
}

if command -v tmux >/dev/null 2>&1; then
  tmux new-session -d -s ps5mon "bash -lc 'source \"$ENVFILE\" 2>/dev/null || true; python3 \"$BASE/promo_watcher.py\"'"
  tmux split-window -v -t ps5mon "bash -lc 'source \"$ENVFILE\" 2>/dev/null || true; bash \"$BASE/presence_watch.sh\"'"
  echo "Started in tmux session 'ps5mon'. Attach with: tmux attach -t ps5mon"
else
  echo "tmux not found. Starting with nohup; logs:"
  start_promo
  nohup bash "$BASE/presence_watch.sh" >> "$PRESENCE_LOG" 2>&1 &
  echo "Promo log: $PROMO_LOG"
  echo "Presence log: $PRESENCE_LOG"
fi
RUN
chmod +x "$RUNNER"

# 5) convenience promo trigger (one-shot)
cat > "$PROMO_TRIGGER_SCRIPT" <<'TRIG'
#!/usr/bin/env bash
ENVFILE="${HOME}/ps5_monitor/.env"
if [[ -f "$ENVFILE" ]]; then
  # shellcheck disable=SC1090
  source "$ENVFILE"
fi
python3 "${HOME}/ps5_monitor/promo_watcher.py" --once 2>/dev/null || true
TRIG
chmod +x "$PROMO_TRIGGER_SCRIPT"

# 6) Attempt to install dependencies (best-effort). Do not fail if package manager not present.
echo
echo "Attempting to install system packages & pip deps (best-effort)..."
if command -v pkg >/dev/null 2>&1; then
  pkg install -y python curl iputils-net-tools fping tmux || true
elif command -v apt-get >/dev/null 2>&1; then
  sudo apt-get update -y || true
  sudo apt-get install -y python3 python3-pip curl net-tools iputils-ping fping tmux nmap || true
fi
python3 -m pip install --upgrade pip >/dev/null 2>&1 || true
python3 -m pip install requests beautifulsoup4 >/dev/null 2>&1 || true

# 7) Final notes + quick-edit reminder
cat <<EOF

INSTALL COMPLETE at: $BASE

IMPORTANT next steps:
1) Edit credentials in: $ENVFILE
   - Set TG_BOT_TOKEN and TG_CHAT_ID (or export them before running run_all.sh).
   - Optional: set PS5_IP (to skip auto-detect) or CHECK_INTERVAL / PROMO_INTERVAL.

2) Start everything:
   bash $RUNNER

3) Useful commands:
   - View logs:
       tail -F $BASE/promo.log
       tail -F $BASE/presence.log
   - Run a one-shot promo check:
       bash $BASE/promo_trigger.sh
   - If using tmux:
       tmux attach -t ps5mon

Security & behavior notes:
 - This monitors public web pages and devices on your LAN; it does not circumvent any payment or entitlement systems.
 - Tokens are read from $ENVFILE; keep that file private (chmod 600 done by installer).
 - If auto-detection fails, set PS5_IP manually in the .env file or when prompted.

EOF

echo "Done. Run: bash $RUNNER"
