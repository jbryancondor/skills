How to fill the report section by section. You will find instrucction inside this character {{ instructions }}.

Squad Report
- Owner: {{ This is the notion used logger. Example: @Bryan Condor}}

# Relevant Incidents from the last week
- {{ Link to Incidents that happened during the on-call week that belongs to "Merchant Platform". If there are no incidents, write "None"}}

# Relevant remediation actions update
- {{ Link to remediation actions P1 that were taken during the on-call week that belongs to "Merchant Platform". If there are no remediation actions, write "None". Find in Rememdiation Action Asana Board }}

# Alert Groups

Pending cases to resolve (to hand over to the next on-call engineer):
- {{ Link to pending cases that were not resolved during the on-call week they are tasked assigned to asana user logged and you could find there in "L3 On Call" parent task and also in "L3 OnCall Escalations - Merchant Channels & Activation" board. }}

Most frequent alerts triggered
High Severity
{{ 
    1. Extract from the image attached all alerts and ask about what of check we should include in "Deep Dive Relevant Alerts" section 
    2. Once the user selected the alerts. You need to extract the solutions and context of those alerts you must find thet information in #merchant-platform-alerts slack channel.
    3. For each alert put the context and solution in the "Deep Dive Relevant Alerts" section.
}}

Low Severity
{{ 
    1. Extract from the image attached all alerts and ask about what of check we should include in "Deep Dive Relevant Alerts" section 
    2. Once the user selected the alerts. You need to extract the solutions and context of those alerts you must find thet information in #merchant-platform-alerts slack channel.
    3. For each alert put the context and solution in the "Deep Dive Relevant Alerts" section.
}}

Deep Dive Relevant Alerts
{{
    Here you include every alert with its context and solution.
}}

# Bugs Backlog
{{ Just include a callout to say the user replace here the screenshot of the bugs backlog}}

Accepted bugs update
- {{ Find in "Merchant Platform Bugs" and "Merchant SMB Bugs" asana boards the bugs that were accepted/created during the on-call week. or it has an relevant updated such it has been completed }}

Outstanding Postmortems
- {{ Ask to user if there are postmortem if so they need to provide the link of the Postmorten normally they are stored in https://www.notion.so/addico/445ce72c5db84b228b4fd1754843a6f9?v=313e8c84a8324b09b78e18dd728483fa So if they provide the link you could extract a brief summary to include additiona to the link.}}

Significant Events
- {{ Ask to user if there are significant events }}

Actionable items
- {{ Here include all tasks created and completed by the user logged in asana. Only the link. You should find those task in parent task "L3 On Call" and "L3 OnCall Escalations - Merchant Channels & Activation"}}


