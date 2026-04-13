local ws = game:GetService("Workspace")
local rs = game:GetService("RunService")
local uis = game:GetService("UserInputService")
local pps = game:GetService("ProximityPromptService")
local plrs = game:GetService("Players")
local lighting = game:GetService("Lighting")
local cg = game:GetService("CoreGui")

local cam = ws.CurrentCamera
local lplr = plrs.LocalPlayer

local toggles = {
	AimAssist = false,
	ESP = false,
	Hitboxes = false,
	Fullbright = false,
	InstantInteract = false,
	DelCorpses = false
}

local cfg = {
	aimStrength = 0.1,        -- Changed default
	fov = 60,
	targetPart = "Head",
	maxDist = 500,
	maxEsp = 10,
	espFill = Color3.fromRGB(255, 0, 0),
	espOutline = Color3.fromRGB(255, 255, 255),
	fillTrans = 0.5,
	outTrans = 0,             -- Always visible outline
	scanRate = 0.5
}

local hitboxCfg = {
	enabled = true,
	part = "Head",
	size = Vector3.new(5, 5, 5),   -- Slightly bigger default for better hitreg
	show = true,
	color = BrickColor.new("Bright red"),
	trans = 0.6,
	refreshRate = 0.3              -- Faster refresh for more reliable hitboxes
}

-- GUI Setup (unchanged)
local gui = Instance.new("ScreenGui")
gui.Name = "OmniMenu"
gui.Parent = cg
gui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling

local main = Instance.new("Frame")
main.Name = "main"
main.Parent = gui
main.BackgroundColor3 = Color3.fromRGB(25, 25, 25)
main.BorderSizePixel = 0
main.Position = UDim2.new(0.05, 0, 0.2, 0)
main.Size = UDim2.new(0.2, 0, 0.6, 0)
main.Active = true
main.Draggable = true
main.ClipsDescendants = true

local sizeConstraint = Instance.new("UISizeConstraint")
sizeConstraint.Parent = main
sizeConstraint.MinSize = Vector2.new(150, 250)
sizeConstraint.MaxSize = Vector2.new(250, 600)

local title = Instance.new("TextLabel")
title.Parent = main
title.BackgroundColor3 = Color3.fromRGB(35, 35, 35)
title.Size = UDim2.new(1, 0, 0, 30)
title.Font = Enum.Font.GothamBold
title.Text = "FOOL"
title.TextColor3 = Color3.fromRGB(255, 255, 255)
title.TextSize = 14

local minBtn = Instance.new("TextButton")
minBtn.Parent = title
minBtn.BackgroundColor3 = Color3.fromRGB(150, 40, 40)
minBtn.Position = UDim2.new(1, -30, 0, 0)
minBtn.Size = UDim2.new(0, 30, 0, 30)
minBtn.Font = Enum.Font.GothamBold
minBtn.Text = "-"
minBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
minBtn.TextSize = 16
minBtn.BorderSizePixel = 0

local scroll = Instance.new("ScrollingFrame")
scroll.Name = "Scroll"
scroll.Parent = main
scroll.BackgroundTransparency = 1
scroll.BorderSizePixel = 0
scroll.Position = UDim2.new(0, 0, 0, 35)
scroll.Size = UDim2.new(1, 0, 1, -35)
scroll.ScrollBarThickness = 2
scroll.AutomaticCanvasSize = Enum.AutomaticSize.Y

local layout = Instance.new("UIListLayout")
layout.Parent = scroll
layout.Padding = UDim.new(0, 5)
layout.HorizontalAlignment = Enum.HorizontalAlignment.Center
layout.SortOrder = Enum.SortOrder.LayoutOrder

local isMin = false
local origSize = main.Size

minBtn.MouseButton1Click:Connect(function()
	isMin = not isMin
	if isMin then
		origSize = main.Size
		scroll.Visible = false
		title.BackgroundTransparency = 1
		title.TextTransparency = 1
		main.BackgroundTransparency = 1
		
		minBtn.Parent = main 
		minBtn.Position = UDim2.new(0, 0, 0, 0)
		main.Size = UDim2.new(0, 30, 0, 30)
		minBtn.Text = "+"
		minBtn.BackgroundColor3 = Color3.fromRGB(40, 150, 40)
	else
		main.Size = origSize
		scroll.Visible = true
		title.BackgroundTransparency = 0
		title.TextTransparency = 0
		main.BackgroundTransparency = 0
		
		minBtn.Parent = title
		minBtn.Position = UDim2.new(1, -30, 0, 0)
		minBtn.Text = "-"
		minBtn.BackgroundColor3 = Color3.fromRGB(150, 40, 40)
	end
end)

