# Daily Fulfillment Reconciliation System

A live multi-team production tracking application built for the fulfillment floor at Boost Oxygen LLC in Milford, Connecticut.

Four team leads submit their daily numbers from any device. The fulfillment manager sees every team roll up into one dashboard in real time. Leadership gets a formatted summary email generated automatically. Nobody chases anybody for a spreadsheet.

Currently on version 12 and running in production.

---

## The Problem

Reconciliation at the end of a fulfillment day used to mean four separate people filling out four separate spreadsheets, emailing them to the manager, and the manager stitching them together by hand before writing a summary to leadership. By the time the numbers were assembled the day was over and anything that went wrong had already gone wrong.

If one team lead forgot to send theirs, the manager did not find out until he went looking. If a team was falling behind at 11 AM, nobody upstream knew until the report landed at 5 PM.

The information existed. It just arrived too late to act on.

---

## What This Does

Every team lead gets their own tab. They enter orders received, orders completed, orders pending, labels not shipped, reshipments, carrier issues, and any notes. The same day completion rate calculates itself as they type and a progress bar fills in live.

The moment they hit submit, the data lands in Google Sheets and appears on the manager dashboard. No refresh needed on his end. The dashboard polls every thirty seconds on its own.

The manager tab shows six live KPIs across all teams, a breakdown table with color coded status per team, every team message in one place, and an alert panel that fires on its own when something crosses a threshold.

When the day is done he clicks one button and gets a fully formatted email addressed to the CEO and COO with the overall numbers, the team by team breakdown, ship date changes, escalations, tomorrow's priorities, and notes. Copy, paste, send.

---

## Teams Covered

| Tab | Team | Lead | Daily Target |
|---|---|---|---|
| Stephanie | Online Set 1 | Stephanie | 1:00 PM |
| Ben | Online Set 2 | Ben | 1:00 PM |
| JL | Online Set 3 | Jonathan L. | 1:00 PM |
| Jonathan | LTL Priority | Jonathan Sr. | 1:00 PM |
| Fred Summary | All teams rolled up | Fulfillment Manager | End of day |

---

## Technical Build

**Frontend**

Single file. Vanilla HTML, CSS, and JavaScript. No framework, no build step, no dependencies. That was deliberate. A warehouse floor tool that breaks because an npm package changed is a tool nobody trusts. This one is a single file that loads on any device with a browser and keeps working.

Responsive down to phone width with breakpoints at 500px and 460px, because team leads are entering numbers from the floor, not from a desk.

**Backend**

Google Apps Script serving as the API layer, writing to Google Sheets as the operational database. No server to maintain, no hosting bill, no downtime window.

**Writes**

Submissions POST through a hidden iframe and a dynamically created form. This sidesteps the CORS restrictions that block a normal fetch call to an Apps Script endpoint from a static host. The iframe onload event fires the confirmation state, marks the tab with a check, and triggers a background refresh of the manager dashboard.

**Reads**

The dashboard pulls data using JSONP. A callback function is registered on the window object under a timestamped name, a script tag is injected pointing at the Apps Script endpoint with that callback name and a cache buster, and the response invokes it. The callback and the script tag both clean themselves up after firing.

**The refresh problem and how it got solved**

Version 1 through 8 wiped the screen on every refresh. The manager would be mid read and the whole dashboard would blank out and redraw. It made the tool feel broken even though the data was correct.

Version 12 refreshes in place. A three pixel progress bar runs across the top of the window and that is the only visual indication anything happened. The DOM updates node by node without clearing. A loading guard flag blocks overlapping fetches so a manual refresh during an automatic one cannot double fire. If the network drops, the fetch fails silently and leaves the last known good data on screen rather than showing an error state over an empty dashboard.

The refresh only runs when the manager tab is actually the visible tab. No point burning requests to update a panel nobody is looking at.

**Time handling**

Everything runs on America/New_York regardless of what timezone the device is set to. A team lead on a personal phone with the wrong timezone was submitting under the wrong date, which pushed their numbers into the next day and made them look like they had not submitted at all.

