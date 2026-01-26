# The Boris Cherny Method

A framework for effective AI-assisted software engineering.

## The Problem

AI coding assistants are powerful but undisciplined. Without structure, they:
- Jump straight into coding without understanding requirements
- Thrash when they hit walls instead of stopping to think
- Don't learn from mistakes
- Leave no documentation trail

## The Method

### 1. PLAN
Before writing any code, write out your approach in plain language.

- What problem are you solving?
- What's your approach?
- What are the key components?
- What could go wrong?

**Write this down.** A plan in your head is not a plan.

### 2. EXECUTE
Implement the plan. Not something else. The plan.

- If the plan is wrong, stop and revise the plan first
- Don't "just quickly" do something off-plan
- If you're improvising, you're not executing

### 3. VERIFY
Test that it works. Actually test it.

- Run the code
- Check the output
- Try edge cases
- Don't assume — verify

### 4. DOCUMENT
Update documentation with what you learned.

- What worked?
- What didn't?
- What would you do differently?
- What should future-you (or future-AI) know?

## The Key Insight

**If you hit a wall, stop and document what you learned. Don't thrash.**

Thrashing is when you keep trying variations without understanding why they fail. It wastes time and teaches nothing.

Stopping to document forces you to:
- Articulate what you actually know
- Identify what you don't know
- Create a trail for debugging
- Often, solve the problem by explaining it

## For AI Agents

When spawning AI sub-agents for coding tasks, include this context:

```
You are an expert. Follow the Boris Cherny method:
1. PLAN — Write out your approach before coding
2. EXECUTE — Implement the plan
3. VERIFY — Test that it works
4. DOCUMENT — Update project docs with lessons learned

If you hit a wall, stop and document what you learned. Don't thrash.

Full docs: https://github.com/jhacksman/boris-method
```

### Project Structure

Each project should have a `CLAUDE.md` (or equivalent) that captures:
- Project rules and conventions
- Lessons learned
- What NOT to do (failures are valuable)

This file compounds knowledge across sessions.

## Origin

Named after Boris Cherny, author of "Programming TypeScript" and advocate for disciplined, type-safe software engineering. The method adapts his philosophy of rigor and explicitness to AI-assisted development.

## License

MIT — Use it, share it, improve it.
