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

## Sample data

What we got when we ran the two URLs from our VM on 7.11.2026 (cut short).

YouTube:
```
{
  "kind": "youtube#videoListResponse",
  "items": [
    {
      "kind": "youtube#video",
      "id": "8kjAA-kDhv8",
      "snippet": {
        "publishedAt": "2026-10-29T08:00:53Z",
        "channelId": "UCqwksirKYsda-w1mGWBtPVg",
        "title": "מור - מטריקס (Prod. By Jo Cohen)",
        ...
```

Apple Music:
```
{"feed":{"title":"Top Songs","id":"https://rss.marketingtools.apple.com/api/v2/il/music/most-played/100/songs.json","author":{"name":"Apple","url":"https://www.apple.com/"},"copyright":"Copyright © 2026 Apple Inc. All rights reserved.","country":"il", ...
```

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

## Second proposal: weather forecast vs what actually happened

This one is completely different. When the weather app says "rain on Thursday" already on Sunday, how often is it right? We want to find out how many days ahead the forecast for the Tel Aviv area can actually be trusted, and if it's worse for rain than for temperature.

**1. The forecast - MET Norway (the API behind yr.no)**

```
https://api.met.no/weatherapi/locationforecast/2.0/compact?lat=32.01&lon=34.89
```

Hourly forecast for the next ~9 days for Ben Gurion airport: temperature, rain, wind, clouds. No key, but they ask that every request sends a name and contact email in the User-Agent, so we run curl with `-A "IT-course-project <our email>"`. The forecast updates about every hour, we sample every 3 hours.

**2. What actually happened - airport weather reports (METAR) from Ben Gurion**

```
https://aviationweather.gov/api/data/metar?ids=LLBG&format=json
```

Real measurements from the airport every 30 minutes: temperature, wind, clouds, visibility, and if it's raining. From aviationweather.gov (US government), no key. We sample every 30 minutes so we don't miss a report.

**Size:** a forecast sample is around 30-40KB and a METAR sample is under 1KB. For the whole semester that's about 25MB.

**Why nobody else has this:** the forecast that yr.no showed on Sunday for Thursday gets overwritten every hour, and nobody keeps it. The METAR reports are saved in archives, but that's the "reality" side and that's fine. What's new is putting what was promised next to what happened.

**Problems we already see:** the forecast is for a point and the airport is one station, so rain that falls 5 km away won't count. METAR doesn't say how much rain fell, only if it's raining at that moment, so for rain we'll compare yes/no and not amounts. There are sites that save old weather model runs (like Open-Meteo), but not the exact forecast yr.no showed people.

**Sample data** (8.11.2026, cut short):

Forecast:
```
{"type":"Feature","geometry":{"type":"Point","coordinates":[34.89,32.01,41]},"properties":{"meta":{"updated_at":"2026-11-08T11:19:44Z","units":{"air_pressure_at_sea_level":"hPa","air_temperature":"celsius","cloud_area_fraction":"%","precipitation_amount":"mm", ...},"timeseries":[{"time":"2026-11-08T11:00:00Z","data":{"instant": ...
```

METAR:
```
[{"icaoId":"LLBG","receiptTime":"2026-11-08T11:22:05.754Z","obsTime":1794136800,"reportTime":"2026-11-08T11:20:00.000Z","temp":23,"dewp":12,"wdir":280,"wspd":18,"visib":"6+","altim":1018,"metarType":"METAR","rawOb":"METAR LLBG 081120Z 28018KT 9999 BKN044 23/12 Q1018 TEMPO 8000 RA FEW030TCU BKN035","lat":32.011,"lon":34.887,"elev":35,"name":"Tel Aviv/Ben Gurion Arpt, C, IL","cover":"BKN" ...
```

## Which one we prefer

We prefer the first proposal (YouTube vs Apple Music). It interests us more and it's connected to Israel. The weather one is a strong second choice, because it's "forecast vs reality" and winter is starting so there will be rain to check.

## Still need to check

- YouTube API terms, especially how long we're allowed to keep the data (raw/ is never deleted)
- Apple RSS terms of use
- robots.txt for both
- That 10 samples in a row aren't all the same (we already checked that both URLs return data for Israel, see the samples above)
- MET Norway and aviationweather.gov terms of use
