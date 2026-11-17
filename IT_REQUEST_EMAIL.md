**Subject:** Request: Entra app registration for internal "Investment Funnel" web app (SPA, delegated Graph)

Hi team,

I'm launching a small internal web app, **Investment Funnel**, that my team will use to submit and rank investment ideas. It's a static page hosted on GitHub Pages. It signs users in with their work account (MSAL.js, authorization code flow with PKCE) and reads and writes one SharePoint list through Microsoft Graph, acting as the signed-in user. There's no backend and no client secret.

Could you create an app registration with these settings?

- **Name:** Investment Funnel
- **Supported account types:** Single tenant (this organization only)
- **Platform:** Single-page application (SPA)
- **Redirect URI:** `https://sethobrechtmck.github.io/investment-funnel/` (exact, with the trailing slash). Optional for testing: `http://localhost:8080/`
- **Implicit grant:** off (neither access nor ID tokens). **No client secret or certificate.**
- **API permissions (Microsoft Graph, delegated), least privilege:**
  - `Sites.Selected`, with admin consent, **plus** a **write** grant for this app on one site only: `https://<tenant>.sharepoint.com/sites/<site>`
    (e.g. `Grant-PnPAzureADAppSitePermission -AppId <client-id> -DisplayName "Investment Funnel" -Site <site-url> -Permissions Write`)
  - If you don't use Sites.Selected, delegated `Sites.ReadWrite.All` with admin consent would also work. Users stay limited to their own SharePoint permissions either way.
- **Optional:** set Assignment required = Yes and assign the group `<team security group>`.

Once it's set up, please send me the **Application (client) ID** and **Directory (tenant) ID**, and tell me which permission option you used. Neither ID is a secret. They go into the page's config.

The README has full setup notes, and I'm happy to share it or walk through the design.

Thanks,
Seth
