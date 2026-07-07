# Jellyfin Media Card — Sensors

The Home Assistant sensor configuration that powers the
[**Jellyfin Media Card**](https://github.com/a4happy20/jellyfin-media-card).
It fetches "Recently Added" and "Next Up" items from your Jellyfin server, tags each
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
| `sensor.jellyfin_recent_card_data` | one REST sensor per library id | Recently added episodes, merged and tagged by library |
| `sensor.jellyfin_next_up_card_data` | Jellyfin `Shows/NextUp` | Your Jellyfin User's "Next Up" queue |

<br>

Each sensor's `episodes` attribute is a list of items shaped exactly the way the card
expects (`id, series, season, episode, title, overview, library, added, episode_art,
series_art`). Point the card's `entity` at whichever sensor you want to display.

```json
[
    {
        'id': 'episode_id',
        'series': 'Series Name',
        'season': 1,
        'episode': 1,
        'title': 'Episode Name',
        'overview': "Episode Description",
        'library': 'Library Name - defined in the sensor',
        'added': 'Date Added',
        'episode_art': 'URL Path to the episode's art',
        'series_art': 'URL Patch to the series's poster art'
    }
]
```

<br>

## Prerequisites

- A running Jellyfin server reachable from Home Assistant.
- A Jellyfin **API key** (Jellyfin dashboard → Advanced → API Keys).
- Your Jellyfin **user ID** and the **ParentId** of each library you want to show. (see below)
- (Optionally) The [Jellyfin Media Card](https://github.com/a4happy20/jellyfin-media-card)
  to actually render the data in the ui.

## Setup

### 0. Enable packages (Optional)
<details>
  <summary>Package Setup</summary>

<br>

  > See the official docs: **[Configuration packages — Home Assistant](https://www.home-assistant.io/docs/configuration/packages/)**.

<br>

In your `configuration.yaml`, tell Home Assistant to load a `packages` folder:

```yaml
homeassistant:
  packages: !include_dir_named packages
```

Be sure to create the packages folder `config/packages/jellyfin_media_card_sensors.yaml`
(If you already have a `homeassistant:` block, just add the `packages:` line under it.)
Alternatively, include this one file directly:

```yaml
homeassistant:
  packages:
    jellyfin_media_card: !include jellyfin_media_card_sensors.yaml
```

</details>

<br>

### 1. Add the package file

Copy `jellyfin_media_card_sensors.yaml` into your `config/packages/jellyfin_media_card_sensors.yaml` folder (create it if it
doesn't exist).

<br>

### 2. Add your secrets

```yaml
jellyfin_nextup_url: "http://YOUR_JELLYFIN_HOST:8096/Shows/NextUp?userId=<UID>&Limit=10&Fields=Overview,LocationType,Path,SeriesId,DateCreated,ParentIndexNumber,IndexNumber&EnableImages=true"
jellyfin_recent_library: "http://YOUR_JELLYFIN_HOST:8096/Users/<UID>/Items?ParentId=<LIB_ID>&IncludeItemTypes=Episode&Recursive=true&SortBy=DateCreated&SortOrder=Descending&Fields=Overview,LocationType,Path,SeriesId,PremiereDate&Limit=3"
```

You can find your 


Copy the entries from `secrets.yaml.example` into your `config/secrets.yaml` and fill in
real values (API key, user ID, library IDs, and your Jellyfin host/URL). The URL WITHOUT
`api_key` — auth is sent via the `Authorization` header.

You must also set the base image URL inside `jellyfin_media_card.yaml`: replace
`https://YOUR_JELLYFIN_URL` (two places) with the address the card's images should load
from. (This lives in the template rather than `secrets.yaml` because `!secret` can't be
resolved inside a Jinja template string.)

#### sensor.jellyfin_recent_card_data
```yaml
    sensor:
      - name: "Jellyfin Recent Card Data"
        unique_id: jellyfin_recent_card_data
        state: >
          {% set eps = this.attributes.episodes | default([]) if this is defined else [] %}
          {{ eps | length }}
        attributes:
          episodes: >
            {% set base = 'https://YOUR_JELLYFIN_URL' %}
```
#### sensor.jellyfin_next_up_card_data
```yaml
    sensor:
      - name: "Jellyfin Next Up Card Data"
        unique_id: jellyfin_next_up_card_data
        state: >
          {% set eps = this.attributes.episodes | default([]) if this is defined else [] %}
          {{ eps | length }}
        attributes:
          episodes: >
            {% set base = 'https://YOUR_JELLYFIN_URL' %}
```

### 4. Adjust libraries

The example ships three "recently added" libraries. To add or remove one, edit three places:
1. the secrets.yaml (see example file),
2. the REST sensor block,
3. the template sensor's `trigger` → `entity_id` list, and
4. the template sensor's `sources` list.

#### See the secrets.yaml example
```yaml
jellyfin_recent_youtube: "http://YOUR_JELLYFIN_HOST:8096/Users/<UID>/Items?ParentId=<LIB_ID>&IncludeItemTypes=Episode&Recursive=true&SortBy=DateCreated&SortOrder=Descending&Fields=Overview,LocationType,Path,SeriesId,PremiereDate&Limit=3"
```

#### sensor.jellyfin_recent_youtube
```yaml
rest:
  - resource: !secret jellyfin_recent_youtube
    scan_interval: 300
    headers:
      Authorization: !secret jellyfin_auth_header
    sensor:
      - name: "Jellyfin Recent YouTube"
        unique_id: jellyfin_recent_youtube
        value_template: >
          {{ (value_json.Items | default([])
              | selectattr('LocationType','eq','FileSystem') | list | length)
             if value_json is defined else 0 }}
        json_attributes:
          - Items
```

#### sensor.jellyfin_recent_card_data
```yaml
template:
  - trigger:
      - trigger: state
        entity_id:
          - sensor.jellyfin_recent_youtube
          - sensor.jellyfin_recent_anime
          - sensor.jellyfin_recent_ecchi
      - trigger: homeassistant
        event: start
    sensor:
      - name: "Jellyfin Recent Card Data"
        unique_id: jellyfin_recent_card_data
        state: >
          {% set eps = this.attributes.episodes | default([]) if this is defined else [] %}
          {{ eps | length }}
        attributes:
          episodes: >
            {% set base = 'https://YOUR_JELLYFIN_URL' %}
            {# --- source libraries: entity + library key. Edit to add/remove. --- #}
            {% set sources = [
                 ('sensor.jellyfin_recent_youtube', 'youtube'),
                 ('sensor.jellyfin_recent_anime',   'anime'),
                 ('sensor.jellyfin_recent_ecchi',   'ecchi')
            ] %}
```


### 5. Check config and restart

Developer Tools → YAML → **Check Configuration** (or `ha core check`), then restart Home
Assistant. On start, the sensors populate; the caching logic keeps the last good list if a
fetch briefly returns nothing.

## Using it with the card

Once `sensor.jellyfin_recent_card_data` exists, add the sensor entity to the card in the dashboard:

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
