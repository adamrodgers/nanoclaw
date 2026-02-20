---

name: add-hevy

description: Add HEVY workout tracker integration to NanoClaw. Access workout data, track personal records, analyze progression, and get weekly summaries. Requires HEVY Pro subscription.

---



\# Add HEVY Integration



This skill integrates HEVY workout tracker with NanoClaw, allowing you to query workout data, track personal records, analyze progression, and receive automated weekly summaries.



\## Prerequisites



\*\*HEVY Pro Subscription Required\*\*



The HEVY API is only available to HEVY Pro users. Before continuing, confirm:



1\. You have an active HEVY Pro subscription

2\. You can access https://hevy.com/settings?developer



If not, upgrade to HEVY Pro first.



---



\## Step 1: Get Your HEVY API Key



\*\*USER ACTION REQUIRED\*\*



Tell the user:



> I need your HEVY API key. Here's how to get it:

>

> 1. Open https://hevy.com/settings?developer in your browser

> 2. Sign in if needed

> 3. You should see an "API Key" section

> 4. Copy your API key

>

> Paste your API key here (it will be stored securely)



Wait for the user to provide their API key.



---



\## Step 2: Store the API Key



Once the user provides their API key, store it securely:



```bash

mkdir -p ~/.hevy-mcp

```



Create the environment file with the API key (replace `USER\_API\_KEY\_HERE` with the actual key they provided):



```bash

cat > ~/.hevy-mcp/.env << 'EOF'

HEVY\_API\_KEY=USER\_API\_KEY\_HERE

EOF

```



Verify it was created:



```bash

ls -la ~/.hevy-mcp/

cat ~/.hevy-mcp/.env

```



---



\## Step 3: Test HEVY MCP Connection



Test that the HEVY MCP server can connect with the API key:



```bash

HEVY\_API\_KEY=$(cat ~/.hevy-mcp/.env | grep HEVY\_API\_KEY | cut -d= -f2) npx -y hevy-mcp

```



This will start the MCP server. You should see it initialize without errors. Press Ctrl+C after a few seconds to stop it.



If you see authentication errors, the API key may be incorrect. Ask the user to verify it.



---



\## Step 4: Add HEVY MCP to Agent Runner



Read `container/agent-runner/src/index.ts` and locate the `mcpServers` configuration in the `query()` function call.



Add the `hevy` MCP server to the configuration:



```typescript

hevy: {

&nbsp; command: 'npx',

&nbsp; args: \['-y', 'hevy-mcp'],

&nbsp; env: {

&nbsp;   HEVY\_API\_KEY: process.env.HEVY\_API\_KEY || ''

&nbsp; }

}

```



Find the `allowedTools` array in the same file and add HEVY tools:



```typescript

'mcp\_\_hevy\_\_\*'

```



The result should look similar to:



```typescript

mcpServers: {

&nbsp; nanoclaw: ipcMcp,

&nbsp; hevy: {

&nbsp;   command: 'npx',

&nbsp;   args: \['-y', 'hevy-mcp'],

&nbsp;   env: {

&nbsp;     HEVY\_API\_KEY: process.env.HEVY\_API\_KEY || ''

&nbsp;   }

&nbsp; }

},

allowedTools: \[

&nbsp; 'Bash',

&nbsp; 'Read', 'Write', 'Edit', 'Glob', 'Grep',

&nbsp; 'WebSearch', 'WebFetch',

&nbsp; 'mcp\_\_nanoclaw\_\_\*',

&nbsp; 'mcp\_\_hevy\_\_\*'

],

```



---



\## Step 5: Mount HEVY Config in Container



Read `src/container-runner.ts` and find the `buildVolumeMounts` function.



Add this mount block (after the `.claude` mount or other credential mounts):



```typescript

// HEVY MCP config directory

const hevyDir = path.join(homeDir, '.hevy-mcp');

if (fs.existsSync(hevyDir)) {

&nbsp; mounts.push({

&nbsp;   hostPath: hevyDir,

&nbsp;   containerPath: '/home/node/.hevy-mcp',

&nbsp;   readonly: true  // API key is read-only

&nbsp; });

}

```



