# API server, view links and SKAI

## Running the server

`server/` is a small FastAPI app serving a read-only JSON API over `data/`. It does not serve the website. It needs Python 3.10 or newer.

```bash
python3 -m venv server/.venv
server/.venv/bin/pip install -r server/requirements.txt
server/.venv/bin/uvicorn app:app --app-dir server      # http://localhost:8000/thresholds
```

With Docker (the image holds the server and only the data files it reads):

```bash
docker build -t tractsexplorer .
docker run --rm -p 8000:8000 -e PUBLIC_URL=https://your-site.example tractsexplorer
```

| Env var | Default | Purpose |
|---|---|---|
| `PUBLIC_URL` | `http://localhost:8000` | base of the `view_url` links the API returns (e.g. `https://brainlife.io/tractexplorer`) |
| `ROOT_PATH` | empty | path prefix, when the reverse proxy passes it through (leave empty if the proxy strips it) |
| `TE_ROOT` | `data/` | folder holding `config.json`, `lookups.json`, the threshold CSVs and `decoding/topic-terms.json` |

Behind a reverse proxy, run uvicorn with `--proxy-headers` and `--forwarded-allow-ips` set to the proxy. Unknown paths return 404. `thresholds/` may be a symlink, e.g. to a data volume.

## API

All responses are JSON. Responses for one threshold include a `threshold` object naming it, and cluster and tract results include a `view_url` that opens the matching view in the explorer.

| Endpoint | Returns |
|---|---|
| `GET /thresholds` | available thresholds and the default |
| `GET /topics?q=` | topics matching a keyword |
| `GET /view-url?thr=&tab=&cluster=&tract=&...` | a validated explorer link (parameters as in *View links*; an unknown tract, cluster or atlas is a 404) |
| `GET /{thr}/clusters` | the clusters |
| `GET /{thr}/clusters/{n}?atlas=&limit=` | one cluster: member tracts, top topics, networks |
| `GET /{thr}/tracts?q=` | find tracts by id or label (`leftArc`, `arcuate`, `L Arcuate`) |
| `GET /{thr}/tracts/{tract}?atlas=&limit=` | one tract: cluster, top topics, networks |
| `GET /{thr}/topics/{topic_id}/tracts?limit=` | tracts ranked by match to a topic |
| `GET /{thr}/networks?entity=tract:leftArc&atlas=&limit=` | Dice overlap with atlas networks (`entity=cluster:2` also works) |

- `{thr}` is a folder (`thr-15`), a value (`0.15`) or `default` (`defaultThreshold` in `config.json`).
- Top topics are ranked by |r|, keeping the sign; `topics/{id}/tracts` is ranked by signed r.
- Without `atlas`, the alphabetically first atlas is used. A network is significant when p ≤ 0.05.
- Errors are `{"error": ..., "hint": ...}`.

## View links

`explorer.html` reads these query parameters on load and keeps the address bar in sync, so any copied link reopens the same view. All are optional and invalid values are ignored. The allowed values live in `data/lookups.json`.

| Parameter | Values | Meaning |
|---|---|---|
| `thr` | folder, value or `default` | threshold |
| `tab` | `cluster`, `tract`, `matrix` | which view (a bare `#cluster` hash also works) |
| `cluster` | positive integer | cluster number |
| `tract` | tract id, e.g. `leftArc` | tract |
| `topic` | positive integer | topic to open in the details panel |
| `section` | `map`, `topics`, `charts`, `networks` | section to scroll to |
| `map` | `both`, `RAS`, `LPI`, `bytract` | endpoint map |
| `hemi` | `both`, `lh`, `rh` | hemisphere |
| `view` | `dorsal`, `ventral`, `anterior`, `posterior`, `lateral-left`, `lateral-right`, `medial-left`, `medial-right` | camera angle |
| `atlas` | atlas code, e.g. `AS200Y17` | network atlas |

Example: `explorer.html?thr=0.15&tab=tract&tract=leftArc&atlas=AS200Y17`.

## SKAI panel

The explorer can embed a chat panel for [SKAI](https://github.com/brainlife/brainlife-assistant), brainlife's assistant (`skai-client.js`, `skai-panel.js`). It appears only when `config.skai.url` is set and a brainlife login is in the browser (`localStorage['jwt']`; for development, `localStorage['tractexplorer.devJwt']`). Set `skai.url` only in the brainlife.io deployment: elsewhere there is no brainlife login to read.

- SKAI answers through its TractsExplorer tool server (in `brainlife-assistant`), which calls this API. The panel applies the explorer links in SKAI's replies to the view and adds a "Showing: ..." chip to reopen each one.
- Each question carries a hidden line describing the current view, e.g. `(Viewing in TractsExplorer: threshold thr-15 · tract leftArc)`, so "this tract" means something.
- All open explorer tabs share one SKAI session, so an answer in one tab moves the others too.
- SKAI must list the explorer's origin in its `WHITELISTED_ORIGINS`.

**Open questions.** The login link sends users to `config.skai.loginUrl` with `?redirect=<current page>`, but brainlife's auth may redirect back differently. `localStorage['jwt']` is only visible on brainlife's own origin. SKAI's `/ticket` may answer 500 instead of 401 for a bad token, which the panel would show as "offline" rather than asking for a new login.
