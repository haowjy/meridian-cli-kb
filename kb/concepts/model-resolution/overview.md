# Model Resolution

Mars resolves aliases, harness routes and launch policy; Meridian consumes the launch bundle instead of guessing harnesses from model names. The model identity and the harness-native token may differ, so launch must project the token for the selected route. See [model resolution decisions](../../decisions/model-resolution.md), [aliases and routing](aliases-and-routing.md), and `docs/configuration/mars.md`.
