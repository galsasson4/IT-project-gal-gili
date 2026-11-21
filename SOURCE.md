# Data source

Gili Barak, Gal Sasson

Our approved source is the music one: we compare the most popular music videos on YouTube in Israel with the Apple Music top 100 chart in Israel, to see where new songs become hits first and how long it takes them to show up on the other platform.

## 1. The source

| | YouTube | Apple Music |
|---|---|---|
| Name | YouTube Data API v3 - most popular videos, music category, Israel | Apple Music RSS - Most Played Songs, Israel |
| URL | `https://www.googleapis.com/youtube/v3/videos?part=snippet,statistics&chart=mostPopular&regionCode=IL&videoCategoryId=10&maxResults=50&fields=items(id,snippet(publishedAt,channelId,channelTitle,title,defaultAudioLanguage),statistics(viewCount,likeCount,commentCount))&key=API_KEY` | `https://rss.marketingtools.apple.com/api/v2/il/music/most-played/100/songs.json` |
| Format | JSON | JSON |
| Key / user | Free API key from Google Cloud. It's kept in `~/.secrets/youtube_key` (outside the project, `chmod 600`) and never goes into the repo | No key, no user |
| License / terms | YouTube API Services Terms of Service and Developer Policies. Free quota of 10,000 units a day, each call costs 1 unit | Apple Marketing Tools terms. Every response says "Copyright © 2026 Apple Inc. All rights reserved." |
| Sampling frequency | Every 2 hours (12 a day), because the list changes several times a day | Every 6 hours (4 a day), because the chart updates about once a day and 6 hours tells us roughly when |

The Apple URL redirects to another address, so it has to be downloaded with `curl -L` (follow redirects).

Each run saves one file, `raw/YYYY-MM-DD/HHMM.json` (UTC), with both responses inside: `{"youtube": ..., "apple": ...}`. In runs where Apple isn't sampled, `apple` is `null`. The 4 manual samples we already have (18-21.11) include both.

## 2. What we collect

From YouTube we get the most popular music videos in Israel at that moment. In all 4 of our samples YouTube returned 30 videos, even though we asked for 50. For each video we get its id, title, channel, upload time, language (when the channel filled it in), and how many views, likes and comments it has right now.

From Apple Music we get the 100 most played songs in Israel, in chart order. For each song we get its id, name, artist, release date and genres. Apple doesn't give play counts, only the order.

The view counts and the order of both lists change all the time, and nobody saves them. That's what we collect.

## 3. Schema (dataset.csv)

Every row is one video or one song in one sample. A YouTube run adds 30 rows, an Apple run adds 100 rows.

| Column | What's in it | Units / format |
|---|---|---|
| `snapshot_utc` | When we collected the sample | ISO 8601, UTC, e.g. `2026-11-21T11:10:00Z` |
| `source` | Which platform | `youtube` or `apple` |
| `rank` | Place in the list at that moment (1 is the top) | Integer, 1-30 on YouTube, 1-100 on Apple |
| `item_id` | Id of the video or the song on its platform | Text |
| `title` | Video title or song name | Text (UTF-8, can be Hebrew) |
| `artist` | Channel name on YouTube, artist name on Apple | Text (UTF-8) |
| `artist_id` | Id of the channel on YouTube / of the artist on Apple | Text |
| `released` | When the video was uploaded / the song was released | ISO 8601. YouTube has date and time in UTC, Apple has only a date (`YYYY-MM-DD`) |
| `views` | Views at that moment | Integer, number of views. Empty for Apple |
| `likes` | Likes at that moment | Integer. Empty for Apple, or if the channel hides likes |
| `comments` | Comments at that moment | Integer. Empty for Apple, or if comments are off |
| `genre` | Main genre | Text, e.g. `Pop`. Empty for YouTube, and empty if Apple gives only "Music" |
| `language` | Language of the song name (not the artist) | `he` if there are Hebrew letters, `ar` if Arabic letters, `en` if only English letters, `other` otherwise |
| `song_key` | The song name only, cleaned: on YouTube, the part of the title after the first " - " (titles are usually "Artist - Song"); on Apple, `name`. In both we remove anything in brackets ("(Prod. By ...)", "[Official Video]", "(לייב ...)"), "feat./ft./&", punctuation and extra spaces, and lowercase it | Text. Used to match the same song on both platforms |

## 4. Why these columns

