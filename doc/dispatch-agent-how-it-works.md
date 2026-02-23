# Dispatch Agent — How It Works

A layman's explanation of what the Dispatch Agent does and how the n8n workflow operates.

---

## The Analogy: A Project Manager With a Notepad and a Phone

The Dispatch Agent is like a **project manager** who:
- Receives a **to-do list** (from the Planning Agent)
- Can only do one thing: **call other people** to do the actual work (Action Agent, Reasoning Agent)
- Keeps a **notepad** (saved to Wasabi S3) so they don't forget where they left off
- Goes to sleep between calls, and **wakes up only when someone calls back**

---

## The To-Do List

The Planning Agent hands over something like this:

```
To-Do List:
  1. Check source URL is valid          (no prerequisites)
  2. Download the content               (needs #1 done first)
  3. Check file is OK                   (needs #2 done first)
  4. Extract audio                      (needs #3 done first)
  5. Extract frames                     (needs #3 done first)  ← can run alongside #4
  6. Detect language from audio         (needs #4 done first)
  7. Transcribe                         (needs #6 done first)
  8. Analyse facial expressions         (needs #5 done first)
  9. Combine everything into a report   (needs #7 + #8 done)
```

The key: some tasks must wait for others. Some can run in parallel.

---

## How It Runs

### Round 1 — First call comes in

> 📞 Planning Agent calls: *"Here's the to-do list."*
>
> Dispatch Agent looks at the list. Only task #1 has no prerequisites, so:
> - Writes down the current status on a **notepad** (saves state to Wasabi)
> - Calls the Action Agent: *"Please do task #1"*
> - Hangs up. Goes to sleep.

### Round 2 — Action Agent calls back

> 📞 Action Agent calls: *"Task #1 done. URL is valid."*
>
> Dispatch Agent picks up the notepad from Wasabi, crosses off #1, checks what's unblocked:
> - Task #2 needs #1 ✅ → it's ready!
> - Updates the notepad, saves it back to Wasabi
> - Calls the Action Agent: *"Please do task #2"*
> - Hangs up. Goes to sleep.

### Round 3 — Another callback

> 📞 Action Agent calls: *"Task #2 done. File stored."*
>
> Crosses off #2, checks the list:
> - Task #3 needs #2 ✅ → ready!
> - Saves the file location to the notepad (so future tasks know where to find it)
> - Sends task #3 to the Action Agent

### Round 4 — The interesting round

> 📞 *"Task #3 done."*
>
> Checks the list:
> - Task #4 needs #3 ✅ → ready!
> - Task #5 needs #3 ✅ → ready!
> - **Both are ready at the same time!**
> - Calls Action Agent twice, in parallel: *"Do #4"* and *"Do #5"*

### ...and so on until everything is done.

### Final Round — All done

> Checks notepad: every task is either ✅ done or ❌ failed.
>
> Writes a final summary and sends it to the Memory Agent: *"Here's what happened."*

---

## What About Failures?

If the Action Agent calls back saying *"Task #4 failed — audio file was corrupt"*:

1. Crosses off #4 as ❌ failed
2. Looks at what depended on #4: tasks #6 and #7 can't run anymore
3. Marks #6 and #7 as ⏭️ skipped
4. But tasks #5 and #8 are on a **different branch** — they keep going
5. Task #9 (the report) runs with whatever results are available

The Dispatch Agent doesn't give up on the entire workflow just because one branch fails. Independent branches continue.

---

## The Key Idea

**The Dispatch Agent doesn't run continuously.** It wakes up when someone calls, looks at the notepad, makes a decision, sends some messages, saves the notepad, and goes back to sleep.

Each "wake up" is one n8n workflow execution:

> **Wake up → Read notepad → Decide → Write notepad → Send messages → Sleep**

---

## The n8n Workflow (5 Nodes)

```
 ┌──────────────┐
 │ 1. Webhook   │  ← Receives calls (from Planning Agent or callbacks)
 └──────┬───────┘
        ▼
 ┌──────────────┐
 │ 2. S3 GET    │  ← Pick up the notepad from Wasabi
 └──────┬───────┘
        ▼
 ┌──────────────┐
 │ 3. Code (JS) │  ← The brain: read notepad, decide what to do next
 └──────┬───────┘
        ▼
 ┌──────────────┐
 │ 4. S3 PUT    │  ← Save updated notepad back to Wasabi
 └──────┬───────┘
        ▼
 ┌──────────────┐
 │ 5. HTTP      │  ← Make the calls (dispatch tasks or send completion)
 └──────────────┘
```

That's it. Five steps, every time:

1. **Listen** for a call
2. **Remember** where we left off
3. **Think** about what to do next
4. **Write down** the new status
5. **Act** on the decision

---

## Where Things Live

| What | Where | Format |
|------|-------|--------|
| Agent instructions | `agent/operational/dispatch.md` | Markdown |
| Decision logic | `dispatchEngine.js` | JavaScript |
| Notepad (workflow state) | `wasabi://bucket/dispatch-state/{session_id}.json` | JSON |
| n8n workflow | n8n UI / exported JSON | n8n workflow |

---

*Last updated: February 23, 2026*
