# Ice Hockey Drill Animator: Design

Date: 2026-11-03. Status: for approval.

Items marked **(my call)** were decided by the writer, not the user. Items marked **(my guess)** need confirming during planning.

## Intent and why

Coaches explain drills with static diagrams: many lines on a rink, no clear order, no sense of timing. Players can't see what happens in parallel or in what sequence. Timing is the crux. One player must be in a spot to receive a pass while another is elsewhere. Players starting from different places must reach their destinations at the same moment an event happens.

The app lets a coach build a drill once, set up its timing, watch it play as an animation, and share it with players as a video or GIF they can open in WhatsApp.

**Coach:** builds drills at home, on a **laptop**. **Players:** watch on their **phones**, with nothing to install, only a file received in WhatsApp.

## Goals

1. Draw a drill on a full or half ice rink: place players and cones, draw paths, define puck actions.
2. Express timing through events and rules, not clock times.
3. Play the drill back with play, pause, scrub and a speed control, including slow motion.
4. Make the sequence readable in playback and in exports.
5. Export the animation as a shareable video or GIF.
6. Save drills locally, and import and export them as files on the coach's device.
7. Undo, and delete actions or players.

## Non-goals (version 1)

- Repeating rounds or "continuous" drills. The animation shows **one round**, with waiting players standing in line.
- Several drills in parallel on one rink.
- Goalies moving to different locations.
- More than one puck. Equipment other than cones. Shared rink zones.
- Alternative branches inside one drill (such as "short pass or wide pass"). Build them as separate drills.
- Editing by dragging bars on the timeline. The timeline is **read-only**; "complex editing" may come later.
- Editing on a phone. Phones are for viewing the exported file only.
- A player-facing app, accounts, a cloud library, a server.
- Automatically fixing timing conflicts.

## Anti-goals

- Anything that makes players install something to view a drill.
- A timing model that forces the coach to compute seconds by hand.
- Playback where a busy drill is unreadable.

## Scope of version 1

- **Rink:** a standard ice rink with two nets. **The coach chooses a full rink or a half rink** for each drill.
- **Puck:** exactly one.
- **Players:** placed on demand as goalie, forward or defenceman. Players are numbered (1, 2, 3...) as in coaching books.
- **Cones:** placed on demand as fixed spots that paths can go around.
- **Goalies:** a goalie stays in a net. A goalie can take puck actions such as passing and receiving. *(my call, from the "Short pass, wide pass" drill)*
- **One round of one drill** on the rink at a time. Waiting players stand in line.

## Puck model

The puck is **thrown, received, carried, dropped and picked up**:

- **Throw** a pass or shot at the one fixed puck speed. Duration is distance divided by that speed.
- **Receive.** The puck goes to a player, who then carries it along their path.
- **Carry.** While carried, the puck moves with the player.
- **Drop** at a location. The puck stays there until picked up.
- **Pick up** at a location.
- **One-touch pass:** receive and immediately pass on without carrying. *(my call)*
- Passes are aimed at **where the receiver will be** when the puck arrives, worked out from the receiver's path and speed.
- Shots end at a net. Whether a goal or a save is shown is left to the builder.

## Timing model

A drill is players plus one puck, each with a path, linked by rules. **The coach edits rules only**; the timeline displays the result.

- **Speed.** Each player has an adjustable speed.
- **"Starts when" rule.** An action begins when another event happens: a player reaches a spot or crosses a line (such as the blue line), a pass is released, or a pass is received.
- **"Delay" rule.** An action begins N seconds after an event. *(my call, from "player 2 has delayed")*
- **"Arrive together" rule.** When players from different locations should reach their destinations at the same event, the app works out when each must start, given their speeds. *(my guess at the mechanism)* Any event can be the anchor. A player can wait at the destination until the event. *(my call)*
- **Conflicts.** If a rule can't be met, the app shows a warning and the coach fixes it. If two rules conflict on one player, that is also a warning. The app never changes anything silently.

## Editor (direction "A")

- The rink with paths is on top, with a **read-only timeline** underneath: one row per player plus a puck row, on a shared clock. (Mockup: `editor-mockups.html`, option A.)
- A play bar has play/pause, a scrubber, elapsed and total time, and a speed control.
- Warnings appear on the timeline and rink at the conflict.
- **Path styles** distinguish skating forward, skating backward, skating with the puck, and passing or shooting, like the wavy and dashed lines in coaching books. *(my call)*
- **Undo**, and **delete** of players, cones and actions. Redo is up to the builder.
- Desktop (laptop) browser is the target. Phone editing is not needed.

## Readability (playback and exports)

- **Numbered badges** on every action. A **step** is a group of actions that start on the same event; parallel actions share a number. *(my call)* Badges must look different from player numbers.
- The **active player and active action are highlighted**.
- **Past and future actions are drawn more transparent.**
- The exported video uses the same look.

## Sharing and storage

- **Export** to MP4 by default, with GIF secondary. Target 720p, at most 60 seconds, under about 16 MB for WhatsApp. *(my call)* Players watch on iPhone or Android, which both play MP4. The recording method is left to the builder, chosen for desktop browsers.
- Drills are stored on the coach's device.
- **Import and export of drills as files** (the user's explicit requirement). The format is documented and **versioned**, and import validates the file as untrusted input. *(my call)*

## Technical choices (my calls, not the user's)

- A **web app** (runs in the browser), not a store-distributed mobile app.
- **TypeScript** with modern tooling.
- A **fresh start**. The repos under `/Users/epaz/development/bitbucket` (`mycoachingassistant`, `mydrillbook`, `mystrategyboard-old`) are sources of ideas only.
- **No server** in v1.
- Video export built into the app.

## Test drills

Four drills from a coaching book, which the user supplied as photos, are the acceptance examples. A good version 1 can build all of them as one round:

1. **Russian Stretch, 2v0:** a delayed start, a puck pickup in the corner, blue-line crossing, and a pass before the shot.
2. **Sweden, three one-touch passes:** cones, skating backward and a pivot, one-touch passes, and a shot.
3. **Short pass, wide pass:** the goalie passes the puck. Build the short and the wide variant as two separate drills.
4. **One-touch pass and shoot:** a half rink, a cone, a waiting line, and a shot.

## Left to the builder

Rendering approach, framework, file format, export pipeline, how paths and speeds become times, preset speeds, rink artwork, how a goal or save is shown, and redo.
