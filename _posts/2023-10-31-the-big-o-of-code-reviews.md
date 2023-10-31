---
layout: post
title:  "The Big O of Code Reviews"
date:   2023-10-31 13:00:00 +0200
---
I constantly rave about how much I enjoy the code review culture at Cash App. My colleague Saket
wrote an excellent post about it titled ["Great teams merge fast"][great-teams-merge-fast], where
he goes into detail on how we approach shipping code. A very important aspect of this culture is
crafting small, well-scoped pull requests and stacking them on top of each other. Here's why I think
this is so important:

It's natural to think of the time and effort required to review a pull request as growing 
linearly with the number of lines of code and affected files in the diff. But I would
argue that not all code changes are the same, and some types of changes are easier to review than
other. And most importantly, mixing different types of changes in a single commit or pull request 
can complicate things tremendously. I think the good old Big O notation serves pretty well in 
describing how the complexity of a code review grows depending on the type of changes!

### Automated refactorings: O(1)

Not so long ago I was asked to review a pull request that renamed a single class. The class 
was used in a few places, so the change affected 7 files. But even 
if the change had affected 1000 files, I wouldn't have needed to open any of them to approve the 
pull request, as I wasn't approving the changes themselves - I made the decision based on whether I 
agree with the author that the new name describes that class better. Besides, I was confident that 
the author used the IDE to perform renaming, and in an unlikely case of an IDE bug it would be 
caught by our continuous integration build.

The same idea applies to other types of changes made with the help of refactoring tools - code
cleanups, extractions of helper methods or interfaces, import optimizations, etc. - the time and
effort it takes to review them is constant, no matter how many files are affected.

### Well-scoped logic changes: O(n)

When it's clear what a pull request is trying to communicate and all changes
are in line with that narrative, the effort it takes to review those changes indeed grows linearly
with the size of the diff. I might make a few passes over the code if it's particularly tricky, but
as long as the changes are consistent with the main goal of the pull request, it's usually 
straightforward to review. It's when unrelated changes creep into a pull request things start 
getting complicated.

### Poorly-scoped logic changes: O(m * n)

(where m is the number of unrelated areas the pull request affects) 

Whenever I notice a change in an 
unrelated area during a code review, I backtrack and try to identify other changes affecting that 
same area, and that usually takes an entire pass over the entire diff. I'd repeat that for every 
unrelated change I discover, hence the total number of passes is proportional to the number of 
unrelated areas the pull request touches. But even in cases when m is small (in practice this would 
sometimes manifest in the form of a fly-by refactoring mixed into an otherwise well-scoped pull 
request), unrelated changes introduce a lot of confusion, making it easier for a reviewer to miss 
important parts of the diff. Just as a simpler algorithm with slightly worse runtime performance 
would often be preferred to a faster but much more intricate one, extra complexity comes at a high 
cost in code reviews!

### Everything, everywhere, all at once: O(n!)

Pull requests that mix changes in different areas, refactoring, random code cleanups and everything
in between, are virtually unreviewable. It's very easy to lose focus while reviewing this type of
pull requests, thus spending a large number of review passes to make sense of the diff. There is a
very high chance of missing important logic changes when they're wedged between otherwise perfectly 
valid code style improvements. The only good way to tackle a pull request like that is to kindly ask 
the code author to split it into smaller, well-scoped, more digestible incremental changes.

And this brings me to my main point: the total time and effort needed to review a stack of 
well-scoped, incremental pull requests will almost always be much smaller than the time and effort
it would take to review all of these changes in a single diff. During feature development, you'll
most certainly encounter many O(1) changes - send them in as separate pull requests and get instant
reviews! Instead of piling up code into a single O(m * n) pull request, split it into a stack of 
O(n) pull requests - it'll most likely take less effort to review, and reviewers will feel a lot 
more confident about their approvals.

Lastly, I wanted to mention [Graphite][graphite], the tool we use to manage stacks of pull requests.
PR stacks aren't that easy to manage, as you constantly need to rebase parts of a stack whenever
you make changes to fix the build, address code review feedback, etc. Graphite's CLI simplifies this 
task. Here are the commands I use pretty much every day:

- `gt track` - whenever I branch off of the current branch to continue making incremental changes,
I tell Graphite to track my new branch and select its parent. That way Graphite can keep track of
my PR stack.
- `gt restack` - making changes to a pull request inside a stack requires rebasing all subsequent 
pull requests, which is a lot of manual work. "restack" does it automatically!
- `gt submit` - once the stack is ready I'd run this command to have Graphite automatically
push code to the repository and open pull requests. Graphite includes links to other pull requests 
in the stack in the body of every request, making it easier for reviewers to navigate between 
related requests.

[great-teams-merge-fast]: https://saket.me/great-teams-merge-fast/
[graphite]: https://graphite.dev/
