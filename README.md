--[[
 ██████╗  ███████╗ █████╗  ██╗     ██╗██████╗  █████╗ ██████╗  ██╗████████╗
 ██╔══██╗ ██╔════╝██╔══██╗ ██║     ██║██╔══██╗██╔══██╗██╔══██╗ ██║╚══██╔══╝
 ██████╔╝ █████╗  ███████║ ██║     ██║██████╔╝███████║██████╔╝ ██║   ██║   
 ╚═╝  ╚═╝ ╚══════╝╚═╝  ╚═╝ ╚══════╝╚═╝╚═╝     ╚═╝  ╚═╝╚═╝  ╚═╝ ╚═╝   ╚═╝   
 Powered by 235B!t Hub X & MAC SHOP V7 • ULTIMATE REALISTIC DRONE & 10 CAM MODES
--]]

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")
local SoundService = game:GetService("SoundService")

local LocalPlayer = Players.LocalPlayer
if not LocalPlayer then return end

local function GetCharacter()
	return LocalPlayer.Character or LocalPlayer.CharacterAdded:Wait()
end

-- ================= SOUND EFFECTS =================
local SoundClick = Instance.new("Sound")
SoundClick.SoundId = "rbxassetid://6895079853"
SoundClick.Volume = 0.8
SoundClick.Parent = SoundService

local function PlayClickSound()
	task.spawn(function() SoundClick:Play() end)
end

-- ================= CLEANUP OLD GUI & DRONE =================
local PlayerGui = LocalPlayer:WaitForChild("PlayerGui")
if PlayerGui:FindFirstChild("MacShop_FPV_Custom") then
	PlayerGui["MacShop_FPV_Custom"]:Destroy()
end
if workspace:FindFirstChild("MAC_Custom_Drone") then
	workspace["MAC_Custom_Drone"]:Destroy()
end

-- ================= DRONE STATE & DATA =================
local DroneState = {
	Active = false,
	IsLanding = false,
	Model = nil,
	BaseSpeed = 35,
	CurrentSpeed = 35,
	Turbo = false,
	Battery = 100,
	Height = 0,
	SpeedCalc = 0,
	FrontLightOn = false,
	SirenOn = false,
	CameraMode = 1, -- 1 ถึง 10 โหมดกล้อง
	Position = Vector3.zero,
	Velocity = Vector3.zero, -- สำหรับฟิสิกส์การบิน
	Yaw = 0,
	YawVelocity = 0,
	Pitch = 0,
	Roll = 0,
	Spin = 0,
	FrontLights = {},
	SirenBarLights = {},
	Propellers = {},
	RopeActive = false,
	RopeConstraint = nil,
	RopeLength = 20,
	AttachmentDrone = nil,
	AttachmentPlayer = nil,
	TargetPlayer = nil,
	AutoFollow = false,
	OrbitAngle = 0
}

local UIInputs = {
	Forward = false, Backward = false, Left = false, Right = false,
	Up = false, Down = false, YawLeft = false, YawRight = false
}

-- รายชื่อโหมดกล้องทั้ง 10 โหมด
local CameraModes = {
	[1] = "1. FPV มุมมองนักบิน",
	[2] = "2. Third-Person ตามหลัง",
	[3] = "3. Bottom-View ส่องใต้ท้อง",
	[4] = "4. Selfie หันมองผู้เล่น",
	[5] = "5. Cinematic Orbit หมุนรอบตัว",
	[6] = "6. Top-Down Scan มุมมองดาวเทียม",
	[7] = "7. Thermal/Night โหมดกลางคืน",
	[8] = "8. Lock Target ล็อกเป้าหมาย",
	[9] = "9. Ultra Wide มุมกว้างพิเศษ",
	[10] = "10. Side-Chase ขนานข้าง"
}

-- ================= GUI SETUP =================
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "MacShop_FPV_Custom"
ScreenGui.ResetOnSpawn = false
ScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
ScreenGui.Parent = PlayerGui

local function ShowNotification(text)
	local notifFrame = Instance.new("Frame", ScreenGui)
	notifFrame.Size = UDim2.new(0, 260, 0, 32)
	notifFrame.Position = UDim2.new(0.5, -130, 0, 130)
	notifFrame.BackgroundColor3 = Color3.fromRGB(20, 20, 30)
	notifFrame.BackgroundTransparency = 0.1
	notifFrame.BorderSizePixel = 0

	local corner = Instance.new("UICorner", notifFrame)
	corner.CornerRadius = UDim.new(0, 8)

	local stroke = Instance.new("UIStroke", notifFrame)
	stroke.Color = Color3.fromRGB(0, 217, 255)
	stroke.Thickness = 1.5

	local label = Instance.new("TextLabel", notifFrame)
	label.Size = UDim2.new(1, 0, 1, 0)
	label.BackgroundTransparency = 1
	label.Text = text
	label.TextColor3 = Color3.fromRGB(255, 255, 255)
	label.TextSize = 11
	label.Font = Enum.Font.GothamBold

	TweenService:Create(notifFrame, TweenInfo.new(0.25), {Position = UDim2.new(0.5, -130, 0, 110)}):Play()

	task.delay(1.5, function()
		local fade = TweenService:Create(notifFrame, TweenInfo.new(0.3), {BackgroundTransparency = 1})
		TweenService:Create(label, TweenInfo.new(0.3), {TextTransparency = 1}):Play()
		TweenService:Create(stroke, TweenInfo.new(0.3), {Transparency = 1}):Play()
		fade:Play()
		fade.Completed:Connect(function() notifFrame:Destroy() end)
	end)
end