We kept what we need to answer our question: when a song shows up on each list (`snapshot_utc`, `rank`), what song it is and who sings it (`title`, `artist`, `song_key`), how new it is (`released`), and how fast it gets views on YouTube (`views`, `likes`, `comments`). `genre` and `language` are for checking if Hebrew songs or certain genres behave differently.

What we didn't keep, and why:

- `kind`, `etag` (YouTube) and `kind` (Apple, always `songs`) are technical and always the same.
- `defaultAudioLanguage` - some channels don't fill it in (empty in some of our samples), and Apple doesn't have it at all. Instead we compute `language` the same way for both platforms from the letters in the song name. We use only the song name and not the artist, because channel names mix languages (like "Young Buta | יאנג בוטה") and would give wrong results. We added `ar` because 4 of the 30 videos in one sample were in Arabic.
- We match songs by song name and not by artist, because in our samples the artist is written differently on each platform ("מור - הערוץ הרשמי" vs "Mor") while the song name is in Hebrew on both ("מטריקס").
- We do keep `channelId` / `artistId` (as `artist_id`). They don't help match songs between YouTube and Apple, but inside each platform they tell apart two songs with the same name, and they stay the same even if a channel changes its name.
- `artworkUrl100`, `url`, `artistUrl` and the genre links/ids - links and pictures, not data we analyze.
- From Apple's `genres` we keep only the first genre that isn't the general "Music" (genre 34), which appears on almost every song.
- The Apple chart info (`title`, `author`, `links`, `copyright`, `country`, `icon`) is the same in every sample. `updated` turned out to be just the time of our request (in the sample from 11:10 it says 11:10:35), so it doesn't tell us when Apple actually updated the chart.

## 5. Field names: source → schema

| Column | YouTube field | Apple field |
|---|---|---|
| `snapshot_utc` | not in the data - from the file path `raw/YYYY-MM-DD/HHMM.json` | same |
| `source` | constant `youtube` | constant `apple` |
| `rank` | position in `items` (index + 1) | position in `feed.results` (index + 1) |
| `item_id` | `id` | `id` |
| `title` | `snippet.title` | `name` |
| `artist` | `snippet.channelTitle` | `artistName` |
| `artist_id` | `snippet.channelId` | `artistId` |
| `released` | `snippet.publishedAt` | `releaseDate` |
| `views` | `statistics.viewCount` (string → integer) | - |
| `likes` | `statistics.likeCount` (string → integer) | - |
| `comments` | `statistics.commentCount` (string → integer) | - |
| `genre` | - | first `genres[].name` that isn't "Music" |
| `language` | computed from the song name in `snippet.title` (part after the first " - ") | computed from `name` |
| `song_key` | computed from `snippet.title` (part after the first " - ") | computed from `name` |

## 6. Expected size

Measured on our 4 samples: a file with both platforms is about **107KB** (106,914 bytes on average). Inside it, the YouTube part is about 15KB and the Apple part about 92KB. A run without Apple is about 15KB.

We plan to collect for about 10 weeks (70 days).

**raw/:**
- Per day: 4 runs with both (4 × 107KB) + 8 runs with YouTube only (8 × 15KB) = about 550KB
- Semester: 550KB × 70 ≈ **38MB**, far below the 500MB limit.

**dataset.csv:**
- Rows per day: 12 × 30 (YouTube) + 4 × 100 (Apple) = 760
- Semester: 760 × 70 ≈ 53,000 rows
- A row is about 250 bytes (titles and artists in Hebrew take 2 bytes per letter)
- Total: 53,000 × 250 ≈ **13MB**

## Limitations

- Matching songs between YouTube and Apple won't be perfect. YouTube titles are written however the channel wants (in our first sample the same song is "מור  - מטריקס (Prod. By Jo Cohen)" on YouTube and "מטריקס" by "Mor" on Apple). Some titles won't follow "Artist - Song", some songs are written in English on one platform and Hebrew on the other, and two different songs with the same name could match wrong. We'll check the matches of the top songs by hand.
- The two lists aren't measured the same way: YouTube gives 30 videos and doesn't say how it ranks them, Apple gives 100 by plays.
- Apple's `updated` field doesn't show when the chart really changed, so on the Apple side we only know the chart changed somewhere between two samples.
- Not everything in YouTube's music category is a song (live shows, compilations), so some videos won't have a match on Apple.
- The language check is simple and will get mixed or transliterated titles wrong.
- If a run fails we don't fill the gap, it stays missing.
- We still need to check how the YouTube Developer Policies about storing API data apply to keeping raw files for the whole semester.
