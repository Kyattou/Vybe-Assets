local RunService=game:GetService("RunService")
local UIS=game:GetService("UserInputService")
local Players=game:GetService("Players")
local WS=game:GetService("Workspace")
local Http=game:GetService("HttpService")
local Lighting=game:GetService("Lighting")
local TweenService=game:GetService("TweenService")
local StarterGui=game:GetService("StarterGui")
local LP=Players.LocalPlayer

local VERSION = "2.1"
local LICENSE_REPO="https://raw.githubusercontent.com/Kyattou/Keys-/refs/heads/main/license.json" 
local FEATURE_REPO="https://raw.githubusercontent.com/Kyattou/Keys-/refs/heads/main/features.json"

-- =============================================
-- UNIVERSAL ACCESS: Everyone gets UNLIMITED tier
-- =============================================
local LicenseData={
	valid=true,
	tier="UNLIMITED",
	features={},
	expiry="",
	username=""
}
-- =============================================
-- IRL TIME SYNC — Syncs Roblox Lighting to real local time
-- =============================================
local TimeSyncSys = {
    active = false,
    conn = nil,
    UPDATE_INTERVAL = 60.0, -- seconds between full re-syncs
    lastSync = 0,
    displayGui = nil,
}

-- =============================================
-- AD BLOCKER MODULE
-- Finds and destroys any object named "Ads"
-- across the entire game hierarchy, with live
-- monitoring for dynamically added Ad objects.
-- =============================================
local AdBlocker = {
	active      = false,
	conn        = nil,
	scanConn    = nil,
	destroyed   = 0,
	SCAN_INTERVAL = 2.0,    -- seconds between full re-scans
	lastScan    = 0,

	-- Any object whose name matches one of these (case-insensitive) will be removed.
	-- Extend this list freely.
	TARGET_NAMES = {
		"Ads",
		"ads",
		"ADS",
		"Ad",
		"ad",
		"Advertisement",
		"advertisement",
		"AdvertGUI",
		"AdFrame",
		"AdBanner",
	},
}

-- Returns true if the object's name matches any TARGET_NAME.
local function isAdObject(obj)
	local name = tostring(obj.Name)
	for _, target in ipairs(AdBlocker.TARGET_NAMES) do
		if name == target then
			return true
		end
	end
	-- Also catch any name that starts with "Ad" followed by a capital (e.g. AdSomething)
	if name:match("^[Aa][Dd][A-Z]") then
		return true
	end
	return false
end

-- Safely destroy a single object and log it.
local function nukeObject(obj)
	safeCall(function()
		if obj and obj.Parent then
			local path = obj:GetFullName()
			obj:Destroy()
			AdBlocker.destroyed = AdBlocker.destroyed + 1
			notify("Ad Blocker", "Removed: " .. path)
			warn("[UFB AdBlocker] Destroyed: " .. path)
		end
	end, "ADBLOCKER_NUKE")
end

-- Scan a container (and all descendants) for Ad objects.
local function scanContainer(container)
	safeCall(function()
		if not container or not container.Parent then return end
		-- GetDescendants is safe even on very large trees
		for _, obj in ipairs(container:GetDescendants()) do
			if isAdObject(obj) then
				nukeObject(obj)
			end
		end
		-- Also check the container itself
		if isAdObject(container) then
			nukeObject(container)
		end
	end, "ADBLOCKER_SCAN")
end

-- Full-world sweep: Workspace, PlayerGui, StarterGui, Lighting, ReplicatedStorage, etc.
local function fullScan()
	local t = tick()
	if t - AdBlocker.lastScan < AdBlocker.SCAN_INTERVAL then return end
	AdBlocker.lastScan = t

	safeCall(function()
		-- Scan the entire game tree
		for _, obj in ipairs(game:GetDescendants()) do
			if isAdObject(obj) then
				nukeObject(obj)
			end
		end
	end, "ADBLOCKER_FULL_SCAN")
end

-- Watch for any object being added to the game in real time.
local function hookDescendantAdded(container)
	safeCall(function()
		if not container then return end
		container.DescendantAdded:Connect(function(obj)
			if isAdObject(obj) then
				-- Slight yield so the object is fully parented before we nuke it.
				task.defer(function()
					nukeObject(obj)
				end)
			end
		end)
	end, "ADBLOCKER_HOOK")
end

function AdBlocker:Enable()
	if self.active then return end
	self.active  = true
	self.destroyed = 0

	-- Immediate first sweep
	safeCall(function() fullScan() end, "ADBLOCKER_INIT_SCAN")

	-- Real-time monitoring: hook the top-level containers that matter most.
	local containers = {
		game,
		WS,
		game:GetService("ReplicatedStorage"),
		game:GetService("StarterGui"),
		game:GetService("Players"),
		game:GetService("Lighting"),
	}
	-- Also hook the local PlayerGui once it exists.
	safeCall(function()
		local pg = LP:WaitForChild("PlayerGui", 5)
		if pg then table.insert(containers, pg) end
	end, "ADBLOCKER_PG_WAIT")

	for _, c in ipairs(containers) do
		hookDescendantAdded(c)
	end

	-- Periodic re-scan (catches anything that slipped through)
	self.scanConn = RunService.Heartbeat:Connect(function()
		fullScan()
	end)

	notify("Ad Blocker", "Active — monitoring for Ad objects")
	warn("[UFB AdBlocker] Enabled. Watching for Ad objects.")
end

function AdBlocker:Disable()
	if not self.active then return end
	self.active = false
	if self.scanConn then
		self.scanConn:Disconnect()
		self.scanConn = nil
	end
	notify("Ad Blocker", "Disabled — removed " .. self.destroyed .. " object(s) this session")
end

function AdBlocker:Toggle()
	if self.active then self:Disable() else self:Enable() end
end

function AdBlocker:Stats()
	print("═══ UFB AdBlocker Stats ═══")
	print("Active:    " .. tostring(self.active))
	print("Destroyed: " .. self.destroyed .. " object(s) this session")
	print("Targets:   " .. table.concat(self.TARGET_NAMES, ", "))
end
local function getRealHour()
    -- os.time() returns UTC epoch. We use the client's local offset.
    -- Roblox doesn't expose timezone, so we derive local time from os.date("*t")
    -- which respects the OS locale on PC. On mobile it may return UTC.
    local t = os.date("*t")
    return t.hour, t.min, t.sec
end

local function hoursToLightingClock(h, m, s)
    -- Roblox Lighting.ClockTime is 0–24 float
    return h + (m / 60) + (s / 3600)
end

local function applyTimeToLighting(clockTime)
    safeCall(function()
        -- Clamp to valid range
        clockTime = clockTime % 24

        -- Set clock
        Lighting.ClockTime = clockTime

        -- Optional: adjust ambient & brightness to feel natural
        local isDaytime = clockTime >= 6 and clockTime < 20
        local isDawn    = clockTime >= 5 and clockTime < 8
        local isDusk    = clockTime >= 18 and clockTime < 21
        local isNight   = clockTime >= 20 or clockTime < 5

        if isDawn or isDusk then
            -- Golden hour
            TweenService:Create(Lighting, TweenInfo.new(4), {
                Brightness = 1.2,
                Ambient = Color3.fromRGB(80, 55, 40),
                OutdoorAmbient = Color3.fromRGB(120, 90, 60),
            }):Play()
        elseif isDaytime then
            TweenService:Create(Lighting, TweenInfo.new(4), {
                Brightness = 2.0,
                Ambient = Color3.fromRGB(70, 70, 80),
                OutdoorAmbient = Color3.fromRGB(130, 140, 160),
            }):Play()
        elseif isNight then
            TweenService:Create(Lighting, TweenInfo.new(4), {
                Brightness = 0.2,
                Ambient = Color3.fromRGB(20, 20, 35),
                OutdoorAmbient = Color3.fromRGB(15, 15, 30),
            }):Play()
        end
    end, "TIME_SYNC_APPLY")
end

local function buildTimeSyncHud()
    safeCall(function()
        local pg = LP:WaitForChild("PlayerGui")
        local old = pg:FindFirstChild("UFB_TimeHud") if old then old:Destroy() end
        local sg = Instance.new("ScreenGui")
        sg.Name = "UFB_TimeHud"
        sg.ResetOnSpawn = false
        sg.DisplayOrder = 9997
        sg.IgnoreGuiInset = true
        sg.Parent = pg

        local frame = Instance.new("Frame")
        frame.AnchorPoint = Vector2.new(1, 1)
        frame.Size = UDim2.new(0, 160, 0, 28)
        frame.Position = UDim2.new(1, -12, 1, -48)
        frame.BackgroundColor3 = Color3.fromRGB(6, 6, 6)
        frame.BackgroundTransparency = 0.30
        frame.BorderSizePixel = 0
        frame.Parent = sg
        Instance.new("UICorner", frame).CornerRadius = UDim.new(0, 15)
        local us = Instance.new("UIStroke", frame)
        us.Color = Color3.fromRGB(45, 45, 45)
        us.Thickness = 1

        -- Clock dot
        local dot = Instance.new("Frame")
        dot.Size = UDim2.new(0, 6, 0, 6)
        dot.Position = UDim2.new(0, 12, 0.5, -3)
        dot.BackgroundColor3 = Color3.fromRGB(100, 180, 255)
        dot.BorderSizePixel = 0
        dot.Parent = frame
        Instance.new("UICorner", dot).CornerRadius = UDim.new(1, 0)

        local lbl = Instance.new("TextLabel")
        lbl.Name = "TimeLabel"
        lbl.Size = UDim2.new(1, -26, 1, 0)
        lbl.Position = UDim2.new(0, 26, 0, 0)
        lbl.BackgroundTransparency = 1
        lbl.Text = "TIME SYNC"
        lbl.TextColor3 = Color3.fromRGB(210, 210, 210)
        lbl.TextSize = 10
        lbl.Font = Enum.Font.GothamBold
        lbl.TextXAlignment = Enum.TextXAlignment.Left
        lbl.Parent = frame

        TimeSyncSys.displayGui = sg
        TimeSyncSys.label = lbl
        TimeSyncSys.dot = dot
    end, "BUILD_TIME_HUD")
end

local function destroyTimeSyncHud()
    if TimeSyncSys.displayGui and TimeSyncSys.displayGui.Parent then
        TimeSyncSys.displayGui:Destroy()
    end
    TimeSyncSys.displayGui = nil
    TimeSyncSys.label = nil
    TimeSyncSys.dot = nil
end

local function formatTime(h, m, s)
    local suffix = h >= 12 and "PM" or "AM"
    local h12 = h % 12
    if h12 == 0 then h12 = 12 end
    return string.format("%d:%02d:%02d %s", h12, m, s, suffix)
end

local function updateTimeSyncLabel()
    if not TimeSyncSys.label then return end
    safeCall(function()
        local h, m, s = getRealHour()
        TimeSyncSys.label.Text = formatTime(h, m, s)
        -- Pulse dot to show it's live
        TweenService:Create(TimeSyncSys.dot, TweenInfo.new(0.15), {BackgroundTransparency = 0.1}):Play()
        task.delay(0.2, function()
            if TimeSyncSys.dot and TimeSyncSys.dot.Parent then
                TweenService:Create(TimeSyncSys.dot, TweenInfo.new(0.4), {BackgroundTransparency = 0.0}):Play()
            end
        end)
    end, "TIME_LABEL_UPDATE")
end

function TimeSyncSys:Enable()
    if self.active then return end
    self.active = true

    -- Immediately apply current time
    safeCall(function()
        local h, m, s = getRealHour()
        local clockTime = hoursToLightingClock(h, m, s)
        applyTimeToLighting(clockTime)
        notify("Time Sync", "Synced to " .. formatTime(h, m, s))
    end, "TIME_SYNC_INIT")

    buildTimeSyncHud()

    -- Second-by-second ticker (updates the HUD label every second, re-syncs Lighting every minute)
    self.conn = RunService.Heartbeat:Connect(function()
        safeCall(function()
            local now = tick()
            local h, m, s = getRealHour()

            -- Update HUD label every second (cheap string update)
            if TimeSyncSys.label then
                updateTimeSyncLabel()
            end

            -- Re-sync Lighting.ClockTime every full minute to stay accurate
            if now - TimeSyncSys.lastSync >= TimeSyncSys.UPDATE_INTERVAL then
                TimeSyncSys.lastSync = now
                local clockTime = hoursToLightingClock(h, m, s)
                applyTimeToLighting(clockTime)
            else
                -- Smooth tick: advance ClockTime by one real second each second
                -- This keeps the sun moving smoothly between re-syncs
                Lighting.ClockTime = hoursToLightingClock(h, m, s)
            end
        end, "TIME_SYNC_TICK")
    end)

    notify("Time Sync", "IRL clock active")
end

function TimeSyncSys:Disable()
    if not self.active then return end
    self.active = false
    if self.conn then self.conn:Disconnect() self.conn = nil end
    destroyTimeSyncHud()
    notify("Time Sync", "Disabled")
end

function TimeSyncSys:Toggle()
    if self.active then self:Disable() else self:Enable() end
end

local function safeCall(fn,tag)
	local ok,err=pcall(fn)
	if not ok then
		warn("[UFB "..VERSION.."] "..tostring(tag)..": "..tostring(err))
		return false,tostring(err)
	end
	return true
end

local function lerp(a,b,t) return a+(b-a)*t end
local function lerpCF(a,b,t) return a:Lerp(b,t) end
local function clamp(v,mn,mx) return math.max(mn,math.min(mx,v)) end
local function dtSmooth(alpha,dt) return 1-(1-clamp(alpha,0,0.9999))^(dt*60) end
local function dist3(a,b) return (a-b).Magnitude end

local function getHRP(p)
	if not p or not p.Character then return nil end
	return p.Character:FindFirstChild("HumanoidRootPart")
end

local function getHead(p)
	if not p or not p.Character then return nil end
	return p.Character:FindFirstChild("Head")
end

local LoadingScreen={gui=nil}

function LoadingScreen:Show()
	safeCall(function()
		local pg=LP:WaitForChild("PlayerGui")
		local old=pg:FindFirstChild("UFB_Loading")
		if old then old:Destroy() end

		local sg=Instance.new("ScreenGui")
		sg.Name="UFB_Loading"
		sg.ResetOnSpawn=false
		sg.DisplayOrder=99999
		sg.IgnoreGuiInset=true
		sg.Parent=pg

		-- Full black canvas
		local bg=Instance.new("Frame")
		bg.Size=UDim2.new(1,0,1,0)
		bg.BackgroundColor3=Color3.fromRGB(0,0,0)
		bg.BorderSizePixel=0
		bg.Parent=sg

		-- Decorative grid lines (horizontal)
		for i=1,8 do
			local line=Instance.new("Frame")
			line.Size=UDim2.new(1,0,0,1)
			line.Position=UDim2.new(0,0,i/9,0)
			line.BackgroundColor3=Color3.fromRGB(255,255,255)
			line.BackgroundTransparency=0.96
			line.BorderSizePixel=0
			line.Parent=bg
		end
		-- Decorative grid lines (vertical)
		for i=1,12 do
			local line=Instance.new("Frame")
			line.Size=UDim2.new(0,1,1,0)
			line.Position=UDim2.new(i/13,0,0,0)
			line.BackgroundColor3=Color3.fromRGB(255,255,255)
			line.BackgroundTransparency=0.96
			line.BorderSizePixel=0
			line.Parent=bg
		end

		-- Top-left corner mark
		local tlCorner=Instance.new("Frame")
		tlCorner.Size=UDim2.new(0,24,0,1)
		tlCorner.Position=UDim2.new(0,32,0,32)
		tlCorner.BackgroundColor3=Color3.fromRGB(255,255,255)
		tlCorner.BackgroundTransparency=0.6
		tlCorner.BorderSizePixel=0
		tlCorner.Parent=bg
		local tlCornerV=Instance.new("Frame")
		tlCornerV.Size=UDim2.new(0,1,0,24)
		tlCornerV.Position=UDim2.new(0,32,0,32)
		tlCornerV.BackgroundColor3=Color3.fromRGB(255,255,255)
		tlCornerV.BackgroundTransparency=0.6
		tlCornerV.BorderSizePixel=0
		tlCornerV.Parent=bg

		-- Bottom-right corner mark
		local brCorner=Instance.new("Frame")
		brCorner.Size=UDim2.new(0,24,0,1)
		brCorner.Position=UDim2.new(1,-56,1,-32)
		brCorner.BackgroundColor3=Color3.fromRGB(255,255,255)
		brCorner.BackgroundTransparency=0.6
		brCorner.BorderSizePixel=0
		brCorner.Parent=bg
		local brCornerV=Instance.new("Frame")
		brCornerV.Size=UDim2.new(0,1,0,24)
		brCornerV.Position=UDim2.new(1,-32,1,-56)
		brCornerV.BackgroundColor3=Color3.fromRGB(255,255,255)
		brCornerV.BackgroundTransparency=0.6
		brCornerV.BorderSizePixel=0
		brCornerV.Parent=bg

		-- Version tag top-right
		local verTag=Instance.new("TextLabel")
		verTag.Size=UDim2.new(0,120,0,14)
		verTag.Position=UDim2.new(1,-152,0,34)
		verTag.BackgroundTransparency=1
		verTag.Text="v"..VERSION.." / UNLIMITED"
		verTag.TextColor3=Color3.fromRGB(255,255,255)
		verTag.TextTransparency=0.7
		verTag.TextSize=9
		verTag.Font=Enum.Font.GothamBold
		verTag.TextXAlignment=Enum.TextXAlignment.Right
		verTag.Parent=bg

		-- Center container — positioned slightly above center
		local center=Instance.new("Frame")
		center.Size=UDim2.new(0,480,0,220)
		center.Position=UDim2.new(0.5,-240,0.5,-140)
		center.BackgroundTransparency=1
		center.Parent=bg

		-- Thin accent line above title
		local accentLine=Instance.new("Frame")
		accentLine.Size=UDim2.new(0,0,0,1)
		accentLine.Position=UDim2.new(0.5,0,0,0)
		accentLine.AnchorPoint=Vector2.new(0.5,0)
		accentLine.BackgroundColor3=Color3.fromRGB(255,255,255)
		accentLine.BorderSizePixel=0
		accentLine.Parent=center

		-- Main logo
		local logo=Instance.new("TextLabel")
		logo.Size=UDim2.new(1,0,0,52)
		logo.Position=UDim2.new(0,0,0,14)
		logo.BackgroundTransparency=1
		logo.Text="UFBARSTOOL"
		logo.TextColor3=Color3.fromRGB(255,255,255)
		logo.TextTransparency=1
		logo.TextSize=48
		logo.Font=Enum.Font.GothamBold
		logo.TextXAlignment=Enum.TextXAlignment.Center
		logo.Parent=center

		-- Subtitle
		local sub=Instance.new("TextLabel")
		sub.Size=UDim2.new(1,0,0,16)
		sub.Position=UDim2.new(0,0,0,70)
		sub.BackgroundTransparency=1
		sub.Text="BROADCAST CAMERA SYSTEM"
		sub.TextColor3=Color3.fromRGB(255,255,255)
		sub.TextTransparency=1
		sub.TextSize=10
		sub.Font=Enum.Font.GothamBold
		sub.TextXAlignment=Enum.TextXAlignment.Center
		sub.Parent=center

		-- Thin divider
		local divider=Instance.new("Frame")
		divider.Size=UDim2.new(0,1,0,1)
		divider.Position=UDim2.new(0.5,0,0,96)
		divider.AnchorPoint=Vector2.new(0.5,0)
		divider.BackgroundColor3=Color3.fromRGB(255,255,255)
		divider.BackgroundTransparency=0.6
		divider.BorderSizePixel=0
		divider.Parent=center

		-- Progress bar background
		local barBg=Instance.new("Frame")
		barBg.Size=UDim2.new(0,320,0,1)
		barBg.Position=UDim2.new(0.5,-160,0,118)
		barBg.BackgroundColor3=Color3.fromRGB(255,255,255)
		barBg.BackgroundTransparency=0.88
		barBg.BorderSizePixel=0
		barBg.Parent=center

		-- Progress bar fill
		local barFill=Instance.new("Frame")
		barFill.Size=UDim2.new(0,0,1,0)
		barFill.BackgroundColor3=Color3.fromRGB(255,255,255)
		barFill.BorderSizePixel=0
		barFill.Parent=barBg

		-- Progress percentage label
		local pctLbl=Instance.new("TextLabel")
		pctLbl.Size=UDim2.new(0,50,0,14)
		pctLbl.Position=UDim2.new(1,4,0.5,-7)
		pctLbl.BackgroundTransparency=1
		pctLbl.Text="0%"
		pctLbl.TextColor3=Color3.fromRGB(255,255,255)
		pctLbl.TextTransparency=0.5
		pctLbl.TextSize=8
		pctLbl.Font=Enum.Font.GothamBold
		pctLbl.TextXAlignment=Enum.TextXAlignment.Left
		pctLbl.Parent=barBg

		-- Status label
		local status=Instance.new("TextLabel")
		status.Name="Status"
		status.Size=UDim2.new(1,0,0,14)
		status.Position=UDim2.new(0,0,0,130)
		status.BackgroundTransparency=1
		status.Text="INITIALIZING"
		status.TextColor3=Color3.fromRGB(255,255,255)
		status.TextTransparency=0.65
		status.TextSize=8
		status.Font=Enum.Font.GothamBold
		status.TextXAlignment=Enum.TextXAlignment.Center
		status.Parent=center

		-- Scanning dots row
		local dotsRow=Instance.new("Frame")
		dotsRow.Size=UDim2.new(0,60,0,8)
		dotsRow.Position=UDim2.new(0.5,-30,0,155)
		dotsRow.BackgroundTransparency=1
		dotsRow.Parent=center

		local dots={}
		for i=1,5 do
			local dot=Instance.new("Frame")
			dot.Size=UDim2.new(0,4,0,4)
			dot.Position=UDim2.new(0,(i-1)*12,0.5,-2)
			dot.BackgroundColor3=Color3.fromRGB(255,255,255)
			dot.BackgroundTransparency=0.85
			dot.BorderSizePixel=0
			dot.Parent=dotsRow
			Instance.new("UICorner",dot).CornerRadius=UDim.new(1,0)
			dots[i]=dot
		end

		-- Bottom-left system info
		local sysInfo=Instance.new("TextLabel")
		sysInfo.Size=UDim2.new(0,200,0,14)
		sysInfo.Position=UDim2.new(0,32,1,-36)
		sysInfo.BackgroundTransparency=1
		sysInfo.Text="UFB/"..LP.Name
		sysInfo.TextColor3=Color3.fromRGB(255,255,255)
		sysInfo.TextTransparency=0.75
		sysInfo.TextSize=8
		sysInfo.Font=Enum.Font.GothamBold
		sysInfo.TextXAlignment=Enum.TextXAlignment.Left
		sysInfo.Parent=bg

		self.gui=sg
		self._barFill=barFill
		self._pctLbl=pctLbl
		self._status=status
		self._bg=bg
		self._dots=dots
		self._logo=logo
		self._sub=sub
		self._divider=divider
		self._accentLine=accentLine

		-- Entrance animations
		task.spawn(function()
			task.wait(0.05)
			-- Expand accent line
			TweenService:Create(accentLine,TweenInfo.new(0.5,Enum.EasingStyle.Quart,Enum.EasingDirection.Out),
				{Size=UDim2.new(0,320,0,1),Position=UDim2.new(0.5,-160,0,0)}):Play()
			task.wait(0.2)
			-- Expand divider
			TweenService:Create(divider,TweenInfo.new(0.4,Enum.EasingStyle.Quad,Enum.EasingDirection.Out),
				{Size=UDim2.new(0,320,0,1),Position=UDim2.new(0.5,-160,0,96)}):Play()
			task.wait(0.1)
			-- Fade in logo
			TweenService:Create(logo,TweenInfo.new(0.6,Enum.EasingStyle.Quad),{TextTransparency=0}):Play()
			task.wait(0.15)
			-- Fade in sub
			TweenService:Create(sub,TweenInfo.new(0.5,Enum.EasingStyle.Quad),{TextTransparency=0.35}):Play()
		end)

		-- Animated scanning dots
		task.spawn(function()
			local idx=1
			while self.gui and self.gui.Parent do
				if dots[idx] and dots[idx].Parent then
					TweenService:Create(dots[idx],TweenInfo.new(0.18,Enum.EasingStyle.Quad),{BackgroundTransparency=0.1}):Play()
					task.wait(0.12)
					TweenService:Create(dots[idx],TweenInfo.new(0.4,Enum.EasingStyle.Quad),{BackgroundTransparency=0.85}):Play()
				end
				idx=(idx%5)+1
				task.wait(0.1)
			end
		end)
	end,"LOADING_SHOW")
end

function LoadingScreen:SetProgress(pct,msg)
	safeCall(function()
		if self._barFill then
			TweenService:Create(self._barFill,TweenInfo.new(0.4,Enum.EasingStyle.Quart,Enum.EasingDirection.Out),
				{Size=UDim2.new(pct,0,1,0)}):Play()
		end
		if self._pctLbl then
			self._pctLbl.Text=math.floor(pct*100).."%"
		end
		if self._status and msg then
			self._status.Text=string.upper(msg)
		end
	end,"LOADING_PROGRESS")
end

function LoadingScreen:Hide()
	safeCall(function()
		if not self.gui then return end
		if self._bg then
			local fade=TweenService:Create(self._bg,TweenInfo.new(0.6,Enum.EasingStyle.Quad),
				{BackgroundTransparency=1})
			fade:Play()
			for _,v in ipairs(self._bg:GetDescendants()) do
				if v:IsA("TextLabel") then
					TweenService:Create(v,TweenInfo.new(0.5),{TextTransparency=1}):Play()
				elseif v:IsA("Frame") then
					TweenService:Create(v,TweenInfo.new(0.5),{BackgroundTransparency=1}):Play()
				end
			end
			task.wait(0.7)
		end
		if self.gui and self.gui.Parent then self.gui:Destroy() end
		self.gui=nil
	end,"LOADING_HIDE")
end

-- =============================================
-- validateLicense: always returns true, sets UNLIMITED
-- =============================================
local function validateLicense()
	LicenseData.valid=true
	LicenseData.tier="UNLIMITED"
	LicenseData.features={}
	LicenseData.expiry="NEVER"
	LicenseData.username=LP.Name
	return true
end

local function hasFeature(feat)
	-- UNLIMITED tier has access to everything
	return true
end

