# doodle-artifacts

List, search or delete GitHub artifacts.

# Usage

## List artifacts in a repository

Use `method` input: `list`.

Checks the connection to the GitHub API with the passed values and generates a JSON array containing all artifacts in the repository, in the form of JSON objects.

The following inputs are required for this, and all subsequent, methods:

* `github_token` (`string`): Token to authenticate with GitHub API.
* `repo` (`string`): The target repository (in the format `owner/repo`).

Optional inputs include:

* `action_fail_on_list_mismatch` (`bool`/`string`): Fail the action (via `exit 1`) if the number of fetched artifacts does not match the number of artifacts reported by the GitHub API. Otherwise, it will notify the mismatch and continue with the fetched artifact array.

It populates the following outputs:

* `no_artifacts`: The number of artifacts in the passed repository (as reported by the GitHub API).
* `artifacts`: JSON array of all artifacts in the repository.

```yml
    steps:
      - name: Get repo artifact list
        id: list-artifacts
        uses: empydoodle/action-artifacts@v2
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          repo: ${{ github.repository }}
          method: 'list'

      - name: List artifact names
        env:
          no_artifacts: ${{ steps.list-artifacts.outputs.no_artifacts }}
          artifacts: ${{ fromJson(steps.list-artifacts.outputs.artifacts) }}
        run: |
          # Get list of artifact names
          echo $artifacts | jq 'map(.name)'
```

The list artifacts is available in the `artifacts_list.json` file.

## Search for artifacts in a repository

Use `method` input: `search`.

Performs the `list` method, and searches the resulting array for any artifacts where the name matches the `search_name` input in part or fully.

It requires the following inputs (in addition to `list` inputs):

* `search_name` (`string`): Query to search artifact.

Optional inputs (in addition to those for `list`):

* `action_fail_on_empty_search` (`bool`/`string`): Fail the action if the search query yields no results. Otherwise, return an empty array

It populates the following outputs (in addition to `list` outputs):

* `search_results`: JSON array containing the objects returned by the search (in the form of JSON objects).

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

The list of search results is available in the `artifacts_search.json` file.

## Delete target artifacts in a repository

Use `method` input: `delete`.

Performs the `search` method, and deletes each result using the GitHub API.

Requires all inputs required by `search` method.

Optional inputs (in addition to those for `search`):

* `action_fail_on_delete_error` (`bool`/`string`): Fail the action if errors are encountered while deleting artifact(s). Otherwise continue.

Populates the following outputs (in addition to `search` outputs):

* `delete_results`: JSON array containing an object-per-artifact with the following values:
  * `name` (`string`): name of the artifact.
  * `id` (`int`): ID of the artifact.
  * `deleted` (`bool`): Indicator of whether deletion was successful.

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

The list of artifact statuses is available in the `artifacts_delete.json` file.

## Additional Inputs

* `cleanup_files`: Remove files after run.
