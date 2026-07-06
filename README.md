# Jellyfin Media Card — Sensors

The Home Assistant sensor configuration that powers the
[**Jellyfin Media Card**](https://github.com/a4happy20/jellyfin-media-card).
It fetches recently added and "Next Up" items from your Jellyfin server, tags each
item by library, caches the last good result, and exposes them as template sensors
the card reads.

> **This is a Home Assistant configuration *package*, not an integration or a HACS
> add-on.** You don't install it through HACS. You copy the YAML into your own
> configuration and enable Home Assistant's packages feature. See the official docs:
> **[Configuration packages — Home Assistant](https://www.home-assistant.io/docs/configuration/packages/)**.

## What it creates

Two trigger-based template sensors:

| Entity | Source | Purpose |
|--------|--------|---------|
| `sensor.jellyfin_recent_card_data` | one REST sensor per library (YouTube / Anime / Ecchi in the example) | Recently added episodes, merged and tagged by library |
| `sensor.jellyfin_next_up_card_data` | Jellyfin `Shows/NextUp` | The user's "Next Up" queue |

Each sensor's `episodes` attribute is a list of items shaped exactly the way the card
expects (`id, series, season, episode, title, overview, library, added, episode_art,
series_art`). Point the card's `entity` at whichever sensor you want to display.

## Prerequisites

- A running Jellyfin server reachable from Home Assistant.
- A Jellyfin **API key** (Jellyfin dashboard → Advanced → API Keys).
- Your Jellyfin **user ID** and the **ParentId** of each library you want to show.
- The [Jellyfin Media Card](https://github.com/a4happy20/jellyfin-media-card)
  installed, to actually render the data.

## Setup

### 1. Enable packages (once)

In your `configuration.yaml`, tell Home Assistant to load a `packages` folder:

```yaml
homeassistant:
  packages: !include_dir_named packages
```

(If you already have a `homeassistant:` block, just add the `packages:` line under it.)
Alternatively, include this one file directly:

```yaml
homeassistant:
  packages:
    jellyfin_media_card: !include jellyfin_media_card.yaml
```

### 2. Add the package file

Copy `jellyfin_media_card.yaml` into your `config/packages/` folder (create it if it
doesn't exist).

### 3. Add your secrets

Copy the entries from `secrets.yaml.example` into your `config/secrets.yaml` and fill in
real values (API key, user ID, library IDs, and your Jellyfin host/URL). The URL WITHOUT
`api_key` — auth is sent via the `Authorization` header.

You must also set the base image URL inside `jellyfin_media_card.yaml`: replace
`https://YOUR_JELLYFIN_URL` (two places) with the address the card's images should load
from. (This lives in the template rather than `secrets.yaml` because `!secret` can't be
resolved inside a Jinja template string.)

### 4. Adjust libraries

The example ships three "recently added" libraries. To add or remove one, edit three
places, all labeled with comments in the YAML:
1. the REST sensor block,
2. the merge sensor's `trigger` → `entity_id` list, and
3. the merge sensor's `sources` list.

### 5. Check config and restart

Developer Tools → YAML → **Check Configuration** (or `ha core check`), then restart Home
Assistant. On start, the sensors populate; the caching logic keeps the last good list if a
fetch briefly returns nothing.

## Using it with the card

Once `sensor.jellyfin_recent_card_data` exists, add the card to a dashboard:

```yaml
type: custom:jellyfin-media-card
entity: sensor.jellyfin_recent_card_data
```

Full card options are documented in the
[Jellyfin Media Card README](https://github.com/a4happy20/jellyfin-media-card).

## A note on the template + packages interaction

This package uses **trigger-based** template sensors written in the modern list style
(`template:` as a list of `- trigger:` blocks), which merges cleanly under packages. If you
add your own template entities elsewhere, keep them in this same list style to avoid merge
conflicts.

## License

Licensed under the [GNU General Public License v3.0](LICENSE) — the same license as the
Jellyfin Media Card.
