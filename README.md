# render-queue

`rq` lets several [Claude Code](https://claude.com/claude-code) sessions share one Mac for rendering, each working on
its own project. Heavy jobs run strictly one at a time, first come first served. Short light jobs run next to them at
low priority. A
local status page shows the queue with expected start and end times, the CPU cores each project uses, the GPU load and
which sessions are waiting for you.

![The status page: sessions waiting for you, CPU and GPU use per project, the job rendering now next to a light job, the queue with expected start and end times, the light queue, and recent jobs with how many outputs each declared](docs/status.png)

## Setup

You talk to two kinds of Claude Code sessions: one **queue session**, which keeps the queue running and makes the
queue decisions you hand it, and one **render session** per project, which submits its renders to the queue instead
of starting them itself.

### Requirements

- macOS. rq uses `caffeinate` (keeps the Mac awake while a job runs), `ioreg` (GPU load, no admin rights needed),
  `lsof` and `ps` (which project a process belongs to, work outside the queue) and `/bin/zsh` (runs job commands).
- Python 3 at `/usr/bin/python3` (it comes with the Xcode Command Line Tools), standard library only.
- Claude Code. The "Waiting for you" list reads the sessions' transcripts under `~/.claude/projects/`.

### Folder layout

Put your projects side by side with the `render-queue` folder, in one root folder. The folder directly under that
root is a job's project everywhere: on the status page, in job ids and in the core usage.

```
~/renders/
  render-queue/      this repository
  project-a/         one render session per project, started in this folder
  project-b/
```

If your projects live somewhere else, set `RQ_ROOT` to the folder that contains them, for the dispatcher and the
render sessions alike. A project folder in the root may be a symlink to a folder elsewhere (a project on another
disk): a session in its real folder belongs to that project too.

### 1. The queue session

A Claude Code session in the `render-queue` folder. It runs no renders itself.

1. Clone this repository into your projects' root and start a Claude Code session in it. Tell it its job, for
   example:

   > You are the queue session for rq, the render queue in this folder. Read README.md. You run no renders yourself.
   > Start the dispatcher, keep it running, and change the queue (rq move, cancel, pause, resume, keep, estimate)
   > only when I ask, or when a render session passes on my "go" for one of its jobs.

2. It starts the dispatcher: `./rq start`. The dispatcher runs detached, so it outlives the session that started it,
   and only one can run at a time. After a restart of the Mac, start it again.
3. Open the status page: http://localhost:8642 (local only; `rq start --port N` for another port).
4. Optional, once the render sessions are running (step 2): have it write `state/sessions.json`, the sessions to
   watch for "Waiting for you": every project folder name, with a link that opens its session if you have one (for
   example from the Claude desktop app), else `null`:

   ```json
   {
     "project-a": "claude://...",
     "project-b": null
   }
   ```

   The page then shows when a session asked you a question or when a tool call may wait for your permission. A
   session that only ended its turn isn't listed. While one of a session's tasks watches its job in the queue (`rq run` or `rq wait`), the
   session waits for that job, not for you. A job that no task watches won't wake the session when it ends, so then
   the session waits for you, with a note for each such job ("preview running, nobody watches it") and each result
   nobody has seen. This works best with sessions whose process rq finds: sessions in the Claude desktop app and
   CLI sessions (`claude --resume ID`, or `claude` started in the project folder); otherwise it reads the project's
   newest transcript only. A watched session without a running process (archived or closed) is left out. Without
   `sessions.json`, the page leaves "Waiting for you" out.

From then on you tell the queue session what to change, and it runs:

| You say | It runs |
| --- | --- |
| "go" for a queued job | `rq move ID first` (it runs next) |
| a running light job may run longer than 5 minutes | `rq keep ID` (before the 5 minutes are up) |
| a job will take longer or shorter | `rq estimate ID 2h` |
| stop a job | `rq cancel ID` |
| hold a running job, keeping its turn | `rq pause ID`, later `rq resume ID` |
| a running light job should wait its turn | `rq move ID queue` |
| a session lost track of its job (after a restart or an account switch; the page says "unwatched" or "unseen") | tells it to run `rq wait ID`, not to resubmit |
| a job whose session ended isn't wanted any more ("unwatched · session ended") | `rq cancel ID` |
| rq was updated and its rules changed | tells every render session that the render-queue rules changed |

### 2. The render sessions

Start one Claude Code session per project, in the project's folder (that is how rq and the transcripts know which
project a session is). The rules they follow are in `render-session-rules.md`: import that file from each project's
`CLAUDE.md`, together with the two names it uses, and replace `/path/to/render-queue` with your path:

```markdown
## Render queue

In the rules below, `rq` is `/path/to/render-queue/rq`, and "the queue session" is the Claude Code session in
`/path/to/render-queue`.

@/path/to/render-queue/render-session-rules.md
```

Not into `~/.claude/CLAUDE.md`: the queue session would get the rules too, and they tell it not to change the queue.
All render sessions then read the same rules. A session keeps the text it read in its context, so when the file
changes, tell the render sessions (or have the queue session tell them) that the render-queue rules changed: the
file asks them to read it again then. The render sessions need to be allowed to write to
`/path/to/render-queue/state`: if a sandbox blocks it, allow that folder.

Optional: so that a session learns about its jobs by itself after a restart or an account switch, add a hook to
each project's `.claude/settings.json` (next to whatever is in it already). When the session starts or gets a
prompt, it tells the session which of its jobs no task watches and which results nobody has seen, each with the
`rq wait ID` to run; it prints nothing while there is nothing to report:

```json
{
  "hooks": {
    "SessionStart": [{"hooks": [{"type": "command", "command": "/path/to/render-queue/rq mine --unwatched --hook"}]}],
    "UserPromptSubmit": [{"hooks": [{"type": "command", "command": "/path/to/render-queue/rq mine --unwatched --hook"}]}]
  }
}
```

### 3. Daily use

- Ask a render session for a render as usual ("render the final cut"). It queues the job, tells you the id and gets a
  notification when the job is done.
- Restarting a session or the app, or switching accounts, doesn't stop the jobs. Afterwards, on its next prompt, a
  session that waited for a job runs `rq mine` and `rq wait ID` (the hook above tells it to then). rq can't wake an
  idle session: tell it, or send it any prompt. The status page tags such a job "unwatched" while it runs, and its
  result "unseen" until the session that queued it has it.
- Watch the status page: what renders now, the queue with expected start and end times, cores per project, the GPU,
  work outside the queue that holds it up, and the sessions waiting for you (click one to open it).
- Click a job under "Recent" to see what it produced: the videos, pictures and frame sequences at the outputs its
  session declared when it queued it.
- Tell the queue session when you want to change the order or a job: "go for ID", "cancel ID", "let ID run on",
  "ID will take 2h". After a restart: "start the dispatcher".
- To change a rule for all render sessions, change `render-session-rules.md` and tell them the render-queue rules
  changed (or ask the queue session to, if your sessions can message each other).

## How jobs run

- `rq run` hands the job to its own rq supervisor, a process it starts detached from the session (its parent is
  launchd), with the session's folder, environment and sandbox. The supervisor waits for the job's turn, runs the
  command and records the result. `rq run` itself only watches: it shows the job's place in the queue, streams its
  output live and exits with the job's exit code. A single command argument runs through `/bin/zsh -c`, several
  run as an argument list. The job's stdin is `/dev/null` and its output goes to its log (`rq log ID`).
- Every `rq run` and `rq light` says where the job writes its pictures and videos, with `--output PATH` (`-o`,
  once per file, folder or frame sequence) or `--no-output` if it writes none. rq doesn't guess: the status page
  shows only what is at the declared paths. rq's own options go before `--` and the command. PATH is one of
  - a file: `renders/final.mp4`
  - a folder: `renders/stills/` (a trailing `/`, or a folder that exists already)
  - a frame pattern, its frame number in the file name as a run of `#` (as Blender writes it) or `%04d` / `%d`
    (ffmpeg, printf), with the zero padding of the files the job writes: `'renders/frames/####.png'`,
    `'out/frame_%04d.exr'`, `'shots/f%d.jpg'`. `####` and `%04d` match 0000 to 9999 and longer numbers unpadded
    (12345), `#` and `%d` unpadded numbers (1, 2, 10); frame numbers can have up to 18 digits. A `#` in a file
    name is always a frame number.

  A relative PATH is from the folder `rq run` runs in. rq refuses (exit 64, nothing queued, a message that says
  what to pass)
  - a call with neither or both of `--output` and `--no-output` (and says so when an `--output` came after the
    command, where it belongs to the command);
  - a PATH outside the project's folder and its Claude Code scratchpads, also through a symlink (a folder in the
    project that links to another disk counts as outside, and the message says so), and the project folder or
    its scratchpads' folder as a whole;
  - a file or pattern that isn't a picture or video the page shows (png, jpg, jpeg, webp, gif, avif, bmp, tif,
    tiff, heic, exr, mp4, mov, m4v, webm, mkv, avi): for frames or files in another format (DPX, MXF, a PDF),
    declare their folder, so the sheet offers Show in Finder;
  - a frame number rq can't read: `%4d` or `%00d` (write `%04d` or `%d`), digits right after a `#` (`take#1.mp4`:
    declare its folder), a frame number in a folder's name, several in one name, and Blender's `//` (relative to
    the .blend file; give the path from the folder you submit from);
  - a `--project` that isn't one folder name in the projects folder (`/`, `..`, `a/b`).

  For example:

  ```
  rq run --name final --estimate 2h -o 'renders/final/####.png' -o renders/final.mp4 -- ./render.sh
  rq run --name shot --estimate 1h -o 'renders/shot/####.png' -- blender -b shot.blend -o '//renders/shot/####' -a
  rq run --name grade --estimate 20m --output renders/graded/ -- python3 grade.py
  rq run --name dpx --estimate 30m --output renders/dpx/ -- ./export-dpx.sh
  rq light --name tests --no-output -- python3 -m pytest
  ```

  rq doesn't check the outputs when a job ends; the status page looks at them when you open the job.