Date matching runs against three separate string formats plus a parsed Date fallback, because Google Sheets does not return dates consistently depending on how the cell was formatted when the row was written.

**Latest submission wins**

Rows are sorted by timestamp before the roll up. If a team lead submits at 1:15 PM and then resubmits corrected numbers at 2:30 PM, the 2:30 submission is what the dashboard and the email use. The earlier row stays in the sheet as an audit trail but does not affect the reported figures.

---

## Automatic Alerts

The dashboard raises a flag on its own when any of these conditions hit:

- More than 10 orders pending on any single team
- More than 3 carrier issues logged by any single team
- More than 5 labels created but not shipped

Team status color codes at 90 percent and 70 percent same day completion. Green reads All Clear, amber reads Minor Issues, red reads Needs Attention.

Nobody has to notice these. The system notices.

---

## Executive Email Generation

One button produces a complete daily report addressed to the CEO and COO. It pulls the live roll up, formats the team breakdown with per team completion percentages, appends any ship date changes, escalations, tomorrow's priorities, and free text notes the manager entered, and signs it off.

What used to be twenty minutes of assembling numbers and writing prose at the end of a long day is now a click and a paste.

---

## What Changed Operationally

Before this existed, the manager found out about a problem after the day ended. Now he sees a team sitting at 60 percent at midday and can move people before it becomes a miss.

Leadership stopped asking where the daily report was, because it arrives every day at the same time in the same format.

The four team leads stopped maintaining separate spreadsheets. They enter numbers once, on a phone, in under two minutes.

And every submission is timestamped and permanent, so a question about what happened on a specific day three weeks ago has an actual answer instead of a guess.

---

## Version History

The interesting parts of getting from version 1 to version 12.

**v1 to v4** — Basic form capture, single team, writing to Sheets. Proved the Apps Script write path worked.

**v5** — Multi team tabs. Four separate submission panels feeding one sheet.

**v6 to v7** — CORS wall. Standard fetch calls to Apps Script kept getting blocked from the static host. Solved with the hidden iframe and form POST pattern.

**v8** — Manager roll up dashboard. Worked, but wiped the screen on every load.

**v9** — JSONP read layer replacing the failed fetch approach.

**v10** — Timezone bug. Team leads on devices set to other timezones were submitting under the wrong date. Forced everything to America/New_York.

**v11** — Multi format date matching, because Sheets was returning the same date three different ways depending on the row.

**v12** — Silent in place refresh, loading guard, graceful network failure, latest submission wins logic, threshold alerts, and automatic executive email generation. Current production version.

---

## Running It

The file is fully self contained. Host it anywhere that serves static files.

1. Deploy the Google Apps Script as a web app with access set to anyone
2. Replace the `GAS` constant near the top of the script block with the deployment URL
3. Upload the HTML file to any static host

That is the entire deployment. No build, no environment variables, no package install.

---

## Design Decisions Worth Explaining

**Why no framework.** This tool lives on a warehouse floor and gets used by people who are not going to file a bug report. It needs to work every single day without maintenance. One file with zero dependencies cannot break because something upstream changed.

**Why Google Sheets as the database.** The fulfillment manager already lived in Sheets. Putting the data somewhere he could open, filter, and export without asking me for anything mattered more than having a proper database. He can pull a month of history himself.

**Why silent refresh instead of a loading spinner.** A spinner tells the user to wait. This dashboard is meant to sit open on a screen all day. It should update the way a clock updates, without asking for attention.

**Why latest submission wins instead of blocking resubmits.** People make mistakes entering numbers. Letting them fix it without calling someone is worth more than enforcing a single submission, and the audit trail is preserved in the sheet either way.

---

## Built By

**Naveen Dadi**
Boost Oxygen LLC, Milford CT

Designed, built, deployed, and maintained solo. Requirements gathered by watching the floor. Iterated in production against real users across twelve versions.

Reports to Rob Neuner, CEO and Mike Grice, COO.

Background in supply chain and business analytics. This is one of several operational systems I built in house at Boost Oxygen rather than buying software that almost fits.

Related work: [Package Scan Confirm System](https://github.com/dadinaveen1729/Bo2)
