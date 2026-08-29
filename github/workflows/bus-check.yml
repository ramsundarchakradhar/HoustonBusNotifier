#!/usr/bin/env python3
"""
Runs ONCE per invocation (GitHub Actions calls this every 5 min).
Checks METRO's live feed for Route 212 at Stop 663. If the next bus is
within NOTIFY_MINUTES, sends a text via email-to-SMS gateway.

Uses state.json (committed back to the repo) to avoid texting you twice
about the same bus.
"""

import os
import json
import time
from datetime import datetime
from zoneinfo import ZoneInfo

import requests
from google.transit import gtfs_realtime_pb2

# ---------------- CONFIG ----------------
ROUTE_ID = "212"
STOP_ID = "663"
NOTIFY_MINUTES = 8

WINDOW_START = "16:00"   # 4:00 PM Central
WINDOW_END = "17:00"     # 5:00 PM Central
LOCAL_TZ = ZoneInfo("America/Chicago")

FEED_URL = "https://api.ridemetro.org/GtfsRealtime/TripUpdates"

STATE_FILE = os.path.join(os.path.dirname(__file__), "state.json")
# ------------------------------------------

# Pulled from GitHub Secrets (set these in repo Settings > Secrets > Actions)
METRO_API_KEY = os.environ["METRO_API_KEY"]
NTFY_TOPIC = os.environ["NTFY_TOPIC"]
NTFY_URL = f"https://ntfy.sh/{NTFY_TOPIC}"


def in_notify_window():
    now = datetime.now(LOCAL_TZ)
    start_h, start_m = map(int, WINDOW_START.split(":"))
    end_h, end_m = map(int, WINDOW_END.split(":"))
    start = now.replace(hour=start_h, minute=start_m, second=0, microsecond=0)
    end = now.replace(hour=end_h, minute=end_m, second=0, microsecond=0)
    return start <= now <= end and now.weekday() < 5  # Mon-Fri


def load_state():
    if os.path.exists(STATE_FILE):
        with open(STATE_FILE) as f:
            data = json.load(f)
        # Reset notified list if it's a new day
        if data.get("date") != datetime.now(LOCAL_TZ).strftime("%Y-%m-%d"):
            return {"date": datetime.now(LOCAL_TZ).strftime("%Y-%m-%d"), "notified_trip_ids": []}
        return data
    return {"date": datetime.now(LOCAL_TZ).strftime("%Y-%m-%d"), "notified_trip_ids": []}


def save_state(state):
    with open(STATE_FILE, "w") as f:
        json.dump(state, f)


def send_text(minutes_out, scheduled_time_str):
    body = (
        f"Route {ROUTE_ID} arriving in ~{minutes_out} min (around {scheduled_time_str}). "
        f"Time to head out!"
    )
    resp = requests.post(
        NTFY_URL,
        data=body.encode("utf-8"),
        headers={
            "Title": "Bus leaving soon",
            "Priority": "high",
            "Tags": "bus",
        },
        timeout=15,
    )
    resp.raise_for_status()
    print(f"Notification sent: {minutes_out} min out")


def main():
    if not in_notify_window():
        print(f"Outside notify window ({WINDOW_START}-{WINDOW_END} Central, weekdays). Skipping.")
        return

    state = load_state()

    headers = {"Ocp-Apim-Subscription-Key": METRO_API_KEY}
    resp = requests.get(FEED_URL, headers=headers, timeout=15)
    resp.raise_for_status()

    feed = gtfs_realtime_pb2.FeedMessage()
    feed.ParseFromString(resp.content)

    now = time.time()
    changed = False

    for entity in feed.entity:
        if not entity.HasField("trip_update"):
            continue
        trip_update = entity.trip_update
        if trip_update.trip.route_id != ROUTE_ID:
            continue

        for stu in trip_update.stop_time_update:
            if stu.stop_id != STOP_ID:
                continue

            arrival_time = None
            if stu.HasField("arrival") and stu.arrival.time:
                arrival_time = stu.arrival.time
            elif stu.HasField("departure") and stu.departure.time:
                arrival_time = stu.departure.time
            if not arrival_time:
                continue

            minutes_out = round((arrival_time - now) / 60)
            trip_id = trip_update.trip.trip_id
            scheduled_time_str = datetime.fromtimestamp(arrival_time, LOCAL_TZ).strftime("%I:%M %p")

            print(f"Trip {trip_id}: ETA {minutes_out} min ({scheduled_time_str})")

            if 0 <= minutes_out <= NOTIFY_MINUTES and trip_id not in state["notified_trip_ids"]:
                send_text(minutes_out, scheduled_time_str)
                state["notified_trip_ids"].append(trip_id)
                changed = True

    if changed:
        save_state(state)


if __name__ == "__main__":
    main()
