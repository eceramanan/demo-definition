# Competition example

`training.json` in this directory is an alternative exercise for the same
sandbox: three scored rounds, 225 points, per-competitor flags. It is not used by the
platform automatically - upload it as its own Linear Training Definition, the
same way the demo one is uploaded.

The repository root `training.json` remains the demo exercise. Nothing here
affects a sandbox build.

## What the sandbox must provide

`competition-training.json` is the exercise. It only works if the sandbox
actually contains the two files it asks for. Two changes are needed in the
sandbox definition.

## 1. variables.yml (repo root) — per-competitor flags

This is the mechanic that matters for a competition: without it, the first team
to solve a round can broadcast the flag and everyone else scores for free.
APG generates a different value per sandbox, so a copied flag does not match.

```yaml
round3_flag:
    type: text

round4_flag:
    type: text
```

The names must match `answer_variable_name` in the JSON exactly
(`round3_flag`, `round4_flag`), and each level has `variant_answers: true`.

## 2. provisioning/playbook.yml — place the flags

Add to the `Set up the server` play. The answers in the JSON are placeholders
(`NCSC{set-by-apg}`); the real value comes from the variable at build time, so
the file and the expected answer are generated from the same source.

```yaml
    - name: Round 3 flag, readable by the participant
      copy:
        dest: /home/participant/round3.txt
        content: "{{ round3_flag }}\n"
        owner: participant
        group: participant
        mode: '0644'

    - name: Round 4 flag, root only
      copy:
        dest: /root/round4.txt
        content: "{{ round4_flag }}\n"
        owner: root
        group: root
        mode: '0600'
```

Order matters: both tasks must come *after* `Create the participant account`,
or the home directory and the owner do not exist yet.

## Verify before a competition

- Two sandboxes from the same pool hold **different** values in
  `/home/participant/round3.txt`. If they match, APG is not wired up and the
  competition is broadcastable.
- `sudo -l` as `participant` on the server lists sudo rights (round 4 needs it).
- The client can reach `192.168.20.5` (rounds 2-4 all depend on the hop across
  office-router and dmz-router).

## Platform limits worth knowing

- `max_score` is capped at **100 per level**. Uploading a level worth more is
  rejected with "Level field 'maxScore' cannot be greater than 100". Differentiate
  rounds within that ceiling (this example uses 50 / 75 / 100) rather than by
  scaling points up.

## Not yet proven on this platform

- `answer_variable_name` / `variant_answers` and the `variables.yml` wiring are
  documented but have not been run here. `variables.yml` is currently `{}` and
  every build so far used a fixed flag.
- Scoring, hint penalties and the solution-zeroes-the-round behaviour are the
  platform's own and should be confirmed once with a throwaway run before a
  real competition.
