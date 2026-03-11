---
description: Detect the project's language, framework, and tooling, then generate autoimprove.config.md which configures the measurement suite for this specific codebase. Run this once before using /autoimprove:improve in a new project.
---

Run the detect-stack skill to analyse this project and produce `autoimprove.config.md`.

After generating the config, automatically run the measure skill to show the baseline score.

End with:
"✅ Setup complete. Run /autoimprove:improve to start the autonomous improvement loop."
