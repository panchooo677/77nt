

local Players = game:GetService("Players")
local LocalPlayer = Players.LocalPlayer
local UserInputService = game:GetService("UserInputService")
local GuiService = game:GetService("GuiService")
local TweenService = game:GetService("TweenService")
local VirtualInputManager = game:GetService("VirtualInputManager")
local ContextActionService = game:GetService("ContextActionService")

-- ============================================
-- WORKSPACE FOLDER INIT
-- ============================================
pcall(function() 
    if not isfolder("My MIDI") then makefolder("My MIDI") end 
    if not isfolder("MIDI Favorites") then makefolder("MIDI Favorites") end 
end)

local function moveFile(srcPath, destFolderName)
    local finalPath = srcPath
    pcall(function()
        local fileName = srcPath:match("([^/\\]+)$")
        if not fileName then return end
        
        local newPath = destFolderName .. "/" .. fileName
        local data = readfile(srcPath)
        if data then
            writefile(newPath, data)
            if isfile(newPath) then delfile(srcPath); finalPath = newPath end
        end
    end)
    return finalPath
end

-- ============================================
-- STRICT 61-KEY MAPPING
-- ============================================
local KEY_MAP = {
    [36]="1", [37]="!", [38]="2", [39]="@", [40]="3", [41]="4", [42]="$", [43]="5", [44]="%", [45]="6", [46]="^", [47]="7",
    [48]="8", [49]="*", [50]="9", [51]="(", [52]="0", [53]="q", [54]="Q", [55]="w", [56]="W", [57]="e", [58]="E", [59]="r",
    [60]="t", [61]="T", [62]="y", [63]="Y", [64]="u", [65]="i", [66]="I", [67]="o", [68]="O", [69]="p", [70]="P", [71]="a",
    [72]="s", [73]="S", [74]="d", [75]="D", [76]="f", [77]="g", [78]="G", [79]="h", [80]="H", [81]="j", [82]="J", [83]="k",
    [84]="l", [85]="L", [86]="z", [87]="Z", [88]="x", [89]="c", [90]="C", [91]="v", [92]="V", [93]="b", [94]="B", [95]="n",
    [96]="m"
}

local SHIFT_KEYS = {
    ["!"]=true, ["@"]=true, ["$"]=true, ["%"]=true, ["^"]=true, ["*"]=true, ["("]=true, 
    ["Q"]=true, ["W"]=true, ["E"]=true, ["T"]=true, ["Y"]=true, ["I"]=true, ["O"]=true, 
    ["P"]=true, ["S"]=true, ["D"]=true, ["G"]=true, ["H"]=true, ["J"]=true, ["L"]=true, 
    ["Z"]=true, ["C"]=true, ["V"]=true, ["B"]=true,
}

local KEY_TO_KEYCODE = {
    ["1"]=Enum.KeyCode.One, ["2"]=Enum.KeyCode.Two, ["3"]=Enum.KeyCode.Three, ["4"]=Enum.KeyCode.Four, ["5"]=Enum.KeyCode.Five, ["6"]=Enum.KeyCode.Six, ["7"]=Enum.KeyCode.Seven, ["8"]=Enum.KeyCode.Eight, ["9"]=Enum.KeyCode.Nine, ["0"]=Enum.KeyCode.Zero,
    ["q"]=Enum.KeyCode.Q, ["w"]=Enum.KeyCode.W, ["e"]=Enum.KeyCode.E, ["r"]=Enum.KeyCode.R, ["t"]=Enum.KeyCode.T, ["y"]=Enum.KeyCode.Y, ["u"]=Enum.KeyCode.U, ["i"]=Enum.KeyCode.I, ["o"]=Enum.KeyCode.O, ["p"]=Enum.KeyCode.P,
    ["a"]=Enum.KeyCode.A, ["s"]=Enum.KeyCode.S, ["d"]=Enum.KeyCode.D, ["f"]=Enum.KeyCode.F, ["g"]=Enum.KeyCode.G, ["h"]=Enum.KeyCode.H, ["j"]=Enum.KeyCode.J, ["k"]=Enum.KeyCode.K, ["l"]=Enum.KeyCode.L,
    ["z"]=Enum.KeyCode.Z, ["x"]=Enum.KeyCode.X, ["c"]=Enum.KeyCode.C, ["v"]=Enum.KeyCode.V, ["b"]=Enum.KeyCode.B, ["n"]=Enum.KeyCode.N, ["m"]=Enum.KeyCode.M,
    
    ["!"]=Enum.KeyCode.One, ["@"]=Enum.KeyCode.Two, ["$"]=Enum.KeyCode.Four, ["%"]=Enum.KeyCode.Five, ["^"]=Enum.KeyCode.Six, ["*"]=Enum.KeyCode.Eight, ["("]=Enum.KeyCode.Nine,
    ["Q"]=Enum.KeyCode.Q, ["W"]=Enum.KeyCode.W, ["E"]=Enum.KeyCode.E, ["T"]=Enum.KeyCode.T, ["Y"]=Enum.KeyCode.Y, ["I"]=Enum.KeyCode.I, ["O"]=Enum.KeyCode.O, ["P"]=Enum.KeyCode.P,
    ["S"]=Enum.KeyCode.S, ["D"]=Enum.KeyCode.D, ["G"]=Enum.KeyCode.G, ["H"]=Enum.KeyCode.H, ["J"]=Enum.KeyCode.J, ["L"]=Enum.KeyCode.L,
    ["Z"]=Enum.KeyCode.Z, ["C"]=Enum.KeyCode.C, ["V"]=Enum.KeyCode.V, ["B"]=Enum.KeyCode.B,
}

-- ============================================
-- STATE VARIABLES
-- ============================================
local isPlaying, isPaused, playCoroutine = false, false, nil
local playSpeed, seekAmount = 1.0, 10
local allSongs, songItems = {}, {}

local selectedSong, playingSong = nil, nil 
local isMinimized, useNoteLengths, blockMenus = false, true, false
local showingFavorites, isEditMode = false, false
local heldKeys = {}
local globalPressCounter, isTimeFrozen = 0, false

local currentFrames, currentPlayTime, totalSongTime = nil, 0, 0
local onFinishCallback = nil

local function formatTime(seconds)
    if not seconds or seconds ~= seconds then seconds = 0 end
    local m = math.floor(seconds / 60)
    local s = math.floor(seconds % 60)
    return string.format("%02d:%02d", m, s)
end

