# blobfs WalkDir reports nested "dir/" markers as phantom files

Found while investigating apache/iceberg-go#2159 (2026-10-10). Not filed upstream yet.
It is a separate bug from #2159 and is not fixed by the #2159 change.

## Summary

On flat object stores (S3, GCS, Azure without hierarchical namespace),
`blobfs.FileIO.WalkDir` reports a trailing-slash directory marker object below
the walk root as a 0-byte file with a made-up path. A marker `tbl/data/` shows
up as the file `tbl/data/data`.

Marker objects like this are written by `blobfs.FileIO.MkdirAll` itself, by the
S3/GCS consoles ("Create folder"), and by tools such as Hadoop's S3A.

## Reproduction

In-memory bucket, run from `io/gocloud/blobfs` (package-internal test):

```go
bucket := memblob.OpenBucket(nil)
for _, k := range []string{"tbl/", "tbl/data/"} {
	bucket.WriteAll(ctx, k, nil, nil)
}
bucket.WriteAll(ctx, "tbl/data/a.parquet", []byte("x"), nil)

bfs := testBlobFileIO(ctx, "b", bucket)
bfs.WalkDir("s3://b/tbl", func(p string, d fs.DirEntry, err error) error {
	fmt.Println(p, d.IsDir())
	return nil
})
```

Output:

```
s3://b/tbl                 true
s3://b/tbl/data            true
s3://b/tbl/data/a.parquet  false
s3://b/tbl/data/data       false   <- phantom: the "tbl/data/" marker
```

Expected: no `tbl/data/data` entry.

## Cause

- `WalkDir` (`io/gocloud/blobfs/blob.go`) lists each directory with prefix
  `dir/` and delimiter `/`. The marker object `dir/` itself matches that
  prefix, so it comes back as a file entry. It is named `path.Base("tbl/data/")`,
  which is `data`, so `fs.WalkDir` joins it to `tbl/data/data`.
- `isDirectoryMarker` filters this entry only for the walk root
  (`obj.Key == directoryMarker(walkRootKey)`). Markers for nested directories
  pass through.
- The existing test `TestBlobFileIOWalkDirSkipsDirectoryMarker` covers only
  the root case.

## Impact

- **Orphan cleanup** (`table/maintenance/orphan_cleanup.go`, `walkDirectory`):
  each nested marker is a candidate orphan with a path that doesn't exist.
  Orphan counts are inflated, and the delete step gets a nonexistent key.
  Missing keys are not treated as delete errors, so the real marker is left in
  place and nothing else is deleted.
- **`PurgeFiles`**: the phantom path is added to the purge set and its delete
  is a no-op.
- **Hadoop catalog**: callbacks skip non-directory entries or only match
  metadata file names, so it is unaffected in practice.

## Suggested fix

After the #2159 change, `WalkDir` lists directories through `walkFS.ReadDir`
in `blob.go`. In that loop, drop the object whose key equals the listing
prefix (`strings.TrimPrefix(obj.Key, prefix) == ""`), because it is the marker
for the directory being listed. Two details:

1. A directory whose only object is its own marker (an empty directory made by
   `MkdirAll`) must still count as existing. Track that the marker was seen and
   don't return `fs.ErrNotExist` for it. `TestBlobFileIOWalkDirSkipsDirectoryMarker`
   fails without this.
2. Once markers are dropped in `ReadDir`, `isDirectoryMarker` and its call in
   `WalkDir` are no longer needed.

Test: walk a tree with `tbl/`, `tbl/data/`, `tbl/metadata/` markers plus real
files, and assert that only the real files are reported as files.

This was prototyped and passed `./io/...`, `./catalog/hadoop/...`,
`./table/maintenance/...` and `./cmd/iceberg/...`, then pulled out of the
#2159 change to keep that change scoped.

## Related: `FileIO.Stat` misreports hierarchical-namespace directories

Also seen during #2159, and not fixed there. On ADLS Gen2 with a hierarchical
namespace, `FileIO.Stat("<dir>")` calls `Attributes` on the key first. The
directory's 0-byte blob exists, so `Stat` returns a file (`IsDir() == false`)
for a directory. Inside this repo, only the Hadoop catalog's namespace and
table checks rely on it. A fix would check for children under `key + "/"`
when the object is 0 bytes, the same way `walkFS.Stat` does.
