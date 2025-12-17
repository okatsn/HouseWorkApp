# AGENTS

## Goal

You are building a "Full Stack" application where the frontend (Observable) runs in the browser but is served by a container, and the backend (Julia) runs purely as an API service.

Here is an implementation plan, including the **Project Structure**, the **Code**, and **Debugging Toolbox**.

The app is split into two separate containers managed by `docker-compose`.

1. **Frontend Container:** A web server that serves the UI.
2. **Worker Container:** A Julia environment waiting for commands.

## Layout, data and flow

### The Flow

0. User select (click on) tasks in the `TODO-Now` and `TODO-Upcoming`.
1. User clicks "Submit" on Frontend.
2. Frontend sends an HTTP request (or puts a message in a queue like Redis) to the Worker.
3. Worker (Julia) receives the signal, runs the task, updates the git repo, and responds "Done".

### The Dashboard layout

- Top row: Tasks of TODO-Now
- middle row: Tasks of TODO-Upcoming
- middle-right: A "submit" button.
- bottom row: A diagram with x axis the date time and a categorical y axis for all recurrent tasks, showing the visualization of scheduled tasks within a one-year scope.

### Data

`todo.yaml` columns:
- **TaskName:** E.g, "mop the floor", "clean toilet".
- **RecurrentPeriod:** The period when a task revives. E.g., 1 week for "clean toilet".
- **LeadingTime:** The time window within which the status changed to "upcoming" from "done". E.g., 2 days for "clean toilet"
- **Status:** "due" (to show on the TODO-Now panel), "upcoming" (to show on the TODO-Upcoming panel), and "done".

`done_log.csv` columns:
- **TaskName**
- **DoneAt:** When the task was done at `Date`.

### Explained with an example

For "clean toilet" that has a recurrent time of 1 week,
when the user select it on either TODO-Now or TODO-Upcoming panel and submit, the `todo.yaml` updates the field `clean_toilet.Status` from "due" or "upcoming" to "done", and note the time at `DoneAt` in `done_log.csv`.



## The Directory Structure

```text
/HouseWorkApp
├── docker-compose.yml          # The conductor
├── .env                        # Environment variables (optional)
├── dev_toolbox.sh              # <--- YOUR NEW DEBUGGING TOOLBOX
├── data/                       # Shared Data Volume
│   ├── todo.yaml
│   └── done_log.csv
├── worker/                     # Julia Backend
│   ├── Dockerfile
│   ├── Project.toml
│   └── src/
│       ├── server.jl
│       └── update_logic.jl
└── frontend/                   # Observable Frontend
    ├── Dockerfile
    ├── package.json
    ├── observable.config.js
    └── src/
        └── index.md            # The Dashboard UI

```


## The Docker Configuration

This `docker-compose.yml` is designed for **development**.
It maps your local folders into the containers so you can edit code without rebuilding.

A **draft** of `docker-compose.yml`:

```yaml
services:
  # 1. The Julia Worker (Backend)
  worker:
    build: ./worker
    container_name: housework-worker
    ports:
      - "8080:8080"  # Expose API to host (so browser can hit localhost:8080)
    volumes:
      - ./data:/data             # Shared data folder
      - ./worker/src:/app/src    # <--- LIVE CODING: Edits in VS Code update the container
    command: julia --project=/app/ -e 'using Pkg; Pkg.instantiate(); include("src/server.jl")'
    environment:
      - JULIA_NUM_THREADS=2

  # 2. The Observable Framework (Frontend)
  frontend:
    build: ./frontend
    container_name: housework-frontend
    ports:
      - "3000:3000"  # Expose UI to host
    volumes:
      - ./frontend/src:/app/src  # <--- LIVE EDITING: Edits to index.md update the UI
    # Run the dev server, binding to all interfaces so Docker can see it
    command: npm run dev -- --host 0.0.0.0

```








## System Design

### The Backend: Julia + Oxygen.jl

We need `Oxygen` (web framework) and `CSV/DataFrames` (logic).

**`worker/Dockerfile`**

```dockerfile
FROM julia:1.10
WORKDIR /app
# Install dependencies first (caching layer)
COPY Project.toml .
RUN julia --project=/app/ -e 'using Pkg; Pkg.add(["Oxygen", "HTTP", "CSV", "DataFrames", "JSON3"]); Pkg.instantiate()'
# Copy source (though docker-compose overrwrites this with volume)
COPY src/ src/

```

**`worker/src/server.jl`**

```julia
using Oxygen
using HTTP
using JSON3
using CSV
using DataFrames

# 1. CORS Setup (Crucial!)
# Since your Frontend runs on port 3000 and Backend on 8080,
# the browser will block requests unless we allow them.
const CORS_HEADERS = [
    "Access-Control-Allow-Origin" => "*",
    "Access-Control-Allow-Headers" => "*",
    "Access-Control-Allow-Methods" => "POST, GET, OPTIONS"
]

# Handle "Preflight" OPTIONS request automatically
@options "/submit" function(req)
    return HTTP.Response(200, CORS_HEADERS, body="")
end

# 2. The Submit Endpoint
@post "/submit" function(req::HTTP.Request)
    try
        data = JSON3.read(req.body)
        task_ids = data["task_ids"]

        println("Received request to complete tasks: ", task_ids)

        # --- YOUR LOGIC HERE ---
        # include("update_logic.jl")
        # update_3a(task_ids)
        # -----------------------

        # Respond with success and CORS headers
        return HTTP.Response(200, CORS_HEADERS, body=JSON3.write(Dict("status" => "success")))
    catch e
        println("Error: ", e)
        return HTTP.Response(500, CORS_HEADERS, body=JSON3.write(Dict("error" => string(e))))
    end
end

# 3. Start Server
# async=false keeps the script running
serve(host="0.0.0.0", port=8080, async=false)

```



