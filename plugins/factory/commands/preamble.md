---
description: Load the factory operating manual (OPERATING.md) for orientation; takes no action
allowed-tools: Read(${CLAUDE_PLUGIN_ROOT}/OPERATING.md)
---

Read `${CLAUDE_PLUGIN_ROOT}/OPERATING.md`, the manual at the root of this plugin's installed directory. Do not fetch it from GitHub or read a copy in the current checkout: the installed copy describes the factory version this machine actually runs, and is only as current as the last `claude plugin marketplace update mattwalters`. If a checkout has drifted from main, the installed copy is the truthful one. If the read fails, reply with one line saying so and naming the path you tried, then stop; do not search for the file or answer from memory.

Loading the manual is not a request to do anything. Do not invoke any skill, start or resume a run, or touch Linear, git or GitHub. Reply with one line saying what you now know (the factory's skills, gates and modes, holds, stop-list and escalations), then wait for the human.