local CFG={
	SYS={AUTO_RECOVERY=true,HEALTH_INTERVAL=5.0,MAX_ERRORS=8,ERROR_COOLDOWN=3.0,MEMORY_INTERVAL=40.0},
	SHADER={
		ENABLED=true,BRIGHTNESS=0.04,CONTRAST=0.15,SATURATION=0.20,
		TINT=Color3.fromRGB(255,253,248),
		BLOOM_INTENSITY=0.32,BLOOM_SIZE=20,BLOOM_THRESHOLD=1.28,
		SUNRAY_INTENSITY=0.10,SUNRAY_SPREAD=0.40,
		DOF_FOCUS=55,DOF_RADIUS=30,DOF_NEAR=0.20,DOF_FAR=0.40,
		ATM_DENSITY=0.28,ATM_COLOR=Color3.fromRGB(200,222,248),
		ATM_DECAY=Color3.fromRGB(108,114,128),ATM_GLARE=0.20,ATM_HAZE=1.4,
		UPDATE_INTERVAL=0.25,
	},
	AI={
		DECISION_INTERVAL=0.10,MIN_SHOT_DURATION=1.6,MAX_SHOT_DURATION=12.0,
		CONFIDENCE_THRESHOLD=0.52,HYSTERESIS=1.35,FALLBACK="BROADCAST",
		URGENCY_OVERRIDE=0.82,MOMENTUM_WEIGHT=0.15,HISTORY_WEIGHT=0.10,
		VARIETY_BONUS=0.08,TRANSITION_SMOOTH=0.12,
	},
	PLAYER={
		ROLE_INTERVAL=0.25,QB_PAD_DIST=14,VEL_SMOOTH=0.12,
		HISTORY_SIZE=48,BALL_CARRIER_RADIUS=5.5,
		SPECTATOR_NAMES={"spectator","spec","spectators","viewer","audience"},
		QBPAD_SCAN_INTERVAL=4.0,PREDICTION_FRAMES=8,
	},
	BALL={
		RESCAN_INTERVAL=0.10,
		NAMES={"Football","Ball","GameBall","football","ball","TheBall","NFL_Ball","SportsBall","Pigskin"},
		AIR_MIN_HEIGHT=4,AIR_MIN_VEL=6,KICK_HEIGHT=12,KICK_VEL=20,
		HISTORY_SIZE=30,PREDICTION_WEIGHT=0.65,CATCH_PREDICT_TIME=0.4,
		LAUNCH_DETECT_ACCEL=40,FLIGHT_SMOOTH_BOOST=0.50,
		GRAVITY=Vector3.new(0,-196.2,0),
	},
	FIELD={REDZONE_Z=60,GOALLINE_Z=14,ENDZONE_DEPTH=300,MIDFIELD_Z=30},
	DRONE={
		BASE_SPEED=38,SPRINT_MULT=3.0,SLOW_MULT=0.20,
		VERTICAL_SPEED=22,ROTATION_SPEED=0.0026,
		INERTIA=0.86,FOV=72,FOV_MIN=20,FOV_MAX=105,FOV_STEP=5,
		LOCK_SMOOTH=0.07,ORBIT_RADIUS=24,ORBIT_SPEED=0.32,ORBIT_HEIGHT=14,
		MIN_HEIGHT=1.5,MAX_HEIGHT=220,
	},
	CAM={
		BROADCAST={HEIGHT=44,DISTANCE=82,SMOOTH=0.055,FOV=66,FOLLOW_Z=18,TILT=8},
		HIGH_BROADCAST={HEIGHT=68,DISTANCE=98,SMOOTH=0.042,FOV=61,TILT=13},
		TIGHT_BROADCAST={HEIGHT=29,DISTANCE=56,SMOOTH=0.068,FOV=72,TILT=6},
		SKYCAM={HEIGHT=60,SMOOTH=0.058,FOV=55,DOWN_ANGLE=42},
		ALL_22={HEIGHT=88,SMOOTH=0.035,FOV=47,DOWN_ANGLE=50},
		QB_CAM={DIST_BEHIND=9,DIST_SIDE=4,HEIGHT=7.5,SMOOTH=0.115,FOV=72,PRESSURE_FOV=9},
		QB_CLOSEUP={DIST=5.5,HEIGHT=6.8,SMOOTH=0.14,FOV=77,SIDE=2.2},
		QB_SCRAMBLE={DIST=14,HEIGHT=9,SMOOTH=0.11,FOV=75,LEAD=12},
		POCKET_CAM={HEIGHT=9,SMOOTH=0.15,FOV=68,RADIUS=12,CIRCLE_SPEED=0.28},
		RECEIVER_CAM={DIST=17,HEIGHT=10,SMOOTH=0.16,FOV=70,LEAD=13,CATCH_DIST=9},
		RECEIVER_ISO={DIST=21,HEIGHT=12,SMOOTH=0.13,FOV=67,SIDE_ANGLE=26},
		BALL_CAM={DIST=48,HEIGHT=24,SMOOTH=0.12,FOV=62,SPEED_FOV_MULT=0.22,MAX_FOV=90},
		BALL_TRACK={DIST=34,HEIGHT=18,SMOOTH=0.12,FOV=69},
		BALL_FLIGHT={DIST=28,HEIGHT=8,SMOOTH=0.22,FOV=65,LEAD_TIME=0.35,ARC_OFFSET=6},
		BALL_SPIRAL={DIST=12,HEIGHT=4,SMOOTH=0.28,FOV=70,ORBIT_SPEED=0.8},
		INTERCEPTION={DIST=18,HEIGHT=11,SMOOTH=0.13,FOV=72,LEAD=10},
		TACKLE_CAM={DIST=12,HEIGHT=7,SMOOTH=0.18,FOV=74},
		FUMBLE_CAM={DIST=15,HEIGHT=9,SMOOTH=0.16,FOV=76},
		SACK_CAM={DIST=11,HEIGHT=7,SMOOTH=0.19,FOV=74},
		BLITZ_CAM={DIST=20,HEIGHT=10,SMOOTH=0.14,FOV=72},
		TACTICAL_CAM={HEIGHT=74,ANGLE=35,SMOOTH=0.036,FOV=50},
		KICKOFF_CAM={HEIGHT=39,DISTANCE=77,SMOOTH=0.068,FOV=64,SWITCH_DELAY=1.0},
		KICKOFF_WIDE={HEIGHT=57,DISTANCE=115,SMOOTH=0.052,FOV=57},
		KICKOFF_RETURN={DIST=22,HEIGHT=13,SMOOTH=0.12,FOV=70,LEAD=14},
		PUNT_CAM={HEIGHT=33,DISTANCE=60,SMOOTH=0.078,FOV=66},
		PUNT_HANG={HEIGHT=45,DISTANCE=50,SMOOTH=0.055,FOV=60},
		PUNT_RETURN={DIST=23,HEIGHT=14,SMOOTH=0.11,FOV=70},
		FIELD_GOAL_CAM={HEIGHT=17,DISTANCE=44,SMOOTH=0.10,FOV=68},
		FIELD_GOAL_SIDE={HEIGHT=23,DISTANCE=67,SMOOTH=0.074,FOV=63,SIDE_ANGLE=45},
		FIELD_GOAL_POSTS={HEIGHT=28,DISTANCE=35,SMOOTH=0.085,FOV=62},
		RETURN_CAM={DIST=25,HEIGHT=14,SMOOTH=0.115,FOV=70,LEAD=15},
		SIDELINE_FOLLOW={DIST=19,HEIGHT=7,SMOOTH=0.13,FOV=73},
		SIDELINE_STATIC={HEIGHT=9,SMOOTH=0.085,FOV=67},
		ENDZONE_STD={DIST=46,HEIGHT=19,SMOOTH=0.072,FOV=65},
		ENDZONE_HIGH={DIST=36,HEIGHT=37,SMOOTH=0.058,FOV=59,DOWN_ANGLE=26},
		LINE_CAM={HEIGHT=5,DIST=13,SMOOTH=0.16,FOV=74},
		PYLON_CAM={HEIGHT=2.2,DIST=8,SMOOTH=0.19,FOV=79},
		CELEBRATION_CAM={DIST=13,HEIGHT=7.5,SMOOTH=0.14,FOV=75,CIRCLE_SPEED=0.18},
		CLUSTER_CAM={HEIGHT=36,DIST=50,SMOOTH=0.048,FOV=69},
		TWO_POINT_CAM={HEIGHT=8,DIST=30,SMOOTH=0.12,FOV=70},
		ONSIDE_CAM={HEIGHT=18,DIST=40,SMOOTH=0.08,FOV=66},
		HURRY_UP_CAM={HEIGHT=32,DISTANCE=70,SMOOTH=0.075,FOV=68,TILT=7},
		FORMATION_CAM={HEIGHT=55,SMOOTH=0.040,FOV=52,DOWN_ANGLE=38},
		CATCH_CAM={DIST=8,HEIGHT=5,SMOOTH=0.24,FOV=76,SLOWMO_FACTOR=0.6},
		REPLAY_CAM={DIST=20,HEIGHT=12,SMOOTH=0.10,FOV=70,ORBIT_SPEED=0.15},
		CINEMATIC={DIST=30,HEIGHT=15,SMOOTH=0.06,FOV=58,DOLLY_SPEED=0.08},
		OVERHEAD_TRACK={HEIGHT=65,SMOOTH=0.04,FOV=50,DOWN_ANGLE=80},
		REVERSE_ANGLE={HEIGHT=38,DISTANCE=75,SMOOTH=0.055,FOV=66,TILT=8},
		LOW_ENDZONE={HEIGHT=3,DIST=12,SMOOTH=0.20,FOV=82},
		SNAP_CAM={HEIGHT=3,DIST=5,SMOOTH=0.22,FOV=80},
		COACH_CAM={HEIGHT=52,SMOOTH=0.038,FOV=48,DOWN_ANGLE=45},
		POV_CAM={
			SMOOTH=0.28,FOV=88,HEIGHT_OFFSET=1.6,LEAD=10,
			BOB_SPEED=9,BOB_AMOUNT=0.22,TILT_SPEED=0.06,MAX_TILT=4,
		},
		OVERVIEW_CAM={
			HEIGHT_BASE=54,HEIGHT_SCALE=0.58,SMOOTH=0.036,
			FOV_BASE=60,FOV_SCALE=0.22,FOV_MAX=95,
			SIDE_OFFSET=52,FORWARD_LEAD=0.30,TILT_DOWN=8,
		},
	},
	PLAY={
		REDZONE_DIST_REDUCE=16,REDZONE_HEIGHT_REDUCE=10,
		GOALLINE_DIST=-24,GOALLINE_HEIGHT=-13,
		BIGPLAY_FOV=11,BIGPLAY_DIST=10,
	},
}

local Settings={
	showShotHud=true,
	notificationsOn=true,
	ballResponsive="HIGH",
	broadcastSmooth=0.055,
	autoReplay=true,
	transitionStyle="SMOOTH",
	hudOpacity=0.65,
	minimalistHud=false,
	soundEffects=true,
	cameraShake=true,
	shakeIntensity=0.5,
	predictiveBall=true,
	smartCuts=true,
}

local BALL_SMOOTH_MAP={LOW=0.14,NORMAL=0.28,HIGH=0.50}

local Health={ok=true,errors=0,lastErrorTime=0,log={},lastCheck=0,lastCleanup=0}
local NotifSys={gui=nil,container=nil,queue={}}
local SettingsSys={gui=nil,visible=false}
local ShotHud={gui=nil,label=nil,visible=true}
local EditorSys={gui=nil,visible=false,customCams={},selectedCam=nil,isDragging=false}

local ShaderSys={
	on=false,cc=nil,bloom=nil,sunRays=nil,dof=nil,atm=nil,
	origLight={},lastUpdate=0,
}

local GameState={
	phase="PRE_SNAP",playType="UNKNOWN",prevPhase="PRE_SNAP",
	ballCarrier=nil,qb=nil,kicker=nil,punter=nil,
	primaryReceiver=nil,returner=nil,
	ballInAir=false,ballKicked=false,ballHeight=0,ballSpeed=0,
	ballVelPrev=Vector3.new(),ballAcceleration=0,
	ballLaunchTime=0,ballLaunchPos=Vector3.new(),
	ballPredictedLanding=Vector3.new(),ballFlightPhase="NONE",
	lastPhaseChange=0,kickTime=0,phaseTime=0,
	isRedZone=false,isGoalLine=false,isMidfield=false,
	yardsFromGoal=50,fieldSide=1,
	offense={qb=nil,receivers={},rbs={}},
	defense={rushers={},lbs={},dbs={}},
	specialTeams={kicker=nil,punter=nil,returner=nil},
	analysis={
		isPressure=false,isBlitz=false,rushersNearQB=0,
		isSack=false,isFumble=false,isInterception=false,
		isTackle=false,isGoalLineTackle=false,
		isBreakaway=false,isOpenField=false,
		isScramble=false,isQBRun=false,
		isBigPlay=false,isHailMary=false,isDeepBall=false,
		isScreenPass=false,isCatch=false,
		receiverSeparation=0,defPlayersNearBall=0,
		isPuntHang=false,isOnsideKick=false,
		isHurryUp=false,
		touchdown=false,isRedZoneTD=false,isTwoPoint=false,
		intensity=0,ballAirTime=0,
		lastTDTime=0,lastBigPlayTime=0,lastSackTime=0,
		lastFumbleTime=0,lastPickTime=0,consecutivePlays=0,
		momentum=0,
		ballCatchImminent=false,catchTarget=nil,catchETA=0,
		throwType="UNKNOWN",spiralQuality=0,
	},
	playerCluster=Vector3.new(),clusterRadius=0,
	offenseCluster=Vector3.new(),defenseCluster=Vector3.new(),
}

local AI={
	shot="BROADCAST",prevShot="BROADCAST",
	shotStart=0,confidence=1.0,
	history={},lastDecision=0,scores={},
	urgentOverride=nil,urgentUntil=0,
	recentShots={},momentum=0,
	transitionQueue=nil,transitionStart=0,
}

local PlayerIntel={
	tracked={},roles={},lastUpdate=0,
	qbPads={},qbPadLastScan=0,
}

local BallTracker={
	posHistory={},velHistory={},accelHistory={},
	launchDetected=false,launchTime=0,launchPos=Vector3.new(),launchVel=Vector3.new(),
	flightPhase="NONE",predictedLanding=Vector3.new(),
	peakHeight=0,timeInAir=0,
	catchTarget=nil,catchETA=0,catchConfidence=0,
	lastTrackTime=0,
	spiralAxis=Vector3.new(),spiralStability=0,
}

local Cam={
	on=false,mode="AI",manual=false,
	camera=WS.CurrentCamera,
	ball=nil,confirmedBall=nil,
	conns={},
	curCF=CFrame.new(),tgtCF=CFrame.new(),
	curFOV=70,tgtFOV=70,
	dynDist=0,dynHeight=0,dynFOV=0,
	lastBallScan=0,frame=0,
	shakeOffset=CFrame.new(),
}

local Drone={
	active=false,pos=Vector3.new(0,40,0),vel=Vector3.new(),
	yaw=0,pitch=0,fov=CFG.DRONE.FOV,
	lockMode=nil,lockPlayer=nil,orbitActive=false,orbitAngle=0,
	hudGui=nil,
}

local PovSys={
	active=false,target=nil,playerList={},playerIdx=1,
	lastCycle=0,CYCLE_COOLDOWN=0.6,
	bobPhase=0,prevVel=Vector3.new(),bankAngle=0,
}

local OverviewState={computedPos=Vector3.new(0,60,0),lastRecompute=0,RECOMPUTE_INTERVAL=0.25}

local function buildNotifGui()
	safeCall(function()
		local pg=LP:WaitForChild("PlayerGui")
		local old=pg:FindFirstChild("UFB_Notifs")
		if old then old:Destroy() end
		local sg=Instance.new("ScreenGui")
		sg.Name="UFB_Notifs"
		sg.ResetOnSpawn=false
		sg.DisplayOrder=9999
		sg.IgnoreGuiInset=true
		sg.ZIndexBehavior=Enum.ZIndexBehavior.Sibling
		sg.Parent=pg
		local c=Instance.new("Frame")
		c.Name="Container"
		c.Size=UDim2.new(0,300,1,0)
		c.Position=UDim2.new(1,-308,0,0)
		c.BackgroundTransparency=1
		c.Parent=sg
		local ul=Instance.new("UIListLayout")
		ul.SortOrder=Enum.SortOrder.LayoutOrder
		ul.VerticalAlignment=Enum.VerticalAlignment.Top
		ul.Padding=UDim.new(0,5)
		ul.Parent=c
		local pad=Instance.new("UIPadding")
		pad.PaddingTop=UDim.new(0,18)
		pad.Parent=c
		NotifSys.container=c
		NotifSys.gui=sg
	end,"BUILD_NOTIF")
end

local function pushNotif(title,msg,isError,dur)
	if not Settings.notificationsOn then return end
	dur=dur or 4.0
	safeCall(function()
		if not NotifSys.container or not NotifSys.container.Parent then buildNotifGui() end
		if not NotifSys.container then return end

		local accent=isError and Color3.fromRGB(210,38,38) or Color3.fromRGB(200,200,200)
		local card=Instance.new("Frame")
		card.Name="UFB_N"
		card.Size=UDim2.new(1,0,0,52)
		card.BackgroundColor3=Color3.fromRGB(8,8,8)
		card.BackgroundTransparency=0.08
		card.BorderSizePixel=0
		card.ClipsDescendants=true
		card.Parent=NotifSys.container

		local uc=Instance.new("UICorner")
		uc.CornerRadius=UDim.new(0,4)
		uc.Parent=card

		local us=Instance.new("UIStroke")
		us.Color=Color3.fromRGB(32,32,32)
		us.Thickness=1
		us.Parent=card

		local ab=Instance.new("Frame")
		ab.Size=UDim2.new(0,2,1,0)
		ab.BackgroundColor3=accent
		ab.BorderSizePixel=0
		ab.Parent=card

		local tl=Instance.new("TextLabel")
		tl.Size=UDim2.new(1,-50,0,14)
		tl.Position=UDim2.new(0,14,0,8)
		tl.BackgroundTransparency=1
		tl.Text=string.upper(title)
		tl.TextColor3=Color3.fromRGB(240,240,240)
		tl.TextSize=10
		tl.Font=Enum.Font.GothamBold
		tl.TextXAlignment=Enum.TextXAlignment.Left
		tl.TextTruncate=Enum.TextTruncate.AtEnd
		tl.Parent=card

		local ml=Instance.new("TextLabel")
		ml.Size=UDim2.new(1,-14,0,22)
		ml.Position=UDim2.new(0,14,0,24)
		ml.BackgroundTransparency=1
		ml.Text=msg
		ml.TextColor3=isError and Color3.fromRGB(168,58,58) or Color3.fromRGB(120,120,120)
		ml.TextSize=9.5
		ml.Font=Enum.Font.Gotham
		ml.TextXAlignment=Enum.TextXAlignment.Left
		ml.TextWrapped=true
		ml.TextTruncate=Enum.TextTruncate.AtEnd
		ml.Parent=card

		card.Size=UDim2.new(1,0,0,0)
		TweenService:Create(card,TweenInfo.new(0.18,Enum.EasingStyle.Quad),
			{Size=UDim2.new(1,0,0,52)}):Play()

		task.delay(dur,function()
			safeCall(function()
				if card and card.Parent then
					local fade=TweenService:Create(card,TweenInfo.new(0.25,Enum.EasingStyle.Quad),
						{Size=UDim2.new(1,0,0,0),BackgroundTransparency=1})
					fade:Play()
					fade.Completed:Connect(function() if card and card.Parent then card:Destroy() end end)
				end
			end,"NOTIF_FADE")
		end)
	end,"PUSH_NOTIF")
end

local function notify(t,m) pushNotif(t,m,false) end
local function notifyErr(t,m) pushNotif(t,m,true,5.5) end

local function buildShotHud()
	safeCall(function()
		local pg=LP:WaitForChild("PlayerGui")
		local old=pg:FindFirstChild("UFB_ShotHud")
		if old then old:Destroy() end
		local sg=Instance.new("ScreenGui")
		sg.Name="UFB_ShotHud"
		sg.ResetOnSpawn=false
		sg.DisplayOrder=9998
		sg.IgnoreGuiInset=true
		sg.Parent=pg

		local frame=Instance.new("Frame")
		frame.AnchorPoint=Vector2.new(0.5,1)
		frame.Size=UDim2.new(0,260,0,30)
		frame.Position=UDim2.new(0.5,0,1,-12)
		frame.BackgroundColor3=Color3.fromRGB(6,6,6)
		frame.BackgroundTransparency=0.30
		frame.BorderSizePixel=0
		frame.Parent=sg

		local uc=Instance.new("UICorner")
		uc.CornerRadius=UDim.new(0,15)
		uc.Parent=frame
		local us=Instance.new("UIStroke")
		us.Color=Color3.fromRGB(45,45,45)
		us.Thickness=1
		us.Parent=frame

		local dot=Instance.new("Frame")
		dot.Size=UDim2.new(0,6,0,6)
		dot.Position=UDim2.new(0,12,0.5,-3)
		dot.BackgroundColor3=Color3.fromRGB(100,220,100)
		dot.BorderSizePixel=0
		dot.Parent=frame
		local dc=Instance.new("UICorner")
		dc.CornerRadius=UDim.new(1,0)
		dc.Parent=dot

		local lbl=Instance.new("TextLabel")
		lbl.Size=UDim2.new(1,-26,1,0)
		lbl.Position=UDim2.new(0,26,0,0)
		lbl.BackgroundTransparency=1
		lbl.Text="AI DIRECTOR"
		lbl.TextColor3=Color3.fromRGB(210,210,210)
		lbl.TextSize=10
		lbl.Font=Enum.Font.GothamBold
		lbl.TextXAlignment=Enum.TextXAlignment.Left
		lbl.Parent=frame

		local tierLabel=Instance.new("TextLabel")
		tierLabel.Size=UDim2.new(0,60,1,0)
		tierLabel.Position=UDim2.new(1,-68,0,0)
		tierLabel.BackgroundTransparency=1
		tierLabel.Text=LicenseData.tier
		tierLabel.TextColor3=Color3.fromRGB(50,50,50)
		tierLabel.TextSize=8
		tierLabel.Font=Enum.Font.GothamBold
		tierLabel.TextXAlignment=Enum.TextXAlignment.Right
		tierLabel.Parent=frame

		ShotHud.gui=sg
		ShotHud.label=lbl
		ShotHud.dot=dot
		ShotHud.frame=frame
		frame.Visible=Settings.showShotHud
	end,"BUILD_SHOT_HUD")
end

local function updateShotHud(mode,isManual)
	if not ShotHud.label then return end
	safeCall(function()
		local display=mode:gsub("_"," ")
		if not isManual then display="AI · "..display end
		ShotHud.label.Text=display
		if ShotHud.dot then
			ShotHud.dot.BackgroundColor3=isManual and Color3.fromRGB(100,160,255) or Color3.fromRGB(100,220,100)
		end
		if ShotHud.frame then ShotHud.frame.Visible=Settings.showShotHud end
	end,"SHOT_HUD_UPDATE")
end

local function isSpectator(p)
	if not p or not p.Team then return true end
	local tn=string.lower(p.Team.Name)
	for _,sn in ipairs(CFG.PLAYER.SPECTATOR_NAMES) do if tn:find(sn) then return true end end
	return false
end

local function scanQBPads()
	local t=tick()
	if t-PlayerIntel.qbPadLastScan<CFG.PLAYER.QBPAD_SCAN_INTERVAL then return end
	PlayerIntel.qbPadLastScan=t
	PlayerIntel.qbPads={}
	for _,obj in ipairs(WS:GetDescendants()) do
		if obj:IsA("BasePart") and (obj.Name=="QuarterBackPad" or obj.Name=="QBPad" or obj.Name=="QB_Pad") then
			table.insert(PlayerIntel.qbPads,obj)
		end
	end
end

local function findBall()
	if Cam.confirmedBall and Cam.confirmedBall.Parent then
		Cam.ball=Cam.confirmedBall
		return true
	end
	local t=tick()
	if t-Cam.lastBallScan<CFG.BALL.RESCAN_INTERVAL then return Cam.ball~=nil end
	Cam.lastBallScan=t
	for _,name in ipairs(CFG.BALL.NAMES) do
		local obj=WS:FindFirstChild(name,true)
		if obj then
			local part=(obj:IsA("BasePart") and obj) or (obj:IsA("Model") and (obj.PrimaryPart or obj:FindFirstChildWhichIsA("BasePart")))
			if part then
				Cam.ball=part
				Cam.confirmedBall=part
				BallTracker.posHistory={}
				BallTracker.velHistory={}
				BallTracker.accelHistory={}
				notify("Ball Found",name)
				return true
			end
		end
	end
	return false
end

