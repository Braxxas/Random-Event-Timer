# Random Event Timer v0.1 — standalone

This version contains only the Random Event Timer. It does not include Inner Self, Auto-Cards, or Narrative Pathfinder.

## Install

Replace the contents of the Library, Input, Context, and Output tabs with the corresponding blocks below. Use all four together.

## Automatic control card

On the first Input hook, the script looks for a Story Card with Type `Random Event Control`. If it does not find one, it creates the card automatically with **Frequency: Rare** and the sample event below. If a card already exists, its settings are preserved. Edit the card's Entry to change the interval or event.

The default sample event is: Dr. Henrietta Jekyll transforms into Ms. Edwina Hyde.

## Configure frequency and event

The control card's Entry uses this format:

```text
Frequency: Rare
CustomMin: 6
CustomMax: 12
Triggered Event:
On the next scene, Dr. Henrietta Jekyll transforms into Ms. Edwina Hyde. Show the change through the characters' actions and reactions.
Notes:
The timer waits the selected random interval, then asks the AI to enact the Triggered Event. Continue actions count. Edit Frequency to change the preset; edit CustomMin and CustomMax for a custom interval. To use those numbers, set Frequency to Custom. Replace the Triggered Event text to change the event. To include another Story Card, mention one of its trigger keys in the event instructions.

Preset intervals (random AI Dungeon actions between events; bounds are inclusive):
- Frequent: 6–12 actions; shortest waits.
- Normal: 12–20 actions; medium waits.
- Rare: 20–40 actions; longest waits (default).
- Custom: Fill in your own numbers above and set Frequency to Custom.
```

Replace the instructions under `Triggered Event:` with any event you want the AI to enact. The event may span multiple lines. Keep `Notes:` after the event; Notes are excluded from the event instruction. If the event is empty, the timer is disabled. To include a Story Card in the prompt, mention one of its trigger keys in the event instruction; broad keys may match extra cards.

## Timer behavior

- Presets use inclusive random action counts: Frequent 6–12, Normal 12–20, Rare 20–40.
- Continue actions count.
- When due, the event instruction and any key-matched Story Card entries are added to Context. After the AI responds, a new interval is rolled.
- Changing the interval range while waiting starts a fresh interval from the current action count.
- The script asks the AI to enact the event naturally; model compliance is not guaranteed.

This standalone version contains no Inner Self source.

## Copy/paste script tabs

Paste each complete block into its matching AI Dungeon scripting tab.

### Library