-- ============================================
-- GAME SKILL / MENU BLOCKER
-- ============================================
local function setGameBlocker(enabled)
    if enabled and blockMenus then
        local blockedKeys = {Enum.KeyCode.Tab, Enum.KeyCode.Backquote, Enum.KeyCode.Space}
        for _, kc in pairs(KEY_TO_KEYCODE) do table.insert(blockedKeys, kc) end
        ContextActionService:BindActionAtPriority("PianoAutoplayBlocker", function() return Enum.ContextActionResult.Sink end, false, 2000, unpack(blockedKeys))
    else
        ContextActionService:UnbindAction("PianoAutoplayBlocker")
    end
end

-- ============================================
-- ANTI-LAG MIDI PARSER
-- ============================================
local function parseMidi(data)
    local pos, yt = 1, os.clock()
    local function yieldCheck() if os.clock() - yt > 0.015 then task.wait(); yt = os.clock() end end

    local function readByte() if pos > #data then return 0 end; local b = string.byte(data, pos); pos = pos + 1; return b end
    local function readBytes(n) local bytes = {}; for i=1,n do table.insert(bytes, readByte()) end; return bytes end
    local function readUInt32() local b = readBytes(4); return b[1]*16777216 + b[2]*65536 + b[3]*256 + b[4] end
    local function readUInt16() local b = readBytes(2); return b[1]*256 + b[2] end
    local function readVarLen() local val, b = 0, 0; repeat b = readByte(); val = bit32.lshift(val, 7) + bit32.band(b, 0x7F) until bit32.band(b, 0x80) == 0; return val end

    local header = {}; for i=1,4 do table.insert(header, string.char(readByte())) end
    if table.concat(header) ~= "MThd" then return nil end

    readUInt32(); readUInt16(); local numTracks = readUInt16(); local division = readUInt16()
    local allEvents, tempoChanges = {}, {{tick = 0, tempo = 500000}}

    for track = 1, numTracks do
        for i=1,4 do readByte() end
        local trackEnd = pos + readUInt32()
        local absoluteTick, lastStatus = 0, 0

        while pos < trackEnd do
            yieldCheck()
            absoluteTick = absoluteTick + readVarLen()
            local statusByte = string.byte(data, pos)
            if bit32.band(statusByte, 0x80) ~= 0 then lastStatus = statusByte; pos = pos + 1 else statusByte = lastStatus end
            local msgType = bit32.band(statusByte, 0xF0)

            if statusByte == 0xFF then
                local metaType = readByte(); local metaLen = readVarLen()
                if metaType == 0x51 then
                    local t = readByte()*65536 + readByte()*256 + readByte()
                    table.insert(tempoChanges, {tick = absoluteTick, tempo = t})
                else for i=1,metaLen do readByte() end end
            elseif statusByte == 0xF0 or statusByte == 0xF7 then
                for i=1,readVarLen() do readByte() end
            elseif msgType == 0x80 or msgType == 0x90 then
                local noteNum, vel = readByte(), readByte()
                local isNoteOn = (msgType == 0x90 and vel > 0)
                table.insert(allEvents, {tick = absoluteTick, note = noteNum, type = isNoteOn and "on" or "off"})
            elseif msgType == 0xA0 or msgType == 0xB0 or msgType == 0xE0 then readByte(); readByte()
            elseif msgType == 0xC0 or msgType == 0xD0 then readByte() else break end
        end
        pos = trackEnd
    end

    table.sort(tempoChanges, function(a, b) return a.tick < b.tick end)
    table.sort(allEvents, function(a, b) return a.tick < b.tick end)

    local function getSeconds(tick)
        local time, lastTick, currentTempo = 0, 0, 500000
        for _, tc in ipairs(tempoChanges) do
            if tc.tick >= tick then break end
            time = time + ((tc.tick - lastTick) / division) * (currentTempo / 1000000)
            currentTempo = tc.tempo; lastTick = tc.tick
        end
        return time + ((tick - lastTick) / division) * (currentTempo / 1000000)
    end

    local activeNotes, timeEvents = {}, {}
    totalSongTime = 0

    for _, event in ipairs(allEvents) do
        yieldCheck()
        local foldedNote = event.note
        while foldedNote < 36 do foldedNote = foldedNote + 12 end
        while foldedNote > 96 do foldedNote = foldedNote - 12 end
        local key = KEY_MAP[foldedNote]
        
        if key then
            local timeSecs = getSeconds(event.tick)
            if event.type == "on" then
                if activeNotes[event.note] then activeNotes[event.note].dur = math.max(0.015, timeSecs - activeNotes[event.note].time - 0.01) end
                local noteObj = {key = key, time = timeSecs, dur = 0.1}
                table.insert(timeEvents, noteObj)
                activeNotes[event.note] = noteObj
            elseif event.type == "off" then
                if activeNotes[event.note] then
                    activeNotes[event.note].dur = math.max(0.02, timeSecs - activeNotes[event.note].time)
                    activeNotes[event.note] = nil
                end
            end
            if timeSecs > totalSongTime then totalSongTime = timeSecs end
        end
    end

    local framesMap, orderedTimes = {}, {}
    for _, note in ipairs(timeEvents) do
        yieldCheck()
        local timeRounded = math.floor(note.time * 1000) / 1000
        if not framesMap[timeRounded] then
            framesMap[timeRounded] = {shifted = {}, unshifted = {}, all = {}, seen = {}}
            table.insert(orderedTimes, timeRounded)
        end
        if not framesMap[timeRounded].seen[note.key] then
            framesMap[timeRounded].seen[note.key] = true
            table.insert(framesMap[timeRounded].all, note)
            if SHIFT_KEYS[note.key] then table.insert(framesMap[timeRounded].shifted, note)
            else table.insert(framesMap[timeRounded].unshifted, note) end
        end
    end
    table.sort(orderedTimes)

    local playbackFrames = {}
    for _, t in ipairs(orderedTimes) do
        yieldCheck()
        table.insert(playbackFrames, {time = t, shifted = framesMap[t].shifted, unshifted = framesMap[t].unshifted, all = framesMap[t].all})
    end
    return playbackFrames
end

-- ============================================
-- DYNAMIC ENGINE (Zero Lag / No Spam)
-- ============================================
local TimeLabel, ProgressBarFill, setStatus, updatePlayBtn, updateListVisuals

local function releaseAllKeys()
    for key, _ in pairs(heldKeys) do
        local kc = KEY_TO_KEYCODE[key]
        if kc then VirtualInputManager:SendKeyEvent(false, kc, false, game) end
    end
    table.clear(heldKeys)
    VirtualInputManager:SendKeyEvent(false, Enum.KeyCode.LeftShift, false, game)
end

local function handleFocusLoss() if isPlaying then isTimeFrozen = true; releaseAllKeys() end end
local function handleFocusGain() isTimeFrozen = false end