local function createToggle(name, key)
	local b = Instance.new("TextButton")
	b.Parent = scroll
	b.BackgroundColor3 = toggles[key] and Color3.fromRGB(40, 150, 40) or Color3.fromRGB(150, 40, 40)
	b.Size = UDim2.new(0.9, 0, 0, 35)
	b.Font = Enum.Font.Gotham
	b.Text = name .. (toggles[key] and ": ON" or ": OFF")
	b.TextColor3 = Color3.fromRGB(255, 255, 255)
	b.TextSize = 12
	b.BorderSizePixel = 0

	b.MouseButton1Click:Connect(function()
		toggles[key] = not toggles[key]
		b.Text = name .. (toggles[key] and ": ON" or ": OFF")
		b.BackgroundColor3 = toggles[key] and Color3.fromRGB(40, 150, 40) or Color3.fromRGB(150, 40, 40)

		if key == "ESP" and not toggles.ESP then
			for _, v in ipairs(ws:GetDescendants()) do
				if v.Name == "NPCHighlight" then v:Destroy() end
			end
		end
	end)
end

local function createSlider(name, minV, maxV, def, cb)
	local container = Instance.new("Frame")
	container.Parent = scroll
	container.BackgroundColor3 = Color3.fromRGB(35, 35, 35)
	container.Size = UDim2.new(0.9, 0, 0, 45)
	container.BorderSizePixel = 0

	local lbl = Instance.new("TextLabel")
	lbl.Parent = container
	lbl.BackgroundTransparency = 1
	lbl.Size = UDim2.new(1, 0, 0.5, 0)
	lbl.Font = Enum.Font.Gotham
	lbl.Text = name .. ": " .. tostring(def)
	lbl.TextColor3 = Color3.fromRGB(255, 255, 255)
	lbl.TextSize = 10

	local back = Instance.new("Frame")
	back.Parent = container
	back.BackgroundColor3 = Color3.fromRGB(20, 20, 20)
	back.Position = UDim2.new(0.1, 0, 0.6, 0)
	back.Size = UDim2.new(0.8, 0, 0, 8)
	back.BorderSizePixel = 0

	local fill = Instance.new("Frame")
	fill.Parent = back
	fill.BackgroundColor3 = Color3.fromRGB(40, 150, 40)
	fill.Size = UDim2.new((def - minV) / (maxV - minV), 0, 1, 0)
	fill.BorderSizePixel = 0

	local dragging = false
	local function update(input)
		local pos = math.clamp((input.Position.X - back.AbsolutePosition.X) / back.AbsoluteSize.X, 0, 1)
		fill.Size = UDim2.new(pos, 0, 1, 0)
		local val = minV + (maxV - minV) * pos
		val = math.floor(val * 100) / 100
		lbl.Text = name .. ": " .. tostring(val)
		if cb then cb(val) end
	end

	back.InputBegan:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
			dragging = true
			update(input)
		end
	end)

	uis.InputEnded:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
			dragging = false
		end
	end)

	uis.InputChanged:Connect(function(input)
		if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
			update(input)
		end
	end)
end

-- Toggles
createToggle("Aim Assist", "AimAssist")
createToggle("Visuals (ESP)", "ESP")
createToggle("Big Hitboxes", "Hitboxes")
createToggle("Fullbright", "Fullbright")
createToggle("Instant Interact", "InstantInteract")
createToggle("Del Corpses", "DelCorpses")

-- Sliders (Aim Strength now 1.0 - 1.3)
createSlider("Aim Strength", 0.1, 1.3, cfg.aimStrength, function(v) cfg.aimStrength = v end)
createSlider("FOV", 10, 300, cfg.fov, function(v) cfg.fov = v end)
createSlider("Hitbox Size", 1, 20, hitboxCfg.size.X, function(v) 
	hitboxCfg.size = Vector3.new(v, v, v) 
end)

