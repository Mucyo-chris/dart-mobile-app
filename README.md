name: Daily contribution updater

on:
  schedule:
    - cron: '0 0 * * *'    # every day at 00:00 UTC
  workflow_dispatch:

permissions:
  contents: write

jobs:
  daily-update:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout (no token persistence)
        uses: actions/checkout@v4
        with:
          persist-credentials: false

      - name: Ensure file exists
        run: |
          if [ ! -f CHANGELOG.md ]; then
            echo "# Changelog" > CHANGELOG.md
            git add CHANGELOG.md
            git commit -m "chore(docs): create CHANGELOG.md" || true
          fi

      - name: Append line if changed
        id: append
        run: |
          NOW="$(date -u +"%Y-%m-%d %H:%M:%S UTC")"
          LINE="- ${NOW} — automated daily update"
          touch CHANGELOG.md
          if ! tail -n 1 CHANGELOG.md 2>/dev/null | grep -Fxq "$LINE"; then
            echo "$LINE" >> CHANGELOG.md
            echo "changed=true" >> $GITHUB_OUTPUT
          else
            echo "changed=false" >> $GITHUB_OUTPUT
          fi

      - name: Commit and push as you (using PAT)
        if: steps.append.outputs.changed == 'true'
        env:
          REPO: ${{ github.repository }}
          PAT: ${{ secrets.PAT_TOKEN }}
          AUTHOR_NAME: ${{ secrets.USER_NAME }}
          AUTHOR_EMAIL: ${{ secrets.USER_EMAIL }}
        run: |
          git config user.name "${AUTHOR_NAME}"
          git config user.email "${AUTHOR_EMAIL}"
          git add CHANGELOG.md
          git commit -m "chore(docs): daily upkeep ${NOW}" || exit 0
          # Push using PAT; persist-credentials was false so we add remote with token
          git push "https://x-access-token:${PAT}@github.com/${REPO}.git" HEAD:${GITHUB_REF#refs/heads/}
