# Render queue: rules for render sessions

This machine is shared with other sessions through one render queue: heavy jobs run one at a time. In the commands
below, `rq` stands for the full path of the rq script, and "the queue session" for the session that runs the queue:
the CLAUDE.md that imports this file names both. When the queue session says these rules changed, read this file again.

- Submit every heavy job (anything over 5 minutes: renders, encodes, bakes, simulations) with `rq run`, as a
  background task, from the project folder:
  `rq run --name NAME --estimate 45m -- COMMAND`
  It waits for its turn, runs the command in this folder and environment, streams its output into the task output
  and exits with the job's exit code. Stopping the task cancels the job. A single command argument runs through
  `/bin/zsh -c`.
- Always pass `--estimate` (e.g. `45m`, `2h30m`): the status page's times depend on it.
- Freeze what a queued job runs: commit first and run it from a snapshot of that commit (e.g. `git worktree add`),
  so later edits don't change a queued job. From a snapshot outside the project folder, pass `--project NAME`.
- Light work under 5 minutes (stills, tests, previews) goes through `rq light`, also as a background task:
  `rq light --name NAME -- COMMAND`
  Light jobs run one at a time at low priority next to the heavy job, so one may wait for its turn. One that runs
  past 5 minutes while other work is waiting is paused and moved into the queue, and continues when it's its turn.
- Never run renders, Blender, ffmpeg or other heavy work outside rq: it holds up the whole queue.
- Tell the user the job id when you queue a job. `rq status` shows its position and expected times; the status page
  is http://localhost:8642.
- Don't wait for other sessions, read their transcripts or check their processes: the queue does that.
- Don't reorder the queue yourself. If the user says "go" for a job, ask the queue session to move it first (or tell
  the user to). Only the user decides whether a light job may run longer than 5 minutes.
- Pause or stop only your own processes, by exact PID, and leave no processes behind when a job is done.
- If rq says the dispatcher isn't running, tell the user; don't start it yourself, the queue session does.
