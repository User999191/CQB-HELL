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
	DelCorpses = false,
	Speed = false,
	Ammo = false,
	MinDamage = false,
	Jammed = false,
	Kickback = false,
	MinRecoil = false,
	MaxRecoil = false,
	GunRecoil = false,
	CamRecoil = false
}

local cfg = {
	aimStrength = 1.0,
	fov = 60,
	targetPart = "Head",
	maxDist = 500,
	maxEsp = 10,
	espFill = Color3.fromRGB(255, 0, 0),
	espOutline = Color3.fromRGB(255, 255, 255),
	fillTrans = 0.5,
	outTrans = 0,
	scanRate = 0.09,
	walkSpeed = 45,
	ammoValue = 10000000,
	storedAmmoValue = math.huge,
	damageValue = 20000,
	kickbackValue = 0,
	minRecoilValue = 0,
	maxRecoilValue = 0,
	gunRecoilValue = 0,
	camRecoilValue = 0
}

local hitboxCfg = {
	enabled = true,
	part = "Head",
	size = Vector3.new(5, 5, 5),
	show = true,
	color = BrickColor.new("Bright red"),
	trans = 0.6,
	refreshRate = 0.3
}

local gunModThread = nil
local gunModEnabled = false

local gui = Instance.new("ScreenGui")
gui.Name = "OmniMenu"
gui.Parent = cg
gui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
gui.IgnoreGuiInset = true

local stroke = Instance.new("UIStroke")
stroke.Color = Color3.fromRGB(0, 0, 0)
stroke.Thickness = 1
stroke.Transparency = 0
stroke.Parent = gui

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
main.ZIndex = 2

local mainStroke = Instance.new("UIStroke")
mainStroke.Color = Color3.fromRGB(0, 0, 0)
mainStroke.Thickness = 1
mainStroke.Transparency = 0
mainStroke.Parent = main

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
title.ZIndex = 3

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
minBtn.ZIndex = 3

local scroll = Instance.new("ScrollingFrame")
scroll.Name = "Scroll"
scroll.Parent = main
scroll.BackgroundTransparency = 1
scroll.BorderSizePixel = 0
scroll.Position = UDim2.new(0, 0, 0, 35)
scroll.Size = UDim2.new(1, 0, 1, -35)
scroll.ScrollBarThickness = 2
scroll.AutomaticCanvasSize = Enum.AutomaticSize.Y
scroll.ZIndex = 2

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

local confirmationActive = false
local pendingGunModToggle = nil

local function showConfirmationDialog(callback)
	if confirmationActive then return end
	confirmationActive = true
	
	local overlay = Instance.new("Frame")
	overlay.Name = "Overlay"
	overlay.Parent = gui
	overlay.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
	overlay.BackgroundTransparency = 0.6
	overlay.Size = UDim2.new(1, 0, 1, 0)
	overlay.ZIndex = 10

	local dialog = Instance.new("Frame")
	dialog.Parent = overlay
	dialog.BackgroundColor3 = Color3.fromRGB(200, 0, 0)
	dialog.BorderSizePixel = 0
	dialog.Position = UDim2.new(0.5, -150, 0.5, -75)
	dialog.Size = UDim2.new(0, 300, 0, 150)
	dialog.ZIndex = 11
	
	local dialogStroke = Instance.new("UIStroke")
	dialogStroke.Color = Color3.fromRGB(0, 0, 0)
	dialogStroke.Thickness = 2
	dialogStroke.Parent = dialog

	local warningLabel = Instance.new("TextLabel")
	warningLabel.Parent = dialog
	warningLabel.BackgroundTransparency = 1
	warningLabel.Size = UDim2.new(1, 0, 0.6, 0)
	warningLabel.Font = Enum.Font.GothamBold
	warningLabel.Text = "⚠️ WARNING ⚠️\nGun Mods are UNBYPASSABLE!\nYou WILL get kicked 100%.\nAre you sure?"
	warningLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
	warningLabel.TextSize = 14
	warningLabel.TextWrapped = true
	warningLabel.ZIndex = 12

	local okBtn = Instance.new("TextButton")
	okBtn.Parent = dialog
	okBtn.BackgroundColor3 = Color3.fromRGB(150, 40, 40)
	okBtn.Position = UDim2.new(0.1, 0, 0.7, 0)
	okBtn.Size = UDim2.new(0.35, 0, 0.2, 0)
	okBtn.Font = Enum.Font.GothamBold
	okBtn.Text = "OKAY"
	okBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
	okBtn.TextSize = 14
	okBtn.ZIndex = 12

	local noBtn = Instance.new("TextButton")
	noBtn.Parent = dialog
	noBtn.BackgroundColor3 = Color3.fromRGB(40, 150, 40)
	noBtn.Position = UDim2.new(0.55, 0, 0.7, 0)
	noBtn.Size = UDim2.new(0.35, 0, 0.2, 0)
	noBtn.Font = Enum.Font.GothamBold
	noBtn.Text = "NO"
	noBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
	noBtn.TextSize = 14
	noBtn.ZIndex = 12

	local closed = false
	okBtn.MouseButton1Click:Connect(function()
		if closed then return end
		closed = true
		confirmationActive = false
		overlay:Destroy()
		if callback then callback(true) end
	end)

	noBtn.MouseButton1Click:Connect(function()
		if closed then return end
		closed = true
		confirmationActive = false
		overlay:Destroy()
		if callback then callback(false) end
	end)
