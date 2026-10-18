# LAFC Gold Matchday — score refresh runbook

Artifact: https://claude.ai/artifact/K25jLgw1KTXypy14CYpWsY
Owner: Pablo (pablo@vibesolar.com). Team: LAFC So Cal ECNL RL S.Cal B2013/14 GOLD (team id 7 in app data).

## Where the official data lives
ECNL schedules/standings are served by Total Global Sports / AthleteOne. For
ECNL RL Boys Southern Cal 2026-27, age BU13:

    https://api.athleteone.com/api/Script/get-conference-schedules/16/83/4354/41628/0

(16 = org, 83 = season, 4354 = event "ECNL RL Boys Southern Cal 2026-27", 41628 = BU13 division.)
It returns an HTML table (~390 KB) with every fixture: game id, date, time, two
teams (home first) with data-team-id, venue, and two score cells when final.
The server only answers requests coming from a browser on theecnl.com, so the
cloud fetch tools fail. Fetch it from inside the ECNL page in the Claude
browser (mcp__remote-devices__Claude_Browser__*), running on Pablo's computer.

Box scores (lineups; goals/assists are NOT tracked by the league):
    https://api.athleteone.com/api/Script/get-box-score/4354/<matchId>/<homeTeamId>

## Refresh procedure
1. Open https://theecnl.com/sports/2023/8/8/ECNLRLB_0808235620.aspx in the Claude browser
   (preview_start). Wait ~5 s. Dismiss the cookie notice if present (click its ×).
2. Run this with javascript_tool (action javascript_exec); it returns the feed text:

```js
const r=await fetch('https://api.athleteone.com/api/Script/get-conference-schedules/16/83/4354/41628/0');const t=await r.text();
const rows=t.split(/<tr[\s>]/).slice(2);const out=[];const tm={};
for(const row of rows){const id=(row.match(/<td[^>]*>\s*(\d{6,8})\s*<\/td>/)||[])[1];if(!id)continue;
 const date=(row.match(/font-weight: 600; font-size: 13px;">([^<]+)</)||[])[1].trim();const time=((row.match(/font-size: 12px; padding: 10px 0;">([^<]+)</)||[])[1]||'').trim();
 const teams=[...row.matchAll(/individual-team-item"[^>]*data-team-id="(\d+)"[^>]*>([^<]+)</g)].map(m=>({tid:m[1],name:m[2].trim()}));teams.forEach(x=>tm[x.tid]=x.name);
 const venue=((row.match(/game-complex-item"[^>]*>([^<]+)</)||[])[1]||'').trim();
 const det=row.split('min-width: 72px')[1]||'';const sc=[...det.matchAll(/margin-bottom: 5px">\s*(\d+)\s*<\/div>/g)].map(m=>+m[1]);
 const d=new Date(date+' 12:00');const iso=d.getFullYear()+'-'+String(d.getMonth()+1).padStart(2,'0')+'-'+String(d.getDate()).padStart(2,'0');
 out.push([id,iso,time==='12:00 AM'?'':time.replace(/^0/,''),teams[0].tid,teams[1].tid,sc.length===2?sc[0]:'',sc.length===2?sc[1]:'',venue==='-'?'':venue].join('|'))}
out.join('\n')+'\n#TEAMS\n'+Object.entries(tm).map(([k,v])=>k+'|'+v).join('\n')
```
   Sanity check: ~153 lines before `#TEAMS`, 18 team lines after. If a team name is
   unknown to the app, sync.py stops — add the team to data.teams first (new id = next integer, keep short/initials).
3. Save that text verbatim to `feed.txt` in the scratchpad (Write tool).
4. Read the live page and tools from the artifact (Artifact tool, action read, path):
   `index.html` → `page.html`; `tools/sync.py` → `sync.py`.
5. Extract the data block from page.html to data.json:
   the JSON between `<script type="application/json" id="gold-data">` and the next `</script>`
   (replace `<\/` with `</` when parsing — it is JSON-escaped). Run:
       python3 -I sync.py feed.txt data.json
   It prints a diff summary (new_finals, changed_scores, new_fixtures, moved, removed) and rewrites data.json.
6. If the summary shows no new finals, no changed scores, no new/removed/moved fixtures: do NOT publish. Report "no changes".
7. Otherwise re-insert the JSON (serialize compactly, escape `</` as `<\/`) into page.html in place of the
   old block contents, and publish it with the Artifact tool using `url` = the artifact URL and the same
   file path. Never change anything else in the page. Then re-read `index.html` once and confirm the new
   finals count appears in the data block.
8. Report: finals count, each new/changed result (date, teams, score), and anything about LAFC Gold.

## Rules
- Blank scores stay null (never 0–0). Postponed flags set in manager mode are preserved by sync.py
  unless the feed moves or scores the game.
- The pool, roster, meta.notes and meta.fullNames in data.json must never be lost; sync.py keeps them.
- If the Claude browser cannot be reached (Pablo's computer asleep/offline), try WebFetch on the feed URL once;
  if that also fails, report that the check could not run and that scores can be entered in manager mode.
