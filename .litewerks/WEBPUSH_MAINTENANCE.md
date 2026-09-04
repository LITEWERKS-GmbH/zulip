# Litewerks Web Push downstream maintenance

This fork exists to carry a minimal downstream Web Push implementation for the LITEWERKS self-hosted Zulip deployment.

## Branch policy

- `main` must remain a clean mirror of `zulip/zulip:main`. Do not merge Litewerks-specific changes into `main`.
- `litewerks-webpush` is the production customization branch.
- The initial baseline is Zulip Server `12.2`, tag commit `1e73e1d754761b73c18135a3f25d0673f31cd8b3`.
- Keep the downstream patch small and isolated. Do not add unrelated customizations to this branch.

## Upstream relationship

Canonical upstream: `https://github.com/zulip/zulip`

This repository is a GitHub fork, so the upstream relationship is preserved by GitHub. Before every Zulip upgrade, sync/fetch upstream first and verify the target stable release/tag.

Recommended local remotes:

```bash
git remote -v
git remote add upstream https://github.com/zulip/zulip.git  # if not already present
git fetch upstream --tags
```

Keep the fork's `main` aligned with upstream `main`:

```bash
git checkout main
git fetch upstream
git reset --hard upstream/main
git push origin main --force-with-lease
```

Do not deploy `main` to production. Production should use the stable-based `litewerks-webpush` branch.

## Updating to a new stable Zulip release

Example: upgrading from `12.2` to a future `12.3` tag.

1. Fetch upstream and tags.
2. Verify the new release is the desired production baseline.
3. Rebase only the Litewerks downstream commits from the old stable tag onto the new stable tag.
4. Resolve conflicts without changing upstream behavior unrelated to Web Push.
5. Run the relevant Zulip backend/frontend tests and manual iPhone Web Push acceptance checks.
6. Update `litewerks-webpush` only after validation.

Typical flow:

```bash
git fetch upstream --tags
git checkout litewerks-webpush
git rebase --onto 12.3 12.2
git push origin litewerks-webpush --force-with-lease
```

For a major release, use the exact old and new stable tags rather than assuming the same branch topology.

## Production deployment

Configure `/etc/zulip/zulip.conf` to point `git_repo_url` at this fork, then deploy the customization branch with Zulip's supported Git upgrade mechanism:

```bash
/home/zulip/deployments/current/scripts/upgrade-zulip-from-git litewerks-webpush
```

Do this only after the branch has been rebased onto the same stable Zulip version intended for production and has passed tests.

## Secrets and persistent configuration

Never commit VAPID private keys or deployment secrets to this repository.

Persistent configuration belongs under `/etc/zulip/` on the server, using Zulip's normal settings/secrets mechanisms. Database Web Push subscriptions live in PostgreSQL and are migrated through normal Zulip migrations.

## Web Push scope

The first implementation is intentionally scoped to modern iOS Home Screen web apps, targeting iOS 18.4+ Declarative Web Push. The downstream patch should reuse Zulip's existing notification eligibility/queueing logic and add Web Push as an additional delivery transport rather than duplicating notification policy.

Expected patch areas:

- installable web app manifest / PWA metadata;
- notification permission and PushManager subscription UI;
- Web Push subscription persistence/API;
- VAPID configuration and sender;
- integration with Zulip's existing missed-message notification worker;
- notification navigation/badge payloads;
- invalid/expired subscription cleanup;
- automated tests and iPhone acceptance testing.

## Retirement

Zulip upstream has an open Web Push/PWA notifications effort. If upstream ships equivalent functionality, prefer migrating to upstream support and deleting the Litewerks downstream patch rather than maintaining duplicate implementations indefinitely.