end

local function startGunMods()
	if gunModThread then
		task.cancel(gunModThread)
		gunModThread = nil
	end
	
	gunModEnabled = true
	gunModThread = task.spawn(function()
		while gunModEnabled do
			task.wait(0.095)
			local success = pcall(function()
				for _, v in pairs(getgc(true)) do
					if type(v) == "table" then
						if toggles.Ammo then
							if rawget(v, "Ammo") then
								v.Ammo = cfg.ammoValue
							end
							if rawget(v, "StoredAmmo") then
								v.StoredAmmo = cfg.storedAmmoValue
							end
						end
						
						if toggles.Kickback and rawget(v, "Kickback") then
							v.Kickback = cfg.kickbackValue
						end
						
						if toggles.MinDamage and rawget(v, "MinDamage") then
							v.MinDamage = cfg.damageValue
						end
						
						if toggles.Jammed and rawget(v, "Jammed") then
							v.Jammed = false
						end
						
						if toggles.MinRecoil and rawget(v, "MinRecoilPower") then
							v.MinRecoilPower = cfg.minRecoilValue
						end
						
						if toggles.MaxRecoil and rawget(v, "MaxRecoilPower") then
							v.MaxRecoilPower = cfg.maxRecoilValue
						end
						
						if toggles.GunRecoil and rawget(v, "gunRecoil") then
							v.gunRecoil = cfg.gunRecoilValue
						end
						
						if toggles.CamRecoil and rawget(v, "camRecoil") then
							v.camRecoil = cfg.camRecoilValue
						end
					end
				end
			end)
		end
	end)
end

local function stopGunMods()
	gunModEnabled = false
	if gunModThread then
		task.cancel(gunModThread)
		gunModThread = nil
	end
end

local function updateGunModsState()
	if toggles.Ammo or toggles.MinDamage or toggles.Jammed or toggles.Kickback or toggles.MinRecoil or toggles.MaxRecoil or toggles.GunRecoil or toggles.CamRecoil then
		if not gunModEnabled then
			startGunMods()
		end
	else
		if gunModEnabled then
			stopGunMods()
		end
	end
end

