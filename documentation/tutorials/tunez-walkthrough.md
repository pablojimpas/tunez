<!--
SPDX-FileCopyrightText: 2026 Pablo Jiménez Pascual <https://github.com/pablojimpas/ash_opencollection>

SPDX-License-Identifier: MIT
-->

# Tunez Showcase: Generating & Running OpenCollection with Bearer Auth, JSON:API, and GraphQL

In this tutorial, you will explore a concrete, schema-validated OpenCollection v1.0.0 test suite for **Tunez**—the music catalog application built with Ash Framework, `AshJsonApi`, `AshGraphql`, and `AshAuthentication`.

All collection files referenced in this guide live in [`collections/tunez/`](../../collections/tunez/opencollection.yml) and are validated against the canonical OpenCollection v1.0.0 JSON Schema and declarative subset rules.

---

## 1. What You Will Learn

1. **Relationship Dependency Ordering (`Artist -> Album -> Track`)**: Why `Artist` must be created and captured as `{{artist_id}}` before creating an `Album` and its nested `Track` items.
2. **Bearer Token Authentication Chaining**: How to register or sign in via `/api/json/users/sign-in` (or GraphQL `signInUser`), extract `token` into a runtime variable, and pass `Authorization: Bearer {{token}}` to protected mutations.
3. **Multi-Transport Support (JSON:API + GraphQL)**: How a single OpenCollection suite combines JSON:API HTTP requests against `/api/json` and GraphQL queries/mutations against `/gql`.
4. **Pure Elixir Validation & Execution**: How to schema-check and execute collections in ExUnit without Node.js or `@usebruno/cli`.

---

## 2. Tunez Domain & Endpoint Overview

Tunez exposes two Ash domains through `TunezWeb.Router`:

- **`Tunez.Accounts`** (`Tunez.Accounts.User`):
  - JSON:API (`/api/json`): `POST /users/register` and `POST /users/sign-in`, returning a JWT in `meta.token`.
  - GraphQL (`/gql`): `mutation registerUser` and `query signInUser`.
- **`Tunez.Music`** (`Artist`, `Album`, `Track`):
  - `Tunez.Music.Album` defines `belongs_to :artist, Tunez.Music.Artist` (`allow_nil?: false`) and manages child tracks via `manage_relationship(:tracks, type: :direct_control, order_is_key: :order)`.
  - `Tunez.Music.Track` defines `belongs_to :album, Tunez.Music.Album` (`allow_nil?: false`), taking a `:duration` argument (`"MM:SS"` format) converted to `duration_seconds`.
  - JSON:API (`/api/json`): `/artists`, `/albums`, `/artists/:id/albums`, and `?include=albums.tracks`.
  - GraphQL (`/gql`): `createArtist`, `createAlbum`, `getArtistById`, and `searchArtists`.

Because `Album` belongs to `Artist` and `Track` belongs to `Album`, requests must run in topological order and authenticate with a Bearer token before creating or modifying catalog data.

---

## 3. Concrete Collection Layout (`collections/tunez/`)

The concrete collection committed in this repository is organized into 5 sequential folders (23 YAML files total):

```text
collections/tunez/
├── opencollection.yml
├── environments/
│   └── local.yml
├── 01-accounts/
│   ├── folder.yml
│   ├── 01_register_user.yml
│   └── 02_sign_in_user.yml
├── 02-artists/
│   ├── folder.yml
│   ├── 01_create_artist.yml
│   ├── 02_search_artists.yml
│   ├── 03_get_artist_by_id.yml
│   └── 04_update_artist.yml
├── 03-albums-and-tracks/
│   ├── folder.yml
│   ├── 01_create_album_with_tracks.yml
│   ├── 02_list_artist_albums.yml
│   ├── 03_get_artist_with_albums_and_tracks.yml
│   └── 04_update_album_tracks.yml
├── 04-graphql/
│   ├── folder.yml
│   ├── 01_sign_in_user_gql.yml
│   ├── 02_create_artist_gql.yml
│   ├── 03_create_album_with_tracks_gql.yml
│   └── 04_get_artist_with_albums_and_tracks_gql.yml
└── 05-cleanup/
    ├── folder.yml
    ├── 01_delete_album.yml
    └── 02_delete_artist.yml
```

You can also scaffold a collection directly from your Ash domains using `mix ash_opencollection.codegen`:

```bash
mix ash_opencollection.codegen \
  --output ./collections/tunez \
  --domains Tunez.Accounts,Tunez.Music \
  --prefix /api/json \
  --base-url http://127.0.0.1:4000
```

Or target GraphQL operations with `--transport graphql`.

---

## 4. Chaining Bearer Token Authentication

In [`collections/tunez/01-accounts/02_sign_in_user.yml`](../../collections/tunez/01-accounts/02_sign_in_user.yml), signing in returns a JWT in `res.body.meta.token`. A declarative `set-variable` action captures that token without any JavaScript:

