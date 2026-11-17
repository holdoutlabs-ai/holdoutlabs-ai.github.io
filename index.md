---
layout: home
---

Holdout Labs builds small, fast models that make decisions inside games.

**GameDecider** is our first release (v0.1, Apache-2.0). You give it a scene, one or more questions, and a
set of options for each question. It returns a calibrated probability for every option in a single forward
pass, with no text generation. When it isn't confident enough it abstains, so your game can fall back to its
own rules. It runs locally, and the runtime makes no network connections.

Typical questions: "Is the player threatening this NPC?", "Did the player accept the quest?", "How does the
shopkeeper feel now: pleased, wary, angry or afraid?"

- [Model, runtime and documentation on Hugging Face](https://huggingface.co/holdoutlabs/gamedecider)
- [Try it in the browser](https://huggingface.co/spaces/P1ayer-1/gamedecider-playground)

The name is about how we test: models are scored on held-out data they were never trained on.

Questions, or a game you'd like to try it in: [noah@holdoutlabs.ai](mailto:noah@holdoutlabs.ai).
