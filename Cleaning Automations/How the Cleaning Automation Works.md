# How the Cleaning Automation Works

This explains what `Detailed Cleaning Automation.cs` (and its shorter twin, `Simplified Cleaning Automation.cs`) does to `Data to Clean/Data to Automatically Clean.csv`. Both scripts do the same cleaning. The simplified one packs the same logic into fewer lines.

## The big picture

1. The script reads every line of the CSV file.
2. It keeps the first line (the header) exactly as it is.
3. For every other line, it splits the row on commas into separate fields.
4. It sends each field to a cleaning function based on its column position:

   | Position | Column | Cleaning function |
   |---|---|---|
   | 0 | UserID | `CleanUserID` |
   | 1 | Name | `CleanName` |
   | 2 | Team | `CleanTeam` |
   | 3 | VideoWatched | `CleanVideoWatched` |
   | 4 | DateWatched | `CleanDateWatched` |
   | 5 | WatchTime | `CleanWatchTime` |
   | 6 | Feedback | `CleanFeedback` |

5. It joins the cleaned fields back together with commas and writes all the rows to a new file, `Cleaned_Data.csv`. (The simplified version saves it inside the `JSON Conversion` folder so the JSON converter can use it next.)

Every cleaning function starts the same way: if the value is blank or only spaces, it returns an empty string. Otherwise it trims extra spaces off the start and end before doing anything else.

## What each column's cleaner does

### UserID
Just trims spaces. The IDs in the file are already fine, so nothing else is needed.

### Name
Makes the first letter uppercase and every other letter lowercase.
- `jane smith` → `Jane smith`

### Team
Uses the same rule as Name: first letter uppercase, rest lowercase.
- `team a` → `Team a`
- `Team A` → `Team a`

Every team ends up in the same format, so `team a` and `Team A` now count as the same team.

### VideoWatched
Just trims spaces. A missing video stays blank.

### DateWatched
The dates were written in several formats. The script tries each of these until one fits:
- `yyyy-MM-dd` (e.g. `2026-01-05`)
- `yyyy/MM/dd` (e.g. `2026/01/07`)
- `dd-MM-yyyy` (e.g. `07-01-2026`)
- `MMM dd yyyy` (e.g. `Jan 10 2026`)

If one matches, it rewrites the date as `YYYY-MM-DD`. If none match, it leaves the original text alone. For example, `2026-13-01` stays the same because there is no 13th month.

### WatchTime
This column had the messiest data, so it gets the most steps:
1. If the value ends in `mins`, it removes that word (`30 mins` → `30`).
2. If what's left is a whole number, it takes the absolute value so negative times become positive (`-20` → `20`).
3. If it isn't a number, it looks the word up in a small dictionary of number words (`forty` → `40`).
4. If none of these work, the value becomes blank. Bad data doesn't stay in the file.

The result is that WatchTime only holds numbers (in minutes) or blanks.

### Feedback
Trims spaces, then checks the value against a list of "no real answer" values: `N/A`, `-`, `none`, `no comment`, `na`. Capitalization doesn't matter. If there's a match, it changes the value to `N/A`. Any other feedback (like `Good` or `bad`) is kept as written.

## Example: before and after

| Before | After |
|---|---|
| `102,jane smith,team a,Video 2,2026/01/07,30 mins` | `102,Jane smith,Team a,Video 2,2026-01-07,30` |
| `103,Mike Johnson,Team B,Video 3,07-01-2026,,Good` | `103,Mike johnson,Team b,Video 3,2026-01-07,,Good` |
| `106,Anna Kim,Team C,Video 2,2026-01-09,-20,ok` | `106,Anna kim,Team c,Video 2,2026-01-09,20,ok` |
| `109,David Clark,Team D,Video 1,2026-01-12,forty,bad` | `109,David clark,Team d,Video 1,2026-01-12,40,bad` |

## Limitations I noticed

- **Last names get lowercased.** The script only capitalizes the first letter of the entire string, so `John Doe` becomes `John doe`. Capitalizing each word would fix this.
- **Team letters get lowercased too**, so `Team A` turns into `Team a`. The output is consistent, but it doesn't look quite right.
- **Blank feedback never becomes `N/A`.** The script returns an empty string for blank values before it checks the N/A list, so the `""` and `" "` entries in that list are never used.
- **Impossible dates aren't fixed or flagged.** `2026-13-01` passes through unchanged and doesn't match the `YYYY-MM-DD` rule for a real date.
- **Rows with missing columns stay short.** Row 101 has no Feedback value at all, so its cleaned row still only has 6 fields.
- **Splitting on commas is basic.** If a feedback comment contained a comma, the row would be split in the wrong place.
- **Missing values are left blank, not filled in.** Missing teams, videos, and watch times stay empty instead of getting a default value or the row being removed.