uis.InputBegan:Connect(function(input, gpe)
	if not gpe and input.KeyCode == Enum.KeyCode.RightControl then
		main.Visible = not main.Visible
	end
end)

pps.PromptShown:Connect(function(prompt)
	if toggles.InstantInteract then
		prompt.HoldDuration = 0
	end
end)

-- ESP with always visible outline
local function applyEsp(model)
	if not toggles.ESP then return end
	local hl = model:FindFirstChild("NPCHighlight")
	if not hl then
		hl = Instance.new("Highlight")
		hl.Name = "NPCHighlight"
		hl.Parent = model
	end
	hl.FillColor = cfg.espFill
	hl.OutlineColor = cfg.espOutline
	hl.FillTransparency = cfg.fillTrans
	hl.OutlineTransparency = 0          -- Always show outline
	hl.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
end

local crossX = Drawing.new("Line")
local crossY = Drawing.new("Line")
crossX.Visible = true
crossY.Visible = true
crossX.Thickness = 2
crossY.Thickness = 2
crossX.Color = Color3.fromRGB(255, 255, 255)
crossY.Color = Color3.fromRGB(255, 255, 255)

local validTargets = {}

local function refreshTargets()
	local tTargets = {}
	local inRange = {}
	local pPos = lplr.Character and lplr.Character:FindFirstChild("HumanoidRootPart") and lplr.Character.HumanoidRootPart.Position or cam.CFrame.Position

	for _, obj in ipairs(ws:GetChildren()) do
		if not obj:IsA("Model") then continue end
		if plrs:GetPlayerFromCharacter(obj) then continue end -- Skip real players

		local root = obj:FindFirstChild("HumanoidRootPart") or obj.PrimaryPart
		local dist = root and (root.Position - pPos).Magnitude or math.huge

		if dist > cfg.maxDist then
			local old = obj:FindFirstChild("NPCHighlight")
			if old then old:Destroy() end
			continue
		end

		local hum = obj:FindFirstChildOfClass("Humanoid")
		if hum then
			if toggles.DelCorpses and hum.Health <= 0 then 
				obj:Destroy()
				continue 
			end

			local part = obj:FindFirstChild(cfg.targetPart) or obj:FindFirstChild("HumanoidRootPart")
			if hum.Health > 0 and part and not obj:FindFirstChild("REVIVE") then
				table.insert(inRange, {model = obj, dist = dist})
			else
				local old = obj:FindFirstChild("NPCHighlight")
				if old then old:Destroy() end
			end
		end
	end

	table.sort(inRange, function(a, b) return a.dist < b.dist end)

	for i, data in ipairs(inRange) do
		table.insert(tTargets, data.model)
		if i <= cfg.maxEsp then
			applyEsp(data.model)
		else
			local old = data.model:FindFirstChild("NPCHighlight")
			if old then old:Destroy() end
		end
	end

	validTargets = tTargets
end

-- Del Corpses loop
task.spawn(function()
	while true do
		if toggles.DelCorpses then
			for _, obj in ipairs(ws:GetChildren()) do
				if obj:IsA("Model") then
					local hum = obj:FindFirstChildOfClass("Humanoid")
					if hum and hum.Health <= 0 then
						obj:Destroy()
					end
				end
			end
		end
		task.wait(0.1)
	end
end)

-- Target refresh loop
task.spawn(function()
	while true do
		refreshTargets()
		task.wait(cfg.scanRate)
	end
end)

