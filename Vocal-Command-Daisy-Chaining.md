# Vocal Daisy Chaining for Home Assistant Assist

This guide explains how to enable simple voice daisy chaining using Home Assistant's Assist. After completing a command, the assistant can prompt for additional input (e.g., "Want me to do anything else?") and keep listening for a short time. This allows you to chain multiple commands together naturally.

## Overview

We use a boolean helper (`input_boolean.follow_up_mode`) and a short timeout to enable and disable follow-up listening. When a command completes, follow-up mode is toggled on, giving the user \~5 seconds to say something else. A separate automation re-triggers the assistant using `conversation.process` and `start_conversation` to continue the flow.

## 1. Create Required Helper

Add the following to your `configuration.yaml`:

```yaml
input_boolean:
  follow_up_mode:
    name: Follow Up Mode
    initial: off
```

This boolean tracks whether follow-up mode is currently active.

### Initialize on Home Assistant startup (recommended)

Ensure follow-up mode begins in a clean state after a restart:

```yaml
- alias: Initialize Follow-Up Mode at startup
  trigger:
    - platform: homeassistant
      event: start
  action:
    - service: input_boolean.turn_off
      target:
        entity_id: input_boolean.follow_up_mode
  mode: single
```

## 2. Enable Follow-Up Mode at End of Automation

At the end of your voice automation, add a **follow-up question** and refresh the listener by toggling the boolean:

```yaml
# Example tail of any voice automation
- set_conversation_response: >
    The action is complete. Would you like me to do anything else?
- service: input_boolean.turn_off
  target:
    entity_id: input_boolean.follow_up_mode
- delay: "00:00:01"
- service: input_boolean.turn_on
  target:
    entity_id: input_boolean.follow_up_mode
```

This ensures the assistant asks a prompt and the follow-up listener is refreshed.

## 3. Continue Conversation When Assistant Becomes Idle

This automation triggers when the assistant finishes responding, and restarts the conversation if follow-up mode is on - You would put your actual entity ID's here:

```yaml
- alias: Voice assistant follow-up
  trigger:
    - platform: state
      entity_id: assist_satellite.home_assistant_voice_XXXXXX_assist_satellite
      from: responding
      to: idle
    - platform: state
      entity_id: assist_satellite.home_assistant_voice_XXXXXY_assist_satellite
      from: responding
      to: idle
  condition:
    - condition: state
      entity_id: input_boolean.follow_up_mode
      state: 'on'
  action:
    - service: conversation.process
      data:
        text: Continue the conversation.
        conversation_id: follow_up_conversation
      response_variable: context_response
    - service: assist_satellite.start_conversation
      data:
        start_message: ''
        preannounce: false
      target:
        entity_id: '{{ trigger.entity_id }}'
  mode: single
```

> Optional: For richer context  - i.e. storing a recent action into short term memory before asking for a follow up  (e.g., "YouTube is now playing"), see **Persistent Memory** 

## 4. Timeout: Auto-End Follow-Up Mode After 5 Seconds

This automation disables follow-up mode if the assistant has been listening without a response - set to a 5 second cooldown. Change as needed:

```yaml
- alias: Simple Follow-Up Timeout
  trigger:
    - platform: state
      entity_id: assist_satellite.home_assistant_voice_XXXXXXX_assist_satellite
      to: listening
    - platform: state
      entity_id: assist_satellite.home_assistant_voice_XXXXXXY_assist_satellite
      to: listening
  condition:
    - condition: state
      entity_id: input_boolean.follow_up_mode
      state: 'on'
  action:
    - delay: 00:00:05
    - condition: template
      value_template: >
        {% set sat_id = trigger.entity_id %}
        {{ states(sat_id) == 'listening' and states('input_boolean.follow_up_mode') == 'on' }}
    - service: input_boolean.turn_off
      target:
        entity_id: input_boolean.follow_up_mode
    - service: assist_satellite.announce
      target:
        entity_id: '{{ trigger.entity_id }}'
      data:
        message: ''
        preannounce: false
  mode: restart
```

## 5. Stop Follow-Up Mode with Voice

Allow the user to vocally cancel follow-up mode:

```yaml
- alias: 'Voice: Stop Follow Up Mode'
  trigger:
    - platform: conversation
      command:
        - stop follow up
        - stop follow up mode
        - stop listening
        - stop talking
        - end conversation
        - conversation over
        - disregard
        - nevermind
        - cancel conversation
        - enough
        - that's enough
        - that is all
        - nothing else
        - no I am good
        - no I'm good
        - that's all
        - no that's all
        - no that is all
        - no that's it
        - no that is it
  action:
    - service: input_boolean.turn_off
      target:
        entity_id: input_boolean.follow_up_mode
    - service: persistent_notification.create
      data:
        title: Follow Up Mode
        message: Follow up mode has been disabled.
    - set_conversation_response: Sounds good. Let me know if you need anything further.
  mode: single
```

## 6. Start Follow-Up Mode with Voice (Optional)

You can also let users start follow-up mode manually - Follow up continually on puts the speakers in a state of always listening:

```yaml
- alias: 'Voice: Start Follow Up Mode'
  trigger:
    - platform: conversation
      command:
        - start follow up
        - start conversation
        - let's chat
        - lets chat
        - resume follow up
        - enable follow up
        - turn on follow up mode
        - turn follow up mode back on
  action:
    - service: input_boolean.turn_on
      target:
        entity_id: input_boolean.follow_up_mode
    - service: persistent_notification.create
      data:
        title: Follow Up Mode
        message: Follow up mode has been enabled.
  mode: single
```

> ⚠️ **Note on Always-On-Follow-Up-Mode**  
> The current design of the voice preview *erases* context whenever a new conversion is generated when forcing a listening state. Since we are essentially forcing new conversations between commands this means that with follow-up *always* on it is not possible to have a full conversation with persistent context. It basically will be in a state of constantly forgetting (without the short or long term memory module storing it's actions or memories) I would suggest keeping follow-up mode on only for the command daisy-chaining with the 5 second cooldown enabled.


## 🔁 Follow-Up Mode Logic Flow

```markdown
1. **User command is received**
   - → Assistant performs the action
   - → Assistant **asks a follow-up question** with `set_conversation_response`
   - → Automation ends by flipping `follow_up_mode` off → on
   - → Assistant enters "listening" state

2. **Assistant finishes responding**
   - → If `follow_up_mode` is on:
     - → Trigger automation to call `conversation.process`
     - → Immediately call `start_conversation` to capture the follow-up utterance

3. **5-Second Timeout**
   - → If assistant is listening and no voice response
   - → `follow_up_mode` is turned off
   - → Optional silent `announce` cleans up session

4. **User can say:**
   - “Nevermind” / “Cancel that” → turns off `follow_up_mode`
   - “Let’s chat” / “Resume follow up” → turns it back on

5. **Startup behavior**
   - → On HA restart, `follow_up_mode` is explicitly set to `off` for a clean state
```

## Summary

- Use `follow_up_mode` to track active listening sessions
- Automations continue the conversation when Assist becomes idle
- Timeout disables it after 5 seconds of no input
- Voice commands allow users to manually toggle follow-up mode

This setup enables natural-feeling voice command chaining without requiring custom components or media handling. 

