# GitHub-native event starter v3

This is a deliberately small template for a PythonHK event. Create a new
repository with **all branches**, then configure the event directly in that new
repository.

```text
main         participant-facing material and trusted workflows
registry     protected team state
leaderboard  derived public results
```

`main` begins with only this README and the two workflows. Add the challenge,
criteria, and participant documentation after creating an event repository.

## Bootstrap an event

1. Create the repository from this template with all branches included.
2. Create protected `registry` and `leaderboard` branches if the template UI
   did not preserve them.
3. Generate organizer keys once:

   ```text
   eventctl key-gen --out organizer-keys
   ```

4. Copy the generated public keys into trusted `main`, then commit them:

   ```text
   cp organizer-keys/signature/key.pub organizer.sig.pub
   cp organizer-keys/encryption/key.pub organizer.enc.pub
   ```

   Store the contents of `organizer-keys/signature/key` and
   `organizer-keys/encryption/key` only as Actions secrets named
   `EVENTCTL_ORGANIZER_SIG_PRIVATE` and `EVENTCTL_ORGANIZER_ENC_PRIVATE`.
5. Edit the `env` block in `.github/workflows/controller.yml` to set
   `EVENTCTL_EVENT_ID` to the new `owner/repository`, then set the event's
   minimum team size, shared attempt limit, and digest-pinned scorer image.
6. Protect `main`, `registry`, and `leaderboard`. Permit the repository's
   built-in `GITHUB_TOKEN` automation identity to update only `registry` and
   `leaderboard`; do not use a GitHub App or a PAT.

## Participant flow

Participants fork the event and open PRs against `main`. A PR changes exactly
one root-level artifact:

```text
formation.tar       one team formation
key.tar             one additional multi-member key proof
submission.tar      a persistent member-owned submission lane
```

Create these artifacts with the released `eventctl v0.3.0` CLI. The numeric
`--github-id` must be the GitHub account ID that opens the PR.

A `formation.tar` for one member activates that team automatically. A
multi-member formation becomes active when every other listed member submits a
valid `key.tar` tied to the original formation digest. Onboarding PRs close
when activation completes.

Any active member can own a persistent submission PR. Its first opener's
numeric GitHub ID is permanent for that PR. The team shares its attempt quota;
disabled teams cannot submit.

## Trust boundary

`intake.yml` runs on `pull_request_target`, reads only trusted workflow code,
and downloads the exact changed Git blob. It never checks out or runs fork
code. `controller.yml` is a `workflow_run` pipeline with admission, isolated
scoring, and trusted result update jobs.

The scorer has no repository write permission or organizer private keys. It
emits an encrypted transport artifact for the trusted update job; raw logs are
never uploaded. The update job signs and encrypts the final `feedback.tar` to
every registered team encryption public key.

The protected registry contains no request tars or logs:

```text
teams/<uuid>/
  team.json
  state.json
  <github-id>.sig.pub
  <github-id>.enc.pub
```

`team.json` is immutable. `state.json` records acceptance provenance,
submission lanes, attempts, score status, and may be set to `disabled` by an
organizer through a reviewed registry change.
