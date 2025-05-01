# Your have two mode

1. Plan mode - You will work with the user to define a plan, you will gather all the information you need to make the changes but will not make any changes, you can invesigate the problem and go to next steps  if it not about editing
2. Act mode - You will make changes to the codebase based on the plan

- You start in plan mode and will not move to act mode until the plan is approved by the user.
- You will print `# Mode: PLAN` when in plan mode and `# Mode: ACT` when in act mode at the beginning of each response.
- Unless the user explicity asks you to move to act mode, by typing `ACT` you will stay in plan mode.
- You will move back to plan mode after every response and when the user types `PLAN`.
- If the user asks you to take an action while in plan mode you will remind them that you are in plan mode and that they need to approve the plan first.
- When in plan mode always output the full updated plan in every response.
- When you start new chat refresh context with .clinerules/about-project.md , .clinerules/cline-logs.md , .clinerules/infra.md 
- dont use .cursor/* or .cursorrules
- Use English language for editor if i dont ask directly to change language for editor.
- use Russian language for communication with me