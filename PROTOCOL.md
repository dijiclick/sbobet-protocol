# SBOBET Protocol Reverse Engineering Findings

**Date**: 2026-03-11  
**Method**: Static JS analysis (Wayback Machine archive), C# odds-grabber source, live HTML page scraping  
**Status**: ✅ Protocol fully documented — no WebSockets found (polling-based HTTP API)

---

## Summary

SBOBET uses a **polling-based HTTP API** (NOT WebSockets). The frontend polls a single endpoint every ~30–60 seconds. The response is JavaScript code that gets `eval()`'d client-side, triggering `P.onUpdate()` callbacks.

---

## 1. Authentication & Session Tokens

### Initial Page Load

When the page loads, two `tilib_Token` objects are created and embedded in the HTML:

```javascript
// Site-level config token
P.setToken('site', new tilib_Token(
  ['ser','dt','ts','lang','euro','seo','asi','uid','rid','sound','tz','sort',
   'info-site-domain','asia-site-domain','bigevent-rid','eufaeuro-rid',
   'wcmt','wconoff','o-token','ds','isNewTV','noLiveScoreSports',
   'euro_cup_2016_enabled','isIomB2c','isHideCricket'],
  ['B03','202603120209','c792b145','en','euro',1,0,'0',145,0,0,1,
   'info.sbobet.com','sports.sbobet.com',0,338,'UEFA Euro 2020',
   'n',0,,1,'','n','n','y']
));

// Odds display token
P.setToken('od', new tilib_Token(
  ['oev','sev','pid','sid','rid','tid','eid','cd','dt','mt','ms','ps'],
  [4587,0,11,1,0,0,0,'20260311',0,0,0,4]
));
```

### Token Fields

**Site token** key fields:
| Field | Value | Description |
|-------|-------|-------------|
| `ts` | `c792b145` | Session timestamp (8-char hex) — **required in all requests** |
| `ser` | `B03` | Server code |
| `dt` | `202603120209` | Date-time string |
| `lang` | `en` | Language code |
| `uid` | `'0'` | User ID (0 = anonymous) |
| `rid` | `145` | Region ID |

**Odds token** (od) key fields:
| Field | Value | Description |
|-------|-------|-------------|
| `oev` | `4587` | Odds event version (increments on changes) |
| `sev` | `0` | Server event version |
| `pid` | `11` | Page type (11=football, 21=live, etc.) |
| `sid` | `1` | Sport ID |
| `rid` | `0` | Region ID |
| `tid` | `0` | Tournament ID |
| `eid` | `0` | Event ID |
| `cd` | `'20260311'` | Current date YYYYMMDD |
| `dt` | `0` | Date tab (0=today, 1=tomorrow, etc.) |
| `mt` | `0` | Market tab |
| `ms` | `0` | Market sub-tab |
| `ps` | `4` | Odds format (4=Malaysian, 1=Euro, etc.) |

### `getToken()` Returns

`tilib_Token.getToken()` simply returns `_values.join(',')`.  
So for the od token: `"4587,0,11,1,0,0,0,20260311,0,0,0,4"`

---

## 2. Main Odds Data Endpoint

### URL Pattern

```
GET https://www.sbobet.com/{lang}/data/event?ts={ts}&tk={od_token}&ac={action}
```

**Example:**
```
GET https://www.sbobet.com/en/data/event?ts=c792b145&tk=4587,0,11,1,0,0,0,20260311,0,0,0,4&ac=1
```

### Parameters

| Param | Description |
|-------|-------------|
| `ts` | Session timestamp from site token |
| `tk` | Full od token (12 comma-separated values) |
| `ac` | Action type (see below) |
| `ex` | Hash (for bookmark navigation, optional) |

### Action Codes (`ac`)

| `ac` | Trigger |
|------|---------|
| `1` | Full reload (new page/sport/region) |
| `2` | Date tab changed |
| `3` | Market type changed |
| `4` | Match status changed |
| `5` | Odds style changed |

### Headers Required

```
Cookie: {SBOBET session cookies}
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36
X-Requested-With: XMLHttpRequest (for some requests)
```

### Polling Interval

- **Standard pages**: 60 seconds
- **Live betting pages** (pid=21): ~30 seconds  
- **Mixed parlay**: 90 seconds

---

## 3. Response Format

The response body is **raw JavaScript code** that gets `eval()`'d:

```javascript
// Normal update
P.onUpdate('od', [
  4588,           // [0] New oev (event version)
  0,              // [1] 1=full reload, 0=incremental  
  [[...], ...],   // [2] Cache entries array
  {...},          // [3] Display counts by sport
  {...},          // [4] Market counts
  false           // [5] Display submarket flag
]);

// Session expired — must reload page to get new ts token
P.onReload('ts');
```

### Cache Entry Structure (`data[2]`)

Each cache entry: `[cacheType, matchesArray, marketsArray, deletions]`

| Cache Type | Content |
|-----------|---------|
| `1` | Standard pre-match events |
| `2` | Today's events |
| `3` | Tomorrow's events |
| `4` | Outright coupons (home wins) |
| `5` | Outright coupons (exact score) |
| `6` | All-sports view |

---

## 4. Match/Event Data Structure

Within `data[2][n][1]` (matches array):

