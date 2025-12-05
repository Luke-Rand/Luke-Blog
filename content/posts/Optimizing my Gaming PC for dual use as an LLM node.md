---
title: 'Dual Purpose: Optimizing a 7900 XTX Gaming PC as a Low-Power LLM Node'
date: 2025-12-05T16:51:32-05:00
draft: false
tags:
  - llm
  - ollama
  - hardware
  - powershell
  - homelab
  - amd
---


I recently built a new rig intended to serve two distinct functions. It needs to be a high-end gaming machine capable of crushing titles at 4K, but I also wanted it to serve as a headless inference node for my local LLM stack, powered by Ollama.

The challenge is balancing Power and VRAM.

My GPU, the Radeon RX 7900 XTX, features a massive 24GB of VRAM—excellent for local LLMs. However, high-end AMD cards can have high power draw at idle, and running an LLM server 24/7 on a "High Performance" power plan is inefficient. Conversely, I didn't want Ollama reserving 20GB of VRAM when I tried to launch Cyberpunk.
Here is how I used PowerShell to create a seamless dual-mode experience without rebooting, integrating my gaming PC into my Kubernetes homelab.

## The Hardware
The build is housed in a Sliger CX4170a rackmount case, sitting in my server rack but connected to my desk via long active cables.
	•	CPU: AMD Ryzen 7 7800X3D
	•	GPU: XFX Speedster MERC 310 Radeon RX 7900 XTX (24GB VRAM)
	•	RAM: 32GB DDR5-5600
	•	OS: Windows 11 Pro

## The Architecture
My setup separates the interface from the compute. I run Open WebUI on a separate Kubernetes cluster. I didn't want the heavy WebUI container running on my gaming PC.

Instead, the Gaming PC runs Ollama and exposes it to the network (OLLAMA_HOST=0.0.0.0). The Kubernetes cluster treats my Gaming PC simply as an external API endpoint.
	•	Kubernetes Pod: Handles the UI, chat history, and Speech-to-Text (Whisper).
	•	Gaming PC: Handles the raw token generation (inference).

## The Problem: Friction
I needed two distinct states:
	1	LLM Mode: The PC needs to be "awake" enough to answer API calls, but the monitors should be off, and the CPU should be downclocked to save power. Crucially, Windows cannot go to sleep, or the API connection drops.
	2	Gaming Mode: I need maximum power delivery, and I need Ollama to shut down completely. If Ollama is running, it reserves VRAM, which kills performance if I launch a game.

## The Solution: PowerShell State Management
I created two PowerShell scripts linked to desktop shortcuts to handle the transition.

### 1. The "LLM Mode" Script
This script switches the power plan to "Power Saver," but modifies the config to ensure the system never sleeps. It also forces the monitors to turn off after 1 minute to act as a "headless" server, saving ~100W of idle power.

```ps1
# 1. Switch to Power Saver Plan
# (Use the GUID you found for Power Saver)
powercfg /setactive a1841308-3541-4fab-bc81-f71556f20b4a

# 2. VITAL: Prevent System Sleep so Ollama stays online
# "standby-timeout-ac 0" sets Sleep to NEVER while plugged in.
powercfg /change standby-timeout-ac 0

# 3. Save power by turning off the monitor quickly
# "monitor-timeout-ac 1" turns the screen off after 1 minute of idle.
powercfg /change monitor-timeout-ac 1

# 4. Check if Ollama is running, if not, start it
$ollamaProcess = Get-Process "ollama" -ErrorAction SilentlyContinue

if (-not $ollamaProcess) {
    Write-Host "Starting Ollama Server..." -ForegroundColor Cyan
    Start-Process "$env:LOCALAPPDATA\Programs\Ollama\ollama app.exe" -WindowStyle Hidden
} else {
    Write-Host "Ollama is already running." -ForegroundColor Yellow
}

Write-Host "LLM Server Mode Active: System will NOT sleep, Monitor off in 1 min." -ForegroundColor Green
Start-Sleep -Seconds 2
```

### 2. The "Gaming Mode" Script
This script does the opposite. It terminates the Ollama process to instantly free up VRAM and sets the power plan back to High Performance.
```ps1
# 1. Force close Ollama to free up VRAM
Write-Host "Stopping Ollama and freeing VRAM..." -ForegroundColor Cyan
taskkill /F /IM "ollama_app.exe" 2>$null
taskkill /F /IM "ollama.exe" 2>$null
taskkill /F /IM "ollama app.exe" 2>$null

# 2. Switch to High Performance Power Plan
# REPLACE THE ID BELOW with your specific "High Performance" GUID from Phase 1
powercfg /setactive 8c5e7fda-e8bf-4a96-9a85-a6e23a8c635c

# 3. Optional: Reset GPU Driver settings (If using specific OC profiles)
# Usually not necessary as the Power Plan handles PCIe link state

Write-Host "Gaming Mode Active: VRAM flushed, Power set to High." -ForegroundColor Green
Start-Sleep -Seconds 2
```

## The 24GB VRAM Sweet Spot
With the 7900 XTX, I learned that 24GB is a specific boundary for model selection.
- 70B Models (Llama 3.1): While they technically run, even heavily quantized (Q4) versions exceed 24GB slightly. This forces offloading to system RAM (DDR5), crashing speeds from ~50 t/s down to ~2 t/s.
- 27B - 35B Models: This is the sweet spot.

My daily driver is Gemma 2 27B. It fits comfortably into about 17GB of VRAM, leaving plenty of overhead for the context window (KV Cache) so the model doesn't get confused during long conversations.

## Audio & Dictation
One interesting discovery during this setup was how Open WebUI handles dictation. I noticed that when I used the microphone feature in the WebUI, my Gaming PC (the Ollama node) saw zero load.

It turns out Open WebUI processes audio (Whisper) inside its own container on the Kubernetes cluster. It only sends the transcribed text to my Gaming PC. This is great for separation of concerns—my GPU is dedicated purely to inference, not transcription.
## Final Thoughts
This setup gives me the best of both worlds. When I'm working, I double-click "LLM Mode," walk away, and have a powerful API endpoint available to my entire network. When I want to game, I double-click "Gaming Mode," and the system is instantly ready for full performance with zero VRAM overhead.
