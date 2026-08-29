# Houston METRO Bus Text Notifier

Texts you when Route 212 is ~8 minutes from Stop 663, weekdays 4:00-5:00 PM Central.
Runs entirely on GitHub — nothing to keep running on your own computer.

## Setup

### 1. Create the repo
- Create a new **private** GitHub repo (e.g. `houston-bus-notifier`)
- Upload these three files, keeping the folder structure:
  - `bus_check.py`
  - `.github/workflows/bus-check.yml`
  - `state.json`

### 2. Get a METRO API key
- Go to https://api-portal.ridemetro.org/, sign up, subscribe to the GTFS Realtime API
- Copy your subscription key

### 3. Set up ntfy push notifications
- Install the **ntfy** app from the App Store (iPhone) or Play Store (Android)
- In the app, tap **+** and subscribe to a topic name you make up — keep it hard to guess, e.g. `ram-212-bus-x7k2` (anyone who knows your topic name can push to it)
- Allow notifications when prompted
- Test it: visit `https://ntfy.sh/your-topic-name` in a browser, or run `curl -d "test" ntfy.sh/your-topic-name` — you should get a push within seconds

### 4. Add secrets to the repo
In your repo: **Settings > Secrets and variables > Actions > New repository secret**. Add both:

| Secret name | Value |
|---|---|
| `METRO_API_KEY` | your METRO subscription key |
| `NTFY_TOPIC` | your topic name, e.g. `ram-212-bus-x7k2` |

### 5. Test it
- Go to the **Actions** tab in your repo > "Bus Check" workflow > **Run workflow** (manual trigger button)
- It'll only actually text you if it's currently within the 4-5 PM Central window AND a bus is within 8 min — so to fully test, either run it during that window, or temporarily edit `WINDOW_START`/`WINDOW_END` in `bus_check.py` to match right now
- Check the run's logs — it prints every ETA it sees, even if it doesn't text you, so you can confirm it's finding your route/stop correctly

### 6. Let it run
Once secrets are set, it runs automatically every 5 minutes, Mon-Fri, no further action needed.

## Notes
- ntfy is free and needs no account — just keep your topic name private since it's the only thing gating who can push to it.
- GitHub's free scheduled runs aren't second-precise — expect the "8 minutes" trigger to sometimes fire at 6-9 min out depending on when in the 5-min cycle it lands.
