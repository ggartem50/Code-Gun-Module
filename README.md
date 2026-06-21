local GunModule = {}

local RS = game:GetService("ReplicatedStorage")
local Players = game:GetService("Players")
local Workspace = game:GetService("Workspace")

local Remotes = RS:WaitForChild("Remotes")
local Modules = RS:WaitForChild("Modules")
local Assets = RS:WaitForChild("Assets")
local GunConfig = require(Modules:WaitForChild("GunConfig"))
local TeamConfig = require(Modules:WaitForChild("TeamConfig"))

local GunShootEvent = Remotes:WaitForChild("GunShootEvent")
local GunEffectsEvent = Remotes:WaitForChild("GunEffectsEvent")
local GunArmEvent = Remotes:WaitForChild("GunArmEvent")
local SprintEvent = Remotes:WaitForChild("SprintEvent")
local DamageIndicatorEvent = Remotes:WaitForChild("DamageIndicatorEvent")
local CrouchEvent = Remotes:WaitForChild("CrouchEvent")
local MediHealEvent = Remotes:WaitForChild("MediHealEvent")
local MediHealEffectEvent = Remotes:WaitForChild("MediHealEffectEvent")
local KillEvent = Remotes:WaitForChild("KillEvent")
local KillFeedEvent = Remotes:WaitForChild("KillFeedEvent")

