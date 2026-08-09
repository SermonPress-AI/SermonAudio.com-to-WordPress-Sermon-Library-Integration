# SermonAudio.com to WordPress Sermon Library Integration

This n8n workflow automatically publishes sermons to WordPress from a church's
[sermonaudio.com](https://www.sermonaudio.com) account. It builds a search engine optimized sermon
library on the church's website. Each sermon is given its own page, and each sermon page features a
video embed, an audio embed, a transcript, and (when present) video clips.

---

## 📌 Quick Links

- [What it does](#-what-it-does)
- [⚠️ Before you start: the WordPress side](#️-before-you-start-the-wordpress-side)
- [Prerequisites](#-prerequisites)
- [Setup instructions](#️-setup-instructions)
- [Configuration reference](#-configuration-reference)
- [WordPress endpoint contract](#-wordpress-endpoint-contract)
- [Known limitations](#-known-limitations)
- [Troubleshooting](#-troubleshooting)
- [Security notes](#-security-notes)

---

## 📖 What it does

Once a day the workflow:

1. Asks the SermonAudio API for the broadcaster's most recent sermons.
2. Ignores anything preached before your configured **start date**.
3. Records each sermon in an n8n **Data Table**, which acts as the memory of what has already been
   published — so a sermon is never posted twice.
4. For each sermon not yet published, fetches full details, any video clips, and the transcript.
5. Creates a WordPress sermon post with the title, date, speaker, Bible book(s), audio embed, video
   embed, transcript, and a clips gallery.
6. Marks the sermon as published in the Data Table.

```
Daily Trigger
   └─ Configuration ......... your five settings live here
      └─ Fetch Sermon List ... SermonAudio API
         └─ Split Out Sermons
            └─ Filter By Start Date
               └─ Upsert Sermon Row ........ writes to the Data Table
                  └─ Filter Unpublished Sermons
                     └─ Loop Over Sermons ── for each sermon:
                          ├─ Fetch Sermon Details
                          ├─ Update Sermon Row
                          ├─ Fetch Sermon Clips
                          ├─ Build Clips Shortcode
                          ├─ Download Transcript
                          ├─ Publish Sermon To WordPress
                          └─ Mark Sermon Published
```

---

## ⚠️ Before you start: the WordPress side

**This workflow is not self-contained.** It posts to a custom WordPress REST route,
`POST /wp-json/5mt/v1/sermons`, and it injects a `[sermon_clips_custom]` shortcode into each post.
Both are provided by a **companion WordPress plugin that is not part of this repository**.

Without that plugin installed on the target site:

- every request returns **404**, and no sermons are published;
- if posts were created some other way, the shortcode renders as literal text on the page.

The plugin exists to bridge a gap in WordPress itself. Sermons are stored using
[**Church Theme Content**](https://github.com/churchthemes/church-theme-content) (the `ctc_sermon`
post type), but Church Theme Content never calls `register_post_meta()`, so its `_ctc_sermon_*`
fields **cannot be written through the standard `/wp-json/wp/v2/` API at all**. A custom route is
the only practical way in. See [WordPress endpoint contract](#-wordpress-endpoint-contract) for
everything the plugin must implement.

---

## 📋 Prerequisites

| Requirement | Notes |
|---|---|
| **n8n 1.113.1 or newer** | Data Tables were introduced in 1.113.1 (Sept 2025). **2.3+ recommended.** Cloud and self-hosted, all plans. |
| **A SermonAudio API key** | Request one from your [SermonAudio dashboard](https://www.sermonaudio.com/dashboard/account/settings/). A key is **required** — there is no anonymous access, even for public sermons. |
| **Your `broadcasterID`** | The last part of your public URL: `https://www.sermonaudio.com/broadcasters/`**`yourbroadcasterid`**. Not a secret. |
| **A WordPress site** with [Church Theme Content](https://wordpress.org/plugins/church-theme-content/) | Install from the WordPress.org plugin directory, **not** from a GitHub ZIP — the GitHub repo uses submodules and a plain download ships broken. |
| **A Church Theme Content–compatible theme** | Church Theme Content only *stores* sermon data. A compatible theme is required to display it. |
| **The companion bridge plugin** | See the section above. |
| **A WordPress Application Password** | WP Admin → Users → your user → Application Passwords. Use a dedicated account, not a personal login. |

---

## ⚙️ Setup instructions

### 1. Create the Data Table

In n8n, open your project → **Data tables** → **Create Data table**. Name it something like
`Sermons`, then add these **11 columns exactly** — names are case-sensitive:

| Column name | Type |
|---|---|
| `sermonID` | String |
| `title` | String |
| `DateSermon` | Date |
| `speaker` | String |
| `books` | String |
| `embedLink` | String |
| `transcriptLink` | String |
| `downloadSermonLink` | String |
| `hasPDF` | String |
| `uploaded` | Boolean |
| `SermonIdWp` | String |

> **Do not skip `SermonIdWp` or `uploaded`.** They are written at the very end of the run, *after*
> the WordPress post is created. If either column is missing, the post is published but never marked
> as done — and the next run publishes it again.

Importing this workflow does **not** create the table. Data Table schemas do not travel inside
workflow JSON.

### 2. Import the workflow

In n8n: **Workflows** → **Import from File** → select `sermonaudio-to-wordpress.json`.

Or **Import from URL** with the raw GitHub link to that file.

### 3. Select your Data Table in three nodes

The Data Table selector cannot be pre-filled by the import, so it ships blank on purpose. Open each
of these nodes and pick your table from the **Data table** dropdown:

- `Upsert Sermon Row`
- `Update Sermon Row`
- `Mark Sermon Published`

### 4. Create the SermonAudio credential

Open the `Fetch Sermon List` node → **Credential for Header Auth** → **Create new credential**:

| Field | Value |
|---|---|
| **Name** (the header name) | `X-Api-Key` |
| **Value** | your SermonAudio API key |

Name the credential something recognisable, like `SermonAudio API`. Then select that same credential
in `Fetch Sermon Details` and `Fetch Sermon Clips`.

> ⚠️ A SermonAudio API key is **read *and* write**. There is no read-only scope — the same key can
> delete your entire sermon archive. Treat it like a password.

### 5. Fill in the Configuration node

Open the **`Configuration`** node and replace all five placeholder values. This is the only node
with settings you need to edit.

| Field | Example | What it is |
|---|---|---|
| `broadcasterID` | `examplechurch` | Your SermonAudio broadcaster ID |
| `wordpressBaseUrl` | `https://example.org` | Your site root — **no trailing slash**, no `/wp-json` |
| `wordpressUsername` | `sermonbot` | The WordPress user the posts are created as |
| `wordpressAppPassword` | `abcd efgh ijkl mnop qrst uvwx` | That user's Application Password (spaces are fine) |
| `startDate` | `2024-01-01T00:00:00` | Sermons preached before this are ignored entirely |

> **Choose `startDate` deliberately.** It is the single most important setting. Setting it far in the
> past will attempt to import your whole archive on the first run; see
> [Known limitations](#-known-limitations) for why that needs care.

### 6. Set the workflow timezone

**Workflow menu (⋯) → Settings → Timezone.** Set it to your church's timezone.

If you leave it on the default, self-hosted n8n falls back to `America/New_York` and n8n Cloud uses
whatever timezone the account owner signed up in. That affects both when the workflow runs and the
dates on your sermon posts.

### 7. Test, then activate

Run it manually once with a **recent** `startDate` so only a sermon or two is processed. Check that
the post appears in WordPress with its media and transcript intact. Then activate the workflow.

The trigger runs daily at midnight. (Precisely, n8n staggers the seconds — it fires at 00:00:*ss*,
where the exact second is derived from the workflow ID. This spreads load and is not something you
need to configure.)

---

## 🔧 Configuration reference

| Setting | Where to set |
|---|---|
| `broadcasterID`, `wordpressBaseUrl`, `wordpressUsername`, `wordpressAppPassword`, `startDate` | ⚙️ **`Configuration`** node |
| SermonAudio API key | n8n **credential** — Header Auth, name `X-Api-Key` |
| Which Data Table to use | The three Data Table nodes |
| Run schedule | `Daily Trigger` node |
| Timezone | Workflow Settings → Timezone |

Everything else — the API URLs, the field mapping, the Bible-book normaliser — is wired to these and
should not need editing.

---

## 🔌 WordPress endpoint contract

Documented here so the companion plugin can be built, audited, or replaced.

### Request

```
POST {wordpressBaseUrl}/wp-json/5mt/v1/sermons
Content-Type: application/json
```

| Parameter | Example | Church Theme Content target |
|---|---|---|
| `username` | `sermonbot` | — (authentication) |
| `password` | `abcd efgh …` | — (authentication) |
| `title` | `The Good Shepherd` | `post_title` |
| `date` | `2026-05-16 10:30:00` | `post_date` / `post_date_gmt` |
| `post_status` | `publish` | `post_status` |
| `book` | `["Psalms","John"]` | `ctc_sermon_book` taxonomy terms |
| `speaker` | `Rev. Jane Doe` | `ctc_sermon_speaker` taxonomy term |
| `audio` | `https://embed.sermonaudio.com/player/a/{id}` | `_ctc_sermon_audio` meta |
| `video` | `{sermonID}` (bare ID) | `_ctc_sermon_video` meta |
| `audioEmbed` | `"true"` | — (bridge-specific flag) |
| `videoEmbed` | `"true"` | — (bridge-specific flag) |
| `has_full_text` | `"1"` | `_ctc_sermon_has_full_text` meta |
| `content` | clips shortcode + transcript | `post_content` |

**The response must be JSON containing a top-level `post_id`.** The workflow reads `$json.post_id`
and stores it; nothing else in the response is used.

### The plugin must transform the media values

This is the least obvious part of the contract. Church Theme Content stores media as either a URL or
raw embed code, and the theme framework decides which by testing whether the string starts with
`http`. Neither value can be stored as sent:

- **`audio`** is an `embed.sermonaudio.com` URL. SermonAudio is **not** a WordPress oEmbed provider,
  so storing it verbatim renders a bare hyperlink instead of a player.
- **`video`** is a bare sermon ID. It is not a URL, so it is treated as embed code — and the theme
  prints the ID as literal text.

Both must be converted to raw `<iframe>` markup before `update_post_meta()`, using
`https://embed.sermonaudio.com/player/a/{id}/` for audio and `/player/v/{id}/` for video. The stored
value must **not** begin with `http`, or the framework routes it back down the oEmbed path.

> **Podcast trade-off:** Church Theme Content builds its podcast feed enclosure from
> `_ctc_sermon_audio`. An iframe is not an audio file, so podcasting breaks. If the podcast feed
> matters, store the direct `.mp3` URL in `_ctc_sermon_audio` (which also yields a real player) and
> keep the iframe on `_ctc_sermon_video`.

### Taxonomy terms

`book` and `speaker` arrive as **names**, not term IDs. The plugin must resolve each to an existing
term or create it. Church Theme Content ships no Bible book terms and validates nothing, so **any
spelling mismatch silently creates a duplicate term** — match case-insensitively before creating.

The workflow normalises SermonAudio's `bibleText` before sending: it strips chapter and verse
(`Psalm 23:1-6` → `Psalm`), converts Roman numeral prefixes (`II Timothy` → `2 Timothy`), and maps
known spelling differences to the names ChurchThemes' framework expects:

| SermonAudio | Church Theme Content |
|---|---|
| `Psalm` | `Psalms` |
| `Song of Songs`, `Canticles` | `Song of Solomon` |
| `Revelations`, `Revelation of John` | `Revelation` |

Any book name not on that list passes through unchanged. If you see duplicate or oddly-named book
terms appearing, add the mapping to the `book` parameter in the `Publish Sermon To WordPress` node.

### Shortcode

```
[sermon_clips_custom ids="123,456" per_view="2" ratio="9:16"]
```

`ids` is a comma-separated list of SermonAudio clip IDs; `per_view` is how many clips are visible at
once; `ratio` is the aspect ratio (`9:16` for vertical clips). It must be registered by the
**plugin**, not the theme — otherwise a theme change turns it into literal text across every post.
When a sermon has no clips the workflow emits an empty string, so the shortcode is simply absent.

---

## 🚧 Known limitations

These are inherited from the original workflow and are **documented, not fixed**. They are listed
worst-first. Ask before relying on this at scale.

| # | Issue | Impact |
|---|---|---|
| 1 | **A missing transcript stops the entire run.** `Download Transcript` has no error handling, and `transcript.downloadURL` may be null. | The loop aborts and every remaining sermon in that run is silently skipped. One transcript-less sermon halts the whole backlog. |
| 2 | **Video-only sermons crash the run.** `Update Sermon Row` reads `media.audio[0].downloadURL` without guarding an empty array. | Same as above — run aborts. |
| 3 | **Only the first 100 sermons are ever seen.** The list request is `pageSize=100&page=1` with no pagination. | Fine for a daily catch-up; **lossy for an initial backfill** of a large archive. Import history in stages by moving `startDate` forward, or publish older sermons manually. |
| 4 | **The two date filters use different fields.** One filters on `preachDate`, the other on a date derived from `publishTimestamp`. | A sermon preached and published in different windows can be recorded but never published, or skipped entirely. |
| 5 | **Publishing is not transactional.** The row is marked `uploaded` *after* the WordPress post is created. | Any failure in between leaves a published post that is not marked done — the next run publishes a duplicate. |
| 6 | **Media may not be ready yet.** A midnight run right after a Sunday upload can find audio still processing and the transcript not yet generated. | The sermon is published without a transcript and never revisited. Consider scheduling later in the day. |
| 7 | **The transcript is an unformatted wall of text.** SermonAudio returns plain text hard-wrapped at ~70 columns with no paragraphs. | It renders as one undifferentiated block unless the plugin reflows it. |
| 8 | **Do not change the loop batch size.** `Loop Over Sermons` relies on a batch size of 1. | At larger sizes every sermon in a batch resolves to the first sermon's data, corrupting output. |
| 9 | **No error notification.** The loop's `done` output is unconnected and no error workflow is set. | A nightly failure is silent. Set an error workflow in Workflow Settings. |
| 10 | **Credentials are sent in the request body**, not an `Authorization` header. | See [Security notes](#-security-notes). |

---

## 🔍 Troubleshooting

**`Could not find the data table: ''`**
You skipped step 3. Open each of the three Data Table nodes and select your table.

**`unknown column name 'SermonIdWp'`**
Your Data Table is missing a column. Compare against the 11 columns in step 1 — names are
case-sensitive. Note this error appears *after* a post has already been published.

**`401 Unauthorized` from `api.sermonaudio.com`**
The Header Auth credential is missing, or its **Name** field is not exactly `X-Api-Key`. The name
field holds the *header name*, not a label for the credential.

**`404` from your own site**
The companion bridge plugin is not installed or not active. Confirm by visiting
`https://yoursite.example/wp-json/` and looking for the `5mt/v1` namespace.

**Sermons are published but the video area shows a long number, or audio is just a link**
The bridge plugin is storing the raw values instead of converting them to iframes. See
[the transform section](#the-plugin-must-transform-the-media-values).

**Nothing happens at all**
Check `startDate` — sermons preached before it are ignored. Check that the workflow is activated and
that the timezone is what you expect.

**A sermon was published twice**
Its Data Table row was never marked `uploaded`. Usually limitation #5 or a missing column.

---

## 🔒 Security notes

- **Never commit a filled-in copy of this workflow.** Once the `Configuration` node contains a real
  Application Password, the JSON is a secret. Add your working copy to `.gitignore`.
- **The credentials in `Configuration` are stored in plain text** inside the workflow and appear in
  execution logs. Use a **dedicated WordPress user** with only the capability to create sermon posts
  — never an administrator account you also log in with. Revoke its Application Password from
  WP Admin → Users → Application Passwords if it is ever exposed.
- **Always use HTTPS** for `wordpressBaseUrl`. The password travels in the request body.
- **Your SermonAudio API key can write and delete.** Store it only in the n8n credential, never in a
  node parameter.
- Sending credentials in the body rather than an `Authorization: Basic` header is a quirk of the
  companion plugin, kept here for compatibility. If you control the plugin, prefer standard
  Application Password Basic Auth and an n8n Basic Auth credential.

---

## 📜 License

MIT — see [LICENSE](LICENSE).

## 🤝 Contributing

Contributions are welcome. Please read the
[SermonPress AI Contributor's Guide](https://github.com/SermonPress-AI/.github/blob/main/CONTRIBUTOR'S_GUIDE.md)
before opening a pull request.

## 📧 Contact

[directors@sermonpress.ai](mailto:directors@sermonpress.ai) · [sermonpress.ai](https://sermonpress.ai/)

> *"How beautiful are the feet of those who preach the good news!"* — Romans 10:15
