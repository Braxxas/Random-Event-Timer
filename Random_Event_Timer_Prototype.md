# AI Dungeon Random Event Timer — Prototype

This prototype reads a control Story Card, rolls an action interval, and injects the configured event when it is due. All hooks count AI Dungeon actions, including `Continue` actions.

## Control Story Card

Create a Story Card with **Type** `Random Event Control`. Its **Entry** can follow this format. The preset event is Dr. Henrietta Jekyll transforming into Ms. Edwina Hyde; edit the text under `Triggered Event` to change it. The event instruction may span multiple lines. Keep `Notes` after the event text.

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

`Frequency` may be `Frequent`, `Normal`, `Rare`, or `Custom`. The built-in ranges are Frequent 6–12, Normal 12–20, and Rare 20–40. Custom uses `CustomMin` and `CustomMax`. Bounds are inclusive. `CustomMin` and `CustomMax` are ignored for the three presets. The control card starts at `Rare`.

For an event that should use another Story Card, mention one of that card's trigger keys in the event text (for example, `Lucy`). The Context hook looks for matching keys across all Story Cards and includes matching entries for that event. Keys are the card's `keys` values, not a separately exposed title field.

## Input hook

```javascript
const modifier = (text) => {
  const findControlCard = () => (storyCards || []).find(card =>
    String(card.type || "").trim().toLowerCase() === "random event control"
  );

  const readSettings = () => {
    const card = findControlCard();
    const entry = card ? String(card.entry || "") : "";
    const frequency = (entry.match(/^\s*Frequency\s*:\s*(.+?)\s*$/im) || [])[1] || "Rare";
    const customMin = Number((entry.match(/^\s*CustomMin\s*:\s*(-?\d+)\s*$/im) || [])[1]);
    const customMax = Number((entry.match(/^\s*CustomMax\s*:\s*(-?\d+)\s*$/im) || [])[1]);
    let min, max;

    switch (frequency.trim().toLowerCase()) {
      case "frequent": [min, max] = [6, 12]; break;
      case "rare": [min, max] = [20, 40]; break;
      case "custom":
        min = Number.isFinite(customMin) && customMin > 0 ? Math.floor(customMin) : 6;
        max = Number.isFinite(customMax) && customMax > 0 ? Math.floor(customMax) : 12;
        break;
      case "normal": [min, max] = [12, 20]; break;
      case "rare":
      default: [min, max] = [20, 40]; break;
    }

    min = Math.max(1, Math.min(10000, min));
    max = Math.max(1, Math.min(10000, max));
    if (min > max) [min, max] = [max, min];

    const eventMarker = entry.match(/^[ \t]*Triggered Event[ \t]*:[ \t]*/im);
    const eventText = eventMarker ? entry.slice(eventMarker.index + eventMarker[0].length) : "";
    const notesAt = eventText.search(/^[ \t]*Notes[ \t]*:/im);
    const event = (notesAt < 0 ? eventText : eventText.slice(0, notesAt)).trim();
    return { card, min, max, event };
  };

  const roll = (min, max) => Math.floor(Math.random() * (max - min + 1)) + min;
  const settings = readSettings();
  const count = Number(info && info.actionCount) || 0;

  // An empty event disables the timer. The countdown starts when an event is supplied.
  if (!settings.event) {
    state.randomEventNextAt = null;
    state.randomEventDue = false;
    state.randomEventRange = null;
    return { text };
  }

  const range = `${settings.min}:${settings.max}`;
  if (!Number.isFinite(state.randomEventNextAt) || state.randomEventRange !== range) {
    state.randomEventRange = range;
    state.randomEventNextAt = count + roll(settings.min, settings.max);
  }

  if (count >= state.randomEventNextAt) state.randomEventDue = true;
  return { text };
};

// Keep this final line in the Input hook.
modifier(text)
```

## Context hook

