# main.py

import os
import openai
import json
import gspread
from datetime import datetime
from pytz import timezone
from oauth2client.service_account import ServiceAccountCredentials

# Time check: run only between 8 AM and 11 PM EST
def within_schedule():
    est = timezone('US/Eastern')
    now = datetime.now(est)
    return 8 <= now.hour <= 23

def get_openai_recommendations():
    openai.api_key = os.environ["OPENAI_API_KEY"]

    prompt = (
        "You are LineWolf, a sharp sports bettor and domain scout.\n"
        "- Recommend high-value sports bets for the next 24 hours\n"
        "- Also include up to 3 valuable domain names currently trending\n"
        "Format:\n"
        "[type:bet|domain] [tag:daily|fade|line|ai|gem|flips|bank]\n"
        "Content: Your alert text goes here"
    )

    response = openai.ChatCompletion.create(
        model="gpt-4",
        messages=[{"role": "user", "content": prompt}]
    )

    return response['choices'][0]['message']['content']

def parse_alerts(text):
    lines = text.strip().split("\n")
    entries = []

    for line in lines:
        if not line.startswith("type:"): continue
        try:
            parts = line.split("]")
            type_part = parts[0].replace("[type:", "")
            tag_part = parts[1].replace(" [tag:", "").replace("]", "")
            content = "]".join(parts[2:]).replace("Content:", "").strip()
            entries.append((type_part, tag_part, content))
        except:
            continue

    return entries

def push_to_google_sheets(entries):
    sheet_id = os.environ["GOOGLE_SHEET_ID"]

    creds_dict = json.loads(os.environ["SERVICE_ACCOUNT_JSON"])
    scope = ["https://spreadsheets.google.com/feeds", "https://www.googleapis.com/auth/drive"]
    creds = ServiceAccountCredentials.from_json_keyfile_dict(creds_dict, scope)
    client = gspread.authorize(creds)

    sheet = client.open_by_key(sheet_id)
    alert_tab = sheet.worksheet("Alerts")

    for _, tag, content in entries:
        timestamp = datetime.now().isoformat()
        alert_tab.append_row([timestamp, content, tag, ""])

if __name__ == "__main__":
    if not within_schedule():
        print("⏱️ Outside active hours. Skipping.")
        exit(0)

    print("⚙️ Polling OpenAI for insights...")
    raw_output = get_openai_recommendations()
    alerts = parse_alerts(raw_output)

    if alerts:
        print(f"📡 Writing {len(alerts)} alerts to Sheet...")
        push_to_google_sheets(alerts)
        print("✅ Done.")
    else:
        print("⚠️ No alerts parsed.")
