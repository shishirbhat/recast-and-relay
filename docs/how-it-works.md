# How it works

Recast and Relay are built on one principle: **nothing important moves without being seen, approved and checked.** Software now acts for us, moving our plans between apps and writing code beside us. Both are fast, and both quietly get things wrong. These products put a person in the loop at the one moment that matters, then check the result against reality instead of trusting a "success" message.

- [Recast](#recast): for everyone's everyday context
- [Relay](#relay): for teams of developers and coding agents
- [What we will not do](#what-we-will-not-do)
- [How we know](#how-we-know)

---

## Recast

### The problem
A plan arrives in pieces: the time is in a chat, the place is on a website, the person is in an email. Moving it into a calendar means retyping. Retyping drops details, mixes up dates, and loses where things came from. Today's assistants guess, and act without showing you what they changed.

### The idea: context objects
Recast turns what you select into **objects**. Select a time, a person, a place, a passage or an error message on any page or in any app, and press the pieces together. They become one object: an event, a message, a note, a task, an issue. Every value in it remembers where it came from.

### Seven steps, every time

1. **Capture.** You select what matters. Recast never reads what you did not select.
2. **Compose.** Pieces pressed together become one object. Recast knows which combinations make sense (a person and a time make an event) and refuses the ones that do not, with the reason.
3. **Preview.** You see every field, its source ("from Maya Rao · Chat") and every assumption ("ends after 60 minutes, a default"). When two sources disagree, for example two different times, Recast shows both with their sources and you choose. It never picks silently, and your choice is remembered with what it overruled.
4. **Approve.** Nothing is written until you say so. How much Recast asks depends on the effect, not on how sure it feels: anything that notifies another person, cannot be undone, or rests on a guess asks you to confirm or to press and hold.
5. **Execute.** The object is placed in the app you chose, exactly once. If the connection drops mid-write, Recast does not know whether it landed, says so, and checks the app before trying again. It never sends it twice.
6. **Verify.** Recast reads the item back from the app and compares it, field by field, with what you approved: "5 of 5 fields match", with a link to it. If something differs, it shows exactly which field.
7. **Confirm.** You see what actually happened, and Undo is there wherever the app allows it.

### Recipes and your own workflows
A **recipe** places one object in several apps in one go: "plan and invite" adds the event and drafts a message to the people in it. You review all the steps together and approve once; each step reports its own result ("2 of 3 done", never a blanket success).

You can **save what you just did as a workflow**, with a name, and run it again on new information. Only the steps and destinations are kept, never what you captured.

### Where it runs
- **Chrome extension.** Carry an object across tabs. Objects live only in the browser session.
- **Desktop app** (Mac, Windows, Linux). Recast floats over any app: select, tap Control twice, and the selection becomes an object where it was.
- **API, SDK and MCP server** for developers and AI agents, with the same consent and trust rules ([API](api.md)).

### Connectors
Google Calendar, Outlook Calendar, Gmail drafts, Google Tasks, Google Docs, GitHub issues, Linear issues, and Google Maps and Apple Maps (open with directions). Each connector declares what it can do: create, read back, undo, or only open. Recast offers only what the connector can keep. Where an app cannot be read back, Recast says so instead of claiming it checked.

### Privacy
- Recast works on what you select, nothing else.
- Sign-in tokens for your apps stay on your own computer, in a file only you can read. The extension never sees them.
- In the browser, what you capture lives only for the browser session. The desktop app keeps your grabs in a private library on your own computer, and you can remove any of them. Saved workflows hold steps, not content.
- Nothing is sent to Recast's servers by the extension or the desktop app. The developer API learns from corrections only for accounts that opt in.

---

## Relay

### The problem
Teams now ship code written by several people and several AI agents at once. Each change passes its own tests, Git merges them without a conflict, and the project still breaks, because the changes only fail **together**. One agent renames a field in an API; another, in a different file, still reads the old name. Today that is found after it has shipped. And an agent saying "tests pass" is not proof.

### The idea: every change must earn its way in

1. **Tasks and contracts.** Work is split into tasks, each with a scope (the files it may change). Interfaces that tasks share are versioned **contracts**, and only people accept them.
2. **Guarded agents.** Each agent works in its own copy of the project. A guard in front of the agent stops edits outside its task's scope, and if the guard itself fails, the edit is stopped, not allowed.
3. **Relay's own evidence.** Relay runs the checks itself. What an agent reports about its tests does not count.
4. **Check Together.** Tasks that share a contract are rehearsed **combined**, on a separate copy of the project as it is now, with the project's checks and each task's checks.
5. **Explain.** When a rehearsal is blocked, Relay traces the failure to its cause and labels how sure it is: **confirmed** (which change removed what, which line still reads it, which test fails), **likely**, or **unknown**. It never guesses, and the explanation never changes the decision.
6. **A person approves.** Only a person applies. Relay checks again that nothing moved since the rehearsal: not the project, not the tasks, not the contracts.

### Where it runs
- **Agents.** Claude Code today, through a guard hook and coordination tools. Agents cannot approve or integrate, by design.
- **Command line.** `relay integrate together T-api,T-ui` rehearses and reports. It never applies.
- **Browser IDE.** Live shared editing, presence, line history ("who wrote this line, when, and with which edit"), an agent panel where every request becomes a scoped task, and a Coordination view of what needs a person.
- **VS Code extension.** Join a Relay project from your own checkout, with the same Coordination view and Check Together.

All of these call one service, so they cannot disagree about a result.

---

## What we will not do

- **Act without showing you.** Every write is previewed first.
- **Guess silently.** Assumptions are labelled; conflicts are shown with their sources; explanations say "likely" or "unknown" when that is the truth.
- **Call a write "done" because the app said OK.** Recast reads it back; Relay runs the checks itself.
- **Send something twice.** A lost answer is checked, never resent.
- **Let software approve itself.** Agents cannot accept contracts, approve tasks or apply changes. A saved workflow cannot skip approval.

---

## How we know

Each claim on these pages is backed by automated tests that run on every change (more than a thousand for Recast) and is labelled with what it was proven against:

| Claim | Proven against |
|---|---|
| Recast creates, deduplicates and undoes a Google Calendar event | The real Google Calendar, by hand, on a test account |
| Recast's other connectors | Stand-ins that behave like each app's real API |
| Recast in the browser and on the desktop | Real Chromium and real Electron, driven by automated tests on Linux; Mac and Windows builds made by CI |
| Relay blocks a combined break before it reaches the project | Real Claude Code agents: 6 of 6 trials blocked, and without Relay 6 of 6 shipped the break |
| Relay explains the cause correctly | Real Claude Code agents: "confirmed" and accurate in 3 of 3 trials, judged by running the code |

What is not proven yet: use by independent people and teams. That is the next milestone.
