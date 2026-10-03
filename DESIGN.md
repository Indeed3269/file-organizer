# File Organizer: Design

> Answer each question in your own words, and keep it to one page. Sketchy answers are fine; "I don't know yet" is a valid answer you'll revisit.

## 1. What it does

It runs in a terminal session: you call the command on a folder, it shows a preview of how the files will be organized, and once you confirm it moves them into their folders. A file's destination is chosen by its extension, using the config (with default destinations, e.g. ~/Documents for pdf, docx, xlsx, csv; ~/Pictures for png, jpeg, ico, svg). It runs once rather than watching the folder.

## 2. Commands

v1

```
Usage
  autosort [options]          sort the origin folder (preview, then confirm)
  autosort undo [options]     reverse the last job (preview, then confirm)

Options
  -o, --origin <path>         folder to sort, for this run only
  -r, --recursive             also sort files inside subfolders
  -a, --all                   include hidden files
  -y, --yes                   skip the confirm prompt (the preview is still printed)
  -h, --help                  show this help

Examples
  autosort                    sort ~/Downloads using the saved config
  autosort -o ~/Desktop -r    sort Desktop and its subfolders, this run only
  autosort undo -y            undo the last job without asking
```

- Flags only affect the current run. Saved settings are changed by editing the config file by hand (see §3).

## 3. Rules config

- Built-in defaults live in the code. A user file at `~/.config/autosort/autosort.config.json` is optional; it overrides only what it declares (origin, or a rule with the same name). No file = defaults only.
- Each rule has a `location` and a list of `members`. A file matches a rule by extension, ignoring case.
- The longest matching extension wins (`*.tar.gz` beats `*.gz`). The same extension in two rules is a config error, reported when the config loads.
- Files that match no rule are skipped (undeclared extensions, files without an extension).
- File names are moved as they are (`photo.Png` stays `photo.Png`). Names that differ only by case count as a clash (`photo.png` vs `photo.PNG`) and get auto-renamed.
- A leading `~` in any path is replaced with the home folder (`os.homedir()`).

```json
{
  "origin": "~/Downloads/",
  "rules": {
    "images": {
      "location": "~/Pictures/",
      "members": ["*.svg", "*.png", "*.ico", "*.jpeg", "*.jpg"]
    },
    "documents": {
      "location": "~/Documents/",
      "members": ["*.pdf"]
    },
    "archives": {
      "location": "~/Documents/CompressedFiles/",
      "members": ["*.gz", "*.tar.gz"]
    }
  }
}
```

<!-- Where does the config file live? Write a real example of its contents.
     How does a rule say "this extension goes to this folder"?
     What happens to a file that matches no rule? -->

## 4. Edge cases

<!-- Decide what happens in each case:
     - The destination already has a file with the same name
     - The destination folder doesn't exist
     - A folder inside Downloads (move it whole? look inside? skip it?)
     - A file that's still downloading (.part, .crdownload)
     - The destination is on another drive
     - Permission denied -->

- Name clash (in the destination, or between two files in the same job): auto-rename with a ` (n)` suffix, shown in the preview. The file with the shorter origin path keeps the name; ties go alphabetically.
- Missing destination folder: listed in the preview as "will create ~/Pictures/", created on confirm.
- Subfolders: skipped unless `-r`. With `-r`, any destination folder inside the origin is skipped silently along with its children.
- Unfinished downloads: `.part` / `.crdownload` files match no rule, so they're skipped. A file with a `.part` twin (`report.pdf` next to `report.pdf.part`) is skipped silently.
- Hidden files: skipped unless `-a`. With `-a` they still need a matching rule.
- Permission denied: stops the job. Files already moved stay moved and are in the log, so `autosort undo` can reverse them.
- Another drive: v1 treats the `EXDEV` error like permission denied (stop the job). v1.1 falls back to copy + delete the source.

## 5. Undo