```yaml
info:
  name: Sign In User
  type: http
  seq: 2
http:
  method: POST
  url: "{{base_url}}/api/json/users/sign-in"
  headers:
    - name: Accept
      value: application/vnd.api+json
    - name: Content-Type
      value: application/vnd.api+json
  body:
    type: json
    data: |-
      {
        "data": {
          "attributes": {
            "email": "{{email}}",
            "password": "{{password}}"
          },
          "type": "user"
        }
      }
runtime:
  actions:
    - type: set-variable
      phase: after-response
      selector:
        expression: res.body.meta.token
        method: jsonq
      variable:
        name: token
        scope: runtime
  assertions:
    - expression: res.status
      operator: eq
      value: "201"
    - expression: res.body.meta.token
      operator: isDefined
```

Subsequent protected requests—such as `02-artists/01_create_artist.yml` and `03-albums-and-tracks/01_create_album_with_tracks.yml`—declare Bearer authentication directly:

```yaml
auth:
  type: bearer
  token: "{{token}}"
```

When executed sequentially in Bruno or `AshOpenCollection.Runner`, `Authorization: Bearer <token>` is injected automatically.

---

## 5. Walking the `Artist -> Album -> Track` Chain

### Step A: Create an Artist and Extract `artist_id`

[`collections/tunez/02-artists/01_create_artist.yml`](../../collections/tunez/02-artists/01_create_artist.yml) posts to `{{base_url}}/api/json/artists` and stores `res.body.data.id` in `artist_id`.

### Step B: Create an Album with Tracks Using `{{artist_id}}`

[`collections/tunez/03-albums-and-tracks/01_create_album_with_tracks.yml`](../../collections/tunez/03-albums-and-tracks/01_create_album_with_tracks.yml) references `{{artist_id}}` and passes nested `tracks` accepted by `Tunez.Music.Album`'s `:create` action:

```yaml
http:
  method: POST
  url: "{{base_url}}/api/json/albums"
  headers:
    - name: Accept
      value: application/vnd.api+json
    - name: Content-Type
      value: application/vnd.api+json
  body:
    type: json
    data: |-
      {
        "data": {
          "attributes": {
            "artist_id": "{{artist_id}}",
            "cover_image_url": "/images/a-night-at-the-opera.jpg",
            "name": "A Night at the Opera",
            "tracks": [
              {
                "duration": "03:43",
                "name": "Death on Two Legs",
                "order": 0
              },
              {
                "duration": "05:55",
                "name": "Bohemian Rhapsody",
                "order": 1
              }
            ],
            "year_released": 1975
          },
          "type": "album"
        }
      }
  auth:
    type: bearer
    token: "{{token}}"
```

Then [`03_get_artist_with_albums_and_tracks.yml`](../../collections/tunez/03-albums-and-tracks/03_get_artist_with_albums_and_tracks.yml) queries `GET {{base_url}}/api/json/artists/{{artist_id}}?include=albums.tracks` to verify the full `Artist -> Album -> Track` relationship graph, and [`04_update_album_tracks.yml`](../../collections/tunez/03-albums-and-tracks/04_update_album_tracks.yml) issues a JSON:API 1.0 compliant `PATCH` (including `"id": "{{album_id}}"` inside `"data"`).

---

## 6. GraphQL Workflow (`04-graphql/`)

Tunez also exposes an Absinthe/AshGraphql schema at `{{base_url}}/gql`. Folder [`04-graphql/`](../../collections/tunez/04-graphql/folder.yml) demonstrates native OpenCollection `type: graphql` requests:

1. `01_sign_in_user_gql.yml`: Runs `query SignInUser($email: String!, $password: String!)` and saves `res.body.data.signInUser.token`.
2. `02_create_artist_gql.yml`: Runs `mutation CreateArtist($input: CreateArtistInput!)` with Bearer auth and saves `gql_artist_id`.
3. `03_create_album_with_tracks_gql.yml`: Runs `mutation CreateAlbum($input: CreateAlbumInput!)` referencing `"artistId": "{{gql_artist_id}}"` with nested `tracks`, saving `gql_album_id`.
4. `04_get_artist_with_albums_and_tracks_gql.yml`: Queries `getArtistById(id: $id)` selecting `albums { id name yearReleased duration tracks { id number name duration } }`.

---

## 7. Validating & Running in Elixir

You can open `collections/tunez/` directly in Bruno Desktop, or validate and execute it in pure Elixir without installing Node.js:

```elixir
defmodule TunezWeb.ApiCollectionTest do
  use TunezWeb.ConnCase, async: false
  use AshOpenCollection.Test

  test_collection "collections/tunez",
    env: "local",
    variables: %{
      "email" => "curator@tunez.fm",
      "password" => "CorrectHorseBatteryStaple123!"
    }
end
```

Notice that every file in `collections/tunez/` also passes static verification via `OpenCollection.parse!/1`, `OpenCollection.Schema.validate/1`, and `OpenCollection.Declarative.validate/1`, which is checked continuously in the workspace E2E suite.