GuiService.MenuOpened:Connect(handleFocusLoss)
GuiService.MenuClosed:Connect(handleFocusGain)
UserInputService.WindowFocusReleased:Connect(handleFocusLoss)
UserInputService.WindowFocused:Connect(handleFocusGain)

local function startPlayLoop(startIndex)
    setGameBlocker(true)
    
    playCoroutine = coroutine.create(function()
        local lastTick = os.clock()
        
        for i = startIndex, #currentFrames do
            local frame = currentFrames[i]
            
            -- DYNAMIC ACCUMULATOR: Adapts instantly to Speed clicks without restarting!
            while true do
                if not isPlaying then break end
                
                if isPaused or isTimeFrozen then
                    lastTick = os.clock()
                    task.wait(0.05)
                    continue
                end
                
                local now = os.clock()
                local dt = now - lastTick
                lastTick = now
                
                currentPlayTime = currentPlayTime + (dt * playSpeed)
                
                if currentPlayTime >= frame.time then 
                    break 
                end
                
                if totalSongTime > 0 then
                    TimeLabel.Text = formatTime(currentPlayTime) .. " / " .. formatTime(totalSongTime)
                    ProgressBarFill.Size = UDim2.new(math.clamp(currentPlayTime / totalSongTime, 0, 1), 0, 1, 0)
                end
                
                task.wait() 
            end
            if not isPlaying then break end

            globalPressCounter = globalPressCounter + 1
            local pressId = globalPressCounter
            
            -- UNIVERSAL INJECTION: Native Texts for GUI Pianos
            for _, note in ipairs(frame.all) do
                pcall(function() VirtualInputManager:SendTextInput(note.key, game) end)
            end

            local function pressList(list)
                for _, note in ipairs(list) do
                    local kc = KEY_TO_KEYCODE[note.key]
                    if kc then VirtualInputManager:SendKeyEvent(true, kc, false, game) end
                    heldKeys[note.key] = { id = pressId }
                    
                    local dur = useNoteLengths and note.dur or 0.05
                    task.delay(dur / math.max(0.1, playSpeed), function()
                        local hk = heldKeys[note.key]
                        if hk and hk.id == pressId then
                            if kc then VirtualInputManager:SendKeyEvent(false, kc, false, game) end
                            heldKeys[note.key] = nil
                        end
                    end)
                end
            end

            -- ZERO-LAG CHORD SPLITTER: Perfect separation instantly without yielding!
            if #frame.shifted > 0 then
                VirtualInputManager:SendKeyEvent(true, Enum.KeyCode.LeftShift, false, game)
                pressList(frame.shifted)
                VirtualInputManager:SendKeyEvent(false, Enum.KeyCode.LeftShift, false, game)
            end
            if #frame.unshifted > 0 then 
                pressList(frame.unshifted) 
            end
        end
        
        if isPlaying then 
            isPlaying = false
            playingSong = nil
            setGameBlocker(false)
            updateListVisuals()
            if onFinishCallback then onFinishCallback() end 
        end
    end)
    coroutine.resume(playCoroutine)
end

local function stopSong()
    isPlaying, isPaused, isTimeFrozen = false, false, false
    playingSong = nil
    setGameBlocker(false)
    if playCoroutine then pcall(function() coroutine.close(playCoroutine) end); playCoroutine = nil end
    releaseAllKeys()
    currentPlayTime, currentFrames = 0, nil
    if TimeLabel then TimeLabel.Text = "00:00 / 00:00" end
    if ProgressBarFill then TweenService:Create(ProgressBarFill, TweenInfo.new(0.01, Enum.EasingStyle.Linear), {Size = UDim2.new(0, 0, 1, 0)}):Play() end
    updateListVisuals()
end

local function playSong(frames, onFinish)
    stopSong()
    if not frames or #frames == 0 then return end
    currentFrames, onFinishCallback = frames, onFinish
    currentPlayTime = 0
    playingSong = selectedSong
    updateListVisuals()
    isPlaying, isPaused, isTimeFrozen = true, false, false
    startPlayLoop(1)
end

local isSeeking = false
local function seekToTime(targetTime)
    if not isPlaying or not currentFrames or isSeeking then return end
    isSeeking = true
    
    -- Kill the old loop cleanly to stop overlapping notes
    if playCoroutine then pcall(function() coroutine.close(playCoroutine) end); playCoroutine = nil end
    releaseAllKeys()
    
    currentPlayTime = math.clamp(targetTime, 0, totalSongTime)
    
    local targetIndex = 1
    for i, frame in ipairs(currentFrames) do 
        if frame.time >= currentPlayTime then targetIndex = i; break end 
    end
    
    TimeLabel.Text = formatTime(currentPlayTime) .. " / " .. formatTime(totalSongTime)
    local progress = totalSongTime > 0 and math.clamp(currentPlayTime / totalSongTime, 0, 1) or 0
    ProgressBarFill.Size = UDim2.new(progress, 0, 1, 0)
    
    startPlayLoop(targetIndex)
    isSeeking = false
end

-- ============================================
-- macOS UI STYLING UTILITY
-- ============================================
local function styleMacButton(btn, isPrimary)
    btn.AutoButtonColor = false
    btn.Font = Enum.Font.GothamSemibold
    btn.TextColor3 = Color3.fromRGB(255, 255, 255)
    local baseColor = isPrimary and Color3.fromRGB(10, 132, 255) or Color3.fromRGB(75, 75, 80)
    local hoverColor = isPrimary and Color3.fromRGB(40, 150, 255) or Color3.fromRGB(90, 90, 95)
    btn.BackgroundColor3 = baseColor
    Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 6)
    local stroke = Instance.new("UIStroke", btn)
    stroke.Color = Color3.fromRGB(0, 0, 0)
    stroke.Transparency = 0.5; stroke.Thickness = 1
    
    btn.MouseEnter:Connect(function() TweenService:Create(btn, TweenInfo.new(0.15), {BackgroundColor3 = hoverColor}):Play() end)
    btn.MouseLeave:Connect(function() TweenService:Create(btn, TweenInfo.new(0.15), {BackgroundColor3 = baseColor}):Play() end)
end

-- ============================================
-- MAIN UI CONSTRUCTION
-- ============================================
local oldGui = LocalPlayer:FindFirstChild("PlayerGui") and LocalPlayer.PlayerGui:FindFirstChild("PianoAutoplay")
if oldGui then oldGui:Destroy() end

local ScreenGui = Instance.new("ScreenGui", LocalPlayer:WaitForChild("PlayerGui"))
ScreenGui.Name, ScreenGui.ResetOnSpawn, ScreenGui.IgnoreGuiInset, ScreenGui.DisplayOrder = "PianoAutoplay", false, true, 999

