# Using `gh` for PR Review Comments

## Prerequisites

```bash
gh auth status
gh pr view 123 --json number,title,url
```

## Add a general PR comment

```bash
gh pr comment 123 --body "Looks good overall. One question on error handling."
```

## Add an inline file/line comment

```bash
PR=123
OWNER_REPO="mnbiehl/reins"
HEAD_SHA=$(gh pr view "$PR" --repo "$OWNER_REPO" --json headRefOid -q .headRefOid)

gh api "repos/$OWNER_REPO/pulls/$PR/comments" -X POST \
  -f body="Consider handling non-zero return here." \
  -f commit_id="$HEAD_SHA" \
  -f path="src/reins/commands/attach.py" \
  -F line=100 \
  -f side=RIGHT
```

## Reply to an existing inline comment

```bash
gh api "repos/$OWNER_REPO/pulls/$PR/comments" -X POST \
  -F in_reply_to=2826986988 \
  -f body="Fixed in latest commit."
```

## Submit an overall review state

```bash
gh pr review 123 --comment --body "Left a few inline suggestions."
gh pr review 123 --approve --body "LGTM."
gh pr review 123 --request-changes --body "Please address inline issues."
```

## Resolve a review thread (GraphQL)

```bash
gh api graphql -f query='
mutation($id:ID!){
  resolveReviewThread(input:{threadId:$id}) {
    thread { id isResolved }
  }
}' -f id='PRRT_xxx'
```

## Notes

- Inline comments require `commit_id`, `path`, and `line`.
- Use `-F` for numeric fields (`line`, `in_reply_to`), `-f` for strings.
- For multi-line bodies, prefer HEREDOCs to avoid quoting issues.
