# Video captions — technical documentation

Oct 22, 2026 · @Ramoju Kalyan Chary

The video library app shows synchronised captions over each SharePoint recording by reading a matching `.vtt` file through a cloud flow and tracking the player's position with a timer. No transcript is stored in the app, and no caption file is attached to the video control.

## How it works

&#91;embedded content: caption pipeline · two routes into SharePoint\]

The caption text is fetched server-side by the flow, so the browser never requests the `.vtt` file and no cross-origin rule applies. The video itself is loaded by the player from its direct SharePoint URL, using whichever account the viewer is signed in with.

## SharePoint setup

Every recording needs one `.vtt` file whose name matches the video exactly. That name match is the only link between the two libraries, so the app needs no lookup column and no per-video configuration.

| What | Where |
| --- | --- |
| Recordings | `/Shared Documents/Recorded Tutorials/Bytesize Recordings/Published/Bytesize Recordings/` |
| Transcripts | `/Shared Documents/Private Documents/Events/Demo/All. vtt Files/` |
| Site | `https://<tenant>.sharepoint.com/teams/powerhub` |

All transcripts sit in one flat folder, not in per-session subfolders. Session names are unique, so there are no clashes, and the flow's path never changes.

**Naming rule.** `Approved Connectors DLP Policies.mp4` needs `Approved Connectors DLP Policies.vtt`. Watch for double spaces in video names: the app collapses them before calling the flow, but an extra or missing character anywhere else means the flow returns file-not-found.

Transcripts are generated in Clipchamp as `.srt`, converted to `.vtt`, then uploaded. Microsoft Stream can also generate a transcript from the video and export it as `.vtt`, which gives the same result with fewer steps.

## The GetVideosContent flow

An instant cloud flow with three actions. It takes a file name, reads that file from SharePoint and returns its text.

| Step | Action | Configuration |
| --- | --- | --- |
| 1 | Trigger: Power Apps (V2) | One text input named `FileName` |
| 2 | SharePoint: Get file content using path | Site: the PowerHub site. Path: the transcripts folder, then the `FileName` token |
| 3 | Respond to a Power App or flow | One text output named `VttText` |

The file path is the folder text followed by the dynamic token, with literal spaces and no URL encoding:

```text
/Shared Documents/Private Documents/Events/Demo/All. vtt Files/[FileName]
```

The output value is an expression, not dynamic content:

```text
base64ToString(outputs('Get_file_content_using_path')?['body/$content'])
```

The action name inside that expression must match the SharePoint action's own name with spaces replaced by underscores. If the action is ever renamed or duplicated, the expression has to be updated to match.

**Run-only users.** In the flow's details, the SharePoint connection is set to **Use this connection (owner)**. Without it, each app user runs the flow with their own connection and anyone without access to the transcripts folder sees no captions.

The flow runs under a service account. That account needs read access to the transcripts folder only — it never loads the video.

## App configuration

Everything sits on the Selected Video screen. Three controls were added: a timer, the caption label, and the container holding the expand icon.

**Screen OnVisible** calls the flow and parses the result. `IfError` keeps the screen working when a video has no transcript.

```text
Set(varVtt, "");
Set(
    varVtt,
    IfError(
        GetVideosContent.Run(
            Substitute(locSelectedVideo.Name, "  ", " ") & ".vtt"
        ).vtttext,
        ""
    )
);
ClearCollect(
    colCues,
    ForAll(
        MatchAll(
            Substitute(varVtt, Char(13), ""),
            "(?<h>\d+):(?<m>\d{2}):(?<s>\d{2}\.\d{3})\s*-->\s*(?<h2>\d+):(?<m2>\d{2}):(?<s2>\d{2}\.\d{3})[^\n]*\n(?<txt>[^\n]+)"
        ) As c,
        {
            StartSec: Value(c.h)*3600 + Value(c.m)*60 + Value(c.s),
            EndSec:   Value(c.h2)*3600 + Value(c.m2)*60 + Value(c.s2),
            CueText:  c.txt
        }
    )
);
```

`locSelectedVideo.Name` carries no file extension, so `.vtt` is appended rather than swapped in. The regex expects `HH:MM:SS.mmm` timestamps and one line of text per cue, which is what the Clipchamp conversion produces.

**Timer** — Duration `300`, Repeat `true`, AutoStart `true`, Visible `false`.

```text
OnTimerEnd:  Set(varNow, Video1.Time)
```

The timer exists because `Video1.Time` does not reliably re-trigger formulas on its own. Reading it into `varNow` on a short interval is what keeps the caption in step.

**Caption label** — the text is whichever cue surrounds the current position.

```text
Text:   LookUp(colCues, StartSec <= varNow && EndSec >= varNow).CueText
Size:   If(varBig, 12, 11)
Height: If(varBig, 80, 40)
```