local Main = Instance.new("Frame", ScreenGui)
Main.Size, Main.Position, Main.BackgroundColor3, Main.Active, Main.Draggable = UDim2.new(0, 280, 0, 485), UDim2.new(1, -295, 0, 10), Color3.fromRGB(30, 30, 32), true, true
Instance.new("UICorner", Main).CornerRadius = UDim.new(0, 10)
local mainStroke = Instance.new("UIStroke", Main); mainStroke.Color = Color3.fromRGB(0, 0, 0); mainStroke.Transparency = 0.4; mainStroke.Thickness = 1

local TitleBar = Instance.new("Frame", Main)
TitleBar.Size, TitleBar.BackgroundColor3 = UDim2.new(1, 0, 0, 38), Color3.fromRGB(45, 45, 48)
Instance.new("UICorner", TitleBar).CornerRadius = UDim.new(0, 10)
local titleFix = Instance.new("Frame", TitleBar); titleFix.Size, titleFix.Position, titleFix.BackgroundColor3, titleFix.BorderSizePixel = UDim2.new(1, 0, 0.5, 0), UDim2.new(0, 0, 0.5, 0), Color3.fromRGB(45, 45, 48), 0
local titleLine = Instance.new("Frame", TitleBar); titleLine.Size, titleLine.Position, titleLine.BackgroundColor3, titleLine.BorderSizePixel = UDim2.new(1, 0, 0, 1), UDim2.new(0, 0, 1, 0), Color3.fromRGB(20, 20, 20), 0

local TitleLbl = Instance.new("TextLabel", TitleBar)
TitleLbl.Size, TitleLbl.Position, TitleLbl.BackgroundTransparency = UDim2.new(1, 0, 1, 0), UDim2.new(0, 0, 0, 0), 1
TitleLbl.Text, TitleLbl.TextSize, TitleLbl.Font, TitleLbl.TextColor3, TitleLbl.TextXAlignment = "Piano Autoplay", 13, Enum.Font.GothamSemibold, Color3.fromRGB(220, 220, 220), Enum.TextXAlignment.Center

local CloseBtn = Instance.new("TextButton", TitleBar)
CloseBtn.Size, CloseBtn.Position, CloseBtn.BackgroundColor3, CloseBtn.Text, CloseBtn.TextSize, CloseBtn.Font, CloseBtn.TextColor3 = UDim2.new(0, 12, 0, 12), UDim2.new(0, 12, 0.5, -6), Color3.fromRGB(255, 95, 86), "", 10, Enum.Font.GothamBold, Color3.fromRGB(70, 20, 20)
Instance.new("UICorner", CloseBtn).CornerRadius = UDim.new(1, 0)
CloseBtn.MouseEnter:Connect(function() CloseBtn.Text = "✕" end); CloseBtn.MouseLeave:Connect(function() CloseBtn.Text = "" end)

local MinBtn = Instance.new("TextButton", TitleBar)
MinBtn.Size, MinBtn.Position, MinBtn.BackgroundColor3, MinBtn.Text, MinBtn.TextSize, MinBtn.Font, MinBtn.TextColor3 = UDim2.new(0, 12, 0, 12), UDim2.new(0, 32, 0.5, -6), Color3.fromRGB(255, 189, 46), "", 12, Enum.Font.GothamBold, Color3.fromRGB(100, 60, 10)
Instance.new("UICorner", MinBtn).CornerRadius = UDim.new(1, 0)
MinBtn.MouseEnter:Connect(function() MinBtn.Text = "-" end); MinBtn.MouseLeave:Connect(function() MinBtn.Text = "" end)

local Content = Instance.new("Frame", Main)
Content.Size, Content.Position, Content.BackgroundTransparency = UDim2.new(1, -20, 1, -48), UDim2.new(0, 10, 0, 44), 1

local NPBox = Instance.new("Frame", Content)
NPBox.Size, NPBox.BackgroundColor3 = UDim2.new(1, 0, 0, 44), Color3.fromRGB(40, 40, 44)
Instance.new("UICorner", NPBox).CornerRadius = UDim.new(0, 6); Instance.new("UIStroke", NPBox).Color = Color3.fromRGB(20, 20, 20); Instance.new("UIStroke", NPBox).Transparency = 0.5
local NPTag = Instance.new("TextLabel", NPBox); NPTag.Size, NPTag.Position, NPTag.BackgroundTransparency, NPTag.Text, NPTag.TextSize, NPTag.Font, NPTag.TextColor3, NPTag.TextXAlignment = UDim2.new(1, -10, 0, 16), UDim2.new(0, 10, 0, 4), 1, "NOW PLAYING", 9, Enum.Font.GothamBold, Color3.fromRGB(150, 150, 160), Enum.TextXAlignment.Left
local NPName = Instance.new("TextLabel", NPBox); NPName.Size, NPName.Position, NPName.BackgroundTransparency, NPName.Text, NPName.TextSize, NPName.Font, NPName.TextColor3, NPName.TextXAlignment, NPName.TextTruncate = UDim2.new(1, -20, 0, 20), UDim2.new(0, 10, 0, 20), 1, "...", 12, Enum.Font.GothamSemibold, Color3.fromRGB(255, 255, 255), Enum.TextXAlignment.Left, Enum.TextTruncate.AtEnd

local CopyOverlay = Instance.new("TextButton", NPBox)
CopyOverlay.Size, CopyOverlay.BackgroundTransparency, CopyOverlay.Text = UDim2.new(1, 0, 1, 0), 1, ""
local lastClick = 0
CopyOverlay.MouseButton1Click:Connect(function()
    local now = os.clock()
    if now - lastClick < 0.4 then
        if playingSong and setclipboard then
            setclipboard(playingSong.name)
            NPName.Text = "Copied to clipboard!"
            NPName.TextColor3 = Color3.fromRGB(100, 255, 150)
            task.delay(1.5, function()
                if playingSong then NPName.Text = playingSong.name else NPName.Text = "..." end
                NPName.TextColor3 = Color3.fromRGB(255, 255, 255)
            end)
        end
    end
    lastClick = now
end)

local SongsLbl = Instance.new("TextLabel", Content)
SongsLbl.Size, SongsLbl.Position, SongsLbl.BackgroundTransparency, SongsLbl.Text, SongsLbl.TextSize, SongsLbl.Font, SongsLbl.TextColor3, SongsLbl.TextXAlignment = UDim2.new(0, 75, 0, 16), UDim2.new(0, 0, 0, 52), 1, "MIDI FILES", 10, Enum.Font.GothamBold, Color3.fromRGB(150, 150, 160), Enum.TextXAlignment.Left

