#!/usr/bin/env python3
# -*- coding: utf-8 -*-

import requests
import sys
import time
import json
import instaloader
import os

from asmix import Instagram

OUTPUT_FILE = 'username.txt'
STATE_FILE = 'state.json'
SAVE_EVERY = 20
REQUEST_DELAY = 0.7

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
    with open(OUTPUT_FILE, 'a', encoding='utf-8') as f:
        f.write(u + '\n')

s_input = input('Enter one or more sessionid (comma separated) ').strip()
if os.path.exists(s_input) and os.path.isfile(s_input):
    with open(s_input, 'r', encoding='utf-8') as f:
        sessionids = [line.strip() for line in f if line.strip()]
else:
    sessionids = [s.strip() for s in s_input.split(',') if s.strip()]

if not sessionids:
    print("At least one sessionid is required.")
    sys.exit(1)

tr = input('Enter target username: ').strip()
if not tr:
    print("No target provided.")
    sys.exit(1)

L = instaloader.Instaloader()
try:
    profile = instaloader.Profile.from_username(L.context, tr)
    target_id = str(profile.userid)
    print("target id:", target_id)
except Exception as e:
    print("Error fetching profile with Instaloader:", e)
    sys.exit(1)

state = load_state()
if state:
    if state.get('target') == tr:
        print("Loaded previous state. Resuming...")
        m_id = state.get('m_id')
        seen_users = set(state.get('seen_users', []))
        p1 = state.get('p1', 0)
        session_index = state.get('session_index', 0)
    else:
        print("Found state file for a different target — ignoring it.")
        state = None
        m_id = None
        seen_users = set()
        p1 = 0
        session_index = 0
else:
    m_id = None
    seen_users = set()
    p1 = 0
    session_index = 0

base_url = f'https://i.instagram.com/api/v1/friendships/{target_id}/following/?count=200'

def safe_get(url, session_index_local):
    attempts = 0
    total_sessions = len(sessionids)
    while attempts < total_sessions:
        sess = sessionids[session_index_local % total_sessions]
        headers = build_headers(sess)
        try:
            r = requests.get(url, headers=headers, timeout=15)
        except requests.RequestException as e:
            print(f'Connection error with session index {session_index_local}: {e}. Switching to next session.')
            session_index_local = (session_index_local + 1) % total_sessions
            attempts += 1
            time.sleep(1)
            continue

        if r.status_code == 401:
            print(f'401 with session index {session_index_local}. Switching to next session.')
            session_index_local = (session_index_local + 1) % total_sessions
            attempts += 1
            time.sleep(1)
            continue

        if '{"message":"","spam":true,"status":"fail"}' in r.text:
            print('Request blocked (spam fail) — stopping.')
            return None, session_index_local, 'spam'

        if 'challenge' in r.text or r.status_code in (429, 403):
            print(f'Received challenge/429/403 or unexpected response (code={r.status_code}). Switching session.')
            session_index_local = (session_index_local + 1) % total_sessions
            attempts += 1
            time.sleep(2)
            continue

        return r, session_index_local, None

    print('All sessionids exhausted or failed.')
    return None, session_index_local, 'no_sessions'

first_url = base_url if not m_id else base_url + (f'&max_id={m_id}' if m_id else '')
current_url = first_url

while True:
    r, session_index, error = safe_get(current_url, session_index)
    if error == 'spam' or error == 'no_sessions':
        print('Execution stopped due to an unrecoverable condition.')
        break
    if r is None:
        print('No valid response received — stopping.')
        break

    print("HTTP", r.status_code)
    try:
        data = r.json()
    except Exception as e:
        print("Failed to parse JSON:", e)
        session_index = (session_index + 1) % len(sessionids)
        time.sleep(1)
        continue

    users = data.get('users', [])
    if not users:
        print("No more users on this page.")
    for u in users:
        userL = u.get('username') or u.get('user') or u.get('pk') or ''
        if not userL:
            continue
        if userL in seen_users:
            continue
        seen_users.add(userL)
        p1 += 1
        print(f'{p1} <> {userL}')
        append_username(userL)

        if p1 % SAVE_EVERY == 0:
            state_to_save = {
                'target': tr,
                'm_id': data.get('next_max_id'),
                'seen_users': list(seen_users),
                'p1': p1,
                'session_index': session_index
            }
            save_state(state_to_save)
            print(f'State saved at {p1} users.')

    next_m = data.get('next_max_id')
    if not next_m:
        print("Reached last page. Finished.")
        if os.path.exists(STATE_FILE):
            try:
                os.remove(STATE_FILE)
            except:
                pass
        break

    m_id = next_m
    current_url = base_url + f'&max_id={m_id}'
    time.sleep(REQUEST_DELAY)

print('Done. Total unique users saved:', p1)
if p1 > 0:
    print('Results in:', OUTPUT_FILE)
else:
    print('No usernames were fetched.')