- What `rq run` and `rq wait` exit with:
  - once the job has ended: 0 done; the job's code if it failed, or 128+n if signal n ended it (143 for SIGTERM,
    also when `rq cancel` stopped it while it ran; 137 for SIGKILL); 130 if it was cancelled before it started;
    75 if it was lost;
  - when they stop watching first (the job carries on): 128+n for the signal they got (143 for a TaskStop), 129 if
    the session they reported to ended, 141 if nobody reads their output any more. So 143 alone doesn't tell a
    stopped watcher from a cancelled job: the last line says which;
  - when nothing was queued: 64 for no `--output` and no `--no-output` (or both), an `--output` rq can't show, or
    a `--project` that isn't one folder name; 65 for an identical job that is still queued or running and
    unwatched, 69 while an older dispatcher runs; 70 if the job's supervisor could not start. `--detach` exits 0
    once the job is submitted.
- `--estimate` (e.g. `45m`, `2h30m`) feeds the status page's times; `rq estimate ID 90m` sets it later. Without it,
  the median of earlier successful runs with the same project and name is used. A job with neither shows unknown
  times, and so does every job after it.
- `--project` defaults to the project folder you are in (also the real folder of a project folder that is a
  symlink in the projects folder). It is one folder name in the projects folder.
- Stopping the `rq run` process (or the background task running it) only stops watching: the job carries on,
  queued or running, and so it does when its session or the app ends or you switch accounts. `rq cancel ID`
  cancels it; `rq wait ID` watches it again, and for a job that has ended, it reports the result at once.
  `rq wait ID1 ID2 ...` watches several jobs at once (all count as watched from the start): it follows them one at
  a time, shows one that ends meanwhile right away, and exits with the first non-zero code in the order given. A
  stopped task's output file gets a note that the job carries on, and later its result. The status page and
  `rq status` mark a job that no task of its session watches "unwatched", and a result that hasn't reached that
  session "unseen".