local RefreshBtn = Instance.new("TextButton", Content)
RefreshBtn.Size, RefreshBtn.Position, RefreshBtn.Text, RefreshBtn.TextSize = UDim2.new(0, 55, 0, 18), UDim2.new(1, -55, 0, 51), "Refresh", 10
styleMacButton(RefreshBtn, false)

local FavViewBtn = Instance.new("TextButton", Content)
FavViewBtn.Size, FavViewBtn.Position, FavViewBtn.Text, FavViewBtn.TextSize = UDim2.new(0, 55, 0, 18), UDim2.new(1, -115, 0, 51), "📜 All", 10
styleMacButton(FavViewBtn, false)

local EditBtn = Instance.new("TextButton", Content)
EditBtn.Size, EditBtn.Position, EditBtn.Text, EditBtn.TextSize = UDim2.new(0, 50, 0, 18), UDim2.new(1, -170, 0, 51), "✏️ Edit", 10
styleMacButton(EditBtn, false)

local Scroll = Instance.new("ScrollingFrame", Content)
Scroll.Size, Scroll.Position, Scroll.BackgroundColor3, Scroll.ScrollBarThickness, Scroll.ScrollBarImageColor3, Scroll.ScrollingDirection = UDim2.new(1, 0, 0, 150), UDim2.new(0, 0, 0, 74), Color3.fromRGB(25, 25, 28), 4, Color3.fromRGB(100, 100, 105), Enum.ScrollingDirection.Y
Instance.new("UICorner", Scroll).CornerRadius = UDim.new(0, 6); local sStroke = Instance.new("UIStroke", Scroll); sStroke.Color = Color3.fromRGB(15, 15, 15); sStroke.Transparency = 0.5
Instance.new("UIListLayout", Scroll).Padding = UDim.new(0, 2); Instance.new("UIPadding", Scroll).PaddingTop = UDim.new(0, 4); Instance.new("UIPadding", Scroll).PaddingBottom = UDim.new(0, 4)

local EmptyLbl = Instance.new("TextLabel", Scroll)
EmptyLbl.Size, EmptyLbl.BackgroundTransparency, EmptyLbl.Text, EmptyLbl.TextSize, EmptyLbl.Font, EmptyLbl.TextColor3, EmptyLbl.TextXAlignment = UDim2.new(1, 0, 0, 40), 1, "No MIDI files found", 11, Enum.Font.Gotham, Color3.fromRGB(120, 120, 130), Enum.TextXAlignment.Center

local ProgressBox = Instance.new("Frame", Content)
ProgressBox.Size, ProgressBox.Position, ProgressBox.BackgroundColor3 = UDim2.new(1, 0, 0, 6), UDim2.new(0, 0, 1, -185), Color3.fromRGB(20, 20, 22)
Instance.new("UICorner", ProgressBox).CornerRadius = UDim.new(1, 0); local pbStroke = Instance.new("UIStroke", ProgressBox); pbStroke.Color = Color3.fromRGB(10, 10, 10)
ProgressBarFill = Instance.new("Frame", ProgressBox); ProgressBarFill.Size, ProgressBarFill.BackgroundColor3 = UDim2.new(0, 0, 1, 0), Color3.fromRGB(10, 132, 255)
Instance.new("UICorner", ProgressBarFill).CornerRadius = UDim.new(1, 0)

TimeLabel = Instance.new("TextLabel", Content)
TimeLabel.Size, TimeLabel.Position, TimeLabel.BackgroundTransparency, TimeLabel.Text, TimeLabel.TextSize, TimeLabel.Font, TimeLabel.TextColor3, TimeLabel.TextXAlignment = UDim2.new(1, -10, 0, 14), UDim2.new(0, 0, 1, -175), 1, "00:00 / 00:00", 10, Enum.Font.Gotham, Color3.fromRGB(180, 180, 190), Enum.TextXAlignment.Right

local ToggleBtn1 = Instance.new("TextButton", Content)
ToggleBtn1.Size, ToggleBtn1.Position, ToggleBtn1.BackgroundTransparency = UDim2.new(0, 90, 0, 18), UDim2.new(0, 10, 1, -177), 1
ToggleBtn1.Text, ToggleBtn1.TextSize, ToggleBtn1.Font, ToggleBtn1.TextColor3, ToggleBtn1.TextXAlignment = "☑ Note Lengths", 10, Enum.Font.Gotham, Color3.fromRGB(220, 220, 220), Enum.TextXAlignment.Left
ToggleBtn1.MouseButton1Click:Connect(function() useNoteLengths = not useNoteLengths; ToggleBtn1.Text = useNoteLengths and "☑ Note Lengths" or "☐ Note Lengths"; ToggleBtn1.TextColor3 = useNoteLengths and Color3.fromRGB(220, 220, 220) or Color3.fromRGB(120, 120, 130) end)

local ToggleBtn2 = Instance.new("TextButton", Content)
ToggleBtn2.Size, ToggleBtn2.Position, ToggleBtn2.BackgroundTransparency = UDim2.new(0, 85, 0, 18), UDim2.new(0.5, -25, 1, -177), 1
ToggleBtn2.Text, ToggleBtn2.TextSize, ToggleBtn2.Font, ToggleBtn2.TextColor3, ToggleBtn2.TextXAlignment = "☐ Block Menus", 10, Enum.Font.Gotham, Color3.fromRGB(120, 120, 130), Enum.TextXAlignment.Left

local InfoBtn = Instance.new("TextButton", Content)
InfoBtn.Size, InfoBtn.Position, InfoBtn.BackgroundColor3 = UDim2.new(0, 16, 0, 16), UDim2.new(0.5, 10, 1, -158), Color3.fromRGB(75, 75, 80)
InfoBtn.Text, InfoBtn.TextSize, InfoBtn.Font, InfoBtn.TextColor3 = "?", 10, Enum.Font.GothamBold, Color3.fromRGB(255, 255, 255)
Instance.new("UICorner", InfoBtn).CornerRadius = UDim.new(1, 0)
InfoBtn.MouseEnter:Connect(function() TweenService:Create(InfoBtn, TweenInfo.new(0.15), {BackgroundColor3 = Color3.fromRGB(100, 100, 105)}):Play() end)
InfoBtn.MouseLeave:Connect(function() TweenService:Create(InfoBtn, TweenInfo.new(0.15), {BackgroundColor3 = Color3.fromRGB(75, 75, 80)}):Play() end)

local Controls = Instance.new("Frame", Content)
Controls.Size, Controls.Position, Controls.BackgroundTransparency = UDim2.new(1, 0, 0, 110), UDim2.new(0, 0, 1, -135), 1

