# Explore and extend a pull request {#pr-extend}

Scenario: you maintain an R package on GitHub with pull requests (PRs) from external contributors. Sometimes you need to experiment with the PR in order to provide feedback or to decide whether or not to merge. Going further, sometimes you want to add a few commits and then merge. Or maybe there are just some merge conflicts that require your personal, local attention. Let's also assume that you want the original PR author to get credit for their commits, i.e. you want to preserve history and provenance, not just diffs.

How do you checkout and possibly extend an external PR?

## Terminology

Vocabulary I use throughout.

**fork branch** The name of the branch in the fork from which the PR was made. Best case scenario: informative name like `fix-fluffy-bunny`. Worst case scenario: PR is from `master`.

**local PR branch** The name of the local branch you'll use to work with the PR. Best case scenario: can be same as fork branch. Worse case scenario: PR is from `master`, so you must make up a new name based on something about the PR, e.g. `pr-666` or `janedoe-master`.

**PR parent** The SHA of the commit in the main repo that is the base for the PR.

**PR remote** The SSH or HTTPS URL for the fork from which the PR was made. Or the nickname of the remote, if you've bothered to set that up.

## Official GitHub advice, Version 1

Every PR on GitHub has a link to "command line instructions" on how to merge the PR locally via command line Git. On this journey, there is a point at which you can pause and explore the PR locally.

Here are their steps with my vocuabulary and some example commands:

  * Create and check out the local PR branch, anticipating its relationship to the fork branch. Template of the Git command, plus an example of how it looks under both naming scenarios:
  
        git checkout -b LOCAL_PR_BRANCH master 
        git checkout -b fix-fluffy-bunny master 
        git checkout -b janedoe-master master 
    
  * Pull from the fork branch of the PR remote:
  
        git pull REMOTE FORK_PR_BRANCH
        git pull https://github.com/janedoe/yourpackage.git fix-fluffy-bunny
        git pull https://github.com/janedoe/yourpackage.git master
  
  * Satisfy yourself that all is well and you want to merge.
  * Checkout `master`:
  
        git checkout master
  
  * Merge the local PR branch into master with `--no-ff`, meaning "no fast forward merge". This ensures you get a true merge commit, with two parents.
  
        git merge --no-ff LOCAL_PR_BRANCH
        git merge --no-ff fix-fluffy-bunny
        git merge --no-ff janedoe-master
  
  * Push `master` to GitHub.
  
        git push origin master
  
What's not to like? The parent commit of the local PR branch will almost certainly not be the parent commit of the fork PR branch, where the external contributor did their work. This often means you get merge conflicts in `git pull`, which you'll have to deal with ASAP. The older the PR, the more likely this is and the hairier the conflicts will be.

I would prefer to deal with the merge conflicts only *after* I've vetted the PR and to resolve the conflicts locally, not on GitHub. So I don't use this exact workflow.

## Official GitHub advice, Version 2

GitHub has another set of instructions: [Checking out pull requests locally](https://help.github.com/articles/checking-out-pull-requests-locally/)

It starts out by referring to the Version 1 instructions, but goes on to address an inactive pull request", defined as a PR "whose owner has either stopped responding, or, more likely, has deleted their fork".

This workflow may NOT give the original PR author credit (next time it's easy to test this, I'll update with a definitive answer). I've never used it verbatim because I've never had this exact problem re: deleted fork.

## Official GitHub advice, Version 3

GitHub has yet another set of instructions: [Committing changes to a pull request branch created from a fork](https://help.github.com/articles/committing-changes-to-a-pull-request-branch-created-from-a-fork/)

The page linked above explains all the pre-conditions, but the short version is that a maintainer can probably push new commits to a PR, effectively pushing commits to a fork. Strange, but true!

This set of instructions suggests that you clone the fork, checkout the branch from which the PR was made, make any commits you wish, and then push. Any new commits you make will appear in the PR. And then you could merge.

My main takeaway: maintainer can push to the branch of a fork associated with a PR.

## My under-development workflow

*work in progress*

This combines ideas from the three above approaches, but with a few tweaks. I have the start of a workflow from R, but it's incomplete, so will sketch with command line Git first, then show my draft R code.

Tweak #1: figure out the PR parent SHA. You can do this by clicking around in the GitHub UI, but my R code below show how to get via the API.

Create and check out a new local PR branch based on the PR parent.

```
git checkout -b completions a1c46a8
git checkout -b PR_BRANCH PARENT_SHA
```

Pull from the fork and branch associated with the PR. Make a deliberate choice re: HTTPS or SSH, based on what you usually use. I show SSH here. Note the branch name here is the *fork branch* and recall that this may or may not be same as PR branch.

```
git pull --ff git@github.com:jimhester/readxl.git completions
git pull --ff git@github.com:OWNER/REPO.git FORK_BRANCH
```

Experiment with the PR and **make more commits if you need to**.

Push back to the fork branch associated with the PR.

```
git push git@github.com:jimhester/readxl.git HEAD:completions
git push git@github.com:OWNER/REPO.git HEAD:FORK_BRANCH
```

Merge or squash-and-merge the updated PR from GitHub in the browser.

Partial draft of R code to do the above


```r
library(gh)
library(git2r)

## the number of the pull request you want to work on
pr <- 320
## assuming wd = active project/package, gets OWNER/REPO
repo_info <- gh_tree_remote(".")

x <- gh(
  "/repos/:owner/:repo/pulls/:number",
  owner = repo_info$username,
  repo = repo_info$repo,
  number = pr
)

## get name of the fork's branch used for the PR
(fork_branch <- x$head$ref)

## determine name of local PR branch
(pr_branch <- if (fork_branch == "master") paste0("pr-", pr) else fork_branch)

## Suggestion from @jimhester for setting pr_branch
## remote_name-remote_branch? and maybe without a condition?

## get the parent SHA for the PR

## why is this not correct?!?!?
# (sha <- x$base$sha)

## alternative approach
## GET all commits for the PR and parent SHA of the first commit
y <- gh(
  "/repos/:owner/:repo/pulls/:number/commits",
  owner = repo_info$username,
  repo = repo_info$repo,
  number = pr
)
(sha <- purrr::pluck(y, list(1, "parents", 1, "sha")))

## Comment from @jimhester
## I don't think this is the right strategy, you should create the new branch
## directly off of the remote, rather than creating a new branch from the
## master.

## create and checkout the local PR branch, with correct parent
b <- branch_create(commit = lookup(sha = sha), name = pr_branch)
checkout(b)

## haven't figured out how to pull / push to arbitrary remote with git2r
## perhaps you'd have to actually add the fork as a remote?
## doing with command line Git for now

## comment from @jimhester:
## It doesn't look like git2r has an API that lets you use anonymous remotes, I
## think you need to explicitly add the remote then fetch, then do the branch,
## e.g. remote_add(), fetch(), branch() etc.

## form the pull command
glue::glue("git pull --ff {x$head$repo$ssh_url} {fork_branch}")

## make commit(s) HERE

## form the push command
glue::glue("git push {x$head$repo$ssh_url} HEAD:{fork_branch}")
```