The label sits at screen level, not inside the video's container, so it can float above the player in both normal and expanded mode. Its X and Y are expressed against `Video1` and switch on `varBig`.

## Expanded mode

The player's own full-screen button hands the video to the browser, which takes it out of the app's page. Nothing drawn by Power Apps can sit on top of it, so captions disappear the moment it is used.

The app therefore provides its own expand control: a transparent icon inside `Container3`, positioned exactly over the native button so a click lands on ours instead.

```text
Container3.X = Video1.X + Video1.Width - If(varBig, 110, 80)
Container3.Y = Video1.Y + Video1.Height - If(varBig, 44, 40)

Icon1.OnSelect  = Set(varBig, !varBig)
Icon1.Color     = RGBA(0, 0, 0, 0)
Icon1.HoverFill = RGBA(0, 0, 0, 0)
```

The icon is fully transparent in every state, so the viewer sees only the native icon underneath and clicks it as usual. The two offsets differ between modes because the player does not keep its button at a fixed distance from the edge when it resizes.

`varBig` then drives the layout: `Video1`, `Container8_12` and every container above it switch to full width and height, while the header, navigation, title, likes and related videos take `Visible = !varBig`. The caption label follows because its own position is written against `Video1`.

The native button still exists and is not removable. A viewer who reaches it another way gets browser full screen with no captions.

## Deployment

Three things must travel together in one solution. The canvas app stores the flow's internal id, so a flow added outside the solution, or rebuilt by hand in the target environment, leaves the app calling something that is not there.

| Component | Why it must be in the solution |
| --- | --- |
| Canvas app | Holds the reference to the flow |
| `GetVideosContent` flow | Carries the trigger, the path and the output |
| SharePoint connection reference | Supplies credentials per environment |

A plain connection created outside the solution does not travel. The flow must point at a connection reference instead, chosen from the solution's own list rather than the "other connection references" group.

**After every import, the person importing must:**

1. Turn the flow on — imported flows arrive disabled
2. Map the connection reference to a SharePoint connection in that environment
3. Confirm the connection's account can read the transcripts folder

The target environment needs its own SharePoint connection; a connection reference is only a pointer. Use the service account, not a personal one, so the flow survives any one person's access changing.

Renaming the flow is safe as long as it is renamed, never deleted and recreated — the id is what the app holds. After a rename, the `OnVisible` formula has to be updated and the flow re-added in the Power Automate pane so Power Apps picks up its outputs.

## Adding a new session recording

No app change and no deployment is needed. The flow derives the transcript name from the video at run time, so the library grows on its own.

1. Publish the recording to the Bytesize Recordings library as usual
2. Generate the transcript — Clipchamp, or Microsoft Stream's own transcript export
3. Save it as `.vtt`, named exactly like the video file
4. Upload it to the `All. vtt Files` folder
5. Open the video in the app and confirm the caption tracks the speech

If step 5 shows no caption, the name does not match. Compare the two file names character by character, including spaces.

## Known limitations and troubleshooting

**Why `ClosedCaptionsUrl` is not used.** The video control's own caption property needs a `.vtt` file the browser can fetch cross-origin, and SharePoint sends no CORS header for it. The caption button appears, the text stays blank, and no error is raised. Reading the file through the flow avoids this entirely.

**Other limits.**

- Browser full screen, reached through the native button, loses the captions
- The transcript panel with click-to-seek was removed; only the on-video caption remains
- The parser reads one line of text per cue, so a multi-line cue loses its continuation
- A video with no matching `.vtt` plays normally with no caption, which is intended

**When something breaks.**

| Symptom | Likely cause | Where to look |
| --- | --- | --- |
| No caption, video plays | Transcript missing or misnamed | Flow run history, the failed action's inputs |
| Flow succeeds, no caption | Output empty or renamed | The Respond action's output name and expression |
| Flow never fires | App not bound to the flow | Power Automate pane in the app, App checker |
| Nothing works after import | Flow disabled or connection unmapped | Solution in the target environment |
| Video shows `00:00:00/00:00:00` | The player cannot load the file | The signed-in account's access to the recordings library |

**Diagnostics.** Two temporary labels answer most questions quickly:

```text
"Chars: " & Len(varVtt) & " | Cues: " & CountRows(colCues)
varNow & " | " & Video1.Time
```

Non-zero characters with zero cues means the flow works and the parse failed. Zero on both means the flow returned nothing. A video stuck at `0 | 0` is not playing at all, which is a separate problem from the captions.

**One account cannot do both jobs.** The service account reads transcripts; the viewer's own account loads the video. A service account signed in to the browser will usually show no video, because the player makes a silent request that cannot complete an interactive sign-in. This is expected and needs no fix — test with a normal end user, not the service account.