local StopBtn = Instance.new("TextButton", Controls)
StopBtn.Size, StopBtn.Position, StopBtn.Text = UDim2.new(0, 36, 0, 32), UDim2.new(0, 0, 0, 0), "⏹"
styleMacButton(StopBtn, false)

local PlayBtn = Instance.new("TextButton", Controls)
PlayBtn.Size, PlayBtn.Position, PlayBtn.Text = UDim2.new(1, -126, 0, 32), UDim2.new(0, 42, 0, 0), "▶ Play"
styleMacButton(PlayBtn, true)

local RewindBtn = Instance.new("TextButton", Controls)
RewindBtn.Size, RewindBtn.Position, RewindBtn.Text = UDim2.new(0, 36, 0, 32), UDim2.new(1, -78, 0, 0), "⏪"
styleMacButton(RewindBtn, false)

local ForwardBtn = Instance.new("TextButton", Controls)
ForwardBtn.Size, ForwardBtn.Position, ForwardBtn.Text = UDim2.new(0, 36, 0, 32), UDim2.new(1, -36, 0, 0), "⏩"
styleMacButton(ForwardBtn, false)

local SpeedLbl = Instance.new("TextLabel", Controls)
SpeedLbl.Size, SpeedLbl.Position, SpeedLbl.BackgroundTransparency, SpeedLbl.Text, SpeedLbl.TextSize, SpeedLbl.Font, SpeedLbl.TextColor3, SpeedLbl.TextXAlignment = UDim2.new(0.5, 0, 0, 16), UDim2.new(0, 0, 0, 42), 1, "Speed: 100%", 11, Enum.Font.Gotham, Color3.fromRGB(200, 200, 210), Enum.TextXAlignment.Left

local SpeedDown = Instance.new("TextButton", Controls)
SpeedDown.Size, SpeedDown.Position, SpeedDown.Text = UDim2.new(0, 24, 0, 22), UDim2.new(0.5, -54, 0, 39), "−"
styleMacButton(SpeedDown, false)

local SpeedUp = Instance.new("TextButton", Controls)
SpeedUp.Size, SpeedUp.Position, SpeedUp.Text = UDim2.new(0, 24, 0, 22), UDim2.new(0.5, -26, 0, 39), "+"
styleMacButton(SpeedUp, false)

local SeekLbl = Instance.new("TextLabel", Controls)
SeekLbl.Size, SeekLbl.Position, SeekLbl.BackgroundTransparency, SeekLbl.Text, SeekLbl.TextSize, SeekLbl.Font, SeekLbl.TextColor3, SeekLbl.TextXAlignment = UDim2.new(0.5, 0, 0, 16), UDim2.new(0.5, 8, 0, 42), 1, "Seek: ±10s", 11, Enum.Font.Gotham, Color3.fromRGB(200, 200, 210), Enum.TextXAlignment.Left

local SeekDown = Instance.new("TextButton", Controls)
SeekDown.Size, SeekDown.Position, SeekDown.Text = UDim2.new(0, 24, 0, 22), UDim2.new(1, -50, 0, 39), "−"
styleMacButton(SeekDown, false)

local SeekUp = Instance.new("TextButton", Controls)
SeekUp.Size, SeekUp.Position, SeekUp.Text = UDim2.new(0, 24, 0, 22), UDim2.new(1, -22, 0, 39), "+"
styleMacButton(SeekUp, false)

local StatusLbl = Instance.new("TextLabel", Controls)
StatusLbl.Size, StatusLbl.Position, StatusLbl.BackgroundTransparency, StatusLbl.Text, StatusLbl.TextSize, StatusLbl.Font, StatusLbl.TextColor3, StatusLbl.TextXAlignment, StatusLbl.TextTruncate = UDim2.new(1, 0, 0, 14), UDim2.new(0, 0, 0, 72), 1, "● Ready", 10, Enum.Font.Gotham, Color3.fromRGB(150, 150, 160), Enum.TextXAlignment.Center, Enum.TextTruncate.AtEnd

function setStatus(text, color) StatusLbl.Text = text; StatusLbl.TextColor3 = color or Color3.fromRGB(150, 150, 160) end

function updateListVisuals()
    for i, data in pairs(songItems) do
        local frame, btn = data.frame, data.btn
        local song = allSongs[i]
        local songName = song and song.name or "Unknown"
        
        if playingSong and playingSong.name == songName then
            TweenService:Create(frame, TweenInfo.new(0.2), {BackgroundColor3 = Color3.fromRGB(10, 132, 255)}):Play()
            btn.TextColor3 = Color3.fromRGB(255, 255, 255)
            btn.Text = "  🔊 " .. songName
        elseif selectedSong and selectedSong.name == songName then
            TweenService:Create(frame, TweenInfo.new(0.2), {BackgroundColor3 = Color3.fromRGB(100, 100, 105)}):Play()
            btn.TextColor3 = Color3.fromRGB(230, 230, 230)
            btn.Text = "  📄 " .. songName
        else
            TweenService:Create(frame, TweenInfo.new(0.2), {BackgroundColor3 = Color3.fromRGB(75, 75, 80)}):Play()
            btn.TextColor3 = Color3.fromRGB(200, 200, 210)
            btn.Text = "  📄 " .. songName
        end
    end
end

