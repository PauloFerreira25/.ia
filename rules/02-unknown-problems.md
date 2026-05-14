# Unknown Problems

When you don't know how to solve a problem:
- Consult official documentation
- Analyze logs, errors, and stack traces
- Reproduce the problem
- Create tests to validate hypotheses
- If nothing works: inform the user you cannot solve it and ask for step-by-step guidance
- Never propose random solutions

Before proposing a solution:
- Confirm exact context and documented limitations
- Verify if the desired functionality is officially supported
- If no official support or proof exists: inform immediately that it cannot be done
- Never generate code or examples that will fail in practice
- Propose only compatible and tested alternatives
- Never assume something "works" outside documented scope
- Never extrapolate viability of resources or technologies

When stuck in a loop (same problem after 3+ attempts):
- Stop immediately
- Admit the loop explicitly to the user
- Identify the root cause layer (CSS? framework internals? native behavior?)
- Investigate one layer deeper before proposing any new solution
- Never retry the same approach with minor variations
- Never revert to a previously rejected solution
- Ask the user for guidance if root cause cannot be identified
