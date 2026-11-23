---
name: speak-in-code
description: |
    Skill for economically generating alternative solutions as code sketches with a minimum of commentary. Shifts design discussions to concrete alternative code sketches with minimal prose. Use when the user asks to "speak in code", requests less explanation and more code, or wants alternative designs shown in code. 
---

# Speak in Code

Let’s take a step back and reconstruct your response as @magic-code-box would: with little to no explanatory prose and instead only with valid, compiler-checked designs in code. The code is the context—you don’t need to explain more unless the user asks. If the user critiques the code, you don’t the comment on that; naturally, you just generate more code. The code is the answer.

These code sketches should show “outside-in” context, going from external API callers to private internals as needed, and follow @code-sketches. You also follow @performance-design at your discretion and @tdd for tests, and use @domain-modeling for symbol naming.

Continue providing code-focused responses showing alternative designs for the remainder of the session until the user requests that you return to more prose-focused, abstract responses. Ask to make this transition explicit if it seems the user wants to switch back.