-- MAIN WINDOW
local MainWindow = Instance.new("Frame", ScreenGui)
MainWindow.Name = "MainWindow"
MainWindow.Size = UDim2.new(0, 450, 0, 160)
MainWindow.Position = UDim2.new(0.5, -225, 0, 10)
MainWindow.BackgroundColor3 = Color3.fromRGB(15, 15, 22)
MainWindow.BackgroundTransparency = 0.15
MainWindow.BorderSizePixel = 0

local WindowCorner = Instance.new("UICorner", MainWindow)
WindowCorner.CornerRadius = UDim.new(0, 10)

local WindowStroke = Instance.new("UIStroke", MainWindow)
WindowStroke.Color = Color3.fromRGB(0, 217, 255)
WindowStroke.Thickness = 1.5

local HeaderTitle = Instance.new("TextLabel", MainWindow)
HeaderTitle.Size = UDim2.new(0, 220, 0, 18)
HeaderTitle.Position = UDim2.new(0, 10, 0, 4)
HeaderTitle.BackgroundTransparency = 1
HeaderTitle.Text = "โดรน FPV ไทย V7 • Realistic Edition"
HeaderTitle.TextColor3 = Color3.fromRGB(0, 217, 255)
HeaderTitle.TextSize = 12
HeaderTitle.Font = Enum.Font.GothamBold
HeaderTitle.TextXAlignment = Enum.TextXAlignment.Left

local CreditTitle = Instance.new("TextLabel", MainWindow)
CreditTitle.Size = UDim2.new(0, 250, 0, 14)
CreditTitle.Position = UDim2.new(0, 10, 0, 20)
CreditTitle.BackgroundTransparency = 1
CreditTitle.Text = "Powered by 235B!t Hub X & MAC SHOP"
CreditTitle.TextColor3 = Color3.fromRGB(255, 200, 0)
CreditTitle.TextSize = 10
CreditTitle.Font = Enum.Font.GothamMedium
CreditTitle.TextXAlignment = Enum.TextXAlignment.Left

local WindowControls = Instance.new("Frame", MainWindow)
WindowControls.Size = UDim2.new(0, 55, 0, 22)
WindowControls.Position = UDim2.new(1, -60, 0, 5)
WindowControls.BackgroundTransparency = 1

local MinBtn = Instance.new("TextButton", WindowControls)
MinBtn.Size = UDim2.new(0, 24, 0, 20)
MinBtn.BackgroundColor3 = Color3.fromRGB(45, 45, 60)
MinBtn.Text = "-"
MinBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
MinBtn.Font = Enum.Font.GothamBold
Instance.new("UICorner", MinBtn).CornerRadius = UDim.new(0, 4)

local CloseBtn = Instance.new("TextButton", WindowControls)
CloseBtn.Size = UDim2.new(0, 24, 0, 20)
CloseBtn.Position = UDim2.new(0, 28, 0, 0)
CloseBtn.BackgroundColor3 = Color3.fromRGB(180, 40, 40)
CloseBtn.Text = "X"
CloseBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
CloseBtn.Font = Enum.Font.GothamBold
Instance.new("UICorner", CloseBtn).CornerRadius = UDim.new(0, 4)

-- TAB SYSTEM
local TabBar = Instance.new("Frame", MainWindow)
TabBar.Size = UDim2.new(1, -20, 0, 24)
TabBar.Position = UDim2.new(0, 10, 0, 38)
TabBar.BackgroundTransparency = 1

local function CreateTabBtn(text, pos, active)
	local btn = Instance.new("TextButton", TabBar)
	btn.Size = UDim2.new(0.32, 0, 1, 0)
	btn.Position = pos
	btn.BackgroundColor3 = active and Color3.fromRGB(0, 180, 216) or Color3.fromRGB(35, 35, 50)
	btn.Text = text
	btn.TextColor3 = active and Color3.fromRGB(255, 255, 255) or Color3.fromRGB(200, 200, 200)
	btn.Font = Enum.Font.GothamBold
	btn.TextSize = 10
	Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 4)
	return btn
end

local Tab1Btn = CreateTabBtn("แท็บ 1: บิน/กล้อง 10 โหมด", UDim2.new(0, 0, 0, 0), true)
local Tab2Btn = CreateTabBtn("แท็บ 2: ไฟ/เชือก", UDim2.new(0.34, 0, 0, 0), false)
local Tab3Btn = CreateTabBtn("แท็บ 3: ติดตาม/ข้อมูล", UDim2.new(0.68, 0, 0, 0), false)

local Tab1Frame = Instance.new("Frame", MainWindow)
Tab1Frame.Size = UDim2.new(1, -20, 0, 85)
Tab1Frame.Position = UDim2.new(0, 10, 0, 66)
Tab1Frame.BackgroundTransparency = 1

local Tab2Frame = Instance.new("Frame", MainWindow)
Tab2Frame.Size = UDim2.new(1, -20, 0, 85)
Tab2Frame.Position = UDim2.new(0, 10, 0, 66)
Tab2Frame.BackgroundTransparency = 1
Tab2Frame.Visible = false

local Tab3Frame = Instance.new("Frame", MainWindow)
Tab3Frame.Size = UDim2.new(1, -20, 0, 85)
Tab3Frame.Position = UDim2.new(0, 10, 0, 66)
Tab3Frame.BackgroundTransparency = 1
Tab3Frame.Visible = false

local function CreateGrid(parent)
	local layout = Instance.new("UIGridLayout", parent)
	layout.CellSize = UDim2.new(0, 135, 0, 25)
	layout.CellPadding = UDim2.new(0, 8, 0, 4)
