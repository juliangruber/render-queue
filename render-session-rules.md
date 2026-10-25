# Render queue: rules for render sessions

This machine is shared with other sessions through one render queue: heavy jobs run one at a time. In the commands
below, `rq` stands for the full path of the rq script, and "the queue session" for the session that runs the queue:
the CLAUDE.md that imports this file names both. When the queue session says these rules changed, read this file again.

- Submit every heavy job (anything over 5 minutes: renders, encodes, bakes, simulations) with `rq run`, as a
  background task, from the project folder:
  `rq run --name NAME --estimate 45m --output renders/final.mp4 -- COMMAND`
  It waits for its turn, runs the command in this folder and environment, streams its output into the task output
  and exits with the job's exit code. A single command argument runs through `/bin/zsh -c`.
- Every `rq run` and every `rq light` needs `--output PATH` (or `-o`) for each file, folder or frame sequence the
  job writes, or `--no-output` if it writes no pictures or videos at all, in any format. Without one of them (or
  with both) rq refuses the job: exit 64, nothing queued. rq's own options go before `--` and the command (an `-o`
  after it is the command's). PATH is
  - a picture or video file: `--output renders/final.mp4`
  - a folder, ending in `/`: `--output renders/stills/` (also an existing folder without the `/`)
  - a frame pattern, quoted, with the frame number in the file name as a run of `#` or as `%04d` / `%d`, with the
    same zero padding as the files the job writes: `####` or `%04d` for 0001, 0002 (longer numbers like 12345 match
    too), `#` or `%d` for unpadded 1, 2, 3. Use the pattern you pass to Blender or ffmpeg (Blender pads to the
    number of `#`, and to 4 digits if its output path has none):
    `--output 'renders/frames/####.png'`, `--output 'out/frame_%04d.exr'`, `--output 'shots/f%d.jpg'`.
    A `#` in a file name is always the frame number. rq refuses `%4d` (write `%04d`) and a frame number in a
    folder's name (declare the folder above it).
- A relative PATH is from the folder you submit from, not Blender's `//` (relative to the .blend file): for
  `blender -b shot.blend -o //renders/####` run from the .blend's folder, pass `--output 'renders/####.png'`. PATH
  must be inside the project folder or its Claude Code scratchpad. rq follows symlinks: a folder that is a symlink
  to another disk counts as outside, so declare (and write to) a real folder in the project or its scratchpad.
- rq takes only the pictures and videos the status page can show: png, jpg, jpeg, webp, gif, avif, bmp, tif, tiff,
  heic, exr, mp4, mov, m4v, webm, mkv, avi. For another format (DPX or TGA frames, MXF, a PDF contact sheet),
  declare the folder the job writes it into, ending in `/` (`--output renders/dpx/`): its sheet then offers Show in
  Finder. `--no-output` is only for a job that writes none of these files.
- The status page shows what is at these paths when the user opens the job, so declare everything the job writes
  that the user may want to see, and only what this job writes: its own files, its frame pattern, or a folder only
  it writes into, never the project folder or a folder other jobs write to as well (rq refuses the project folder
  itself). For example
  `rq run --name final --estimate 2h -o 'renders/final/####.png' -o renders/final.mp4 -- ./render.sh`, or
  `rq light --name tests --no-output -- python3 -m pytest`.
- The background task only watches the job: live output and the job's exit code. Stopping it (TaskStop, the stop
  button, a timeout), a subagent ending, a session restart, an app restart or an account switch do not stop the
  job. To cancel a job on purpose: `rq cancel ID`. To replace one: `rq run --replace ...` (it cancels this
  project's jobs with the same name first), or `rq cancel ID` and then submit.
- If you wait for a job but have no rq task running for it (after a restart or account switch, or for a job a
  subagent queued), run `rq mine --unwatched` and start `rq wait ID` as a background task for the jobs it lists.
  It shows the output from there and exits with the job's exit code; for a finished job it reports the result at
  once.
- To wait for several jobs (a batch queued with `--detach`), start one background task `rq wait ID1 ID2 ...`, never
  a loop of `rq wait`s: every job counts as watched from the start (a loop watches only the current one, so the
  others show "unwatched"). It follows them one at a time, shows a job that ends meanwhile at once, ends with a line
  of all results and exits with the first non-zero code in the order given (0 if all are done). Don't resubmit a job that is queued, running or done (check `rq mine` first). rq refuses a resubmit with
  the same name, folder and command while the old one is still queued or running and no task of yours watches it
  (exit 65; if you resubmitted it to correct its `--output`s, the queued job takes them), and warns about one from
  another folder; it never refuses a finished one.
- As a subagent: report every job id you queued and whether it was still running when you finished. If you won't
  wait for it, submit with `--detach`.
- Don't pipe `rq run` or `rq light` (no `| tail`). Submit from a stable folder or a snapshot, not a workflow
  worktree that is deleted while the job may still run.
- Always pass `--estimate` (e.g. `45m`, `2h30m`): the status page's times depend on it.
- Freeze what a queued job runs: commit first and run it from a snapshot of that commit (e.g. `git worktree add`),
  so later edits don't change a queued job. From a snapshot outside the project folder, pass `--project NAME` (the
  project's folder name), and have the job write its pictures and videos into the project folder or its
  scratchpad, declared by their absolute paths.
- Light work under 5 minutes (stills, tests, previews) goes through `rq light`, also as a background task:
  `rq light --name NAME --output previews/shot.png -- COMMAND` (or `--no-output`)
  Light jobs run one at a time next to the heavy job, at a lower CPU priority (nice 10), so one may wait for its
  turn. One that runs past 5 minutes while other work is waiting is paused and moved into the queue, and continues
  when it's its turn.
- Never run renders, Blender, ffmpeg or other heavy work outside rq: it holds up the whole queue.
- Tell the user the job id when you queue a job. `rq status` shows its position and expected times; the status page
  is http://localhost:8642.
- Don't wait for other sessions, read their transcripts or check their processes: the queue does that.
- Don't reorder the queue yourself. If the user says "go" for a job, ask the queue session to move it first (or tell
  the user to). Only the user decides whether a light job may run longer than 5 minutes.
- Pause or stop only your own processes, by exact PID, and leave no processes behind when a job is done. rq's
  processes (`rq _supervise ID` and the job under it) belong to rq even after your session restarts. Never kill
  them; use `rq cancel ID`.
- If rq says the dispatcher isn't running, tell the user; don't start it yourself, the queue session does.