### The Frontend: Observable Framework

**`frontend/Dockerfile`**

```dockerfile
FROM node:18-alpine
WORKDIR /app
# Install Observable Framework globally or locally
COPY package.json .
RUN npm install
COPY . .
# The command is handled in docker-compose

```

**`frontend/package.json`**

```json
{
  "name": "housework-frontend",
  "scripts": {
    "dev": "observable preview",
    "build": "observable build"
  },
  "dependencies": {
    "@observablehq/framework": "latest"
  }
}

```

**`frontend/src/index.md`** (Your Dashboard)
This uses a mix of Markdown and JavaScript.

```markdown
# My Housework Dashboard

<div class="grid grid-cols-2">
  <div class="card">
    <h2>TODO Now</h2>
    ${
      Inputs.checkbox(["Mop Floor", "Vacuum Bed", "Change Filter"], {label: "Select tasks to complete"})
    }
  </div>

  <div class="card">
    <h2>Actions</h2>
    <button id="submitBtn" style="background: #4CAF50; color: white; padding: 10px;">
      Submit Selected Tasks
    </button>
    <div id="statusMsg"></div>
  </div>
</div>

```js
// This JavaScript block handles the click
const btn = document.getElementById("submitBtn");
const status = document.getElementById("statusMsg");

btn.onclick = async () => {
  // Get checked boxes (Note: In real Observable, you'd bind the input above to a variable)
  // For simplicity here, we assume hardcoded for the demo or use Input binding standard

  status.innerText = "Submitting...";

  try {
    const response = await fetch("http://localhost:8080/submit", {
      method: "POST",
      body: JSON.stringify({ task_ids: ["Mop Floor"] }), // Replace with actual selection
      headers: { "Content-Type": "application/json" }
    });

    if(response.ok) {
      status.innerText = "Success! Backend updated.";
      // Trigger a refresh or data reload here
    } else {
      status.innerText = "Server Error.";
    }
  } catch (e) {
    status.innerText = "Network Error: " + e;
  }
};

```



### The Debugging Toolbox

Create a file named `dev_toolbox.sh` in your root folder. This script codifies the three techniques you wanted to learn.

Make it executable: `chmod +x dev_toolbox.sh`.

```bash
#!/bin/bash

# A simple menu for your Container learning workflow
echo "========================================"
echo "   HOUSEWORK APP - DEVELOPER TOOLBOX"
echo "========================================"
echo "1. [START]   Start system (Live Code Mode)"
echo "2. [SHELL]   Enter Worker Container (Debug inside)"
echo "3. [TEST]    Run Julia Tests (One-off)"
echo "4. [LOGS]    Watch Worker Logs"
echo "5. [CLEAN]   Stop and Remove Containers"
echo "----------------------------------------"
read -p "Select an option (1-5): " opt

case $opt in
  1)
    echo "Starting containers... Go to http://localhost:3000 for UI"
    # -d runs in background
    docker-compose up -d
    ;;

  2)
    # TECHNIQUE 3: Attach Shell
    echo "Entering the Worker container..."
    echo "Tip: You are now 'inside' the linux machine. Try running 'julia' manually."
    docker-compose exec worker /bin/bash
    ;;

  3)
    # TECHNIQUE 2: The Test Runner
    # We run a temporary command using the 'worker' service definition
    echo "Running tests..."
    docker-compose run --rm worker julia -e 'println("Running Tests..."); # include("test/runtests.jl")'
    ;;

  4)
    # Monitoring
    docker-compose logs -f worker
    ;;

  5)
    echo "Stopping..."
    docker-compose down
    ;;

  *)
    echo "Invalid option."
    ;;
esac

```


## How to use:

1. **Run `./dev_toolbox.sh` and pick Option 1.**
* Open `http://localhost:3000`. You see your dashboard.
* Open `http://localhost:8080/submit` (using Postman or curl). You see the Julia API.


2. **Live Debugging:**
* Keep the app running.
* Open `worker/src/server.jl` in your text editor.
* Change the print statement `println("Received request...")` to something else.
* *Note:* Since Julia doesn't auto-reload code by default like Python, you usually need to restart the julia process.
* **Challenge:** Update `server.jl` to use `Revise.jl` for hot-reloading, OR just restart the container: `docker-compose restart worker`.


3. **"Inside" Debugging:**
* Run `./dev_toolbox.sh` -> Option 2.
* You are now in the shell. Check if your data is there: `ls /data`.
* Manually run a specific script to see if it breaks: `julia src/update_3a.jl`. This is how you debug logic errors without the UI getting in the way.