local function createToggle(name, key, requiresConfirm)
	local b = Instance.new("TextButton")
	b.Parent = scroll
	b.BackgroundColor3 = toggles[key] and Color3.fromRGB(40, 150, 40) or Color3.fromRGB(150, 40, 40)
	b.Size = UDim2.new(0.9, 0, 0, 35)
	b.Font = Enum.Font.Gotham
	b.Text = name .. (toggles[key] and ": ON" or ": OFF")
	b.TextColor3 = Color3.fromRGB(255, 255, 255)
	b.TextSize = 12
	b.BorderSizePixel = 0
	b.ZIndex = 2

	local btnStroke = Instance.new("UIStroke")
	btnStroke.Color = Color3.fromRGB(0, 0, 0)
	btnStroke.Thickness = 0.5
	btnStroke.Parent = b

	b.MouseButton1Click:Connect(function()
		local function applyToggle()
			toggles[key] = not toggles[key]
			b.Text = name .. (toggles[key] and ": ON" or ": OFF")
			b.BackgroundColor3 = toggles[key] and Color3.fromRGB(40, 150, 40) or Color3.fromRGB(150, 40, 40)

			if key == "ESP" and not toggles.ESP then
				for _, v in ipairs(ws:GetDescendants()) do
					if v.Name == "NPCHighlight" then v:Destroy() end
				end
			end
			
			if key == "Speed" then
				local char = lplr.Character
				local hum = char and char:FindFirstChildOfClass("Humanoid")
				if hum then
					if toggles.Speed then
						hum.WalkSpeed = cfg.walkSpeed
					else
						hum.WalkSpeed = 16
					end
				end
			end
			
			if key == "Ammo" or key == "MinDamage" or key == "Jammed" or key == "Kickback" or key == "MinRecoil" or key == "MaxRecoil" or key == "GunRecoil" or key == "CamRecoil" then
				updateGunModsState()
			end
		end
		
		if requiresConfirm and not toggles[key] then
			showConfirmationDialog(function(confirmed)
				if confirmed then
					applyToggle()
				end
			end)
		else
			applyToggle()
		end
	end)
end

local function createSlider(name, minV, maxV, def, cb)
	local container = Instance.new("Frame")
	container.Parent = scroll
	container.BackgroundColor3 = Color3.fromRGB(35, 35, 35)
	container.Size = UDim2.new(0.9, 0, 0, 45)
	container.BorderSizePixel = 0
	container.ZIndex = 2

	local lbl = Instance.new("TextLabel")
	lbl.Parent = container
	lbl.BackgroundTransparency = 1
	lbl.Size = UDim2.new(1, 0, 0.5, 0)
	lbl.Font = Enum.Font.Gotham
	lbl.Text = name .. ": " .. tostring(def)
	lbl.TextColor3 = Color3.fromRGB(255, 255, 255)
	lbl.TextSize = 10
	lbl.ZIndex = 3

	local back = Instance.new("Frame")
	back.Parent = container
	back.BackgroundColor3 = Color3.fromRGB(20, 20, 20)
	back.Position = UDim2.new(0.1, 0, 0.6, 0)
	back.Size = UDim2.new(0.8, 0, 0, 8)
	back.BorderSizePixel = 0
	back.ZIndex = 2

	local fill = Instance.new("Frame")
	fill.Parent = back
	fill.BackgroundColor3 = Color3.fromRGB(40, 150, 40)
	fill.Size = UDim2.new((def - minV) / (maxV - minV), 0, 1, 0)
	fill.BorderSizePixel = 0
	fill.ZIndex = 3

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

createToggle("Aim Assist", "AimAssist", false)
createToggle("Visuals (ESP)", "ESP", false)
createToggle("Big Hitboxes", "Hitboxes", false)
createToggle("Fullbright", "Fullbright", false)
createToggle("Instant Interact", "InstantInteract", false)
createToggle("Del Corpses", "DelCorpses", false)
createToggle("Speed 45", "Speed", false)

local gunCategory = Instance.new("TextLabel")
gunCategory.Parent = scroll
gunCategory.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
gunCategory.Size = UDim2.new(0.9, 0, 0, 25)
gunCategory.Font = Enum.Font.GothamBold
gunCategory.Text = "--- GUN MODS (RISKY) ---"
gunCategory.TextColor3 = Color3.fromRGB(255, 200, 0)
gunCategory.TextSize = 11
gunCategory.ZIndex = 2

createToggle("Ammo (10000)", "Ammo", true)
createToggle("Min Damage (70)", "MinDamage", true)
createToggle("No Jam", "Jammed", true)
createToggle("Kickback", "Kickback", true)
createToggle("Min Recoil (0)", "MinRecoil", true)
createToggle("Max Recoil (0)", "MaxRecoil", true)
createToggle("Gun Recoil (0)", "GunRecoil", true)
createToggle("Cam Recoil (0)", "CamRecoil", true)

