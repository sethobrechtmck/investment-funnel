# Investment Funnel

An internal web app for submitting, scoring and ranking investment ideas. Users sign in with their Microsoft work account. Everything is saved to a SharePoint list named **Investment Funnel** through Microsoft Graph.

- **Submit an idea:** a 5-step wizard (Header, Rationale, ROIC, Confidence, Review) with a live ROI estimate.
- **Submissions:** ranked tiles you drag to reorder (saved to the list's **Rank** column), a status filter, "Sort by score" and CSV export.
- **Detail view:** full rationale and financials, the score breakdown, status changes, and reviewer scoring (saved to **Strategic fit**, **Urgency** and **Reviewer notes**).
- Follows the system light/dark setting.

## How it works

```
Browser (GitHub Pages, static files)
  │  1. Sign in: MSAL.js, authorization code flow + PKCE (redirect)
  ▼
Microsoft Entra ID ──► access token (delegated Microsoft Graph permission)
  │  2. Read/write list items
  ▼
Microsoft Graph ──► SharePoint site ──► "Investment Funnel" list
```

- There is no server and no client secret. GitHub Pages only serves `index.html` and the bundled sign-in library (`vendor/msal-browser.min.js`, MSAL.js v4.30.0).
- The app acts **as the signed-in user**. A user can only do what their own SharePoint permissions on the list allow.
- The page itself is reachable by anyone who has the URL, but it contains no data. All data stays in SharePoint behind sign-in.

| File | Purpose |
|---|---|
| `index.html` | The app. The config block is at the top. |
| `vendor/msal-browser.min.js` | Microsoft Authentication Library for the browser, bundled with the app (no CDN dependency). |
| `Investment_Funnel_SharePoint_Setup.xlsx` | The list design: columns, types, choices and score formulas. |
| `IT_REQUEST_EMAIL.md` | Draft email asking IT for the app registration. |
| `investment-funnel.html` | The original Claude-only version, kept for reference. |

---

## 1. Create the SharePoint list from the workbook

You need permission to create lists on a SharePoint team site. Use a team site, not your OneDrive or "My lists", so the funnel doesn't depend on one person's account.

1. On the site, select **New › List › From Excel**. Upload `Investment_Funnel_SharePoint_Setup.xlsx` and pick the table named **Submissions**.
2. Set each column's type using the **Type at import** column on the workbook's **Column setup** tab. Keep **Title** as Title.
3. Name the list **Investment Funnel** and select **Create**. The three example rows let you check the scores. Delete them before launch.
4. Finish the columns, following the **Column setup** tab:
   - **Choice columns** (Type of ask, Mandatory, Estimate confidence, Execution risk, Status): paste the full choice list. The app writes these exact values. **Mandatory** must offer `Yes - required` and `No - discretionary`.
   - **Long-answer columns** (Reason for the ask, Capabilities unlocked, Key assumptions, Cost of inaction, Success metrics, Reviewer notes): change to **Multiple lines of text**. Plain text is best. Rich text also works, and the app converts it.
   - **Date submitted**: Date only, default Today. **Status**: default Submitted.
   - **Strategic fit** and **Urgency**: Number, 0 decimals, min 1, max 5. **Rank**: Number, 0 decimals.
   - Turn on **Require that this column contains information** where the tab says Required.
5. Optional: add the calculated score columns from the **Score formulas** tab, in order. The app doesn't need them because it calculates the same scores in the browser, but they make sorted views and Excel exports in SharePoint show the same numbers as the app.
6. Optional: create the views from the workbook's **Start here** tab (Ranked, By score, Pipeline board).

### Column names matter; internal names don't

At sign-in, the app reads the list's columns from Graph and matches them by **display name**. Then it uses each column's real internal name, which can look like `Typeofask` or `field_3` after an Excel import. If you rename a column in SharePoint, either change it back or update `FUNNEL_COLUMNS` in the config block. If a column is missing, the app lists it on an error screen.

| App field | List column (display name) | Notes |
|---|---|---|
| Title | Title | |
| Type of ask | Type of ask | Choice |
| Mandatory | Mandatory | Choice `Yes - required` / `No - discretionary` (a Yes/No column also works) |
| Submitted by | Submitted by | Pre-filled with the signed-in user's name |
| Executive sponsor | Executive sponsor | |
| Date submitted | Date submitted | |
| Reason, Capabilities, Assumptions, Strategic priority, Cost of inaction, Success metrics | Same names as in the workbook | |
| One-time, Ongoing, Y1 revenue, Y1–Y3 GP, Savings | One-time investment, Annual ongoing cost, Y1 revenue, Y1 GP, Y2 GP, Y3 GP, Annual cost savings | Currency |
| Estimate confidence / Execution risk | Estimate confidence / Execution risk | Choice |
| Status | Status | Choice |
| Reviewer strategic fit / urgency | Strategic fit / Urgency | Number 1–5 |
| Reviewer comments | Reviewer notes | Each review adds a dated line |
| Drag-to-rank position | Rank | 1 = top |
| Idea ID (INV-001…) | *(SharePoint item ID)* | Shown as `INV-` + the item ID |

### List permissions

The app can do only what the user can do in SharePoint:

| Who | SharePoint permission on the list | What works in the app |
|---|---|---|
| Everyone who submits ideas | **Contribute** or **Edit** | Submit ideas, see all ideas |
| Reviewers | **Edit** | Also score, rank, change status, delete |
| View-only users | **Read** | See ideas and export CSV. Saving anything shows a "view-only access" message. |

Ranking and scoring update other people's items, so the people who do them need Edit on the whole list.

---

## 2. Microsoft Entra app registration (for IT)

`IT_REQUEST_EMAIL.md` has a ready-to-send request. The settings IT needs:

1. **Microsoft Entra admin center › App registrations › New registration**
   - Name: `Investment Funnel`
   - Supported account types: **Accounts in this organizational directory only (single tenant)**
   - Redirect URI: platform **Single-page application (SPA)**, URI **`https://sethobrechtmck.github.io/investment-funnel/`**
     - Use this exact URL, including the trailing slash. It must match the address users open.
     - Optional for local testing: also add `http://localhost:8080/`.
2. **Authentication**: leave both *Implicit grant* boxes (access tokens, ID tokens) **unchecked**. The app uses the authorization code flow with PKCE. Leave *Allow public client flows* set to No.
3. **Certificates & secrets**: none. A single-page app can't keep a secret, so don't create one.
4. **API permissions** (Microsoft Graph, **Delegated**). Choose one option:
   - **Option A, least privilege (recommended): `Sites.Selected` (delegated).** Grant admin consent. Then give the app **write** access to the one site that holds the list. Until this step is done, users get "site not found" or "no access":
     - PnP PowerShell:
       `Grant-PnPAzureADAppSitePermission -AppId <client-id> -DisplayName "Investment Funnel" -Site https://<tenant>.sharepoint.com/sites/<site> -Permissions Write`
     - Or Microsoft Graph (an admin with `Sites.FullControl.All`):
       `POST https://graph.microsoft.com/v1.0/sites/{site-id}/permissions`
       `{ "roles": ["write"], "grantedToIdentities": [{ "application": { "id": "<client-id>", "displayName": "Investment Funnel" } }] }`
     - The app then reaches only that site, and each user is still limited by their own permissions on the list.
   - **Option B, simpler: `Sites.ReadWrite.All` (delegated).** Many tenants require admin consent for this as well. Users are still limited by their own SharePoint permissions, but the token isn't restricted to one site.
   - `openid`, `profile` and `offline_access` are requested automatically for sign-in. `User.Read`, which is added by default, isn't used by the app and can be removed.
5. Optional: **Enterprise applications › Investment Funnel › Properties › Assignment required = Yes**. Then assign the team's security group under **Users and groups**, so only that group can sign in.
6. Send back the **Application (client) ID** and **Directory (tenant) ID** from the Overview page. If you used Option A, say so. Optionally, also send the Graph site ID.

---

## 3. Fill in the config

Open `index.html`. The config block is at the very top, marked `CONFIG — fill these in`:

```js
window.FUNNEL_CONFIG = {
  tenantId: '00000000-0000-0000-0000-000000000000',      // Directory (tenant) ID
  clientId: '11111111-1111-1111-1111-111111111111',      // Application (client) ID
  siteUrl:  'https://contoso.sharepoint.com/sites/FinanceLeadership',
  siteId:   '',                                          // optional; overrides siteUrl
  listName: 'Investment Funnel',
  scopes:   ['Sites.Selected'],                          // or ['Sites.ReadWrite.All'] if IT chose Option B
  redirectUri: ''                                        // '' = this page's address (recommended)
};
```

- **siteUrl** is the site's home address, not the list's address. If you paste a list or page URL, the app trims it back to the site.
- **scopes** must match the permission IT granted. If they differ, sign-in fails with a consent error.
- **redirectUri**: leave it empty. The app uses its own address (for example `https://<owner>.github.io/<repo>/`), and that address must be registered as an SPA redirect URI.
- None of these values are secrets. They're safe to commit to the repository.

Commit and push. GitHub Pages republishes within a minute or two.

## 4. Publish on GitHub Pages

1. Push this folder to a GitHub repository. `index.html` and `vendor/` must be at the root of the published branch.
2. Go to **Settings › Pages › Build and deployment**, set Source to **Deploy from a branch**, and choose branch `main`, folder `/ (root)`.
3. The site is published at `https://<owner>.github.io/<repo>/`. That address is the redirect URI IT registers.

Note: on most GitHub plans, a Pages site is **public on the internet even if the repository is private**. That's acceptable here because the page holds no data or secrets, and data access needs Microsoft sign-in. If your organization has GitHub Enterprise Cloud, Pages can be restricted to organization members. Check your company's policy on publishing internal tools to GitHub.

## Testing locally

Sign-in doesn't work when you open the file directly (`file://`). Serve the folder over http and register the localhost address as an SPA redirect URI, for example:

```bash
npx http-server -p 8080
```

Then open `http://localhost:8080/`.

## Behavior notes

- **Scores** are calculated in the browser with the same weights and formulas as the original app and the workbook's calculated columns: financial 35%, strategic fit 25%, urgency 15%, risk & confidence 25%. Strategic fit and urgency count as 3 (Provisional) until both are set.
- **Reviews.** The list has one Strategic fit and one Urgency column per idea, so a new review **replaces** the current scores. The original app averaged all reviews. Each review also appends a line like `2026-11-17 · Jane Doe · fit 4, urgency 3` plus the note to **Reviewer notes**, which keeps the history.
- **Ranking.** Dragging a tile or selecting "Sort by score" writes 1…n to **Rank**, updating only the items whose position changed (Graph batch requests). Ideas without a Rank appear after ranked ones, oldest first. New ideas go to the bottom. If two people reorder at the same moment, the last save wins.
- **Refresh.** The Submissions tab reloads every 30 seconds and when you return to the tab.
- **Drafts.** If the sign-in session expires mid-form, the draft is kept in the browser tab and restored after you sign in again.

## Troubleshooting

| Message in the app | Likely cause and fix |
|---|---|
| "This app isn't set up yet" | Config values are still placeholders, the page was opened as a local file, or `vendor/` wasn't published. |
| "This page's address isn't registered for sign-in" (AADSTS50011) | The redirect URI in Entra doesn't exactly match the page URL. Check the trailing slash and capitalization of the repo name. |
| "App registration uses the wrong platform" (AADSTS9002326) | The redirect URI was added under **Web** instead of **Single-page application**. |
| "Admin consent needed" (AADSTS65001) | IT needs to select **Grant admin consent** for the Graph permission. |
| "You're not assigned to this app" (AADSTS50105) | Assignment is required and the user isn't in the assigned group. |
| "SharePoint site not found" / "You don't have access to this list" | `siteUrl` is wrong, the user doesn't have access to the site or list, or (with Sites.Selected) IT hasn't granted the app write access to the site yet. |
| "The list is missing some columns" | A column's display name doesn't match the workbook. Rename it or update `FUNNEL_COLUMNS`. |
| "…not saved: your account has view-only access" | The user has Read on the list. Give them Contribute/Edit. |

The error screens show an error code, request ID and timestamp. Include them when contacting IT.

## Updating MSAL.js

`vendor/msal-browser.min.js` is `@azure/msal-browser` v4.30.0 (UMD build). To update it, download `lib/msal-browser.min.js` from the npm package of the version you want and replace the file. Stay on 4.x unless you also review the v5 migration guide.
