# Data source

Gili Barak, Gal Sasson

## What we want to find out

We want to see where new songs in Israel become hits first, on YouTube or on Apple Music, and how long it takes a song to show up on the other one. We also want to check if Hebrew songs and English songs behave differently.

## Sources

**1. YouTube Data API**

```
https://www.googleapis.com/youtube/v3/videos?part=snippet,statistics&chart=mostPopular&regionCode=IL&videoCategoryId=10&maxResults=50&key=API_KEY
```

Returns the 50 most popular music videos in Israel right now, with views, likes and comments for each one. We keep only the fields we need: id, title, channel, publish date, views, likes, comments.

The list changes a few times a day, so we sample every 2 hours. It needs an API key (free, we opened one on Google Cloud). The key is kept outside the project folder with `chmod 600`.

**2. Apple Music top 100 in Israel**

```
https://rss.marketingtools.apple.com/api/v2/il/music/most-played/100/songs.json
```

The 100 most played songs on Apple Music in Israel. No key needed. The chart updates about once a day, but we sample every 6 hours so we know more or less when the update happened.

**Size:** a YouTube sample is around 10-15KB and an Apple sample around 40KB. For the whole semester that's about 35MB total.

## Why nobody else has this

YouTube only shows what's popular right now and the current view count. There is no way to go back and see how many views a song had last Tuesday, or when exactly it got into the list. There might be sites that archive the Apple chart, but we didn't find anyone who saves both charts side by side, and that comparison is what our project is about.

## Raw files

Every run of `collect.sh` saves one file to `raw/YYYY-MM-DD/HHMM.json` (UTC). The file has both responses in it. When it's not an Apple run (Apple is only every 6 hours), the `apple` part is `null`. We did it this way so two files never get the same name.

## dataset.csv

Each row is one song/video in one snapshot.

- `snapshot_utc` - when we collected it (UTC), taken from the file path
- `source` - `youtube` or `apple`
- `rank` - place in the chart at that moment (1 is the top)
- `item_id` - the video id on YouTube, or the song id on Apple
- `title` - video title or song name
- `artist` - channel name on YouTube, artist name on Apple
- `released_utc` - when the video was uploaded / when the song was released
- `views` - YouTube views at that moment (empty for Apple)
- `likes` - YouTube likes (empty for Apple, or if the channel hides them)
- `comments` - YouTube comments (empty for Apple, or if comments are off)
- `genre` - genre from Apple (empty for YouTube)
- `language` - `he` if the title or artist has Hebrew letters, `en` if it's only English letters, `other` otherwise
- `song_key` - artist + title in lowercase, without stuff like "Official Video", so we can match the same song on both platforms

## Limitations

- Matching songs between YouTube and Apple won't be perfect. YouTube titles are written however the channel wants, sometimes in Hebrew and sometimes in English. Some songs won't match and some will match wrong.
- The two charts aren't measured the same way. YouTube gives 50 songs and doesn't say how it ranks them. Apple gives 100 by number of plays.
- Apple updates only once a day, so on that side we only know the day, not the hour.
- The language check is simple and will get mixed or transliterated titles wrong.
- Not everything in YouTube's music category is a song (live shows, compilations etc.), so some videos won't have a match on Apple.
- YouTube has a daily quota for the key. 12 calls a day is fine, but a bug could use it up.
- If a run fails we don't fill the gap, it stays missing.

## Second proposal: hits from abroad

Same two sources, but for the US and the UK in addition to Israel (`regionCode=US`/`GB` on YouTube, `us`/`gb` in the Apple URL). Here we want to check how many days it takes a song that enters the US chart to reach Israel, and which genres get here at all.

Same API key and same sampling times as the first proposal. Each run fetches three countries, so the size is about 3 times bigger (around 105MB for the semester), still under the limit. The responses look the same as in the first proposal.

We prefer the first proposal. It interests us more and it's more connected to Israel. Also, the US charts are more likely to already be archived on other sites, so the second proposal might not pass the "lost week" rule.

## Still need to check

- YouTube API terms, especially how long we're allowed to keep the data (raw/ is never deleted)
- Apple RSS terms of use
- robots.txt for both
- That both URLs actually return data for Israel, and that 10 samples in a row aren't all the same
