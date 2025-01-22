---
title: Shotgun surgery - How to avoid it and achieve safety and high productivity
date: 2024-09-18
categories: [Programming, Good practices]
permalink: /2024/09/shotgun-surgery-how-to-avoid-it-and-achieve-safety-and-high-productivity/
image:
  path: /assets/img/2024-09-18/featured.png
---
Imagine an engineer tasked with updating a feature who makes changes across the entire codebase without a clear plan or structure. Instead of committing small, focused updates, they introduce a massive set of changes at once, touching numerous files and modules. Without TDD or incremental commits, tracking all these modifications becomes overwhelming, making even simple tasks like code reviews or context switching costly and confusing. This disorganized approach, known as shotgun surgery, increases the likelihood of bugs and errors, making the code harder to maintain. In this article, we’ll explore how to avoid this pitfall and improve both productivity and code safety.

## Causes
### Bad design
Shotgun surgery is considered one of the classic code smells. It refers to a situation where introducing a change or feature requires modifying many different files. If this happens frequently, it likely indicates an underlying design issue, such as violations of the Single Responsibility Principle, lack of cohesion, or related logic being scattered across various places. In some cases, we may be forced to make many changes at once. However, in this article, I want to focus on a different perspective: when the engineer has the choice to work more efficiently but ends up resorting to shotgun surgery due to their working style.

### Ineffective and chaotic approach
Imagine an engineer who needs to implement Feature A. They dive into the code and modify numerous files, affecting various parts of the system. There is no structured plan for how to approach the changes, and the implementation lacks manageable steps; everything is done at once. The engineer doesn’t follow TDD practices and writes all the tests at the end, or perhaps even skips them entirely. While there may be more than one commit, the commits are chaotic, lacking any logical separation. The first commits are often the largest, while subsequent ones consist of random fixes for bugs, typically caught through manual testing.

## Consequences
### Diminished productivity
When an engineer is working on a feature and has just modified many files, it becomes difficult to keep track of all the changes mentally. Any interruption, such as a new message, email, code review request, or meeting, makes context-switching costly. The engineer has to revisit the same areas repeatedly, gather information, recall the full context, and remember what has been done and what still needs to be completed. The status of the current implementation becomes unclear, which can be overwhelming. Adding to this, if tests are written only at the end, there is no immediate feedback to confirm that the changes are correct.

### High potential for bugs and regressions
Using this chaotic approach makes it easier to introduce regressions and potential bugs. It’s difficult to keep track of numerous changes at once, maintain a high level of focus, thoroughly think through each change, and write effective tests to verify them.

## Prevention
### Divide and conquer
It’s like an old rule that always proves useful: it’s hard to tackle a full problem at once, so we need to break it into smaller pieces. Given a task, we can plan the necessary changes and create a TODO list.

**Example:**

Let’s say the task is to send a new notification to a user when an AI agent successfully responds for the first time:
- Prepare translations for the email (we need to wait for these, so it’s good to have them prepared in advance)
- Dispatch a new event in the AIAgent module (Successful response)
- Prepare a mechanism to send notifications only once (such as a log)
- Create a template for the notification
- Implement a service to send the notification
- Implement a handler for the event and use the service to send the notification

### Apply Test Driven Development
Before each implementation, write a test. TDD forces us to think about a small change that can be tested immediately, making it the best way to avoid shotgun surgery. It also provides additional benefits: fast feedback, minimal manual testing, and safety when making further changes. It helps maintain better focus on edge cases when the scope is relatively small and significantly speeds up development because there’s no need to run the entire project. I usually rely solely on automated tests and, in the end, check the entire feature and its interaction with other parts, like the front end, just to ensure the product feels cohesive.

### Isolate change
It would be ideal to push code directly to the master branch (Trunk-Based Development), but this isn’t always possible due to varying company rules and practices. However, even if we work in an environment where we need to wait for manual testing of the full feature, we can still isolate safe changes that don’t impact any currently working code in production. For instance, using the previous example, we can push all changes except the last one, which can then be manually tested in accordance with the company’s rules. This approach is relatively safe because these changes are isolated and won’t affect the production environment.

### Commit Frequently
To keep the codebase clear and manageable, commit changes after each logical step. This creates a good commit history and, more importantly, helps us move on from one change and focus on the next step.

### Modularity
Writing modular code aids in planning the steps required to introduce a particular change to the system. By breaking the code into distinct, manageable components, we can more easily identify which modules need to be modified and how those changes will impact the overall system. This structured approach makes it simpler to isolate and address specific issues, ensuring that each change is well-defined and contained. Furthermore, modularity enhances the clarity of the development process, allowing for more precise and incremental updates.

## Conclusion
In conclusion, avoiding shotgun surgery is crucial for achieving high productivity and maintaining a positive development environment. By adhering to practices such as modularity, frequent commits, and Test-Driven Development (TDD), teams can significantly enhance their efficiency and code quality. These practices not only improve the safety of the codebase but also contribute to the happiness of engineers. By minimizing context switching and reducing the cognitive load associated with large, disruptive changes, engineers can focus on their work with greater clarity and less frustration.

By making incremental, logical changes and committing frequently, engineers can provide their teammates with timely feedback, which helps to unblock them and keeps the development process flowing smoothly. Fast feedback during code reviews accelerates the integration of changes, prevents bottlenecks, and fosters a more collaborative and supportive team environment. In this way, minimizing the cost of context switching and streamlining code reviews contribute to a more cohesive and high-performing team, ultimately leading to a more efficient, predictable, safer, and less disruptive development cycle.
