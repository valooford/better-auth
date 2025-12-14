## Upstream

Configure the new remote.

```shell
git remote add upstream https://github.com/better-auth/better-auth.git
# git remote -v
git fetch upstream
git checkout -b main upstream/main # just go to the `main` branch
git push -u origin main # choose "Push to..."
```

Update from `upstream/main` through rebase:

```shell
git checkout main
git fetch upstream --tags
# git tag # > q
# git tag --list
# git tag -l "v1.3.*"
git rebase upstream/main
git push origin main --force-with-lease --tags
```

Check out specific tag:

```shell
git checkout <tag-name>
git checkout -b refs/heads/<branch-name> refs/tags/<tag-name>
# Tweak the publishing configuration (workflows, private=true)
```

## Take fix from another fork

```shell
# fetch `canary` and create local branch `fix/5824` without adding a new remote
git fetch https://github.com/RodrigoRafaelSantos7/better-auth-original.git canary:fix/5824
git checkout fix/5824
git rebase main # v1.3.34/github-packages
# *resolve conflicts*
# git rebase --abort
# > Current branch fix/5824 is up to date.
git push -u origin fix/5824
```

Sometimes it's easier to translate changes manually (rebase conflicts hell). \
Just create a new branch from desired version of the lib. \
Commit the changes, bump the package version. \
Prefer using prepatch `v1.3.34-issue-5824.0` over metadata `v1.3.34+issue-5824.convex-anonymous`.

## Auto-sync

Create `.github/workflows/sync.yml` file in default (`canary`) branch.

```yml
name: Sync with upstream

on:
  schedule:
    - cron: "0 3 * * *"  # daily
  workflow_dispatch:

jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          repository: valooford/better-auth
          token: ${{ secrets.GITHUB_TOKEN }}

      - name: Add upstream
        run: |
          git config user.name "GitHub Action"
          git config user.email "action@github.com"
          git remote add upstream https://github.com/better-auth/better-auth.git
          git fetch upstream

      - name: Sync main
        run: |
          git checkout main
          git rebase upstream/main
          git push origin main --force-with-lease
```

## Release specific branch to GitHub Packages

https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-npm-registry

When "403 Forbidden Permission installation not allowed to Create organization package" \
https://docs.github.com/en/actions/tutorials/publish-packages/publish-docker-images#publishing-images-to-github-packages

```
setup-node with registry-url: 'https://npm.pkg.github.com'
env NODE_AUTH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
# simplify "Determine npm tag" step
run: pnpm -r publish --registry https://npm.pkg.github.com
permissions packages=write attestation=write
```

Set `private=true` for other packages. \
Rename patched package to have @scope equal to the user/organization name of the fork owner.

```shell
git checkout fix/5824
# git tag v1.4.6
# cd packages/better-auth
# pnpm version prepatch --preid issue-5824 # v1.4.6-issue-5824.0
pnpm bump # interactive using `bumpp` package
# Undo last commit (message is too verbose)
git commit -m "chore: release v1.4.6-issue-5824.0"
git tag v1.4.6-issue-5824.0
git push -u origin fix/5824
git push origin v1.4.6-issue-5824.0 # git push --tags
# git tag -d v1.0.0
# git push --delete origin v1.0.0
```

Add `.npmrc` file:

```
@valooford:registry=https://npm.pkg.github.com/
```