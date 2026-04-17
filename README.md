# PES-VCS Lab Report

## Student Details
**Name:** Shriniketh Kana  
**SRN:** PES1UG24CS446  

---

## Phase 1: Object Storage Foundation

### Screenshot 1A: test_objects passing
![1A Screenshot](/home/kana/Downloads/os-orange-u4/1A.png)

### Screenshot 1B: Sharded Directory Structure
![1B Screenshot](/home/kana/Downloads/os-orange-u4/1B.png)

---

## Phase 2: Tree Objects

### Screenshot 2A: test_tree passing
![2A Screenshot](/home/kana/Downloads/os-orange-u4/2A.png)

### Screenshot 2B: xxd of raw tree object
![2B Screenshot](/home/kana/Downloads/os-orange-u4/2B.png)

---

## Phase 3: The Index (Staging Area)

### Screenshot 3A: sequence of pes init, add, and status
![3A Screenshot](/home/kana/Downloads/os-orange-u4/3A.png)

### Screenshot 3B: Text format index
![3B Screenshot](/home/kana/Downloads/os-orange-u4/3B.png)

---

## Phase 4: Commits and History

### Screenshot 4A: pes log Output
![4A Screenshot](/home/kana/Downloads/os-orange-u4/4A.png)

### Screenshot 4B: Object Growth
![4B Screenshot](/home/kana/Downloads/os-orange-u4/4B.png)

### Screenshot 4C: References and HEAD
![4C Screenshot](/home/kana/Downloads/os-orange-u4/4C.png)

---

## Final Phase: Full Integration Test
![Final Integration Test](/home/kana/Downloads/os-orange-u4/Final.png)

---

## Analysis Questions (Phase 5 & 6)

### Branching and Checkout

**Q5.1:** A branch in Git is just a file in `.git/refs/heads/` containing a commit hash. Creating a branch is creating a file. Given this, how would you implement `pes checkout <branch>` — what files need to change in `.pes/`, and what must happen to the working directory? What makes this operation complex?

**Answer:** To implement `pes checkout <branch>`, I would first verify that the specified branch file exists in `.pes/refs/heads/`. Then I would update `.pes/HEAD` to contain `ref: refs/heads/<branch>`. Lastly, I would read the target commit, load its tree snapshot, and overwrite the working directory files to match the snapshot. This operation is complex because we must safely replace files in the working directory without destroying untracked files or unstaged changes, and also properly transition the `.pes/index` file to match the new HEAD commit so the staging area is correctly synced with the newly checked-out branch.

**Q5.2:** When switching branches, the working directory must be updated to match the target branch's tree. If the user has uncommitted changes to a tracked file, and that file differs between branches, checkout must refuse. Describe how you would detect this "dirty working directory" conflict using only the index and the object store.

**Answer:** We first iterate through the target branch's tree blobs. For each file, we compare its content hash against what is stored in the current `.pes/index`. If the entries disagree, we verify the physical file on disk using `stat()` checking if its metadata (size/mtime) differs from that of the index. If it differs, that indicates local uncommitted or unstaged changes. Alternatively, any file staged in the index with a blob hash that differs from both the current branch's commit tree and the target branch's checkout tree would instantly signal a conflict, forcing checkout to abort before overwriting.

**Q5.3:** "Detached HEAD" means HEAD contains a commit hash directly instead of a branch reference. What happens if you make commits in this state? How could a user recover those commits?

**Answer:** If you make commits in a "Detached HEAD" state, `.pes/HEAD` will be overwritten with the new commit hash, but no branch reference within `.pes/refs/heads/` will be updated to point to this new commit. If you later checkout a different branch, those commits become "unreachable" (dangling) since nothing tracks them. A user can recover them by inspecting terminal logs for the hashes or walking through `.pes/objects/` to manually extract the orphan commit's hash to establish a new branch reference (simulating standard git reflog recovery mechanisms).

### Garbage Collection and Space Reclamation

**Q6.1:** Over time, the object store accumulates unreachable objects — blobs, trees, or commits that no branch points to (directly or transitively). Describe an algorithm to find and delete these objects. What data structure would you use to track "reachable" hashes efficiently? For a repository with 100,000 commits and 50 branches, estimate how many objects you'd need to visit.

**Answer:** I would build a Mark-and-Sweep garbage collection algorithm. First, I would use a `Hash Set` (or a probabilistic Bloom filter alongside a localized array) to track reachable hashes efficiently. In the "Mark" phase, I would iterate through every branch pointer in `.pes/refs/heads/` and traverse backward through their linked parent commits. For each commit, I add its hash to the set, then load its associated tree, recursively adding every child tree and linked blob hash to the set. In the "Sweep" phase, I would iterate through `.pes/objects/` and delete any file whose hash name isn't contained within the reachable set. 
For 100,000 commits, assuming roughly 10 uniquely modified trees/blobs per commit diff on average, we could visit around 1,000,000 distinct objects. Using a reliable hash set mapping for these object hashes ensures $O(1)$ lookups, optimizing the time taken substantially.

**Q6.2:** Why is it dangerous to run garbage collection concurrently with a commit operation? Describe a race condition where GC could delete an object that a concurrent commit is about to reference. How does Git's real GC avoid this?

**Answer:** A race condition occurs if `pes commit` relies on staging previously written blobs up to a new commit tree object in the background. A running GC might scan the system's active refs and conclude those staged blobs are officially "unreachable" (as the commit hasn't completed to bind them historically to the HEAD reference yet). The GC then deletes the blobs maliciously just as the commit finishes writing referencing them, rendering a corrupted tree. Git natively avoids this via a two-week grace period restriction: It deliberately skips pruning unreachable objects created within the last 14 days. It also tightly coordinates index locks (e.g. `index.lock` configuration) mitigating simultaneous destructive commands.