function GunModule.initClient()
	local UIS        = game:GetService("UserInputService")
	local RunService = game:GetService("RunService")
	local TweenService = game:GetService("TweenService")

	local tracersFolder = Assets:WaitForChild("Tracers")
	local beamTemplates = {
		Hostile   = tracersFolder:WaitForChild("Hostile"),
		Defenders = tracersFolder:WaitForChild("Defenders"),
		Neutral   = tracersFolder:WaitForChild("Neutral"),
		Heal      = tracersFolder:WaitForChild("Heal"),
	}

	local function getBeam(teamName)
		if teamName == "Hostiles" then return beamTemplates.Defenders
		elseif teamName == "Slavic Corps" or teamName == "Slavic Cadet Military Corps" then return beamTemplates.Hostile
		else return beamTemplates.Neutral end
	end

	local selBoxTemplate   = Assets:WaitForChild("HitOutline"):WaitForChild("SelectionBox")
	local hitSound         = Assets:WaitForChild("HitSounds"):WaitForChild("HitSound")
	local killSound        = Assets:WaitForChild("EnemyKilled"):WaitForChild("KillSound")
	local damageTakenSound = Assets:WaitForChild("DamageSound"):WaitForChild("DamageTaken")

	local vals = RS:WaitForChild("Values")
	local weaponSoundsEnabled      = vals:WaitForChild("WeaponSoundsEnabled")
	local hitSoundsEnabled         = vals:WaitForChild("HitSoundsEnabled")
	local damageTakenSoundsEnabled = vals:WaitForChild("DamageTakenSoundsEnabled")
	local enemyKilledSoundsEnabled = vals:WaitForChild("EnemyKilledSoundsEnabled")

	local function playFireSound(muzzle)
		if not weaponSoundsEnabled.Value then return end
		local s = muzzle:FindFirstChild("FireSound")
		if s then s:Play() end
	end

	local TRACER_LIFE   = 0.025
	local MEDI_RADIUS   = 15
	local MEDI_TLIFE    = 0.025
	local healBeams     = {}

	local plr     = Players.LocalPlayer
	local mouse   = plr:GetMouse()
	local char    = plr.Character or plr.CharacterAdded:Wait()

	local equippedGun = nil
	local canShoot    = true
	local mouseHeld   = false
	local autoLoop    = nil
	local isSprinting = false
	local isReloading = false
	local isAlive     = true
	local isCrouching = false
	local crouchTrack = nil
	local crouchAnimId = nil
	local sprintTrack = nil
	local fireTrack   = nil

	do
		local nv = game.StarterPlayer.StarterCharacterScripts:WaitForChild("CrouchAnimation", 5)
		if nv then crouchAnimId = nv.Value end
	end

	local ammoState   = {}
	local currentAmmo = 0
	local totalAmmo   = 0

	local gunGui, frameImg, frameAmmo, gunImage, lblCurrent, lblTotal, ammoBarFill, lblGunName
	local AMMO_BAR_MAX = 0.644736826

	local function refreshGuiRefs()
		gunGui   = plr.PlayerGui:WaitForChild("GunGui")
		local f2 = gunGui:WaitForChild("Frame2")
		local f  = f2:WaitForChild("Frame")
		local f1 = f2:WaitForChild("Frame1")
		frameImg     = f2
		frameAmmo    = f2
		gunImage     = f:WaitForChild("ImageLabel")
		lblCurrent   = f1:WaitForChild("CurrentAmmo")
		lblTotal     = f1:WaitForChild("TotalAmmo")
		lblCurrent.RichText = true
		lblTotal.RichText   = true
		ammoBarFill = f2:WaitForChild("AmmoBar")
		lblGunName  = f2:WaitForChild("GunName")
	end
	refreshGuiRefs()

	local GREY  = "rgb(120,120,120)"
	local WHITE = "rgb(255,255,255)"

	local function richAmmo(n)
		local h = math.floor(n / 100)
		local t = math.floor((n % 100) / 10)
		local o = n % 10
		local hc = (h == 0) and GREY or WHITE
		local tc = (h == 0 and t == 0) and GREY or WHITE
		local oc = (n == 0) and GREY or WHITE
		return string.format('<font color="%s">%d</font><font color="%s">%d</font><font color="%s">%d</font>', hc,h, tc,t, oc,o)
	end

	local function isMedi(name) return name == "Medi-Gun" end
	local function cfgName(toolName) return toolName:gsub("_Equipped$", "") end

	local function updateAmmoGui()
		local gn  = equippedGun and cfgName(equippedGun.Name)
		local cfg = gn and GunConfig[gn]
		local dc  = currentAmmo
		local dt  = totalAmmo
		if cfg and isMedi(gn) then
			dc = math.floor(currentAmmo / cfg.AmmoInMag * 100)
			dt = 100
		end
		lblCurrent.Text = richAmmo(dc)
		lblTotal.Text   = richAmmo(dt)
		if ammoBarFill then
			local ratio = dt > 0 and (dc / dt) or 0
			local fill  = AMMO_BAR_MAX * ratio
			ammoBarFill.Visible = dc > 0
			TweenService:Create(ammoBarFill, TweenInfo.new(0.1, Enum.EasingStyle.Quint, Enum.EasingDirection.Out), {
				Size = UDim2.new(-fill, 0, ammoBarFill.Size.Y.Scale, ammoBarFill.Size.Y.Offset)
			}):Play()
		end
	end

	local FRAME_IN   = UDim2.new(0.76561296, 0, 0.709999979, 0)
	local FRAME_OUT  = UDim2.new(1.3, 0, 0.709999979, 0)
	local twIn       = TweenInfo.new(0.35, Enum.EasingStyle.Quint, Enum.EasingDirection.Out)
	local twOut      = TweenInfo.new(0.25, Enum.EasingStyle.Quint, Enum.EasingDirection.In)
	local curTween   = nil
	local hideThread = nil

	local function showGunGui(cfg, gn)
		if curTween then curTween:Cancel() end
		if hideThread then task.cancel(hideThread) end
		gunImage.Image   = cfg.Image or ""
		totalAmmo = isMedi(gn) and 100 or cfg.AmmoInMag
		lblTotal.Text    = richAmmo(totalAmmo)
		if lblGunName then lblGunName.Text = gn or "" end
		frameImg.Visible = true
		curTween = TweenService:Create(frameImg, twIn, {Position = FRAME_IN})
		curTween:Play()
	end

	local function hideGunGui()
		if curTween then curTween:Cancel() end
		if hideThread then task.cancel(hideThread) end
		curTween = TweenService:Create(frameImg, twOut, {Position = FRAME_OUT})
		curTween:Play()
		hideThread = task.delay(0.25, function()
			frameImg.Visible = false
			hideThread = nil
		end)
	end

	local hitOutlineEnabled = RS:WaitForChild("Values"):WaitForChild("HitOutlineEnabled")
	local function showHitOutline(part, color)
		if not hitOutlineEnabled.Value then return end
		if not part or not part:IsA("BasePart") then return end
		local box = selBoxTemplate:Clone()
		box.Adornee = part
		if color then box.Color3 = color box.SurfaceColor3 = color end
		box.Transparency = 1
		box.Parent = Workspace
		local tIn  = TweenService:Create(box, TweenInfo.new(0.05, Enum.EasingStyle.Linear), {Transparency = 0})
		local tOut = TweenService:Create(box, TweenInfo.new(0.1,  Enum.EasingStyle.Linear), {Transparency = 1})
		tIn:Play()
		tIn.Completed:Connect(function()
			tOut:Play()
			tOut.Completed:Connect(function() box:Destroy() end)
		end)
	end

	local function showHitMarker()
		local vf = RS:WaitForChild("Values")
		if not vf:FindFirstChild("HitMarkersEnabled") then return end
		if not vf.HitMarkersEnabled.Value then return end
		local styleNum  = (vf:FindFirstChild("HitMarkerStyle") and vf.HitMarkerStyle.Value) or 1
		local moveStyle = (vf:FindFirstChild("HitMarkerMoveStyle") and vf.HitMarkerMoveStyle.Value) or "Rotate"
		local pg = plr:FindFirstChild("PlayerGui"); if not pg then return end
		local main = pg:FindFirstChild("Menu") and pg.Menu:FindFirstChild("Main"); if not main then return end
		local s  = main:FindFirstChild("Settings"); if not s then return end
		local iv = s:FindFirstChild("INTERFACE"); if not iv then return end
		local hv = iv:FindFirstChild("HITVISUALS"); if not hv then return end
		local ct = hv:FindFirstChild("CursorHitMarkersText"); if not ct then return end
		local sf = ct:FindFirstChild("Settings"); if not sf then return end
		local ff = sf:FindFirstChild("Frame"); if not ff then return end
		local mf = ff:FindFirstChild("HitMarkers"); if not mf then return end
		local selectedImg = nil
		for _, img in ipairs(mf:GetChildren()) do
			if img:IsA("ImageLabel") then
				local n = tonumber(img.Name:match("%d+"))
				if n == styleNum then selectedImg = img.Image break end
			end
		end
		if not selectedImg or selectedImg == "" then return end
		local sg = pg:FindFirstChild("DamageIndicatorGui"); if not sg then return end
		local SIZE = 28
		local mk = Instance.new("ImageLabel")
		mk.BackgroundTransparency = 1
		mk.Image  = selectedImg
		mk.Size   = UDim2.new(0, SIZE, 0, SIZE)
		mk.AnchorPoint = Vector2.new(0.5, 0.5)
		mk.ImageTransparency = 0
		mk.ZIndex  = 20
		mk.Parent  = sg
		local conn
		conn = RunService.RenderStepped:Connect(function()
			if not mk.Parent then conn:Disconnect() return end
			local mp = UIS:GetMouseLocation()
			mk.Position = UDim2.new(0, mp.X, 0, mp.Y)
		end)
		local dur  = 0.25
		local goal = {}
		if moveStyle == "Rotate" then
			local startR = math.random(0, 359)
			local dir    = math.random() > 0.5 and 1 or -1
			mk.Rotation  = startR
			goal.Rotation = startR + dir * math.random(60, 120)
			goal.ImageTransparency = 1
		else
			goal.Size  = UDim2.new(0, SIZE * 2.2, 0, SIZE * 2.2)
			goal.ImageTransparency = 1
		end
		local t = TweenService:Create(mk, TweenInfo.new(dur, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), goal)
		t:Play()
		t.Completed:Connect(function() conn:Disconnect() mk:Destroy() end)
	end

	local function fireTracer(muzzleAtt, hitPos, teamName)
		local a0 = Instance.new("Attachment")
		a0.Parent = Workspace.Terrain
		a0.WorldPosition = muzzleAtt.WorldPosition
		local a1 = Instance.new("Attachment")
		a1.Parent = Workspace.Terrain
		a1.WorldPosition = hitPos
		local beam = getBeam(teamName):Clone()
		beam.Attachment0 = a0
		beam.Attachment1 = a1
		beam.Enabled = true
		beam.Parent = Workspace.Terrain
		task.delay(TRACER_LIFE, function()
			beam:Destroy() a0:Destroy() a1:Destroy()
		end)
	end

	local function clearHealBeam(tc)
		local t = healBeams[tc]
		if not t then return end
		if t.beam and t.beam.Parent then t.beam:Destroy() end
		if t.a1  and t.a1.Parent  then t.a1:Destroy()  end
		healBeams[tc] = nil
	end

	local function showHealBeam(muzzleAtt, tc)
		clearHealBeam(tc)
		local torso = tc:FindFirstChild("Torso") or tc:FindFirstChild("UpperTorso")
		if not torso then return end
		local a1 = Instance.new("Attachment"); a1.Parent = torso
		local beam = beamTemplates.Heal:Clone()
		beam.Attachment0 = muzzleAtt
		beam.Attachment1 = a1
		beam.Enabled = true
		beam.Parent = Workspace.Terrain
		healBeams[tc] = {beam = beam, a1 = a1}
		task.delay(MEDI_TLIFE, function() clearHealBeam(tc) end)
	end

	local function findHealTarget()
		local myTeam = plr.Team and plr.Team.Name
		local root   = char:FindFirstChild("HumanoidRootPart")
		if not root then return nil end
		local myPos = root.Position
		local best, bestDist = nil, MEDI_RADIUS + 1
		for _, other in ipairs(Players:GetPlayers()) do
			if other == plr then continue end
			if not TeamConfig.canHeal(myTeam, other.Team and other.Team.Name) then continue end
			local oc = other.Character; if not oc then continue end
			local oh = oc:FindFirstChildOfClass("Humanoid")
			if not oh or oh.Health <= 0 or oh.Health >= oh.MaxHealth then continue end
			local or2 = oc:FindFirstChild("HumanoidRootPart") or oc:FindFirstChild("Torso")
			if not or2 then continue end
			local dist = (or2.Position - myPos).Magnitude
			if dist > MEDI_RADIUS then continue end
			local rp = RaycastParams.new()
			rp.FilterType = Enum.RaycastFilterType.Exclude
			rp.FilterDescendantsInstances = {char, oc}
			if Workspace:Raycast(myPos, or2.Position - myPos, rp) then continue end
			if dist < bestDist then bestDist = dist best = other end
		end
		return best
	end

	local SPRINT_SPD = 22
	local NORMAL_SPD = 16
	local CROUCH_SPD = 12

	local function setCrouch(active)
		if isCrouching == active then return end
		isCrouching = active
		CrouchEvent:FireServer(active)
		local hum = char and char:FindFirstChildOfClass("Humanoid")
		if hum then hum.WalkSpeed = active and CROUCH_SPD or NORMAL_SPD end
		if not active then
			if crouchTrack then crouchTrack:Stop() crouchTrack = nil end
			return
		end
		local animator = hum and hum:FindFirstChildOfClass("Animator")
		if animator and crouchAnimId then
			local anim = Instance.new("Animation")
			anim.AnimationId = "rbxassetid://" .. tostring(crouchAnimId)
			local track = animator:LoadAnimation(anim)
			anim:Destroy()
			track.Priority = Enum.AnimationPriority.Action
			track.Looped   = true
			track:Play()
			local moving = hum.MoveDirection.Magnitude > 0.1
			track:AdjustSpeed(moving and 1 or 0)
			wasMoving    = moving
			crouchTrack  = track
		end
	end

	local function stopSprintAnim()
		if sprintTrack then sprintTrack:Stop() sprintTrack = nil end
	end

	local function loadFireAnim()
		if fireTrack then fireTrack:Stop() fireTrack = nil end
		if not equippedGun then return end
		local cfg = GunConfig[cfgName(equippedGun.Name)]
		if not cfg or not cfg.FireAnimId then return end
		local hum = char and char:FindFirstChildOfClass("Humanoid")
		local animator = hum and hum:FindFirstChildOfClass("Animator")
		if not animator then return end
		local anim = Instance.new("Animation")
		anim.AnimationId = "rbxassetid://" .. tostring(cfg.FireAnimId)
		local track = animator:LoadAnimation(anim)
		anim:Destroy()
		track.Priority = Enum.AnimationPriority.Action3
		track.Looped   = false
		fireTrack = track
	end

	local function playFireAnim()
		if not fireTrack then return end
		if fireTrack.IsPlaying then fireTrack:Stop() end
		fireTrack:Play()
	end

	local function startSprintAnim()
		stopSprintAnim()
		if not equippedGun then return end
		local cfg = GunConfig[cfgName(equippedGun.Name)]
		if not cfg or not cfg.SprintAnimId then return end
		local hum = char and char:FindFirstChildOfClass("Humanoid")
		local animator = hum and hum:FindFirstChildOfClass("Animator")
		if not animator then return end
		local anim = Instance.new("Animation")
		anim.AnimationId = "rbxassetid://" .. tostring(cfg.SprintAnimId)
		local track = animator:LoadAnimation(anim)
		anim:Destroy()
		track.Priority = Enum.AnimationPriority.Action4
		track.Looped   = true
		track:Play()
		sprintTrack = track
	end

	local function setSprint(active)
		isSprinting = active
		local hum = char and char:FindFirstChildOfClass("Humanoid")
		if hum then hum.WalkSpeed = active and SPRINT_SPD or NORMAL_SPD end
		if active then startSprintAnim() else stopSprintAnim() end
		SprintEvent:FireServer(active)
	end

	local mediThread = nil
	local function startMediCharge(cfg)
		if mediThread then task.cancel(mediThread) mediThread = nil end
		mediThread = task.spawn(function()
			while equippedGun and cfgName(equippedGun.Name) == "Medi-Gun" do
				task.wait(cfg.ReloadTime)
				if not equippedGun then break end
				if mouseHeld then continue end
				if currentAmmo < cfg.AmmoInMag then
					currentAmmo += 1
					ammoState["Medi-Gun"] = currentAmmo
					updateAmmoGui()
				end
			end
		end)
	end

	local function stopMediCharge()
		if mediThread then task.cancel(mediThread) mediThread = nil end
	end

	local function mediHeal(cfg)
		if not isAlive or not canShoot or not equippedGun or currentAmmo <= 0 then return end
		local target = findHealTarget()
		if not target then return end
		local tc = target.Character; if not tc then return end
		local gn      = cfgName(equippedGun.Name)
		local gunMdl  = char:FindFirstChild(gn .. "_Equipped")
		local muzzle  = gunMdl and gunMdl:FindFirstChild("Muzzle", true)
		local muzzAtt = muzzle and muzzle:FindFirstChild("MuzzleAttachment")
		if muzzAtt then showHealBeam(muzzAtt, tc) end
		if muzzle then
			playFireSound(muzzle)
			for _, pe in ipairs(muzzle:GetDescendants()) do
				if pe:IsA("ParticleEmitter") then pe:Emit(pe:GetAttribute("EmitCount") or 5) end
			end
		end
		local torso = tc:FindFirstChild("Torso") or tc:FindFirstChild("UpperTorso")
		if torso then showHitOutline(torso, Color3.fromRGB(0, 255, 100)) end
		local th = tc:FindFirstChildOfClass("Humanoid")
		MediHealEvent:FireServer(th)
		currentAmmo -= 1
		ammoState[gn] = currentAmmo
		updateAmmoGui()
		canShoot = false
		task.delay(cfg.FireRate, function() canShoot = true end)
	end

	local reload

	local function shoot(cfg)
		if not isAlive or not canShoot or isReloading or not equippedGun or currentAmmo <= 0 then return end
		local gn = cfgName(equippedGun.Name)
		if isMedi(gn) then mediHeal(cfg) return end
		if isSprinting then setSprint(false) end
		local head = char:FindFirstChild("Head"); if not head then return end
		local origin = head.Position
		local dir    = (mouse.Hit.Position - origin).Unit * 1000
		if cfg.Spread and cfg.SpreadStartDist then
			local dist = (mouse.Hit.Position - origin).Magnitude
			if dist > cfg.SpreadStartDist then
				local angle = math.rad(cfg.Spread)
				local rx = (math.random() * 2 - 1) * angle
				local ry = (math.random() * 2 - 1) * angle
				local cf = CFrame.new(Vector3.zero, dir) * CFrame.Angles(rx, ry, 0)
				dir = cf.LookVector * 1000
			end
		end
		local rp = RaycastParams.new()
		rp.FilterType = Enum.RaycastFilterType.Exclude
		rp.FilterDescendantsInstances = {char}
		local res    = Workspace:Raycast(origin, dir, rp)
		local hitPos = res and res.Position or (origin + dir)
		local hitInst = res and res.Instance
		local gunMdl  = char:FindFirstChild(gn .. "_Equipped")
		local muzzle  = gunMdl and gunMdl:FindFirstChild("Muzzle", true)
		local muzzAtt = muzzle and muzzle:FindFirstChild("MuzzleAttachment")
		local teamName = plr.Team and plr.Team.Name
		if muzzAtt then
			fireTracer(muzzAtt, hitPos, teamName)
			playFireSound(muzzle)
			for _, pe in ipairs(muzzle:GetDescendants()) do
				if pe:IsA("ParticleEmitter") then pe:Emit(pe:GetAttribute("EmitCount") or 5) end
			end
			playFireAnim()
			GunEffectsEvent:FireServer(muzzAtt.WorldPosition, hitPos, teamName)
		end
		if hitInst then
			local hitMdl = hitInst.Parent
			local hum    = hitMdl:FindFirstChildOfClass("Humanoid")
				or (hitMdl.Parent and hitMdl.Parent:FindFirstChildOfClass("Humanoid"))
			if hum and hum.Parent ~= char and hum.Health > 0 then
				local isHS      = hitInst.Name == "Head"
				local hitPlr    = Players:GetPlayerFromCharacter(hum.Parent)
				local myTeam    = plr.Team and plr.Team.Name
				local theirTeam = hitPlr and hitPlr.Team and hitPlr.Team.Name
				if not hitPlr or TeamConfig.canDamage(myTeam, theirTeam) then
					showHitOutline(hitInst)
					if hitSoundsEnabled.Value then hitSound:Play() end
					showHitMarker()
				end
				GunShootEvent:FireServer(hum, gn, isHS)
			end
		end
		currentAmmo -= 1
		ammoState[gn] = currentAmmo
		updateAmmoGui()
		if currentAmmo <= 0 then
			task.delay(0.1, function() if equippedGun then reload(cfg) end end)
		end
		canShoot = false
		task.delay(cfg.FireRate, function() canShoot = true end)
	end

	reload = function(cfg)
		if isReloading then return end
		if currentAmmo == cfg.AmmoInMag then return end
		local gn = equippedGun and cfgName(equippedGun.Name)
		if gn and isMedi(gn) then return end
		isReloading = true
		canShoot    = false
		local interval = cfg.ReloadTime / cfg.AmmoInMag
		task.spawn(function()
			while isReloading and equippedGun and currentAmmo < cfg.AmmoInMag do
				task.wait(interval)
				if not isReloading or not equippedGun then break end
				currentAmmo += 1
				if gn then ammoState[gn] = currentAmmo end
				updateAmmoGui()
			end
			isReloading = false
			canShoot    = true
		end)
	end

	local function onEquip(tool)
		local cfg = GunConfig[cfgName(tool.Name)]
		if not cfg then return end
		equippedGun = tool
		local gn    = cfgName(tool.Name)
		currentAmmo = ammoState[gn] ~= nil and ammoState[gn] or cfg.AmmoInMag
		isReloading = false
		canShoot    = true
		showGunGui(cfg, gn)
		updateAmmoGui()
		if isMedi(gn) then startMediCharge(cfg) end
		task.defer(loadFireAnim)
		if isSprinting then startSprintAnim() end
	end

	local function onUnequip()
		stopMediCharge()
		for tc, _ in pairs(healBeams) do clearHealBeam(tc) end
		isReloading = false
		canShoot    = true
		if equippedGun then
			ammoState[cfgName(equippedGun.Name)] = currentAmmo
		end
		equippedGun = nil
		if fireTrack then fireTrack:Stop() fireTrack = nil end
		stopSprintAnim()
		hideGunGui()
	end

	local function hookTools(c)
		for _, tool in ipairs(c:GetChildren()) do
			if tool:IsA("Tool") then
				tool.Equipped:Connect(function() onEquip(tool) end)
				tool.Unequipped:Connect(onUnequip)
			end
		end
		c.ChildAdded:Connect(function(obj)
			if obj:IsA("Tool") then
				obj.Equipped:Connect(function() onEquip(obj) end)
				obj.Unequipped:Connect(onUnequip)
			end
		end)
	end
	hookTools(char)

	local function onDied()
		isAlive     = false
		isReloading = false
		canShoot    = false
		stopSprintAnim()
		isSprinting = false
		hideGunGui()
	end

	local function connectDeath(c)
		local hum = c:FindFirstChildOfClass("Humanoid") or c:WaitForChild("Humanoid", 5)
		if not hum then return end
		hum.Died:Connect(onDied)
		local lastHP = hum.Health
		hum.HealthChanged:Connect(function(hp)
			if hp < lastHP and damageTakenSoundsEnabled.Value then damageTakenSound:Play() end
			lastHP = hp
		end)
	end
	connectDeath(char)

	local wasMoving = false
	RunService.Heartbeat:Connect(function()
		if not isCrouching or not crouchTrack then return end
		local hum = char and char:FindFirstChildOfClass("Humanoid"); if not hum then return end
		local moving = hum.MoveDirection.Magnitude > 0.1
		if moving ~= wasMoving then
			wasMoving = moving
			crouchTrack:AdjustSpeed(moving and 1 or 0)
		end
	end)

	local function connectJump(c)
		local hum = c:FindFirstChildOfClass("Humanoid") or c:WaitForChild("Humanoid", 5)
		if hum then
			hum.Jumping:Connect(function(active)
				if active and isCrouching then setCrouch(false) end
			end)
		end
	end
	connectJump(char)

	local function preloadAnims(c)
		local hum      = c:FindFirstChildOfClass("Humanoid")
		local animator = hum and hum:FindFirstChildOfClass("Animator")
		if not animator or not crouchAnimId then return end
		local anim = Instance.new("Animation")
		anim.AnimationId = "rbxassetid://" .. tostring(crouchAnimId)
		animator:LoadAnimation(anim):Stop()
		anim:Destroy()
	end
	task.defer(function() preloadAnims(char) end)

	plr.CharacterAdded:Connect(function(c)
		task.defer(function() preloadAnims(c) end)
		char        = c
		isAlive     = true
		equippedGun = nil
		isReloading = false
		canShoot    = true
		isSprinting = false
		isCrouching = false
		crouchTrack = nil
		fireTrack   = nil
		wasMoving   = nil
		ammoState   = {}
		local hum = c:FindFirstChildOfClass("Humanoid") or c:WaitForChild("Humanoid", 5)
		if hum then hum.WalkSpeed = NORMAL_SPD end
		task.defer(refreshGuiRefs)
		hideGunGui()
		hookTools(c)
		connectDeath(c)
		connectJump(c)
	end)

	local armThrottle = 0
	RunService.Heartbeat:Connect(function(dt)
		if not char then return end
		local torso = char:FindFirstChild("Torso"); if not torso then return end
		local rs = torso:FindFirstChild("Right Shoulder")
		local ls = torso:FindFirstChild("Left Shoulder")
		armThrottle += dt
		if armThrottle >= 0.1 and equippedGun and not isMedi(cfgName(equippedGun.Name)) then
			armThrottle = 0
			if rs and ls then GunArmEvent:FireServer(rs.C0, ls.C0) end
		end
	end)

	UIS.InputBegan:Connect(function(input, gpe)
		if gpe then return end
		if input.KeyCode == Enum.KeyCode.R then
			if not equippedGun then return end
			local cfg = GunConfig[cfgName(equippedGun.Name)]
			if cfg then reload(cfg) end
			return
		end
		if input.KeyCode == Enum.KeyCode.F then
			local gn  = equippedGun and cfgName(equippedGun.Name)
			local new = not isSprinting
			if new then
				if not (gn and isMedi(gn)) then
					mouseHeld = false
					if autoLoop then task.cancel(autoLoop) autoLoop = nil end
				end
				if isCrouching then setCrouch(false) end
			end
			setSprint(new)
			return
		end
		if input.KeyCode == Enum.KeyCode.C then
			if isSprinting then setSprint(false) end
			setCrouch(not isCrouching)
		end
	end)

	UIS.InputBegan:Connect(function(input, gpe)
		if gpe then return end
		if input.UserInputType ~= Enum.UserInputType.MouseButton1 then return end
		if not equippedGun then return end
		local cfg = GunConfig[cfgName(equippedGun.Name)]; if not cfg then return end
		local gn  = cfgName(equippedGun.Name)
		mouseHeld = true
		shoot(cfg)
		if cfg.FireMode == "Auto" or isMedi(gn) then
			autoLoop = task.spawn(function()
				while mouseHeld and equippedGun do
					task.wait(cfg.FireRate)
					if mouseHeld and equippedGun then shoot(cfg) end
				end
			end)
		end
	end)

	UIS.InputEnded:Connect(function(input)
		if input.UserInputType ~= Enum.UserInputType.MouseButton1 then return end
		mouseHeld = false
	end)

	MediHealEffectEvent.OnClientEvent:Connect(function(healerPlr, targetChar)
		if healerPlr == plr then return end
		local hc = healerPlr.Character; if not hc then return end
		local gunMdl = nil
		for _, obj in ipairs(hc:GetChildren()) do
			if obj:IsA("Model") and obj.Name == "Medi-Gun_Equipped" then gunMdl = obj break end
		end
		local muzzle  = gunMdl and gunMdl:FindFirstChild("Muzzle", true)
		local muzzAtt = muzzle and muzzle:FindFirstChild("MuzzleAttachment")
		if not muzzAtt or not targetChar then return end
		playFireSound(muzzle)
		for _, pe in ipairs(muzzle:GetDescendants()) do
			if pe:IsA("ParticleEmitter") then pe:Emit(pe:GetAttribute("EmitCount") or 5) end
		end
		local torso = targetChar:FindFirstChild("Torso") or targetChar:FindFirstChild("UpperTorso")
		if not torso then return end
		local a1 = Instance.new("Attachment"); a1.Parent = torso
		local beam = beamTemplates.Heal:Clone()
		beam.Attachment0 = muzzAtt
		beam.Attachment1 = a1
		beam.Enabled = true
		beam.Parent  = Workspace.Terrain
		task.delay(MEDI_TLIFE, function()
			if beam.Parent then beam:Destroy() end
			if a1.Parent   then a1:Destroy()   end
		end)
	end)

	GunEffectsEvent.OnClientEvent:Connect(function(shooterPlr, muzzlePos, hitPos, teamName)
		if shooterPlr == plr then return end
		local oc       = shooterPlr.Character
		local muzzAtt  = nil
		if oc then
			for _, obj in ipairs(oc:GetChildren()) do
				if obj:IsA("Model") and obj.Name:find("_Equipped") then
					local mp = obj:FindFirstChild("Muzzle", true)
					if mp then muzzAtt = mp:FindFirstChild("MuzzleAttachment") end
					if muzzAtt then break end
				end
			end
		end
		local a1 = Instance.new("Attachment")
		a1.Parent = Workspace.Terrain
		a1.WorldPosition = hitPos
		local beam = getBeam(teamName):Clone()
		if muzzAtt then
			beam.Attachment0 = muzzAtt
		else
			local a0 = Instance.new("Attachment")
			a0.Parent = Workspace.Terrain
			a0.WorldPosition = muzzlePos
			beam.Attachment0 = a0
			task.delay(TRACER_LIFE, function() a0:Destroy() end)
		end
		beam.Attachment1 = a1
		beam.Enabled = true
		beam.Parent  = Workspace.Terrain
		task.delay(TRACER_LIFE, function() beam:Destroy() a1:Destroy() end)
		if oc then
			local muzzPart = nil
			for _, obj in ipairs(oc:GetChildren()) do
				if obj:IsA("Model") and obj.Name:find("_Equipped") then
					muzzPart = obj:FindFirstChild("Muzzle", true)
					if muzzPart then break end
				end
			end
			if muzzPart then
				playFireSound(muzzPart)
				for _, pe in ipairs(muzzPart:GetDescendants()) do
					if pe:IsA("ParticleEmitter") then pe:Emit(pe:GetAttribute("EmitCount") or 5) end
				end
			end
		end
	end)

	GunArmEvent.OnClientEvent:Connect(function(shooterPlr, rsC0, lsC0)
		if shooterPlr == plr then return end
		local oc = shooterPlr.Character; if not oc then return end
		local ot = oc:FindFirstChild("Torso"); if not ot then return end
		local rs = ot:FindFirstChild("Right Shoulder")
		local ls = ot:FindFirstChild("Left Shoulder")
		local hasGun = false
		for _, obj in ipairs(oc:GetChildren()) do
			if obj:IsA("Model") and obj.Name:find("_Equipped") then hasGun = true break end
		end
		if not hasGun then return end
		if rs then rs.C0 = rsC0 end
		if ls then ls.C0 = lsC0 end
	end)

	-- damage indicators
	local dmgEnabledVal = RS:WaitForChild("Values"):WaitForChild("DamageIndicatorsEnabled")
	local dmgStyleVal   = RS:WaitForChild("Values"):WaitForChild("DamageIndicatorStyle")
	local dmgSizeVal    = RS:WaitForChild("Values"):WaitForChild("DamageIndicatorSize")

	local DI_MODE = dmgStyleVal.Value:lower()
	dmgStyleVal.Changed:Connect(function(v) DI_MODE = v:lower() end)

	local pg2       = plr:WaitForChild("PlayerGui")
	local screenGui = pg2:WaitForChild("DamageIndicatorGui")
	local lbl       = screenGui:WaitForChild("DmgLabel")

	local BASE_SIZE = dmgSizeVal.Value
	dmgSizeVal.Changed:Connect(function(v)
		BASE_SIZE = v
		if not lbl.Visible then lbl.TextSize = v end
	end)

	local STACK_DELAY  = 1.5
	local STACK_OFFSET = Vector2.new(0, 10)
	local FLOAT_SX     = 40
	local FLOAT_SY     = 20
	local FLOAT_RISE   = 55
	local FLOAT_DUR    = 0.9
	local FLOAT_FONT   = lbl.Font
	local FLOAT_COL    = lbl.TextColor3
	local FLOAT_STROKE = lbl.TextStrokeColor3

	local accumulated = 0
	local resetTimer  = nil
	local diVisible   = false
	local activeTw    = nil

	local function diUpdatePos()
		local mp = UIS:GetMouseLocation()
		lbl.Position = UDim2.new(0, mp.X + STACK_OFFSET.X, 0, mp.Y + STACK_OFFSET.Y)
	end

	RunService.RenderStepped:Connect(function()
		if DI_MODE == "stack" and lbl.Visible then diUpdatePos() end
	end)

	local function showLabel()
		if diVisible then return end
		if activeTw then activeTw:Cancel() end
		diVisible = true
		lbl.TextSize = 0
		lbl.Visible  = true
		activeTw = TweenService:Create(lbl, TweenInfo.new(0.15, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {TextSize = BASE_SIZE})
		activeTw:Play()
	end

	local function hideLabel()
		if not diVisible then return end
		diVisible = false
		if activeTw then activeTw:Cancel() end
		local ht = TweenService:Create(lbl, TweenInfo.new(0.12, Enum.EasingStyle.Quad, Enum.EasingDirection.In), {TextSize = 0})
		activeTw  = ht
		ht:Play()
		ht.Completed:Connect(function()
			if activeTw == ht then
				lbl.Visible  = false
				lbl.TextSize = BASE_SIZE
				activeTw     = nil
			end
		end)
	end

	local function spawnFloat(dmg)
		local mp  = UIS:GetMouseLocation()
		local rx  = math.random(-FLOAT_SX, FLOAT_SX)
		local ry  = math.random(-FLOAT_SY, FLOAT_SY)
		local sx  = mp.X + rx
		local sy  = mp.Y + ry
		local fl  = Instance.new("TextLabel")
		fl.Name   = "FloatDmg"
		fl.AnchorPoint           = Vector2.new(0.5, 0.5)
		fl.BackgroundTransparency = 1
		fl.Font                  = FLOAT_FONT
		fl.TextColor3            = FLOAT_COL
		fl.TextStrokeColor3      = FLOAT_STROKE
		fl.TextStrokeTransparency = 0
		fl.TextSize  = 0
		fl.Text      = tostring(dmg)
		fl.Position  = UDim2.new(0, sx, 0, sy)
		fl.ZIndex    = 10
		fl.Parent    = screenGui
		local popIn = TweenService:Create(fl, TweenInfo.new(0.12, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {TextSize = BASE_SIZE})
		popIn:Play()
		task.delay(0.2, function()
			if not fl or not fl.Parent then return end
			local rise = TweenService:Create(fl, TweenInfo.new(FLOAT_DUR, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
				Position             = UDim2.new(0, sx, 0, sy - FLOAT_RISE),
				TextTransparency     = 1,
				TextStrokeTransparency = 1,
				TextSize             = BASE_SIZE * 0.7,
			})
			rise:Play()
			rise.Completed:Connect(function() fl:Destroy() end)
		end)
	end

	DamageIndicatorEvent.OnClientEvent:Connect(function(dmg)
		if not dmgEnabledVal.Value then return end
		if DI_MODE == "float" then
			spawnFloat(dmg)
		else
			accumulated += dmg
			lbl.Text = tostring(accumulated)
			diUpdatePos()
			showLabel()
			if resetTimer then resetTimer:Disconnect() resetTimer = nil end
			local t0 = tick()
			resetTimer = RunService.Heartbeat:Connect(function()
				if tick() - t0 >= STACK_DELAY then
					accumulated = 0
					resetTimer:Disconnect()
					resetTimer = nil
					hideLabel()
				end
			end)
		end
	end)

	KillEvent.OnClientEvent:Connect(function()
		if enemyKilledSoundsEnabled.Value then killSound:Play() end
	end)

	hideGunGui()
end

function GunModule.initServer()
	local MAX_RANGE  = 1000
	local sprintActive = {}
	local killWatched  = {}

	CrouchEvent.OnServerEvent:Connect(function(plr, active) end)

	SprintEvent.OnServerEvent:Connect(function(plr, active)
		sprintActive[plr] = active
	end)

	Players.PlayerRemoving:Connect(function(plr)
		sprintActive[plr] = nil
	end)

	local function onCharAdded(plr, c)
		sprintActive[plr] = false
		local hum = c:WaitForChild("Humanoid", 5)
		if hum then hum.Died:Connect(function() sprintActive[plr] = false end) end
	end

	Players.PlayerAdded:Connect(function(plr)
		plr.CharacterAdded:Connect(function(c) onCharAdded(plr, c) end)
		if plr.Character then onCharAdded(plr, plr.Character) end
	end)
	for _, plr in ipairs(Players:GetPlayers()) do
		if plr.Character then onCharAdded(plr, plr.Character) end
		plr.CharacterAdded:Connect(function(c) onCharAdded(plr, c) end)
	end

	GunEffectsEvent.OnServerEvent:Connect(function(plr, muzzlePos, hitPos, teamName)
		GunEffectsEvent:FireAllClients(plr, muzzlePos, hitPos, teamName)
	end)

	GunArmEvent.OnServerEvent:Connect(function(plr, rsC0, lsC0)
		GunArmEvent:FireAllClients(plr, rsC0, lsC0)
	end)

	GunShootEvent.OnServerEvent:Connect(function(plr, humanoid, gunName, isHeadshot)
		local cfg = GunConfig[gunName]; if not cfg then return end
		if not humanoid or not humanoid:IsA("Humanoid") then return end
		if humanoid.Health <= 0 then return end
		local c = plr.Character; if not c then return end
		if humanoid.Parent == c then return end

		local shooterTeam = plr.Team and plr.Team.Name
		local targetPlr   = Players:GetPlayerFromCharacter(humanoid.Parent)
		local targetTeam  = targetPlr and targetPlr.Team and targetPlr.Team.Name
		if targetPlr and not TeamConfig.canDamage(shooterTeam, targetTeam) then return end

		local head = c:FindFirstChild("Head"); if not head then return end
		local tr   = humanoid.Parent:FindFirstChild("HumanoidRootPart")
			or humanoid.Parent:FindFirstChild("Torso")
			or humanoid.Parent:FindFirstChild("UpperTorso")
		if not tr then return end
		local dir = tr.Position - head.Position
		if dir.Magnitude > MAX_RANGE then return end
		local rp = RaycastParams.new()
		rp.FilterType = Enum.RaycastFilterType.Exclude
		rp.FilterDescendantsInstances = {c}
		local res = Workspace:Raycast(head.Position, dir, rp)
		if res then
			local hm = res.Instance.Parent
			if hm ~= humanoid.Parent and hm.Parent ~= humanoid.Parent then return end
		end
		local dmg   = isHeadshot and (cfg.HeadDamage or cfg.Damage) or cfg.Damage
		local prevHP = humanoid.Health
		humanoid:TakeDamage(dmg)
		local actual = math.floor(math.max(0, prevHP - humanoid.Health))
		DamageIndicatorEvent:FireClient(plr, actual > 0 and actual or dmg)
		local ls = plr:FindFirstChild("leaderstats")
		if ls then
			if ls:FindFirstChild("Damage") then
				ls.Damage.Value += actual > 0 and actual or dmg
			end
			if not killWatched[humanoid] then
				killWatched[humanoid] = true
				humanoid.Died:Once(function()
					killWatched[humanoid] = nil
					if ls:FindFirstChild("Kills") then ls.Kills.Value += 1 end
					KillEvent:FireClient(plr)
					local kn     = plr.Name
					local kt     = plr.Team and plr.Team.Name or nil
					local khp    = math.floor(plr.Character and plr.Character:FindFirstChildOfClass("Humanoid") and plr.Character:FindFirstChildOfClass("Humanoid").Health or 0)
					local vp     = Players:GetPlayerFromCharacter(humanoid.Parent)
					local vn     = vp and vp.Name or (humanoid.Parent and humanoid.Parent.Name or "NPC")
					local vt     = vp and vp.Team and vp.Team.Name or nil
					KillFeedEvent:FireAllClients(kn, kt, khp, gunName, vn, vt)
				end)
			end
		end
	end)

	local MEDI_SV_RANGE = 8.5
	MediHealEvent.OnServerEvent:Connect(function(plr, targetHum)
		if not targetHum or not targetHum:IsA("Humanoid") then return end
		if targetHum.Health <= 0 or targetHum.Health >= targetHum.MaxHealth then return end
		local c = plr.Character; if not c then return end
		local tp = Players:GetPlayerFromCharacter(targetHum.Parent); if not tp then return end
		if not TeamConfig.canHeal(plr.Team and plr.Team.Name, tp.Team and tp.Team.Name) then return end
		if targetHum.Parent == c then return end
		local hr = c:FindFirstChild("HumanoidRootPart") or c:FindFirstChild("Torso")
		local tr = targetHum.Parent:FindFirstChild("HumanoidRootPart") or targetHum.Parent:FindFirstChild("Torso")
		if not hr or not tr then return end
		if (tr.Position - hr.Position).Magnitude > MEDI_SV_RANGE then return end
		local rp = RaycastParams.new()
		rp.FilterType = Enum.RaycastFilterType.Exclude
		rp.FilterDescendantsInstances = {c, targetHum.Parent}
		if Workspace:Raycast(hr.Position, tr.Position - hr.Position, rp) then return end
		local cfg    = GunConfig["Medi-Gun"]
		local healAmt = math.abs(cfg.Damage)
		local prevHP  = targetHum.Health
		targetHum.Health = math.min(prevHP + healAmt, targetHum.MaxHealth)
		local got = math.floor(targetHum.Health - prevHP)
		if got > 0 then
			local ls = plr:FindFirstChild("leaderstats")
			if ls and ls:FindFirstChild("Heals") then ls.Heals.Value += got end
		end
		MediHealEffectEvent:FireAllClients(plr, targetHum.Parent)
	end)
end

return GunModule
