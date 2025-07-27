# Persistent Memory (Short‑ & Long‑Term) for Home Assistant 

This guide documents a memory system that lets Assist recall **long‑term facts** and **short‑term recent actions**, and injects that context into the LLM prompt at runtime.

It uses:

- A **To‑do list** (`todo.ai_persistent_memory`) as the canonical store
- Two **tags** to separate item types:
  - `USER FACTS AND MEMORIES:`  ← long‑term facts you explicitly ask it to remember
  - `RECENT ACTION:`            ← short‑term actions the assistant just performed
- A compact **JSON mirror** of prioritized items in `input_text.ai_persistent_memory` for easy prompt injection
- An optional `input_text.recent_action_memory` for a single last‑action string (useful for follow‑ups)

> This file focuses on memory. If you also want multi‑turn follow‑ups, see **Vocal Daisy Chaining** and optionally add its “follow‑up tail” to the examples below. You can also find more example code at the end of this document covering this.

---

## 1) Create Helpers (configuration.yaml)

```yaml
# Stores the prioritized memory list (JSON array of strings)
input_text:
  ai_persistent_memory:
    name: AI Persistent Memory
    max: 255

  # Optional: the last single action, for lightweight context
  recent_action_memory:
    name: Recent Action Memory
    initial: ""
```

> Note: `input_text.max: 255` is usually enough for prioritized items. If your list grows or you keep longer strings, consider a higher limit or splitting across multiple helpers.

---

## 2) Create the To‑Do List (UI)

Create a list named **AI persistent memory** (or your preferred name) in **Settings → Devices & Services → Helpers → To‑do list**. Confirm its entity id (e.g., `todo.ai_persistent_memory`).

This list is the **single source of truth**. Long‑term facts and short‑term actions are appended here with a prefix tag.

---

## 3) Capture Long‑Term Facts by Voice

Users can say: *“Remember that {memory\_text}”*. The automation below prefixes the text and appends it to the to‑do list.

> Place in **automations.yaml**

```yaml
- alias: Memory Capture (Long‑Term Facts)
  trigger:
    - platform: conversation
      command:
        - remember that {memory_text}
        - remember this {memory_text}
        - remember the {memory_text}
        - please remember {memory_text}
        - i want you to remember {memory_text}
  action:
    - service: todo.add_item
      data:
        entity_id: todo.ai_persistent_memory
        item: "USER FACTS AND MEMORIES: {{ trigger.slots.memory_text }}"
    - service: script.prioritize_memories
    # Optional: add a follow‑up prompt + boolean flip (see Vocal Daisy Chaining)
```

**Why prefix?** The prefix makes it trivial to classify items during prioritization.

---

## 4) Log Short‑Term Recent Actions

Whenever an automation performs something on behalf of the user (playlist, TV action, app launch, etc.), append a `RECENT ACTION:` item and (optionally) capture a compact last‑action string.

```yaml
# Example tail inside any action automation (e.g., after launching YouTube)
- service: todo.add_item
  target:
    entity_id: todo.ai_persistent_memory
  data:
    item: 'RECENT ACTION: I launched YouTube on their TV'
- service: script.prioritize_memories
- service: input_text.set_value
  data:
    entity_id: input_text.recent_action_memory
    value: launched_youtube
# Optional: add a follow‑up prompt + boolean flip (see Vocal Daisy Chaining)
```

> The **recent action** list is **rolling**. The prioritization script (below) keeps only the **latest 4** action items and discards older ones.

---

## 5) Prioritize & Trim (scripts.yaml)

This script fetches to‑do items, keeps **all** long‑term facts, and only the **last four** recent actions. It writes a JSON array (just the `summary` strings) to `input_text.ai_persistent_memory`. Older actions are marked **completed** and then removed.

