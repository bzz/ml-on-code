Sample data from PGA to be used offline during a workshop.

```sh
pga list -u github.com/src-d/ -f json | jq -r 'select(.fileCount > 60) | .sivaFilenames[]' | pga get -i -o repositories
```