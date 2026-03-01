# CLAUDE.md: OpenCode & Local GLM-4 Tool-Use Fix & Air-Gapped Deployment

## 1. Context & Objective
- **Software:** OpenCode (Open-source AI Coding Agent from opencode.ai).
- **Setup:** Air-gapped environment simulation on Colab A100.
- **Backend:** Local vLLM serving `THUDM/glm-4-9b-chat` at `http://localhost:8000/v1`.
- **Primary Goal:** Fix "File Writing" (Tool Use) failure by identifying and bypassing external API triggers.
- **Secondary Goal:** Generate comprehensive deployment guides for two scenarios:
  1. Transferring an NPM-installed version from an open network to a closed network.
  2. Configuring the standalone Release Binary in a closed network.

## 2. Phase 1: Diagnosis & Fix (A100 Environment)
1. **Initialize vLLM:** Run GLM-4-9B-Chat on port 8000.
2. **Reproduce Failure:** Configure OpenCode to use `glm-4` and trigger a "Write a file" task.
3. **Analyze & Bypass:**
   - Detect hidden external calls (strace/tcpdump).
   - Test "Identity Masking" (serving as `gpt-4` to bypass GLM-specific logic).
   - Identify if `opencode.json` or environment variables can override the behavior.

## 3. Phase 2: Generate Deployment Guides
After finding the solution, you MUST generate and save two separate guides in the current directory:

### Task A: `CLOSED_NETWORK_GUIDE_NPM.md`
This guide is for users who install OpenCode via `npm` on an open network and move it to a closed network.
- **Packaging:** How to bundle `node_modules` and the `opencode` executable (using `npm pack` or directory zipping).
- **Environment Prep:** Required Node.js version and dependencies.
- **Configuration:** Steps to modify the config files (e.g., `opencode.json`) to point to the local vLLM.
- **Fix Implementation:** Instructions on how to apply the "Tool Use Fix" discovered in Phase 1 (e.g., changing the model name or using a local proxy).

### Task B: `CLOSED_NETWORK_GUIDE_BINARY.md`
This guide is for users using the pre-compiled Release Binary directly in a closed network.
- **Installation:** Where to place the binary and how to set execution permissions.
- **Setup:** Initializing the configuration without internet access.
- **Configuration:** Detailed JSON mapping for `opencode.json` to ensure 100% local operation.
- **Fix Implementation:** Specific steps to solve the "File Writing" issue (e.g., env vars or identity spoofing).

## 4. Quality Standards for Guides
- Must be **Step-by-Step** and easy to follow for IT admins.
- Must include **Troubleshooting** section (specifically for "Tool Use" errors).
- Must include **Local Endpoint Validation** (how to check if OpenCode is talking to vLLM correctly).

---
**Claude, begin Phase 1 now. Once the fix is confirmed, proceed to generate the two .md guides.**