-- Main Aimbot Loop
rs.RenderStepped:Connect(function(dt)
	local center = Vector2.new(cam.ViewportSize.X / 2, cam.ViewportSize.Y / 2)

	crossX.From = Vector2.new(center.X - 10, center.Y)
	crossX.To = Vector2.new(center.X + 10, center.Y)
	crossY.From = Vector2.new(center.X, center.Y - 10)
	crossY.To = Vector2.new(center.X, center.Y + 10)

	if not toggles.AimAssist then
		crossX.Color = Color3.fromRGB(255, 255, 255)
		crossY.Color = Color3.fromRGB(255, 255, 255)
		return
	end

	local bestTorso = nil
	local shortest = cfg.fov
	local ignore = {lplr.Character}

	for _, npc in ipairs(validTargets) do
		local torso = npc:FindFirstChild(cfg.targetPart) or npc:FindFirstChild("HumanoidRootPart")
		if not torso then continue end

		local sPos, onScreen = cam:WorldToViewportPoint(torso.Position)
		if onScreen then
			local dist = (Vector2.new(sPos.X, sPos.Y) - center).Magnitude
			if dist < shortest then
				local rayParams = RaycastParams.new()
				rayParams.FilterDescendantsInstances = ignore
				rayParams.FilterType = Enum.RaycastFilterType.Exclude

				local result = workspace:Raycast(cam.CFrame.Position, (torso.Position - cam.CFrame.Position).Unit * 1000, rayParams)
				
				if not result or result.Instance:IsDescendantOf(torso.Parent) then
					bestTorso = torso
					shortest = dist
				end
			end
		end
	end

	if bestTorso then
		crossX.Color = Color3.fromRGB(255, 0, 0)
		crossY.Color = Color3.fromRGB(255, 0, 0)

		local root = bestTorso.Parent:FindFirstChild("HumanoidRootPart")
		local vel = root and root.AssemblyLinearVelocity or Vector3.zero
		local predictedPos = bestTorso.Position + vel * 0.055

		local sPos = cam:WorldToViewportPoint(predictedPos)
		local dx = sPos.X - center.X
		local dy = sPos.Y - center.Y
		local dist2D = math.sqrt(dx*dx + dy*dy)

		if dist2D > 0.5 then
			local strength = cfg.aimStrength

			if mousemoverel and not uis.TouchEnabled then
				mousemoverel(dx * strength, dy * strength)
			else
				local lerpAlpha = math.clamp(strength * dt * 20, 0, 0.6)
				local targetCF = CFrame.new(cam.CFrame.Position, predictedPos)
				cam.CFrame = cam.CFrame:Lerp(targetCF, lerpAlpha)
			end
		end
	else
		crossX.Color = Color3.fromRGB(255, 255, 255)
		crossY.Color = Color3.fromRGB(255, 255, 255)
	end
end)

-- Improved Hitbox Loop (fixes NPCs not getting hitboxes)
task.spawn(function()
	while true do
		if toggles.Hitboxes then
			for _, obj in ipairs(ws:GetChildren()) do
				if not obj:IsA("Model") or plrs:GetPlayerFromCharacter(obj) then continue end

				local hum = obj:FindFirstChildOfClass("Humanoid")
				if not hum or hum.Health <= 0 then continue end

				-- Try multiple common parts
				for _, partName in ipairs({"Head", "HumanoidRootPart", "Torso", "UpperTorso"}) do
					local tPart = obj:FindFirstChild(partName)
					if tPart and tPart:IsA("BasePart") then
						if not tPart:GetAttribute("OrigSize") then
							tPart:SetAttribute("OrigSize", tPart.Size)
							tPart:SetAttribute("OrigTrans", tPart.Transparency)
							tPart:SetAttribute("OrigColor", tPart.BrickColor.Name)
						end

						tPart.Size = hitboxCfg.size
						tPart.CanCollide = false
						tPart.Massless = true

						if hitboxCfg.show then
							tPart.Transparency = hitboxCfg.trans
							tPart.BrickColor = hitboxCfg.color
						end
					end
				end
			end
		else
			-- Reset hitboxes when toggled off
			for _, obj in ipairs(ws:GetChildren()) do
				if obj:IsA("Model") then
					for _, partName in ipairs({"Head", "HumanoidRootPart", "Torso", "UpperTorso"}) do
						local tPart = obj:FindFirstChild(partName)
						if tPart and tPart:GetAttribute("OrigSize") then
							tPart.Size = tPart:GetAttribute("OrigSize")
							tPart.Transparency = tPart:GetAttribute("OrigTrans")
							tPart.BrickColor = BrickColor.new(tPart:GetAttribute("OrigColor"))
						end
					end
				end
			end
		end
		task.wait(hitboxCfg.refreshRate)
	end
end)

-- Fullbright
lighting.Changed:Connect(function()
	if not toggles.Fullbright then return end
	lighting.Brightness = 2
	lighting.ClockTime = 14
	lighting.FogEnd = 100000
	lighting.GlobalShadows = false
	lighting.Ambient = Color3.fromRGB(178, 178, 178)
	lighting.OutdoorAmbient = Color3.fromRGB(178, 178, 178)
end)

print("FOOL script")
print("fixed hitbox and aim assist")
