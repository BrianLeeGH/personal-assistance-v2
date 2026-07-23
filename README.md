# Personal Assistance v2

Hermes Agent profile distribution for a single-user personal assistant.

## Install

Install `hermes-soul-manage` into the Hermes Python environment first, then:

```bash
hermes profile install \
  https://github.com/BrianLeeGH/personal-assistance-v2.git \
  --name personal-assistance-v2 \
  --yes
```

Configure the model credentials for the installed profile using the normal
Hermes model and secret management flow.

ChatHub applications use this profile as a `PerUserClone` template.