local function buildList()
    for _, item in pairs(songItems) do pcall(function() item.frame:Destroy() end) end
    songItems = {}; EmptyLbl.Visible = false
    
    local targetFolder = showingFavorites and "MIDI Favorites" or "My MIDI"
    local files = {}
    
    pcall(function()
        for _, path in ipairs(listfiles(targetFolder)) do
            if path:lower():match("%.mid$") or path:lower():match("%.midi$") then
                local name = (path:match("([^/\\]+)$") or path):gsub("%.[Mm][Ii][Dd][Ii]?$", "")
                table.insert(files, {name = name, path = path})
            end
        end
    end)
    allSongs = files

    local count = #allSongs
    for i, song in ipairs(allSongs) do
        local ItemFrame = Instance.new("Frame", Scroll)
        ItemFrame.Size, ItemFrame.BackgroundColor3 = UDim2.new(1, 0, 0, 28), Color3.fromRGB(75, 75, 80)
        Instance.new("UICorner", ItemFrame).CornerRadius = UDim.new(0, 6)
        local stroke = Instance.new("UIStroke", ItemFrame); stroke.Color = Color3.fromRGB(0, 0, 0); stroke.Transparency = 0.5; stroke.Thickness = 1
        
        local Btn = Instance.new("TextButton", ItemFrame)
        Btn.Size, Btn.BackgroundTransparency = UDim2.new(1, isEditMode and -30 or 0, 1, 0), 1
        Btn.Text, Btn.TextSize, Btn.Font = "  📄 " .. song.name, 11, Enum.Font.Gotham
        Btn.TextXAlignment, Btn.TextTruncate = Enum.TextXAlignment.Left, Enum.TextTruncate.AtEnd
        
        local StarBtn = Instance.new("TextButton", ItemFrame)
        StarBtn.Size, StarBtn.Position, StarBtn.BackgroundTransparency = UDim2.new(0, 30, 1, 0), UDim2.new(1, -30, 0, 0), 1
        StarBtn.TextSize = 12
        StarBtn.Visible = isEditMode
        
        if showingFavorites then
            StarBtn.Text = "⭐"
            StarBtn.TextColor3 = Color3.fromRGB(255, 215, 0)
        else
            StarBtn.Text = "☆"
            StarBtn.TextColor3 = Color3.fromRGB(150, 150, 160)
        end
        
        Btn.MouseButton1Click:Connect(function()
            selectedSong = song
            updateListVisuals()
        end)
        
        StarBtn.MouseButton1Click:Connect(function()
            local newPath = ""
            if showingFavorites then newPath = moveFile(song.path, "My MIDI") else newPath = moveFile(song.path, "MIDI Favorites") end
            if selectedSong and selectedSong.name == song.name then selectedSong.path = newPath end
            if playingSong and playingSong.name == song.name then playingSong.path = newPath end
            buildList() 
        end)
        
        Btn.MouseEnter:Connect(function() 
            local isPlayingThis = (playingSong and playingSong.name == song.name)
            local isSelectedThis = (selectedSong and selectedSong.name == song.name)
            if not isPlayingThis and not isSelectedThis then 
                TweenService:Create(ItemFrame, TweenInfo.new(0.15), {BackgroundColor3 = Color3.fromRGB(90, 90, 95)}):Play() 
            end 
        end)
        Btn.MouseLeave:Connect(function() updateListVisuals() end)
        
        songItems[i] = {frame = ItemFrame, btn = Btn}
    end
    
    if count == 0 then 
        EmptyLbl.Visible = true
        EmptyLbl.Text = showingFavorites and "No favorite songs yet" or "No MIDI files found"
        Scroll.CanvasSize = UDim2.new(0, 0, 0, 50) 
    else 
        Scroll.CanvasSize = UDim2.new(0, 0, 0, count * 30 + 10) 
    end
    updateListVisuals()
end

RefreshBtn.MouseButton1Click:Connect(function() RefreshBtn.Text = "..."; task.wait(0.1); buildList(); RefreshBtn.Text = "Refresh" end)

EditBtn.MouseButton1Click:Connect(function()
    isEditMode = not isEditMode
    EditBtn.Text = isEditMode and "✅ Done" or "✏️ Edit"
    EditBtn.BackgroundColor3 = isEditMode and Color3.fromRGB(10, 132, 255) or Color3.fromRGB(75, 75, 80)
    buildList()
end)

FavViewBtn.MouseButton1Click:Connect(function()
    showingFavorites = not showingFavorites
    FavViewBtn.Text = showingFavorites and "⭐ Favs" or "📜 All"
    FavViewBtn.BackgroundColor3 = showingFavorites and Color3.fromRGB(10, 132, 255) or Color3.fromRGB(75, 75, 80)
    buildList()
end)

ToggleBtn2.MouseButton1Click:Connect(function() 
    blockMenus = not blockMenus; 
    ToggleBtn2.Text = blockMenus and "☑ Block Menus" or "☐ Block Menus"; 
    ToggleBtn2.TextColor3 = blockMenus and Color3.fromRGB(255, 100, 100) or Color3.fromRGB(120, 120, 130)
    if isPlaying then setGameBlocker(true) end
end)

function updatePlayBtn()
    if isPlaying and not isPaused then PlayBtn.Text = "⏸ Pause"
    else PlayBtn.Text = "▶ Play" end
end

PlayBtn.MouseButton1Click:Connect(function()
    if not isPlaying then
        if not selectedSong then setStatus("Please select a MIDI file first!", Color3.fromRGB(255, 100, 100)); return end
        
        NPName.Text = selectedSong.name
        playingSong = selectedSong
        updateListVisuals()
        
        setStatus("Parsing...", Color3.fromRGB(200, 200, 200))
        task.spawn(function()
            local data; pcall(function() data = readfile(selectedSong.path) end)
            if not data or #data < 14 then setStatus("File read error", Color3.fromRGB(255, 100, 100)); return end
            local frames = parseMidi(data)
            if not frames then setStatus("Corrupt MIDI", Color3.fromRGB(255, 100, 100)); return end
            TimeLabel.Text = "00:00 / " .. formatTime(totalSongTime)
            setStatus("♪ Playing...", Color3.fromRGB(100, 255, 150))
            updatePlayBtn()
            playSong(frames, function() updatePlayBtn(); NPName.Text = "..."; setStatus("Finished", Color3.fromRGB(150, 150, 160)) end)
        end)
    elseif isPlaying and not isPaused then 
        isPaused = true; setStatus("⏸ Paused", Color3.fromRGB(255, 200, 100))
    elseif isPlaying and isPaused then 
        isPaused = false; setStatus("♪ Playing...", Color3.fromRGB(100, 255, 150)) 
    end
    updatePlayBtn()
end)

StopBtn.MouseButton1Click:Connect(function() stopSong(); updatePlayBtn(); NPName.Text = "..."; setStatus("⏹ Stopped", Color3.fromRGB(255, 100, 100)) end)

RewindBtn.MouseButton1Click:Connect(function() seekToTime(currentPlayTime - seekAmount) end)
ForwardBtn.MouseButton1Click:Connect(function() seekToTime(currentPlayTime + seekAmount) end)

-- PERFECT SPEED FIX: No longer restarts the song thread. Scales the time perfectly on-the-fly!
SpeedDown.MouseButton1Click:Connect(function() 
    playSpeed = math.max(0.05, playSpeed - 0.05); 
    SpeedLbl.Text = string.format("Speed: %d%%", math.floor(playSpeed * 100 + 0.5))
end)
SpeedUp.MouseButton1Click:Connect(function() 
    playSpeed = math.min(10.0, playSpeed + 0.05); 
    SpeedLbl.Text = string.format("Speed: %d%%", math.floor(playSpeed * 100 + 0.5))
end)

SeekDown.MouseButton1Click:Connect(function() seekAmount = math.max(1, seekAmount - 1); SeekLbl.Text = "Seek: ±"..seekAmount.."s" end)
SeekUp.MouseButton1Click:Connect(function() seekAmount = math.min(300, seekAmount + 1); SeekLbl.Text = "Seek: ±"..seekAmount.."s" end)