```yaml
prioritize_memories:
  alias: Prioritize AI Memories
  sequence:
    - service: todo.get_items
      target:
        entity_id: todo.ai_persistent_memory
      data:
        status: needs_action
      response_variable: all_memories
    - variables:
        all_items: "{{ all_memories['todo.ai_persistent_memory']['items'] }}"
        # Split by type
        user_memories: "{{ all_items | selectattr('summary', 'contains', 'USER FACTS AND MEMORIES:') | list }}"
        action_items: "{{ all_items | selectattr('summary', 'contains', 'RECENT ACTION:') | list }}"
        # For actions, keep only the last 4 (most recent)
        limited_actions: "{{ action_items[-4:] }}"
        excess_actions: "{{ action_items[:-4] if action_items|length > 4 else [] }}"
        # Combine with user memories first
        prioritized: "{{ user_memories + limited_actions }}"
    # Store prioritized items as a JSON array of strings
    - service: input_text.set_value
      target:
        entity_id: input_text.ai_persistent_memory
      data:
        value: "{{ prioritized | map(attribute='summary') | list | tojson }}"
    # Mark excess action items as completed (will be removed next step)
    - repeat:
        count: "{{ excess_actions | length }}"
        sequence:
          - service: todo.update_item
            target:
              entity_id: todo.ai_persistent_memory
            data:
              item: "{{ excess_actions[repeat.index-1].summary }}"
              status: "completed"
    # Remove completed items to keep the list tidy
    - service: todo.remove_completed_items
      target:
        entity_id: todo.ai_persistent_memory
```

> If you ever want to change the rolling window, adjust `action_items[-4:]` and `action_items[:-4]` accordingly.

---

## 6) Keep Memory Fresh (automations.yaml)

Run prioritization on a schedule and on HA startup:

```yaml
- alias: Update AI Persistent Memory
  trigger:
    - platform: time_pattern
      minutes: "/2"          # every 2 minutes
    - platform: homeassistant
      event: start
  action:
    - service: script.prioritize_memories
  mode: single
```

---

## 7) Maintenance Utilities (optional)

### Nightly clean (automations.yaml)

Removes any stray completed items and refreshes the JSON mirror. You can also just delete items using the HA UI directly.

```yaml
- alias: Clean AI Memory List
  trigger:
    - platform: time
      at: "04:00:00"
  action:
    - service: todo.remove_completed_items
      target:
        entity_id: todo.ai_persistent_memory
    - service: todo.update_item
      target:
        entity_id: todo.ai_persistent_memory
      data:
        item: '*'
        status: completed
    - service: todo.remove_completed_items
      target:
        entity_id: todo.ai_persistent_memory
    - service: script.prioritize_memories
  mode: single
```

### Manual clear (scripts.yaml)

Clears the list entirely if you need to reset.

```yaml
clear_ai_memory:
  alias: Clear AI Memory List
  sequence:
    # 1) Read all items
    - service: todo.get_items
      target:
        entity_id: todo.ai_persistent_memory
      data: {}
      response_variable: todo_list
    # 2) Mark each as completed
    - repeat:
        for_each: "{{ todo_list['todo.ai_persistent_memory']['items'] }}"
        sequence:
          - service: todo.update_item
            data:
              item: "{{ repeat.item.summary }}"
              status: "completed"
            target:
              entity_id: todo.ai_persistent_memory
    # 3) Remove completed
    - service: todo.remove_completed_items
      target:
        entity_id: todo.ai_persistent_memory
```

---

## 8) Inject Memory into the Assist Prompt

In your Assist (system) prompt, render the JSON mirror as **plain text** the model can read.

```jinja
Below are RECENT ACTIONS taken by you and also long term USER FACTS AND MEMORIES the user wants you to keep in mind. Only mention or reference the memories or facts if it makes sense in context. RECENT ACTIONS can be referenced if it is relevant to the discussion or makes sense to keep the conversation going with the user. For example asking how their experience was with music or entertainment.

# Memory
{% set items = states('input_text.ai_persistent_memory') | from_json %}
{% for item in items %}{{ loop.index }}. {{ item }}
{% endfor %}

Do NOT use any special characters, markdown, asterisk, fancy text markdown or anything other than plain text responses. Reply as if the text will be read aloud through a text to speech tts system so plain text is crucial.
```

