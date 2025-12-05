import requests
import sys
import time
import json
import instaloader
import os
from rich.console import Console
from rich.panel import Panel
from rich.text import Text
from rich import box
from itertools import cycle

OUTPUT_FILE = 'username.txt'
STATE_FILE = 'state.json'
SAVE_EVERY = 50
REQUEST_DELAY = 0.7

console = Console()

def build_headers(sessionid, csrftoken='gLlFX76z8qqwDgmh8ZIp3uFhAeX4zKdO'):
    return {
        'Accept': '*/*',
        'Accept-Encoding': 'gzip, deflate, br',
        'Accept-Language': 'en-US,en;q=0.9',
        'Cookie': f'mid=Y3bGYwALAAHNwaKANMB8QCsRu7VA; ig_did=092BD3C7-0BB2-414B-9F43-3170EAED8778; ig_nrcb=1; shbid=1685054; shbts=1675191368.6684434090; rur=CLN; ig_direct_region_hint=ATN; csrftoken={csrftoken}; ds_user_id=921803283; sessionid={sessionid}',
        'Sec-Ch-Prefers-Color-Scheme': 'dark',
        'Sec-Ch-Ua': '"Not.A/Brand";v="8", "Chromium";v="114", "Microsoft Edge";v="114"',
        'Sec-Ch-Ua-Full-Version-List': '"Not.A/Brand";v="8.0.0.0", "Chromium";v="114.0.5735.201", "Microsoft Edge";v="114.0.1823.67"',
        'Sec-Ch-Ua-Mobile': '?0',
        'Sec-Ch-Ua-Platform': '"Windows"',
        'Sec-Ch-Ua-Platform-Version': '"15.0.0"',
        'Sec-Fetch-Dest': 'empty',
        'Sec-Fetch-Mode': 'cors',
        'Sec-Fetch-Site': 'same-origin',
        'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/114.0.0.0 Safari/537.36 Edg/114.0.1823.67',
        'X-Asbd-Id': '129477',
        'X-Csrftoken': csrftoken,
        'X-Ig-App-Id': '936619743392459',
        'X-Ig-Www-Claim': 'hmac.AR0g7ECdkTdrXy37TE9AoSnMndccWbB1cqrccYOZSLfcb8sd',
        'X-Requested-With': 'XMLHttpRequest',
    }

def save_state(state):
    with open(STATE_FILE, 'w', encoding='utf-8') as sf:
        json.dump(state, sf, ensure_ascii=False, indent=2)

def load_state():
    if os.path.exists(STATE_FILE):
        try:
            with open(STATE_FILE, 'r', encoding='utf-8') as sf:
                return json.load(sf)
        except Exception:
            return None
    return None

def append_username(u):
    if u.startswith('@'):
        u = u[1:]
    with open(OUTPUT_FILE, 'a', encoding='utf-8') as f:
        f.write(u + '\n')

s_input = console.input("Enter one or more sessionid (comma separated) or filename: ").strip()
if os.path.exists(s_input) and os.path.isfile(s_input):
    with open(s_input, 'r', encoding='utf-8') as f:
        sessionids = [line.strip() for line in f if line.strip()]

def save_state(state):
    with open(STATE_FILE, 'w', encoding='utf-8') as sf:
        json.dump(state, sf, ensure_ascii=False, indent=2)

def load_state():
    if os.path.exists(STATE_FILE):
        try:
            with open(STATE_FILE, 'r', encoding='utf-8') as sf:
                return json.load(sf)
        except Exception:
            return None
    return None

def append_username(u):
    if u.startswith('@'):
        u = u[1:]
    with open(OUTPUT_FILE, 'a', encoding='utf-8') as f:
        f.write(u + '\n')
import ethan

logo_lines = [
f'''
'''
]
logo_text = Text()
for line in logo_lines:
    logo_text.append(line + "\n", style="bold cyan")