- `--detach` submits without watching. An identical job (same project, name, folder and command) that is still
  queued or running and that no task of its session watches isn't submitted again: `--again` queues another copy
  anyway, `--replace` first cancels this project's jobs with that name. If only its outputs differ (a session
  correcting its `--output`s), the job, queued or running, takes the new ones when the same session resubmits it
  (another session's job keeps its own, and the message says they differ). A job with the same name from another
  folder (a new snapshot) or with another command isn't refused, but `rq run` names it, since it was not cancelled.
  A finished job is never refused: check `rq mine` first.
- `rq mine` lists the current session's jobs (by `CLAUDE_CODE_SESSION_ID`, else by project) that no task of that
  session watches, and the results of the last 24 hours that didn't reach it. A terminal or another session
  following a job doesn't count for the session that queued it. With `--hook` it reads a Claude Code hook's JSON
  on stdin; it prints nothing when there is nothing to report and always exits 0.
- **Light work** (`rq light`) has its own queue: one light job at a time, first come first served, next to the heavy
  job (the two queues don't wait for each other). It runs at low priority (`nice -n 10`), streams its output the same
  way and shows on the status page under "Rendering now". It never holds up the main queue.
- Light work is capped at 5 minutes while other work is active (a heavy job running or queued, or a light job
  waiting); alone, a light job runs on. Once it is past 5 minutes and other work shows up, it is paused where it is
  and moved to the end of the queue as a heavy job; it continues, with its progress, when it's its turn (the status
  page marks it "paused · was light"). `rq keep ID` before the 5 minutes are up lets it run on; once it is paused,
  `rq move ID first` makes it the next job. The 5 minutes are running time: while `rq pause` holds a light job, it
  isn't moved, and a job that `rq pause` holds when its turn comes keeps its turn and stays paused until
  `rq resume ID`.
- **Work outside the queue:** a busy or paused Blender or ffmpeg anywhere on the machine, or any other process group
  in a video project's folder (or its Claude Code scratchpad) that uses more than about one core at normal priority,
  holds up the queue until it has ended, and the status page names it. A video project is one in
  `state/sessions.json` or one that has queued a job with rq; other projects' work (a benchmark in a code project,
  say) doesn't hold the queue. Heavy low-priority work outside rq is shown but doesn't
  hold the queue. Once outside work that held the queue ends, the next job starts after 2 minutes without it
  (a gap between two steps of a pipeline doesn't let a job in). Right after the dispatcher starts, a job starts at
  once.

## Commands

```
rq run --name NAME (--output PATH... | --no-output) [--project P] [--estimate 90m] [--detach]
       [--again | --replace] -- COMMAND...
rq light --name NAME (--output PATH... | --no-output) [--project P] [--detach] [--again | --replace] -- COMMAND...
                             (light work: the light queue, one at a time, at nice 10)
                             (--output PATH, -o PATH: a file, a folder/ or a frame pattern like 'renders/####.png'
                             the job writes, repeated for several; --no-output: it writes no pictures or videos;
                             every rq run and rq light needs one of them, before -- COMMAND)
rq wait ID... [-n 20]        (watch jobs again: their output from here and their exit codes; at once if ended;
                             several IDs count as watched at once, the first non-zero code is the exit code)
rq mine [--unwatched] [--hook]   (this session's jobs no task of it watches, results that didn't reach it)
rq status [--json]           (also: jobs nobody watches, results nobody has seen, with their ids)
rq log ID [-n 40] [-f]
rq cancel ID
rq pause ID / rq resume ID   (stop or continue the job's processes; it keeps its turn)
rq move ID first             (when the user says go: this queued job runs next)
rq move ID queue             (pause a running light job, move it to the end of the main queue)
rq keep ID                   (when the user says so: this light job may run longer than 5 minutes)
rq estimate ID 90m           (set or correct a job's expected duration)
rq start [--port 8642]       (start the dispatcher and the status page detached, one per machine)
rq daemon [--port 8642]      (the same in the foreground)
```

IDs can be shortened to any part that matches only one job, finished jobs included (`rq status` lists the ids).

## Status page

http://localhost:8642 shows which sessions are waiting for you, the cores each project uses (in total and per
project), the GPU utilization, the running job and light work, the queue with expected start and end times, heavy
work outside rq, and recent jobs with how long they ran (without the time they were paused or waiting in the
queue; "-" when the job's owner was gone and nobody saw it end) and how many outputs they declared ("2 outputs",
"none" for `--no-output`, nothing for jobs queued before rq had `--output`). It listens on 127.0.0.1 only and shows no job
output and no command lines, only the program a job runs: rq works that out itself, so the page never gets a job's
command line or folder.

Tags say who sees a job (hover over one for more):

- **unwatched**: no task of the session that queued the job watches it, so that session gets no output or exit
  code when it ends (a terminal or another session following it doesn't count). It runs on all the same (stopping a task doesn't cancel a job; `rq cancel ID` does), and `rq wait ID` watches it
  again. **unwatched · session ended**: the session that queued it doesn't run any more (archived or closed); unless
  you still want the result, have the queue session cancel it.
- **unseen**: a job that ended in the last 24 hours whose result hasn't reached the session that queued it (no
  task of that session showed it). That session gets it with `rq mine` or `rq wait ID`.
- **supervisor gone**: the job runs on without its rq supervisor (killed or crashed); it keeps its turn until its
  processes end and is then recorded as lost.
- A lost, cancelled or failed job shows why under its name (the full reason is in its sheet), for example "the Mac
  restarted", "rq cancel" or "its rq supervisor ended while the job was queued".

The tab's icon, an old movie camera, is in color while a job runs (heavy or light) and gray while the queue is idle.
It gets an amber dot while a session is listed under "Waiting for you": it asked a question, a tool call
may wait for your permission, or its turn ended while its jobs are unwatched or its results unseen.

