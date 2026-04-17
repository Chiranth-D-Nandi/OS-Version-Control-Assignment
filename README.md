PES-VCS Report

Chiranth D Nandi | PES2UG24AM048
OS Orange Assignment – Unit 4 

Phase 1: Object Storage

object_write and object_read implemented in object.c. Write prepends a type header ("blob/tree/commit <size>\0"), computes SHA-256 over the full object, and uses temp-file + rename for atomic writes. Read parses the header, recomputes the hash to verify integrity, and returns the data portion.

Phase 2: Tree Objects

tree_from_index implemented in tree.c. Loads the index, groups entries by directory prefix, recursively writes subtrees, and returns the root tree hash. Entries are sorted before serialization for deterministic output.

Phase 3: Index (Staging Area)

index_load, index_save, and index_add implemented in index.c. Text format:
<mode> <hash-hex> <mtime> <size> <path> per line.

Saves are atomic via fsync + rename. index_add reads the file, writes a blob to the object store, and updates the entry with stat metadata.

Phase 4: Commits and History

commit_create implemented in commit.c. Builds a tree from the staged index via tree_from_index, reads HEAD as parent (skipped for first commit), serializes the commit struct, writes it to the object store, and updates the branch ref via head_update.

Integration Test

All four phases pass.

Phase 5: Branching Analysis
Q5.1 – Implementing pes checkout <branch>

HEAD gets updated to ref: refs/heads/<branch>. The working tree is then rebuilt by walking the target commit's root tree recursively and writing each blob to its path. Files present in the current tree but absent in the target must be deleted. The complexity comes from handling deletions, directory creation/removal, and conflicts cleanly.

Q5.2 – Detecting Dirty Working Directory

For each index entry, stat the working copy and compare mtime and size against stored values — a mismatch means locally modified. Then compare the target branch's tree hashes against the current index. If a file differs between branches and is also locally dirty, checkout aborts with a conflict error.

Q5.3 – Detached HEAD

In detached HEAD state, commits are written normally but HEAD points directly to the commit hash instead of a branch ref. These commits become unreachable (dangling) the moment HEAD moves. Recovery requires noting the hash before moving and creating a branch pointing to it — objects stay in the store until GC runs.

Phase 6: Garbage Collection Analysis
Q6.1 – Finding Unreachable Objects

Start from all branch refs and HEAD, BFS/DFS following commit parents → trees → subtrees → blobs. Store every visited hash in a set. After traversal, list all files under .pes/objects/ and delete any hash not in the set.

A hash set is the right structure — O(1) lookup, no duplicates. For 100k commits across 50 branches, expect a few hundred thousand objects visited total depending on file churn per commit.

Q6.2 – GC Race Condition

Race: GC scans reachable objects and marks a blob unreachable. Concurrently, a commit writes that blob to the index but hasn't linked it to a commit object yet — the blob is live but not yet referenced. GC deletes it. The commit finishes pointing to a now-missing blob.

Git avoids this with a grace period (objects newer than ~2 weeks are never deleted) and a repo-level lock during GC. The temp-write-then-rename pattern also ensures partial writes are never the only copy.
