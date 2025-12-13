+++
date = '2025-12-12T10:41:55+02:00'
draft = false
title = 'How to Solve Hard Problems'
+++

A friend once asked me how I actually think when I try to solve a problem.

I realized that I had never consciously reflected on this. After some thought, I noticed that my approach tends to follow a small set of recurring principles. Below are the core ideas that guide how I approach problem-solving.

**General principles:**
1. Always aim for the simplest solution that fully addresses the problem. A short, clean solution is almost always preferable to a complex one that achieves the same outcome. Introducing unnecessary complexity is usually a sign that the problem has not been fully understood.
2. Take breaks.

---

## 1. Understand

1. Before attempting to solve anything, ask whether the problem actually *needs* a solution--or whether the need for the problem can be removed entirely.
2. Make sure you truly understand the problem. Identify ambiguities, unclear requirements, or hidden assumptions, and resolve them first.
3. If the problem genuinely needs solving, focus on *why* it exists. A good solution is built around the underlying reason the problem matters, not just its surface symptoms.

---

## 2. Abstract

1. Do not get overwhelmed by complexity, jargon, or formal wording. No matter how complicated a problem sounds, its core idea can usually be expressed in one or two simple sentences. Identify that core.
2. Strip away irrelevant details and think in terms of concepts rather than implementations.
3. Visualize the problem--through sketches, diagrams, mental models, or simple examples. If you cannot explain it visually or intuitively, you likely do not understand it well enough yet.

---

## 3. Divide

1. Break the problem down into smaller, independent subproblems. Each subproblem should be solvable on its own without tightly coupling it to the others.
2. Define clear interfaces between these subproblems. Good boundaries reduce cognitive load and prevent accidental complexity.
3. Solve the subproblems one at a time. Focus only on the current piece instead of the full problem space.
4. Continually simplify each sub-solution. Complexity compounds quickly; simplicity scales.

---

## 4. Implement

1. Only start implementing once you have done the necessary thinking. If you begin coding or building before truly understanding the problem, you will end up iterating over the entire solution far more often than necessary, repeatedly running into issues that could have been anticipated.
2. Even if you believe you have found the simplest and most correct solution upfront, implementation will still expose flaws or gaps in your reasoning. Ideally, these revisions should be limited to individual subproblems rather than forcing you to rethink the entire solution.
3. The implementation should clearly reflect the decomposition of the problem. Solutions to individual subproblems should be cleanly separated and loosely coupled, allowing parts to be refined, replaced, or extended without cascading changes across the whole system.

---

## 5. Verify

1. No matter how confident you are in a solution, you only know whether it works by verifying it. Assumptions without validation are indistinguishable from errors.
2. Start by testing the subproblems in isolation--this can and should happen while they are being implemented. Once each part behaves correctly on its own, test the interactions between them to ensure the complete solution works as intended.
3. Verification is not just about correctness but also about robustness. A solution that only works under ideal conditions is rarely sufficient in practice.

---

## 6. Reduce

1. Ideally, this step should be applied continuously—starting before implementation and revisited afterward. Actively remove complexity, unnecessary features, and special cases, and verify that the solution still works.
2. Question whether each component is truly required. Many problems appear difficult only because they include functionality that is not essential to solving the core problem.
3. A good solution often becomes clearer after reduction. If removing a part does not break the solution—or even improves it—that part was likely noise rather than value.

---

## Conclusion

Problem solving is rarely about clever tricks or complex solutions. In practice, it is mostly about resisting complexity and being disciplined in how you think. The steps described above are not meant to be followed strictly in order--they often loop back into each other. Understanding leads to abstraction, abstraction reveals better divisions, implementation exposes gaps, verification forces clarification, and reduction simplifies everything again.
Most difficult problems become manageable once they are stripped down to their core. If a solution feels complicated, that is usually a signal that something is not yet understood well enough.
Over time, this process becomes less explicit and more intuitive. But the principles remain the same: understand first, solve what actually matters, and remove everything that does not.