```javascript
// Each match entry: [sportId, tournamentId, matchData, marketData, betTypes]
[
  sportId,         // [0] Sport ID
  tournamentId,    // [1] Tournament ID
  [                // [2] Match data array
    eventId,       // [0] Unique event/match ID
    homeName,      // [1] Home team name
    awayName,      // [2] Away team name
    // ... 
    kickoffDate,   // [5] "MM/dd/yyyy HH:mm" format
    // ...
    homeScore,     // [7] Current home score
    awayScore,     // [8] Current away score
    // ...
    [              // [3] or [11] Live time data
      period,      // [1] Period: 1=1H, 2=2H, 3=1ET, 4=2ET, 5=HT
      elapsedMins, // [2] Minutes elapsed in this period
      totalMins,   // [3] Total minutes in period
      // ...
      statusText,  // [7] "HT", "FT", etc.
      injuryTime   // [9] Added time minutes
    ]
  ],
  marketData,      // [3] Market/odds data
  betTypes         // [4] Available bet types
]
```

### Market/Bet Type IDs

| ID | Type | Fields |
|----|------|--------|
| `1` | Full Time Handicap (FT HDP) | `[1][5]`=handicap line, `[2][0]`=home odds, `[2][1]`=away odds |
| `2` | Full Time Odd/Even | `[2][0]`=odd odds, `[2][1]`=even odds |
| `3` | Full Time Over/Under | `[1][5]`=OU line, `[2][0]`=over, `[2][1]`=under |
| `5` | Full Time 1X2 | `[2][0]`=1 odds, `[2][1]`=X odds, `[2][2]`=2 odds |
| `7` | First Half Handicap | Same as FT HDP |
| `8` | First Half 1X2 | Same as FT 1X2 |
| `9` | First Half Over/Under | Same as FT O/U |

---

## 5. Live Score Endpoint

```
GET https://www.sbobet.com/en/data/live-court?token={eventId},{getNewStatistic},{mode},{rsynid}
```

**Parameters:**
- `eventId`: Match event ID
- `getNewStatistic`: `s` if requesting new stats, empty otherwise
- `mode`: Display mode (0=standard)
- `rsynid`: Sync ID (starts -1, updated from response)

**Response**: eval'd JavaScript calling `P.onUpdate('lc', [...])` with:
- `[0]` = new rsynid
- `[1]` = court animation state
- `[2]` = score data
- `[3]` = home player/team stats
- `[4]` = away player/team stats
- `[5]` = event timeline

---

## 6. Other Endpoints

| URL | Description |
|-----|-------------|
| `GET /en/data/favourite?tk={token}&act={action}&uid={uid}` | Favourite matches |
| `GET /en/rdata/userpref?ts={ts}&param={prefs}` | User preferences |
| `GET /en/rdata/bet-slip` | Bet slip data |
| `GET /en/data/event?ts={ts}&upfv=1` | Live video flash vars |
| `GET /en/page/redirect` | Personal message |
| `GET /en/data/logout` | Session logout |

---

## 7. Page ID (pid) Values for Live Betting

| pid | Type |
|-----|------|
| `11` | Football (soccer) pre-match |
| `21` | Live betting (all sports) |
| `12` | Region view |
| `13` | Tournament view |
| `14` | Outright |
| `40` | Mix Parlay |

---

## 8. Sports ID Mapping

| sid | Sport |
|-----|-------|
| `1` | Football/Soccer |
| `2` | Basketball |
| `3` | Baseball |
| `5` | Tennis |
| `8` | Volleyball |
| `9` | Badminton |
| `19` | American Football |
| `36` | E-Sports |

---

## 9. CDN / Server Infrastructure

| Domain | Role |
|--------|------|
| `www.sbobet.com` | Main site + data API |
| `m.sbobet.com` | Mobile site |
| `sports.sbobet.com` | Asia sports site |
| `info.sbobet.com` | Info/articles |
| `txt-1-3.speedysurfcdn.net` | JS/CSS CDN |
| `txt-1-79.cloudswiftcdn.net` | UK user redirect |
| `txt-1-72.cloudswiftcdn.net` | OAuth |

---

## 10. How to Get a Valid Session

1. Log in to SBOBET with credentials (username/password)
2. Navigate to `/en/football` or `/en/betting/live-betting`
3. Extract from page HTML:
   - `ts` value from `tilib_Token(..., [..., 'c792b145', ...])` (3rd value)
   - `od` token values from second `tilib_Token` call
4. Copy session cookies from browser
5. Use `ts` and constructed `tk` in API requests

**Token extraction regex:**
```javascript
// Extract ts
const tsMatch = html.match(/tilib_Token\(\[.*?'ts'.*?\],\[.*?'([a-f0-9]+)'/);
const ts = tsMatch[1];

// Extract od token
const odMatch = html.match(/P\.setToken\('od',new tilib_Token\([^)]+\),\[([^\]]+)\]\)/);
const odValues = odMatch[1].split(',');
const tk = odValues.join(','); // joins to "oev,sev,pid,sid,rid,tid,eid,cd,dt,mt,ms,ps"
```

---

## 11. No WebSocket Found

SBOBET's euro (non-Asia) platform does **not use WebSockets**. All data is fetched via:
- Standard XHR GET requests
- JSONP-style eval'd JavaScript responses  
- Polling interval: 30–60 seconds

The Asia platform (`sports.sbobet.com`) may have different architecture.

---

## 12. Rate Limiting & Anti-Bot

- SBOBET is geo-blocked (requires VPN or ISP in allowed regions)
- Cloudflare protection on CDN assets
- Session validation via `ts` token (resets on re-login)
- `P.onReload('ts')` response indicates expired session
- IP-based blocking for automated scrapers
