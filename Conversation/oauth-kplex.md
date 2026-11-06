<img src="https://r2cdn.perplexity.ai/pplx-full-logo-primary-dark%402x.png" style="height:64px;margin-right:32px"/>

# take this, and begin refining it into milestones, projects and RFCs + tasks in [https://github.com/orgs/HermesKPlex/projects](https://github.com/orgs/HermesKPlex/projects)

Creating the initial roadmap and planning draft in HermesKPlex

<span style="display:none">[^1_1]</span>

<div align="center">⁂</div>

[^1_1]: kplex.md


---

# yes

**Created** the approved roadmap tracking issue:

[HermesKPlex/HermesKPlex \#1 — Companion milestones, workstreams, RFCs and initial backlog](https://github.com/HermesKPlex/HermesKPlex/issues/1)

## What’s recorded

The issue turns the research into a proposed delivery baseline with:

- Six milestones, M0–M5, each with exit criteria and dependencies.
- Five workstreams covering foundation, transport, Hermes integration, Desktop workflows, and release engineering.
- Six RFC outlines covering architecture, CLI contracts, authorization, tool hooks, Desktop UX, and packaging.
- 33 initial checklist tasks grouped by milestone.
- Refinement rules for promoting those tasks into individual issues with acceptance criteria and evidence requirements.

The plan follows the document’s later CLI-first companion direction: original Obsidian continues hosting K-Plex, while Hermes supplies bounded tools and companion UI. Earlier renderer-extraction and independent graph-runtime proposals are excluded from the initial delivery path.[^2_1]

## What remains pending

Only the roadmap issue has been created. Organization Projects, actual GitHub milestones, individual RFC issues, and individual task issues have not yet been created.

The tracking issue identifies the first refinement batch: the foundational RFCs, version/source baselines, synthetic fixture design, and the acceptance specification for one safe end-to-end search-and-evidence query. No repository code or vendor files were changed.

<div align="center">⁂</div>

[^2_1]: kplex.md


---

# continue, refine more

Creating the refined RFC-002 issue under roadmap \#1

---

# yes

The RFC creation was **blocked** by GitHub’s organization access policy. RFC-002 was not created, and no repository changes were made.

GitHub reports that `HermesKPlex` has enabled OAuth App access restrictions, preventing this connector from creating the issue—even though the earlier roadmap write succeeded.

## What needs to change

An organization owner needs to approve the GitHub connector’s OAuth app for `HermesKPlex`. GitHub documents the relevant controls in its [organization OAuth app access guidance](https://docs.github.com/articles/restricting-access-to-your-organization-s-data/).

The approved RFC draft remains in this conversation, including its module boundaries, resource budgets, eight implementation tasks, acceptance matrix, and unresolved decisions. Its intended location is a child issue of [roadmap \#1](https://github.com/HermesKPlex/HermesKPlex/issues/1).

Once organization access is approved, reply “retry” so I can attempt the same approved creation without changing its content or target.

---

# retry

**Created** successfully:

[RFC-002 — Bounded Obsidian CLI transport and K-Plex public API adapter (\#2)](https://github.com/HermesKPlex/HermesKPlex/issues/2), attached as a child of [roadmap \#1](https://github.com/HermesKPlex/HermesKPlex/issues/1).

The RFC contains:

- Seven module boundaries and a ten-step request lifecycle.
- Proposed queue, timeout, output-size, and retry limits.
- Scope-isolation, cancellation, security, and privacy requirements.
- Unit, fake-process, and native acceptance criteria.
- Eight implementation checklist tasks with dependencies.
- Definitions of ready and done, alternatives, and unresolved decisions.

Its status remains Draft: creating it does not accept the architecture or authorize implementation. No code, vendor files, Project boards, or milestone objects were changed.

---

# also create [https://github.com/HermesKPlex/HermesKPlex/projects](https://github.com/HermesKPlex/HermesKPlex/projects), separate projects, and continue

Creating a separate project structure for the proposed execution plan

---

# how can I configure the Repo I see nothing in the GitHub OAuth

You do not need another repository OAuth setting to fix the current Projects limitation. There are two separate issues: GitHub authorization and the operations exposed by this connector. **Permissions** alone cannot add missing connector tools.

The earlier authorization problem appears resolved: the retry successfully created [RFC-002 as issue \#2](https://github.com/HermesKPlex/HermesKPlex/issues/2). The remaining blocker is that this session’s GitHub connector exposes issue operations, but no operations to create or manage GitHub Projects.

## Where to check GitHub authorization

If you are looking under “Developer settings → OAuth Apps,” that is not the list of third-party apps you have authorized. Use:

1. Your GitHub profile picture → Settings.
2. Integrations → Applications.
3. Authorized OAuth Apps.
4. Open the app used for the connection and inspect its organization access. GitHub documents this location for reviewing authorized apps and requesting organization approval.[^7_1][^7_2]

For organization-level approval, an organization owner uses:

1. Open the `HermesKPlex` organization.
2. Settings → Third-party Access → OAuth app policy.
3. Find the application → Review → Grant access, if approval is pending.[^7_3]

OAuth app organization approval is not configured per repository. GitHub explicitly distinguishes that from GitHub App installation controls, which can be repository-specific.[^7_4]

## Why you may not see a Projects permission switch

OAuth permissions are scopes requested by the application during authorization—not arbitrary switches you can add in repository settings. GitHub defines `project` for read/write access to user and organization Projects, and `read:project` for read-only access. An application must request the appropriate access.[^7_5][^7_6]

However, even an appropriately authorized connection would still need to expose Project-management operations to me. Our current connector does not. Reauthorizing it or broadening access is therefore not an established fix, and there is no reason to disable organization-wide OAuth restrictions just for this.

## What this means for our plan

- Repository issue creation works.
- Actual Project creation and item routing are unavailable through the tools exposed in this session.
- The five-project structure is drafted, but the boards have not been created.
- The separate-project planning issue is still awaiting your approval; it has not been written.

You do not need to keep hunting through repository OAuth settings. To create the boards from this conversation, we need a GitHub integration that exposes Projects operations; otherwise, the boards must be created through GitHub’s interface while I continue refining the RFCs and task issues here.

<span style="display:none">[^7_10][^7_11][^7_12][^7_13][^7_14][^7_15][^7_7][^7_8][^7_9]</span>

<div align="center">⁂</div>

[^7_1]: https://docs.github.com/en/account-and-profile/how-tos/organization-membership/requesting-organization-approval-for-oauth-apps

[^7_2]: https://docs.github.com/en/apps/oauth-apps/using-oauth-apps/reviewing-your-authorized-oauth-apps

[^7_3]: https://docs.github.com/en/organizations/managing-oauth-access-to-your-organizations-data/approving-oauth-apps-for-your-organization

[^7_4]: https://docs.github.com/en/organizations/managing-programmatic-access-to-your-organization/limiting-oauth-app-and-github-app-access-requests-and-installations

[^7_5]: https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/scopes-for-oauth-apps

[^7_6]: https://docs.github.com/en/apps/oauth-apps/using-oauth-apps/authorizing-oauth-apps

[^7_7]: https://docs.github.com/en/organizations/managing-oauth-access-to-your-organizations-data/about-oauth-app-access-restrictions

[^7_8]: https://docs.github.com/apps/oauth-apps

[^7_9]: https://docs.github.com/en/organizations/managing-oauth-access-to-your-organizations-data/enabling-oauth-app-access-restrictions-for-your-organization

[^7_10]: https://docs.github.com/en/organizations/managing-oauth-access-to-your-organizations-data/disabling-oauth-app-access-restrictions-for-your-organization

[^7_11]: https://docs.github.com/en/organizations/managing-oauth-access-to-your-organizations-data/denying-access-to-a-previously-approved-oauth-app-for-your-organization

[^7_12]: https://docs.github.com/api/article/body?pathname=/en/apps/oauth-apps/building-oauth-apps/scopes-for-oauth-apps

[^7_13]: https://docs.github.com/en/enterprise-server@latest/apps/building-oauth-apps/understanding-scopes-for-oauth-apps

[^7_14]: https://docs.github.com/apps/building-oauth-apps

[^7_15]: https://docs.github.com/en/apps/oauth-apps/using-oauth-apps/connecting-with-third-party-applications

