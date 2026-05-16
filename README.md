# from-zero-to-genlayerFrom Zero to GenLayer: Build an AI-Powered Job Marketplace in 4 Steps
> **Mission:** GenLayer Introductory Tutorial  
> **Track:** Educational Content  
> **Difficulty:** Beginner-friendly (Python knowledge helpful but not required)  
> **Time:** ~60 minutes  
> **What you'll build:** AgentWork — a decentralized job marketplace where AI validators autonomously evaluate submitted work using GenLayer's Optimistic Democracy consensus
---
What You Will Learn
By the end of this tutorial, you will understand:
✅ What GenLayer's Optimistic Democracy Consensus is and why it matters
✅ What the Equivalence Principle is and how to use it in contracts
✅ How to use GenLayer Studio to write, deploy, and test contracts
✅ How to write a Python Intelligent Contract from scratch
✅ How to build a frontend that talks to your contract using `genlayer-js`
---
The Big Idea: Why GenLayer?
Traditional blockchains like Ethereum are powerful — but they have one fatal flaw: they can only understand rigid code. They cannot:
Read a webpage and reason about it
Evaluate whether a piece of writing is good
Understand natural language instructions
Make subjective decisions
GenLayer solves all of this. It lets you write Python smart contracts that can:
Call LLMs (like GPT or Claude) directly inside contract logic
Fetch live data from the web
Make subjective, AI-powered decisions — verified by decentralized consensus
Let's build something that shows all of this in action.
---
What We're Building: AgentWork
AgentWork is a decentralized job marketplace where:
Anyone posts a job with a description and budget
A worker submits their completed work
AI validators automatically evaluate the work against the job description
The verdict (approved/rejected) is stored on-chain — no human arbitrator needed
This is only possible on GenLayer. Let's build it.
---
Part 1: Core Concepts (Read This First!)
Before writing any code, let's understand the two most important ideas in GenLayer.
1.1 — Optimistic Democracy Consensus
In a traditional blockchain, every validator runs the exact same code and gets the exact same result. This works for math (`2 + 2 = 4`) but breaks for anything involving AI or the web (LLM responses vary between runs).
GenLayer solves this with Optimistic Democracy:
```
┌─────────────────────────────────────────────────────┐
│              TRANSACTION SUBMITTED                   │
└─────────────────────┬───────────────────────────────┘
                      │
          ┌───────────▼───────────┐
          │   LEADER VALIDATOR    │  ← Runs the contract, proposes result
          │   (randomly selected) │
          └───────────┬───────────┘
                      │ proposes result
          ┌───────────▼───────────┐
          │  VALIDATOR POOL (4+)  │  ← Each runs independently with own LLM
          │  V1  V2  V3  V4  V5  │
          └───────────┬───────────┘
                      │ majority agree?
              ┌───────┴────────┐
              │                │
           YES ▼            NO ▼
        COMMITTED          APPEAL
        ON-CHAIN           ROUND
```
Key insight: Multiple validators each run the LLM independently. If most agree on the outcome, it's committed. If not, an appeal round with more validators kicks in. This is how subjective AI decisions become trustless.
1.2 — The Equivalence Principle
Here's a challenge: if you ask two LLMs "Is this work good?", they might respond differently even if they agree. One says "Yes, approved." Another says "This submission meets the requirements." They mean the same thing — but they're different strings.
The Equivalence Principle tells GenLayer how to compare validator outputs:
Method	When to Use	Example
`gl.eq_principle_strict_eq`	Boolean or exact outputs	`True`/`False`
`gl.eq_principle_prompt_comparative`	Natural language where meaning matters	"approved" vs "looks good"
In our contract, we use `gl.eq_principle_prompt_comparative` because LLMs might word their verdict differently, but mean the same thing.
---
Part 2: Setting Up GenLayer Studio
GenLayer Studio is a browser-based IDE — no installation needed.
Step 1 — Open the Studio
👉 Go to studio.genlayer.com
You'll see:
A code editor on the left
A deploy/interact panel on the right
Node logs at the bottom
Step 2 — Understand the Interface
```
┌──────────────────────────────────────────────────────────────┐
│  GenLayer Studio                                             │
├─────────────────────────┬────────────────────────────────────┤
│                         │                                    │
│   CONTRACT EDITOR       │   DEPLOY & INTERACT                │
│                         │                                    │
│   # Write your Python   │   [ Deploy Contract ]              │
│   # contract here       │                                    │
│                         │   Constructor Params:              │
│                         │   • (none for this contract)       │
│                         │                                    │
│                         │   Methods:                         │
│                         │   • post_job()                     │
│                         │   • submit_work()                  │
│                         │   • evaluate_work()  ← AI magic!  │
│                         │   • get_job()                      │
├─────────────────────────┴────────────────────────────────────┤
│  NODE LOGS — Watch validators reach consensus in real time   │
└──────────────────────────────────────────────────────────────┘
```
Step 3 (Optional) — Local Setup
If you prefer working locally:
```bash
# Prerequisites: Node.js 18+, Docker 26+
npm install -g genlayer
genlayer init        # Select your LLM provider (OpenAI, Anthropic, etc.)
genlayer up          # Opens at http://localhost:8080
```
---
Part 3: Writing the Intelligent Contract
Now let's write the contract. Create a new file in Studio called `agent_job_marketplace.py`.
Step 1 — The Boilerplate
Every GenLayer contract starts with this structure:
```python
# { "Depends": "py-genlayer:test" }   ← Declares the GenVM SDK dependency

from genlayer import *                 ← Imports all GenLayer primitives

class AgentJobMarketplace(gl.Contract): ← Extend gl.Contract
    # State variables go here
    # Methods go here
```
The `# { "Depends": ... }` comment at the top is not optional — it tells the GenVM which SDK version to load.
Step 2 — State Variables
State variables are stored on-chain between transactions. GenLayer uses strong typing:
```python
class AgentJobMarketplace(gl.Contract):
    jobs: TreeMap[u256, DynArray[str]]
    # TreeMap = on-chain key-value store
    # u256 = unsigned 256-bit integer (job ID)
    # DynArray[str] = dynamic array of strings (job data)

    job_count: u256
    evaluations: TreeMap[u256, str]

    def __init__(self):
        self.job_count = u256(0)
        self.jobs = TreeMap[u256, DynArray[str]]()
        self.evaluations = TreeMap[u256, str]()
```
Step 3 — Write Methods (State-Changing)
Write methods modify on-chain state. They're decorated with `@gl.public.write`:
```python
@gl.public.write
def post_job(self, title: str, description: str, budget: str) -> None:
    """Anyone can post a job — human or AI agent."""
    job_id = self.job_count
    job_data: DynArray[str] = DynArray[str]()
    job_data.append(title)          # Index 0: title
    job_data.append(description)    # Index 1: description
    job_data.append(budget)         # Index 2: budget
    job_data.append("open")         # Index 3: status
    job_data.append(gl.message.sender_account)  # Index 4: poster address
    job_data.append("")             # Index 5: worker address
    job_data.append("")             # Index 6: submitted work

    self.jobs[job_id] = job_data
    self.job_count = u256(int(self.job_count) + 1)
```
```python
@gl.public.write
def submit_work(self, job_id: u256, work_result: str) -> None:
    """Worker submits completed work."""
    assert job_id < self.job_count, "Job does not exist"
    job = self.jobs[job_id]
    assert job[3] == "open", "Job is not open"

    job[3] = "submitted"
    job[5] = gl.message.sender_account
    job[6] = work_result
    self.jobs[job_id] = job
```
Step 4 — The AI Evaluation Method (The Magic Part!)
This is where GenLayer's power comes in. Study this carefully:
```python
@gl.public.write
def evaluate_work(self, job_id: u256) -> None:
    """
    Uses LLM consensus to autonomously evaluate submitted work.
    This is non-deterministic — different validators will get slightly 
    different LLM responses, but the Equivalence Principle reconciles them.
    """
    assert job_id < self.job_count, "Job does not exist"
    job = self.jobs[job_id]
    assert job[3] == "submitted", "No work submitted yet"

    job_title = job[0]
    job_description = job[1]
    submitted_work = job[6]

    # ⚠️ IMPORTANT: Non-deterministic code MUST live inside a function
    # that is passed to an eq_principle_* method. This is mandatory.
    def ai_evaluate() -> str:
        prompt = f"""
You are an autonomous AI evaluator for a decentralized job marketplace.

JOB TITLE: {job_title}
JOB DESCRIPTION: {job_description}
SUBMITTED WORK: {submitted_work}

Evaluate if the submitted work satisfactorily fulfills the job description.

Respond ONLY with a JSON object in this exact format:
{{
  "verdict": "approved" or "rejected",
  "reason": "one sentence explaining your decision",
  "score": a number from 1 to 10
}}
"""
        result = gl.exec_prompt(prompt)   # ← Calls the LLM
        return result

    # The Equivalence Principle: tells validators how to compare results
    # "approved"/"rejected" must match; scores within 2 points are equivalent
    evaluation_result = gl.eq_principle_prompt_comparative(
        ai_evaluate,
        "The verdict field must be exactly 'approved' or 'rejected'. "
        "Scores within 2 points are equivalent."
    )

    self.evaluations[job_id] = evaluation_result

    # Update job status based on AI verdict
    if "approved" in evaluation_result.lower():
        job[3] = "approved"
    else:
        job[3] = "rejected"

    self.jobs[job_id] = job
```
What's happening here:
`ai_evaluate()` is a regular Python function that calls `gl.exec_prompt()` to query an LLM
This function is passed to `gl.eq_principle_prompt_comparative()` — NOT called directly
Each validator calls this function independently with their own LLM
The equivalence principle defined in the second argument tells validators when to consider results "the same"
Step 5 — View Methods (Read-Only)
View methods don't change state and don't require gas. They're decorated with `@gl.public.view`:
```python
@gl.public.view
def get_job(self, job_id: u256) -> DynArray[str]:
    assert job_id < self.job_count, "Job does not exist"
    return self.jobs[job_id]

@gl.public.view
def get_evaluation(self, job_id: u256) -> str:
    if job_id in self.evaluations:
        return self.evaluations[job_id]
    return "No evaluation yet"

@gl.public.view
def get_total_jobs(self) -> u256:
    return self.job_count

@gl.public.view
def get_job_status(self, job_id: u256) -> str:
    assert job_id < self.job_count, "Job does not exist"
    return self.jobs[job_id][3]
```
The Full Contract
Here is the complete contract in one place:
```python
# { "Depends": "py-genlayer:test" }

from genlayer import *


class AgentJobMarketplace(gl.Contract):
    jobs: TreeMap[u256, DynArray[str]]
    job_count: u256
    evaluations: TreeMap[u256, str]

    def __init__(self):
        self.job_count = u256(0)
        self.jobs = TreeMap[u256, DynArray[str]]()
        self.evaluations = TreeMap[u256, str]()

    @gl.public.write
    def post_job(self, title: str, description: str, budget: str) -> None:
        job_id = self.job_count
        job_data: DynArray[str] = DynArray[str]()
        job_data.append(title)
        job_data.append(description)
        job_data.append(budget)
        job_data.append("open")
        job_data.append(gl.message.sender_account)
        job_data.append("")
        job_data.append("")
        self.jobs[job_id] = job_data
        self.job_count = u256(int(self.job_count) + 1)

    @gl.public.write
    def submit_work(self, job_id: u256, work_result: str) -> None:
        assert job_id < self.job_count, "Job does not exist"
        job = self.jobs[job_id]
        assert job[3] == "open", "Job is not open"
        job[3] = "submitted"
        job[5] = gl.message.sender_account
        job[6] = work_result
        self.jobs[job_id] = job

    @gl.public.write
    def evaluate_work(self, job_id: u256) -> None:
        assert job_id < self.job_count, "Job does not exist"
        job = self.jobs[job_id]
        assert job[3] == "submitted", "No work submitted yet"

        job_title = job[0]
        job_description = job[1]
        submitted_work = job[6]

        def ai_evaluate() -> str:
            prompt = f"""
You are an autonomous AI evaluator for a decentralized job marketplace.
JOB TITLE: {job_title}
JOB DESCRIPTION: {job_description}
SUBMITTED WORK: {submitted_work}
Evaluate if the submitted work satisfactorily fulfills the job description.
Respond ONLY with JSON: {{"verdict": "approved"/"rejected", "reason": "...", "score": 1-10}}
"""
            return gl.exec_prompt(prompt)

        evaluation_result = gl.eq_principle_prompt_comparative(
            ai_evaluate,
            "verdict must be 'approved' or 'rejected'. Scores within 2 points are equivalent."
        )

        self.evaluations[job_id] = evaluation_result
        job[3] = "approved" if "approved" in evaluation_result.lower() else "rejected"
        self.jobs[job_id] = job

    @gl.public.view
    def get_job(self, job_id: u256) -> DynArray[str]:
        assert job_id < self.job_count, "Job does not exist"
        return self.jobs[job_id]

    @gl.public.view
    def get_evaluation(self, job_id: u256) -> str:
        if job_id in self.evaluations:
            return self.evaluations[job_id]
        return "No evaluation yet"

    @gl.public.view
    def get_total_jobs(self) -> u256:
        return self.job_count

    @gl.public.view
    def get_job_status(self, job_id: u256) -> str:
        assert job_id < self.job_count, "Job does not exist"
        return self.jobs[job_id][3]
```
---
Part 4: Deploy & Test in Studio
Deploy the Contract
Paste the full contract into Studio
Click Deploy
No constructor parameters needed
Copy the contract address shown after deployment
Test It Step by Step
Test 1 — Post a Job
```
Method: post_job
title: "Write a Python function"
description: "Write a Python function that takes a list of numbers and returns the sum"
budget: "5"
```
Test 2 — Submit Work
```
Method: submit_work
job_id: 0
work_result: "def sum_numbers(lst): return sum(lst)"
```
Test 3 — Trigger AI Evaluation
```
Method: evaluate_work
job_id: 0
```
Watch the Node Logs at the bottom — you'll see each validator running the LLM independently, then reaching consensus. This is Optimistic Democracy in action!
Test 4 — Read the Result
```
Method: get_evaluation
job_id: 0
```
You should see a JSON verdict like:
```json
{
  "verdict": "approved",
  "reason": "The function correctly returns the sum of a list of numbers.",
  "score": 9
}
```
---
Part 5: Building the Frontend with genlayer-js
Now let's build a simple frontend that talks to your deployed contract.
Setup
```bash
# Clone the boilerplate
git clone https://github.com/genlayerlabs/genlayer-project-boilerplate
cd genlayer-project-boilerplate
npm install
```
Connect to Your Contract
Create `src/contract.js`:
```javascript
import { createClient, simulator } from "@genlayer/js";

// Connect to the GenLayer simulator
const client = createClient(simulator());

// Your deployed contract address (copy from Studio after deploying)
const CONTRACT_ADDRESS = "0xYourContractAddressHere";

// Post a new job
export async function postJob(title, description, budget) {
  const result = await client.writeContract({
    address: CONTRACT_ADDRESS,
    functionName: "post_job",
    args: [title, description, budget],
  });
  return result;
}

// Submit work for a job
export async function submitWork(jobId, workResult) {
  const result = await client.writeContract({
    address: CONTRACT_ADDRESS,
    functionName: "submit_work",
    args: [jobId, workResult],
  });
  return result;
}

// Trigger AI evaluation
export async function evaluateWork(jobId) {
  const result = await client.writeContract({
    address: CONTRACT_ADDRESS,
    functionName: "evaluate_work",
    args: [jobId],
  });
  return result;
}

// Read job details
export async function getJob(jobId) {
  const result = await client.readContract({
    address: CONTRACT_ADDRESS,
    functionName: "get_job",
    args: [jobId],
  });
  return result;
}

// Read AI evaluation result
export async function getEvaluation(jobId) {
  const result = await client.readContract({
    address: CONTRACT_ADDRESS,
    functionName: "get_evaluation",
    args: [jobId],
  });
  return result;
}
```
A Minimal Frontend (React)
Create `src/App.jsx`:
```jsx
import { useState } from "react";
import { postJob, submitWork, evaluateWork, getJob, getEvaluation } from "./contract";

export default function App() {
  const [jobTitle, setJobTitle] = useState("");
  const [jobDesc, setJobDesc] = useState("");
  const [jobId, setJobId] = useState(0);
  const [workResult, setWorkResult] = useState("");
  const [evaluation, setEvaluation] = useState("");
  const [status, setStatus] = useState("");

  async function handlePostJob() {
    setStatus("Posting job...");
    await postJob(jobTitle, jobDesc, "5");
    setStatus("✅ Job posted! Job ID: 0");
  }

  async function handleSubmitWork() {
    setStatus("Submitting work...");
    await submitWork(jobId, workResult);
    setStatus("✅ Work submitted!");
  }

  async function handleEvaluate() {
    setStatus("⚙️ AI validators evaluating... (this takes ~30s)");
    await evaluateWork(jobId);
    const result = await getEvaluation(jobId);
    setEvaluation(result);
    setStatus("✅ Evaluation complete!");
  }

  return (
    <div style={{ maxWidth: 600, margin: "40px auto", fontFamily: "monospace" }}>
      <h1>🤖 AgentWork</h1>
      <p>AI-Powered Job Marketplace on GenLayer</p>

      <h2>1. Post a Job</h2>
      <input placeholder="Job title" value={jobTitle} onChange={e => setJobTitle(e.target.value)} />
      <textarea placeholder="Job description" value={jobDesc} onChange={e => setJobDesc(e.target.value)} />
      <button onClick={handlePostJob}>Post Job</button>

      <h2>2. Submit Work</h2>
      <input type="number" placeholder="Job ID" value={jobId} onChange={e => setJobId(Number(e.target.value))} />
      <textarea placeholder="Your work result" value={workResult} onChange={e => setWorkResult(e.target.value)} />
      <button onClick={handleSubmitWork}>Submit Work</button>

      <h2>3. Trigger AI Evaluation</h2>
      <button onClick={handleEvaluate}>Evaluate Work (AI Consensus)</button>

      {evaluation && (
        <div>
          <h2>4. AI Verdict</h2>
          <pre>{evaluation}</pre>
        </div>
      )}

      {status && <p><strong>Status:</strong> {status}</p>}
    </div>
  );
}
```
Run the Frontend
```bash
npm run dev
# Opens at http://localhost:5173
```
---
What You've Learned
Congratulations! You've just built a full GenLayer dApp. Here's a recap:
Concept	What You Learned
Optimistic Democracy	Multiple AI validators reach consensus independently
Equivalence Principle	How validators compare non-deterministic LLM outputs
Intelligent Contracts	Python contracts that call LLMs and fetch web data
GenLayer Studio	Browser-based IDE for writing, deploying, testing
genlayer-js	JavaScript SDK for frontend-to-contract interaction
---
What to Build Next
Now that you know the basics, here are some ideas to extend AgentWork:
Escrow payments — Lock GEN tokens on job post, release on approval
Worker reputation — Track approval rate per address on-chain
Multi-round appeals — Let workers appeal a rejected verdict
Web-verified jobs — Use `gl.get_webpage()` to verify external deliverables
---
Resources
Resource	Link
GenLayer Docs	docs.genlayer.com
GenLayer Studio	studio.genlayer.com
Boilerplate Repo	github.com/genlayerlabs/genlayer-project-boilerplate
genlayer-js Docs	docs.genlayer.com/developers/genlayer-js
Discord	discord.gg/genlayer
---
Tutorial by a GenLayer Builder — Testnet Bradbury Hackathon participant  
Built during the GenLayer Builders Program
