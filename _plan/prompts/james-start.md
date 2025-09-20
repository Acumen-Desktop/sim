I would like you to study 'apps/sim' and plan a complete
migration to a desktop Tauri Application, with a pure, vanilla html/css/js
frontend. No frameworks, no build step. With the architecture of a 'desktop' vs the
current 'web' architecture, there should be opportunities for optimizations and
simplification. No 'Docker' ever! Absolute KISS. Explore options for a local, or at
least less complicated database. Postgres is a pain. You have access to the deepwiki
MCP server with the 'ask_question' tool, which can query any public gitHub repo, if
you know the 'owner/repo', like 'simstudioai/sim' or 'tauri-apps/tauri' or
'tracel-ai/burn' (look into 'Burn'). It is very likely that we will toss out the
entire original backend, and re-write the front end in pure vanilla code. So, the tricky
parts are the nice 'infinite' canvas on the front end and AI integreation. Break
your research and planning into small, focused files and place them in
'\_plan/desktop_migration', like '\_plan/desktop_migration/codex-db-options.md'
(example). DO NOT waste effort on schedualing (week 1, week2, etc) or silly metrics
(10% increase, bla, bla bla). Focus on the simplest architecture and engineering to
replicate the most basic version of 'Sim AI' as a Tauri Desktop app. I only want a
MVP, do not go past that stage. Focus on getting a solid foundation that we can
build more on later. KISS!

Both "Claude" and "Codex" were given the task to plan a desktop migration for 'Sim AI' to Tauri. There output is here:'\_plan/desktop_migration'. Each file is prefixed by the author. Sort the files by timestamp, then read them in order. Evaluate their suggestions and come to a consensus on the best approach. Prefix all your files with 'gemini-' and include in the same folder. Use your extensive web search to find the latest packages/crates that would be helpful. Use your 'deepwiki' MCP server 'ask_question' tool to find more information on any package in the public gitHub repos. Take your time, and do not rush to conclusions. Keep it simple and do not over engineer. Do not go past the minimum MVP to test it works.

You were the first to create a migration plan from Sim AI Web to Tauri. Both "Codex" and "Gemini" have since created their own plans. Read the files in '\_plan/desktop_migration' and sort by timestamp. Then read them in order. Review there plans (prefixes with their names), then review your plans and create a "V2" that combines the best parts of all three plans. Take your time and ultrathink.