This ensures the container can access the HEVY API key.



---



\## Step 6: Update Group Memory



Update the global memory file to document HEVY capabilities.



Read `groups/global/CLAUDE.md` (or create it if it doesn't exist), then append:



```markdown



\## HEVY Workout Tracker



You have access to HEVY workout data via MCP tools. Available tools:



\### Workout Management

\- `mcp\_\_hevy\_\_get-workouts` - Fetch workouts between dates (max 10 per query, sorted by date descending)

\- `mcp\_\_hevy\_\_get-workout` - Get a single workout by ID

\- `mcp\_\_hevy\_\_create-workout` - Create a new workout

\- `mcp\_\_hevy\_\_update-workout` - Update an existing workout

\- `mcp\_\_hevy\_\_get-workout-count` - Get total workout count

\- `mcp\_\_hevy\_\_get-workout-events` - Track workout updates/deletions



\### Exercise \& Progress

\- `mcp\_\_hevy\_\_get-exercises` - Get exercises sorted by frequency (filter by name, date range)

\- `mcp\_\_hevy\_\_get-exercise-progress-by-ids` - Track progress for specific exercises over time

\- `mcp\_\_hevy\_\_get-exercise-templates` - Browse available exercise templates

\- `mcp\_\_hevy\_\_get-exercise-template` - Get specific exercise template by ID



\### Routines

\- `mcp\_\_hevy\_\_get-routines` - Get saved workout routines

\- `mcp\_\_hevy\_\_create-routine` - Create a new routine

\- `mcp\_\_hevy\_\_update-routine` - Update an existing routine

\- `mcp\_\_hevy\_\_get-routine-by-id` - Get specific routine by ID

\- `mcp\_\_hevy\_\_get-routine-folders` - Access routine organization folders

\- `mcp\_\_hevy\_\_create-routine-folder` - Create a new folder for organizing routines



\### Analysis Capabilities

When asked about workout progress, you can:

\- Identify personal records (PRs) by exercise

\- Calculate volume trends (sets × reps × weight)

\- Track progression over time

\- Analyze workout frequency by muscle group

\- Provide suggestions for progressive overload



\### Example Queries

\- "What workouts did I do this week?"

\- "Show me my PRs for bench press"

\- "What's my total volume trend for the past month?"

\- "How often have I trained legs in the last 30 days?"

\- "Give me a weekly workout summary"

```



Also update `groups/main/CLAUDE.md` with the same content.



---



\## Step 7: Rebuild Container and Restart Service



Rebuild the container to include the HEVY MCP integration:



```bash

cd /workspace/project/container \&\& ./build.sh

```



Wait for the build to complete (this may take a few minutes).



Once complete, compile the TypeScript:



```bash

cd /workspace/project \&\& npm run build

```



Then restart the NanoClaw service:



```bash

launchctl kickstart -k gui/$(id -u)/com.nanoclaw

```



Wait a couple seconds, then verify it started:



```bash

sleep 3 \&\& launchctl list | grep nanoclaw

```



Check the logs for any startup errors:



```bash

tail -20 /workspace/project/logs/nanoclaw.log

```



---



\## Step 8: Test HEVY Integration



Tell the user:



> HEVY integration is complete! Test it by sending this message in your WhatsApp:

>

> `@Andy how many workouts have I done?`

>

> Or try:

>

> `@Andy what exercises did I do this week?`



Monitor the logs to see the HEVY MCP tools being called:



```bash

tail -f /workspace/project/logs/nanoclaw.log | grep -i hevy

```



If you see HEVY tools being invoked successfully, the integration is working!



---



\## Step 9: Set Up Weekly Workout Summary (Optional)



Ask the user:



> Would you like me to set up an automated weekly workout summary?

>

> I can send you a summary every week with:

> - Total workouts completed

> - Volume trends

> - New personal records

> - Exercise frequency by muscle group

> - Suggestions for next week

>

> When would you like to receive it? (e.g., "Sunday at 8pm", "Friday at 6pm")



If they agree, create a scheduled task. For example, if they want it Sunday at 8pm:



```bash

\# This would be done via a message to @Andy in WhatsApp:

\# "@Andy schedule a weekly workout summary every Sunday at 8pm"

```



Or you can create it programmatically by telling the user to send:



`@Andy schedule a task to run every Sunday at 8pm with this prompt: "Generate a comprehensive weekly workout summary including: total workouts, volume trends, new PRs, muscle group frequency, and suggestions for next week's training"`



---



\## Advanced: Custom Analysis Prompts



You can now create sophisticated workout analysis. Here are some example prompts users can try:



\### Progressive Overload Tracking

```

@Andy analyze my bench press progression over the last 8 weeks and tell me if I'm applying progressive overload effectively

```



\### Volume Analysis

```

@Andy calculate my total training volume by muscle group for this month and compare it to last month

```



\### Recovery Patterns

```

@Andy how many rest days have I taken in the past 4 weeks? Am I recovering enough?

```



\### Personal Record Summary

```

@Andy list all the PRs I've hit in the past month

```



\### Exercise Variety

```

@Andy what exercises have I done most frequently? Am I overusing any movements?

```



---



\## Troubleshooting



\### HEVY MCP Not Responding



Test the MCP server directly:



```bash

HEVY\_API\_KEY=$(cat ~/.hevy-mcp/.env | grep HEVY\_API\_KEY | cut -d= -f2) npx -y hevy-mcp

```



If it fails to start, verify:

1\. API key is correct

2\. You have HEVY Pro subscription

3\. Network connectivity to api.hevyapp.com



\### API Key Issues



Re-verify your API key:



```bash

cat ~/.hevy-mcp/.env

```



If needed, update it:



```bash

nano ~/.hevy-mcp/.env

\# Edit HEVY\_API\_KEY=your\_new\_key

```



Then restart:



```bash

launchctl kickstart -k gui/$(id -u)/com.nanoclaw

```



\### Container Can't Access HEVY



Verify the mount was added correctly:



```bash

grep -A 5 "hevy-mcp" /workspace/project/src/container-runner.ts

```



Check container logs for HEVY-related errors:



```bash

ls -la /workspace/project/groups/main/logs/

tail -50 /workspace/project/groups/main/logs/container-\*.log | grep -i hevy

```



\### No Workout Data Returned



The HEVY API requires date ranges for most queries. Make sure your prompts include timeframes:



\- ✅ "workouts from this week"

\- ✅ "exercises in the last 7 days"

\- ❌ "all my workouts" (too broad, API has limits)



---



\## Removing HEVY Integration



To remove HEVY completely:



1\. \*\*Remove from agent runner:\*\*

&nbsp;  - Edit `container/agent-runner/src/index.ts`

&nbsp;  - Delete `hevy` from `mcpServers`

&nbsp;  - Remove `mcp\_\_hevy\_\_\*` from `allowedTools`



2\. \*\*Remove mount:\*\*

&nbsp;  - Edit `src/container-runner.ts`

&nbsp;  - Delete the `~/.hevy-mcp` mount block



3\. \*\*Remove from memory:\*\*

&nbsp;  - Edit `groups/global/CLAUDE.md` and `groups/main/CLAUDE.md`

&nbsp;  - Delete the "## HEVY Workout Tracker" section



4\. \*\*Rebuild:\*\*

&nbsp;  ```bash

&nbsp;  cd /workspace/project/container \&\& ./build.sh

&nbsp;  cd /workspace/project \&\& npm run build

&nbsp;  launchctl kickstart -k gui/$(id -u)/com.nanoclaw

&nbsp;  ```



5\. \*\*Optional - Remove API key:\*\*

&nbsp;  ```bash

&nbsp;  rm -rf ~/.hevy-mcp

&nbsp;  ```



---



\## Privacy \& Security



\- Your HEVY API key is stored locally in `~/.hevy-mcp/.env`

\- It's mounted read-only into the container

\- No workout data is stored by NanoClaw (all queries are live to HEVY API)

\- The API key never leaves your machine



\## References



\- HEVY MCP GitHub: https://github.com/chrisdoc/hevy-mcp

\- HEVY API Documentation: https://api.hevyapp.com/docs/

\- HEVY Developer Settings: https://hevy.com/settings?developer

