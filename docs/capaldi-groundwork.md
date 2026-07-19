# Capaldi — groundwork for the real thing

The prototype (`public/capaldi/index.html`) is a single self-contained HTML file:
open it anywhere and play against bots. This doc is the plan for turning it into
the real multiplayer game on your site, and the answer to "how do my friends
submit custom card ideas if the site can't actually implement them?"

## The one architectural decision that matters

**Cards are data. Behavior lives in a small set of effect primitives.**

The prototype already works this way, on purpose:

```js
{ key:'kb', color:'wild', name:"Killer Boy's in a Bad Mood",
  needs:['target','color'], effects:[{t:'targetDraw', n:4}] }
```

A card is a JSON-ish record. Its behavior is the list of `effects` it fires, and
every effect is one of the primitives in the `EFFECTS` registry (`draw`, `skip`,
`reverse`, `targetDraw`, `allOthersDraw`, `rotateHands`, ...). House rules
(stacking, draw-to-match) are toggles that change how those primitives resolve.

This is what makes everything else on your wishlist tractable:

- **Most new card ideas need zero new code.** "Everyone with a red card draws 2"
  is a new combination of existing primitives, not a new mechanic. You paste a
  new record into the pack and it's in the deck.
- **Rule packs compose by construction.** If eight packs exist and the lobby
  votes three in, they can't step on each other, because none of them contain
  code — they all speak the same small vocabulary, and the engine resolves one
  effect queue in play order. Harmony isn't something you test for afterwards;
  it falls out of the design.
- **Genuinely new mechanics are small, contained additions.** "Rotate all
  hands" needed one new primitive (~5 lines). That's the unit of work you bring
  to Claude — not "rewrite the game."

## Target stack (kept deliberately light)

| Piece | Choice | Why |
|---|---|---|
| Server | Node + Express + socket.io (or Fastify + ws) | One process, no build step required, WebSockets for realtime turns |
| DB | SQLite via `better-sqlite3` | Single file on disk, zero admin, perfect for 4–10 friends |
| Client | The prototype's vanilla HTML/JS, split into a couple of files | It already renders the whole game; it just needs to draw *server* state instead of local state |
| Auth | Username + password (bcrypt), cookie session | "Make an account super quick" = one form, two fields |
| Hosting | Any small VPS / existing site of yours | ~one process + one .sqlite file + an uploads folder |

The engine moves server-side and becomes the single source of truth; clients
send intents ("play card 7", "choose blue") and receive state snapshots. That's
also your anti-cheat: nobody's browser knows anyone else's hand.

### Schema sketch

```sql
users        (id, username, pass_hash, avatar_path, created_at)
cards        (id, key, name, color, corner, art_path, effects_json,
              needs_json, stack, copies, pack_id, enabled)
packs        (id, name, description, enabled)
rules        (id, key, name, description, default_on)
submissions  (id, user_id, idea_text, image_path, status, created_at)
games        (id, started_at, winner_id, rules_json)   -- optional history/stats
```

Cards and packs live in the DB (or a `cards/` folder of JSON the server loads —
either works; DB wins once art uploads exist).

## The custom-card submission flow — your actual question

You're right that the site can't implement cards itself. But the site doesn't
need to — it needs to be a good **inbox**, and the repo needs to be easy for an
LLM to extend. The loop:

1. **Submit (friends, in-app):** a "Card Ideas" page — plain text box plus an
   optional image upload ("Killer Boy's in a bad mood, draw four" + a photo of
   Kyle). Row goes into `submissions` with status `new`.
2. **Review (you, admin account):** an admin-only inbox listing submissions.
   Buttons: approve → `accepted`, reject → `rejected` (with a note, so the group
   chat can argue about it).
3. **Implement (you + Claude):** paste the accepted idea into Claude with this
   repo. Because of the data/primitives split, the deliverable is tiny: a card
   record, plus *maybe* one new primitive. This is where the "many iterations of
   what if this card was this" happens — in chat, not in the app.
4. **Ship:** insert the card row (an admin "Add card" form, or a seed script),
   attach the uploaded image as its art, flip `enabled`. Next lobby, it's in the
   ballot. Cards without art get the auto-generated face the prototype already
   draws (color + glyph + name strip).

So: yes — plain-text submissions reviewed on your admin account, implemented
through Claude, shipped as data. The only ideas that touch real code are new
*mechanics*, and those arrive as one small registry function.

**Make the engine legible to LLMs.** Keep the `EFFECTS` registry and pack files
heavily commented (the prototype's header comment is the template). A good
prompt is then just: the idea text + `EFFECTS` + one example card.

## Lobby & rule voting

- Rooms with join codes; lobby shows everyone's avatar (their profile pic).
- The ballot the prototype fakes becomes real: every enabled pack and house
  rule is listed; players tap IN/OUT; majority wins, host breaks ties.
- The winning rule set is stamped into the game row (`rules_json`) so the
  engine for that game is fully described by data.

## Roadmap

- **v0.1 (done, this prototype):** full turn engine vs bots — stacking draw
  chains, hand rotation, targeted draws, wilds, draw-to-match, 2–10 players,
  rule toggles, phone-first table UI.
- **v0.2:** split engine from UI (`engine.js` runs headless — the prototype's
  `G.auto()` bot-vs-bot harness becomes the regression test); targeted-draw
  stacking ("pass the +10 on") as a rule toggle.
- **v0.3:** server + SQLite + accounts + one room, hands hidden server-side.
- **v0.4:** lobby ballot, profile pictures, card art uploads, submissions inbox.
- **v0.5:** polish — scoring across rounds, stats ("Kyle has eaten 212 cards"),
  spectator view, sounds off by default.

## Things decided so you don't have to re-decide them

- **Reverse + Musical Chairs** already interact correctly: rotation reads the
  *current* direction at play time.
- **Stacking is one mechanism**, not per-card logic: a `pending` counter that
  any card with a `stack` value may answer. New draw cards join the rule by
  setting one field.
- **One card left auto-shouts "CAPALDI!"** — no call button in v0; a manual
  call-out with penalty can be a later house rule on the ballot.
- **Wild Draw 4 has no challenge rule** — house games don't want it.