```javascript
// Random Event Timer v0.1 — standalone, without Inner Self.
globalThis.RandomEventTimer = {
  ensureControlCard() {
    const cards = Array.isArray(globalThis.storyCards) ? globalThis.storyCards : [];
    let card = cards.find(item => String(item.type || "").trim().toLowerCase() === "random event control");
    if (!card) {
      card = cards.find(item => String(item.title || "").trim().toLowerCase() === "random event control");
      if (card) card.type = "Random Event Control";
    }
    if (card || typeof addStoryCard !== "function") return card || null;

    const entry = [
      "Frequency: Rare",
      "CustomMin: 6",
      "CustomMax: 12",
      "Triggered Event:",
      "On the next scene, Dr. Henrietta Jekyll transforms into Ms. Edwina Hyde. Show the change through the characters' actions and reactions.",
      "Notes:",
      "The timer waits the selected random interval, then asks the AI to enact the Triggered Event. Continue actions count. Edit Frequency to change the preset; edit CustomMin and CustomMax for a custom interval. To use those numbers, set Frequency to Custom. Replace the Triggered Event text to change the event. To include another Story Card, mention one of its trigger keys in the event instructions.",
      "",
      "Preset intervals (random AI Dungeon actions between events; bounds are inclusive):",
      "- Frequent: 6-12 actions; shortest waits.",
      "- Normal: 12-20 actions; medium waits.",
      "- Rare: 20-40 actions; longest waits (default).",
      "- Custom: Fill in your own numbers above and set Frequency to Custom."
    ].join("\n");

    try {
      addStoryCard("", entry, "Random Event Control", "Random Event Control", "Automatically created by the Random Event Timer.");
    } catch (error) {
      if (typeof log === "function") log(`Random Event Timer could not create its control card: ${error.message}`);
      return null;
    }
    return cards.find(item => String(item.type || "").trim().toLowerCase() === "random event control") || null;
  },

  settings() {
    const cards = Array.isArray(globalThis.storyCards) ? globalThis.storyCards : [];
    const card = cards.find(item => String(item.type || "").trim().toLowerCase() === "random event control");
    const entry = card ? String(card.entry || "") : "";
    const frequency = (entry.match(/^\s*Frequency\s*:\s*(.+?)\s*$/im) || [])[1] || "Rare";
    const customMin = Number((entry.match(/^\s*CustomMin\s*:\s*(-?\d+)\s*$/im) || [])[1]);
    const customMax = Number((entry.match(/^\s*CustomMax\s*:\s*(-?\d+)\s*$/im) || [])[1]);
    let min, max;
    switch (frequency.trim().toLowerCase()) {
      case "frequent": [min, max] = [6, 12]; break;
      case "normal": [min, max] = [12, 20]; break;
      case "custom":
        min = Number.isFinite(customMin) && customMin > 0 ? Math.floor(customMin) : 6;
        max = Number.isFinite(customMax) && customMax > 0 ? Math.floor(customMax) : 12;
        break;
      case "rare":
      default: [min, max] = [20, 40]; break;
    }
    min = Math.max(1, Math.min(10000, min));
    max = Math.max(1, Math.min(10000, max));
    if (min > max) [min, max] = [max, min];

    const marker = entry.match(/^[ \t]*Triggered Event[ \t]*:[ \t]*/im);
    const remaining = marker ? entry.slice(marker.index + marker[0].length) : "";
    const notesAt = remaining.search(/^[ \t]*Notes[ \t]*:/im);
    const event = (notesAt < 0 ? remaining : remaining.slice(0, notesAt)).trim();
    return { card, min, max, event };
  },

  roll(min, max) {
    return Math.floor(Math.random() * (max - min + 1)) + min;
  },

  beforeInput() {
    this.ensureControlCard();
    const config = this.settings();
    const count = Number(globalThis.info && globalThis.info.actionCount) || 0;
    if (!config.event) {
      state.randomEventNextAt = null;
      state.randomEventDue = false;
      state.randomEventRange = null;
      return;
    }
    const range = `${config.min}:${config.max}`;
    if (!Number.isFinite(state.randomEventNextAt) || state.randomEventRange !== range) {
      state.randomEventRange = range;
      state.randomEventNextAt = count + this.roll(config.min, config.max);
    }
    if (count >= state.randomEventNextAt) state.randomEventDue = true;
  },

  inject(context) {
    if (!state.randomEventDue) return context;
    const config = this.settings();
    if (!config.event) return context;
    const cards = Array.isArray(globalThis.storyCards) ? globalThis.storyCards : [];
    const keysOf = card => Array.isArray(card.keys)
      ? card.keys.map(String)
      : String(card.keys || "").split(",").map(key => key.trim()).filter(Boolean);
    const matching = cards.filter(card => {
      if (card === config.card) return false;
      return keysOf(card).some(key => key.length >= 3 && config.event.toLowerCase().includes(key.toLowerCase()));
    });
    const cardText = matching.map(card => {
      const label = keysOf(card)[0] || card.type || "Story Card";
      return `\n[Story Card: ${label}]\n${String(card.entry || "")}`;
    }).join("\n");
    return String(context || "") +
      `\n\n[Random Event — enact this event naturally in the story now. Use the referenced Story Card information when provided. Do not merely repeat these instructions.]\n${config.event}${cardText}\n[End Random Event]`;
  },

  afterOutput() {
    if (!state.randomEventDue) return;
    const config = this.settings();
    const count = Number(globalThis.info && globalThis.info.actionCount) || 0;
    state.randomEventDue = false;
    if (!config.event) {
      state.randomEventNextAt = null;
      state.randomEventRange = null;
      return;
    }
    state.randomEventRange = `${config.min}:${config.max}`;
    state.randomEventNextAt = count + this.roll(config.min, config.max);
  }
};
```

### Input

```javascript
const modifier = (text) => {
  RandomEventTimer.beforeInput();
  return { text };
};
modifier(text)
```

### Context

```javascript
const modifier = (text) => {
  text = RandomEventTimer.inject(text);
  return { text };
};
modifier(text)
```

### Output

```javascript
const modifier = (text) => {
  RandomEventTimer.afterOutput();
  return { text };
};
modifier(text)
```
