---
fileClass: Project
Category: Documentation
Status: Active
Authors: Mason Bendixen
Last Updated: 4/16/2026
Version: 0.1
tags: 
---
# Overview
Claude broke bracketed paste. The current version is:
```
PS C:\Users\mason> claude --version
2.1.111 (Claude Code)
```
The last version that worked is 2.1.104. This is how you you go back to that version:
```
npm install -g @anthropic-ai/claude-code@2.1.104
```
When I want to go back to the latest, I can do:
```
npm install -g @anthropic-ai/claude-code@2.1.104
```
Here is a prompt to copy into ChatGPT to periodically check to see if this issue has been fixed:

You are helping me track a known regression in Claude Code on Windows.

Context:

- Multiline paste / bracketed paste is broken in Claude Code starting in v2.1.105+
    
- It worked correctly in v2.1.104
    
- Symptoms include:
    
    - Only part of pasted text appearing
        
    - Missing lines or truncation
        
    - Paste behaving inconsistently across PowerShell and Git Bash
        

Tasks:

1. Search for recent Claude Code releases and changelog entries
    
2. Check GitHub issues for:
    
    - paste
        
    - bracketed paste
        
    - Windows terminal input issues
        
3. Determine whether this issue has been:
    
    - fixed
        
    - partially fixed
        
    - still open
        

Output:

- Latest Claude Code version
    
- Whether the paste issue is fixed
    
- If fixed: first version where it works again
    
- If not fixed: current status and any known workarounds
    

Be specific and reference concrete versions and issue reports.

# Related Documents
- Links to related documents

# System Overview
Is this client, server, or both. List the major components and how they work together.

# Setup Instructions
- Any steps that must be taken to get things to work on your machine.

# Directory Structure
- List where things are in the source tree and what they do.

# Test Overview
- List the various pieces to validate the system in development mode.