createSlider("Aim Strength", 0.1, 1.5, cfg.aimStrength, function(v) cfg.aimStrength = v end)
createSlider("FOV", 10, 300, cfg.fov, function(v) cfg.fov = v end)
createSlider("Hitbox Size", 1, 18, hitboxCfg.size.X, function(v) hitboxCfg.size = Vector3.new(v, v, v) end)

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
	hl.OutlineTransparency = 0
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
		if plrs:GetPlayerFromCharacter(obj) then continue end

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

task.spawn(function()
	while true do
		refreshTargets()
		task.wait(cfg.scanRate)
	end
end)

task.spawn(function()
	while true do
		if toggles.Speed then
			local char = lplr.Character
			local hum = char and char:FindFirstChildOfClass("Humanoid")
			if hum and hum.WalkSpeed ~= cfg.walkSpeed then
				hum.WalkSpeed = cfg.walkSpeed
			end
		end
		task.wait()
	end
end)

lplr.CharacterAdded:Connect(function(char)
	task.wait(0.5)
	if toggles.Speed then
		local hum = char:FindFirstChildOfClass("Humanoid")
		if hum then
			hum.WalkSpeed = cfg.walkSpeed
		end
	end
end)

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

		local root = bestTorso.Parent and bestTorso.Parent:FindFirstChild("HumanoidRootPart")
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

task.spawn(function()
	while true do
		if toggles.Hitboxes then
			for _, obj in ipairs(ws:GetChildren()) do
				if not obj:IsA("Model") or plrs:GetPlayerFromCharacter(obj) then continue end

				local hum = obj:FindFirstChildOfClass("Humanoid")
				if not hum or hum.Health <= 0 then continue end

				local tPart = obj:FindFirstChild("Head")
				if tPart and tPart:IsA("BasePart") then
					if not tPart:GetAttribute("OrigSize") then
						tPart:SetAttribute("OrigSize", tPart.Size)
						tPart:SetAttribute("OrigTrans", tPart.Transparency)
						tPart:SetAttribute("OrigColor", tPart.BrickColor.Name)
					end

					if tPart.Size ~= hitboxCfg.size then
						tPart.Size = hitboxCfg.size
					end
					tPart.CanCollide = false
					tPart.Massless = true

					if hitboxCfg.show then
						tPart.Transparency = hitboxCfg.trans
						tPart.BrickColor = hitboxCfg.color
					else
						tPart.Transparency = tPart:GetAttribute("OrigTrans") or 0
					end
				end
			end
		else
			for _, obj in ipairs(ws:GetChildren()) do
				if obj:IsA("Model") then
					local tPart = obj:FindFirstChild("Head")
					if tPart and tPart:GetAttribute("OrigSize") then
						tPart.Size = tPart:GetAttribute("OrigSize")
						tPart.Transparency = tPart:GetAttribute("OrigTrans") or 0
						tPart.BrickColor = BrickColor.new(tPart:GetAttribute("OrigColor"))
						tPart.CanCollide = true
						tPart.Massless = false
					end
				end
			end
		end
		task.wait(hitboxCfg.refreshRate)
	end
end)

lighting.Changed:Connect(function()
	if not toggles.Fullbright then return end
	lighting.Brightness = 2
	lighting.ClockTime = 14
	lighting.FogEnd = 100000
	lighting.GlobalShadows = false
	lighting.Ambient = Color3.fromRGB(178, 178, 178)
	lighting.OutdoorAmbient = Color3.fromRGB(178, 178, 178)
end)	fillTrans = 0.5,
	outTrans = 0,
	scanRate = 0.5,
	walkSpeed = 45
}

local hitboxCfg = {
	enabled = true,
	part = "Head",
	size = Vector3.new(5, 5, 5),
	show = true,
	color = BrickColor.new("Bright red"),
	trans = 0.6,
	refreshRate = 0.3
}

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
		
		if key == "Speed" then
			local char = lplr.Character
			local hum = char and char:FindFirstChildOfClass("Humanoid")
			if hum then
				if toggles.Speed then
					hum.WalkSpeed = cfg.walkSpeed
				else
					hum.WalkSpeed = 16
				end
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