**Notes**

- The list is already prioritized/trimmed by the script, so rendering is fast.
- Keep this in the **system** prompt so the model consistently sees it.
- Because the facts and actions are tagged up front, your LLM can choose to reference only what’s relevant.

---


## 9) FAQ & Tuning

- **Why a To‑do list?** It’s a durable, UI‑editable store and works well with Assist.
- **Why also a JSON mirror?** Injecting a compact JSON array into the prompt is fast and avoids heavy templating each turn.
- **Change the action window**: Edit `action_items[-4:]` to the number of recent actions you want.
- **Reorder priorities**: If you want actions to appear before facts in the prompt, swap the order when building `prioritized`.
- **Multi‑user?** Add additional user‑scoped tags (e.g., `@alice`, `@bob`) and filter for them in the prioritization variables.

---

### Putting it together (pattern recap)

1. **Capture** facts/actions → append to `todo.ai_persistent_memory` with the appropriate prefix.
2. **Prioritize** on a timer or after each append → update `input_text.ai_persistent_memory`.
3. **Inject** the JSON mirror into the Assist prompt.
4. *(Optional)* Use `input_text.recent_action_memory` in your Vocal Daisy Chaining flow to add a one‑line context for follow‑ups.

---

## 10) Examples (Copy‑Paste Ready)

Below are **sanitized** examples that you can drop into your setup. Replace placeholder entity IDs and addresses (e.g., `media_player.your_tv`, `AA:BB:CC:DD:EE:FF`) with your own.

> These examples show **memory capture** and (optionally) **vocal daisy chaining** together, but you can omit the follow‑up tail if you prefer single‑turn behavior.

### 10.1 Memory Capture — Minimal (no follow‑up)

```yaml
- alias: Memory Capture (Long‑Term Facts) — Minimal
  trigger:
    - platform: conversation
      command:
        - remember that {memory_text}
        - remember this {memory_text}
        - remember the {memory_text}
        - please remember {memory_text}
        - i want you to remember {memory_text}
  action:
    - service: todo.add_item
      data:
        entity_id: todo.ai_persistent_memory
        item: "USER FACTS AND MEMORIES: {{ trigger.slots.memory_text }}"
    - service: script.prioritize_memories
  mode: single
```

### 10.2 Memory Capture — With Follow‑Up (vocal daisy chaining)

```yaml
- alias: Memory Capture with Slots and Follow‑Up
  trigger:
    - platform: conversation
      command:
        - remember that {memory_text}
        - remember this {memory_text}
        - remember the {memory_text}
        - please remember {memory_text}
        - i want you to remember {memory_text}
  action:
    - service: todo.add_item
      data:
        entity_id: todo.ai_persistent_memory
        item: 'USER FACTS AND MEMORIES: {{ trigger.slots.memory_text }}'
    - service: script.prioritize_memories
    - set_conversation_response: >
        {{ [
          "I'll make sure to remember that. Is there anything else you need?",
          "I've noted that down. Is there anything else you want me to remember or do?",
          "Got it! I'll remember that. Anything else?",
          "Noted. Is that everything for now or do you need anything else?",
        ] | random }}
    - service: input_boolean.turn_off
      target:
        entity_id: input_boolean.follow_up_mode
    - delay: "00:00:01"
    - service: input_boolean.turn_on
      target:
        entity_id: input_boolean.follow_up_mode
  mode: single
```

### 10.3 End‑to‑End: TV Power On → Log Action → Ask Follow‑Up

> Example shown for an LG webOS TV. Wake‑on‑LAN and source selection can differ by brand/model. Replace entities and MAC address accordingly.