-- ============================================
-- INFO POPUP OVERLAY
-- ============================================
local InfoOverlay = Instance.new("Frame", Main)
InfoOverlay.Size, InfoOverlay.BackgroundColor3, InfoOverlay.BackgroundTransparency, InfoOverlay.ZIndex, InfoOverlay.Visible = UDim2.new(1, 0, 1, 0), Color3.fromRGB(0, 0, 0), 0.5, 60, false
Instance.new("UICorner", InfoOverlay).CornerRadius = UDim.new(0, 10)

local InfoBox = Instance.new("Frame", InfoOverlay)
InfoBox.Size, InfoBox.Position, InfoBox.BackgroundColor3, InfoBox.ZIndex = UDim2.new(0, 250, 0, 160), UDim2.new(0.5, -125, 0.5, -80), Color3.fromRGB(40, 40, 44), 61
Instance.new("UICorner", InfoBox).CornerRadius = UDim.new(0, 10); local iStroke = Instance.new("UIStroke", InfoBox); iStroke.Color = Color3.fromRGB(20, 20, 20); iStroke.Thickness = 1

local InfoTitle = Instance.new("TextLabel", InfoBox)
InfoTitle.Size, InfoTitle.Position, InfoTitle.BackgroundTransparency, InfoTitle.ZIndex = UDim2.new(1, 0, 0, 30), UDim2.new(0, 0, 0, 5), 1, 62
InfoTitle.Text, InfoTitle.TextColor3, InfoTitle.Font, InfoTitle.TextSize = "ℹ️ About Block Menus", Color3.fromRGB(255, 200, 100), Enum.Font.GothamBold, 13

local InfoDesc = Instance.new("TextLabel", InfoBox)
InfoDesc.Size, InfoDesc.Position, InfoDesc.BackgroundTransparency, InfoDesc.ZIndex = UDim2.new(1, -20, 0, 80), UDim2.new(0, 10, 0, 35), 1, 62
InfoDesc.Text = "Enabling this prevents the game's menus and combat skills from opening while playing.\n\n⚠️ IMPORTANT: In many games, enabling this ALSO blocks the piano from working! If the piano breaks, turn this OFF."
InfoDesc.TextColor3, InfoDesc.Font, InfoDesc.TextSize, InfoDesc.TextWrapped, InfoDesc.TextYAlignment = Color3.fromRGB(220, 220, 220), Enum.Font.Gotham, 11, true, Enum.TextYAlignment.Top

local CloseInfoBtn = Instance.new("TextButton", InfoBox)
CloseInfoBtn.Size, CloseInfoBtn.Position, CloseInfoBtn.ZIndex, CloseInfoBtn.Text = UDim2.new(0, 100, 0, 30), UDim2.new(0.5, -50, 1, -35), 62, "Understood"
styleMacButton(CloseInfoBtn, true)

InfoBtn.MouseButton1Click:Connect(function() InfoOverlay.Visible = true end)
CloseInfoBtn.MouseButton1Click:Connect(function() InfoOverlay.Visible = false end)

-- ============================================
-- EXIT WARNING OVERLAY
-- ============================================
local WarningOverlay = Instance.new("Frame", Main)
WarningOverlay.Size, WarningOverlay.BackgroundColor3, WarningOverlay.BackgroundTransparency, WarningOverlay.ZIndex, WarningOverlay.Visible = UDim2.new(1, 0, 1, 0), Color3.fromRGB(0, 0, 0), 0.5, 50, false
Instance.new("UICorner", WarningOverlay).CornerRadius = UDim.new(0, 10)

local WarningBox = Instance.new("Frame", WarningOverlay)
WarningBox.Size, WarningBox.Position, WarningBox.BackgroundColor3, WarningBox.ZIndex = UDim2.new(0, 240, 0, 110), UDim2.new(0.5, -120, 0.5, -55), Color3.fromRGB(40, 40, 44), 51
Instance.new("UICorner", WarningBox).CornerRadius = UDim.new(0, 10); local wStroke = Instance.new("UIStroke", WarningBox); wStroke.Color = Color3.fromRGB(20, 20, 20); wStroke.Thickness = 1

local WarningText = Instance.new("TextLabel", WarningBox)
WarningText.Size, WarningText.Position, WarningText.BackgroundTransparency, WarningText.ZIndex = UDim2.new(1, -20, 0, 50), UDim2.new(0, 10, 0, 10), 1, 52
WarningText.Text, WarningText.TextColor3, WarningText.Font, WarningText.TextSize, WarningText.TextWrapped = "Are you sure you want to exit?", Color3.fromRGB(240, 240, 240), Enum.Font.GothamSemibold, 13, true

local YesBtn = Instance.new("TextButton", WarningBox)
YesBtn.Size, YesBtn.Position, YesBtn.Text, YesBtn.ZIndex = UDim2.new(0, 90, 0, 30), UDim2.new(0, 20, 1, -40), "Exit", 52
styleMacButton(YesBtn, false)
YesBtn.BackgroundColor3 = Color3.fromRGB(200, 50, 50)
YesBtn.MouseEnter:Connect(function() TweenService:Create(YesBtn, TweenInfo.new(0.15), {BackgroundColor3 = Color3.fromRGB(220, 70, 70)}):Play() end)
YesBtn.MouseLeave:Connect(function() TweenService:Create(YesBtn, TweenInfo.new(0.15), {BackgroundColor3 = Color3.fromRGB(200, 50, 50)}):Play() end)

local NoBtn = Instance.new("TextButton", WarningBox)
NoBtn.Size, NoBtn.Position, NoBtn.Text, NoBtn.ZIndex = UDim2.new(0, 90, 0, 30), UDim2.new(1, -110, 1, -40), "Cancel", 52
styleMacButton(NoBtn, true)

CloseBtn.MouseButton1Click:Connect(function()
    if isMinimized then isMinimized = false; Content.Visible = true; TweenService:Create(Main, TweenInfo.new(0.25), {Size = UDim2.new(0, 280, 0, 485)}):Play(); task.wait(0.25) end
    WarningOverlay.Visible = true
end)

NoBtn.MouseButton1Click:Connect(function() WarningOverlay.Visible = false end)
YesBtn.MouseButton1Click:Connect(function() stopSong(); ScreenGui:Destroy() end)
MinBtn.MouseButton1Click:Connect(function() isMinimized = not isMinimized; Content.Visible = not isMinimized; TweenService:Create(Main, TweenInfo.new(0.25), {Size = UDim2.new(0, 280, 0, isMinimized and 38 or 485)}):Play() end)

buildList()
