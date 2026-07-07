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

<br>

![license](https://img.shields.io/badge/License-GPLv3-blue.svg)


## What is it?

Two trigger-based template sensors that pull information from a/multiple REST sensors:

| Entity | Source | Purpose |
|--------|--------|---------|
| `sensor.jellyfin_recent_card_data` | one REST sensor per library id | Recently added episodes, merged and tagged by library |
| `sensor.jellyfin_next_up_card_data` | Jellyfin `Shows/NextUp` | Your Jellyfin User's "Next Up" queue |

<br>

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
        'series_art': 'URL Path to the series's poster art'
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

<br>

## Setup

<br>

### Easy to setup
High level overview:

    • add the sensors to home assistant
    • add the urls to your secrets file - replacing api key, user id, and library ids
    • edit the template sensor pointing the url to your jellyfin instance

<br>

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

Copy `jellyfin_media_card_sensors.yaml` into your `config/packages/jellyfin_media_card_sensors.yaml` folder.

> You don't need to setup packages if you don't want to.
> Instead you would just put the REST sensors and Template sensors
> where they belong in your setup.

<br>

### 2. Add your secrets

```yaml
# ======================================================================================
# Find the values:
#   <UID>       Jellyfin user ID  (Jellyfin dashboard > Users > click user > URL)
#   YOURKEY     Jellyfin API key  (Jellyfin dashboard > Advanced > API Keys)
#   <LIB_ID>    Library ParentId  (open a library in Jellyfin, copy the id in the URL)
#   Limit=3     Amount of items   (Set how many episodes the sensor should detect)
#   Limit=10 
# =====================================================================================
```

```yaml
jellyfin_auth_header: 'MediaBrowser Token="YOURKEY"'
jellyfin_nextup_url: "http://YOUR_JELLYFIN_HOST:8096/Shows/NextUp?userId=<UID>&Limit=10&Fields=Overview,LocationType,Path,SeriesId,DateCreated,ParentIndexNumber,IndexNumber&EnableImages=true"
jellyfin_recent_library: "http://YOUR_JELLYFIN_HOST:8096/Users/<UID>/Items?ParentId=<LIB_ID>&IncludeItemTypes=Episode&Recursive=true&SortBy=DateCreated&SortOrder=Descending&Fields=Overview,LocationType,Path,SeriesId,PremiereDate&Limit=3"
```

<br>

### 3. Edit the template sensor

You must also set the base image URL inside `jellyfin_media_card_sensors.yaml`: replace
`http://YOUR_JELLYFIN_HOST:8096` (two places) with the url of your Jellyfin instance.
(This lives in the template rather than `secrets.yaml` because `!secret` can't be
resolved inside a Jinja template string.)

<br>

### Template sensor.jellyfin_recent_card_data (NOT the full sensor, see the actual file)

<details>
    <summary>info</summary>

```yaml
        sensor:
          - name: "Jellyfin Recent Card Data"
            unique_id: jellyfin_recent_card_data
            state: >
              {% set eps = this.attributes.episodes | default([]) if this is defined else [] %}
              {{ eps | length }}
            attributes:
              episodes: >
                {% set base = 'http://YOUR_JELLYFIN_HOST:8096' %}
```
</details>

<br>

### Template sensor.jellyfin_next_up_card_data (NOT the full sensor, see the actual file)
<details>
    <summary>info</summary>

```yaml
        sensor:
          - name: "Jellyfin Next Up Card Data"
            unique_id: jellyfin_next_up_card_data
            state: >
              {% set eps = this.attributes.episodes | default([]) if this is defined else [] %}
              {{ eps | length }}
            attributes:
              episodes: >
                {% set base = 'http://YOUR_JELLYFIN_HOST:8096' %}
```
</details>

<br>

### 4. Adjust libraries

Using the url you just set up and added to your secrets.yaml:
1. you need to tell the sensor what library to monitor,
2. locate the REST sensor block, and add your secret
```yaml
- resource: !secret jellyfin_recent_library
```
4. create separate sensors for each library you want to monitor
5. the template sensor's `trigger` → `entity_id` list, and
6. the template sensor's `sources` list.

<br>

### REST sensor.jellyfin_recent_library (NOT the full sensor, see the actual file)
<details>
    <summary>info</summary>

```yaml
    rest:
      - resource: !secret jellyfin_recent_library            # your library url secret
        scan_interval: 300
        headers:
          Authorization: !secret jellyfin_auth_header
        sensor:
          - name: "Jellyfin Recent Library"                 # sensor name
            unique_id: jellyfin_recent_library              # sensor id
            value_template: >
              {{ (value_json.Items | default([])
                  | selectattr('LocationType','eq','FileSystem') | list | length)
                 if value_json is defined else 0 }}
            json_attributes:
              - Items
```

</details>

<br>

### Template sensor.jellyfin_recent_card_data (NOT the full sensor, see the actual file)
<details>
    <summary>info</summary>

```yaml
    template:
      - trigger:
          - trigger: state
            entity_id:
              - sensor.jellyfin_recent_library            # your library REST sensor
              - sensor.jellyfin_recent_library_2          # your additional library REST sensor
              - sensor.jellyfin_recent_library_3          # your additional library REST sensor
          - trigger: homeassistant
            event: start
        sensor:
          - name: "Jellyfin Recent Card Data"             # sensor name
            unique_id: jellyfin_recent_card_data          # sensor id
            state: >
              {% set eps = this.attributes.episodes | default([]) if this is defined else [] %}
              {{ eps | length }}
            attributes:
              episodes: >
                {% set base = 'http://YOUR_JELLYFIN_HOST:8096' %}
                {# --- source libraries: entity + library key. Edit to add/remove. --- #}
                {% set sources = [
                     ('sensor.jellyfin_recent_library', 'library'),                 # your library REST sensor and a name (jellyfin-media-card will read this name for setting art_overrides)
                     ('sensor.jellyfin_recent_library_2',   'library2'),            # your library REST sensor and a name (jellyfin-media-card will read this name for setting art_overrides)
                     ('sensor.jellyfin_recent_library_3',   'library3')             # your library REST sensor and a name (jellyfin-media-card will read this name for setting art_overrides)
                ] %}
```

</details>

<br>

### 5. Check config and restart

Developer Tools → YAML → **Check Configuration**, then restart Home
Assistant. On start, the sensors populate; the caching logic keeps the last good list if a
fetch briefly returns nothing.

<br>

## Using it with the jellyfin-media-card

Once `sensor.jellyfin_recent_card_data` exists, add the sensor entity to the card in the dashboard:

```yaml
type: custom:jellyfin-media-card
entity: sensor.jellyfin_recent_card_data OR sensor.jellyfin_next_up_card_data
```

<br>

Full card options are documented in the
[Jellyfin Media Card README](https://github.com/a4happy20/jellyfin-media-card).

<br>

## License

Licensed under the [GNU General Public License v3.0](LICENSE)
