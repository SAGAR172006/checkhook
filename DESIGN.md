1. Layout Structure
The website is a clean, single-page application focused entirely within the hero section.


Hero Section:


App Title: Checkhook

Form: Contains the Webhook URL input, Message input, and the Submit button.

Result Panel: Appears directly within the form area upon receiving a response.

Footer/Credits: Displays project credits alongside a clickable LinkedIn logo icon directly below the form container.
2. Color Palette (Light Theme: Creme & Blue)
Token             Hex Value     Usage
--bg-primary      #FDFBF7       Page background (Light Creme)
--bg-card         #FFFFFF       Main form background (White)
--bg-input        #F8FAFC       Input field backgrounds (Very light slate/blue)
--text-primary    #1E293B       Main text (Dark Blue/Slate)
--text-muted      #64748B       Labels, placeholders (Muted Slate)
--accent          #2563EB       Buttons, active states (Primary Blue)
--accent-hover    #1D4ED8       Button hover state (Deep Blue)
--border          #E2E8F0       Input borders, dividers
3. Component Specifications
3.1 Webhook URL Input


Type: <input type="url">

Label: Webhook URL

Placeholder: https://your-n8n.com/webhook/...
3.2 Message Input


Type: <textarea> or <input type="text">

Label: Message

Placeholder: Enter your payload message here
3.3 Submit Button


Label: Test Webhook

Design: Uses --accent (Blue) background with white text.
3.4 Credits Section


Located directly under the main card/form.

Content: A brief credit line (e.g., "Built by [Your Name]") next to a standalone LinkedIn Logo (SVG/Icon).

Interaction: The logo serves as a hyperlink <a href="..."> that redirects to your LinkedIn profile when clicked. The icon color should match --text-muted and transition to --accent on hover.
4. Application Logic & Result State


Action: The user clicks "Test Webhook".

Condition: If a response is successfully received back from the exact webhook URL provided in the input.

Success Display: The UI will display the following exact message:
"n8n workflow is working with webhook."
5. Wireframe
Plaintext
┌─────────────────────────────────────────────┐
│ [Background: Creme] │
│ │
│ ┌───────────────────────────────────┐ │
│ │ Checkhook │ │
│ ├───────────────────────────────────┤ │
│ │ ┌─────────────────────────────┐ │ │
│ │ │ Webhook URL input │ │ │
│ │ └─────────────────────────────┘ │ │
│ │ ┌─────────────────────────────┐ │ │
│ │ │ Message │ │ │
│ │ └─────────────────────────────┘ │ │
│ │ │ │
│ │ [ Test Webhook (Blue Button) ] │ │
│ ├───────────────────────────────────┤ │
│ │ RESULT: │ │
│ │ "n8n workflow is working with │ │
│ │ webhook." │ │
│ └───────────────────────────────────┘ │
│ │
│ Credits • [in] │
│ │
└─────────────────────────────────────────────┘

(Note in wireframe: [in] represents the clickable LinkedIn logo SVG).