- Logs live in `$XDG_STATE_HOME/autosort/` (default `~/.local/state/autosort/`), one file per job named `yyyy-MM-dd_HHmmss.json`. The last 5 jobs are kept (configurable); older ones are deleted automatically.
- The log is written before the moves. After a crash, undo finds the unmoved files still at `was`, warns, and moves the rest back.
- Only the last job can be undone. Undo swaps `was` and `is` and writes its own log, so an undo can itself be undone.
- Undo conflicts: files missing from `is` are listed in a warning, with a confirm to move the ones that still exist. If none exist, it stops with "none of the files from the last job exist anymore". A file that now sits at `was` triggers the same auto-rename as a normal sort.
- Undo doesn't remove folders created by the job.

```json
[
  { "was": "/home/me/Downloads/report.pdf", "is": "/home/me/Documents/report.pdf" },
  { "was": "/home/me/Downloads/photo.png", "is": "/home/me/Pictures/photo (1).png" }
]
```

## 6. Folder structure

<!-- Sketch the project's files and folders. What is each file responsible for?
     Hint: keep "deciding what should move" separate from "actually moving files".
     Ask yourself why that split makes testing easier. -->

Grouped by what each file touches. Only `plan.js` decides what moves, and it never touches the disk.

```
package.json          "bin": { "autosort": "src/cli.js" }
src/
  cli.js      talks to the user
                parse args (util.parseArgs), print preview, ask y/N, call the others in order
  config.js   reads the disk
                loadConfig(userConfigPath) -> defaults merged with the user file
                expandHome(path) -> path with ~ replaced
  scan.js     reads the disk
                listFiles(origin, { recursive, all, skipFolders }) -> [originPath]
                listNames(folders) -> names already in each destination folder
  plan.js     pure, no disk
                matchRule(fileName, rules) -> rule or null
                pickFreeName(name, takenNames) -> "name (n).ext"
                buildPlan(files, rules, takenNames) -> { moves: [{ was, is }], newFolders: [] }
  move.js     writes the disk
                runPlan(plan, logPath): create folders, write log, rename each file
  log.js      reads/writes state
                writeLog(moves), readLastLog(), pruneLogs(keep)
test/
  plan.test.js   plain arrays in, plain arrays out, no disk
  move.test.js   temp folders
```

## 7. Testing

<!-- What will you test, and how do you avoid touching your real Downloads folder? -->

Paths are always passed in as parameters (origin, rules, log folder), never read from inside a function. Tests pass a temp folder from `fs.mkdtemp(os.tmpdir())`, so the real ~/Downloads is never touched.

plan.test.js (no disk)
- a file that matches no rule is skipped
- `*.tar.gz` beats `*.gz`
- `photo.png` and `photo.PNG` clash; the second becomes `photo (1).PNG`
- `report.pdf` next to `report.pdf.part` is skipped
- hidden files are skipped without `-a`
- o/hi.txt and o/l1/hi.txt -> d/hi.txt and d/hi (1).txt (shorter path keeps the name)

move.test.js (temp folders)
- moves o/photo.PNG to d/photo.PNG
- moves o/hi.txt ("this is parent") and o/l1/hi.txt ("this is child"); d/hi.txt contains "this is parent", d/hi (1).txt contains "this is child"
- creates a missing destination folder
- moves o/fleeting.ico, deletes d/fleeting.ico, runs undo: expects the "none exist" error
- undo after a normal job puts every file back at `was`

## 8. Out of scope (for now)

<!-- What are you deliberately NOT building in v1? -->

- v1.1: moving to another drive (copy + delete on `EXDEV`).
- v2: project flow (`autosort -p`): a project config (`autosort.config.json` in the project folder) that wins over the user config, and an interactive menu (space to select, r to select and rename) to pull files into the project.
- v2: an `autosort config` command to change settings from the terminal.
- v3: undo removes folders the job created.

## Open questions

<!-- Anything you're unsure about; bring these to the review. -->

I have a somewhat concrete idea of the general setup of the project, I might develop a better structure as I go.