createToggle("Aim Assist", "AimAssist")
createToggle("Visuals (ESP)", "ESP")
createToggle("Big Hitboxes", "Hitboxes")
createToggle("Fullbright", "Fullbright")
createToggle("Instant Interact", "InstantInteract")
createToggle("Del Corpses", "DelCorpses")
createToggle("Speed 45", "Speed")

createSlider("Aim Strength", 0.1, 1.5, cfg.aimStrength, function(v) cfg.aimStrength = v end)
createSlider("FOV", 10, 300, cfg.fov, function(v) cfg.fov = v end)
createSlider("Hitbox Size", 1, 18, hitboxCfg.size.X, function(v) hitboxCfg.size = Vector3.new(v, v, v) end)

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
	hl.OutlineTransparency = 0
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
		if plrs:GetPlayerFromCharacter(obj) then continue end

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

task.spawn(function()
	while true do
		refreshTargets()
		task.wait(cfg.scanRate)
	end
end)

-- Speed control
task.spawn(function()
	while true do
		if toggles.Speed then
			local char = lplr.Character
			local hum = char and char:FindFirstChildOfClass("Humanoid")
			if hum and hum.WalkSpeed ~= cfg.walkSpeed then
				hum.WalkSpeed = cfg.walkSpeed
			end
		end
		task.wait(0.5)
	end
end)

-- Auto apply speed when character respawns
lplr.CharacterAdded:Connect(function(char)
	task.wait(0.5)
	if toggles.Speed then
		local hum = char:FindFirstChildOfClass("Humanoid")
		if hum then
			hum.WalkSpeed = cfg.walkSpeed
		end
	end
end)

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

		local root = bestTorso.Parent and bestTorso.Parent:FindFirstChild("HumanoidRootPart")
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

task.spawn(function()
	while true do
		if toggles.Hitboxes then
			for _, obj in ipairs(ws:GetChildren()) do
				if not obj:IsA("Model") or plrs:GetPlayerFromCharacter(obj) then continue end

				local hum = obj:FindFirstChildOfClass("Humanoid")
				if not hum or hum.Health <= 0 then continue end

				local tPart = obj:FindFirstChild("Head")
				if tPart and tPart:IsA("BasePart") then
					if not tPart:GetAttribute("OrigSize") then
						tPart:SetAttribute("OrigSize", tPart.Size)
						tPart:SetAttribute("OrigTrans", tPart.Transparency)
						tPart:SetAttribute("OrigColor", tPart.BrickColor.Name)
					end

					if tPart.Size ~= hitboxCfg.size then
						tPart.Size = hitboxCfg.size
					end
					tPart.CanCollide = false
					tPart.Massless = true

					if hitboxCfg.show then
						tPart.Transparency = hitboxCfg.trans
						tPart.BrickColor = hitboxCfg.color
					else
						tPart.Transparency = tPart:GetAttribute("OrigTrans") or 0
					end
				end
			end
		else
			for _, obj in ipairs(ws:GetChildren()) do
				if obj:IsA("Model") then
					local tPart = obj:FindFirstChild("Head")
					if tPart and tPart:GetAttribute("OrigSize") then
						tPart.Size = tPart:GetAttribute("OrigSize")
						tPart.Transparency = tPart:GetAttribute("OrigTrans") or 0
						tPart.BrickColor = BrickColor.new(tPart:GetAttribute("OrigColor"))
						tPart.CanCollide = true
						tPart.Massless = false
					end
				end
			end
		end
		task.wait(hitboxCfg.refreshRate)
	end
end)

lighting.Changed:Connect(function()
	if not toggles.Fullbright then return end
	lighting.Brightness = 2
	lighting.ClockTime = 14
	lighting.FogEnd = 100000
	lighting.GlobalShadows = false
	lighting.Ambient = Color3.fromRGB(178, 178, 178)
	lighting.OutdoorAmbient = Color3.fromRGB(178, 178, 178)
end)