"Notify me" under "All done" turns on browser notifications, while the page is open in a tab: when a job that isn't
light ends (not when it was cancelled; click to open its sheet) and when a session shows up under "Waiting for you"
(click to open the session, if `sessions.json` has a link for it).

**Outputs** (http://localhost:8642/feed, linked next to the title) shows every picture, video and frame sequence the
jobs of the last 24 hours wrote, the newest first, one under the other at full width, to catch a bug early by looking.
A file shows once, under the job that ran when it was written (a declared folder also holds what earlier work wrote
there); a running job's files show as they are written. Under each: the session, the job, when, and a Copy button
that copies `<job name> - <path> (job <id>)` (for a sequence, the frame on screen) to tell that session about it.

Click a job under "Recent" (or open `http://localhost:8642/#job=ID`) to see what is at the outputs it declared:

- First the outputs themselves, one line each: a file (or "not there"), a folder with what it holds, a frame
  pattern with its frames. A folder can be shown in the Finder.
- **Videos** play in the page, **pictures** show in a grid with a full-size view (arrow keys step through them), and
  **frame sequences** get a scrubber over their frames: a declared frame pattern is one sequence, and in a declared
  folder 24 or more numbered pictures in one folder are one. In a folder the sheet looks 4 folders deep, at up to
  20000 files and folders, and shows up to 500 single pictures and videos (it says when there is more); declared
  files and frame patterns always show, ahead of what declared folders hold.
- What the browser can't show (EXR always; ProRes outside Safari, TIFF outside Safari) isn't converted: the page
  offers **Open on Mac** (opens the file in its usual app) and **Show in Finder** instead. A whole sequence opens as
  its folder; a folder that is a package (an `.app` or another bundle) is only shown in the Finder, never opened.
- The sheet shows what is at the declared paths when you open it: files written there after the job ended show up
  too, and files removed since don't. A job that never ran (cancelled before it started, or still queued) says
  that what shows was written by other work. A job queued with `--no-output` says it writes no pictures or videos;
  a job queued before rq had `--output` says no outputs were declared. A declared folder with nothing the page can
  show (DPX frames, say) still offers Show in Finder.

## How it works

- `state/` (not in git) holds one JSON file per job (with its declared outputs: their kind, absolute path and the
  path as given), the logs, the current turns, the dispatcher's heartbeat, an owner lock per job
  (`state/owners/`) and a file per watcher (`state/watch/`).
- The dispatcher checks every 2 s. When nothing is granted, it gives the turn to the first job in the queue (the
  oldest, unless one was moved first), once no heavy work at normal priority has been busy or paused outside the
  queue for 2 minutes (or since the dispatcher started).
- `rq run` forks its supervisor twice with a new session in between, so it runs with launchd as its parent, out of
  reach of a session's task kill (Claude Code stops a task by signalling its process group and every descendant).
  The supervisor starts the command in a new process group (with `caffeinate -i`), its output into the log, and
  records the result. Before the job's record exists, `rq run` takes an flock on `state/owners/ID.lock` and hands
  that open file to the supervisor, which holds it until it has recorded the end: the lock, not a pid, says that
  the job has an owner.
- If a supervisor dies (`kill -9`), its job keeps the turn until its processes have ended; then the dispatcher
  records it as lost, or as cancelled if it was being cancelled, with the reason. Queued jobs whose supervisor is
  gone are cleaned up on every check, also while another job has the turn. A job from before the Mac restarted is
  recorded as lost ("the Mac restarted") without signalling anything, since its process ids belong to other
  processes by then. SIGTERM to a supervisor (a logout or shutdown) stops its job, recorded as failed
  ("interrupted"). Each change to a job's record re-reads it under `state/jobs.lock`, so a stale copy never
  overwrites a result.
- Jobs submitted by an older rq, still running at an update, run as before (their `rq run` runs the job itself)
  until they end. A new `rq run` refuses to submit while an older dispatcher runs: restart it first.
- rq never signals a process group that isn't a job's any more: before it pauses, continues, cancels or kills the
  processes of a job it doesn't own, it checks that the group's leader is still the process the job started (its
  pid and start time). A group whose id another process got since counts as ended.
- Every 5 s the dispatcher reads the tail of each watched session's transcript (and checks its Claude process) to show
  which sessions wait for you: they asked a question, a tool call may wait for permission, or their turn ended while
  they have jobs that no task watches or results they haven't seen (a turn that only ended isn't shown). Turns
  that only answered another session are skipped. A session one of whose tasks watches a job in rq waits for that
  job, not for you.