```yaml
- alias: 'Voice: Turn On TV with Follow‑Up and Memory'
  trigger:
    - platform: conversation
      command:
        - turn on the tv
        - power on the tv
  action:
    - service: wake_on_lan.send_magic_packet
      data:
        mac: AA:BB:CC:DD:EE:FF   # ← replace with your TV's MAC
    - delay: "00:00:05"
    - service: todo.add_item
      target:
        entity_id: todo.ai_persistent_memory
      data:
        item: 'RECENT ACTION: I turned on their TV'
    - service: script.prioritize_memories
    - service: input_text.set_value
      data:
        entity_id: input_text.recent_action_memory
        value: turned_on_tv
    - set_conversation_response: >
        The TV is now on. Would you like to open an app or do something else?
    - service: input_boolean.turn_off
      target:
        entity_id: input_boolean.follow_up_mode
    - delay: "00:00:01"
    - service: input_boolean.turn_on
      target:
        entity_id: input_boolean.follow_up_mode
  mode: single
```

### 10.4 End‑to‑End: TV Power Off → Log Action → Ask Follow‑Up

```yaml
- alias: 'Voice: Turn Off TV with Follow‑Up and Memory'
  trigger:
    - platform: conversation
      command:
        - turn off the tv
        - power off the tv
        - shut off the tv
        - turn the tv off
        - switch off the tv
  action:
    - service: media_player.turn_off
      target:
        entity_id: media_player.your_tv   # ← replace with your TV entity id
    - service: todo.add_item
      target:
        entity_id: todo.ai_persistent_memory
      data:
        item: 'RECENT ACTION: I turned off their TV'
    - service: script.prioritize_memories
    - service: input_text.set_value
      data:
        entity_id: input_text.recent_action_memory
        value: turned_off_tv
    - set_conversation_response: >
        I've turned off the TV. Is there anything else you need?
    - service: input_boolean.turn_off
      target:
        entity_id: input_boolean.follow_up_mode
    - delay: "00:00:01"
    - service: input_boolean.turn_on
      target:
        entity_id: input_boolean.follow_up_mode
  mode: single
```

### 10.5 End‑to‑End: Launch YouTube → Log Action → Ask Follow‑Up

```yaml
- alias: 'Voice: Launch YouTube with Follow‑Up and Memory'
  trigger:
    - platform: conversation
      command:
        - open youtube
        - launch youtube
        - start youtube
        - turn on youtube
        - put on youtube
  variables:
    tv_is_off_or_unavailable: >
      {% set tv_state = states('media_player.your_tv') %}
      {{ tv_state in ['off', 'unavailable'] }}
  action:
    - choose:
        - conditions:
            - condition: template
              value_template: "{{ tv_is_off_or_unavailable }}"
          sequence:
            - service: wake_on_lan.send_magic_packet
              data:
                mac: AA:BB:CC:DD:EE:FF   # ← replace with your TV's MAC
            - delay: "00:00:05"
    - service: media_player.select_source
      data:
        entity_id: media_player.your_tv
        source: YouTube
    - service: todo.add_item
      target:
        entity_id: todo.ai_persistent_memory
      data:
        item: 'RECENT ACTION: I launched YouTube on their TV'
    - service: script.prioritize_memories
    - set_conversation_response: >
        YouTube is up. Anything else?
    - service: input_boolean.turn_off
      target:
        entity_id: input_boolean.follow_up_mode
    - delay: "00:00:01"
    - service: input_boolean.turn_on
      target:
        entity_id: input_boolean.follow_up_mode
  mode: single
```

> Tip: If you maintain the **Vocal Daisy Chaining** setup, you can add the follow‑up tail to any action automation to keep the session alive for quick chaining. If not, just omit the final four lines in each example.

Please see the Vocal-Command-Daisy-Chaining section for more details on the follow-up mode.