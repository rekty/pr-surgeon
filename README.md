# pr-surgeon

Reviews a pull request link with the code around it, and pushes the smallest correct fix to a branch in a repository you control.

It is a [Pollinations code agent](https://github.com/pollinations/pollinations/blob/main/BUILD_YOUR_OWN_AGENT.md): this repository is the deployment unit. [`agent.ts`](./agent.ts) is self-contained — no imports, one default export that returns a `Response`, only standard Worker APIs — and Pollinations runs `openai/gpt-6.1-sol` for the review.

```
## Review of https://github.com/pollinations/pollinations/pull/17422

The enclosure parser rejects valid Link headers when rel is unquoted, follows
another parameter, or contains multiple relation types. Parse the link
parameters so these responses return their stored URL instead of throwing
NO_VIDEO_URL.

### Findings
- **medium** `packages/sdk/src/client.ts:765` — Accept valid enclosure Link
  header forms: the regex requires rel="enclosure" immediately after the URL …

### Fix pushed
Branch `fix-sdk-enclosure-link-parsing` on `you/pollinations`.
```

## Publish it

The dashboard takes the repository and nothing else. `code-agent.json` in this
repository is that configuration, kept here so the intent is versioned:

```json
{
    "type": "code_agent",
    "repository": "https://github.com/rekty/pr-surgeon"
}
```

1. Open [My Models](https://enter.pollinations.ai/my-models) and add an agent.
2. Choose **Code agent** and paste the repository URL.
3. Save. The repository name becomes the model ID (`rekty/pr-surgeon`), and the
   repository description becomes the catalog description.

The agent is deployed from the head commit, so a push to the default branch is a
new revision.

## Call it

Input is the pull request URL — that is all it needs:

```bash
curl https://gen.pollinations.ai/v1/responses \
  -H "authorization: Bearer $POLLINATIONS_API_KEY" \
  -H "content-type: application/json" \
  -H "x-github-token: $GITHUB_TOKEN" \
  -d '{"input":"https://github.com/pollinations/pollinations/pull/17422","metadata":{"push_repo":"you/pollinations"}}'
```

| field | meaning |
| --- | --- |
| `input` | the pull request URL (the only required input) |
| `x-github-token` header | token for GitHub rate limits; needs `repo` scope to push |
| `metadata.push_repo` | repository that receives the fix branch, usually your fork |
| `metadata.review_only` | `1` reviews and returns the diff without touching a repository |
| `metadata.branch` | override the branch name the agent chose |
| `metadata.model` | override the review model |

Without a token, or without write access, it still answers: the review plus the
fix and the exact commands to run, marked `patch_only`.

## What it does

1. The diff is fetched from `patch-diff.githubusercontent.com` (not rate limited),
   with title, head repository and head revision from the GitHub API.
2. The code around every changed line is fetched from the PR head as numbered
   windows, so the review sees real code rather than hunks.
3. One model call returns JSON: a summary, findings with file and line, and the
   fix as whole files or exact `find`/`replace` snippets.
4. The agent applies the fix itself. A `find` that does not match, or matches
   twice, is refused instead of guessed at; paths and branch names are validated
   before they reach the shell.
5. It clones the PR head in the caller's computer, writes, commits and pushes a
   new branch, then prints its own unified diff. The sandbox git has no
   `git apply`, so the agent writes files and computes the diff itself.

## Limits

- Text files only; a binary change is skipped, not guessed at.
- One model call and a character budget per fix: a pull request that needs four
  unrelated fixes gets the one the model considers most important.
- It does not run the tests it proposes, so a fix is a proposal for a maintainer,
  not a verified change.

## Evidence and write-up

Three real pull requests, with the cost of each review, and what blocked the
agent — repository access, the sandbox, the cost ledger — are written up in the
[pollinations PR that added the original source](https://github.com/pollinations/pollinations/pull/17466)
(`apps/agent-pr-surgeon/`). The runner used to produce that evidence is
[`scripts/live-review.mjs`](./scripts/live-review.mjs), and the agent's own tests
are [`agent.test.ts`](./agent.test.ts):

```bash
node --test agent.test.ts
```
