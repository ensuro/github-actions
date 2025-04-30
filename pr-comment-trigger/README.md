# PR Comment Trigger

This action simplifies the creation of workflows that trigger on a comment on a PR.

It checks that the comment matches a specific pattern, optionally extracting data from it, and returns the context of the PR.

Inputs that it receives:

- `match`: The regex pattern to match the comment against. By default, it will match any comment.
- `ack`: A thumbs up reaction is added to the comment if it matched. Enabled by default.

Notice that you'll need to grant the workflow `issues: write` permission for the ack to work.

Outputs:

- pr-number: The PR number where the triggering comment was added
- pr-head-sha: The SHA of the HEAD commit of the PR
- pr-head-ref: The PR branch
- pr-base-ref: The base branch of the PR
- comment-body: The full comment body
- matches: A JSON object with the regex matches
- match-found: Whether the regex matched or not
- comment-id: The ID of the triggering comment

Notice that you will also have a lot of information available in the context for `issue_comment`:

- `github.event.comment`: The [full comment object](https://docs.github.com/en/rest/issues/comments?apiVersion=2022-11-28#get-an-issue-comment)
- `github.event.issue`: The [full issue object](https://docs.github.com/en/rest/issues/issues?apiVersion=2022-11-28#get-an-issue)
- [And more](https://docs.github.com/en/webhooks/webhook-events-and-payloads#issue_comment)

## Usage

Here's a simple workflow that uses this action:

```yaml
name: Trigger on a PR comment

permissions:
  issues: write # Required to ack the comment

on:
  issue_comment:
    types: [created]

jobs:
  check-comment:
    # It's recommended that you use the action in a separate job.
    # This makes it easier to skip the rest of the workflow if there was no match.

    # The match on the comment body is not necessary, but it's a good way to skip unrelated comments quickly.
    if: ${{ github.event.issue.pull_request && startsWith(github.event.comment.body, '/foo') }}

    runs-on: ubuntu-latest

    outputs:
      match-found: ${{ steps.comment-check.outputs.match-found }}
      matches: ${{ steps.comment-check.outputs.matches }}
      pr-head-sha: ${{ steps.comment-check.outputs.pr-head-sha }}
      pr-head-ref: ${{ steps.comment-check.outputs.pr-head-ref }}
      pr-base-ref: ${{ steps.comment-check.outputs.pr-base-ref }}
    steps:
      - id: comment-check
        uses: ensuro/github-actions/pr-comment-trigger@pr-comment-trigger # Use a tag or commit hash here
        with:
          match: "^/foo (?<bar>.*?) (?<baz>.*?)$"

  process-command:
    needs: check-comment
    if: ${{ needs.check-comment.outputs.match-found == 'true' }}
    runs-on: ubuntu-latest
    steps:
      - name: Process Command
        run: |
          echo "Processing /check on PR #${{ github.event.issue.number }} ${{ needs.check-comment.outputs.pr-head-ref }} -> ${{ needs.check-comment.outputs.pr-base-ref }}" >> $GITHUB_STEP_SUMMARY
          echo "bar: ${{ fromJson(needs.check-comment.outputs.matches).bar }}" >> $GITHUB_STEP_SUMMARY
          echo "baz: ${{ fromJson(needs.check-comment.outputs.matches).baz }}" >> $GITHUB_STEP_SUMMARY
```

Notice that this job will be associated with the main branch and not with the PR. If you want to provide feedback on the PR besides the reaction, you can use checks and/or comments on the PR.

A more comprehensive workflow example showcasing this is provided in the [examples](./examples) directory.
