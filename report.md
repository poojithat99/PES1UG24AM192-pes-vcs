# PES-VCS Analysis Report

## Phase 5: Branching and Checkout

**Q5.1: A branch in Git is just a file in `.git/refs/heads/` containing a commit hash. Creating a branch is creating a file. Given this, how would you implement `pes checkout <branch>` — what files need to change in `.pes/`, and what must happen to the working directory? What makes this operation complex?**

To implement `pes checkout <branch>`, the following steps are needed:
1. Update `.pes/HEAD` to contain `ref: refs/heads/<branch>`.
2. The working directory needs to be structurally aligned with the target branch's tree: any files present in the new branch but not in the current working directory must be created, differing files must be modified, and files present in the current branch but missing in the new branch must be deleted.
3. The `.pes/index` file must be rewritten entirely to match the new branch's current tree to reset the staging area.

The primary source of complexity is handling uncommitted (dirty) changes in the working directory. A simple robust implementation requires determining if resetting a file to match the target branch will overwrite or delete unsaved work, potentially involving complex conflict-resolution logic.

**Q5.2: When switching branches, the working directory must be updated to match the target branch's tree. If the user has uncommitted changes to a tracked file, and that file differs between branches, checkout must refuse. Describe how you would detect this "dirty working directory" conflict using only the index and the object store.**

Conflict detection using the index and object store:
1. Read the `index` for the current active branch.
2. For every active index entry, do a metadata-based check (`mtime` and `size`) against the actual file in the filesystem. If the metadata differs, re-hash the active file's contents and compare it against the expected blob hash stored in the index entry. A mismatch implies an "unstaged/uncommitted modification."
3. Read the target branch's tree from the object store. 
4. Check if the modified file has a different hash in the target branch's tree compared to the current branch's commit tree. If it differs, checking out would overwrite the user's uncommitted work with the target branch's data, so the checkout must refuse and prompt the user to commit or stash.

**Q5.3: "Detached HEAD" means HEAD contains a commit hash directly instead of a branch reference. What happens if you make commits in this state? How could a user recover those commits?**

When making commits in a detached HEAD state, the commit objects are written to the object store and `.pes/HEAD` updates to point directly to these newly generated hashes. However, because `HEAD` is not tracking any branch file (like `refs/heads/main`), no branch pointer advances.

If the user eventually checks out a different branch, those commits will become orphaned, as there will be no branch or tag referencing them. To recover those commits, the user would need to investigate their `.pes/objects` (or use a reflog tool if one existed) to find the orphaned commit hash manually, then create a new branch reference pointing to that hash.

---

## Phase 6: Garbage Collection

**Q6.1: Over time, the object store accumulates unreachable objects. Describe an algorithm to find and delete these objects. What data structure would you use to track "reachable" hashes efficiently? For a repository with 100,000 commits and 50 branches, estimate how many objects you'd need to visit.**

**Algorithm:**
A classic Mark-and-Sweep garbage collection mechanism serves best:
1. **Mark Phase**: Initialize a `visited` set to track reachable objects. Begin from the roots: all references in `.pes/refs/heads/`, tags, and `HEAD`. Traverse over the graph by examining commits, adding their hashes to `visited`. For each commit, recursively traverse into its parent commits and its associated directory tree. For every tree, queue every child tree and blob entry into `visited`.
2. **Sweep Phase**: Iterate over every object shard directory in `.pes/objects/`. If an object's derived hash does not exist in the `visited` set, delete the file to recover space.

**Data Structure:**
Use a hash set (potentially backed by a Bloom filter combined with standard open addressing) to efficiently track the reachability of the 32-byte hash sequences in memory without redundant traversal.

**Scale Estimation:**
For 100,000 commits over 50 branches, traversing would be extremely optimized due to the `visited` hash set skipping large portions of overlapping history. We’d parse exactly 100,000 commit objects, but the tree and blob traversal would depend on object reuse. Even though there are millions of tree states, only unique trees and newly modified blobs per commit are examined. The search space bounds closely to the sum of unique objects generated historically, roughly parsing a few million objects.

**Q6.2: Why is it dangerous to run garbage collection concurrently with a commit operation? Describe a race condition where GC could delete an object that a concurrent commit is about to reference. How does Git's real GC avoid this?**

**Race Condition:**
1. A user stages a new file. `pes add new_file.txt` reads the contents and writes a new `blob` to `.pes/objects/`.
2. A concurrent GC process initiates its mark phase. It begins scaling the graph backwards from `HEAD`. Since `pes commit` hasn't generated a commit or a tree pointing to the newly staged `blob` yet, the GC marks it as unreachable. 
3. The GC process executes its sweep phase, subsequently deleting the new `blob`.
4. The user runs `pes commit`. The commit utilizes the index to build the `tree`, assuming the initial blob resides smoothly in the object store. The resulting commit is now corrupt because it points to an underlying tree missing a blob.

**How Git Resolves This:**
Git effectively implements a grace period metric utilizing the latest modified timestamp of objects in the filesystem. By default, `git gc` only considers an object "prunable" or safely unreachable if it has not been accessed/modified in the last 2 weeks. Additionally, active operations utilize transaction-like lock files to temporarily ensure atomicity, keeping GC processes from sweeping active references out.