- Who watches a job: every `rq run` or `rq wait` that follows one holds an flock on its file in `state/watch/`,
  which also names the Claude Code session it reports to (`CLAUDE_CODE_SESSION_ID`, and `CLAUDE_PID` with that
  process's start time). A watcher counts while it holds the lock and that Claude process runs. Whether the session
  that queued a job has ended comes from its session id: the dispatcher looks for a Claude process that runs it
  (`--resume ID` on its command line). A resumed session keeps its id but gets a new process, so a process id alone
  never marks a session ended. When rq can't tell (a CLI session started without an id runs in that project, or the
  dispatcher isn't running), it shows no "session ended". A result is "unseen" until a task of the session that
  queued the job has shown it (`rq run` or `rq wait`, while that session runs); a job queued from a terminal counts
  as seen once any watcher showed it. A terminal or another session following the job doesn't count for the
  session that queued it.
- When a job is submitted, rq records which of Blender, Python and ffmpeg it runs: from the command and the scripts it
  starts (two levels deep). The status page shows them as logos in the Tool(s) column.
- Cores per project come from `ps` and each process's working directory (or a parent's) under the projects' root, or
  its Claude Code scratchpad.
- The page fetches a job's files by the job and a key, never by path: a declared output's key comes from its path,
  a file's in a declared folder from that folder and the file's place in it, so a link stays the same for the same
  path. On every request rq checks again, on disk with symlinks resolved, that the file is one the job declared (or
  one in a declared folder, and still inside it) and a regular picture or video file inside the project's folders.
  A file the sheet found in a declared folder is served only while the job still declares that folder. Opening one
  on the Mac needs a POST from the page itself. `/api/status` carries only how many outputs a job declared, never
  their paths, and neither it nor `/api/job` carries a job's command line or folder (they usually name those
  paths): only the program it runs.

## Configuration

| Variable | Default | Read by | What it is |
| --- | --- | --- | --- |
| `RQ_ROOT` | the folder that contains `render-queue` | the dispatcher, `rq run`, `rq light` | where the projects are |
| `RQ_STATE` | `render-queue/state` | every rq command | jobs, logs, the dispatcher's state, `sessions.json` |
| `RQ_LIGHT_CAP` | `300` | `rq light` | seconds a light job may run before it moves into the queue |
| `RQ_CANCEL_KILL` | `30` | `rq run`, `rq light` | seconds from `rq cancel` to SIGKILL of a job that still runs |
| `RQ_STOP_KILL` | `10` | `rq run`, `rq light` | the same after the job's supervisor got SIGTERM |
| `RQ_QUIET_WINDOW` | `120` | the dispatcher | seconds without outside work before the next job starts |
| `RQ_SCRATCH_GLOB` | `/private/tmp/claude-UID/` | the dispatcher, `rq run`, `rq light` | where Claude Code keeps this user's scratchpads (where `--output` may point besides the project folder) |

Set `RQ_ROOT`, `RQ_STATE` and (if you change it) `RQ_SCRATCH_GLOB` the same for the queue session, which starts the
dispatcher (it inherits the session's environment) and runs the queue commands, and for the render sessions: in the
`env` of `.claude/settings.json` in the `render-queue` folder and in each project folder. `rq run` checks an
`--output` against `RQ_ROOT` and `RQ_SCRATCH_GLOB` when it submits, and the dispatcher checks them again when it
serves a file, so both need the same values. The dispatcher reads its variables when it starts: to change them,
stop it (`kill` the pid that `rq start` prints, also in `state/dispatcher.json`) and run `rq start` again. Jobs keep
running meanwhile.

## Updating rq

Copy `rq`, `status.html`, `feed.html` and `render-session-rules.md` together (the rules describe what that `rq` does), then have
the queue session restart the dispatcher (`kill` the pid in `state/dispatcher.json`, then `rq start`): jobs keep
running meanwhile. `rq run` refuses to queue while an older dispatcher runs. Jobs whose `rq run` is older run as
before until they end. Then tell the render sessions that the render-queue rules changed. Since rq has
`--output`, every `rq run` and `rq light` needs `--output` or `--no-output`: a session that still uses the old form
gets exit 64 and a message that says what to pass. The dispatcher removes `state/artifacts/`, where an older rq kept
what it guessed jobs produced; those jobs' sheets say no outputs were declared.

## License

MIT