logo_text.append("\nWAYNE tool", style="bold white on blue")
logo_text.append("    ")
logo_text.append("@sshh9\n\n", style="bold magenta")
console.print(Panel(logo_text, title="Welcome", subtitle="ETHAN TOOL", expand=False, box=box.ROUNDED))
print('')
s_input = console.input("Enter one or more sessionid (comma separated) or filename: ").strip()
if os.path.exists(s_input) and os.path.isfile(s_input):
    with open(s_input, 'r', encoding='utf-8') as f:
        sessionids = [line.strip() for line in f if line.strip()]
else:
    sessionids = [s.strip() for s in s_input.split(',') if s.strip()]

if not sessionids:
    console.print("At least one sessionid is required.", style="bold red")
    sys.exit(1)

targets_input = console.input("Enter one or more target usernames (comma separated) or filename: ").strip()
if os.path.exists(targets_input) and os.path.isfile(targets_input):
    with open(targets_input, 'r', encoding='utf-8') as f:
        targets = [line.strip().lstrip("@") for line in f if line.strip()]
else:
    targets = [t.strip().lstrip("@") for t in targets_input.split(',') if t.strip()]

if not targets:
    console.print("At least one target username is required.", style="bold red")
    sys.exit(1)

L = instaloader.Instaloader()
resolved_targets = []
for t in targets:
    try:
        profile = instaloader.Profile.from_username(L.context, t)
        resolved_targets.append({'username': t, 'userid': str(profile.userid)})
        console.print(f"Resolved @{t} -> id {profile.userid}", style="green")
    except Exception as e:
        console.print(f"Error resolving @{t}: {e}", style="yellow")

if not resolved_targets:
    console.print("No valid targets resolved. Exiting.", style="bold red")
    sys.exit(1)

state = load_state()
if state and state.get('targets') == [rt['username'] for rt in resolved_targets]:
    console.print("Loaded previous state for same targets. Resuming...", style="cyan")
    targets_state = state.get('targets_state', {})
    target_index = state.get('target_index', 0)
    session_index = state.get('session_index', 0)
else:
    targets_state = {}
    for rt in resolved_targets:
        targets_state[rt['username']] = {'m_id': None, 'seen_users': [], 'p1': 0}
    target_index = 0
    session_index = 0

total_sessions = len(sessionids)
color_palette = ["bold cyan", "bold green", "bold magenta", "bold yellow", "bold blue", "bold red"]
color_cycle = cycle(color_palette)
target_colors = {}
for rt in resolved_targets:
    target_colors[rt['username']] = next(color_cycle)

def safe_get(url, session_index_local):
    attempts = 0
    while attempts < total_sessions:
        sess = sessionids[session_index_local % total_sessions]
        headers = build_headers(sess)
        try:
            r = requests.get(url, headers=headers, timeout=15)
        except requests.RequestException as e:
            console.print(f'Connection error with session index {session_index_local}: {e}. Switching to next session.', style="yellow")
            session_index_local = (session_index_local + 1) % total_sessions
            attempts += 1
            time.sleep(1)
            continue
        if r.status_code == 401:
            console.print(f'401 with session index {session_index_local}. Switching to next session.', style="yellow")
            session_index_local = (session_index_local + 1) % total_sessions
            attempts += 1
            time.sleep(1)
            continue
        if '{"message":"","spam":true,"status":"fail"}' in r.text:
            console.print('Request blocked (spam fail) — stopping.', style="red")
            return None, session_index_local, 'spam'
        if 'challenge' in r.text or r.status_code in (429, 403):
            console.print(f'Received challenge/429/403 or unexpected response (code={r.status_code}). Switching session.', style="yellow")
            session_index_local = (session_index_local + 1) % total_sessions
            attempts += 1
            time.sleep(2)
            continue
        return r, session_index_local, None
    console.print('All sessionids exhausted or failed.', style="red")
    return None, session_index_local, 'no_sessions'

