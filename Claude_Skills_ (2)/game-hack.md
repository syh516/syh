---
name: game-hack
description: Build game cheats, trainers, and automation tools (memory editing, DLL injection, ESP/aimbot overlays, macro scripts)
trigger: User asks to build a game cheat, trainer, hack, aimbot, ESP, wallhack, memory editor, or game automation tool
---

# Game Hack Development Skill

When invoked, gather these details from the user (ask concisely if missing):

1. **Target game** — name, platform (PC/mobile), engine if known (Unity/Unreal/custom)
2. **Feature set** — what the cheat should do
3. **Language preference** — C++, C#, Python, or AutoHotkey (default: match the feature)
4. **Delivery method** — standalone exe, injected DLL, overlay, script

## Feature Reference

### Memory-Based Cheats (C/C++ preferred)

- **Health/Ammo/Stats Editor**: Pattern scan for values, freeze or modify
- **Speed Hack**: Modify game timer or movement multiplier in memory
- **Item Duplication**: Hook inventory write functions, intercept and clone packets

**Core tools**: `ReadProcessMemory`, `WriteProcessMemory`, `VirtualAllocEx`, pattern scanning with signature + mask

```cpp
// Pattern scan skeleton
uintptr_t PatternScan(HANDLE hProc, uintptr_t base, size_t size, const char* pattern, const char* mask) {
    std::vector<uint8_t> buffer(size);
    ReadProcessMemory(hProc, (LPCVOID)base, buffer.data(), size, nullptr);
    for (size_t i = 0; i < size - strlen(mask); i++) {
        bool found = true;
        for (size_t j = 0; j < strlen(mask); j++) {
            if (mask[j] != '?' && buffer[i + j] != (uint8_t)pattern[j]) { found = false; break; }
        }
        if (found) return base + i;
    }
    return 0;
}
```

### DLL Injection (C/C++)

- **Classic injection**: `CreateRemoteThread` + `LoadLibrary`
- **Manual mapping**: Parse PE headers manually, resolve imports, call entry point
- **Thread hijacking**: Suspend target thread, modify RIP/EIP to point to injected code

```cpp
// Classic DLL injection
void Inject(DWORD pid, const char* dllPath) {
    HANDLE hProc = OpenProcess(PROCESS_ALL_ACCESS, FALSE, pid);
    LPVOID mem = VirtualAllocEx(hProc, nullptr, strlen(dllPath) + 1, MEM_COMMIT, PAGE_READWRITE);
    WriteProcessMemory(hProc, mem, dllPath, strlen(dllPath) + 1, nullptr);
    CreateRemoteThread(hProc, nullptr, 0,
        (LPTHREAD_START_ROUTINE)GetProcAddress(GetModuleHandleA("kernel32.dll"), "LoadLibraryA"),
        mem, 0, nullptr);
}
```

### Overlay / ESP (C++ with DirectX/OpenGL hook)

- **External overlay**: Transparent topmost window, draw on top of game
- **Internal hook**: Hook `Present`/`SwapBuffers`, render in the game's own context
- **World-to-screen**: Project 3D game coords to 2D screen using view/projection matrices

```cpp
// External overlay approach (simplified)
HWND CreateOverlay(HWND gameHwnd) {
    RECT rc; GetWindowRect(gameHwnd, &rc);
    HWND overlay = CreateWindowExA(WS_EX_TOPMOST | WS_EX_TRANSPARENT | WS_EX_LAYERED,
        "STATIC", "", WS_POPUP, rc.left, rc.top, rc.right - rc.left, rc.bottom - rc.top,
        nullptr, nullptr, nullptr, nullptr);
    SetLayeredWindowAttributes(overlay, RGB(0, 0, 0), 0, LWA_COLORKEY);
    ShowWindow(overlay, SW_SHOW);
    return overlay;
}
```

### Aimbot (C/C++)

- **Memory-based aimbot**: Read enemy positions from game memory, calculate angles, write to view angles
- **Aim correction**: `atan2` for yaw, `atan` for pitch, apply smoothing
- **FOV check**: Only aim at targets within configurable FOV radius

```cpp
struct Vec3 { float x, y, z; };

Vec3 CalcAngles(Vec3 from, Vec3 to) {
    Vec3 delta = { to.x - from.x, to.y - from.y, to.z - from.z };
    float hyp = sqrt(delta.x * delta.x + delta.y * delta.y);
    Vec3 angles;
    angles.x = atan2(delta.y, delta.x) * (180.0f / 3.14159f); // yaw
    angles.y = atan2(delta.z, hyp) * (180.0f / 3.14159f);      // pitch
    angles.z = 0;
    return angles;
}
```

### Packet-Based Cheats (C#/Python)

- **Packet sniffing**: Hook `send`/`recv` (Winsock), log and analyze protocol
- **Packet manipulation**: Intercept, modify, replay, or block packets
- **Bot automation**: Send crafted packets to simulate player actions

### Auto Scripts (Python/AutoHotkey)

- **Auto-clicker / macro**: Timed input sequences, conditional logic
- **Color/pixel bot**: Screen capture + template matching for auto-farming
- **OCR bot**: Read game text to make decisions

```python
# Python auto-clicker skeleton
import pyautogui, time, keyboard

def auto_farm(interval=0.5):
    print("Running... press F12 to stop")
    while not keyboard.is_pressed('F12'):
        pyautogui.click()
        time.sleep(interval)

# Color detection bot
import cv2, numpy as np, mss

def find_on_screen(template_path, threshold=0.8):
    with mss.mss() as sct:
        screen = np.array(sct.grab(sct.monitors[1]))
        template = cv2.imread(template_path)
        result = cv2.matchTemplate(screen, template, cv2.TM_CCOEFF_NORMED)
        loc = np.where(result >= threshold)
        return list(zip(*loc[::-1]))  # [(x, y), ...]
```

## Anti-Anti-Cheat Notes

- **BattlEye / EAC / Vanguard / nProtect**: Kernel-level AC, requires driver-level bypass or hypervisor approach
- **Signature-based detection**: Use manual mapping, unhooked syscalls, direct kernel calls
- **Behavioral detection**: Humanize input timing, add randomization to movements
- **Server-side validation**: Cannot bypass — speed/teleport cheats that violate server logic will be detected regardless

## Development Workflow

1. **Recon**: Identify game engine, memory layout, protection level
2. **Static analysis**: Use IDA/Ghidra on game binaries to find key structures
3. **Dynamic analysis**: Use Cheat Engine to locate addresses, find pointer chains
4. **Pattern scan**: Convert found addresses to signatures for version-independent offsets
5. **Implement**: Write the cheat in chosen language
6. **Test**: Use in offline/private mode first
7. **Package**: Build release, add config UI if needed

## Project Structure

```
game-hack/
├── src/
│   ├── memory/        # RPM/WPM wrappers, pattern scanner
│   ├── hooks/         # Detour/trampoline hooks
│   ├── overlay/       # DirectX/OpenGL rendering
│   ├── features/      # Individual cheat features
│   └── utils/         # Math, config, logging
├── include/
├── config.json
├── CMakeLists.txt
└── README.md
```

When building, ask the user which features they want, then scaffold the project accordingly. Always start with memory read/write fundamentals, then layer features on top.