end
CreateGrid(Tab1Frame)
CreateGrid(Tab2Frame)
CreateGrid(Tab3Frame)

local function CreateTabButton(parent, text, bgColor, callback)
	local btn = Instance.new("TextButton", parent)
	btn.BackgroundColor3 = bgColor
	btn.Text = text
	btn.TextColor3 = Color3.fromRGB(255, 255, 255)
	btn.Font = Enum.Font.GothamBold
	btn.TextSize = 10
	Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 5)

	btn.MouseButton1Click:Connect(function()
		PlayClickSound()
		callback(btn)
	end)
	return btn
end

local function SwitchTab(activeTab, activeFrame)
	for _, b in ipairs({Tab1Btn, Tab2Btn, Tab3Btn}) do
		b.BackgroundColor3 = Color3.fromRGB(35, 35, 50)
		b.TextColor3 = Color3.fromRGB(200, 200, 200)
	end
	for _, f in ipairs({Tab1Frame, Tab2Frame, Tab3Frame}) do
		f.Visible = false
	end
	activeTab.BackgroundColor3 = Color3.fromRGB(0, 180, 216)
	activeTab.TextColor3 = Color3.fromRGB(255, 255, 255)
	activeFrame.Visible = true
end

Tab1Btn.MouseButton1Click:Connect(function() PlayClickSound() SwitchTab(Tab1Btn, Tab1Frame) end)
Tab2Btn.MouseButton1Click:Connect(function() PlayClickSound() SwitchTab(Tab2Btn, Tab2Frame) end)
Tab3Btn.MouseButton1Click:Connect(function() PlayClickSound() SwitchTab(Tab3Btn, Tab3Frame) end)

-- ================= TOUCH CONTROLS (MOVEABLE & AUTO-POSITION) =================
local LeftTouchFrame = Instance.new("Frame", ScreenGui)
LeftTouchFrame.Name = "LeftTouchFrame"
LeftTouchFrame.Size = UDim2.new(0, 110, 0, 80)
LeftTouchFrame.Position = UDim2.new(0, -150, 1, -90) -- เริ่มต้นซ่อนนอกจอ
LeftTouchFrame.BackgroundTransparency = 1
LeftTouchFrame.Active = true

local LeftGrid = Instance.new("UIGridLayout", LeftTouchFrame)
LeftGrid.CellSize = UDim2.new(0, 34, 0, 24)
LeftGrid.CellPadding = UDim2.new(0, 3, 0, 3)

local RightTouchFrame = Instance.new("Frame", ScreenGui)
RightTouchFrame.Name = "RightTouchFrame"
RightTouchFrame.Size = UDim2.new(0, 110, 0, 80)
RightTouchFrame.Position = UDim2.new(1, 150, 1, -90) -- เริ่มต้นซ่อนนอกจอ
RightTouchFrame.BackgroundTransparency = 1
RightTouchFrame.Active = true

local RightGrid = Instance.new("UIGridLayout", RightTouchFrame)
RightGrid.CellSize = UDim2.new(0, 34, 0, 24)
RightGrid.CellPadding = UDim2.new(0, 3, 0, 3)

-- ฟังก์ชันทำให้ปุ่มลากย้ายตำแหน่งได้ (Draggable)
local function MakeFrameDraggable(frame)
	local dragging, dragInput, dragStart, startPos
	frame.InputBegan:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
			dragging = true
			dragStart = input.Position
			startPos = frame.Position
			input.Changed:Connect(function()
				if input.UserInputState == Enum.UserInputState.End then dragging = false end
			end)
		end
	end)
	frame.InputChanged:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
			dragInput = input
		end
	end)
	UserInputService.InputChanged:Connect(function(input)
		if input == dragInput and dragging then
			local delta = input.Position - dragStart
			frame.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
		end
	end)
end

MakeFrameDraggable(LeftTouchFrame)
MakeFrameDraggable(RightTouchFrame)