```javascript
const modifier = (text) => {
  if (!state.randomEventDue) return { text };

  const controlCard = (storyCards || []).find(card =>
    String(card.type || "").trim().toLowerCase() === "random event control"
  );
  const entry = controlCard ? String(controlCard.entry || "") : "";
  const eventMarker = entry.match(/^[ \t]*Triggered Event[ \t]*:[ \t]*/im);
  const eventText = eventMarker ? entry.slice(eventMarker.index + eventMarker[0].length) : "";
  const notesAt = eventText.search(/^[ \t]*Notes[ \t]*:/im);
  const event = (notesAt < 0 ? eventText : eventText.slice(0, notesAt)).trim();
  if (!event) return { text };

  const keysOf = card => {
    if (Array.isArray(card.keys)) return card.keys.map(String);
    return String(card.keys || "").split(",").map(key => key.trim()).filter(Boolean);
  };

  // Any Story Card can be selected: a key must appear in the event instruction.
  const matchingCards = (storyCards || []).filter(card => {
    if (card === controlCard) return false;
    const keys = keysOf(card).filter(key => key.length >= 3);
    return keys.some(key => event.toLowerCase().includes(key.toLowerCase()));
  });

  const cardText = matchingCards.map(card => {
    const label = keysOf(card)[0] || card.type || "Story Card";
    return `\n[Story Card: ${label}]\n${String(card.entry || "")}`;
  }).join("\n");

  const injected = `\n\n[Random Event — enact this event naturally in the story now. Use the referenced Story Card information when provided. Do not merely repeat these instructions.]\n${event}${cardText}\n[End Random Event]`;
  return { text: text + injected };
};

// Keep this final line in the Context hook.
modifier(text)
```

## Output hook

```javascript
const modifier = (text) => {
  if (!state.randomEventDue) return { text };

  const controlCard = (storyCards || []).find(card =>
    String(card.type || "").trim().toLowerCase() === "random event control"
  );
  const entry = controlCard ? String(controlCard.entry || "") : "";
  const frequency = (entry.match(/^\s*Frequency\s*:\s*(.+?)\s*$/im) || [])[1] || "Rare";
  const customMin = Number((entry.match(/^\s*CustomMin\s*:\s*(-?\d+)\s*$/im) || [])[1]);
  const customMax = Number((entry.match(/^\s*CustomMax\s*:\s*(-?\d+)\s*$/im) || [])[1]);
  let min, max;

  switch (frequency.trim().toLowerCase()) {
    case "frequent": [min, max] = [6, 12]; break;
    case "rare": [min, max] = [20, 40]; break;
    case "custom":
      min = Number.isFinite(customMin) && customMin > 0 ? Math.floor(customMin) : 6;
      max = Number.isFinite(customMax) && customMax > 0 ? Math.floor(customMax) : 12;
      break;
    case "normal": [min, max] = [12, 20]; break;
    case "rare":
    default: [min, max] = [20, 40]; break;
  }

  min = Math.max(1, Math.min(10000, min));
  max = Math.max(1, Math.min(10000, max));
  if (min > max) [min, max] = [max, min];

  const roll = Math.floor(Math.random() * (max - min + 1)) + min;
  const count = Number(info && info.actionCount) || 0;
  state.randomEventDue = false;
  state.randomEventRange = `${min}:${max}`;
  state.randomEventNextAt = count + roll;
  return { text };
};

// Keep this final line in the Output hook.
modifier(text)
```

## Prototype limits

- The event is offered to the AI in its next model context after the action count reaches the threshold; the script cannot guarantee that every model will execute the instruction exactly.
- Story Card selection matches event text against card trigger keys. Give each card a distinctive key that the event instruction can name. Very broad keys can match extra cards.
- If the event text is empty, the timer is disabled. If the control card is missing, the timer has no event to trigger.
- This prototype schedules the next interval after the AI responds to a triggered event. Changing frequency/custom bounds while waiting starts a fresh interval from the current action count.
