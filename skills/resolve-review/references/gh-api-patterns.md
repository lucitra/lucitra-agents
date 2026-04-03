# GitHub API Patterns for PR Review Resolution

## REST API

### List PR comments
```bash
gh api repos/{owner}/{repo}/pulls/{pr}/comments
```

### Reply to a comment
```bash
gh api repos/{owner}/{repo}/pulls/{pr}/comments \
  -f body="your reply" \
  -F in_reply_to={comment_id}
```

### List PR reviews
```bash
gh api repos/{owner}/{repo}/pulls/{pr}/reviews \
  --jq '.[] | {id, state, body, user: .user.login}'
```

### Dismiss a review
```bash
gh api -X PUT repos/{owner}/{repo}/pulls/{pr}/reviews/{review_id}/dismissals \
  -f message="Issues addressed in {sha}"
```

## GraphQL API

### Fetch review threads with resolution status
```graphql
{
  repository(owner: "{owner}", name: "{repo}") {
    pullRequest(number: {pr}) {
      reviewThreads(first: 50) {
        nodes {
          id
          isResolved
          isOutdated
          path
          line
          comments(first: 5) {
            nodes {
              databaseId
              body
              author { login }
              createdAt
            }
          }
        }
      }
    }
  }
}
```

### Resolve a thread
```graphql
mutation {
  resolveReviewThread(input: {threadId: "{id}"}) {
    thread { isResolved }
  }
}
```

### Unresolve a thread (if needed)
```graphql
mutation {
  unresolveReviewThread(input: {threadId: "{id}"}) {
    thread { isResolved }
  }
}
```

## Matching REST comment IDs to GraphQL thread IDs

The REST API returns `id` (databaseId) for each comment. The GraphQL API returns `reviewThreads` with nested `comments.nodes[].databaseId`. Match them:

```python
# Pseudocode
for thread in graphql_threads:
    for comment in thread.comments.nodes:
        if comment.databaseId == rest_comment.id:
            # This thread corresponds to this REST comment
            thread_id = thread.id
```

## Code Scanning Alerts

Code scanning comments have a specific format:
- Body contains `## Workflow does not contain permissions` or similar
- Body contains a `[Show more details]` link to `/security/code-scanning/N`
- These auto-resolve when the underlying code is fixed and re-scanned

To check if a code scanning alert is fixed:
```bash
gh api repos/{owner}/{repo}/code-scanning/alerts/{alert_number} \
  --jq '{state, rule: .rule.description}'
```
