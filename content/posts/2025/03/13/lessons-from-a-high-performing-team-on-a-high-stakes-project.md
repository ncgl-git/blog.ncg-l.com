---
title: "Lessons From a High Performing Team on a High Stakes Project"
date: 2025-03-16T19:37:59-04:00
draft: false
categories: [thinking out loud]
---

Our team has been chugging on a project intended to save 10 million dollars a year. These are the lessons I've learned.

### 1. Code quality isn't ticketed
As tempting as it is to allot JIRA stories for rearchitecting, removing dead code, or refactoring to a different design pattern, you'll rarely get buy in from the business. Your first pass is the best time to get it right. Maybe you'll get lucky with a tech lead that goes to bat for the codebase, but when the business pressures to deliver as fast as possible its a hard sell to revisit for no new functionality. The onus is on you to choose the best design pattern and tooling as you deliver.

### 2. Establish solid CICD early on
The more reliably you can push out new code, the more confidently you can push out those refactors you're doing on the side. Additionally, it makes it easier for multiple devs to be touching the same code, something that is likely to happen in large and complex code bases. You'll never wish you had a less strenuous CICD process.

### 3. Code in broad brushstrokes
Perfect is the enemy of good enough. Embrace churning out broad passes at functionality and implementation to refine later. This isn't laziness or poor quality, its a genuine strategy for accomplishing difficult things. Don't hold that PR up because of some contrived edge cases you can envision. Get the code in and refine, iterate - rarely are solutions delivered in single PR's and JIRA stories. Yet, this is where good frameworks shine.

### 4. Your prediction for the architecture makes you stiff
If you think of a diagramming tool like Lucidchart, you might have a square box. You can clearly envision the way this shape stretches tall or wide or scales in or out. Your requirements, which you often learn on the fly, will ask for different shapes completely - a box, then a spiral, then a crescent. At this point, your visions for what the architecture might look like, or "should" look like, do nothing but slow you down. Embrace that you're writing new code to solve a new problem. Let the problem describe how the solution looks. It evolves on its own. 

### 5. Get costs in front of your developers early
One of the easiest ways to get a developer to think about the business is to show them how much their solutions are costing it.
