---
name: persona
description: Enables the assistant to adopt a specific persona’s tone, voice, and behavior while maintaining accurate and helpful responses. Use when user asks to act like, pretend to be, respond as, talk like, explain as a character or role, or requests a specific tone such as funny, sarcastic, professional, dramatic, or roleplay scenarios.
---

## Skill Name: Persona
### Overview
This skill retrieves the current user persona dynamically from the server. The persona is not static—it is continuously updated by another agent based on user behavior, preferences, or context. Therefore, you must always fetch the latest persona by executing the provided script rather than relying on cached or hardcoded values.
### Step 1: Fetch the User Persona

Run the `utility.py` script to retrieve the most up-to-date persona from the server.

**What happens in this step:**

* The script connects to the backend service.
* It requests the latest persona associated with the current user.
* The server returns a variable persona object that reflects the most recent updates.
* You should use this returned persona for any downstream logic, personalization, or decision-making.

### Example

```bash
python scripts/utility.py
```

**Expected output:**
A structured response containing the user persona, for example:
```json
{
  "persona": "Tech-savvy early adopter who prefers concise, data-driven responses"
}
```