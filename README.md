# doodle-artifacts

List, search or delete GitHub artifacts.

# Usage

## List artifacts in a repository

Use `method` input: `list`.

The following inputs are required for this, and all subsequent, methods:

* `github_token`: Token to authenticate with GitHub API.
* `repo`: The target repository (in the format `owner/repo`).

```yml
    steps:
      - name: Get repo artifact list
        id: list-artifacts
        uses: empydoodle/action-artifacts@v1
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          repo: ${{ github.repository }}
          method: 'list'

      - name: List artifact names
        env:
          no_artifacts: ${{ fromJson(steps.list-artifacts.outputs.artifacts).total_count }}
          artifacts_json: ${{ fromJson(steps.list-artifacts.outputs.artifacts).artifacts }}
        run: |
          echo $json | jq -r '.artifacts.[].name'
```

Returns standard API response of a JSON array of artifacts in the target repository (`repo`) - see [GitHub API documentation](https://docs.github.com/en/rest/actions/artifacts#list-artifacts-for-a-repository) for schema.

## Search for artifacts in a repository

Use `method` input: `search`.

Performs the `list` method, and searches the resulting array for any artifacts where the name matches the regex of the `search_name` input.

By default, an empty search result (i.e. no artifacts found) will not fail the action (but will show in log output). To enable this behaviour, set `action_fail_on_empty_search` input to `'true'` (as string).

```yml
    steps:
      - name: Search for artifacts
        id: search-artifacts
        uses: empydoodle/action-artifacts@v1
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          repo: ${{ github.repository }}
          method: 'search'
          search_name: 'target-artifact'
          action_fail_on_empty_search: true # optional, defaults to false

      - name: Show artifact data
        env:
          json: ${{ steps.search-artifacts.outputs.search_results }}
        run: |
          for entry in $(echo $json | jq -r '.[]') ; do
            echo $entry | jq '{"name", "id", "size_in_bytes", "updated_at", "archive_download_url"}'
          done
```

Returns a JSON array of artifacts that match the search criteria.

## Delete target artifacts in a repository

Use `method` input: `delete`.

Performs the `search` method, and deletes each artifact entry using the GitHub API. All inputs passed to `search` are applicable to this method as well.

By default, a failed delete operation will not fail the action (but will show in log output). To enable this behaviour, set `action_fail_on_delete_error` input to `'true'` (as string).

```yml
    steps:
      - name: Delete artifacts
        id: delete-artifacts
        uses: empydoodle/action-artifacts@v1
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          repo: ${{ github.repository }}
          method: 'delete'
          search_name: 'target-artifact'
          action_fail_on_empty_search: true # optional, defaults to false
          action_fail_on_delete_error: true # optional, defaults to false

      - name: Show artifact delete results
        env:
          json: ${{ steps.delete-artifacts.outputs.delete_results }}
        run: |
          for entry in $(echo $json | jq -r '.[]') ; do
            name=$(echo $entry | jq -r '.name')
            if [[ "$(echo $entry | jq '.deleted") == "true" ]]; then
              echo "Artifact $name was deleted!"
            else
              echo "Artifact $name was not deleted."
            fi
          done
```

Returns a JSON array containing the deletion result of each targeted artifact, e.g.:

```json
[
  {
    "name": "target-artifact-0",
    "id": 123456789,
    "deleted": true
  },
  {
    "name": "target-artifact-1",
    "id": 234567890,
    "deleted": false
  }
]
```