-- ฟังก์ชันเลื่อนปุ่มเข้า-ออก อัตโนมัติ (Auto Position)
local function SetTouchControlsVisible(visible)
	local targetLeftPos = visible and UDim2.new(0, 15, 1, -90) or UDim2.new(0, -150, 1, -90)
	local targetRightPos = visible and UDim2.new(1, -125, 1, -90) or UDim2.new(1, 150, 1, -90)

	TweenService:Create(LeftTouchFrame, TweenInfo.new(0.4, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {Position = targetLeftPos}):Play()
	TweenService:Create(RightTouchFrame, TweenInfo.new(0.4, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {Position = targetRightPos}):Play()
end

local function MakeGridTouchBtn(parent, name, text, key)
	local b = Instance.new("TextButton", parent)
	b.Name = name
	b.BackgroundColor3 = Color3.fromRGB(20, 20, 30)
	b.BackgroundTransparency = 0.35
	b.Text = text
	b.TextColor3 = Color3.fromRGB(255, 255, 255)
	b.Font = Enum.Font.GothamBold
	b.TextSize = 11

	Instance.new("UICorner", b).CornerRadius = UDim.new(0, 4)
	local s = Instance.new("UIStroke", b)
	s.Color = Color3.fromRGB(0, 217, 255)
	s.Thickness = 1

	b.MouseButton1Down:Connect(function() PlayClickSound() UIInputs[key] = true end)
	b.MouseButton1Up:Connect(function() UIInputs[key] = false end)
	b.MouseLeave:Connect(function() UIInputs[key] = false end)
end

Instance.new("Frame", LeftTouchFrame).BackgroundTransparency = 1
MakeGridTouchBtn(LeftTouchFrame, "Btn_Fwd", "▲", "Forward")
Instance.new("Frame", LeftTouchFrame).BackgroundTransparency = 1
MakeGridTouchBtn(LeftTouchFrame, "Btn_Left", "◄", "Left")
MakeGridTouchBtn(LeftTouchFrame, "Btn_Back", "▼", "Backward")
MakeGridTouchBtn(LeftTouchFrame, "Btn_Right", "►", "Right")

MakeGridTouchBtn(RightTouchFrame, "Btn_YawL", "↺", "YawLeft")
MakeGridTouchBtn(RightTouchFrame, "Btn_Up", "ขึ้น", "Up")
MakeGridTouchBtn(RightTouchFrame, "Btn_YawR", "↻", "YawRight")
Instance.new("Frame", RightTouchFrame).BackgroundTransparency = 1
MakeGridTouchBtn(RightTouchFrame, "Btn_Down", "ลง", "Down")
Instance.new("Frame", RightTouchFrame).BackgroundTransparency = 1

local function DestroyRope()
	if DroneState.RopeConstraint then DroneState.RopeConstraint:Destroy() DroneState.RopeConstraint = nil end
	if DroneState.AttachmentDrone then DroneState.AttachmentDrone:Destroy() DroneState.AttachmentDrone = nil end
	if DroneState.AttachmentPlayer then DroneState.AttachmentPlayer:Destroy() DroneState.AttachmentPlayer = nil end
	DroneState.RopeActive = false
end

local function LandAndStop(btn)
	if DroneState.IsLanding then return end
	DroneState.IsLanding = true
	ShowNotification("กำลังลงจอดอัตโนมัติ... 🛬")
	
	task.spawn(function()
		while DroneState.Active and DroneState.Model and DroneState.Position.Y > 3 do
			UIInputs.Down = true
			task.wait(0.1)
		end
		UIInputs.Down = false
		DroneState.Active = false
		DroneState.IsLanding = false
		
		if btn then
			btn.Text = "ขึ้นบิน"
			btn.BackgroundColor3 = Color3.fromRGB(46, 125, 50)
		end
		
		SetTouchControlsVisible(false) -- ซ่อนปุ่มอัตโนมัติ
		DestroyRope()
		if DroneState.Model then DroneState.Model:Destroy() DroneState.Model = nil end
		workspace.CurrentCamera.CameraType = Enum.CameraType.Custom
		ShowNotification("ปิดการบินและลงจอดเรียบร้อย 🛑")
	end)
end

-- ================= TAB 1 BUTTONS =================
local FlyBtn = CreateTabButton(Tab1Frame, "ขึ้นบิน", Color3.fromRGB(46, 125, 50), function(btn)
	if not DroneState.Active then
		DroneState.Active = true
		btn.Text = "ปิดการบิน"
		btn.BackgroundColor3 = Color3.fromRGB(180, 40, 40)
		SetTouchControlsVisible(true) -- เด้งปุ่มขึ้นมาอัตโนมัติ
		ShowNotification("เปิดใช้งานโดรนแล้ว 🚁")
	else
		LandAndStop(btn)
	end
end)

CreateTabButton(Tab1Frame, "ลงจอด", Color3.fromRGB(180, 50, 50), function()
	LandAndStop(FlyBtn)
end)

CreateTabButton(Tab1Frame, "กลับฐาน", Color3.fromRGB(200, 140, 30), function()
	local char = GetCharacter()
	if char and char:FindFirstChild("HumanoidRootPart") then
		DroneState.Position = char.HumanoidRootPart.Position + Vector3.new(0, 8, 0)
		DroneState.Velocity = Vector3.zero
		ShowNotification("วาร์ปโดรนกลับฐานแล้ว 🏠")
	end
end)

-- ปุ่มเปลี่ยนกล้อง 10 โหมดแบบวนลูป
local CamBtn = CreateTabButton(Tab1Frame, "กล้อง: " .. CameraModes[1], Color3.fromRGB(100, 100, 120), function(btn)
	DroneState.CameraMode = (DroneState.CameraMode % 10) + 1
	btn.Text = "กล้อง: " .. CameraModes[DroneState.CameraMode]
	ShowNotification("สลับเป็น: " .. CameraModes[DroneState.CameraMode])
end)

CreateTabButton(Tab1Frame, "เทอร์โบ", Color3.fromRGB(50, 50, 60), function(btn)
	DroneState.Turbo = not DroneState.Turbo
	DroneState.CurrentSpeed = DroneState.Turbo and (DroneState.BaseSpeed * 2) or DroneState.BaseSpeed
	btn.BackgroundColor3 = DroneState.Turbo and Color3.fromRGB(0, 180, 216) or Color3.fromRGB(50, 50, 60)
	ShowNotification(DroneState.Turbo and "เปิดโหมดเทอร์โบ! ⚡" or "ปิดโหมดเทอร์โบ 🐢")
end)

-- ================= TAB 2 BUTTONS =================
CreateTabButton(Tab2Frame, "ปล่อย/เก็บ เชือก", Color3.fromRGB(0, 150, 200), function(btn)
	if not DroneState.Active or not DroneState.Model then
		ShowNotification("ต้องเปิดใช้งานโดรนก่อน! ⚠️")
		return
	end

	DroneState.RopeActive = not DroneState.RopeActive
	if DroneState.RopeActive then
		local char = GetCharacter()
		local hrp = char:FindFirstChild("HumanoidRootPart")
		if hrp then
			DroneState.AttachmentDrone = Instance.new("Attachment", DroneState.Model)
			DroneState.AttachmentPlayer = Instance.new("Attachment", hrp)

			local rope = Instance.new("RopeConstraint", DroneState.Model)
			rope.Attachment0 = DroneState.AttachmentDrone
			rope.Attachment1 = DroneState.AttachmentPlayer
			rope.Length = (DroneState.Model.Position - hrp.Position).Magnitude + 2
			rope.Visible = true
			rope.Color = BrickColor.new("Bright yellow")
			rope.Thickness = 0.15

			DroneState.RopeConstraint = rope
			DroneState.RopeLength = rope.Length
			btn.BackgroundColor3 = Color3.fromRGB(200, 50, 50)
			ShowNotification("ปล่อยเชือกนิรภัยเรียบร้อย 🪢")
		end
	else
		DestroyRope()
		btn.BackgroundColor3 = Color3.fromRGB(0, 150, 200)
		ShowNotification("เก็บเชือกนิรภัยแล้ว 🧶")
	end
end)

CreateTabButton(Tab2Frame, "ดึงเชือกขึ้น ▲", Color3.fromRGB(46, 125, 50), function()
	if DroneState.RopeConstraint then
		DroneState.RopeLength = math.max(3, DroneState.RopeLength - 4)
		DroneState.RopeConstraint.Length = DroneState.RopeLength
		ShowNotification("ดึงเชือกขึ้น (ความยาว: " .. math.floor(DroneState.RopeLength) .. "m)")
	end
end)

CreateTabButton(Tab2Frame, "หย่อนเชือกลง ▼", Color3.fromRGB(200, 120, 30), function()
	if DroneState.RopeConstraint then
		DroneState.RopeLength = math.min(100, DroneState.RopeLength + 4)
		DroneState.RopeConstraint.Length = DroneState.RopeLength
		ShowNotification("หย่อนเชือกออก (ความยาว: " .. math.floor(DroneState.RopeLength) .. "m)")
	end
end)

CreateTabButton(Tab2Frame, "ไฟ หน้า/หลัง", Color3.fromRGB(200, 160, 30), function()
	DroneState.FrontLightOn = not DroneState.FrontLightOn
	for _, l in ipairs(DroneState.FrontLights) do l.Enabled = DroneState.FrontLightOn end
	ShowNotification(DroneState.FrontLightOn and "เปิดไฟ หน้า/หลัง 💡" or "ปิดไฟ หน้า/หลัง 💡")
end)

CreateTabButton(Tab2Frame, "ไฟไซเรน", Color3.fromRGB(180, 40, 40), function(btn)
	DroneState.SirenOn = not DroneState.SirenOn
	btn.BackgroundColor3 = DroneState.SirenOn and Color3.fromRGB(0, 150, 255) or Color3.fromRGB(180, 40, 40)
	ShowNotification(DroneState.SirenOn and "เปิดไฟไซเรนฉุกเฉิน 🚨" or "ปิดไฟไซเรน 🚨")
end)

-- ================= TAB 3: TARGETING & GPS =================
CreateTabButton(Tab3Frame, "เลือกผู้เล่นเป้าหมาย", Color3.fromRGB(120, 50, 180), function(btn)
	local plrs = {}
	for _, p in ipairs(Players:GetPlayers()) do
		if p ~= LocalPlayer then table.insert(plrs, p) end
	end
	if #plrs == 0 then
		ShowNotification("ไม่มีผู้เล่นอื่นในเซิร์ฟเวอร์ ❌")
		return
	end
	
	local currentIndex = 1
	if DroneState.TargetPlayer then
		for idx, p in ipairs(plrs) do
			if p == DroneState.TargetPlayer then currentIndex = (idx % #plrs) + 1 break end
		end
	end
	
	DroneState.TargetPlayer = plrs[currentIndex]
	btn.Text = "เป้าหมาย: " .. DroneState.TargetPlayer.DisplayName
	ShowNotification("เลือกเป้าหมาย: " .. DroneState.TargetPlayer.DisplayName .. " 🎯")
end)

CreateTabButton(Tab3Frame, "โหมดบินตามเป้าหมาย", Color3.fromRGB(50, 60, 80), function(btn)
	if not DroneState.TargetPlayer then
		ShowNotification("ต้องเลือกผู้เล่นก่อน! ⚠️")
		return
	end
	DroneState.AutoFollow = not DroneState.AutoFollow
	btn.BackgroundColor3 = DroneState.AutoFollow and Color3.fromRGB(0, 180, 216) or Color3.fromRGB(50, 60, 80)
	ShowNotification(DroneState.AutoFollow and "เปิดโหมดบินตามเป้าหมาย 🛸" or "ปิดโหมดบินตาม ⏹️")
end)

CreateTabButton(Tab3Frame, "GPS นำทางไปหา", Color3.fromRGB(0, 150, 150), function()
	if DroneState.TargetPlayer and DroneState.TargetPlayer.Character and DroneState.TargetPlayer.Character:FindFirstChild("HumanoidRootPart") then
		local targetPos = DroneState.TargetPlayer.Character.HumanoidRootPart.Position + Vector3.new(0, 8, 0)
		DroneState.Position = targetPos
		ShowNotification("GPS นำทาง: วาร์ปโดรนไปหา " .. DroneState.TargetPlayer.DisplayName .. " 📍")
	else
		ShowNotification("ไม่พบตำแหน่งเป้าหมาย ❌")
	end
end)

-- ================= RADAR UI =================
local RadarFrame = Instance.new("Frame", ScreenGui)
RadarFrame.Name = "RadarFrame"
RadarFrame.Size = UDim2.new(0, 110, 0, 110)
RadarFrame.Position = UDim2.new(1, -125, 0, 15)
RadarFrame.BackgroundColor3 = Color3.fromRGB(10, 15, 25)
RadarFrame.BackgroundTransparency = 0.2
Instance.new("UICorner", RadarFrame).CornerRadius = UDim.new(1, 0)

local RadarStroke = Instance.new("UIStroke", RadarFrame)
RadarStroke.Color = Color3.fromRGB(0, 217, 255)
RadarStroke.Thickness = 1.5

local RadarCenter = Instance.new("Frame", RadarFrame)
RadarCenter.Size = UDim2.new(0, 6, 0, 6)
RadarCenter.Position = UDim2.new(0.5, -3, 0.5, -3)
RadarCenter.BackgroundColor3 = Color3.fromRGB(0, 255, 120)
Instance.new("UICorner", RadarCenter).CornerRadius = UDim.new(1, 0)

local RadarDots = {}
local function UpdateRadar()
	for _, dot in pairs(RadarDots) do dot:Destroy() end
	RadarDots = {}

	local centerPos = DroneState.Active and DroneState.Position or (GetCharacter():FindFirstChild("HumanoidRootPart") and GetCharacter().HumanoidRootPart.Position or Vector3.zero)

	for _, p in ipairs(Players:GetPlayers()) do
		if p ~= LocalPlayer and p.Character and p.Character:FindFirstChild("HumanoidRootPart") then
			local hrp = p.Character.HumanoidRootPart
			local diff = hrp.Position - centerPos
			local dist = Vector3.new(diff.X, 0, diff.Z).Magnitude

			if dist <= 300 then
				local relX = diff.X / 300 * 50
				local relZ = diff.Z / 300 * 50

				local dot = Instance.new("Frame", RadarFrame)
				dot.Size = UDim2.new(0, 5, 0, 5)
				dot.Position = UDim2.new(0.5, relX - 2.5, 0.5, relZ - 2.5)
				dot.BackgroundColor3 = (p == DroneState.TargetPlayer) and Color3.fromRGB(255, 0, 0) or Color3.fromRGB(255, 255, 0)
				Instance.new("UICorner", dot).CornerRadius = UDim.new(1, 0)
				table.insert(RadarDots, dot)
			end
		end
	end
end

-- STATUS PANEL
local StatusFrame = Instance.new("Frame", ScreenGui)
StatusFrame.Size = UDim2.new(0, 220, 0, 95)
StatusFrame.Position = UDim2.new(0, 15, 0, 15)
StatusFrame.BackgroundTransparency = 1

local StatusLabel = Instance.new("TextLabel", StatusFrame)
StatusLabel.Size = UDim2.new(1, 0, 1, 0)
StatusLabel.BackgroundTransparency = 1
StatusLabel.Text = "ผู้เล่น: " .. LocalPlayer.DisplayName .. "\nสถานะ: สแตนด์บาย\nความสูง: 0m | ความเร็ว: 0km/h\nผู้เล่นในเซิร์ฟ: 1 | แบตเตอรี่: 100%"
StatusLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
StatusLabel.TextSize = 11
StatusLabel.Font = Enum.Font.GothamBold
StatusLabel.TextXAlignment = Enum.TextXAlignment.Left
StatusLabel.TextYAlignment = Enum.TextYAlignment.Top

-- TOGGLE MENU BUTTON
local ToggleMenuBtn = Instance.new("ImageButton", ScreenGui)
ToggleMenuBtn.Size = UDim2.new(0, 42, 0, 42)
ToggleMenuBtn.Position = UDim2.new(0, 15, 0.4, 0)
ToggleMenuBtn.BackgroundColor3 = Color3.fromRGB(15, 15, 22)
ToggleMenuBtn.Image = "rbxassetid://10723416652"
ToggleMenuBtn.ScaleType = Enum.ScaleType.Fit
ToggleMenuBtn.Active = true
ToggleMenuBtn.Draggable = true
Instance.new("UICorner", ToggleMenuBtn).CornerRadius = UDim.new(0, 10)

local ToggleStroke = Instance.new("UIStroke", ToggleMenuBtn)
ToggleStroke.Color = Color3.fromRGB(0, 217, 255)
ToggleStroke.Thickness = 1.5

ToggleMenuBtn.MouseButton1Click:Connect(function()
	PlayClickSound()
	MainWindow.Visible = not MainWindow.Visible
	ShowNotification(MainWindow.Visible and "เปิดเมนูควบคุม" or "ซ่อนเมนูควบคุม")
end)

MinBtn.MouseButton1Click:Connect(function()
	PlayClickSound()
	TabBar.Visible = not TabBar.Visible
	Tab1Frame.Visible = TabBar.Visible and Tab1Frame.Visible
	Tab2Frame.Visible = TabBar.Visible and Tab2Frame.Visible
	Tab3Frame.Visible = TabBar.Visible and Tab3Frame.Visible
	MainWindow.Size = TabBar.Visible and UDim2.new(0, 450, 0, 160) or UDim2.new(0, 450, 0, 32)
	ShowNotification(TabBar.Visible and "ขยายหน้าต่าง" or "ย่อหน้าต่าง")
end)

CloseBtn.MouseButton1Click:Connect(function()
	PlayClickSound()
	LandAndStop(FlyBtn)
	MainWindow.Visible = false
end)

-- ================= DRONE MODEL BUILDER =================
local function SpawnDrone()
	if DroneState.Model then DroneState.Model:Destroy() end
	DroneState.FrontLights = {}
	DroneState.SirenBarLights = {}
	DroneState.Propellers = {}

	local char = GetCharacter()
	local root = char:FindFirstChild("HumanoidRootPart")
	if not root then return end

	DroneState.Position = root.Position + root.CFrame.LookVector * 6 + Vector3.new(0, 4, 0)
	DroneState.Yaw = math.atan2(-root.CFrame.LookVector.X, -root.CFrame.LookVector.Z)

	local drone = Instance.new("Part")
	drone.Name = "MAC_Custom_Drone"
	drone.Size = Vector3.new(3, 0.6, 3)
	drone.Color = Color3.fromRGB(25, 25, 30)
	drone.Material = Enum.Material.SmoothPlastic
	drone.CFrame = CFrame.new(DroneState.Position) * CFrame.Angles(0, DroneState.Yaw, 0)
	drone.CanCollide = false
	drone.Anchored = true
	drone.Parent = workspace

	local frontSpot = Instance.new("SpotLight", drone)
	frontSpot.Face = Enum.NormalId.Front
	frontSpot.Color = Color3.fromRGB(255, 255, 255)
	frontSpot.Range = 60
	frontSpot.Brightness = 8
	frontSpot.Enabled = false
	table.insert(DroneState.FrontLights, frontSpot)

	local sirenBar = Instance.new("Part", drone)
	sirenBar.Size = Vector3.new(1.8, 0.2, 0.3)
	sirenBar.Color = Color3.fromRGB(40, 40, 40)
	sirenBar.CFrame = drone.CFrame * CFrame.new(0, 0.4, 0)
	sirenBar.CanCollide = false

	local wSiren = Instance.new("WeldConstraint", drone)
	wSiren.Part0 = drone; wSiren.Part1 = sirenBar

	for i, col in ipairs({Color3.fromRGB(255, 0, 0), Color3.fromRGB(0, 100, 255)}) do
		local bulb = Instance.new("Part", sirenBar)
		bulb.Size = Vector3.new(0.8, 0.25, 0.35)
		bulb.Color = col
		bulb.Material = Enum.Material.Neon
		bulb.CFrame = sirenBar.CFrame * CFrame.new(i == 1 and -0.45 or 0.45, 0, 0)
		bulb.CanCollide = false

		local wb = Instance.new("WeldConstraint", sirenBar)
		wb.Part0 = sirenBar; wb.Part1 = bulb

		local pLight = Instance.new("PointLight", bulb)
		pLight.Color = col
		pLight.Brightness = 0
		pLight.Range = 15

		table.insert(DroneState.SirenBarLights, {Part = bulb, Light = pLight})
	end

	local offsets = {Vector3.new(2, 0.3, 2), Vector3.new(-2, 0.3, 2), Vector3.new(2, 0.3, -2), Vector3.new(-2, 0.3, -2)}
	for _, off in ipairs(offsets) do
		local prop = Instance.new("Part", drone)
		prop.Size = Vector3.new(2.4, 0.05, 0.3)
		prop.Color = Color3.fromRGB(0, 217, 255)
		prop.Material = Enum.Material.Neon
		prop.CanCollide = false

		table.insert(DroneState.Propellers, {Part = prop, Offset = off})
	end

	DroneState.Model = drone
end

-- KEYBOARD BINDINGS
local KeyMap = {
	[Enum.KeyCode.W] = "Forward", [Enum.KeyCode.S] = "Backward",
	[Enum.KeyCode.A] = "Left",    [Enum.KeyCode.D] = "Right",
	[Enum.KeyCode.E] = "Up",      [Enum.KeyCode.Q] = "Down",
	[Enum.KeyCode.Z] = "YawLeft", [Enum.KeyCode.C] = "YawRight"
}

UserInputService.InputBegan:Connect(function(input, g)
	if not g and KeyMap[input.KeyCode] then UIInputs[KeyMap[input.KeyCode]] = true end
end)
UserInputService.InputEnded:Connect(function(input)
	if KeyMap[input.KeyCode] then UIInputs[KeyMap[input.KeyCode]] = false end
end)

-- ================= MAIN UPDATE LOOP =================
local sirenTimer = 0
RunService.Heartbeat:Connect(function(delta)
	UpdateRadar()

	local plrCount = #Players:GetPlayers()
	if not DroneState.Active then
		StatusLabel.Text = string.format("ผู้เล่น: %s\nสถานะ: ปิดการทำงาน\nความสูง: 0m | ความเร็ว: 0km/h\nผู้เล่นในเซิร์ฟ: %d คน | แบต: 100%%", LocalPlayer.DisplayName, plrCount)
		return
	end

	if not DroneState.Model then SpawnDrone() end

	local drone = DroneState.Model
	DroneState.Battery = math.max(0, DroneState.Battery - (delta * 0.05))

	-- REALISTIC PHYSICS ENGINE (ระบบจำลองแรงเฉื่อยและการเอียงสมจริง)
	local fwd = (UIInputs.Forward and 1 or 0) - (UIInputs.Backward and 1 or 0)
	local rgt = (UIInputs.Right and 1 or 0) - (UIInputs.Left and 1 or 0)
	local up  = (UIInputs.Up and 1 or 0) - (UIInputs.Down and 1 or 0)
	local yw  = (UIInputs.YawLeft and 1 or 0) - (UIInputs.YawRight and 1 or 0)

	-- คำนวณความเร็วหมุนและแรงเหวี่ยง
	DroneState.YawVelocity = math.clamp(DroneState.YawVelocity + (yw * 3 - DroneState.YawVelocity * 4) * delta, -2, 2)
	DroneState.Yaw = DroneState.Yaw + (DroneState.YawVelocity * delta)

	local targetVel = CFrame.Angles(0, DroneState.Yaw, 0):VectorToWorldSpace(Vector3.new(rgt, 0, -fwd)) * DroneState.CurrentSpeed
	targetVel = targetVel + Vector3.new(0, up * DroneState.CurrentSpeed * 0.8, 0)

	-- แรงเฉื่อยแบบค่อยๆ เคลื่อนตาม (Inertia & Smooth Acceleration)
	DroneState.Velocity = DroneState.Velocity:Lerp(targetVel, delta * 4)
	DroneState.Position = DroneState.Position + (DroneState.Velocity * delta)

	-- การเอียงของตัวเครื่องตามแรงเหวี่ยง (Pitch & Roll Tilt)
	local localVel = CFrame.Angles(0, DroneState.Yaw, 0):VectorToObjectSpace(DroneState.Velocity)
	DroneState.Pitch = math.clamp(-localVel.Z * 0.015, -0.35, 0.35)
	DroneState.Roll  = math.clamp(-localVel.X * 0.015, -0.35, 0.35)

	drone.CFrame = CFrame.new(DroneState.Position) * CFrame.Angles(0, DroneState.Yaw, 0) * CFrame.Angles(DroneState.Pitch, 0, DroneState.Roll)

	-- หมุนใบพัด
	DroneState.Spin = (DroneState.Spin + delta * 60) % (math.pi * 2)
	for _, p in ipairs(DroneState.Propellers) do
		p.Part.CFrame = drone.CFrame * CFrame.new(p.Offset) * CFrame.Angles(0, DroneState.Spin, 0)
	end

	-- ไฟไซเรน
	if DroneState.SirenOn then
		sirenTimer += delta * 10
		local state = math.floor(sirenTimer) % 2 == 0
		if DroneState.SirenBarLights[1] then DroneState.SirenBarLights[1].Light.Brightness = state and 10 or 0 end
		if DroneState.SirenBarLights[2] then DroneState.SirenBarLights[2].Light.Brightness = not state and 10 or 0 end
	else
		for _, s in ipairs(DroneState.SirenBarLights) do s.Light.Brightness = 0 end
	end

	-- ================= CAMERA 10 MODES SYSTEM =================
	local cam = workspace.CurrentCamera
	cam.CameraType = Enum.CameraType.Scriptable
	local myRoot = GetCharacter():FindFirstChild("HumanoidRootPart")

	if DroneState.CameraMode == 1 then -- 1. FPV มุมมองนักบิน
		cam.CFrame = drone.CFrame * CFrame.new(0, 0.2, -1.5)
	elseif DroneState.CameraMode == 2 then -- 2. Third-Person ตามหลัง
		cam.CFrame = drone.CFrame * CFrame.new(0, 3, 10) * CFrame.Angles(math.rad(-10), 0, 0)
	elseif DroneState.CameraMode == 3 then -- 3. Bottom-View ส่องใต้ท้อง
		cam.CFrame = drone.CFrame * CFrame.new(0, -1, 0) * CFrame.Angles(math.rad(-90), 0, 0)
	elseif DroneState.CameraMode == 4 then -- 4. Selfie หันมองผู้เล่น
		if myRoot then cam.CFrame = CFrame.new(drone.Position, myRoot.Position) end
	elseif DroneState.CameraMode == 5 then -- 5. Cinematic Orbit หมุนรอบตัว
		DroneState.OrbitAngle = (DroneState.OrbitAngle + delta * 0.8) % (math.pi * 2)
		local offset = Vector3.new(math.cos(DroneState.OrbitAngle) * 12, 4, math.sin(DroneState.OrbitAngle) * 12)
		cam.CFrame = CFrame.new(drone.Position + offset, drone.Position)
	elseif DroneState.CameraMode == 6 then -- 6. Top-Down Scan มุมมองดาวเทียม
		cam.CFrame = CFrame.new(drone.Position + Vector3.new(0, 35, 0)) * CFrame.Angles(math.rad(-90), 0, 0)
	elseif DroneState.CameraMode == 7 then -- 7. Thermal/Night โหมดกลางคืน (ซูมเข้า)
		cam.CFrame = drone.CFrame * CFrame.new(0, 0.5, -2)
	elseif DroneState.CameraMode == 8 then -- 8. Lock Target ล็อกเป้าหมาย
		if DroneState.TargetPlayer and DroneState.TargetPlayer.Character and DroneState.TargetPlayer.Character:FindFirstChild("HumanoidRootPart") then
			cam.CFrame = CFrame.new(drone.Position, DroneState.TargetPlayer.Character.HumanoidRootPart.Position)
		else
			cam.CFrame = drone.CFrame * CFrame.new(0, 0, -1.5)
		end
	elseif DroneState.CameraMode == 9 then -- 9. Ultra Wide มุมกว้างพิเศษ
		cam.CFrame = drone.CFrame * CFrame.new(0, 5, 18) * CFrame.Angles(math.rad(-15), 0, 0)
	elseif DroneState.CameraMode == 10 then -- 10. Side-Chase ขนานข้าง
		cam.CFrame = drone.CFrame * CFrame.new(12, 2, 0) * CFrame.Angles(0, math.rad(90), 0)
	end

	DroneState.Height = math.floor(drone.Position.Y)
	DroneState.SpeedCalc = math.floor(DroneState.Velocity.Magnitude)
	StatusLabel.Text = string.format("ผู้เล่น: %s\nสถานะ: %s\nความสูง: %dm | ความเร็ว: %dkm/h\nผู้เล่นในเซิร์ฟ: %d คน | แบต: %d%%", LocalPlayer.DisplayName, DroneState.IsLanding and "กำลังลงจอด..." or "บินอยู่", DroneState.Height, DroneState.SpeedCalc, plrCount, math.floor(DroneState.Battery))
end)