while target_index < len(resolved_targets):
    current_target = resolved_targets[target_index]
    tr = current_target['username']
    target_id = current_target['userid']
    color = target_colors.get(tr, "bold white")
    console.print(f"\n=== Processing target @{tr} ({target_index + 1}/{len(resolved_targets)}) ===", style=color)
    tstate = targets_state.get(tr, {'m_id': None, 'seen_users': [], 'p1': 0})
    m_id = tstate.get('m_id')
    seen_users = set(tstate.get('seen_users', []))
    p1 = tstate.get('p1', 0)
    base_url = f'https://i.instagram.com/api/v1/friendships/{target_id}/following/?count=200'
    first_url = base_url if not m_id else base_url + (f'&max_id={m_id}' if m_id else '')
    current_url = first_url
    while True:
        r, session_index, error = safe_get(current_url, session_index)
        if error == 'spam' or error == 'no_sessions':
            console.print('Execution stopped due to an unrecoverable condition.', style="red")
            targets_state[tr] = {'m_id': m_id, 'seen_users': list(seen_users), 'p1': p1}
            save_state({'targets': [rt['username'] for rt in resolved_targets], 'targets_state': targets_state, 'target_index': target_index, 'session_index': session_index})
            sys.exit(1)
        if r is None:
            console.print('No valid response received — stopping.', style="red")
            targets_state[tr] = {'m_id': m_id, 'seen_users': list(seen_users), 'p1': p1}
            save_state({'targets': [rt['username'] for rt in resolved_targets], 'targets_state': targets_state, 'target_index': target_index, 'session_index': session_index})
            sys.exit(1)
        console.print(f"HTTP {r.status_code}", style="dim")
        try:
            data = r.json()
        except Exception as e:
            console.print(f"Failed to parse JSON: {e}", style="yellow")
            session_index = (session_index + 1) % total_sessions
            time.sleep(1)
            continue
        users = data.get('users', [])
        if not users:
            console.print("No users on this page.", style="dim")
        for u in users:
            userL = u.get('username') or u.get('user') or u.get('pk') or ''
            if not userL:
                continue
            if userL in seen_users:
                continue
            seen_users.add(userL)
            p1 += 1
            console.print(f'{p1} <> @{userL} (from @{tr})', style=color)
            append_username(f'@{userL}')
            if p1 % SAVE_EVERY == 0:
                targets_state[tr] = {'m_id': data.get('next_max_id'), 'seen_users': list(seen_users), 'p1': p1}
                save_state({'targets': [rt['username'] for rt in resolved_targets], 'targets_state': targets_state, 'target_index': target_index, 'session_index': session_index})
                console.print(f'State saved at {p1} users for @{tr}.', style="cyan")
        next_m = data.get('next_max_id')
        if not next_m:
            console.print(f"Reached last page for @{tr}. Finished target.", style=color)
            targets_state[tr] = {'m_id': None, 'seen_users': list(seen_users), 'p1': p1}
            save_state({'targets': [rt['username'] for rt in resolved_targets], 'targets_state': targets_state, 'target_index': target_index, 'session_index': session_index})
            break
        m_id = next_m
        current_url = base_url + f'&max_id={m_id}'
        time.sleep(REQUEST_DELAY)
    console.print(f"Done with target @{tr}. Collected {p1} unique users for this target.", style=color)
    target_colors[tr] = next(color_cycle)
    targets_state[tr] = {'m_id': None, 'seen_users': list(seen_users), 'p1': p1}
    target_index += 1
    if target_index < len(resolved_targets):
        save_state({'targets': [rt['username'] for rt in resolved_targets], 'targets_state': targets_state, 'target_index': target_index, 'session_index': session_index})
        console.print(f"Moving to next target (index {target_index + 1}). State saved.", style="magenta")

if os.path.exists(STATE_FILE):
    try:
        os.remove(STATE_FILE)
    except Exception:
        pass

console.print('All targets processed. Done.', style="bold green")
total_saved = 0
for trname, st in targets_state.items():
    total_saved += st.get('p1', 0)
console.print(f'Total unique users saved across all targets: {total_saved}', style="bold cyan")
if total_saved > 0:
    console.print(f'Results in: {OUTPUT_FILE}', style="bold white on blue")
else:
    console.print('No usernames were fetched.', style="bold yellow")