function BallTracker:Update(dt)
	if not Cam.ball or not Cam.ball.Parent then return end
	local now=tick()
	local bp=Cam.ball.Position
	local physVel=Cam.ball.AssemblyLinearVelocity or Vector3.new()

	table.insert(self.posHistory,{p=bp,t=now})
	if #self.posHistory>CFG.BALL.HISTORY_SIZE then table.remove(self.posHistory,1) end

	local computedVel=physVel
	if #self.posHistory>=3 then
		local recent=self.posHistory[#self.posHistory]
		local older=self.posHistory[math.max(1,#self.posHistory-4)]
		local tDelta=recent.t-older.t
		if tDelta>0.008 then
			local deltaVel=(recent.p-older.p)/tDelta
			if deltaVel.Magnitude>physVel.Magnitude*1.3 or physVel.Magnitude<3 then
				computedVel=deltaVel
			end
		end
	end

	table.insert(self.velHistory,{v=computedVel,t=now})
	if #self.velHistory>20 then table.remove(self.velHistory,1) end

	if #self.velHistory>=2 then
		local cur=self.velHistory[#self.velHistory]
		local prev=self.velHistory[#self.velHistory-1]
		local tD=cur.t-prev.t
		if tD>0.001 then
			local accel=(cur.v-prev.v)/tD
			table.insert(self.accelHistory,{a=accel,t=now})
			if #self.accelHistory>15 then table.remove(self.accelHistory,1) end
		end
	end

	local speed=computedVel.Magnitude
	local height=bp.Y
	local wasInAir=self.flightPhase~="NONE"

	if not wasInAir and height>CFG.BALL.AIR_MIN_HEIGHT and speed>CFG.BALL.AIR_MIN_VEL then
		if #self.accelHistory>=2 then
			local recentAccel=self.accelHistory[#self.accelHistory].a.Magnitude
			if recentAccel>CFG.BALL.LAUNCH_DETECT_ACCEL or speed>15 then
				self.launchDetected=true
				self.launchTime=now
				self.launchPos=bp
				self.launchVel=computedVel
				self.flightPhase="RISING"
				self.peakHeight=height
				self.timeInAir=0
				GameState.ballLaunchTime=now
				GameState.ballLaunchPos=bp

				if speed>35 and height>15 then
					GameState.analysis.throwType="DEEP"
				elseif speed>20 and height>8 then
					GameState.analysis.throwType="MEDIUM"
				elseif speed<18 then
					GameState.analysis.throwType="SHORT"
				end
			end
		end
	end

	if self.flightPhase~="NONE" then
		self.timeInAir=now-self.launchTime
		GameState.analysis.ballAirTime=self.timeInAir

		if height>self.peakHeight then
			self.peakHeight=height
			self.flightPhase="RISING"
		elseif computedVel.Y<-2 and self.flightPhase=="RISING" then
			self.flightPhase="FALLING"
		end

		if Settings.predictiveBall then
			self:PredictLanding(bp,computedVel)
			self:PredictCatch(bp,computedVel)
		end

		if height<3 and self.timeInAir>0.3 then
			self.flightPhase="NONE"
			self.launchDetected=false
			GameState.ballFlightPhase="NONE"
		end

		self:AnalyzeSpiral(computedVel)
	end

	GameState.ballSpeed=speed
	GameState.ballHeight=height
	GameState.ballInAir=self.flightPhase~="NONE"
	GameState.ballFlightPhase=self.flightPhase
	self.lastTrackTime=now
end

function BallTracker:PredictLanding(pos,vel)
	if vel.Magnitude<2 then return end
	local g=-196.2
	local vy=vel.Y
	local t=0
	if vy>0 then
		t=(-vy-math.sqrt(vy*vy-2*g*(pos.Y-2)))/g
	else
		local disc=vy*vy-2*g*(pos.Y-2)
		if disc>0 then t=(-vy+math.sqrt(disc))/g end
	end
	if t>0 and t<10 then
		self.predictedLanding=Vector3.new(pos.X+vel.X*t,2,pos.Z+vel.Z*t)
		GameState.ballPredictedLanding=self.predictedLanding
	end
end

function BallTracker:PredictCatch(pos,vel)
	local receivers=GameState.offense.receivers
	local bestTarget,bestETA,bestConf=nil,math.huge,0

	for _,p in ipairs(receivers) do
		local hrp=getHRP(p)
		if hrp then
			local rpos=hrp.Position
			local rvel=hrp.AssemblyLinearVelocity or Vector3.new()
			for t=0.1,2.0,0.1 do
				local ballFuture=pos+vel*t+Vector3.new(0,-98.1*t*t,0)
				local playerFuture=rpos+rvel*t
				local d=dist3(ballFuture,playerFuture)
				if d<8 then
					local conf=1.0-d/8.0
					if t<bestETA then
						bestTarget=p
						bestETA=t
						bestConf=conf
					end
					break
				end
			end
		end
	end

	if bestTarget and bestETA<3.0 then
		self.catchTarget=bestTarget
		self.catchETA=bestETA
		self.catchConfidence=bestConf
		GameState.analysis.ballCatchImminent=bestETA<CFG.BALL.CATCH_PREDICT_TIME
		GameState.analysis.catchTarget=bestTarget
		GameState.analysis.catchETA=bestETA
	else
		self.catchTarget=nil
		self.catchETA=math.huge
		self.catchConfidence=0
		GameState.analysis.ballCatchImminent=false
	end
end

function BallTracker:AnalyzeSpiral(vel)
	if vel.Magnitude<5 then self.spiralStability=0 return end
	local dir=vel.Unit
	if #self.velHistory<5 then return end
	local deviation=0
	for i=math.max(1,#self.velHistory-5),#self.velHistory do
		local v=self.velHistory[i].v
		if v.Magnitude>3 then
			deviation=deviation+(1-math.abs(v.Unit:Dot(dir)))
		end
	end
	self.spiralStability=math.max(0,1-deviation/5)
	GameState.analysis.spiralQuality=self.spiralStability
end

function BallTracker:GetVelocity()
	if not Cam.ball or not Cam.ball.Parent then return Vector3.new() end
	if #self.velHistory>0 then
		return self.velHistory[#self.velHistory].v
	end
	return Cam.ball.AssemblyLinearVelocity or Vector3.new()
end

function BallTracker:GetSmoothedVelocity(frames)
	frames=frames or 5
	if #self.velHistory<2 then return self:GetVelocity() end
	local sum=Vector3.new()
	local count=0
	for i=math.max(1,#self.velHistory-frames+1),#self.velHistory do
		sum=sum+self.velHistory[i].v
		count=count+1
	end
	return count>0 and sum/count or Vector3.new()
end

function BallTracker:GetPredictedPosition(timeAhead)
	if not Cam.ball or not Cam.ball.Parent then return Vector3.new() end
	local pos=Cam.ball.Position
	local vel=self:GetVelocity()
	return pos+vel*timeAhead+Vector3.new(0,-98.1*timeAhead*timeAhead,0)
end

local function getBallVel() return BallTracker:GetVelocity() end
local function getBallSpeed() return BallTracker:GetVelocity().Magnitude end
local function getBallHeight() return Cam.ball and Cam.ball.Parent and Cam.ball.Position.Y or 0 end
local function ballInAir() return GameState.ballInAir end
local function ballKicked() return Cam.ball and Cam.ball.Parent and Cam.ball.Position.Y>CFG.BALL.KICK_HEIGHT and getBallSpeed()>CFG.BALL.KICK_VEL end

local function newPlayerData(p)
	return {
		player=p,role="UNKNOWN",position=Vector3.new(),velocity=Vector3.new(),
		speed=0,hasBall=false,isNearBall=false,posHist={},velHist={},
		distToBall=math.huge,pattern="STATIC",prevPos=Vector3.new(),
		acceleration=0,onOffense=false,predictedPos=Vector3.new(),
		directionAngle=0,lastSpeedChange=0,
	}
end

local function analyzePattern(pd)
	if #pd.posHist<8 then return "STATIC" end
	local total=0
	for i=2,#pd.posHist do total=total+(pd.posHist[i]-pd.posHist[i-1]).Magnitude end
	local straight=(pd.posHist[#pd.posHist]-pd.posHist[1]).Magnitude
	if total<4 then return "STATIC" end
	if straight/total>0.82 then return "LINEAR" end
	if total>30 and straight/total<0.4 then return "ROUTE" end
	return "ERRATIC"
end

local function detectRole(pd)
	local char=pd.player.Character
	if not char then return "UNKNOWN" end
	local hrp=char:FindFirstChild("HumanoidRootPart")
	if not hrp then return "UNKNOWN" end
	local pos=hrp.Position
	table.insert(pd.posHist,pos)
	if #pd.posHist>CFG.PLAYER.HISTORY_SIZE then table.remove(pd.posHist,1) end
	pd.pattern=analyzePattern(pd)

	if #pd.posHist>1 then
		pd.acceleration=(pd.speed-(pd.posHist[#pd.posHist]-pd.posHist[math.max(1,#pd.posHist-3)]).Magnitude)
	end

	if pd.speed>1 and pd.velocity.Magnitude>1 then
		pd.predictedPos=pos+pd.velocity.Unit*pd.speed*0.3
	else
		pd.predictedPos=pos
	end

	scanQBPads()
	for _,pad in ipairs(PlayerIntel.qbPads) do
		if dist3(pos,pad.Position)<CFG.PLAYER.QB_PAD_DIST then return "QB" end
	end

	if Cam.ball and Cam.ball.Parent then
		local bp=Cam.ball.Position
		local d=dist3(pos,bp)
		pd.distToBall=d
		pd.hasBall=d<CFG.PLAYER.BALL_CARRIER_RADIUS
		pd.isNearBall=d<11

		if pd.hasBall then
			if ballKicked() then return "K" end
			if GameState.playType=="PUNT" then return "P" end
			if GameState.playType=="KICKOFF" then return pd.speed<5 and "K" or "RETURNER" end
			if pd.speed<7 then return "QB"
			elseif pd.speed>20 then return "WR"
			else return "RB" end
		end

		local wx=math.abs(pos.X)
		if wx>18 and (pd.speed>14 or pd.pattern=="LINEAR" or pd.pattern=="ROUTE") then return "WR" end
		if wx>8 and wx<18 and pd.speed>9 then return "TE" end
		if pd.speed>24 then return "WR" end
		if pd.speed>17 and d>10 then return "RB" end
		if pd.speed>11 and d<15 then return "LB" end
		if pd.speed>14 and wx>11 then return "DB" end
		if math.abs(pos.Z-bp.Z)<8 and pd.speed<5 then return wx<8 and "OL" or "DL" end
	end
	return "UNKNOWN"
end

local function updatePlayerIntel()
	safeCall(function()
		local t=tick()
		if t-PlayerIntel.lastUpdate<CFG.PLAYER.ROLE_INTERVAL then return end
		PlayerIntel.lastUpdate=t

		GameState.offense.receivers={}
		GameState.offense.rbs={}
		GameState.defense.rushers={}
		GameState.defense.lbs={}
		GameState.defense.dbs={}

		local clusterSum,clusterCount=Vector3.new(),0
		local offSum,offCount=Vector3.new(),0
		local defSum,defCount=Vector3.new(),0
		local activePlayers={}

		for _,p in ipairs(Players:GetPlayers()) do
			if not isSpectator(p) and p.Character then
				local hrp=p.Character:FindFirstChild("HumanoidRootPart")
				if hrp then
					table.insert(activePlayers,p)
					clusterSum=clusterSum+hrp.Position
					clusterCount=clusterCount+1

					if not PlayerIntel.tracked[p.UserId] then
						PlayerIntel.tracked[p.UserId]=newPlayerData(p)
					end
					local pd=PlayerIntel.tracked[p.UserId]
					local vel=hrp.AssemblyLinearVelocity or Vector3.new()
					table.insert(pd.velHist,vel)
					if #pd.velHist>8 then table.remove(pd.velHist,1) end
					local avgVel=Vector3.new()
					for _,v in ipairs(pd.velHist) do avgVel=avgVel+v end
					avgVel=avgVel/math.max(1,#pd.velHist)
					pd.velocity=pd.velocity:Lerp(avgVel,CFG.PLAYER.VEL_SMOOTH)
					pd.speed=pd.velocity.Magnitude
					pd.position=hrp.Position

					local role=detectRole(pd)
					pd.role=role
					PlayerIntel.roles[p.UserId]=role

					if role=="QB" then
						GameState.offense.qb=p
						GameState.qb=p
						offSum=offSum+hrp.Position
						offCount=offCount+1
					elseif role=="WR" or role=="TE" then
						table.insert(GameState.offense.receivers,p)
						offSum=offSum+hrp.Position
						offCount=offCount+1
					elseif role=="RB" then
						table.insert(GameState.offense.rbs,p)
						offSum=offSum+hrp.Position
						offCount=offCount+1
					elseif role=="K" then
						GameState.specialTeams.kicker=p
						GameState.kicker=p
					elseif role=="P" then
						GameState.specialTeams.punter=p
						GameState.punter=p
					elseif role=="RETURNER" then
						GameState.specialTeams.returner=p
						GameState.returner=p
					elseif role=="LB" then
						table.insert(GameState.defense.lbs,p)
						defSum=defSum+hrp.Position
						defCount=defCount+1
					elseif role=="DB" then
						table.insert(GameState.defense.dbs,p)
						defSum=defSum+hrp.Position
						defCount=defCount+1
					elseif role=="DL" then
						table.insert(GameState.defense.rushers,p)
						defSum=defSum+hrp.Position
						defCount=defCount+1
					end
				end
			end
		end

		if clusterCount>0 then
			GameState.playerCluster=clusterSum/clusterCount
			local maxR=0
			for _,p in ipairs(activePlayers) do
				local hrp=getHRP(p)
				if hrp then
					local d=dist3(hrp.Position,GameState.playerCluster)
					if d>maxR then maxR=d end
				end
			end
			GameState.clusterRadius=maxR
		end
		if offCount>0 then GameState.offenseCluster=offSum/offCount end
		if defCount>0 then GameState.defenseCluster=defSum/defCount end
	end,"PLAYER_INTEL")
end

local function findQB() return GameState.offense.qb or GameState.qb end

local function findBallCarrier()
	if not Cam.ball or not Cam.ball.Parent then return nil end
	local bp=Cam.ball.Position
	local closest,closestDist=nil,math.huge
	for _,pd in pairs(PlayerIntel.tracked) do
		if not isSpectator(pd.player) and pd.player.Character then
			local hrp=pd.player.Character:FindFirstChild("HumanoidRootPart")
			if hrp then
				local d=dist3(hrp.Position,bp)
				if d<closestDist and d<CFG.PLAYER.BALL_CARRIER_RADIUS then
					closestDist=d
					closest=pd.player
				end
			end
		end
	end
	return closest
end

local function countPlayersNearPoint(point,radius)
	local count=0
	for _,pd in pairs(PlayerIntel.tracked) do
		if pd.player.Character then
			local hrp=pd.player.Character:FindFirstChild("HumanoidRootPart")
			if hrp and dist3(hrp.Position,point)<radius then count=count+1 end
		end
	end
	return count
end

local function analyzePlaySituation()
	safeCall(function()
		local an=GameState.analysis
		local qb=findQB()
		local carrier=findBallCarrier()
		local bp=(Cam.ball and Cam.ball.Parent) and Cam.ball.Position or Vector3.new()
		local t=tick()

		an.rushersNearQB=0
		if qb then
			local qhrp=getHRP(qb)
			if qhrp then
				for _,p in ipairs(GameState.defense.rushers) do
					local h=getHRP(p)
					if h and dist3(h.Position,qhrp.Position)<14 then an.rushersNearQB=an.rushersNearQB+1 end
				end
				for _,p in ipairs(GameState.defense.lbs) do
					local h=getHRP(p)
					if h and dist3(h.Position,qhrp.Position)<10 then an.rushersNearQB=an.rushersNearQB+1 end
				end
			end
		end

		an.isPressure=an.rushersNearQB>=2 and GameState.phase=="PASS"
		an.isBlitz=an.rushersNearQB>=3
		an.defPlayersNearBall=Cam.ball and Cam.ball.Parent and countPlayersNearPoint(bp,8) or 0

		an.isHailMary=GameState.ballInAir and an.ballAirTime>2.5 and GameState.isRedZone
		an.isPuntHang=GameState.playType=="PUNT" and GameState.ballInAir and an.ballAirTime>1.8
		an.isDeepBall=GameState.ballInAir and an.ballAirTime>1.5 and GameState.ballSpeed>25

		an.isBreakaway=false
		if carrier then
			local cpd=PlayerIntel.tracked[carrier.UserId]
			if cpd and cpd.speed>22 then
				local nearDef=0
				local chrp=getHRP(carrier)
				if chrp then
					for _,pd in pairs(PlayerIntel.tracked) do
						if pd.role=="LB" or pd.role=="DB" or pd.role=="DL" then
							if pd.player.Character then
								local h=pd.player.Character:FindFirstChild("HumanoidRootPart")
								if h and dist3(h.Position,chrp.Position)<18 then nearDef=nearDef+1 end
							end
						end
					end
				end
				an.isBreakaway=nearDef==0 and cpd.speed>22
			end
		end

		an.isScramble=false
		if qb then
			local qpd=PlayerIntel.tracked[qb.UserId]
			if qpd and qpd.speed>14 and GameState.phase~="PRE_SNAP" then an.isScramble=true end
		end

		if carrier then
			local cpd=PlayerIntel.tracked[carrier.UserId]
			if cpd then
				an.isTackle=(GameState.phase=="RUN" or GameState.phase=="PASS") and cpd.speed<3 and cpd.speed>0 and an.defPlayersNearBall>=1
				an.isFumble=an.isTackle and GameState.ballSpeed>8 and not cpd.hasBall
				an.isGoalLineTackle=an.isTackle and GameState.isGoalLine
			end
		end

		an.isSack=GameState.phase=="PASS" and qb~=nil and (function()
			local qhrp=getHRP(qb)
			if not qhrp then return false end
			local qpd=PlayerIntel.tracked[qb.UserId]
			return qpd and qpd.speed<2 and an.rushersNearQB>=1
		end)()

		an.receiverSeparation=0
		if GameState.primaryReceiver and Cam.ball and Cam.ball.Parent then
			local rhrp=getHRP(GameState.primaryReceiver)
			if rhrp then
				local minDefDist=math.huge
				for _,pd in pairs(PlayerIntel.tracked) do
					if pd.role=="DB" or pd.role=="LB" then
						if pd.player.Character then
							local h=pd.player.Character:FindFirstChild("HumanoidRootPart")
							if h then
								local d=dist3(h.Position,rhrp.Position)
								if d<minDefDist then minDefDist=d end
							end
						end
					end
				end
				an.receiverSeparation=minDefDist==math.huge and 0 or minDefDist
			end
		end

		an.isHurryUp=(GameState.phase=="PRE_SNAP") and #GameState.offense.receivers>2 and (an.consecutivePlays or 0)>2

		an.intensity=0
		if an.isPressure then an.intensity=an.intensity+0.2 end
		if an.isBlitz then an.intensity=an.intensity+0.15 end
		if an.isBreakaway then an.intensity=an.intensity+0.3 end
		if GameState.isRedZone then an.intensity=an.intensity+0.15 end
		if GameState.isGoalLine then an.intensity=an.intensity+0.2 end
		if an.ballCatchImminent then an.intensity=an.intensity+0.25 end

		an.isBigPlay=(carrier~=nil and (function()
			local cpd=PlayerIntel.tracked[carrier.UserId]
			return cpd and cpd.speed>18 and GameState.yardsFromGoal<40
		end)()) or an.isBreakaway or an.isScramble

		if an.touchdown and (t-an.lastTDTime)>0.5 then
			an.lastTDTime=t
			an.consecutivePlays=0
		end
		if an.isBigPlay and (t-an.lastBigPlayTime)>2.0 then an.lastBigPlayTime=t end

		an.momentum=clamp(an.intensity*2+
			(an.isBigPlay and 0.3 or 0)+
			(GameState.ballInAir and 0.15 or 0)+
			(an.isScramble and 0.2 or 0),0,1)
	end,"ANALYZE_PLAY")
end

local function detectPlayType()
	local speed=getBallSpeed()
	local height=getBallHeight()
	local inAir=ballInAir()
	local kicked=ballKicked()
	if height>20 and speed>35 then return "KICKOFF" end
	if height>12 and height<26 and speed>24 and GameState.kicker then
		if Cam.ball and math.abs(Cam.ball.Position.X)<32 then return "FIELD_GOAL" end
	end
	if kicked and height>14 and speed>19 and speed<42 then return "PUNT" end
	if inAir and height>7 and height<22 then return "PASS" end
	local carrier=findBallCarrier()
	if carrier then
		local pd=PlayerIntel.tracked[carrier.UserId]
		if pd and pd.speed>7 then return "RUN" end
	end
	return "UNKNOWN"
end

local function detectGamePhase()
	safeCall(function()
		local t=tick()
		local inAir=ballInAir()
		local kicked=ballKicked()
		local playType=detectPlayType()
		GameState.prevPhase=GameState.phase
		GameState.playType=playType
		GameState.ballKicked=kicked
		local bv=getBallVel()
		GameState.ballAcceleration=(bv-GameState.ballVelPrev).Magnitude
		GameState.ballVelPrev=bv

		if kicked or playType=="KICKOFF" or playType=="PUNT" or playType=="FIELD_GOAL" then
			if GameState.phase~=playType then
				GameState.phase=playType
				GameState.kickTime=t
				GameState.lastPhaseChange=t
			end
			return
		end
		if inAir and playType=="PASS" then
			if GameState.phase~="PASS" then
				GameState.phase="PASS"
				GameState.lastPhaseChange=t
			end
			return
		end
		local carrier=findBallCarrier()
		if carrier then
			GameState.ballCarrier=carrier
			local pd=PlayerIntel.tracked[carrier.UserId]
			if pd and pd.speed>7 then
				if GameState.phase~="RUN" then
					GameState.phase="RUN"
					GameState.lastPhaseChange=t
				end
				return
			end
		end
		if GameState.returner and (playType=="KICKOFF" or playType=="PUNT") then
			if t-GameState.kickTime>1.4 then
				if GameState.phase~="RETURN" then
					GameState.phase="RETURN"
					GameState.lastPhaseChange=t
				end
				return
			end
		end
		if t-GameState.lastPhaseChange>2.8 then GameState.phase="PRE_SNAP" end
		GameState.phaseTime=t-GameState.lastPhaseChange
	end,"PHASE_DETECT")
end

local function analyzeField()
	safeCall(function()
		if not Cam.ball or not Cam.ball.Parent then return end
		local bp=Cam.ball.Position
		local absZ=math.abs(bp.Z)
		GameState.yardsFromGoal=absZ/3
		GameState.isRedZone=absZ<CFG.FIELD.REDZONE_Z
		GameState.isGoalLine=absZ<CFG.FIELD.GOALLINE_Z
		GameState.isMidfield=absZ>CFG.FIELD.MIDFIELD_Z and absZ<(CFG.FIELD.REDZONE_Z+20)
		GameState.fieldSide=bp.Z>0 and 1 or -1
	end,"ANALYZE_FIELD")
end

local function calcDynamic()
	Cam.dynDist=0
	Cam.dynHeight=0
	Cam.dynFOV=0
	if GameState.isRedZone then
		Cam.dynDist=-CFG.PLAY.REDZONE_DIST_REDUCE
		Cam.dynHeight=-CFG.PLAY.REDZONE_HEIGHT_REDUCE
	end
	if GameState.isGoalLine then
		Cam.dynDist=CFG.PLAY.GOALLINE_DIST
		Cam.dynHeight=CFG.PLAY.GOALLINE_HEIGHT
		Cam.dynFOV=4
	end
	if GameState.analysis.isBigPlay then
		Cam.dynFOV=CFG.PLAY.BIGPLAY_FOV
		Cam.dynDist=CFG.PLAY.BIGPLAY_DIST
	end
end

local function applyTilt(cf,deg) return cf*CFrame.Angles(math.rad(deg),0,0) end

local function applyCameraShake(dt)
	if not Settings.cameraShake then
		Cam.shakeOffset=CFrame.new()
		return
	end
	local intensity=GameState.analysis.intensity*Settings.shakeIntensity
	if intensity<0.05 then Cam.shakeOffset=CFrame.new() return end
	local t=tick()*12
	local sx=math.sin(t*1.1)*intensity*0.15
	local sy=math.cos(t*0.9)*intensity*0.10
	local sr=math.sin(t*1.3)*intensity*0.08
	Cam.shakeOffset=CFrame.new(sx,sy,0)*CFrame.Angles(0,0,math.rad(sr))
end

local function camBROADCAST()
	safeCall(function()
		if not Cam.ball or not Cam.ball.Parent then return end
		local bp=Cam.ball.Position
		local c=CFG.CAM.BROADCAST
		local bv=BallTracker:GetSmoothedVelocity(3)
		local leadZ=0
		if bv.Magnitude>8 then leadZ=bv.Z*0.12 end
		local tp=Vector3.new(c.DISTANCE+Cam.dynDist,c.HEIGHT+Cam.dynHeight,bp.Z+c.FOLLOW_Z+leadZ)
		Cam.tgtCF=applyTilt(CFrame.new(tp,bp+Vector3.new(0,3,0)),c.TILT)
		Cam.tgtFOV=c.FOV+Cam.dynFOV
	end,"CAM_BROADCAST")
end

local function camHIGH_BROADCAST()
	safeCall(function()
		if not Cam.ball or not Cam.ball.Parent then return end
		local bp=Cam.ball.Position
		local c=CFG.CAM.HIGH_BROADCAST
		Cam.tgtCF=applyTilt(CFrame.new(Vector3.new(c.DISTANCE,c.HEIGHT,bp.Z+22),bp+Vector3.new(0,5,0)),c.TILT)
		Cam.tgtFOV=c.FOV
	end,"CAM_HIGH_BROADCAST")
end

local function camTIGHT_BROADCAST()
	safeCall(function()
		if not Cam.ball or not Cam.ball.Parent then return end
		local bp=Cam.ball.Position
		local c=CFG.CAM.TIGHT_BROADCAST
		Cam.tgtCF=applyTilt(CFrame.new(Vector3.new(c.DISTANCE,c.HEIGHT+Cam.dynHeight,bp.Z+13),bp+Vector3.new(0,2,0)),c.TILT)
		Cam.tgtFOV=c.FOV+Cam.dynFOV
	end,"CAM_TIGHT_BROADCAST")
end

local function camSKYCAM()
	safeCall(function()
		if not Cam.ball or not Cam.ball.Parent then return end
		local bp=Cam.ball.Position
		local c=CFG.CAM.SKYCAM
		local rad=math.rad(c.DOWN_ANGLE)
		Cam.tgtCF=CFrame.new(Vector3.new(bp.X,c.HEIGHT+Cam.dynHeight,bp.Z),bp+Vector3.new(0,-math.sin(rad)*10,math.cos(rad)*10))
		Cam.tgtFOV=c.FOV+Cam.dynFOV
	end,"CAM_SKYCAM")
end

local function camALL_22()
	safeCall(function()
		if not Cam.ball or not Cam.ball.Parent then return end
		local bp=Cam.ball.Position
		local c=CFG.CAM.ALL_22
		local rad=math.rad(c.DOWN_ANGLE)
		Cam.tgtCF=CFrame.new(Vector3.new(bp.X,c.HEIGHT,bp.Z),bp+Vector3.new(0,-math.sin(rad)*16,math.cos(rad)*16))
		Cam.tgtFOV=c.FOV
	end,"CAM_ALL22")
end

local function camFORMATION_CAM()
	safeCall(function()
		local focus=GameState.playerCluster
		local c=CFG.CAM.FORMATION_CAM
		local rad=math.rad(c.DOWN_ANGLE)
		Cam.tgtCF=CFrame.new(Vector3.new(focus.X,c.HEIGHT,focus.Z),focus+Vector3.new(0,-math.sin(rad)*14,math.cos(rad)*14))
		Cam.tgtFOV=c.FOV
	end,"CAM_FORMATION")
end

local function camHURRY_UP_CAM()
	safeCall(function()
		if not Cam.ball or not Cam.ball.Parent then return end
		local bp=Cam.ball.Position
		local c=CFG.CAM.HURRY_UP_CAM
		local tp=Vector3.new(c.DISTANCE+Cam.dynDist,c.HEIGHT+Cam.dynHeight,bp.Z+14)
		Cam.tgtCF=applyTilt(CFrame.new(tp,bp+Vector3.new(0,3,0)),c.TILT)
		Cam.tgtFOV=c.FOV+Cam.dynFOV
	end,"CAM_HURRY_UP")
end

local function camQB_CAM()
	safeCall(function()
		local qb=findQB()
		if not qb or not qb.Character then camBROADCAST() return end
		local hrp=getHRP(qb)
		if not hrp then return end
		local c=CFG.CAM.QB_CAM
		local fov=c.FOV+(GameState.analysis.isPressure and c.PRESSURE_FOV or 0)
		local dist=c.DIST_BEHIND-(GameState.analysis.isPressure and 2 or 0)
		local look=hrp.CFrame.LookVector
		local right=hrp.CFrame.RightVector
		local tp=hrp.Position+(-look*dist)+(right*c.DIST_SIDE)+Vector3.new(0,c.HEIGHT,0)
		Cam.tgtCF=CFrame.new(tp,hrp.Position+look*22+Vector3.new(0,2,0))
		Cam.tgtFOV=fov+Cam.dynFOV
	end,"CAM_QB")
end

local function camQB_CLOSEUP()
	safeCall(function()
		local qb=findQB()
		if not qb or not qb.Character then return end
		local hrp=getHRP(qb)
		if not hrp then return end
		local c=CFG.CAM.QB_CLOSEUP
		local look=hrp.CFrame.LookVector
		local right=hrp.CFrame.RightVector
		Cam.tgtCF=CFrame.new(hrp.Position+(-look*c.DIST)+(right*c.SIDE)+Vector3.new(0,c.HEIGHT,0),hrp.Position+look*10)
		Cam.tgtFOV=c.FOV
	end,"CAM_QB_CLOSEUP")
end

local function camQB_SCRAMBLE()
	safeCall(function()
		local qb=findQB()
		if not qb or not qb.Character then return end
		local hrp=getHRP(qb)
		if not hrp then return end
		local c=CFG.CAM.QB_SCRAMBLE
		local vel=hrp.AssemblyLinearVelocity or Vector3.new()
		local dir=vel.Magnitude>2 and vel.Unit or hrp.CFrame.LookVector
		local tp=hrp.Position+(-dir*c.DIST)+Vector3.new(0,c.HEIGHT,0)
		Cam.tgtCF=CFrame.new(tp,hrp.Position+dir*c.LEAD+Vector3.new(0,2,0))
		Cam.tgtFOV=c.FOV
	end,"CAM_QB_SCRAMBLE")
end

local function camPOCKET_CAM()
	safeCall(function()
		local qb=findQB()
		if not qb or not qb.Character then return end
		local hrp=getHRP(qb)
		if not hrp then return end
		local c=CFG.CAM.POCKET_CAM
		local angle=(tick()*c.CIRCLE_SPEED)%(math.pi*2)
		local r=c.RADIUS+Cam.dynDist
		Cam.tgtCF=CFrame.new(hrp.Position+Vector3.new(math.cos(angle)*r,c.HEIGHT,math.sin(angle)*r),hrp.Position+Vector3.new(0,2,0))
		Cam.tgtFOV=c.FOV+Cam.dynFOV
	end,"CAM_POCKET")
end

local function camBLITZ_CAM()
	safeCall(function()
		local qb=findQB()
		if not qb or not qb.Character then return end
		local hrp=getHRP(qb)
		if not hrp then return end
		local c=CFG.CAM.BLITZ_CAM
		Cam.tgtCF=CFrame.new(hrp.Position+Vector3.new(c.DIST,c.HEIGHT,0),hrp.Position+Vector3.new(0,2,0))
		Cam.tgtFOV=c.FOV
	end,"CAM_BLITZ")
end

local function camSACK_CAM()
	safeCall(function()
		local qb=findQB()
		if not qb or not qb.Character then return end
		local hrp=getHRP(qb)
		if not hrp then return end
		local c=CFG.CAM.SACK_CAM
		Cam.tgtCF=CFrame.new(hrp.Position+Vector3.new(c.DIST,c.HEIGHT,0),hrp.Position+Vector3.new(0,1,0))
		Cam.tgtFOV=c.FOV
	end,"CAM_SACK")
end

local function camRECEIVER_CAM()
	safeCall(function()
		local receivers=GameState.offense.receivers
		if #receivers==0 then return end
		local target=receivers[1]

		if GameState.analysis.ballCatchImminent and GameState.analysis.catchTarget then
			target=GameState.analysis.catchTarget
		elseif GameState.ballInAir and Cam.ball and Cam.ball.Parent then
			local bp=Cam.ball.Position
			local cd=math.huge
			for _,wr in ipairs(receivers) do
				local h=getHRP(wr)
				if h then
					local predicted=BallTracker:GetPredictedPosition(0.3)
					local d=dist3(h.Position,predicted)
					if d<cd then cd=d target=wr end
				end
			end
		end

		GameState.primaryReceiver=target
		local hrp=getHRP(target)
		if not hrp then return end
		local c=CFG.CAM.RECEIVER_CAM
		local vel=hrp.AssemblyLinearVelocity or Vector3.new()
		local dir=vel.Magnitude>1 and vel.Unit or hrp.CFrame.LookVector
		local dist2=c.DIST
		if GameState.ballInAir and Cam.ball and dist3(hrp.Position,Cam.ball.Position)<20 then dist2=c.CATCH_DIST end
		Cam.tgtCF=CFrame.new(hrp.Position+(-dir*(dist2+Cam.dynDist))+Vector3.new(0,c.HEIGHT,0),hrp.Position+dir*c.LEAD+Vector3.new(0,2,0))
		Cam.tgtFOV=c.FOV+Cam.dynFOV
	end,"CAM_RECEIVER")
end

local function camRECEIVER_ISO()
	safeCall(function()
		if not GameState.primaryReceiver or not GameState.primaryReceiver.Character then return end
		local hrp=getHRP(GameState.primaryReceiver)
		if not hrp then return end
		local c=CFG.CAM.RECEIVER_ISO
		local vel=hrp.AssemblyLinearVelocity or Vector3.new()
		local dir=vel.Magnitude>1 and vel.Unit or hrp.CFrame.LookVector
		local right=dir:Cross(Vector3.new(0,1,0))
		if right.Magnitude>0.01 then right=right.Unit else right=Vector3.new(1,0,0) end
		local rad=math.rad(c.SIDE_ANGLE)
		Cam.tgtCF=CFrame.new(hrp.Position+(-dir*(c.DIST*math.cos(rad)))+(right*(c.DIST*math.sin(rad)))+Vector3.new(0,c.HEIGHT,0),hrp.Position+Vector3.new(0,2,0))
		Cam.tgtFOV=c.FOV
	end,"CAM_RECEIVER_ISO")
end

local function camINTERCEPTION()
	safeCall(function()
		local defPl,bestDist=nil,math.huge
		if Cam.ball and Cam.ball.Parent then
			local bp=Cam.ball.Position
			for _,pd in pairs(PlayerIntel.tracked) do
				if pd.role=="DB" or pd.role=="LB" then
					local h=getHRP(pd.player)
					if h then
						local d=dist3(h.Position,bp)
						if d<bestDist then bestDist=d defPl=pd.player end
					end
				end
			end
		end
		if not defPl then return end
		local hrp=getHRP(defPl)
		if not hrp then return end
		local c=CFG.CAM.INTERCEPTION
		local vel=hrp.AssemblyLinearVelocity or Vector3.new()
		local dir=vel.Magnitude>1 and vel.Unit or hrp.CFrame.LookVector
		Cam.tgtCF=CFrame.new(hrp.Position+(-dir*c.DIST)+Vector3.new(0,c.HEIGHT,0),hrp.Position+dir*c.LEAD+Vector3.new(0,2,0))
		Cam.tgtFOV=c.FOV
	end,"CAM_INTERCEPTION")
end

local function camBALL_CAM()
	safeCall(function()
		if not Cam.ball or not Cam.ball.Parent then return end
		local bp=Cam.ball.Position
		local bv=BallTracker:GetSmoothedVelocity(3)
		local spd=bv.Magnitude
		local c=CFG.CAM.BALL_CAM

		local minSpd=GameState.ballInAir and 18 or 4
		local dir=spd>minSpd and bv.Unit or Vector3.new(0,0,1)

		local lookAhead=bp
		if GameState.ballInAir and BallTracker.catchTarget then
			local catchHrp=getHRP(BallTracker.catchTarget)
			if catchHrp then
				local blend=clamp(1-BallTracker.catchETA/2,0,0.6)
				lookAhead=bp:Lerp(catchHrp.Position,blend)
			end
		else
			lookAhead=bp+dir*(math.max(spd,minSpd)*0.14)
		end

		Cam.tgtCF=CFrame.new(
			bp+(-dir*(c.DIST+spd*0.28+Cam.dynDist))+Vector3.new(0,c.HEIGHT+spd*0.18+Cam.dynHeight,0),
			lookAhead
		)
		Cam.tgtFOV=math.min(c.FOV+spd*c.SPEED_FOV_MULT+Cam.dynFOV,c.MAX_FOV)
	end,"CAM_BALL")
end

local function camBALL_TRACK()
	safeCall(function()
		if not Cam.ball or not Cam.ball.Parent then return end
		local bp=Cam.ball.Position
		local bv=BallTracker:GetSmoothedVelocity(3)
		local spd=bv.Magnitude
		local c=CFG.CAM.BALL_TRACK
		local minSpd=GameState.ballInAir and 15 or 4
		local dir=spd>minSpd and bv.Unit or Vector3.new(0,0,1)
		Cam.tgtCF=CFrame.new(bp+(-dir*(c.DIST+spd*0.18))+Vector3.new(0,c.HEIGHT+spd*0.14,0),bp)
		Cam.tgtFOV=c.FOV
	end,"CAM_BALL_TRACK")
end

local function camBALL_FLIGHT()
	safeCall(function()
		if not Cam.ball or not Cam.ball.Parent then return end
		local bp=Cam.ball.Position
		local bv=BallTracker:GetSmoothedVelocity(3)
		local spd=bv.Magnitude
		local c=CFG.CAM.BALL_FLIGHT

		local futurePos=BallTracker:GetPredictedPosition(c.LEAD_TIME)
		local midPoint=(bp+futurePos)/2+Vector3.new(0,c.ARC_OFFSET,0)
		local dir=spd>5 and bv.Unit or Vector3.new(0,0,1)
		local right=dir:Cross(Vector3.new(0,1,0))
		if right.Magnitude<0.01 then right=Vector3.new(1,0,0) else right=right.Unit end

		local camPos=midPoint+right*c.DIST+Vector3.new(0,c.HEIGHT,0)

		local lookTarget=bp
		if BallTracker.catchTarget and BallTracker.catchETA<1.0 then
			local catchHrp=getHRP(BallTracker.catchTarget)
			if catchHrp then
				lookTarget=bp:Lerp(catchHrp.Position,clamp(1-BallTracker.catchETA,0,0.7))
			end
		end

		Cam.tgtCF=CFrame.new(camPos,lookTarget)
		Cam.tgtFOV=c.FOV+clamp(spd*0.1,0,10)
	end,"CAM_BALL_FLIGHT")
end

local function camBALL_SPIRAL()
	safeCall(function()
		if not Cam.ball or not Cam.ball.Parent then return end
		local bp=Cam.ball.Position
		local bv=BallTracker:GetSmoothedVelocity(3)
		local spd=bv.Magnitude
		local c=CFG.CAM.BALL_SPIRAL

		local dir=spd>5 and bv.Unit or Vector3.new(0,0,1)
		local angle=(tick()*c.ORBIT_SPEED)%(math.pi*2)
		local right=dir:Cross(Vector3.new(0,1,0))
		if right.Magnitude<0.01 then right=Vector3.new(1,0,0) else right=right.Unit end
		local up=right:Cross(dir).Unit

		local orbitPos=bp+(right*math.cos(angle)+up*math.sin(angle))*c.DIST+(-dir*c.DIST*0.5)+Vector3.new(0,c.HEIGHT,0)

		Cam.tgtCF=CFrame.new(orbitPos,bp)
		Cam.tgtFOV=c.FOV
	end,"CAM_BALL_SPIRAL")
end

local function camCATCH_CAM()
	safeCall(function()
		local target=GameState.analysis.catchTarget or GameState.primaryReceiver
		if not target or not target.Character then camRECEIVER_CAM() return end
		local hrp=getHRP(target)
		if not hrp then return end
		local c=CFG.CAM.CATCH_CAM
		local vel=hrp.AssemblyLinearVelocity or Vector3.new()
		local dir=vel.Magnitude>1 and vel.Unit or hrp.CFrame.LookVector

		local camPos=hrp.Position+(-dir*c.DIST)+Vector3.new(0,c.HEIGHT,0)
		local lookTarget=hrp.Position+Vector3.new(0,2,0)

		if Cam.ball and Cam.ball.Parent then
			lookTarget=(hrp.Position+Cam.ball.Position)/2+Vector3.new(0,1,0)
		end

		Cam.tgtCF=CFrame.new(camPos,lookTarget)
		Cam.tgtFOV=c.FOV
	end,"CAM_CATCH")
end

local function camCINEMATIC()
	safeCall(function()
		if not Cam.ball or not Cam.ball.Parent then return end
		local bp=Cam.ball.Position
		local c=CFG.CAM.CINEMATIC
		local t=tick()*c.DOLLY_SPEED
		local angle=t%(math.pi*2)
		local camPos=bp+Vector3.new(math.cos(angle)*c.DIST,c.HEIGHT,math.sin(angle)*c.DIST)
		Cam.tgtCF=CFrame.new(camPos,bp+Vector3.new(0,3,0))
		Cam.tgtFOV=c.FOV
	end,"CAM_CINEMATIC")
end

local function camOVERHEAD_TRACK()
	safeCall(function()
		if not Cam.ball or not Cam.ball.Parent then return end
		local bp=Cam.ball.Position
		local c=CFG.CAM.OVERHEAD_TRACK
		local rad=math.rad(c.DOWN_ANGLE)
		Cam.tgtCF=CFrame.new(Vector3.new(bp.X,c.HEIGHT,bp.Z),bp+Vector3.new(0,-math.sin(rad)*20,math.cos(rad)*5))
		Cam.tgtFOV=c.FOV
	end,"CAM_OVERHEAD_TRACK")
end

local function camREVERSE_ANGLE()
	safeCall(function()
		if not Cam.ball or not Cam.ball.Parent then return end
		local bp=Cam.ball.Position
		local c=CFG.CAM.REVERSE_ANGLE
		local tp=Vector3.new(-c.DISTANCE+Cam.dynDist,c.HEIGHT+Cam.dynHeight,bp.Z+18)
		Cam.tgtCF=applyTilt(CFrame.new(tp,bp+Vector3.new(0,3,0)),c.TILT)
		Cam.tgtFOV=c.FOV+Cam.dynFOV
	end,"CAM_REVERSE_ANGLE")
end

local function camLOW_ENDZONE()
	safeCall(function()
		if not Cam.ball or not Cam.ball.Parent then return end
		local bp=Cam.ball.Position
		local c=CFG.CAM.LOW_ENDZONE
		local sign=bp.Z>0 and 1 or -1
		Cam.tgtCF=CFrame.new(Vector3.new(0,c.HEIGHT,bp.Z+c.DIST*-sign),bp+Vector3.new(0,1,0))
		Cam.tgtFOV=c.FOV
	end,"CAM_LOW_ENDZONE")
end

local function camSNAP_CAM()
	safeCall(function()
		local qb=findQB()
		if not qb or not qb.Character then return end
		local hrp=getHRP(qb)
		if not hrp then return end
		local c=CFG.CAM.SNAP_CAM
		local look=hrp.CFrame.LookVector
		Cam.tgtCF=CFrame.new(hrp.Position+(look*c.DIST)+Vector3.new(0,c.HEIGHT,0),hrp.Position+look*15+Vector3.new(0,3,0))
		Cam.tgtFOV=c.FOV
	end,"CAM_SNAP")
end

local function camCOACH_CAM()
	safeCall(function()
		local focus=GameState.playerCluster
		local c=CFG.CAM.COACH_CAM
		local rad=math.rad(c.DOWN_ANGLE)
		Cam.tgtCF=CFrame.new(Vector3.new(focus.X+40,c.HEIGHT,focus.Z),focus+Vector3.new(0,-math.sin(rad)*14,math.cos(rad)*14))
		Cam.tgtFOV=c.FOV
	end,"CAM_COACH")
end

local function camTACKLE_CAM()
	safeCall(function()
		local carrier=findBallCarrier() or GameState.ballCarrier
		if not carrier or not carrier.Character then return end
		local hrp=getHRP(carrier)
		if not hrp then return end
		local c=CFG.CAM.TACKLE_CAM
		Cam.tgtCF=CFrame.new(hrp.Position+Vector3.new(c.DIST,c.HEIGHT,0),hrp.Position+Vector3.new(0,1,0))
		Cam.tgtFOV=c.FOV
	end,"CAM_TACKLE")
end

local function camFUMBLE_CAM()
	safeCall(function()
		if not Cam.ball or not Cam.ball.Parent then return end
		local bp=Cam.ball.Position
		local c=CFG.CAM.FUMBLE_CAM
		Cam.tgtCF=CFrame.new(bp+Vector3.new(c.DIST,c.HEIGHT,0),bp)
		Cam.tgtFOV=c.FOV
	end,"CAM_FUMBLE")
end

local function camTACTICAL_CAM()
	safeCall(function()
		if not Cam.ball or not Cam.ball.Parent then return end
		local bp=Cam.ball.Position
		local c=CFG.CAM.TACTICAL_CAM
		Cam.tgtCF=CFrame.new(Vector3.new(bp.X,c.HEIGHT+Cam.dynHeight,bp.Z+math.cos(math.rad(c.ANGLE))*60),bp)
		Cam.tgtFOV=c.FOV+Cam.dynFOV
	end,"CAM_TACTICAL")
end

local function camKICKOFF_CAM()
	safeCall(function()
		if not Cam.ball or not Cam.ball.Parent then return end
		local bp=Cam.ball.Position
		local c=CFG.CAM.KICKOFF_CAM
		if GameState.kicker and (tick()-GameState.kickTime)<c.SWITCH_DELAY then
			local hrp=getHRP(GameState.kicker)
			if hrp then
				Cam.tgtCF=CFrame.new(Vector3.new(c.DISTANCE,c.HEIGHT,hrp.Position.Z+22),bp+Vector3.new(0,5,0))
				Cam.tgtFOV=c.FOV
				return
			end
		end
		Cam.tgtCF=CFrame.new(Vector3.new(c.DISTANCE,c.HEIGHT,bp.Z+16),bp+Vector3.new(0,3,0))
		Cam.tgtFOV=c.FOV
	end,"CAM_KICKOFF")
end

local function camKICKOFF_WIDE()
	safeCall(function()
		if not Cam.ball or not Cam.ball.Parent then return end
		local bp=Cam.ball.Position
		local c=CFG.CAM.KICKOFF_WIDE
		Cam.tgtCF=CFrame.new(Vector3.new(c.DISTANCE,c.HEIGHT,bp.Z+28),bp)
		Cam.tgtFOV=c.FOV
	end,"CAM_KICKOFF_WIDE")
end

local function camKICKOFF_RETURN()
	safeCall(function()
		local ret=GameState.returner or findBallCarrier()
		if not ret or not ret.Character then return end
		local hrp=getHRP(ret)
		if not hrp then return end
		local c=CFG.CAM.KICKOFF_RETURN
		local vel=hrp.AssemblyLinearVelocity or Vector3.new()
		local dir=vel.Magnitude>1 and vel.Unit or hrp.CFrame.LookVector
		Cam.tgtCF=CFrame.new(hrp.Position+(-dir*c.DIST)+Vector3.new(0,c.HEIGHT,0),hrp.Position+dir*c.LEAD+Vector3.new(0,2,0))
		Cam.tgtFOV=c.FOV
	end,"CAM_KICKOFF_RETURN")
end

local function camPUNT_CAM()
	safeCall(function()
		if not Cam.ball or not Cam.ball.Parent then return end
		local bp=Cam.ball.Position
		local spd=getBallSpeed()
		local c=CFG.CAM.PUNT_CAM
		Cam.tgtCF=CFrame.new(Vector3.new(c.DISTANCE+spd*0.18,c.HEIGHT+bp.Y*0.28,bp.Z+12),bp)
		Cam.tgtFOV=c.FOV
	end,"CAM_PUNT")
end

local function camPUNT_HANG()
	safeCall(function()
		if not Cam.ball or not Cam.ball.Parent then return end
		local bp=Cam.ball.Position
		local c=CFG.CAM.PUNT_HANG
		Cam.tgtCF=CFrame.new(Vector3.new(c.DISTANCE,c.HEIGHT,bp.Z+8),bp)
		Cam.tgtFOV=c.FOV
	end,"CAM_PUNT_HANG")
end

local function camPUNT_RETURN()
	safeCall(function()
		local ret=GameState.returner or findBallCarrier()
		if not ret or not ret.Character then return end
		local hrp=getHRP(ret)
		if not hrp then return end
		local c=CFG.CAM.PUNT_RETURN
		local vel=hrp.AssemblyLinearVelocity or Vector3.new()
		local dir=vel.Magnitude>1 and vel.Unit or hrp.CFrame.LookVector
		Cam.tgtCF=CFrame.new(hrp.Position+(-dir*c.DIST)+Vector3.new(0,c.HEIGHT,0),hrp.Position+dir*13)
		Cam.tgtFOV=c.FOV
	end,"CAM_PUNT_RETURN")
end

local function camFIELD_GOAL_CAM()
	safeCall(function()
		if not Cam.ball or not Cam.ball.Parent then return end
		local bp=Cam.ball.Position
		local c=CFG.CAM.FIELD_GOAL_CAM
		if GameState.kicker then
			local hrp=getHRP(GameState.kicker)
			if hrp then
				Cam.tgtCF=CFrame.new(hrp.Position+(-hrp.CFrame.LookVector*c.DISTANCE)+Vector3.new(0,c.HEIGHT,0),bp+Vector3.new(0,2,0))
				Cam.tgtFOV=c.FOV
				return
			end
		end
		Cam.tgtCF=CFrame.new(Vector3.new(c.DISTANCE,c.HEIGHT,bp.Z+6),bp)
		Cam.tgtFOV=c.FOV
	end,"CAM_FG")
end

local function camFIELD_GOAL_SIDE()
	safeCall(function()
		if not Cam.ball or not Cam.ball.Parent then return end
		local bp=Cam.ball.Position
		local c=CFG.CAM.FIELD_GOAL_SIDE
		local rad=math.rad(c.SIDE_ANGLE)
		Cam.tgtCF=CFrame.new(bp+Vector3.new(c.DISTANCE*math.sin(rad),c.HEIGHT,c.DISTANCE*math.cos(rad)),bp)
		Cam.tgtFOV=c.FOV
	end,"CAM_FG_SIDE")
end

local function camFIELD_GOAL_POSTS()
	safeCall(function()
		if not Cam.ball or not Cam.ball.Parent then return end
		local bp=Cam.ball.Position
		local c=CFG.CAM.FIELD_GOAL_POSTS
		local sign=GameState.fieldSide
		Cam.tgtCF=CFrame.new(Vector3.new(0,c.HEIGHT,bp.Z+(c.DISTANCE*sign)),bp+Vector3.new(0,8,0))
		Cam.tgtFOV=c.FOV
	end,"CAM_FG_POSTS")
end

local function camRETURN_CAM()
	safeCall(function()
		local ret=GameState.returner or findBallCarrier()
		if not ret or not ret.Character then return end
		local hrp=getHRP(ret)
		if not hrp then return end
		local c=CFG.CAM.RETURN_CAM
		local vel=hrp.AssemblyLinearVelocity or Vector3.new()
		local dir=vel.Magnitude>1 and vel.Unit or hrp.CFrame.LookVector
		Cam.tgtCF=CFrame.new(hrp.Position+(-dir*(c.DIST+Cam.dynDist))+Vector3.new(0,c.HEIGHT,0),hrp.Position+dir*c.LEAD+Vector3.new(0,2,0))
		Cam.tgtFOV=c.FOV+Cam.dynFOV
	end,"CAM_RETURN")
end

local function camSIDELINE_FOLLOW()
	safeCall(function()
		local carrier=findBallCarrier() or GameState.ballCarrier
		if not carrier or not carrier.Character then return end
		local hrp=getHRP(carrier)
		if not hrp then return end
		local c=CFG.CAM.SIDELINE_FOLLOW
		local vel=hrp.AssemblyLinearVelocity or Vector3.new()
		local dir=vel.Magnitude>2 and vel.Unit or hrp.CFrame.LookVector
		Cam.tgtCF=CFrame.new(Vector3.new(c.DIST,c.HEIGHT,hrp.Position.Z),hrp.Position+dir*11+Vector3.new(0,2,0))
		Cam.tgtFOV=c.FOV
	end,"CAM_SIDELINE_FOLLOW")
end

local function camSIDELINE_STATIC()
	safeCall(function()
		if not Cam.ball or not Cam.ball.Parent then return end
		local bp=Cam.ball.Position
		local c=CFG.CAM.SIDELINE_STATIC
		Cam.tgtCF=CFrame.new(Vector3.new(72,c.HEIGHT,bp.Z),bp+Vector3.new(0,2,0))
		Cam.tgtFOV=c.FOV
	end,"CAM_SIDELINE_STATIC")
end

local function camENDZONE_STD()
	safeCall(function()
		if not Cam.ball or not Cam.ball.Parent then return end
		local bp=Cam.ball.Position
		local c=CFG.CAM.ENDZONE_STD
		local sign=bp.Z>0 and 1 or -1
		Cam.tgtCF=CFrame.new(Vector3.new(0,c.HEIGHT,bp.Z+c.DIST*-sign),bp+Vector3.new(0,2,0))
		Cam.tgtFOV=c.FOV
	end,"CAM_ENDZONE_STD")
end

local function camENDZONE_HIGH()
	safeCall(function()
		if not Cam.ball or not Cam.ball.Parent then return end
		local bp=Cam.ball.Position
		local c=CFG.CAM.ENDZONE_HIGH
		local sign=bp.Z>0 and 1 or -1
		local rad=math.rad(c.DOWN_ANGLE)
		Cam.tgtCF=CFrame.new(Vector3.new(0,c.HEIGHT,bp.Z+c.DIST*-sign),bp+Vector3.new(0,-math.sin(rad)*10,math.cos(rad)*10*sign))
		Cam.tgtFOV=c.FOV
	end,"CAM_ENDZONE_HIGH")
end

local function camLINE_CAM()
	safeCall(function()
		if not Cam.ball or not Cam.ball.Parent then return end
		local bp=Cam.ball.Position
		local c=CFG.CAM.LINE_CAM
		Cam.tgtCF=CFrame.new(Vector3.new(c.DIST,c.HEIGHT,bp.Z),bp+Vector3.new(0,1,0))
		Cam.tgtFOV=c.FOV
	end,"CAM_LINE")
end

local function camPYLON_CAM()
	safeCall(function()
		if not Cam.ball or not Cam.ball.Parent then return end
		local bp=Cam.ball.Position
		local c=CFG.CAM.PYLON_CAM
		local sign=bp.Z>0 and 1 or -1
		Cam.tgtCF=CFrame.new(Vector3.new(c.DIST,c.HEIGHT,sign*CFG.FIELD.ENDZONE_DEPTH),bp)
		Cam.tgtFOV=c.FOV
	end,"CAM_PYLON")
end

local function camTWO_POINT_CAM()
	safeCall(function()
		if not Cam.ball or not Cam.ball.Parent then return end
		local bp=Cam.ball.Position
		local c=CFG.CAM.TWO_POINT_CAM
		local sign=bp.Z>0 and 1 or -1
		Cam.tgtCF=CFrame.new(Vector3.new(0,c.HEIGHT,bp.Z+c.DIST*-sign),bp+Vector3.new(0,2,0))
		Cam.tgtFOV=c.FOV
	end,"CAM_TWO_POINT")
end

local function camONSIDE_CAM()
	safeCall(function()
		if not Cam.ball or not Cam.ball.Parent then return end
		local bp=Cam.ball.Position
		local c=CFG.CAM.ONSIDE_CAM
		Cam.tgtCF=CFrame.new(Vector3.new(c.DIST,c.HEIGHT,bp.Z+10),bp)
		Cam.tgtFOV=c.FOV
	end,"CAM_ONSIDE")
end

local function camCELEBRATION_CAM()
	safeCall(function()
		local carrier=findBallCarrier() or GameState.ballCarrier
		if not carrier or not carrier.Character then return end
		local hrp=getHRP(carrier)
		if not hrp then return end
		local c=CFG.CAM.CELEBRATION_CAM
		local angle=(tick()*c.CIRCLE_SPEED)%(math.pi*2)
		Cam.tgtCF=CFrame.new(hrp.Position+Vector3.new(math.cos(angle)*c.DIST,c.HEIGHT,math.sin(angle)*c.DIST),hrp.Position+Vector3.new(0,3,0))
		Cam.tgtFOV=c.FOV
	end,"CAM_CELEBRATION")
end

local function camCLUSTER_CAM()
	safeCall(function()
		local c=CFG.CAM.CLUSTER_CAM
		local focus=(Cam.ball and Cam.ball.Parent) and Cam.ball.Position or GameState.playerCluster
		local radius=math.max(GameState.clusterRadius,20)
		Cam.tgtCF=CFrame.new(Vector3.new(c.DIST+radius*0.4+Cam.dynDist,c.HEIGHT+radius*0.2+Cam.dynHeight,focus.Z+14),focus+Vector3.new(0,2,0))
		Cam.tgtFOV=c.FOV+Cam.dynFOV+radius*0.15
	end,"CAM_CLUSTER")
end

local function camOVERVIEW_CAM()
	safeCall(function()
		local c=CFG.CAM.OVERVIEW_CAM
		local t=tick()
		local focus=(Cam.ball and Cam.ball.Parent) and Cam.ball.Position or GameState.playerCluster

		if t-OverviewState.lastRecompute>OverviewState.RECOMPUTE_INTERVAL then
			OverviewState.lastRecompute=t
			local minX,maxX,minZ,maxZ=math.huge,-math.huge,math.huge,-math.huge
			local leftCount,rightCount=0,0
			local centroidX,centroidZ,total=0,0,0
			for _,p in ipairs(Players:GetPlayers()) do
				if not isSpectator(p) and p.Character then
					local hrp=p.Character:FindFirstChild("HumanoidRootPart")
					if hrp then
						local pos=hrp.Position
						centroidX=centroidX+pos.X
						centroidZ=centroidZ+pos.Z
						total=total+1
						if pos.X<0 then leftCount=leftCount+1 else rightCount=rightCount+1 end
						if pos.X<minX then minX=pos.X end
						if pos.X>maxX then maxX=pos.X end
						if pos.Z<minZ then minZ=pos.Z end
						if pos.Z>maxZ then maxZ=pos.Z end
					end
				end
			end
			if total==0 then return end
			centroidX=centroidX/total
			centroidZ=centroidZ/total
			local spreadX=math.max(maxX-minX,20)
			local spreadZ=math.max(maxZ-minZ,20)
			local spread=math.sqrt(spreadX*spreadX+spreadZ*spreadZ)
			local height=c.HEIGHT_BASE+spread*c.HEIGHT_SCALE
			local fov=math.min(c.FOV_BASE+spread*c.FOV_SCALE,c.FOV_MAX)
			Cam.tgtFOV=fov
			local sparseSide=(leftCount<=rightCount) and -1 or 1
			local camX=centroidX+sparseSide*(spreadX*0.5+c.SIDE_OFFSET)
			local leadZ=centroidZ+(focus.Z-centroidZ)*c.FORWARD_LEAD
			OverviewState.computedPos=Vector3.new(camX,height,leadZ)
		end

		Cam.tgtCF=CFrame.new(OverviewState.computedPos,focus+Vector3.new(0,4,0))*CFrame.Angles(math.rad(-c.TILT_DOWN),0,0)
	end,"CAM_OVERVIEW")
end

local function povCycleTarget()
	local t=tick()
	if t-PovSys.lastCycle<PovSys.CYCLE_COOLDOWN then return end
	PovSys.lastCycle=t
	PovSys.playerList={}
	for _,p in ipairs(Players:GetPlayers()) do
		if not isSpectator(p) and p.Character and p.Character:FindFirstChild("HumanoidRootPart") then
			table.insert(PovSys.playerList,p)
		end
	end
	if #PovSys.playerList==0 then PovSys.target=nil notify("POV","No players found") return end
	local found=false
	if PovSys.target then
		for i,p in ipairs(PovSys.playerList) do
			if p==PovSys.target then PovSys.playerIdx=(i%#PovSys.playerList)+1 found=true break end
		end
	end
	if not found then PovSys.playerIdx=1 end
	PovSys.target=PovSys.playerList[PovSys.playerIdx]
	PovSys.bobPhase=0
	PovSys.bankAngle=0
	notify("POV",PovSys.target.Name)
end

local function camPOV_CAM(dt)
	safeCall(function()
		dt=dt or (1/60)
		local c=CFG.CAM.POV_CAM
		if not PovSys.target or not PovSys.target.Character then
			for _,p in ipairs(Players:GetPlayers()) do
				if not isSpectator(p) and p.Character and getHRP(p) then PovSys.target=p break end
			end
		end
		if not PovSys.target or not PovSys.target.Character then camBROADCAST() return end
		local hrp=getHRP(PovSys.target)
		if not hrp then camBROADCAST() return end
		local vel=hrp.AssemblyLinearVelocity or Vector3.new()
		local speed=vel.Magnitude
		local eyePos=hrp.Position+Vector3.new(0,c.HEIGHT_OFFSET,0)
		local charLook=hrp.CFrame.LookVector
		local velDir=speed>2 and vel.Unit or charLook
		local blendT=clamp(speed/18,0,1)
		local lookDir=charLook:Lerp(velDir,blendT)
		if lookDir.Magnitude<0.01 then lookDir=charLook end
		lookDir=lookDir.Unit
		PovSys.bobPhase=PovSys.bobPhase+(speed*c.BOB_SPEED*dt)
		local bobOffset=math.sin(PovSys.bobPhase)*c.BOB_AMOUNT*clamp(speed/14,0,1)
		local rightVec=lookDir:Cross(Vector3.new(0,1,0))
		local lateralSpeed=0
		if rightVec.Magnitude>0.01 then lateralSpeed=vel:Dot(rightVec.Unit) end
		local targetBank=clamp(-lateralSpeed*c.TILT_SPEED,-c.MAX_TILT,c.MAX_TILT)
		PovSys.bankAngle=lerp(PovSys.bankAngle,targetBank,0.12)
		local lookTarget=eyePos+lookDir*c.LEAD+Vector3.new(0,bobOffset,0)
		local baseCF=CFrame.new(eyePos+Vector3.new(0,bobOffset,0),lookTarget)
		Cam.tgtCF=baseCF*CFrame.Angles(0,0,math.rad(PovSys.bankAngle))
		Cam.tgtFOV=c.FOV+clamp(speed*0.18,0,14)
	end,"CAM_POV")
end

local CAM_FN={
	BROADCAST=camBROADCAST,HIGH_BROADCAST=camHIGH_BROADCAST,TIGHT_BROADCAST=camTIGHT_BROADCAST,
	HURRY_UP_CAM=camHURRY_UP_CAM,FORMATION_CAM=camFORMATION_CAM,
	SKYCAM=camSKYCAM,ALL_22=camALL_22,
	QB_CAM=camQB_CAM,QB_CLOSEUP=camQB_CLOSEUP,QB_SCRAMBLE=camQB_SCRAMBLE,
	POCKET_CAM=camPOCKET_CAM,BLITZ_CAM=camBLITZ_CAM,SACK_CAM=camSACK_CAM,
	RECEIVER_CAM=camRECEIVER_CAM,RECEIVER_ISO=camRECEIVER_ISO,
	INTERCEPTION=camINTERCEPTION,
	BALL_CAM=camBALL_CAM,BALL_TRACK=camBALL_TRACK,
	BALL_FLIGHT=camBALL_FLIGHT,BALL_SPIRAL=camBALL_SPIRAL,
	CATCH_CAM=camCATCH_CAM,CINEMATIC=camCINEMATIC,
	OVERHEAD_TRACK=camOVERHEAD_TRACK,REVERSE_ANGLE=camREVERSE_ANGLE,
	LOW_ENDZONE=camLOW_ENDZONE,SNAP_CAM=camSNAP_CAM,COACH_CAM=camCOACH_CAM,
	TACKLE_CAM=camTACKLE_CAM,FUMBLE_CAM=camFUMBLE_CAM,
	TACTICAL_CAM=camTACTICAL_CAM,CLUSTER_CAM=camCLUSTER_CAM,
	KICKOFF_CAM=camKICKOFF_CAM,KICKOFF_WIDE=camKICKOFF_WIDE,KICKOFF_RETURN=camKICKOFF_RETURN,
	PUNT_CAM=camPUNT_CAM,PUNT_HANG=camPUNT_HANG,PUNT_RETURN=camPUNT_RETURN,
	FIELD_GOAL_CAM=camFIELD_GOAL_CAM,FIELD_GOAL_SIDE=camFIELD_GOAL_SIDE,FIELD_GOAL_POSTS=camFIELD_GOAL_POSTS,
	RETURN_CAM=camRETURN_CAM,SIDELINE_FOLLOW=camSIDELINE_FOLLOW,SIDELINE_STATIC=camSIDELINE_STATIC,
	ENDZONE_STD=camENDZONE_STD,ENDZONE_HIGH=camENDZONE_HIGH,
	LINE_CAM=camLINE_CAM,PYLON_CAM=camPYLON_CAM,
	TWO_POINT_CAM=camTWO_POINT_CAM,ONSIDE_CAM=camONSIDE_CAM,
	CELEBRATION_CAM=camCELEBRATION_CAM,
	OVERVIEW_CAM=camOVERVIEW_CAM,
	POV_CAM=camPOV_CAM,
}

local SHOTS={
	"BROADCAST","HIGH_BROADCAST","TIGHT_BROADCAST","HURRY_UP_CAM","FORMATION_CAM",
	"SKYCAM","ALL_22","CLUSTER_CAM","TACTICAL_CAM",
	"QB_CAM","QB_CLOSEUP","QB_SCRAMBLE","POCKET_CAM","BLITZ_CAM","SACK_CAM",
	"RECEIVER_CAM","RECEIVER_ISO","INTERCEPTION","CATCH_CAM",
	"BALL_CAM","BALL_TRACK","BALL_FLIGHT","BALL_SPIRAL",
	"TACKLE_CAM","FUMBLE_CAM",
	"SIDELINE_FOLLOW","SIDELINE_STATIC",
	"KICKOFF_CAM","KICKOFF_WIDE","KICKOFF_RETURN",
	"PUNT_CAM","PUNT_HANG","PUNT_RETURN",
	"FIELD_GOAL_CAM","FIELD_GOAL_SIDE","FIELD_GOAL_POSTS",
	"RETURN_CAM",
	"ENDZONE_STD","ENDZONE_HIGH","LINE_CAM","PYLON_CAM",
	"TWO_POINT_CAM","ONSIDE_CAM",
	"CELEBRATION_CAM","CINEMATIC","OVERHEAD_TRACK","REVERSE_ANGLE",
	"LOW_ENDZONE","SNAP_CAM","COACH_CAM",
	"OVERVIEW_CAM",
}

local function scoreShot(s)
	local sc=0
	local ph=GameState.phase
	local pt=GameState.playType
	local inAir=GameState.ballInAir
	local bspd=GameState.ballSpeed
	local an=GameState.analysis

	if s=="BROADCAST" then
		sc=0.46
		if ph=="PRE_SNAP" then sc=sc+0.22 end
		if pt=="RUN" then sc=sc+0.15 end
		if GameState.clusterRadius>28 then sc=sc+0.12 end
		if GameState.isMidfield and ph=="PRE_SNAP" then sc=sc+0.10 end
	elseif s=="HURRY_UP_CAM" then
		sc=0.05
		if an.isHurryUp then sc=sc+0.72 end
		if ph=="PRE_SNAP" and GameState.phaseTime<1.5 then sc=sc+0.30 end
	elseif s=="FORMATION_CAM" then
		sc=0.08
		if ph=="PRE_SNAP" and GameState.phaseTime>3.0 then sc=sc+0.38 end
		if ph=="PRE_SNAP" and GameState.isRedZone then sc=sc+0.22 end
	elseif s=="HIGH_BROADCAST" then
		sc=0.26
		if inAir and bspd>28 then sc=sc+0.36 end
		if pt=="KICKOFF" or pt=="PUNT" then sc=sc+0.20 end
		if an.isHailMary then sc=sc+0.50 end
	elseif s=="TIGHT_BROADCAST" then
		sc=0.16
		if GameState.isRedZone then sc=sc+0.30 end
		if GameState.isGoalLine then sc=sc+0.46 end
	elseif s=="SKYCAM" then
		sc=0.26
		if ph=="PRE_SNAP" then sc=sc+0.30 end
		if pt=="RUN" then sc=sc+0.16 end
		if GameState.isRedZone and ph=="PRE_SNAP" then sc=sc+0.14 end
	elseif s=="ALL_22" then
		sc=0.10
		if ph=="PRE_SNAP" then sc=sc+0.26 end
		if GameState.clusterRadius<22 then sc=sc+0.10 end
	elseif s=="CLUSTER_CAM" then
		sc=0.12
		if pt=="RUN" and GameState.clusterRadius<18 then sc=sc+0.35 end
		if an.isTackle then sc=sc+0.28 end
	elseif s=="QB_CAM" then
		sc=0.16
		if ph=="PRE_SNAP" and GameState.qb then sc=sc+0.36 end
		if ph=="PASS" and GameState.qb then sc=sc+0.16 end
		if an.isHurryUp and GameState.qb then sc=sc+0.20 end
	elseif s=="QB_CLOSEUP" then
		sc=0.06
		if an.isPressure and GameState.qb then sc=sc+0.42 end
		if an.isSack then sc=sc+0.65 end
	elseif s=="QB_SCRAMBLE" then
		sc=0.04
		if an.isScramble then sc=sc+0.82 end
	elseif s=="POCKET_CAM" then
		sc=0.12
		if ph=="PASS" and GameState.qb then sc=sc+0.44 end
		if an.isPressure then sc=sc+0.16 end
		if an.isBlitz then sc=sc+0.24 end
	elseif s=="BLITZ_CAM" then
		sc=0.04
		if an.isBlitz then sc=sc+0.76 end
		if an.rushersNearQB>=4 then sc=sc+0.15 end
	elseif s=="SACK_CAM" then
		sc=0.02
		if an.isSack then sc=sc+0.92 end
	elseif s=="RECEIVER_CAM" then
		sc=0.12
		if ph=="PASS" and #GameState.offense.receivers>0 then sc=sc+0.40 end
		if inAir then sc=sc+0.20 end
		if an.receiverSeparation>8 then sc=sc+0.15 end
	elseif s=="RECEIVER_ISO" then
		sc=0.06
		if inAir and GameState.primaryReceiver then sc=sc+0.45 end
		if an.isHailMary then sc=sc+0.30 end
	elseif s=="CATCH_CAM" then
		sc=0.04
		if an.ballCatchImminent then sc=sc+0.88 end
		if an.catchETA and an.catchETA<0.5 then sc=sc+0.10 end
	elseif s=="INTERCEPTION" then
		sc=0.02
		if an.isInterception then sc=sc+0.95 end
		if inAir and an.defPlayersNearBall>0 and bspd>15 then sc=sc+0.28 end
	elseif s=="BALL_CAM" then
		sc=0.20
		if inAir then sc=sc+0.55 end
		if bspd>28 then sc=sc+0.12 end
		if an.isHailMary then sc=sc+0.25 end
	elseif s=="BALL_TRACK" then
		sc=0.16
		if bspd>38 then sc=sc+0.46 end
		if pt=="KICKOFF" and inAir then sc=sc+0.28 end
	elseif s=="BALL_FLIGHT" then
		sc=0.08
		if inAir and an.ballAirTime>0.3 and an.ballAirTime<2.5 then sc=sc+0.68 end
		if an.isDeepBall then sc=sc+0.20 end
		if an.throwType=="DEEP" then sc=sc+0.15 end
		if an.spiralQuality>0.7 then sc=sc+0.10 end
	elseif s=="BALL_SPIRAL" then
		sc=0.03
		if inAir and an.spiralQuality>0.8 and an.ballAirTime>0.5 then sc=sc+0.55 end
		if an.throwType=="DEEP" and inAir then sc=sc+0.25 end
	elseif s=="TACKLE_CAM" then
		sc=0.04
		if an.isTackle then sc=sc+0.86 end
		if an.isGoalLineTackle then sc=sc+0.10 end
	elseif s=="FUMBLE_CAM" then
		sc=0.02
		if an.isFumble then sc=sc+0.96 end
	elseif s=="SIDELINE_FOLLOW" then
		sc=0.16
		if pt=="RUN" then sc=sc+0.28 end
		if GameState.ballCarrier then sc=sc+0.20 end
		if an.isBreakaway then sc=sc+0.22 end
	elseif s=="SIDELINE_STATIC" then
		sc=0.08
		if pt=="RUN" then sc=sc+0.14 end
		if GameState.isGoalLine then sc=sc+0.12 end
	elseif s=="KICKOFF_CAM" then
		sc=0.03
		if pt=="KICKOFF" then sc=sc+0.90 end
	elseif s=="KICKOFF_WIDE" then
		sc=0.03
		if pt=="KICKOFF" and bspd>38 then sc=sc+0.70 end
		if an.isOnsideKick then sc=sc-0.30 end
	elseif s=="KICKOFF_RETURN" then
		sc=0.03
		if ph=="RETURN" and pt=="KICKOFF" then sc=sc+0.80 end
		if GameState.returner then sc=sc+0.12 end
	elseif s=="PUNT_CAM" then
		sc=0.03
		if pt=="PUNT" then sc=sc+0.86 end
	elseif s=="PUNT_HANG" then
		sc=0.03
		if an.isPuntHang then sc=sc+0.88 end
	elseif s=="PUNT_RETURN" then
		sc=0.03
		if ph=="RETURN" and pt=="PUNT" then sc=sc+0.75 end
	elseif s=="FIELD_GOAL_CAM" then
		sc=0.03
		if pt=="FIELD_GOAL" then sc=sc+0.90 end
	elseif s=="FIELD_GOAL_SIDE" then
		sc=0.03
		if pt=="FIELD_GOAL" and inAir then sc=sc+0.66 end
	elseif s=="FIELD_GOAL_POSTS" then
		sc=0.03
		if pt=="FIELD_GOAL" and inAir and bspd<20 then sc=sc+0.78 end
	elseif s=="RETURN_CAM" then
		sc=0.03
		if ph=="RETURN" then sc=sc+0.80 end
		if an.isBreakaway and ph=="RETURN" then sc=sc+0.15 end
	elseif s=="ENDZONE_STD" then
		sc=0.07
		if GameState.isGoalLine then sc=sc+0.40 end
		if pt=="PASS" and GameState.isRedZone then sc=sc+0.20 end
		if an.isTwoPoint then sc=sc+0.22 end
	elseif s=="ENDZONE_HIGH" then
		sc=0.07
		if GameState.isRedZone then sc=sc+0.28 end
	elseif s=="LINE_CAM" then
		sc=0.06
		if ph=="PRE_SNAP" and GameState.isGoalLine then sc=sc+0.46 end
		if an.isTwoPoint and ph=="PRE_SNAP" then sc=sc+0.30 end
	elseif s=="PYLON_CAM" then
		sc=0.03
		if GameState.isGoalLine and GameState.ballCarrier then sc=sc+0.76 end
	elseif s=="TWO_POINT_CAM" then
		sc=0.02
		if an.isTwoPoint then sc=sc+0.92 end
	elseif s=="ONSIDE_CAM" then
		sc=0.02
		if an.isOnsideKick then sc=sc+0.95 end
	elseif s=="CELEBRATION_CAM" then
		sc=0.02
		if an.touchdown then sc=sc+0.86 end
	elseif s=="TACTICAL_CAM" then
		sc=0.07
		if ph=="PRE_SNAP" then sc=sc+0.24 end
		if GameState.isRedZone and ph=="PRE_SNAP" then sc=sc+0.12 end
	elseif s=="CINEMATIC" then
		sc=0.03
		if an.touchdown then sc=sc+0.40 end
		if ph=="PRE_SNAP" and GameState.phaseTime>4.0 then sc=sc+0.25 end
	elseif s=="OVERHEAD_TRACK" then
		sc=0.06
		if pt=="RUN" and GameState.clusterRadius>25 then sc=sc+0.30 end
		if ph=="RETURN" then sc=sc+0.22 end
	elseif s=="REVERSE_ANGLE" then
		sc=0.08
		if ph=="PRE_SNAP" then sc=sc+0.18 end
		if pt=="RUN" then sc=sc+0.15 end
	elseif s=="LOW_ENDZONE" then
		sc=0.03
		if GameState.isGoalLine and GameState.ballCarrier then sc=sc+0.68 end
		if an.isTwoPoint then sc=sc+0.25 end
	elseif s=="SNAP_CAM" then
		sc=0.04
		if ph=="PRE_SNAP" and GameState.qb and GameState.isGoalLine then sc=sc+0.55 end
	elseif s=="COACH_CAM" then
		sc=0.05
		if ph=="PRE_SNAP" and GameState.phaseTime>3.5 then sc=sc+0.30 end
	elseif s=="OVERVIEW_CAM" then
		sc=0.18
		if GameState.clusterRadius>35 then sc=sc+0.24 end
		if GameState.clusterRadius>50 then sc=sc+0.16 end
		if ph=="RUN" then sc=sc+0.22 end
		if ph=="PASS" then sc=sc+0.18 end
		if ph=="RETURN" then sc=sc+0.30 end
		if pt=="KICKOFF" then sc=sc+0.35 end
		if pt=="PUNT" then sc=sc+0.25 end
		if GameState.isGoalLine then sc=sc-0.20 end
		if an.isBreakaway then sc=sc+0.28 end
		if an.isHailMary then sc=sc+0.32 end
	end

	if an.isBigPlay then
		local bigSafe={SIDELINE_FOLLOW=true,BALL_CAM=true,RETURN_CAM=true,QB_SCRAMBLE=true,CELEBRATION_CAM=true,OVERVIEW_CAM=true,BALL_FLIGHT=true}
		if bigSafe[s] then sc=sc*1.20 end
	end

	local varietyPenalty=0
	for _,prev in ipairs(AI.recentShots) do
		if prev==s then varietyPenalty=varietyPenalty+0.04 end
	end
	sc=sc-varietyPenalty

	return math.min(math.max(sc,0),1.0)
end

local function makeAIDecision()
	safeCall(function()
		if Cam.manual then return end
		local t=tick()
		if AI.urgentOverride and t<AI.urgentUntil then
			if AI.shot~=AI.urgentOverride then
				AI.prevShot=AI.shot
				AI.shot=AI.urgentOverride
				AI.shotStart=t
				Cam:setMode(AI.urgentOverride)
			end
			return
		end
		AI.urgentOverride=nil
		if t-AI.lastDecision<CFG.AI.DECISION_INTERVAL then return end
		AI.lastDecision=t
		local dur=t-AI.shotStart
		if dur<CFG.AI.MIN_SHOT_DURATION then return end
		local force=dur>CFG.AI.MAX_SHOT_DURATION
		calcDynamic()
		local best,bestScore=AI.shot,0
		for _,s in ipairs(SHOTS) do
			local sc=scoreShot(s)
			if s==AI.shot then sc=sc*CFG.AI.HYSTERESIS end
			AI.scores[s]=sc
			if sc>bestScore then bestScore=sc best=s end
		end

		for _,customCam in pairs(EditorSys.customCams) do
			if CAM_FN[customCam.name] then
				local sc=scoreShot(customCam.name)
				if sc>bestScore then bestScore=sc best=customCam.name end
			end
		end

		if (best~=AI.shot and bestScore>CFG.AI.CONFIDENCE_THRESHOLD) or force then
			AI.prevShot=AI.shot
			AI.shot=best
			AI.shotStart=t
			AI.confidence=bestScore
			table.insert(AI.history,{shot=best,t=t,ph=GameState.phase,sc=bestScore})
			if #AI.history>150 then table.remove(AI.history,1) end
			table.insert(AI.recentShots,best)
			if #AI.recentShots>8 then table.remove(AI.recentShots,1) end
			Cam:setMode(best)
		end
	end,"AI_DECISION")
end

local function urgentCut(shot,duration)
	AI.urgentOverride=shot
	AI.urgentUntil=tick()+(duration or 3.5)
end

function Cam:setMode(mode)
	safeCall(function()
		if mode=="AI" or mode=="AI_DIRECTOR" then
			self.manual=false
			notify("AI Director","Auto mode active")
		else
			self.manual=true
			self.mode=mode
			if mode=="POV_CAM" then
				if not PovSys.target then povCycleTarget() end
				notify("POV Cam","Active — J to cycle")
			else
				notify("Manual",mode:gsub("_"," "))
			end
		end
	end,"SET_MODE")
end

function ShaderSys:Enable()
	if self.on then return end
	safeCall(function()
		self.origLight={
			Brightness=Lighting.Brightness,Ambient=Lighting.Ambient,
			OutdoorAmbient=Lighting.OutdoorAmbient,
			ColorShift_Top=Lighting.ColorShift_Top,ColorShift_Bottom=Lighting.ColorShift_Bottom,
			EnvironmentDiffuseScale=Lighting.EnvironmentDiffuseScale,
			EnvironmentSpecularScale=Lighting.EnvironmentSpecularScale,
		}
		for _,n in ipairs({"UFB_CC","UFB_Bloom","UFB_SR","UFB_DOF"}) do
			local e=Lighting:FindFirstChild(n) if e then e:Destroy() end
		end
		local cc=Instance.new("ColorCorrectionEffect")
		cc.Name="UFB_CC" cc.Brightness=CFG.SHADER.BRIGHTNESS cc.Contrast=CFG.SHADER.CONTRAST
		cc.Saturation=CFG.SHADER.SATURATION cc.TintColor=CFG.SHADER.TINT cc.Enabled=true cc.Parent=Lighting
		self.cc=cc
		local bloom=Instance.new("BloomEffect")
		bloom.Name="UFB_Bloom" bloom.Intensity=CFG.SHADER.BLOOM_INTENSITY
		bloom.Size=CFG.SHADER.BLOOM_SIZE bloom.Threshold=CFG.SHADER.BLOOM_THRESHOLD
		bloom.Enabled=true bloom.Parent=Lighting self.bloom=bloom
		local sr=Instance.new("SunRaysEffect")
		sr.Name="UFB_SR" sr.Intensity=CFG.SHADER.SUNRAY_INTENSITY
		sr.Spread=CFG.SHADER.SUNRAY_SPREAD sr.Enabled=true sr.Parent=Lighting self.sunRays=sr
		local dof=Instance.new("DepthOfFieldEffect")
		dof.Name="UFB_DOF" dof.FocusDistance=CFG.SHADER.DOF_FOCUS
		dof.InFocusRadius=CFG.SHADER.DOF_RADIUS dof.NearIntensity=CFG.SHADER.DOF_NEAR
		dof.FarIntensity=CFG.SHADER.DOF_FAR dof.Enabled=true dof.Parent=Lighting self.dof=dof
		if not Lighting:FindFirstChildOfClass("Atmosphere") then
			local atm=Instance.new("Atmosphere")
			atm.Name="UFB_Atm" atm.Density=CFG.SHADER.ATM_DENSITY
			atm.Color=CFG.SHADER.ATM_COLOR atm.Decay=CFG.SHADER.ATM_DECAY
			atm.Glare=CFG.SHADER.ATM_GLARE atm.Haze=CFG.SHADER.ATM_HAZE atm.Parent=Lighting
			self.atm=atm
		end
		Lighting.Brightness=Lighting.Brightness*1.28
		Lighting.EnvironmentDiffuseScale=0.82
		Lighting.EnvironmentSpecularScale=0.82
		self.on=true
		notify("Shaders","Broadcast shaders on")
	end,"SHADER_ENABLE")
end

function ShaderSys:Disable()
	if not self.on then return end
	safeCall(function()
		for _,ref in ipairs({self.cc,self.bloom,self.sunRays,self.dof}) do
			if ref and ref.Parent then ref:Destroy() end
		end
		self.cc=nil self.bloom=nil self.sunRays=nil self.dof=nil
		if self.origLight.Brightness then
			Lighting.Brightness=self.origLight.Brightness
			Lighting.Ambient=self.origLight.Ambient
			Lighting.OutdoorAmbient=self.origLight.OutdoorAmbient
			Lighting.ColorShift_Top=self.origLight.ColorShift_Top
			Lighting.ColorShift_Bottom=self.origLight.ColorShift_Bottom
			Lighting.EnvironmentDiffuseScale=self.origLight.EnvironmentDiffuseScale
			Lighting.EnvironmentSpecularScale=self.origLight.EnvironmentSpecularScale
		end
		self.on=false
		notify("Shaders","Disabled")
	end,"SHADER_DISABLE")
end

function ShaderSys:Toggle() if self.on then self:Disable() else self:Enable() end end

function ShaderSys:Update()
	if not self.on then return end
	local t=tick()
	if t-self.lastUpdate<CFG.SHADER.UPDATE_INTERVAL then return end
	self.lastUpdate=t
	safeCall(function()
		if self.dof and Cam.ball and Cam.ball.Parent then
			local d=(Cam.camera.CFrame.Position-Cam.ball.Position).Magnitude
			self.dof.FocusDistance=d
			self.dof.InFocusRadius=CFG.SHADER.DOF_RADIUS*(d/55)
		end
		if self.bloom then
			local mult=1.0
			if GameState.analysis.touchdown then mult=1.45 end
			if GameState.analysis.isBigPlay then mult=math.max(mult,1.25) end
			self.bloom.Intensity=CFG.SHADER.BLOOM_INTENSITY*mult
		end
	end,"SHADER_UPDATE")
end

local function destroySettingsGui()
	if SettingsSys.gui and SettingsSys.gui.Parent then SettingsSys.gui:Destroy() end
	SettingsSys.gui=nil
	SettingsSys.visible=false
end

local function buildSettingsGui()
	safeCall(function()
		destroySettingsGui()
		local pg=LP:WaitForChild("PlayerGui")
		local sg=Instance.new("ScreenGui")
		sg.Name="UFB_Settings" sg.ResetOnSpawn=false
		sg.DisplayOrder=10000 sg.IgnoreGuiInset=true sg.Parent=pg
		SettingsSys.gui=sg SettingsSys.visible=true

		local panel=Instance.new("Frame")
		panel.Size=UDim2.new(0,280,0,420)
		panel.Position=UDim2.new(0,16,0.5,-210)
		panel.BackgroundColor3=Color3.fromRGB(10,10,10)
		panel.BorderSizePixel=0 panel.ClipsDescendants=true panel.Parent=sg
		local uc=Instance.new("UICorner") uc.CornerRadius=UDim.new(0,6) uc.Parent=panel
		local us=Instance.new("UIStroke") us.Color=Color3.fromRGB(42,42,42) us.Thickness=1 us.Parent=panel

		local accent=Instance.new("Frame")
		accent.Size=UDim2.new(0,3,1,0) accent.BackgroundColor3=Color3.fromRGB(180,180,180)
		accent.BorderSizePixel=0 accent.Parent=panel

		local header=Instance.new("Frame")
		header.Size=UDim2.new(1,-3,0,38) header.Position=UDim2.new(0,3,0,0)
		header.BackgroundColor3=Color3.fromRGB(16,16,16) header.BorderSizePixel=0 header.Parent=panel
		local htl=Instance.new("TextLabel")
		htl.Size=UDim2.new(1,-44,1,0) htl.Position=UDim2.new(0,14,0,0)
		htl.BackgroundTransparency=1 htl.Text="CAMERA SETTINGS"
		htl.TextColor3=Color3.fromRGB(230,230,230) htl.TextSize=11
		htl.Font=Enum.Font.GothamBold htl.TextXAlignment=Enum.TextXAlignment.Left htl.Parent=header

		local vtl=Instance.new("TextLabel")
		vtl.Size=UDim2.new(0,50,1,0) vtl.Position=UDim2.new(1,-52,0,0)
		vtl.BackgroundTransparency=1 vtl.Text="v"..VERSION
		vtl.TextColor3=Color3.fromRGB(55,55,55) vtl.TextSize=9
		vtl.Font=Enum.Font.GothamBold vtl.TextXAlignment=Enum.TextXAlignment.Right vtl.Parent=header

		local closebtn=Instance.new("TextButton")
		closebtn.Size=UDim2.new(0,28,0,28) closebtn.Position=UDim2.new(1,-32,0,5)
		closebtn.BackgroundColor3=Color3.fromRGB(40,40,40) closebtn.BorderSizePixel=0
		closebtn.Text="✕" closebtn.TextColor3=Color3.fromRGB(160,160,160)
		closebtn.TextSize=10 closebtn.Font=Enum.Font.GothamBold closebtn.Parent=header
		Instance.new("UICorner",closebtn).CornerRadius=UDim.new(0,4)
		closebtn.MouseButton1Click:Connect(destroySettingsGui)

		local scroll=Instance.new("ScrollingFrame")
		scroll.Size=UDim2.new(1,-3,1,-38) scroll.Position=UDim2.new(0,3,0,38)
		scroll.BackgroundTransparency=1 scroll.BorderSizePixel=0
		scroll.ScrollBarThickness=3 scroll.ScrollBarImageColor3=Color3.fromRGB(60,60,60)
		scroll.CanvasSize=UDim2.new(0,0,0,520) scroll.Parent=panel

		local yOffset=10

		local function sectionLabel(text)
			local lbl=Instance.new("TextLabel")
			lbl.Size=UDim2.new(1,-20,0,14) lbl.Position=UDim2.new(0,10,0,yOffset)
			lbl.BackgroundTransparency=1 lbl.Text=string.upper(text)
			lbl.TextColor3=Color3.fromRGB(80,80,80) lbl.TextSize=8.5
			lbl.Font=Enum.Font.GothamBold lbl.TextXAlignment=Enum.TextXAlignment.Left lbl.Parent=scroll
			yOffset=yOffset+18
		end

		local function toggleRow(label,getValue,onToggle)
			local row=Instance.new("Frame")
			row.Size=UDim2.new(1,-20,0,30) row.Position=UDim2.new(0,10,0,yOffset)
			row.BackgroundColor3=Color3.fromRGB(18,18,18) row.BorderSizePixel=0 row.Parent=scroll
			Instance.new("UICorner",row).CornerRadius=UDim.new(0,4)
			local rl=Instance.new("TextLabel")
			rl.Size=UDim2.new(1,-60,1,0) rl.Position=UDim2.new(0,12,0,0)
			rl.BackgroundTransparency=1 rl.Text=label
			rl.TextColor3=Color3.fromRGB(190,190,190) rl.TextSize=10
			rl.Font=Enum.Font.Gotham rl.TextXAlignment=Enum.TextXAlignment.Left rl.Parent=row

			local pillBg=Instance.new("Frame")
			pillBg.Size=UDim2.new(0,38,0,18) pillBg.Position=UDim2.new(1,-48,0.5,-9)
			pillBg.BorderSizePixel=0 pillBg.Parent=row
			Instance.new("UICorner",pillBg).CornerRadius=UDim.new(1,0)

			local thumb=Instance.new("Frame")
			thumb.Size=UDim2.new(0,14,0,14) thumb.BackgroundColor3=Color3.fromRGB(255,255,255)
			thumb.BorderSizePixel=0 thumb.Parent=pillBg
			Instance.new("UICorner",thumb).CornerRadius=UDim.new(1,0)

			local function refresh()
				local on=getValue()
				pillBg.BackgroundColor3=on and Color3.fromRGB(70,160,70) or Color3.fromRGB(55,55,55)
				thumb.Position=on and UDim2.new(1,-16,0.5,-7) or UDim2.new(0,2,0.5,-7)
			end
			refresh()

			local btn=Instance.new("TextButton")
			btn.Size=UDim2.new(1,0,1,0) btn.BackgroundTransparency=1
			btn.Text="" btn.Parent=row
			btn.MouseButton1Click:Connect(function() onToggle() refresh() end)
			yOffset=yOffset+36
		end

		local function selectorRow(label,options,getValue,onSelect)
			local row=Instance.new("Frame")
			row.Size=UDim2.new(1,-20,0,30) row.Position=UDim2.new(0,10,0,yOffset)
			row.BackgroundColor3=Color3.fromRGB(18,18,18) row.BorderSizePixel=0 row.Parent=scroll
			Instance.new("UICorner",row).CornerRadius=UDim.new(0,4)
			local rl=Instance.new("TextLabel")
			rl.Size=UDim2.new(0,90,1,0) rl.Position=UDim2.new(0,12,0,0)
			rl.BackgroundTransparency=1 rl.Text=label
			rl.TextColor3=Color3.fromRGB(190,190,190) rl.TextSize=10
			rl.Font=Enum.Font.Gotham rl.TextXAlignment=Enum.TextXAlignment.Left rl.Parent=row

			local btnFrames={}
			local btnW=math.floor(140/#options)
			for i,opt in ipairs(options) do
				local ob=Instance.new("TextButton")
				ob.Size=UDim2.new(0,btnW-2,0,20)
				ob.Position=UDim2.new(0,100+(i-1)*btnW,0.5,-10)
				ob.Text=opt ob.TextSize=8.5 ob.Font=Enum.Font.GothamBold
				ob.BorderSizePixel=0 ob.Parent=row
				Instance.new("UICorner",ob).CornerRadius=UDim.new(0,3)
				table.insert(btnFrames,ob)
				ob.MouseButton1Click:Connect(function()
					onSelect(opt)
					for j,f in ipairs(btnFrames) do
						local active=(options[j]==getValue())
						f.BackgroundColor3=active and Color3.fromRGB(65,65,65) or Color3.fromRGB(30,30,30)
						f.TextColor3=active and Color3.fromRGB(230,230,230) or Color3.fromRGB(100,100,100)
					end
				end)
			end
			for j,f in ipairs(btnFrames) do
				local active=(options[j]==getValue())
				f.BackgroundColor3=active and Color3.fromRGB(65,65,65) or Color3.fromRGB(30,30,30)
				f.TextColor3=active and Color3.fromRGB(230,230,230) or Color3.fromRGB(100,100,100)
			end
			yOffset=yOffset+36
		end

		local function sep()
			local line=Instance.new("Frame")
			line.Size=UDim2.new(1,-20,0,1) line.Position=UDim2.new(0,10,0,yOffset)
			line.BackgroundColor3=Color3.fromRGB(30,30,30) line.BorderSizePixel=0 line.Parent=scroll
			yOffset=yOffset+9
		end

		sectionLabel("Display")
		toggleRow("Show Shot Name",function() return Settings.showShotHud end,function()
			Settings.showShotHud=not Settings.showShotHud
			if ShotHud.frame then ShotHud.frame.Visible=Settings.showShotHud end
		end)
		toggleRow("Notifications",function() return Settings.notificationsOn end,function()
			Settings.notificationsOn=not Settings.notificationsOn
		end)
		sep()

		sectionLabel("Camera")
		selectorRow("Ball Follow",{"LOW","NORMAL","HIGH"},function() return Settings.ballResponsive end,function(v) Settings.ballResponsive=v end)
		toggleRow("Predictive Ball",function() return Settings.predictiveBall end,function()
			Settings.predictiveBall=not Settings.predictiveBall
		end)
		toggleRow("Camera Shake",function() return Settings.cameraShake end,function()
			Settings.cameraShake=not Settings.cameraShake
		end)
		toggleRow("Smart Cuts",function() return Settings.smartCuts end,function()
			Settings.smartCuts=not Settings.smartCuts
		end)
		sep()

		sectionLabel("Visuals")
		toggleRow("Broadcast Shaders",function() return ShaderSys.on end,function()
			if ShaderSys.on then ShaderSys:Disable() else ShaderSys:Enable() end
		end)
		sep()

		sectionLabel("System")
		local rstBtn=Instance.new("TextButton")
		rstBtn.Size=UDim2.new(1,-20,0,28) rstBtn.Position=UDim2.new(0,10,0,yOffset)
		rstBtn.BackgroundColor3=Color3.fromRGB(24,24,24) rstBtn.BorderSizePixel=0
		rstBtn.Text="↺  RESTART CAMERA" rstBtn.TextColor3=Color3.fromRGB(180,180,180) rstBtn.TextSize=10
		rstBtn.Font=Enum.Font.GothamBold rstBtn.Parent=scroll
		Instance.new("UICorner",rstBtn).CornerRadius=UDim.new(0,4)
		Instance.new("UIStroke",rstBtn).Color=Color3.fromRGB(50,50,50)
		rstBtn.MouseButton1Click:Connect(function()
			destroySettingsGui()
			task.spawn(function() task.wait(0.1) restartSystem() end)
		end)
		yOffset=yOffset+34

		local editorBtn=Instance.new("TextButton")
		editorBtn.Size=UDim2.new(1,-20,0,28) editorBtn.Position=UDim2.new(0,10,0,yOffset)
		editorBtn.BackgroundColor3=Color3.fromRGB(24,24,24) editorBtn.BorderSizePixel=0
		editorBtn.Text="🎬  CAMERA EDITOR" editorBtn.TextColor3=Color3.fromRGB(140,180,255) editorBtn.TextSize=10
		editorBtn.Font=Enum.Font.GothamBold editorBtn.Parent=scroll
		Instance.new("UICorner",editorBtn).CornerRadius=UDim.new(0,4)
		Instance.new("UIStroke",editorBtn).Color=Color3.fromRGB(50,50,50)
		editorBtn.MouseButton1Click:Connect(function()
			destroySettingsGui()
			task.spawn(function() task.wait(0.1) buildCameraEditor() end)
		end)
		yOffset=yOffset+40

		local tierInfo=Instance.new("TextLabel")
		tierInfo.Size=UDim2.new(1,-20,0,16) tierInfo.Position=UDim2.new(0,10,0,yOffset)
		tierInfo.BackgroundTransparency=1
		tierInfo.Text="License: "..LicenseData.tier.."  ·  "..LicenseData.username
		tierInfo.TextColor3=Color3.fromRGB(45,45,45) tierInfo.TextSize=8
		tierInfo.Font=Enum.Font.Gotham tierInfo.TextXAlignment=Enum.TextXAlignment.Center
		tierInfo.Parent=scroll
		yOffset=yOffset+20

		local hint=Instance.new("TextLabel")
		hint.Size=UDim2.new(1,-20,0,24) hint.Position=UDim2.new(0,10,0,yOffset)
		hint.BackgroundTransparency=1
		hint.Text="] Settings  ·  [ Shot HUD  ·  ` Restart  ·  \\ Editor"
		hint.TextColor3=Color3.fromRGB(42,42,42) hint.TextSize=8
		hint.Font=Enum.Font.Gotham hint.TextXAlignment=Enum.TextXAlignment.Center
		hint.TextWrapped=true hint.Parent=scroll

		scroll.CanvasSize=UDim2.new(0,0,0,yOffset+40)

		panel.Position=UDim2.new(-0.15,0,0.5,-210)
		TweenService:Create(panel,TweenInfo.new(0.22,Enum.EasingStyle.Quart,Enum.EasingDirection.Out),
			{Position=UDim2.new(0,16,0.5,-210)}):Play()
	end,"BUILD_SETTINGS_GUI")
end

local function toggleSettingsGui()
	if SettingsSys.visible then destroySettingsGui() else buildSettingsGui() end
end

local function destroyCameraEditor()
	if EditorSys.gui and EditorSys.gui.Parent then EditorSys.gui:Destroy() end
	EditorSys.gui=nil EditorSys.visible=false
end

function buildCameraEditor()
	safeCall(function()
		destroyCameraEditor()
		local pg=LP:WaitForChild("PlayerGui")
		local sg=Instance.new("ScreenGui")
		sg.Name="UFB_Editor"
		sg.ResetOnSpawn=false
		sg.DisplayOrder=10001
		sg.IgnoreGuiInset=true
		sg.Parent=pg
		EditorSys.gui=sg
		EditorSys.visible=true

		-- ─────────────────────────────────────────────────────────────────
		-- BACKDROP
		-- ─────────────────────────────────────────────────────────────────
		local backdrop=Instance.new("Frame")
		backdrop.Size=UDim2.new(1,0,1,0)
		backdrop.BackgroundColor3=Color3.fromRGB(0,0,0)
		backdrop.BackgroundTransparency=0.55
		backdrop.BorderSizePixel=0
		backdrop.Parent=sg

		-- ─────────────────────────────────────────────────────────────────
		-- PANEL  (no rounded corners — sharp-edged client aesthetic)
		-- ─────────────────────────────────────────────────────────────────
		local W,H=720,640
		local panel=Instance.new("Frame")
		panel.Size=UDim2.new(0,W,0,H)
		panel.Position=UDim2.new(0.5,-W/2,0.5,-H/2)
		panel.BackgroundColor3=Color3.fromRGB(8,8,8)
		panel.BorderSizePixel=0
		panel.ClipsDescendants=true
		panel.Parent=sg

		-- Outer border stroke
		local outerStroke=Instance.new("UIStroke",panel)
		outerStroke.Color=Color3.fromRGB(45,45,45)
		outerStroke.Thickness=1

		-- Top highlight line
		local topHL=Instance.new("Frame")
		topHL.Size=UDim2.new(1,0,0,1)
		topHL.BackgroundColor3=Color3.fromRGB(255,255,255)
		topHL.BackgroundTransparency=0.80
		topHL.BorderSizePixel=0
		topHL.Parent=panel

		-- ─────────────────────────────────────────────────────────────────
		-- TITLE BAR
		-- ─────────────────────────────────────────────────────────────────
		local titleBar=Instance.new("TextButton")
		titleBar.Size=UDim2.new(1,0,0,46)
		titleBar.BackgroundColor3=Color3.fromRGB(11,11,11)
		titleBar.BorderSizePixel=0
		titleBar.Text=""
		titleBar.AutoButtonColor=false
		titleBar.ZIndex=10
		titleBar.Parent=panel

		-- Title bar separator
		local titleSep=Instance.new("Frame")
		titleSep.Size=UDim2.new(1,0,0,1)
		titleSep.Position=UDim2.new(0,0,1,0)
		titleSep.BackgroundColor3=Color3.fromRGB(255,255,255)
		titleSep.BackgroundTransparency=0.87
		titleSep.BorderSizePixel=0
		titleSep.ZIndex=10
		titleSep.Parent=titleBar

		-- Left accent stripe
		local accentStripe=Instance.new("Frame")
		accentStripe.Size=UDim2.new(0,2,1,0)
		accentStripe.BackgroundColor3=Color3.fromRGB(255,255,255)
		accentStripe.BackgroundTransparency=0.45
		accentStripe.BorderSizePixel=0
		accentStripe.ZIndex=10
		accentStripe.Parent=titleBar

		-- App identifier dots (macOS-inspired but white)
		for i,tr in ipairs({0,0.6,0.78}) do
			local dot=Instance.new("Frame")
			dot.Size=UDim2.new(0,8,0,8)
			dot.Position=UDim2.new(0,(i-1)*14+10,0.5,-4)
			dot.BackgroundColor3=Color3.fromRGB(255,255,255)
			dot.BackgroundTransparency=tr
			dot.BorderSizePixel=0
			dot.ZIndex=11
			dot.Parent=titleBar
			Instance.new("UICorner",dot).CornerRadius=UDim.new(1,0)
		end

		-- Title text
		local titleLbl=Instance.new("TextLabel")
		titleLbl.Size=UDim2.new(1,-220,1,0)
		titleLbl.Position=UDim2.new(0,56,0,0)
		titleLbl.BackgroundTransparency=1
		titleLbl.Text="CAMERA EDITOR"
		titleLbl.TextColor3=Color3.fromRGB(255,255,255)
		titleLbl.TextTransparency=0.05
		titleLbl.TextSize=11
		titleLbl.Font=Enum.Font.GothamBold
		titleLbl.TextXAlignment=Enum.TextXAlignment.Left
		titleLbl.ZIndex=10
		titleLbl.Parent=titleBar

		-- Version / tier badge
		local tierBadge=Instance.new("Frame")
		tierBadge.Size=UDim2.new(0,90,0,18)
		tierBadge.Position=UDim2.new(0,56+120,0.5,-9)
		tierBadge.BackgroundColor3=Color3.fromRGB(255,255,255)
		tierBadge.BackgroundTransparency=0.88
		tierBadge.BorderSizePixel=0
		tierBadge.ZIndex=10
		tierBadge.Parent=titleBar
		Instance.new("UICorner",tierBadge).CornerRadius=UDim.new(0,2)
		local tierLbl=Instance.new("TextLabel")
		tierLbl.Size=UDim2.new(1,0,1,0)
		tierLbl.BackgroundTransparency=1
		tierLbl.Text="UNLIMITED"
		tierLbl.TextColor3=Color3.fromRGB(255,255,255)
		tierLbl.TextTransparency=0.3
		tierLbl.TextSize=7.5
		tierLbl.Font=Enum.Font.GothamBold
		tierLbl.ZIndex=11
		tierLbl.Parent=tierBadge

		-- Window controls
		local function mkWinBtn(offsetR,label)
			local b=Instance.new("TextButton")
			b.Size=UDim2.new(0,28,0,28)
			b.Position=UDim2.new(1,-offsetR,0.5,-14)
			b.BackgroundColor3=Color3.fromRGB(255,255,255)
			b.BackgroundTransparency=0.93
			b.BorderSizePixel=0
			b.Text=label
			b.TextColor3=Color3.fromRGB(255,255,255)
			b.TextTransparency=0.4
			b.TextSize=10
			b.Font=Enum.Font.GothamBold
			b.ZIndex=12
			b.Parent=titleBar
			Instance.new("UICorner",b).CornerRadius=UDim.new(0,2)
			b.MouseEnter:Connect(function()
				TweenService:Create(b,TweenInfo.new(0.1),{BackgroundTransparency=0.72,TextTransparency=0}):Play()
			end)
			b.MouseLeave:Connect(function()
				TweenService:Create(b,TweenInfo.new(0.12),{BackgroundTransparency=0.93,TextTransparency=0.4}):Play()
			end)
			return b
		end
		local closeBtn=mkWinBtn(36,"✕")
		closeBtn.MouseButton1Click:Connect(destroyCameraEditor)

		local minimized=false
		local minBtn=mkWinBtn(70,"—")
		minBtn.MouseButton1Click:Connect(function()
			minimized=not minimized
			TweenService:Create(panel,TweenInfo.new(0.2,Enum.EasingStyle.Quart,Enum.EasingDirection.Out),
				{Size=UDim2.new(0,W,0,minimized and 46 or H)}):Play()
		end)

		-- ─────────────────────────────────────────────────────────────────
		-- DRAG
		-- ─────────────────────────────────────────────────────────────────
		local dragging,dragStart,startPos=false,nil,nil
		titleBar.InputBegan:Connect(function(i)
			if i.UserInputType==Enum.UserInputType.MouseButton1 then
				dragging=true dragStart=i.Position startPos=panel.Position
			end
		end)
		titleBar.InputEnded:Connect(function(i)
			if i.UserInputType==Enum.UserInputType.MouseButton1 then dragging=false end
		end)
		UIS.InputChanged:Connect(function(i)
			if dragging and i.UserInputType==Enum.UserInputType.MouseMovement then
				local d=i.Position-dragStart
				panel.Position=UDim2.new(startPos.X.Scale,startPos.X.Offset+d.X,startPos.Y.Scale,startPos.Y.Offset+d.Y)
			end
		end)

		-- ─────────────────────────────────────────────────────────────────
		-- SIDEBAR
		-- ─────────────────────────────────────────────────────────────────
		local SW=130
		local sidebar=Instance.new("Frame")
		sidebar.Size=UDim2.new(0,SW,1,-46)
		sidebar.Position=UDim2.new(0,0,0,46)
		sidebar.BackgroundColor3=Color3.fromRGB(10,10,10)
		sidebar.BorderSizePixel=0
		sidebar.ZIndex=5
		sidebar.Parent=panel

		local sideR=Instance.new("Frame")
		sideR.Size=UDim2.new(0,1,1,0)
		sideR.Position=UDim2.new(1,-1,0,0)
		sideR.BackgroundColor3=Color3.fromRGB(255,255,255)
		sideR.BackgroundTransparency=0.88
		sideR.BorderSizePixel=0
		sideR.Parent=sidebar

		-- ─────────────────────────────────────────────────────────────────
		-- CONTENT AREA
		-- ─────────────────────────────────────────────────────────────────
		local contentArea=Instance.new("Frame")
		contentArea.Size=UDim2.new(1,-SW,1,-68)  -- leave 22px for status bar
		contentArea.Position=UDim2.new(0,SW,0,46)
		contentArea.BackgroundTransparency=1
		contentArea.BorderSizePixel=0
		contentArea.ClipsDescendants=true
		contentArea.ZIndex=2
		contentArea.Parent=panel

		-- ─────────────────────────────────────────────────────────────────
		-- HELPERS
		-- ─────────────────────────────────────────────────────────────────
		local function makePage()
			local sf=Instance.new("ScrollingFrame")
			sf.Size=UDim2.new(1,0,1,0)
			sf.BackgroundTransparency=1
			sf.BorderSizePixel=0
			sf.ScrollBarThickness=2
			sf.ScrollBarImageColor3=Color3.fromRGB(255,255,255)
			sf.ScrollBarImageTransparency=0.75
			sf.CanvasSize=UDim2.new(0,0,0,1200)
			sf.Visible=false
			sf.ZIndex=2
			sf.Parent=contentArea
			return sf
		end

		local function secHead(parent,y,txt)
			local f=Instance.new("Frame")
			f.Size=UDim2.new(1,-24,0,26)
			f.Position=UDim2.new(0,12,0,y)
			f.BackgroundTransparency=1
			f.ZIndex=3
			f.Parent=parent

			local sep=Instance.new("Frame")
			sep.Size=UDim2.new(1,0,0,1)
			sep.Position=UDim2.new(0,0,1,0)
			sep.BackgroundColor3=Color3.fromRGB(255,255,255)
			sep.BackgroundTransparency=0.90
			sep.BorderSizePixel=0
			sep.ZIndex=3
			sep.Parent=f

			local lbl=Instance.new("TextLabel")
			lbl.Size=UDim2.new(1,0,1,0)
			lbl.BackgroundTransparency=1
			lbl.Text=string.upper(txt)
			lbl.TextColor3=Color3.fromRGB(255,255,255)
			lbl.TextTransparency=0.62
			lbl.TextSize=8
			lbl.Font=Enum.Font.GothamBold
			lbl.TextXAlignment=Enum.TextXAlignment.Left
			lbl.ZIndex=3
			lbl.Parent=f
			return y+34
		end

		-- Slider: returns nextY, getter(), setter(v)
		local function mkSlider(parent,y,lbl,mn,mx,def,dec,cb)
			dec=dec or 0
			local fmt=dec>0 and ("%."..dec.."f") or "%d"
			local row=Instance.new("Frame")
			row.Size=UDim2.new(1,-24,0,46)
			row.Position=UDim2.new(0,12,0,y)
			row.BackgroundColor3=Color3.fromRGB(255,255,255)
			row.BackgroundTransparency=0.965
			row.BorderSizePixel=0
			row.ZIndex=3
			row.Parent=parent
			Instance.new("UICorner",row).CornerRadius=UDim.new(0,2)

			local nameLbl=Instance.new("TextLabel")
			nameLbl.Size=UDim2.new(0.58,0,0,14)
			nameLbl.Position=UDim2.new(0,10,0,7)
			nameLbl.BackgroundTransparency=1
			nameLbl.Text=lbl
			nameLbl.TextColor3=Color3.fromRGB(255,255,255)
			nameLbl.TextTransparency=0.2
			nameLbl.TextSize=9
			nameLbl.Font=Enum.Font.GothamBold
			nameLbl.TextXAlignment=Enum.TextXAlignment.Left
			nameLbl.ZIndex=4
			nameLbl.Parent=row

			local valLbl=Instance.new("TextLabel")
			valLbl.Size=UDim2.new(0.38,0,0,14)
			valLbl.Position=UDim2.new(0.6,0,0,7)
			valLbl.BackgroundTransparency=1
			valLbl.Text=string.format(fmt,def)
			valLbl.TextColor3=Color3.fromRGB(255,255,255)
			valLbl.TextTransparency=0.48
			valLbl.TextSize=9
			valLbl.Font=Enum.Font.GothamBold
			valLbl.TextXAlignment=Enum.TextXAlignment.Right
			valLbl.ZIndex=4
			valLbl.Parent=row

			-- Range labels
			local minLbl=Instance.new("TextLabel")
			minLbl.Size=UDim2.new(0,28,0,10)
			minLbl.Position=UDim2.new(0,10,0,33)
			minLbl.BackgroundTransparency=1
			minLbl.Text=string.format(fmt,mn)
			minLbl.TextColor3=Color3.fromRGB(255,255,255)
			minLbl.TextTransparency=0.78
			minLbl.TextSize=7
			minLbl.Font=Enum.Font.Gotham
			minLbl.TextXAlignment=Enum.TextXAlignment.Left
			minLbl.ZIndex=4
			minLbl.Parent=row

			local maxLbl=Instance.new("TextLabel")
			maxLbl.Size=UDim2.new(0,28,0,10)
			maxLbl.Position=UDim2.new(1,-38,0,33)
			maxLbl.BackgroundTransparency=1
			maxLbl.Text=string.format(fmt,mx)
			maxLbl.TextColor3=Color3.fromRGB(255,255,255)
			maxLbl.TextTransparency=0.78
			maxLbl.TextSize=7
			maxLbl.Font=Enum.Font.Gotham
			maxLbl.TextXAlignment=Enum.TextXAlignment.Right
			maxLbl.ZIndex=4
			maxLbl.Parent=row

			local track=Instance.new("Frame")
			track.Size=UDim2.new(1,-20,0,2)
			track.Position=UDim2.new(0,10,0,27)
			track.BackgroundColor3=Color3.fromRGB(255,255,255)
			track.BackgroundTransparency=0.87
			track.BorderSizePixel=0
			track.ZIndex=4
			track.Parent=row
			Instance.new("UICorner",track).CornerRadius=UDim.new(1,0)

			local p0=(def-mn)/(mx-mn)
			local fill=Instance.new("Frame")
			fill.Size=UDim2.new(p0,0,1,0)
			fill.BackgroundColor3=Color3.fromRGB(255,255,255)
			fill.BorderSizePixel=0
			fill.ZIndex=5
			fill.Parent=track
			Instance.new("UICorner",fill).CornerRadius=UDim.new(1,0)

			local knob=Instance.new("Frame")
			knob.Size=UDim2.new(0,9,0,9)
			knob.Position=UDim2.new(p0,-4,0.5,-4)
			knob.BackgroundColor3=Color3.fromRGB(255,255,255)
			knob.BorderSizePixel=0
			knob.ZIndex=6
			knob.Parent=track
			Instance.new("UICorner",knob).CornerRadius=UDim.new(1,0)

			local hit=Instance.new("TextButton")
			hit.Size=UDim2.new(1,0,0,22)
			hit.Position=UDim2.new(0,0,0,-10)
			hit.BackgroundTransparency=1
			hit.Text=""
			hit.ZIndex=7
			hit.Parent=track

			local cur=def
			local sliding=false

			local function applyRel(rel)
				rel=clamp(rel,0,1)
				local raw=mn+rel*(mx-mn)
				local m=10^dec
				cur=math.floor(raw*m+0.5)/m
				fill.Size=UDim2.new(rel,0,1,0)
				knob.Position=UDim2.new(rel,-4,0.5,-4)
				valLbl.Text=string.format(fmt,cur)
				if cb then cb(cur) end
			end

			hit.InputBegan:Connect(function(i)
				if i.UserInputType==Enum.UserInputType.MouseButton1 then
					sliding=true
					applyRel((i.Position.X-track.AbsolutePosition.X)/track.AbsoluteSize.X)
				end
			end)
			hit.InputEnded:Connect(function(i)
				if i.UserInputType==Enum.UserInputType.MouseButton1 then sliding=false end
			end)
			UIS.InputChanged:Connect(function(i)
				if sliding and i.UserInputType==Enum.UserInputType.MouseMovement then
					applyRel((i.Position.X-track.AbsolutePosition.X)/track.AbsoluteSize.X)
				end
			end)

			return y+54,
				function() return cur end,
				function(v)
					cur=v
					local r=clamp((v-mn)/(mx-mn),0,1)
					fill.Size=UDim2.new(r,0,1,0)
					knob.Position=UDim2.new(r,-4,0.5,-4)
					valLbl.Text=string.format(fmt,cur)
				end
		end

		-- Toggle: returns nextY, getter()
		local function mkToggle(parent,y,lbl,def,cb)
			local row=Instance.new("Frame")
			row.Size=UDim2.new(1,-24,0,36)
			row.Position=UDim2.new(0,12,0,y)
			row.BackgroundColor3=Color3.fromRGB(255,255,255)
			row.BackgroundTransparency=0.965
			row.BorderSizePixel=0
			row.ZIndex=3
			row.Parent=parent
			Instance.new("UICorner",row).CornerRadius=UDim.new(0,2)

			local nameLbl=Instance.new("TextLabel")
			nameLbl.Size=UDim2.new(1,-64,1,0)
			nameLbl.Position=UDim2.new(0,10,0,0)
			nameLbl.BackgroundTransparency=1
			nameLbl.Text=lbl
			nameLbl.TextColor3=Color3.fromRGB(255,255,255)
			nameLbl.TextTransparency=0.25
			nameLbl.TextSize=9.5
			nameLbl.Font=Enum.Font.GothamBold
			nameLbl.TextXAlignment=Enum.TextXAlignment.Left
			nameLbl.ZIndex=4
			nameLbl.Parent=row

			local pill=Instance.new("Frame")
			pill.Size=UDim2.new(0,36,0,18)
			pill.Position=UDim2.new(1,-50,0.5,-9)
			pill.BackgroundColor3=def and Color3.fromRGB(255,255,255) or Color3.fromRGB(38,38,38)
			pill.BorderSizePixel=0
			pill.ZIndex=4
			pill.Parent=row
			Instance.new("UICorner",pill).CornerRadius=UDim.new(1,0)
			local ps=Instance.new("UIStroke",pill)
			ps.Color=Color3.fromRGB(70,70,70)
			ps.Thickness=1

			local knob=Instance.new("Frame")
			knob.Size=UDim2.new(0,13,0,13)
			knob.Position=def and UDim2.new(1,-15,0.5,-6) or UDim2.new(0,2,0.5,-6)
			knob.BackgroundColor3=def and Color3.fromRGB(0,0,0) or Color3.fromRGB(110,110,110)
			knob.BorderSizePixel=0
			knob.ZIndex=5
			knob.Parent=pill
			Instance.new("UICorner",knob).CornerRadius=UDim.new(1,0)

			local state=def
			local hit=Instance.new("TextButton")
			hit.Size=UDim2.new(1,0,1,0)
			hit.BackgroundTransparency=1
			hit.Text=""
			hit.ZIndex=6
			hit.Parent=row
			hit.MouseButton1Click:Connect(function()
				state=not state
				TweenService:Create(pill,TweenInfo.new(0.14),{
					BackgroundColor3=state and Color3.fromRGB(255,255,255) or Color3.fromRGB(38,38,38)
				}):Play()
				TweenService:Create(knob,TweenInfo.new(0.14),{
					Position=state and UDim2.new(1,-15,0.5,-6) or UDim2.new(0,2,0.5,-6),
					BackgroundColor3=state and Color3.fromRGB(0,0,0) or Color3.fromRGB(110,110,110)
				}):Play()
				if cb then cb(state) end
			end)
			return y+44, function() return state end
		end

		-- TextInput: returns nextY, box
		local function mkInput(parent,y,placeholder,default)
			local box=Instance.new("TextBox")
			box.Size=UDim2.new(1,-24,0,34)
			box.Position=UDim2.new(0,12,0,y)
			box.BackgroundColor3=Color3.fromRGB(255,255,255)
			box.BackgroundTransparency=0.94
			box.BorderSizePixel=0
			box.PlaceholderText=placeholder
			box.PlaceholderColor3=Color3.fromRGB(255,255,255)
			box.Text=default or ""
			box.TextColor3=Color3.fromRGB(255,255,255)
			box.TextTransparency=0.08
			box.TextSize=10
			box.Font=Enum.Font.GothamBold
			box.ClearTextOnFocus=false
			box.ZIndex=4
			box.Parent=parent
			Instance.new("UICorner",box).CornerRadius=UDim.new(0,2)
			local pad=Instance.new("UIPadding",box)
			pad.PaddingLeft=UDim.new(0,10)
			local stroke=Instance.new("UIStroke",box)
			stroke.Color=Color3.fromRGB(255,255,255)
			stroke.Transparency=0.87
			box.Focused:Connect(function()
				TweenService:Create(stroke,TweenInfo.new(0.12),{Transparency=0.55}):Play()
			end)
			box.FocusLost:Connect(function()
				TweenService:Create(stroke,TweenInfo.new(0.18),{Transparency=0.87}):Play()
			end)
			return y+42, box
		end

		-- Button: returns nextY, btn
		local function mkBtn(parent,y,txt,primary)
			local btn=Instance.new("TextButton")
			btn.Size=UDim2.new(1,-24,0,34)
			btn.Position=UDim2.new(0,12,0,y)
			btn.BackgroundColor3=Color3.fromRGB(255,255,255)
			btn.BackgroundTransparency=primary and 0 or 0.92
			btn.BorderSizePixel=0
			btn.Text=txt
			btn.TextColor3=primary and Color3.fromRGB(0,0,0) or Color3.fromRGB(255,255,255)
			btn.TextTransparency=primary and 0 or 0.18
			btn.TextSize=10
			btn.Font=Enum.Font.GothamBold
			btn.ZIndex=4
			btn.Parent=parent
			Instance.new("UICorner",btn).CornerRadius=UDim.new(0,2)
			if not primary then
				local st=Instance.new("UIStroke",btn)
				st.Color=Color3.fromRGB(255,255,255)
				st.Transparency=0.82
			end
			btn.MouseEnter:Connect(function()
				TweenService:Create(btn,TweenInfo.new(0.1),{
					BackgroundTransparency=primary and 0.1 or 0.80
				}):Play()
			end)
			btn.MouseLeave:Connect(function()
				TweenService:Create(btn,TweenInfo.new(0.14),{
					BackgroundTransparency=primary and 0 or 0.92
				}):Play()
			end)
			return y+42, btn
		end

		-- Split-row: two half-width buttons side by side
		local function mkBtnRow(parent,y,txtL,txtR)
			local gap=8
			local halfW=(contentArea.AbsoluteSize.X-24-gap)/2
			local function halfBtn(xOff,txt)
				local b=Instance.new("TextButton")
				b.Size=UDim2.new(0,halfW,0,34)
				b.Position=UDim2.new(0,xOff,0,y)
				b.BackgroundColor3=Color3.fromRGB(255,255,255)
				b.BackgroundTransparency=0.92
				b.BorderSizePixel=0
				b.Text=txt
				b.TextColor3=Color3.fromRGB(255,255,255)
				b.TextTransparency=0.18
				b.TextSize=9.5
				b.Font=Enum.Font.GothamBold
				b.ZIndex=4
				b.Parent=parent
				Instance.new("UICorner",b).CornerRadius=UDim.new(0,2)
				local st=Instance.new("UIStroke",b)
				st.Color=Color3.fromRGB(255,255,255)
				st.Transparency=0.82
				b.MouseEnter:Connect(function()
					TweenService:Create(b,TweenInfo.new(0.1),{BackgroundTransparency=0.78}):Play()
				end)
				b.MouseLeave:Connect(function()
					TweenService:Create(b,TweenInfo.new(0.14),{BackgroundTransparency=0.92}):Play()
				end)
				return b
			end
			local bL=halfBtn(12,txtL)
			local bR=halfBtn(12+halfW+gap,txtR)
			return y+42,bL,bR
		end

		-- DropDown: returns nextY, getter()
		local function mkDropdown(parent,y,lbl,options,defIdx)
			local selIdx=defIdx or 1
			local row=Instance.new("Frame")
			row.Size=UDim2.new(1,-24,0,36)
			row.Position=UDim2.new(0,12,0,y)
			row.BackgroundColor3=Color3.fromRGB(255,255,255)
			row.BackgroundTransparency=0.965
			row.BorderSizePixel=0
			row.ZIndex=3
			row.Parent=parent
			Instance.new("UICorner",row).CornerRadius=UDim.new(0,2)

			local nameLbl=Instance.new("TextLabel")
			nameLbl.Size=UDim2.new(0.45,0,1,0)
			nameLbl.Position=UDim2.new(0,10,0,0)
			nameLbl.BackgroundTransparency=1
			nameLbl.Text=lbl
			nameLbl.TextColor3=Color3.fromRGB(255,255,255)
			nameLbl.TextTransparency=0.25
			nameLbl.TextSize=9
			nameLbl.Font=Enum.Font.GothamBold
			nameLbl.TextXAlignment=Enum.TextXAlignment.Left
			nameLbl.ZIndex=4
			nameLbl.Parent=row

			local selBtn=Instance.new("TextButton")
			selBtn.Size=UDim2.new(0.50,0,0,24)
			selBtn.Position=UDim2.new(0.48,0,0.5,-12)
			selBtn.BackgroundColor3=Color3.fromRGB(255,255,255)
			selBtn.BackgroundTransparency=0.88
			selBtn.BorderSizePixel=0
			selBtn.Text=options[selIdx].." ▾"
			selBtn.TextColor3=Color3.fromRGB(255,255,255)
			selBtn.TextTransparency=0.2
			selBtn.TextSize=8.5
			selBtn.Font=Enum.Font.GothamBold
			selBtn.ZIndex=5
			selBtn.Parent=row
			Instance.new("UICorner",selBtn).CornerRadius=UDim.new(0,2)

			-- Cycle on click (simple cycling dropdown for Roblox)
			selBtn.MouseButton1Click:Connect(function()
				selIdx=(selIdx%#options)+1
				selBtn.Text=options[selIdx].." ▾"
			end)
			return y+44, function() return options[selIdx] end
		end

		-- ─────────────────────────────────────────────────────────────────
		-- TAB DATA
		-- ─────────────────────────────────────────────────────────────────
		local tabDefs={
			{id="BUILD",   icon="◈",label="BUILD"},
			{id="PRESETS", icon="▦",label="PRESETS"},
			{id="LIBRARY", icon="≡",label="LIBRARY"},
			{id="TUNING",  icon="⚙",label="TUNING"},
			{id="LIVE",    icon="◉",label="LIVE"},
		}
		local tabBtns={}
		local tabPages={}
		local activeTab="BUILD"

		-- ─────────────────────────────────────────────────────────────────
		-- SHARED SLIDER STATE (shared between BUILD & PRESETS loading)
		-- ─────────────────────────────────────────────────────────────────
		local SV={
			HEIGHT=44, DISTANCE=82, FOV=66, SMOOTH=0.055,
			TILT=8,    FOLLOW_Z=18, LEAD=12, ORBIT_RADIUS=20,
			ORBIT_SPEED=0.30, SIDE_OFFSET=0, MIN_HEIGHT=3, MAX_HEIGHT=160,
			SHAKE_INTENSITY=0.4, SHAKE_FREQ=0.08,
		}
		local SR={}  -- setter references: SR[key](v)
		local nameBox=nil  -- set during BUILD tab construction

		-- ─────────────────────────────────────────────────────────────────
		-- ════ TAB 1: BUILD ════════════════════════════════════════════════
		-- ─────────────────────────────────────────────────────────────────
		local buildPage=makePage()
		tabPages["BUILD"]=buildPage

		local bY=14

		-- ── Identity ──
		bY=secHead(buildPage,bY,"Camera Identity")
		bY,nameBox=mkInput(buildPage,bY,"NAME  (e.g.  MY_ENDZONE_CAM)","")

		-- Base type chips
		do
			local typeLbl=Instance.new("TextLabel")
			typeLbl.Size=UDim2.new(1,-24,0,11)
			typeLbl.Position=UDim2.new(0,12,0,bY)
			typeLbl.BackgroundTransparency=1
			typeLbl.Text="BASE TYPE"
			typeLbl.TextColor3=Color3.fromRGB(255,255,255)
			typeLbl.TextTransparency=0.65
			typeLbl.TextSize=7.5
			typeLbl.Font=Enum.Font.GothamBold
			typeLbl.TextXAlignment=Enum.TextXAlignment.Left
			typeLbl.ZIndex=3
			typeLbl.Parent=buildPage
			bY=bY+16

			local types={"BROADCAST","FOLLOW","ORBIT","SIDELINE","AERIAL"}
			local selType="BROADCAST"
			local chipBtns={}
			local cW=math.floor((contentArea.AbsoluteSize.X-24-4*(#types-1))/#types)
			for i2,tp in ipairs(types) do
				local c=Instance.new("TextButton")
				c.Size=UDim2.new(0,cW,0,26)
				c.Position=UDim2.new(0,12+(i2-1)*(cW+4),0,bY)
				c.BackgroundColor3=Color3.fromRGB(255,255,255)
				c.BackgroundTransparency=tp==selType and 0.82 or 0.94
				c.BorderSizePixel=0
				c.Text=tp
				c.TextColor3=Color3.fromRGB(255,255,255)
				c.TextTransparency=tp==selType and 0.08 or 0.52
				c.TextSize=7.5
				c.Font=Enum.Font.GothamBold
				c.ZIndex=4
				c.Parent=buildPage
				Instance.new("UICorner",c).CornerRadius=UDim.new(0,2)
				chipBtns[tp]=c
				c.MouseButton1Click:Connect(function()
					selType=tp
					for _,k in pairs(chipBtns) do
						k.BackgroundTransparency=0.94 k.TextTransparency=0.52
					end
					c.BackgroundTransparency=0.82 c.TextTransparency=0.08
					-- Auto-populate sensible defaults
					local defaults={
						BROADCAST={HEIGHT=44,DISTANCE=82,FOV=66,TILT=8,FOLLOW_Z=18},
						FOLLOW    ={HEIGHT=10,DISTANCE=20,FOV=74,TILT=3,FOLLOW_Z=10},
						ORBIT     ={HEIGHT=14,DISTANCE=0, FOV=68,TILT=0,FOLLOW_Z=0},
						SIDELINE  ={HEIGHT=6, DISTANCE=18,FOV=76,TILT=4,FOLLOW_Z=8},
						AERIAL    ={HEIGHT=90,DISTANCE=0, FOV=52,TILT=55,FOLLOW_Z=25},
					}
					local d=defaults[tp]
					if d then
						for k,v in pairs(d) do
							SV[k]=v
							if SR[k] then SR[k](v) end
						end
					end
				end)
			end
			bY=bY+34
		end

		-- ── Position & Optics ──
		bY=bY+4
		bY=secHead(buildPage,bY,"Position & Optics")

		local function addS(label,key,mn,mx,dec)
			local ny,getter,setter=mkSlider(buildPage,bY,label,mn,mx,SV[key],dec,function(v) SV[key]=v end)
			bY=ny SR[key]=setter
		end

		addS("HEIGHT",         "HEIGHT",       1,  220, 0)
		addS("DISTANCE",       "DISTANCE",     0,  260, 0)
		addS("FIELD OF VIEW",  "FOV",          20, 120, 0)
		addS("TILT ANGLE",     "TILT",         -40, 40, 0)
		addS("FOLLOW Z OFFSET","FOLLOW_Z",     -80, 80, 0)
		addS("LEAD AHEAD",     "LEAD",         0,  50,  0)
		addS("SIDE OFFSET",    "SIDE_OFFSET",  -80, 80, 0)

		-- ── Height Clamp ──
		bY=bY+4
		bY=secHead(buildPage,bY,"Height Bounds")
		addS("MIN HEIGHT",  "MIN_HEIGHT",  0,  30, 0)
		addS("MAX HEIGHT",  "MAX_HEIGHT",  20, 250, 0)

		-- ── Motion ──
		bY=bY+4
		bY=secHead(buildPage,bY,"Motion & Smoothing")
		addS("SMOOTH FACTOR", "SMOOTH",       0.005, 0.60, 3)
		addS("ORBIT RADIUS",  "ORBIT_RADIUS", 5,     100,  0)
		addS("ORBIT SPEED",   "ORBIT_SPEED",  0.01,  3.0,  2)

		-- ── Shake ──
		bY=bY+4
		bY=secHead(buildPage,bY,"Camera Shake")
		addS("SHAKE INTENSITY","SHAKE_INTENSITY",0,2.0,2)
		addS("SHAKE FREQUENCY","SHAKE_FREQ",     0.01,0.5,2)

		-- ── Flags ──
		bY=bY+4
		bY=secHead(buildPage,bY,"Behaviour Flags")
		local _,getAutoAI    =mkToggle(buildPage,bY,"Include in AI Director",true,nil) bY=bY+44
		local _,getPredictive=mkToggle(buildPage,bY,"Predictive Ball Lead",true,nil)   bY=bY+44
		local _,getFollowVel =mkToggle(buildPage,bY,"Velocity-Based Lead",true,nil)    bY=bY+44
		local _,getShake     =mkToggle(buildPage,bY,"Enable Camera Shake",false,nil)   bY=bY+44
		local _,getDOF       =mkToggle(buildPage,bY,"Depth of Field Hint",false,nil)   bY=bY+44
		local _,getLockY     =mkToggle(buildPage,bY,"Lock Vertical Aim",false,nil)     bY=bY+44

		-- ── Focus target ──
		bY=bY+4
		bY=secHead(buildPage,bY,"Focus Target")
		local _,getFocusMode=mkDropdown(buildPage,bY,"MODE",{"BALL","BALL_CARRIER","NEAREST_PLAYER","CUSTOM_POINT"},1)
		bY=bY+44

		-- ── Transition ──
		bY=secHead(buildPage,bY,"Transition")
		local _,getTransMode=mkDropdown(buildPage,bY,"CUT STYLE",{"SMOOTH","HARD CUT","WIPE","FADE"},1)
		bY=bY+44
		addS("TRANSITION SPEED","SMOOTH",0.005,0.60,3)  -- reuses smooth

		-- ── Actions ──
		bY=bY+8
		local _,prevBtn=mkBtn(buildPage,bY,"▶  PREVIEW",false) bY=bY+42
		local _,saveBtn=mkBtn(buildPage,bY,"  SAVE CAMERA",true) bY=bY+42
		local ny2,rstBtn,dupBtn=mkBtnRow(buildPage,bY,"↺  RESET","⊕  DUPLICATE") bY=ny2+4

		buildPage.CanvasSize=UDim2.new(0,0,0,bY+20)

		-- ── Button logic ──
		prevBtn.MouseButton1Click:Connect(function()
			if not Cam.ball or not Cam.ball.Parent then
				notifyErr("Editor","No ball in workspace") return
			end
			local bp=Cam.ball.Position
			local tp=Vector3.new(SV.DISTANCE+Cam.dynDist,SV.HEIGHT+Cam.dynHeight,bp.Z+SV.FOLLOW_Z)
			Cam.tgtCF=applyTilt(CFrame.new(tp,bp+Vector3.new(0,3,0)),SV.TILT)
			Cam.tgtFOV=SV.FOV
			Cam.manual=true
			notify("Editor","Preview active — press A to re-enable AI")
		end)

		rstBtn.MouseButton1Click:Connect(function()
			local def={HEIGHT=44,DISTANCE=82,FOV=66,SMOOTH=0.055,TILT=8,FOLLOW_Z=18,
				LEAD=12,SIDE_OFFSET=0,ORBIT_RADIUS=20,ORBIT_SPEED=0.30,
				MIN_HEIGHT=3,MAX_HEIGHT=160,SHAKE_INTENSITY=0.4,SHAKE_FREQ=0.08}
			for k,v in pairs(def) do
				SV[k]=v if SR[k] then SR[k](v) end
			end
			notify("Editor","Defaults restored")
		end)

		dupBtn.MouseButton1Click:Connect(function()
			local base=nameBox.Text:gsub("%s+","_"):upper()
			if base=="" then notifyErr("Editor","Name the camera first") return end
			nameBox.Text=base.."_COPY"
			notify("Editor","Duplicated — edit name then save")
		end)

		saveBtn.MouseButton1Click:Connect(function()
			local rawN=nameBox.Text
			local camName=rawN:upper():gsub("%s+","_"):gsub("[^%w_]","")
			if camName=="" then notifyErr("Editor","Enter a name first") return end

			local cd={
				name=camName,
				HEIGHT=SV.HEIGHT, DISTANCE=SV.DISTANCE, FOV=SV.FOV,
				SMOOTH=SV.SMOOTH, TILT=SV.TILT, FOLLOW_Z=SV.FOLLOW_Z,
				LEAD=SV.LEAD, SIDE_OFFSET=SV.SIDE_OFFSET,
				ORBIT_RADIUS=SV.ORBIT_RADIUS, ORBIT_SPEED=SV.ORBIT_SPEED,
				MIN_HEIGHT=SV.MIN_HEIGHT, MAX_HEIGHT=SV.MAX_HEIGHT,
				SHAKE_INTENSITY=SV.SHAKE_INTENSITY, SHAKE_FREQ=SV.SHAKE_FREQ,
				AUTO_AI=getAutoAI(), PREDICTIVE=getPredictive(),
				FOLLOW_VEL=getFollowVel(), SHAKE=getShake(),
				DOF=getDOF(), LOCK_Y=getLockY(),
				FOCUS_MODE=getFocusMode(), TRANSITION=getTransMode(),
			}

			CFG.CAM[camName]=cd
			EditorSys.customCams[camName]=cd

			CAM_FN[camName]=function()
				safeCall(function()
					if not Cam.ball or not Cam.ball.Parent then return end
					local bp=Cam.ball.Position
					local bv=BallTracker:GetSmoothedVelocity(3)
					local leadZ=0
					if bv.Magnitude>5 and cd.FOLLOW_VEL then
						leadZ=bv.Z*(cd.LEAD or 0)/math.max(bv.Magnitude,1)*0.5
					end
					local h=clamp((cd.HEIGHT or 44)+Cam.dynHeight, cd.MIN_HEIGHT or 1, cd.MAX_HEIGHT or 220)
					local tp=Vector3.new(
						(cd.DISTANCE or 60)+(cd.SIDE_OFFSET or 0)+Cam.dynDist,
						h,
						bp.Z+(cd.FOLLOW_Z or 18)+leadZ
					)
					local look=bp+Vector3.new(0,3,0)
					if cd.PREDICTIVE and BallTracker.catchTarget then
						local ch=getHRP(BallTracker.catchTarget)
						if ch then look=bp:Lerp(ch.Position+Vector3.new(0,2,0),0.3) end
					end
					if cd.LOCK_Y then look=Vector3.new(look.X,tp.Y,look.Z) end
					Cam.tgtCF=applyTilt(CFrame.new(tp,look),cd.TILT or 0)
					Cam.tgtFOV=(cd.FOV or 66)+Cam.dynFOV
				end,"CAM_CUSTOM_"..camName)
			end

			if cd.AUTO_AI then
				local found=false
				for _,s in ipairs(SHOTS) do if s==camName then found=true break end end
				if not found then table.insert(SHOTS,camName) end
			end

			notify("Editor","Saved — "..camName)
			TweenService:Create(saveBtn,TweenInfo.new(0.1),{BackgroundTransparency=0.25}):Play()
			task.delay(0.35,function()
				TweenService:Create(saveBtn,TweenInfo.new(0.22),{BackgroundTransparency=0}):Play()
			end)
		end)

		-- ─────────────────────────────────────────────────────────────────
		-- ════ TAB 2: PRESETS ══════════════════════════════════════════════
		-- ─────────────────────────────────────────────────────────────────
		local presetsPage=makePage()
		tabPages["PRESETS"]=presetsPage

		local presets={
			{name="TV BROADCAST",  tag="CLASSIC",
			 desc="Elevated sideline, standard NFL television framing",
			 vals={HEIGHT=44,DISTANCE=82,FOV=66,SMOOTH=0.055,TILT=8,FOLLOW_Z=18,LEAD=12,SIDE_OFFSET=0}},
			{name="TIGHT SIDELINE",tag="DRAMA",
			 desc="Low sideline, pads-level tension shot",
			 vals={HEIGHT=5,DISTANCE=22,FOV=76,SMOOTH=0.14,TILT=3,FOLLOW_Z=9,LEAD=15,SIDE_OFFSET=0}},
			{name="HIGH AERIAL",   tag="TACTICAL",
			 desc="All-22 bird's-eye formation read",
			 vals={HEIGHT=96,DISTANCE=120,FOV=54,SMOOTH=0.036,TILT=15,FOLLOW_Z=32,LEAD=8,SIDE_OFFSET=0}},
			{name="SKYCAM",        tag="CABLE",
			 desc="Overhead cable-cam, center field",
			 vals={HEIGHT=60,DISTANCE=42,FOV=50,SMOOTH=0.055,TILT=48,FOLLOW_Z=0,LEAD=0,SIDE_OFFSET=0}},
			{name="POCKET VIEW",   tag="QB",
			 desc="Behind QB at shoulder height, pressure visible",
			 vals={HEIGHT=8,DISTANCE=11,FOV=78,SMOOTH=0.15,TILT=4,FOLLOW_Z=5,LEAD=18,SIDE_OFFSET=3}},
			{name="TIGHT FOLLOW",  tag="CARRIER",
			 desc="Intimate over-shoulder of ball carrier",
			 vals={HEIGHT=9,DISTANCE=17,FOV=75,SMOOTH=0.20,TILT=5,FOLLOW_Z=11,LEAD=17,SIDE_OFFSET=2}},
			{name="END ZONE",      tag="REDZONE",
			 desc="Looking into the end zone from behind",
			 vals={HEIGHT=22,DISTANCE=46,FOV=64,SMOOTH=0.070,TILT=7,FOLLOW_Z=-30,LEAD=5,SIDE_OFFSET=0}},
			{name="PYLON CAM",     tag="ULTRA LOW",
			 desc="Ground level, pylon perspective",
			 vals={HEIGHT=1,DISTANCE=7,FOV=85,SMOOTH=0.22,TILT=1,FOLLOW_Z=0,LEAD=4,SIDE_OFFSET=5}},
			{name="CINEMATIC WIDE",tag="FILM",
			 desc="Anamorphic-style slow wide dolly",
			 vals={HEIGHT=14,DISTANCE=28,FOV=56,SMOOTH=0.055,TILT=5,FOLLOW_Z=0,LEAD=0,SIDE_OFFSET=0}},
			{name="BLITZ CAM",     tag="PRESSURE",
			 desc="Edge rush angle, pressure visibility",
			 vals={HEIGHT=7,DISTANCE=13,FOV=82,SMOOTH=0.18,TILT=3,FOLLOW_Z=6,LEAD=12,SIDE_OFFSET=2}},
			{name="STADIUM SWEEP", tag="AMBIENT",
			 desc="Wide sweeping stadium perspective",
			 vals={HEIGHT=55,DISTANCE=100,FOV=60,SMOOTH=0.038,TILT=12,FOLLOW_Z=20,LEAD=6,SIDE_OFFSET=0}},
			{name="LINE OF SCRIMMAGE",tag="TRENCHES",
			 desc="Right at the line, O-line vs D-line",
			 vals={HEIGHT=4,DISTANCE=5,FOV=80,SMOOTH=0.19,TILT=2,FOLLOW_Z=2,LEAD=8,SIDE_OFFSET=0}},
		}

		local pY=14
		pY=secHead(presetsPage,pY,"Built-In Presets  ·  Click to load into Build tab")

		for _,p in ipairs(presets) do
			local card=Instance.new("TextButton")
			card.Size=UDim2.new(1,-24,0,56)
			card.Position=UDim2.new(0,12,0,pY)
			card.BackgroundColor3=Color3.fromRGB(255,255,255)
			card.BackgroundTransparency=0.965
			card.BorderSizePixel=0
			card.Text=""
			card.ZIndex=3
			card.Parent=presetsPage
			Instance.new("UICorner",card).CornerRadius=UDim.new(0,2)
			local cs=Instance.new("UIStroke",card)
			cs.Color=Color3.fromRGB(255,255,255)
			cs.Transparency=0.90

			local tagLbl=Instance.new("TextLabel")
			tagLbl.Size=UDim2.new(0,72,0,15)
			tagLbl.Position=UDim2.new(0,10,0,8)
			tagLbl.BackgroundColor3=Color3.fromRGB(255,255,255)
			tagLbl.BackgroundTransparency=0.88
			tagLbl.Text=p.tag
			tagLbl.TextColor3=Color3.fromRGB(255,255,255)
			tagLbl.TextTransparency=0.25
			tagLbl.TextSize=6.5
			tagLbl.Font=Enum.Font.GothamBold
			tagLbl.ZIndex=4
			tagLbl.Parent=card
			Instance.new("UICorner",tagLbl).CornerRadius=UDim.new(0,2)

			local nameLbl=Instance.new("TextLabel")
			nameLbl.Size=UDim2.new(1,-130,0,16)
			nameLbl.Position=UDim2.new(0,88,0,8)
			nameLbl.BackgroundTransparency=1
			nameLbl.Text=p.name
			nameLbl.TextColor3=Color3.fromRGB(255,255,255)
			nameLbl.TextTransparency=0.06
			nameLbl.TextSize=10
			nameLbl.Font=Enum.Font.GothamBold
			nameLbl.TextXAlignment=Enum.TextXAlignment.Left
			nameLbl.ZIndex=4
			nameLbl.Parent=card

			local descLbl=Instance.new("TextLabel")
			descLbl.Size=UDim2.new(1,-130,0,13)
			descLbl.Position=UDim2.new(0,88,0,28)
			descLbl.BackgroundTransparency=1
			descLbl.Text=p.desc
			descLbl.TextColor3=Color3.fromRGB(255,255,255)
			descLbl.TextTransparency=0.58
			descLbl.TextSize=8.5
			descLbl.Font=Enum.Font.Gotham
			descLbl.TextXAlignment=Enum.TextXAlignment.Left
			descLbl.ZIndex=4
			descLbl.Parent=card

			-- Mini param strip
			local params=string.format("H:%d  D:%d  F°:%d  SM:%.3f",
				p.vals.HEIGHT or 0,p.vals.DISTANCE or 0,
				p.vals.FOV or 0,p.vals.SMOOTH or 0)
			local paramLbl=Instance.new("TextLabel")
			paramLbl.Size=UDim2.new(1,-130,0,10)
			paramLbl.Position=UDim2.new(0,88,0,43)
			paramLbl.BackgroundTransparency=1
			paramLbl.Text=params
			paramLbl.TextColor3=Color3.fromRGB(255,255,255)
			paramLbl.TextTransparency=0.72
			paramLbl.TextSize=7.5
			paramLbl.Font=Enum.Font.Gotham
			paramLbl.TextXAlignment=Enum.TextXAlignment.Left
			paramLbl.ZIndex=4
			paramLbl.Parent=card

			local arrow=Instance.new("TextLabel")
			arrow.Size=UDim2.new(0,32,1,0)
			arrow.Position=UDim2.new(1,-40,0,0)
			arrow.BackgroundTransparency=1
			arrow.Text="→"
			arrow.TextColor3=Color3.fromRGB(255,255,255)
			arrow.TextTransparency=0.6
			arrow.TextSize=14
			arrow.Font=Enum.Font.GothamBold
			arrow.ZIndex=4
			arrow.Parent=card

			card.MouseEnter:Connect(function()
				TweenService:Create(card,TweenInfo.new(0.1),{BackgroundTransparency=0.87}):Play()
				TweenService:Create(cs,TweenInfo.new(0.1),{Transparency=0.72}):Play()
			end)
			card.MouseLeave:Connect(function()
				TweenService:Create(card,TweenInfo.new(0.14),{BackgroundTransparency=0.965}):Play()
				TweenService:Create(cs,TweenInfo.new(0.14),{Transparency=0.90}):Play()
			end)
			card.MouseButton1Click:Connect(function()
				for k,v in pairs(p.vals) do
					SV[k]=v if SR[k] then SR[k](v) end
				end
				if nameBox then nameBox.Text=p.name:gsub("%s+","_") end
				-- Switch to BUILD
				for _,pg in pairs(tabPages) do pg.Visible=false end
				for _,tb in pairs(tabBtns) do tb.BackgroundTransparency=0.94 tb.TextTransparency=0.52 end
				tabPages["BUILD"].Visible=true
				if tabBtns["BUILD"] then tabBtns["BUILD"].BackgroundTransparency=0.80 tabBtns["BUILD"].TextTransparency=0.08 end
				activeTab="BUILD"
				notify("Editor","Loaded: "..p.name)
			end)

			pY=pY+64
		end
		presetsPage.CanvasSize=UDim2.new(0,0,0,pY+20)

		-- ─────────────────────────────────────────────────────────────────
		-- ════ TAB 3: LIBRARY ══════════════════════════════════════════════
		-- ─────────────────────────────────────────────────────────────────
		local libraryPage=makePage()
		tabPages["LIBRARY"]=libraryPage

		local function rebuildLibrary()
			-- Clear old content
			for _,c in ipairs(libraryPage:GetChildren()) do
				if not c:IsA("UIScrollingFrameConstraint") then c:Destroy() end
			end
			local lY=14
			lY=secHead(libraryPage,lY,"Saved Custom Cameras")

			local count=0
			for cname,cdata in pairs(EditorSys.customCams) do
				count=count+1
				local card=Instance.new("Frame")
				card.Size=UDim2.new(1,-24,0,64)
				card.Position=UDim2.new(0,12,0,lY)
				card.BackgroundColor3=Color3.fromRGB(255,255,255)
				card.BackgroundTransparency=0.965
				card.BorderSizePixel=0
				card.ZIndex=3
				card.Parent=libraryPage
				Instance.new("UICorner",card).CornerRadius=UDim.new(0,2)
				Instance.new("UIStroke",card).Color=Color3.fromRGB(55,55,55)

				local cNLbl=Instance.new("TextLabel")
				cNLbl.Size=UDim2.new(0.65,0,0,16)
				cNLbl.Position=UDim2.new(0,12,0,8)
				cNLbl.BackgroundTransparency=1
				cNLbl.Text=cname
				cNLbl.TextColor3=Color3.fromRGB(255,255,255)
				cNLbl.TextTransparency=0.06
				cNLbl.TextSize=10
				cNLbl.Font=Enum.Font.GothamBold
				cNLbl.TextXAlignment=Enum.TextXAlignment.Left
				cNLbl.ZIndex=4
				cNLbl.Parent=card

				local cStats=Instance.new("TextLabel")
				cStats.Size=UDim2.new(1,-24,0,12)
				cStats.Position=UDim2.new(0,12,0,28)
				cStats.BackgroundTransparency=1
				cStats.Text=string.format("H:%.0f  D:%.0f  FOV:%.0f  TILT:%.0f  SM:%.3f",
					cdata.HEIGHT or 0,cdata.DISTANCE or 0,
					cdata.FOV or 66,cdata.TILT or 0,cdata.SMOOTH or 0.055)
				cStats.TextColor3=Color3.fromRGB(255,255,255)
				cStats.TextTransparency=0.60
				cStats.TextSize=8
				cStats.Font=Enum.Font.Gotham
				cStats.TextXAlignment=Enum.TextXAlignment.Left
				cStats.ZIndex=4
				cStats.Parent=card

				local cFlags=Instance.new("TextLabel")
				cFlags.Size=UDim2.new(1,-24,0,11)
				cFlags.Position=UDim2.new(0,12,0,43)
				cFlags.BackgroundTransparency=1
				local flagStr=""
				if cdata.AUTO_AI then flagStr=flagStr.."AI  " end
				if cdata.PREDICTIVE then flagStr=flagStr.."PRED  " end
				if cdata.SHAKE then flagStr=flagStr.."SHAKE  " end
				if cdata.DOF then flagStr=flagStr.."DOF  " end
				cFlags.Text=flagStr~="" and ("FLAGS: "..flagStr) or "NO FLAGS"
				cFlags.TextColor3=Color3.fromRGB(255,255,255)
				cFlags.TextTransparency=0.72
				cFlags.TextSize=7.5
				cFlags.Font=Enum.Font.Gotham
				cFlags.TextXAlignment=Enum.TextXAlignment.Left
				cFlags.ZIndex=4
				cFlags.Parent=card

				-- Action buttons
				local bW=56
				local function libBtn(xFromRight,label)
					local b=Instance.new("TextButton")
					b.Size=UDim2.new(0,bW,0,20)
					b.Position=UDim2.new(1,-xFromRight,1,-28)
					b.BackgroundColor3=Color3.fromRGB(255,255,255)
					b.BackgroundTransparency=0.88
					b.BorderSizePixel=0
					b.Text=label
					b.TextColor3=Color3.fromRGB(255,255,255)
					b.TextTransparency=0.15
					b.TextSize=7.5
					b.Font=Enum.Font.GothamBold
					b.ZIndex=5
					b.Parent=card
					Instance.new("UICorner",b).CornerRadius=UDim.new(0,2)
					b.MouseEnter:Connect(function()
						TweenService:Create(b,TweenInfo.new(0.1),{BackgroundTransparency=0.65}):Play()
					end)
					b.MouseLeave:Connect(function()
						TweenService:Create(b,TweenInfo.new(0.12),{BackgroundTransparency=0.88}):Play()
					end)
					return b
				end

				local actBtn=libBtn(180,"ACTIVATE")
				actBtn.MouseButton1Click:Connect(function()
					Cam:setMode(cname)
					notify("Camera","→ "..cname)
				end)

				local editBtn=libBtn(118,"EDIT")
				editBtn.MouseButton1Click:Connect(function()
					if nameBox then nameBox.Text=cname end
					for k,v in pairs(cdata) do
						if SV[k]~=nil then SV[k]=v if SR[k] then SR[k](v) end end
					end
					for _,pg in pairs(tabPages) do pg.Visible=false end
					for _,tb in pairs(tabBtns) do tb.BackgroundTransparency=0.94 tb.TextTransparency=0.52 end
					tabPages["BUILD"].Visible=true
					if tabBtns["BUILD"] then tabBtns["BUILD"].BackgroundTransparency=0.80 tabBtns["BUILD"].TextTransparency=0.08 end
					activeTab="BUILD"
					notify("Editor","Editing: "..cname)
				end)

				local delBtn=libBtn(56,"DELETE")
				delBtn.MouseButton1Click:Connect(function()
					EditorSys.customCams[cname]=nil
					CFG.CAM[cname]=nil
					CAM_FN[cname]=nil
					for i,s in ipairs(SHOTS) do if s==cname then table.remove(SHOTS,i) break end end
					TweenService:Create(card,TweenInfo.new(0.16),{BackgroundTransparency=1}):Play()
					task.delay(0.18,function() if card.Parent then card:Destroy() end end)
					notify("Editor","Deleted: "..cname)
				end)

				lY=lY+72
			end

			if count==0 then
				local empty=Instance.new("TextLabel")
				empty.Size=UDim2.new(1,-24,0,50)
				empty.Position=UDim2.new(0,12,0,lY)
				empty.BackgroundTransparency=1
				empty.Text="No saved cameras.\nCreate one in the BUILD tab."
				empty.TextColor3=Color3.fromRGB(255,255,255)
				empty.TextTransparency=0.65
				empty.TextSize=9
				empty.Font=Enum.Font.Gotham
				empty.TextWrapped=true
				empty.ZIndex=3
				empty.Parent=libraryPage
				lY=lY+56
			end
			libraryPage.CanvasSize=UDim2.new(0,0,0,lY+20)
		end
		rebuildLibrary()

		-- ─────────────────────────────────────────────────────────────────
		-- ════ TAB 4: AI TUNING ════════════════════════════════════════════
		-- ─────────────────────────────────────────────────────────────────
		local tuningPage=makePage()
		tabPages["TUNING"]=tuningPage

		local tY=14
		tY=secHead(tuningPage,tY,"AI Director")

		local function tSlider(label,key,mn,mx,dec,target,targetKey)
			local ny,_,setter=mkSlider(tuningPage,tY,label,mn,mx,target[targetKey],dec,function(v) target[targetKey]=v end)
			tY=ny
			return setter
		end

		local setDecInt  =tSlider("DECISION INTERVAL (s)",     "DI",   0.03,1.0,  3, CFG.AI,"DECISION_INTERVAL")
		local setMinShot =tSlider("MIN SHOT DURATION (s)",      "MS",   0.5, 8.0,  1, CFG.AI,"MIN_SHOT_DURATION")
		local setMaxShot =tSlider("MAX SHOT DURATION (s)",      "XS",   3.0, 30.0, 1, CFG.AI,"MAX_SHOT_DURATION")
		local setConf    =tSlider("CONFIDENCE THRESHOLD",       "CT",   0.2, 0.95, 2, CFG.AI,"CONFIDENCE_THRESHOLD")
		local setHyst    =tSlider("HYSTERESIS (stay bias)",     "HY",   1.0, 2.5,  2, CFG.AI,"HYSTERESIS")
		local setUrgency =tSlider("URGENCY OVERRIDE",           "UO",   0.5, 1.0,  2, CFG.AI,"URGENCY_OVERRIDE")
		local setMoment  =tSlider("MOMENTUM WEIGHT",            "MW",   0,   0.5,  2, CFG.AI,"MOMENTUM_WEIGHT")
		local setVariety =tSlider("VARIETY BONUS",              "VB",   0,   0.3,  2, CFG.AI,"VARIETY_BONUS")
		local setTransSm =tSlider("TRANSITION SMOOTH",          "TS",   0.02,0.5,  2, CFG.AI,"TRANSITION_SMOOTH")

		tY=tY+4
		tY=secHead(tuningPage,tY,"Ball Tracking")
		local setBallH=tSlider("BALL SMOOTH — IN FLIGHT",  "BH",0.05,1.0,2,BALL_SMOOTH_MAP,"HIGH")
		local setBallN=tSlider("BALL SMOOTH — NORMAL",     "BN",0.05,0.9,2,BALL_SMOOTH_MAP,"NORMAL")
		local setCatch=tSlider("CATCH PREDICT TIME (s)",   "CP",0.1, 1.5,2,CFG.BALL,"CATCH_PREDICT_TIME")
		local setPredW=tSlider("PREDICTION WEIGHT",        "PW",0.1, 1.0,2,CFG.BALL,"PREDICTION_WEIGHT")

		tY=tY+4
		tY=secHead(tuningPage,tY,"Shader — Live Adjust")
		local setBloom  =tSlider("BLOOM INTENSITY",  "BI",0,   1.5, 2, CFG.SHADER,"BLOOM_INTENSITY")
		local setBloomSz=tSlider("BLOOM SIZE",        "BS",5,   40,  0, CFG.SHADER,"BLOOM_SIZE")
		local setSat    =tSlider("SATURATION",        "SA",0,   1.0, 2, CFG.SHADER,"SATURATION")
		local setBright =tSlider("BRIGHTNESS",        "BR",-0.2,0.4, 2, CFG.SHADER,"BRIGHTNESS")
		local setContrst=tSlider("CONTRAST",          "CO",0,   0.6, 2, CFG.SHADER,"CONTRAST")
		local setSunRay =tSlider("SUNRAY INTENSITY",  "SR",0,   0.5, 2, CFG.SHADER,"SUNRAY_INTENSITY")
		local setDOFR   =tSlider("DOF RADIUS",        "DR",0,   80,  0, CFG.SHADER,"DOF_RADIUS")

		-- Live shader apply
		tuningPage.ChildAdded:Connect(function()end)  -- keep page active
		local shaderConn
		shaderConn=RunService.Heartbeat:Connect(function()
			if not (sg and sg.Parent) then shaderConn:Disconnect() return end
			if ShaderSys.bloom then ShaderSys.bloom.Intensity=CFG.SHADER.BLOOM_INTENSITY end
			if ShaderSys.cc then
				ShaderSys.cc.Saturation=CFG.SHADER.SATURATION
				ShaderSys.cc.Brightness=CFG.SHADER.BRIGHTNESS
				ShaderSys.cc.Contrast=CFG.SHADER.CONTRAST
			end
			if ShaderSys.sunRay then ShaderSys.sunRay.Intensity=CFG.SHADER.SUNRAY_INTENSITY end
			if ShaderSys.dof then ShaderSys.dof.FarIntensity=CFG.SHADER.DOF_RADIUS/80 end
		end)

		tY=tY+8
		local _,rstAIBtn=mkBtn(tuningPage,tY,"↺  RESET AI & SHADER DEFAULTS",false) tY=tY+42
		rstAIBtn.MouseButton1Click:Connect(function()
			CFG.AI.DECISION_INTERVAL=0.10    setDecInt(0.10)
			CFG.AI.MIN_SHOT_DURATION=1.6     setMinShot(1.6)
			CFG.AI.MAX_SHOT_DURATION=12.0    setMaxShot(12.0)
			CFG.AI.CONFIDENCE_THRESHOLD=0.52  setConf(0.52)
			CFG.AI.HYSTERESIS=1.35           setHyst(1.35)
			CFG.AI.URGENCY_OVERRIDE=0.82     setUrgency(0.82)
			CFG.AI.MOMENTUM_WEIGHT=0.15      setMoment(0.15)
			CFG.AI.VARIETY_BONUS=0.08        setVariety(0.08)
			CFG.AI.TRANSITION_SMOOTH=0.12    setTransSm(0.12)
			CFG.SHADER.BLOOM_INTENSITY=0.32  setBloom(0.32)
			CFG.SHADER.BLOOM_SIZE=20         setBloomSz(20)
			CFG.SHADER.SATURATION=0.20       setSat(0.20)
			CFG.SHADER.BRIGHTNESS=0.04       setBright(0.04)
			CFG.SHADER.CONTRAST=0.15         setContrst(0.15)
			CFG.SHADER.SUNRAY_INTENSITY=0.10 setSunRay(0.10)
			CFG.SHADER.DOF_RADIUS=30         setDOFR(30)
			notify("AI Tuning","All defaults restored")
		end)
		tuningPage.CanvasSize=UDim2.new(0,0,0,tY+20)

		-- ─────────────────────────────────────────────────────────────────
		-- ════ TAB 5: LIVE ═════════════════════════════════════════════════
		-- Live read-only diagnostics dashboard
		-- ─────────────────────────────────────────────────────────────────
		local livePage=makePage()
		tabPages["LIVE"]=livePage

		local function mkStatRow(parent,y,label,key)
			local row=Instance.new("Frame")
			row.Size=UDim2.new(1,-24,0,28)
			row.Position=UDim2.new(0,12,0,y)
			row.BackgroundColor3=Color3.fromRGB(255,255,255)
			row.BackgroundTransparency=0.97
			row.BorderSizePixel=0
			row.ZIndex=3
			row.Parent=parent
			Instance.new("UICorner",row).CornerRadius=UDim.new(0,2)

			local kLbl=Instance.new("TextLabel")
			kLbl.Size=UDim2.new(0.52,0,1,0)
			kLbl.Position=UDim2.new(0,10,0,0)
			kLbl.BackgroundTransparency=1
			kLbl.Text=label
			kLbl.TextColor3=Color3.fromRGB(255,255,255)
			kLbl.TextTransparency=0.45
			kLbl.TextSize=8.5
			kLbl.Font=Enum.Font.GothamBold
			kLbl.TextXAlignment=Enum.TextXAlignment.Left
			kLbl.ZIndex=4
			kLbl.Parent=row

			local vLbl=Instance.new("TextLabel")
			vLbl.Size=UDim2.new(0.46,0,1,0)
			vLbl.Position=UDim2.new(0.52,0,0,0)
			vLbl.BackgroundTransparency=1
			vLbl.Text="—"
			vLbl.TextColor3=Color3.fromRGB(255,255,255)
			vLbl.TextTransparency=0.12
			vLbl.TextSize=8.5
			vLbl.Font=Enum.Font.GothamBold
			vLbl.TextXAlignment=Enum.TextXAlignment.Right
			vLbl.ZIndex=4
			vLbl.Parent=row
			return y+34, vLbl
		end

		local dY=14
		dY=secHead(livePage,dY,"Camera State")
		local _,vMode    =mkStatRow(livePage,dY,"ACTIVE SHOT")       dY=dY+34
		local _,vFOV     =mkStatRow(livePage,dY,"CURRENT FOV")       dY=dY+34
		local _,vSmooth  =mkStatRow(livePage,dY,"SMOOTH FACTOR")     dY=dY+34
		local _,vManual  =mkStatRow(livePage,dY,"MANUAL OVERRIDE")   dY=dY+34

		dY=dY+4
		dY=secHead(livePage,dY,"AI Director")
		local _,vAIShot  =mkStatRow(livePage,dY,"AI SHOT")           dY=dY+34
		local _,vAIPhase =mkStatRow(livePage,dY,"GAME PHASE")        dY=dY+34
		local _,vAIConf  =mkStatRow(livePage,dY,"LAST CONFIDENCE")   dY=dY+34

		dY=dY+4
		dY=secHead(livePage,dY,"Ball")
		local _,vBallPos =mkStatRow(livePage,dY,"POSITION")          dY=dY+34
		local _,vBallVel =mkStatRow(livePage,dY,"VELOCITY MAG")      dY=dY+34
		local _,vBallAir =mkStatRow(livePage,dY,"AIRBORNE")          dY=dY+34

		dY=dY+4
		dY=secHead(livePage,dY,"System")
		local _,vErrors  =mkStatRow(livePage,dY,"ERROR COUNT")       dY=dY+34
		local _,vCustom  =mkStatRow(livePage,dY,"CUSTOM CAMERAS")    dY=dY+34
		local _,vShots   =mkStatRow(livePage,dY,"AI SHOT POOL SIZE") dY=dY+34
		local _,vPlayer  =mkStatRow(livePage,dY,"PLAYER")            dY=dY+34

		livePage.CanvasSize=UDim2.new(0,0,0,dY+20)

		-- Live update loop
		task.spawn(function()
			while sg and sg.Parent do
				if livePage.Visible then
					pcall(function()
						vMode.Text    =tostring(Cam.mode or "—")
						vFOV.Text     =string.format("%.1f°",Cam.camera and Cam.camera.FieldOfView or 0)
						vSmooth.Text  =string.format("%.3f",CFG.CAM[Cam.mode] and (CFG.CAM[Cam.mode].SMOOTH or 0) or 0)
						vManual.Text  =Cam.manual and "YES" or "NO"
						vAIShot.Text  =tostring(AI.shot or "—")
						vAIPhase.Text =tostring(GameState.phase or "—")
						vAIConf.Text  =string.format("%.2f",AI.lastConf or 0)
						if Cam.ball and Cam.ball.Parent then
							local bp=Cam.ball.Position
							vBallPos.Text =string.format("%.0f, %.0f, %.0f",bp.X,bp.Y,bp.Z)
							local bv=BallTracker:GetSmoothedVelocity(3)
							vBallVel.Text =string.format("%.1f studs/s",bv.Magnitude)
							vBallAir.Text =BallTracker.airborne and "YES" or "NO"
						else
							vBallPos.Text="NO BALL" vBallVel.Text="—" vBallAir.Text="—"
						end
						vErrors.Text  =tostring(SysMon and SysMon.errorCount or 0)
						local cc=0 for _ in pairs(EditorSys.customCams) do cc=cc+1 end
						vCustom.Text  =tostring(cc)
						vShots.Text   =tostring(#SHOTS)
						vPlayer.Text  =LP.Name
					end)
				end
				task.wait(0.25)
			end
		end)

		-- ─────────────────────────────────────────────────────────────────
		-- SIDEBAR TABS
		-- ─────────────────────────────────────────────────────────────────
		local function switchTab(id)
			activeTab=id
			for _,pg in pairs(tabPages) do pg.Visible=false end
			for tid,tb in pairs(tabBtns) do
				local on=tid==id
				TweenService:Create(tb,TweenInfo.new(0.12),{BackgroundTransparency=on and 0.80 or 0.94}):Play()
			end
			if tabPages[id] then tabPages[id].Visible=true end
			if id=="LIBRARY" then rebuildLibrary() end
		end

		for i,td in ipairs(tabDefs) do
			local isActive=td.id==activeTab
			local tb=Instance.new("TextButton")
			tb.Size=UDim2.new(1,0,0,44)
			tb.Position=UDim2.new(0,0,0,8+(i-1)*48)
			tb.BackgroundColor3=Color3.fromRGB(255,255,255)
			tb.BackgroundTransparency=isActive and 0.80 or 0.94
			tb.BorderSizePixel=0
			tb.Text=""
			tb.AutoButtonColor=false
			tb.ZIndex=6
			tb.Parent=sidebar
			tabBtns[td.id]=tb

			-- Active indicator bar
			local indic=Instance.new("Frame")
			indic.Size=UDim2.new(0,2,0,22)
			indic.Position=UDim2.new(0,0,0.5,-11)
			indic.BackgroundColor3=Color3.fromRGB(255,255,255)
			indic.BackgroundTransparency=isActive and 0.25 or 1
			indic.BorderSizePixel=0
			indic.ZIndex=7
			indic.Parent=tb

			local icon=Instance.new("TextLabel")
			icon.Size=UDim2.new(1,0,0,16)
			icon.Position=UDim2.new(0,0,0,7)
			icon.BackgroundTransparency=1
			icon.Text=td.icon
			icon.TextColor3=Color3.fromRGB(255,255,255)
			icon.TextTransparency=isActive and 0.06 or 0.52
			icon.TextSize=12
			icon.Font=Enum.Font.GothamBold
			icon.ZIndex=7
			icon.Parent=tb

			local lbl=Instance.new("TextLabel")
			lbl.Size=UDim2.new(1,0,0,10)
			lbl.Position=UDim2.new(0,0,0,25)
			lbl.BackgroundTransparency=1
			lbl.Text=td.label
			lbl.TextColor3=Color3.fromRGB(255,255,255)
			lbl.TextTransparency=isActive and 0.15 or 0.60
			lbl.TextSize=6.5
			lbl.Font=Enum.Font.GothamBold
			lbl.ZIndex=7
			lbl.Parent=tb

			tb.MouseButton1Click:Connect(function()
				-- Update indicator
				for _,o in pairs(tabBtns) do
					local oIndic=o:FindFirstChildOfClass("Frame")
					if oIndic then oIndic.BackgroundTransparency=1 end
				end
				indic.BackgroundTransparency=0.25
				switchTab(td.id)
			end)
			tb.MouseEnter:Connect(function()
				if activeTab~=td.id then
					TweenService:Create(tb,TweenInfo.new(0.1),{BackgroundTransparency=0.88}):Play()
				end
			end)
			tb.MouseLeave:Connect(function()
				if activeTab~=td.id then
					TweenService:Create(tb,TweenInfo.new(0.12),{BackgroundTransparency=0.94}):Play()
				end
			end)
		end
		switchTab("BUILD")

		-- ─────────────────────────────────────────────────────────────────
		-- STATUS BAR
		-- ─────────────────────────────────────────────────────────────────
		local statusBar=Instance.new("Frame")
		statusBar.Size=UDim2.new(1,0,0,22)
		statusBar.Position=UDim2.new(0,0,1,-22)
		statusBar.BackgroundColor3=Color3.fromRGB(11,11,11)
		statusBar.BorderSizePixel=0
		statusBar.ZIndex=8
		statusBar.Parent=panel

		local sbTop=Instance.new("Frame")
		sbTop.Size=UDim2.new(1,0,0,1)
		sbTop.BackgroundColor3=Color3.fromRGB(255,255,255)
		sbTop.BackgroundTransparency=0.88
		sbTop.BorderSizePixel=0
		sbTop.ZIndex=9
		sbTop.Parent=statusBar

		local sbLeft=Instance.new("TextLabel")
		sbLeft.Size=UDim2.new(0.5,0,1,0)
		sbLeft.Position=UDim2.new(0,SW+10,0,0)
		sbLeft.BackgroundTransparency=1
		sbLeft.Text="UFBarstool v"..VERSION.."  ·  "..LP.Name.."  ·  UNLIMITED"
		sbLeft.TextColor3=Color3.fromRGB(255,255,255)
		sbLeft.TextTransparency=0.65
		sbLeft.TextSize=7.5
		sbLeft.Font=Enum.Font.Gotham
		sbLeft.TextXAlignment=Enum.TextXAlignment.Left
		sbLeft.ZIndex=9
		sbLeft.Parent=statusBar

		local sbRight=Instance.new("TextLabel")
		sbRight.Size=UDim2.new(0.45,0,1,0)
		sbRight.Position=UDim2.new(0.55,-8,0,0)
		sbRight.BackgroundTransparency=1
		sbRight.Text="AI: —  ·  PHASE: —"
		sbRight.TextColor3=Color3.fromRGB(255,255,255)
		sbRight.TextTransparency=0.65
		sbRight.TextSize=7.5
		sbRight.Font=Enum.Font.GothamBold
		sbRight.TextXAlignment=Enum.TextXAlignment.Right
		sbRight.ZIndex=9
		sbRight.Parent=statusBar

		task.spawn(function()
			while sg and sg.Parent do
				pcall(function()
					if sbRight and sbRight.Parent then
						sbRight.Text="AI: "..(AI.shot or "—").."  ·  "..tostring(GameState.phase or "—")
					end
				end)
				task.wait(0.5)
			end
		end)

		-- ─────────────────────────────────────────────────────────────────
		-- ENTRANCE ANIMATION
		-- ─────────────────────────────────────────────────────────────────
		backdrop.BackgroundTransparency=1
		panel.Position=UDim2.new(0.5,-W/2,0.5,-H/2+20)
		panel.BackgroundTransparency=1

		TweenService:Create(backdrop,TweenInfo.new(0.18),{BackgroundTransparency=0.55}):Play()
		TweenService:Create(panel,TweenInfo.new(0.24,Enum.EasingStyle.Quart,Enum.EasingDirection.Out),{
			Position=UDim2.new(0.5,-W/2,0.5,-H/2),
			BackgroundTransparency=0
		}):Play()

	end,"BUILD_CAMERA_EDITOR")
end

local function buildDroneHUD()
	safeCall(function()
		local pg=LP:WaitForChild("PlayerGui")
		local old=pg:FindFirstChild("UFB_DroneHUD") if old then old:Destroy() end
		local sg=Instance.new("ScreenGui")
		sg.Name="UFB_DroneHUD" sg.ResetOnSpawn=false
		sg.DisplayOrder=998 sg.IgnoreGuiInset=true sg.Parent=pg
		local frame=Instance.new("Frame")
		frame.Name="HUD" frame.Size=UDim2.new(0,220,0,80) frame.Position=UDim2.new(0,12,1,-92)
		frame.BackgroundColor3=Color3.fromRGB(10,10,10) frame.BackgroundTransparency=0.25
		frame.BorderSizePixel=0 frame.Parent=sg
		Instance.new("UICorner",frame).CornerRadius=UDim.new(0,4)
		Instance.new("UIStroke",frame).Color=Color3.fromRGB(50,50,50)
		local ab=Instance.new("Frame")
		ab.Size=UDim2.new(0,2,1,0) ab.BackgroundColor3=Color3.fromRGB(190,190,190)
		ab.BorderSizePixel=0 ab.Parent=frame
		local function mkRow(y,txt)
			local l=Instance.new("TextLabel")
			l.Size=UDim2.new(1,-10,0,12) l.Position=UDim2.new(0,10,0,y)
			l.BackgroundTransparency=1 l.Text=txt
			l.TextColor3=Color3.fromRGB(120,120,120) l.TextSize=9
			l.Font=Enum.Font.Gotham l.TextXAlignment=Enum.TextXAlignment.Left l.Parent=frame
		end
		mkRow(6,"DRONE CAM  ·  ACTIVE")
		mkRow(22,"WASD Move  ·  Q/E Up/Down  ·  Shift Sprint")
		mkRow(36,"F Lock Ball  ·  G Lock Player  ·  C Orbit")
		mkRow(50,"Z/X Zoom  ·  [D] Exit")
		mkRow(64,"Ctrl Slow")
		Drone.hudGui=sg
	end,"BUILD_DRONE_HUD")
end

local function destroyDroneHUD()
	if Drone.hudGui and Drone.hudGui.Parent then Drone.hudGui:Destroy() end
	Drone.hudGui=nil
end

local function droneEnter()
	if Drone.active then return end
	Drone.active=true
	Cam.camera.CameraType=Enum.CameraType.Scriptable
	local hrp=getHRP(LP)
	if hrp then Drone.pos=hrp.Position+Vector3.new(0,30,0) else Drone.pos=Vector3.new(0,40,0) end
	Drone.vel=Vector3.new() Drone.yaw=0 Drone.pitch=0 Drone.fov=CFG.DRONE.FOV
	Drone.lockMode=nil Drone.lockPlayer=nil Drone.orbitActive=false
	UIS.MouseBehavior=Enum.MouseBehavior.LockCenter
	UIS.MouseIconEnabled=false
	buildDroneHUD()
	notify("Drone","Active — mouse look, WASD fly")
end

local function droneExit()
	if not Drone.active then return end
	Drone.active=false
	UIS.MouseBehavior=Enum.MouseBehavior.Default
	UIS.MouseIconEnabled=true
	destroyDroneHUD()
	Drone.lockMode=nil
	if Cam.on then Cam.camera.CameraType=Enum.CameraType.Scriptable
	else Cam.camera.CameraType=Enum.CameraType.Custom end
	notify("Drone","Exited")
end

local function updateDrone(dt)
	if not Drone.active then return end
	safeCall(function()
		local delta=UIS:GetMouseDelta()
		Drone.yaw=Drone.yaw-delta.X*CFG.DRONE.ROTATION_SPEED*60*dt
		Drone.pitch=clamp(Drone.pitch-delta.Y*CFG.DRONE.ROTATION_SPEED*60*dt,-math.pi*0.47,math.pi*0.47)
		local rotCF=CFrame.Angles(0,Drone.yaw,0)*CFrame.Angles(Drone.pitch,0,0)
		local fwd=rotCF.LookVector local right=rotCF.RightVector local up=Vector3.new(0,1,0)
		local spd=CFG.DRONE.BASE_SPEED
		if UIS:IsKeyDown(Enum.KeyCode.LeftShift) then spd=spd*CFG.DRONE.SPRINT_MULT end
		if UIS:IsKeyDown(Enum.KeyCode.LeftControl) then spd=spd*CFG.DRONE.SLOW_MULT end
		local moveInput=Vector3.new()
		if UIS:IsKeyDown(Enum.KeyCode.W) then moveInput=moveInput+fwd end
		if UIS:IsKeyDown(Enum.KeyCode.S) then moveInput=moveInput-fwd end
		if UIS:IsKeyDown(Enum.KeyCode.A) then moveInput=moveInput-right end
		if UIS:IsKeyDown(Enum.KeyCode.D) then moveInput=moveInput+right end
		if UIS:IsKeyDown(Enum.KeyCode.E) then moveInput=moveInput+up end
		if UIS:IsKeyDown(Enum.KeyCode.Q) then moveInput=moveInput-up end
		if moveInput.Magnitude>0 then moveInput=moveInput.Unit*spd end
		Drone.vel=Drone.vel:Lerp(moveInput,1-CFG.DRONE.INERTIA)
		Drone.pos=Drone.pos+Drone.vel*dt
		Drone.pos=Vector3.new(Drone.pos.X,clamp(Drone.pos.Y,CFG.DRONE.MIN_HEIGHT,CFG.DRONE.MAX_HEIGHT),Drone.pos.Z)
		local finalCF
		if Drone.orbitActive and (Drone.lockMode=="BALL" or Drone.lockMode=="PLAYER") then
			local target
			if Drone.lockMode=="BALL" and Cam.ball and Cam.ball.Parent then target=Cam.ball.Position
			elseif Drone.lockMode=="PLAYER" and Drone.lockPlayer then
				local h=getHRP(Drone.lockPlayer) if h then target=h.Position end
			end
			if target then
				Drone.orbitAngle=Drone.orbitAngle+(CFG.DRONE.ORBIT_SPEED*dt)
				local op=target+Vector3.new(math.cos(Drone.orbitAngle)*CFG.DRONE.ORBIT_RADIUS,CFG.DRONE.ORBIT_HEIGHT,math.sin(Drone.orbitAngle)*CFG.DRONE.ORBIT_RADIUS)
				Drone.pos=Drone.pos:Lerp(op,CFG.DRONE.LOCK_SMOOTH*2)
				finalCF=CFrame.new(Drone.pos,target)
				Cam.camera.CFrame=finalCF Cam.camera.FieldOfView=Drone.fov return
			end
		end
		if Drone.lockMode=="BALL" and Cam.ball and Cam.ball.Parent then
			finalCF=CFrame.new(Drone.pos,Cam.ball.Position):Lerp(CFrame.new(Drone.pos)*rotCF,0.15)
		elseif Drone.lockMode=="PLAYER" and Drone.lockPlayer then
			local h=getHRP(Drone.lockPlayer)
			if h then finalCF=CFrame.new(Drone.pos,h.Position):Lerp(CFrame.new(Drone.pos)*rotCF,0.15)
			else finalCF=CFrame.new(Drone.pos)*rotCF end
		else
			finalCF=CFrame.new(Drone.pos)*rotCF
		end
		Cam.camera.CFrame=finalCF Cam.camera.FieldOfView=Drone.fov
	end,"DRONE_UPDATE")
end

local function checkHealth()
	local t=tick()
	if t-Health.lastCheck<CFG.SYS.HEALTH_INTERVAL then return end
	Health.lastCheck=t
	if t-Health.lastErrorTime>CFG.SYS.ERROR_COOLDOWN then Health.errors=0 end
	if Health.errors>=CFG.SYS.MAX_ERRORS then
		Health.ok=false
		notifyErr("System","Error threshold — restarting")
		if CFG.SYS.AUTO_RECOVERY then
			Health.errors=0
			if Cam.on then Cam:Stop() task.wait(1) Cam:Start() end
		end
	end
end

local function doCleanup()
	local t=tick()
	if t-Health.lastCleanup<CFG.SYS.MEMORY_INTERVAL then return end
	Health.lastCleanup=t
	safeCall(function()
		for uid in pairs(PlayerIntel.tracked) do
			if not Players:GetPlayerByUserId(uid) then
				PlayerIntel.tracked[uid]=nil PlayerIntel.roles[uid]=nil
			end
		end
		while #AI.history>120 do table.remove(AI.history,1) end
		while #Health.log>60 do table.remove(Health.log,1) end
		while #BallTracker.posHistory>CFG.BALL.HISTORY_SIZE do table.remove(BallTracker.posHistory,1) end
	end,"CLEANUP")
end

local function updateCam(dt)
	dt=dt or (1/60)
	if Drone.active then updateDrone(dt) return end
	if not Cam.on then return end
	safeCall(function()
		Cam.frame=Cam.frame+1

		if not findBall() and Cam.frame%120==0 then
			notifyErr("Ball","Searching — check CFG.BALL.NAMES")
		end

		BallTracker:Update(dt)
		updatePlayerIntel()
		detectGamePhase()
		analyzeField()
		analyzePlaySituation()

		local an=GameState.analysis
		if an.isFumble and AI.urgentOverride~="FUMBLE_CAM" then urgentCut("FUMBLE_CAM",4.5) end
		if an.isSack and AI.urgentOverride~="SACK_CAM" then urgentCut("SACK_CAM",3.8) end
		if an.isInterception and AI.urgentOverride~="INTERCEPTION" then urgentCut("INTERCEPTION",5.0) end
		if an.touchdown and AI.urgentOverride~="CELEBRATION_CAM" then urgentCut("CELEBRATION_CAM",6.0) end
		if an.isScramble and AI.urgentOverride~="QB_SCRAMBLE" then urgentCut("QB_SCRAMBLE",4.0) end

		if Settings.smartCuts and an.ballCatchImminent and an.catchETA<0.4 then
			if AI.urgentOverride~="CATCH_CAM" then urgentCut("CATCH_CAM",2.5) end
		end

		if not Cam.manual then makeAIDecision() end

		checkHealth()
		doCleanup()
		ShaderSys:Update()
		applyCameraShake(dt)

		local mode=Cam.manual and Cam.mode or AI.shot
		local fn=CAM_FN[mode]
		if fn then
			if mode=="POV_CAM" then fn(dt) else fn() end
		else
			camBROADCAST()
		end

		local camCFG=CFG.CAM[mode]
		local smooth=(camCFG and camCFG.SMOOTH) or 0.055

		if GameState.ballInAir then
			local boost=BALL_SMOOTH_MAP[Settings.ballResponsive] or 0.50
			local ballShots={
				BALL_CAM=true,BALL_TRACK=true,BALL_FLIGHT=true,BALL_SPIRAL=true,
				HIGH_BROADCAST=true,CATCH_CAM=true,
				KICKOFF_CAM=true,KICKOFF_WIDE=true,
				PUNT_CAM=true,PUNT_HANG=true,
				FIELD_GOAL_CAM=true,FIELD_GOAL_SIDE=true,FIELD_GOAL_POSTS=true,
			}
			if ballShots[mode] then smooth=math.max(smooth,boost) end
		end

		local dtFactor=math.min(dt*60,3)
		local lerpT=1-(1-clamp(smooth,0,0.9999))^dtFactor
		local fovLerpT=1-(1-0.08)^dtFactor

		Cam.curCF=lerpCF(Cam.curCF,Cam.tgtCF,lerpT)
		Cam.curFOV=lerp(Cam.curFOV,Cam.tgtFOV,fovLerpT)
		Cam.camera.CFrame=Cam.curCF*Cam.shakeOffset
		Cam.camera.FieldOfView=clamp(Cam.curFOV,30,120)

		updateShotHud(mode,Cam.manual)
	end,"CAM_UPDATE")
end

function Cam:Start()
	safeCall(function()
		if self.on then self:Stop() task.wait(0.1) end
		self.on=true
		self.camera.CameraType=Enum.CameraType.Scriptable
		findBall()
		AI.shotStart=tick()
		self.conns.rs=RunService.RenderStepped:Connect(updateCam)
		notify("Camera v"..VERSION,"System active — AI Director running")
	end,"CAM_START")
end

function Cam:Stop()
	safeCall(function()
		if not self.on then return end
		self.on=false
		for _,c in pairs(self.conns) do c:Disconnect() end
		self.conns={}
		if not Drone.active then self.camera.CameraType=Enum.CameraType.Custom end
		notify("Camera","Stopped")
	end,"CAM_STOP")
end

function restartSystem()
	notify("System","Restarting...")
	safeCall(function()
		if Cam.on then Cam:Stop() end
		if Drone.active then droneExit() end
		if ShaderSys.on then ShaderSys:Disable() end
		AI.shot="BROADCAST" AI.prevShot="BROADCAST" AI.shotStart=0 AI.confidence=1.0
		AI.history={} AI.lastDecision=0 AI.scores={} AI.urgentOverride=nil AI.urgentUntil=0
		AI.recentShots={}
		PlayerIntel.tracked={} PlayerIntel.roles={} PlayerIntel.lastUpdate=0
		PlayerIntel.qbPads={} PlayerIntel.qbPadLastScan=0
		GameState.phase="PRE_SNAP" GameState.playType="UNKNOWN"
		GameState.ballCarrier=nil GameState.qb=nil GameState.ballInAir=false
		GameState.ballKicked=false GameState.ballHeight=0 GameState.ballSpeed=0
		GameState.lastPhaseChange=0 GameState.kickTime=0 GameState.phaseTime=0
		BallTracker.posHistory={} BallTracker.velHistory={} BallTracker.accelHistory={}
		BallTracker.flightPhase="NONE" BallTracker.launchDetected=false
		BallTracker.catchTarget=nil BallTracker.catchETA=math.huge
		PovSys.target=nil PovSys.playerList={} PovSys.playerIdx=1
		PovSys.bobPhase=0 PovSys.bankAngle=0
		Cam.ball=nil Cam.confirmedBall=nil
		Cam.lastBallScan=0 Cam.frame=0
		Cam.curCF=CFrame.new() Cam.tgtCF=CFrame.new() Cam.curFOV=70 Cam.tgtFOV=70
		Health.errors=0 Health.ok=true Health.lastCheck=0 Health.lastCleanup=0
		OverviewState.computedPos=Vector3.new(0,60,0) OverviewState.lastRecompute=0
	end,"RESTART_CLEANUP")
	task.wait(0.35)
	safeCall(function() if CFG.SHADER.ENABLED then ShaderSys:Enable() end end,"RESTART_SHADERS")
	task.wait(0.4)
	safeCall(function()
		Cam:Start()
		notify("System","Restart complete")
	end,"RESTART_START")
end

function Cam:Stats()
	local tracked=0 for _ in pairs(PlayerIntel.tracked) do tracked=tracked+1 end
	local an=GameState.analysis
	print("═══ UFBarstool Camera "..VERSION.." Stats ═══")
	print("Shot: "..AI.shot.." | Conf: "..string.format("%.2f",AI.confidence))
	print("Phase: "..GameState.phase.." | Play: "..GameState.playType)
	print("Ball: "..tostring(Cam.ball~=nil).." | Frame: "..Cam.frame)
	print("Players: "..tracked.." | ClusterR: "..string.format("%.1f",GameState.clusterRadius))
	print("Flight: "..BallTracker.flightPhase.." | AirTime: "..string.format("%.1f",an.ballAirTime))
	print("CatchETA: "..string.format("%.2f",BallTracker.catchETA).." | Spiral: "..string.format("%.2f",an.spiralQuality))
	print("Pressure: "..tostring(an.isPressure).." | Blitz: "..tostring(an.isBlitz))
	print("Scramble: "..tostring(an.isScramble).." | Breakaway: "..tostring(an.isBreakaway))
	print("Errors: "..Health.errors.." | Health: "..(Health.ok and "OK" or "DEGRADED"))
	print("License: "..LicenseData.tier.." | Custom Cams: "..tostring(#EditorSys.customCams or 0))
end

local function setupKeys()
	UIS.InputBegan:Connect(function(input,gp)
		if gp then return end
		local kc=input.KeyCode

		if kc==Enum.KeyCode.Backquote then restartSystem() return end
		if kc==Enum.KeyCode.RightBracket then toggleSettingsGui() return end
		if kc==Enum.KeyCode.LeftBracket then
			Settings.showShotHud=not Settings.showShotHud
			if ShotHud.frame then ShotHud.frame.Visible=Settings.showShotHud end
			notify("Shot HUD",Settings.showShotHud and "On" or "Off")
			return
		end
		if kc==Enum.KeyCode.BackSlash then
			if EditorSys.visible then destroyCameraEditor() else buildCameraEditor() end
			return
		end

		if kc==Enum.KeyCode.Z then
			if Drone.active then Drone.fov=math.max(CFG.DRONE.FOV_MIN,Drone.fov-CFG.DRONE.FOV_STEP) return end
			if Cam.on then Cam:Stop() else Cam:Start() end
		elseif kc==Enum.KeyCode.X then
			if Drone.active then Drone.fov=math.min(CFG.DRONE.FOV_MAX,Drone.fov+CFG.DRONE.FOV_STEP) return end
			ShaderSys:Toggle()
		elseif kc == Enum.KeyCode.N then
    		TimeSyncSys:Toggle()
		elseif kc==Enum.KeyCode.A then
			if Drone.active then return end
			if Cam.manual then Cam:setMode("AI") else Cam:setMode("BROADCAST") end
		elseif kc==Enum.KeyCode.S then
			if Drone.active then return end Cam:Stats()
		elseif kc==Enum.KeyCode.D then
			if Drone.active then droneExit() else droneEnter() end
		elseif kc==Enum.KeyCode.F then
			if not Drone.active then return end
			if Drone.lockMode=="BALL" then Drone.lockMode=nil notify("Drone","Lock off")
			else Drone.lockMode="BALL" notify("Drone","Locked → Ball") end
		elseif kc==Enum.KeyCode.G then
			if not Drone.active then return end
			if Drone.lockMode=="PLAYER" then Drone.lockMode=nil Drone.lockPlayer=nil notify("Drone","Lock off") return end
			local closestP,closestDist=nil,math.huge
			local camPos=Cam.camera.CFrame.Position
			for _,p in ipairs(Players:GetPlayers()) do
				if not isSpectator(p) then
					local h=getHRP(p)
					if h then local d=dist3(h.Position,camPos) if d<closestDist then closestDist=d closestP=p end end
				end
			end
			if closestP then Drone.lockMode="PLAYER" Drone.lockPlayer=closestP notify("Drone","Locked → "..closestP.Name) end
		elseif kc==Enum.KeyCode.C then
			if Drone.active then
				Drone.orbitActive=not Drone.orbitActive
				notify("Drone",Drone.orbitActive and "Orbit ON" or "Orbit OFF") return
			end
		elseif kc==Enum.KeyCode.B then if not Drone.active then Cam:setMode("BROADCAST") end
		elseif kc==Enum.KeyCode.H then if not Drone.active then Cam:setMode("HIGH_BROADCAST") end
		elseif kc==Enum.KeyCode.T then if not Drone.active then Cam:setMode("TIGHT_BROADCAST") end
		elseif kc==Enum.KeyCode.K then if not Drone.active then Cam:setMode("SKYCAM") end
		elseif kc==Enum.KeyCode.L then if not Drone.active then Cam:setMode("ALL_22") end
		elseif kc==Enum.KeyCode.Q then if not Drone.active then Cam:setMode("QB_CAM") end
		elseif kc==Enum.KeyCode.U then if not Drone.active then Cam:setMode("QB_CLOSEUP") end
		elseif kc==Enum.KeyCode.P then if not Drone.active then Cam:setMode("POCKET_CAM") end
		elseif kc==Enum.KeyCode.W then if not Drone.active then Cam:setMode("RECEIVER_CAM") end
		elseif kc==Enum.KeyCode.E then if not Drone.active then Cam:setMode("BALL_CAM") end
		elseif kc==Enum.KeyCode.R then if not Drone.active then Cam:setMode("BALL_TRACK") end
		elseif kc==Enum.KeyCode.Y then if not Drone.active then Cam:setMode("TACTICAL_CAM") end
		elseif kc==Enum.KeyCode.Six then if not Drone.active then Cam:setMode("SIDELINE_FOLLOW") end
		elseif kc==Enum.KeyCode.One then if not Drone.active then Cam:setMode("KICKOFF_CAM") end
		elseif kc==Enum.KeyCode.Two then if not Drone.active then Cam:setMode("PUNT_CAM") end
		elseif kc==Enum.KeyCode.Three then if not Drone.active then Cam:setMode("FIELD_GOAL_CAM") end
		elseif kc==Enum.KeyCode.Four then if not Drone.active then Cam:setMode("RETURN_CAM") end
		elseif kc==Enum.KeyCode.Five then if not Drone.active then Cam:setMode("CELEBRATION_CAM") end
		elseif kc==Enum.KeyCode.Seven then if not Drone.active then Cam:setMode("BALL_FLIGHT") end
		elseif kc==Enum.KeyCode.Eight then if not Drone.active then Cam:setMode("OVERVIEW_CAM") end
		elseif kc==Enum.KeyCode.Nine then if not Drone.active then Cam:setMode("POV_CAM") end
		elseif kc==Enum.KeyCode.Zero then if not Drone.active then Cam:setMode("CINEMATIC") end
		elseif kc==Enum.KeyCode.J then if not Drone.active then povCycleTarget() end
		end
	end)
end

-- =============================================
-- STARTUP — license check skipped, UNLIMITED granted
-- =============================================
LoadingScreen:Show()
LoadingScreen:SetProgress(0.1,"Initializing...")
task.wait(0.4)

validateLicense() -- Always sets UNLIMITED, no HTTP request needed
LicenseData.username=LP.Name

LoadingScreen:SetProgress(0.4,"Building UI...")
task.wait(0.3)
buildNotifGui()
buildShotHud()

LoadingScreen:SetProgress(0.6,"Setting up controls...")
task.wait(0.2)
setupKeys()



LoadingScreen:SetProgress(0.75,"Initializing shaders...")
task.wait(0.3)
if CFG.SHADER.ENABLED then ShaderSys:Enable() end

LoadingScreen:SetProgress(0.8,"Starting camera system...")
task.wait(0.4)
Cam:Start()

LoadingScreen:SetProgress(1.0,"Ready")
task.wait(0.5)
LoadingScreen:Hide()



notify("Welcome","UFBarstool Camera v"..VERSION.." — "..LicenseData.tier.." tier")

return Cam
