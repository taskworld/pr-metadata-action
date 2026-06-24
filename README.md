# pr-metadata-action

Two composite GitHub Actions for stashing structured JSON inside a pull request body, hidden in an HTML comment. Useful for passing state between workflow runs without a separate store.

The metadata block looks like this and is invisible in the rendered PR description:

```html
<!-- pr-meta: {"version":"1.2.3","reviewed":true} -->
```

- [`read`](./read/action.yml) — extract the JSON from a PR body
- [`write`](./write/action.yml) — write or merge JSON into the current PR body

Both are composite actions that wrap `actions/github-script@v7`. No build step, no extra setup.

## `read`

### Inputs

| Name           | Required | Default            | Description                                                |
| -------------- | -------- | ------------------ | ---------------------------------------------------------- |
| `github-token` | no       | `${{ github.token }}` | Token used for the REST API fallback                       |
| `marker`       | no       | `pr-meta`      | Marker name inside the HTML comment                        |
| `pr-number`    | no       | _triggering PR_    | PR number to read from. Defaults to the PR from the event  |

### Outputs

| Name           | Description                                                  |
| -------------- | ------------------------------------------------------------ |
| `has-metadata` | `"true"` if the marker was found, `"false"` otherwise        |
| `result`       | Parsed JSON, stringified. Empty string if the action failed  |

### Example

```yaml
on:
  pull_request:

jobs:
  read-metadata:
    runs-on: ubuntu-latest
    steps:
      - id: meta
        uses: taskworld/pr-meta-action/read@v1
      - run: echo '${{ steps.meta.outputs.result }}'
        if: steps.meta.outputs.has-metadata == 'true'
```

## `write`

Must run on a `pull_request` event — it updates the PR body via the REST API.

### Permissions

```yaml
permissions:
  pull-requests: write
```

### Inputs

| Name           | Required | Default               | Description                                                          |
| -------------- | -------- | --------------------- | -------------------------------------------------------------------- |
| `github-token` | no       | `${{ github.token }}` | Token used to update the PR body                                     |
| `marker`       | no       | `pr-meta`         | Marker name inside the HTML comment                                  |
| `data`         | **yes**  | —                     | JSON-stringified object to write under the marker                    |
| `merge`        | no       | `"false"`             | If `"true"`, shallow-merge with existing metadata instead of replacing |

When `merge` is `true`, both the existing and incoming values must be plain JSON objects.

### Outputs

| Name      | Description                                                                       |
| --------- | --------------------------------------------------------------------------------- |
| `result`  | JSON-stringified metadata that was written, after the merge if `merge` is enabled |
| `changed` | `"true"` if the PR body was updated, `"false"` if it was already up to date        |

`result` is the effective stored value, which is the most reliable way to learn the result of a `merge: "true"` write without reading the body back.

### Example

```yaml
on:
  pull_request:

permissions:
  pull-requests: write

jobs:
  write-metadata:
    runs-on: ubuntu-latest
    steps:
      - id: meta
        uses: taskworld/pr-meta-action/write@v1
        with:
          data: '{"reviewed":true,"version":"1.2.3"}'
          merge: "true"
      - run: echo 'wrote ${{ steps.meta.outputs.result }} (changed=${{ steps.meta.outputs.changed }})'
```

## Notes

- Both actions accept a custom `marker`, so you can store multiple independent JSON blobs in one PR body by using different names.
- `write` is a no-op when the rendered body would be unchanged, so it's safe to run on every push.
