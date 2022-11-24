[[_TOC_]]

Milestones in gitlab are essentially one order of organization above issues. Currently, we have decided to use them in two general ways (at two different levels of organization), though this could potentially evolve.

### Tracking a specific goal that is slightly too large for one or two tickets

This is the more straightforward use (in my view at least, apologies to those that disagree). Milestones like %4, %5, %15 fit fairly cleanly into this category. All the issues tracked by this milestone are a part of a specific larger project, with a given set of requirements and clear, defined measure of when the milestone is complete (well, we're working on that part). A general rule of thumb is that if the requirements go beyond 2-3 tickets, they should be linked by a milestone as to not get too sprawling and hard to track. This provides a resource both to developers and to users to keep track of the various components of a complex request.

A couple best practices:
- Any tickets related to a goal-specific milestone should be assigned to said milestone.
- Milestone description should be a high-level but broad description of its goal that effectively summarizes it in a way readable to both developers and non-developers.

### "Meta-Milestones" for longer-term tracking

We've also decided to have slightly more vague, overarching milestones, which we have currently named "meta-milestones". These currently fit into two categories
- Tracking goals and progress by quarter, see %13. Both internally and externally, this serves to keep us organized and informed on what work is going on within the group and to help ensure that we make progress in areas that we had hoped to, and better track longer-term goals.
- Tracking goals and progress by domain, see %7. Again this serves as both an internal and external-facing organizational tool to track ongoing work, but in this case in a specific domain. Currently DNA-focused work is the only domain in which this has been used.

Best practices:
- Description section of milestone is essentially all that is used, mapping out work in bullet points, checkboxes, etc (see above linked milestones as examples)
- If goal-specific milestones fall under the umbrella of a meta-milestone (which they almost always should, at least in the case quarter-planning meta-milestones), they should be linked in the body of the meta-milestone.
- Any tickets related to a meta-milestone (that are not assigned to a goal-specific milestone) should not be assigned to said milestone, but just linked to within milestone description.

 
