# CurseForge Packaging: How to Trigger a Build

This project uses the CurseForge automatic packager via a GitHub webhook. To create a packaged ZIP, push an annotated tag. The release type is inferred from the tag name.

- Alpha: tag contains `alpha`
- Beta: tag contains `beta`
- Release: tag contains neither `alpha` nor `beta`

The packaged file will be named `<package-as>-<project-version>.zip`.
For this repo, `package-as: Accountant_Classic`, so the ZIP looks like:

- `Accountant_Classic-<tag>.zip`

## Prerequisites
- Webhook already configured in GitHub → Settings → Webhooks
  - Payload URL: `https://www.curseforge.com/api/projects/<projectID>/package?token=<token>`
  - Content type: `application/json`
  - Events: `Just the push event`
- Branch pushed to GitHub (avoid creating a tag that points to a commit not yet on the remote).

## Typical Flow (Alpha)
```bash
# 1) Push your branch first (if needed)
git push origin master

# 2) Create an annotated alpha tag on the current commit
git tag -a 3.0.00-alpha.2 -m "Release 3.0.00-alpha.2"

# or want to tag a specific commit:
git tag -a 3.0.00-alpha.3 638e8592eafc51f4aaef436d640637af8ecbe5a6 -m "Release 3.0.00-alpha.3 (Currency Tracker UI prototype)"

# 3) Push the tag to trigger the packager
git push origin 3.0.00-alpha.2
```

## Beta Example
```bash
git push origin master

git tag -a 3.0.00-beta.1 -m "Release 3.0.00-beta.1"

git push origin 3.0.00-beta.1
```

## Release Example
```bash
git push origin master

git tag -a 3.0.00 -m "Release 3.0.00"

git push origin 3.0.00
```

## Re-tag (overwrite an existing tag)
If you need to reuse a tag name, delete it remotely and locally, recreate, then push:
```bash
# Delete local tag
git tag -d 3.0.00-alpha.2
# Delete remote tag
git push origin :refs/tags/3.0.00-alpha.2
# Recreate and push again
git tag -a 3.0.00-alpha.2 -m "Release 3.0.00-alpha.2 (repack)"
git push origin 3.0.00-alpha.2
```

## Verify
- Go to the CurseForge project → Files
- Look for a new build matching the tag you pushed
- ZIP name should be `Accountant_Classic-<tag>.zip`
- Contents will respect `pkgmeta.yaml` `ignore:` rules
